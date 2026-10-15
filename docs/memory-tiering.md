# 内存分层（Memory Tiering）架构与使用

> 把客户机的**热页**留在宿主 RAM、**冷页**换出到后端块设备（NVMe/文件），访问时
> 按需换回，从而对宿主内存做**超分（oversubscription）**。完全在 QEMU 内用
> `userfaultfd` + `io_uring` 实现，**不改 KVM**；由 xvirt 控制面编排、主机级总控
> 按优先级在内存预算内实时调度。
>
> 适用场景：单机/集群上跑的虚机内存总和超过物理内存，用「冷页下刷」把不活跃内存
> 让给活跃负载，提高装箱密度。

---

## 1. 总体架构

内存分层横跨 5 层，机制在 QEMU、编排在 xvirt：

```
浏览器（xvirt-client）
  └─ 新建虚拟机向导：内存分层开关 + 策略下拉
  └─ VM 详情 / 监控：内存分层统计卡（每 5s 拉一次）
        │  REST
        ▼
ferrovirt（控制面 / DB owner）
  └─ POST /api/vcenter/vm  { memtier:{enabled,policy,priority,tunables?} }
  └─ GET  /api/vcenter/vm/{vm}/memtier   →  x-memtier-query 透传
  └─ 落库 vm_config.memtier_*；编辑时回放；与 petrel 盘互斥（HTTP 400）
        │  ZMQ（瘦代理）
        ▼
hyper-agent（宿主机底层服务）
  └─ create_vm/update_vm：写 <vm_dir>/memtier.json sidecar + 预分配后端文件
  └─ 主机级总控环 support/memtier/controller.py（后台线程）★
        每 5s：读本机内存压力 + 各 VM 统计 → 按优先级算每 VM 边界 → set-policy
        │  ZMQ → lictor :5560
        ▼
lictor（持 QMP）
  └─ 启动后 _memtier_enable_on_start：读 sidecar → x-memtier-enable
  └─ actions：memtier_query / memtier_set_policy / memtier_disable
        │  QMP
        ▼
QEMU（memtier 子系统，分支 memory-tiering）
  └─ uffd-wp + io_uring；per-VM worker：实时热度扫描 + 换出 + 换入
```

★ = 「底层内存分层服务」就是这个主机级总控环（见 §4），**不在 concilium 集群服务里**。

---

## 2. QEMU 内核机制（memtier 子系统）

> 裸 QEMU 层面的能力说明 + QMP 接口逐条用法（不经 xvirt 编排）见独立文档
> [`qemu-memtier-usage.md`](./qemu-memtier-usage.md)。本节只做架构层面的概述。

代码在 QEMU 工作树 `memtier/`（分支 `memory-tiering`），产物即编进
`qemu-system-x86_64`（需 `--enable-linux-io-uring` + 运行时 `CAP_SYS_ADMIN`）。

### 2.1 数据流与页状态机

```
空闲扫描挑出冷页
  → uffd 写保护该段
  → io_uring O_DIRECT 顺序大块写到后端设备
  → 确认未被重新写脏后 MADV_DONTNEED 释放宿主页
     （内核 mmu_notifier 让 KVM 同步 zap 掉 EPT 项）
  → 客户机再访问 → uffd 缺页打回 QEMU
  → io_uring 读回该页 + UFFDIO_COPY 填回
```

| 状态 | 含义 |
|---|---|
| `RESIDENT` | 在宿主 RAM、可写 |
| `EVICTING` | 在 RAM 但已写保护；写 → WP 缺页 |
| `EVICTING_DIRTIED` | 换出途中被写 → 回滚 |
| `SWAPPED` | 已 `MADV_DONTNEED`；任何访问 → MISSING 缺页 |
| `FAULTING_IN` | 读回在途；并发访问者串行等待 |

### 2.2 实时热度获取（scan.c）

判冷靠内核 `page_idle`：把页标记 idle → 等一个 100ms 窗口 → 回读，仍 idle 即
窗口内未经 EPT 访问过 = 真冷（`page_idle`/pagemap 的 young 位是 mmu-notifier 感知
的，能抓**读热**，弥补 uffd-wp 只看写的盲区）。需 `/proc/self/pagemap` +
`/sys/kernel/mm/page_idle/bitmap`（CAP_SYS_ADMIN）。

**关键设计（实时性）**：100ms 是个**墙钟时间窗口**，对窗口内被标记的所有页共享。
因此 `memtier_scan_range_cold()` 一次标记**一整批 cell**、只睡**一个**窗口、再回读
→ 得到整段近乎同一时刻的**连贯热度快照**。PFN 用 pagemap **批量读**、page_idle
位图按 PFN 词跨度做**一次性批量读-改-写**（整批几个 syscall，取代每页 2 个 syscall）。

> 历史：早期逐 cell 扫描（每 2MB 一个独立 100ms 窗口、round-robin）冷页探测仅
> ~6 MiB/s，多 GB region 要几分钟才扫一遍。批量化后整 region 秒级出快照，详见 §8。

### 2.3 换出（worker.c）

每个分层 region 一个 worker 线程：
- 每拍按「到常驻下限的差额」自适应批量大小（`SCAN_CELLS_MIN=8` .. `SCAN_CELLS_MAX=4096` cell，即 16MB..8GB），压力下一拍快照整个 region。
- 把 coldmap 切成最大连续冷段，每段一次 `io_uring` 大块写换出；写保护是唯一同步原语，写完在分片锁下判净/脏（脏则回滚）。
- 受**常驻下限**（min-resident）约束：到下限即停。
- **cold-runs 去抖**：一个 cell 需连续被判冷 N 次才换出。因为批量化后每拍都采样整
  region，N 次去抖在 **~N 个窗口（约 N×0.1s+）** 内完成，既准（不误换偶发热页）又快。
- `evict-mbps` 统计 + `max-evict-mbps` 滚动窗口限速。

### 2.4 换入（fault.c）

uffd poll 循环：MISSING 缺页 → io_uring 读回 → UFFDIO_COPY 填回 → 状态回 RESIDENT
并记 `fault_tick`（供 cooldown 去抖：刚换回的页 cooldown_ms 内不再换出，防抖动）。

---

## 3. QMP 接口（实验性 `x-`）

| 命令 | 作用 |
|---|---|
| `x-memtier-enable {*id, path, *policy, *tunables}` | 对一个 RAM region 开启分层。`path`=后端 raw 设备/文件（O_DIRECT）；`id` 缺省=主 guest RAM 块 |
| `x-memtier-query {*id}` | 返回各 region 的状态 + 累计统计（`MemTierInfo` 列表） |
| `x-memtier-set-policy {*id, *policy, *tunables}` | 运行中动态改策略/旋钮（总控用） |
| `x-memtier-disable {*id}` | 先把所有换出页强制拉回 RAM，再拆除分层（迁移/快照前应先 disable 收敛） |

`MemTierInfo` 字段：`total/resident/swapped-bytes`、`backing-total/used/largest-free-bytes`、
`evicted-pages`、`evict-aborts`、`swapin-faults`、`swapin-errors`、`evict-mbps`、`psi-some-avg10`，
换入延迟分解 `swapin-lat-samples / swapin-avg-{wait,device,publish}-ns / swapin-max-ns`，以及
后端写入统计（NVMe 磨损研究，见 [`memtier-nvme-wear.md`](./memtier-nvme-wear.md)）：
`evict-ios / evict-io-bytes / evict-io-hist`（写次数、字节、按 4K…1M+ 分桶的大小直方图）、
`evict-zero-pages`（换出时全零、未写后端的页）、`zero-fills`（用零页填回的缺页）、`evict-reevict-pages`（换入后再次换出的页）、
`evict-sampled-pages / evict-sampled-clean-pages`（再换出页与换入时哈希比对，未变的比例）；
swap-cache：`clean-resident-bytes`（驻留且仍持有 slot 的页）、`evict-clean-pages`（免写的再换出）、
`evict-verify-reads`、`clean-slot-reclaims`；诊断：`wp-faults-evicting / wp-faults-other`。
旋钮 `swapcache-verify`（默认关）让哈希相等的页再读回逐字节比对。

### 3.1 策略预设

| 旋钮 | conservative | balanced | aggressive |
|---|---|---|---|
| cold-threshold | 0 | 2 | 8 |
| cold-runs | 8 | 5 | 2 |
| psi-trigger (%) | 20 | 10 | 2 |
| max-evict-mbps | 200 | 500 | 1000 |
| 常驻下限 | 75% | 50% | 25% |
| cooldown-ms | 2000 | 1000 | 500 |

`tunables` 可逐项覆盖预设：`cold-threshold / cold-runs / psi-trigger / max-evict-mbps /
min-resident-mb / cooldown-ms`。

> **何时换出**：`psi >= psi-trigger` 且 `resident > min-resident`。把 `psi-trigger=0`
> 即「无视主机压力立刻换出到下限」（测试/总控强制换出用）。

---

## 4. 主机级总控（底层内存分层服务，模型 A）

**这就是用户所说的「底层内存分层服务」。** QEMU 内每 VM 的 worker 仍做实际换出/换入，
但**主机唯一的大脑**是 hyper-agent 里的总控环 `support/memtier/controller.py`
（boot 时起后台守护线程，与 petrel-reconcile 同模式）。

它独占「按主机全局压力决定**谁换出、换到多深**」，避免 N 个 VM 的 worker 各自盯全局
PSI 抢着换。每 5s 一拍：

1. 扫 `/datastore/*/<uuid>/memtier.json` 找本机启用分层的 VM（含优先级）。
2. 读本机内存压力 → 合成 0..1：`max(PSI_some_avg10 / PSI_HIGH, (FREE_LOW_PCT - MemAvailable%) / FREE_LOW_PCT)`。
3. 对每个**正在运行**的 VM（经 lictor `memtier_query` 确认有 region）按 (压力, 优先级)
   算 `(psi-trigger, min-resident-mb, max-evict-mbps)`，经 lictor `memtier_set_policy` 下发：
   - **无压力** → `psi-trigger=100`（暂停换出）+ 常驻下限=全量；已换出页按访问自然换回。
   - **有压力** → `psi-trigger=0` + 常驻下限按优先级压低（high 保 75% / normal 50% /
     low 25%，再随压力线性下探，最多压到基准的 20%）+ 换出速率（低优先级换得更快）。

优先级缺省时按 policy 预设回退：`aggressive→low / balanced→normal / conservative→high`。

**回血（主动换回）**：总控对称地管换出与回血，用两条水位线 `FREE_LOW_PCT`(10，以下→换出) 与
`FREE_HIGH_PCT`(50，以上→回血)，中间死区仅被动，防振荡。有富余时把每 VM 的 `target-resident-mb`
按 富余×优先级权重 抬向全量、下发 `swapin-watermark`（引擎钳制 < psi-trigger）→ QEMU 把最近换出的
页预取回 RAM（消缺页尾延迟）；压力回来 target 压回 min、回血自动停。详见
[`memtier-heat-and-swapback.md`](./memtier-heat-and-swapback.md)。

环境变量调阈值：`XVIRT_MEMTIER_INTERVAL`(默认 5s) / `XVIRT_MEMTIER_PSI_HIGH`(10) /
`XVIRT_MEMTIER_FREE_LOW_PCT`(10) / `XVIRT_MEMTIER_FREE_HIGH_PCT`(50)。

**NVMe 写入预算**（`support/memtier/write_budget.py`，详见 [`memtier-nvme-wear.md`](./memtier-nvme-wear.md) §5）：
按后端盘记当天分层写入量，把「剩余预算 / 剩余时间」按优先级分给同盘 VM 作 `max-evict-mbps`
上限，主机压力 ≥ 0.9 时越过。配置 plan `memory_tiering.write_budget_gb_per_day` 或
`XVIRT_MEMTIER_WRITE_BUDGET_GB_PER_DAY`，0 = 不限。

### 为什么放 hyper-agent 而**不放 concilium**

分层是**纯主机本地**的事（受本机 RAM + 本机后端盘 + 本机 QEMU 约束，没有任何跨主机
动作），且节点被网络分区时**必须继续缓解内存压力**否则 OOM。concilium 是集群域
（quorum / 放置 / HA，少数派会 self-fence，且刻意不直连 QMP）。把一个本地快环塞进
集群共识守护进程是层级与可用性双重错配。concilium 只在**生命周期触点**接触分层——例如
迁移前由 migrate RA 触发 `x-memtier-disable` 收敛 RAM（该步仍由 hyper-agent 执行）。

---

## 5. 控制面与持久化

- **DB**（`vm_config`）：`memtier_enabled` / `memtier_policy` / `memtier_backing_path` /
  `memtier_priority`。建 VM 落库；编辑 VM 时 `regen_inner` 回放，避免丢 sidecar。
- **sidecar**（`<vm_dir>/memtier.json`，hyper-agent 写）：`{enabled, policy, priority, path,
  tunables}`，是 agent 侧运行时真相 —— lictor 启动时读它 `x-memtier-enable`，总控读它拿优先级。
- **后端文件**：`x-memtier-enable.path` 缺省时 hyper-agent 在 `<vm_dir>/memtier-backing.raw`
  按**客户机内存大小** `fallocate`（可容纳 100% 换出）；也可在请求里给 `path` 指向专用
  NVMe 设备/分区。

---

## 6. 使用方法

### 6.1 前端向导（推荐）

「新建虚拟机」→「自定义硬件 → 高级」：
- **内存分层** 开关：勾选即启用。
- **分层策略** 下拉：保守 / 均衡（推荐）/ 激进。
- 选了 **petrel 磁盘**时开关自动置灰（互斥，见 §7）。

VM「监控」tab 有**内存分层卡片**（每 5s 刷新）：常驻/换出占比条 + 已换出页 / 换入缺页 /
换入错误 / 换出速率 / 主机 PSI / 后端用量 / 后端设备路径。

### 6.2 REST API

```bash
# 创建带内存分层的 VM（policy 可选 conservative|balanced|aggressive；priority 可选 high|normal|low）
POST /api/vcenter/vm
{
  "host": "...", "name": "...", "datastore": "<lvm/local ds>",
  "vcpu":2, "memory":4096, "firmware_type":"bios",
  "disks":[{"type":"file","device":"disk","datastore":"<lvm/local>","size":20,"create":true}],
  "memtier": { "enabled": true, "policy": "balanced", "priority": "low",
               "path": "/dev/nvme1n1p3",            // 可选；不填则虚机目录内自动预分配
               "tunables": { "min-resident-mb": 1024 } }   // 可选
}

# 查询运行时分层统计（关机/不支持不报 4xx，返回软状态）
GET /api/vcenter/vm/{vm}/memtier
→ { "configured":true, "policy":"balanced", "running":true,
    "regions":[ { ...MemTierInfo... } ], "error":null }
```

### 6.3 运行时手动调（绕过总控，调试用）

经 hyper-agent ZMQ → lictor（vm_id = vm uuid）：
- `memtier_set_policy {vm_id, policy?, tunables?}` —— 改策略/旋钮
- `memtier_disable {vm_id}` —— 拉回全部换出页并停止分层

> 注意：总控每 5s 会按全局压力重新下发 set-policy，手动改的旋钮可能被它覆盖。要纯手动
> 控制可临时停掉 `memtier-controller` 线程（重启 hyper-agent 恢复）或调高 `psi-trigger`。

---

## 7. 约束与注意事项

- **必须匿名内存**：分层目标是主 guest RAM 块（普通 `-m N` 的匿名内存，并被强制
  `MADV_NOHUGEPAGE`）。
- **与 petrel 盘互斥**：petrel 走 vhost-user-blk，需 `share=on` 的 memfd 共享内存；
  memtier 需匿名内存。两者不能共存——ferrovirt 在建 VM 时**直接拒绝**（HTTP 400），
  lictor 启动时也会跳过。memtier VM 用 lvm/local/file 盘。
- **运行权限**：QEMU 进程需 `CAP_SYS_ADMIN`（读 pagemap + page_idle 位图）。xvirt 下
  QEMU 以 root 跑，满足。
- **内核**：Linux 6.x（匿名内存 uffd-wp，`UFFD_FEATURE_PAGEFAULT_FLAG_WP`）；构建需
  `liburing`（`--enable-linux-io-uring`）。
- **后端容量**：后端文件/设备需容纳预期换出量（自动预分配 = 全 RAM 大小）。后端满会发
  `MEMTIER_BACKING_FULL` 事件并暂停换出（无正确性影响）。
- **迁移/快照**：在线迁移前应 `x-memtier-disable` 先收敛（共享后备除外）。内存快照改为只导出常驻页 + 分层文件副本，恢复时可继续分层或全部导入，设计见 [`memtier-snapshot-design.md`](./memtier-snapshot-design.md)。
- **大页未支持**：目标 region 被强制 4K（`MADV_NOHUGEPAGE`）。

---

## 8. 性能与验证（2026-06-03）

测试：4GB 真实 Linux 客户机，工作负载填充 3GB（其中 256MB 持续保持热），aggressive、
常驻下限 512MB、psi-trigger=0 强制换出。

| 指标 | 改进前 | 改进后 |
|---|---|---|
| 冷页探测速率 | ~6 MiB/s（扫描受限） | 整批快照，秒级 |
| 到 >50% 下刷 | **4 分钟都没到** | **21 秒** |
| 终态 | — | 常驻 727MB / 换出 3368MB（**~82% 下刷**） |
| 热度准确性（换入抖动） | — | 低（cold-runs=2 保住热集；cold-runs=1 抖动约高 16 倍） |
| 数据完整性 | — | 400MB 换出后重读 **sha256 逐字节一致** |
| evict-mbps 统计 | 恒 0（bug） | 正常 |

### 8.1 主机级总控在真实压力下自动压缩（完整整合 E2E）

经 REST API 建一个 xvirt 托管 VM（4GB，aggressive、优先级 low），叠加 node.qcow2 使其能
启动真实 Ubuntu 并注入开机自启的 memhog（填 3GB、保 256MB 热）；启动走 API（lictor 启动时
`x-memtier-enable`，运行中的 hyper-agent 总控经 sidecar 自动纳管）。**全程由总控驱动，无任何
手工 QMP**：

| 阶段 | 主机 free | 总控自动决策 | VM 状态 |
|---|---|---|---|
| 无压力（灌内存中） | 91%→79% | **暂停** psi-trigger=100 / 下限=全量 | resident 4096MiB（不换出，正确） |
| **真实加压**（12GB 主机内存 hog） | 79%→**50%**（< 阈值 70%） | **立即** psi-trigger=0 / 下限≈700MiB | resident 4096**→~790MiB**、swapped **~3300MiB（~81% 下刷），约 10 秒** |
| 释放压力（杀 hog） | →89% | **回落** psi-trigger=100 / 下限=全量 | 已换出页留盘上（按访问惰性换回） |

证明完整链路：REST 建 VM（memtier）→ lictor `x-memtier-enable` → 总控读**真实
`/proc/meminfo`** 算压力 → lictor `memtier_set_policy` → QEMU 批量扫描快速换出。即内存超分
闭环：压力来了自动把低优先级 VM 压到内存预算内，压力走了自动松手。

> 复现要点：阈值用 systemd drop-in `XVIRT_MEMTIER_FREE_LOW_PCT=70`（默认 10 偏保守，便于
> 用较小安全的主机 hog 触发，测完恢复）；让 xvirt VM 可启动 = 把 node.qcow2 叠加到
> `/datastore/<seg>/<vm>/<disk>.qcow2` 并用 qemu-nbd 注入一个 memhog systemd unit（免 SSH）。

---

## 9. 部署

- **QEMU**：分支 `memory-tiering`（HEAD 含 batched-scan 修复）。重建 deb：
  `build-deb/build_deb.sh ../qemu memory-tiering` → `dpkg -i` 装
  `qemu-system-*` / `qemu-utils` 等整套。装好后控制器 + xvirt 托管虚机新启动即用改进版。
- **xvirt 组件**：lictor / hyper-agent / ferrovirt / xvirt-client 用 `./xvirtctl deploy <comp>`；
  `vm_config` 新列在 `build-deb-xvirt/assets/ferrovirt-schema.sql`（新装自带；老库 `ALTER TABLE`）。
- **日志**：QEMU memtier 警告在虚机日志；总控环日志在 `journalctl -u xvirt-hyper-agent`
  （搜 `memtier`）；lictor 在 `journalctl -u xvirt-lictor`。

---

## 10. 代码索引

| 层 | 文件 | 关键符号 |
|---|---|---|
| QEMU | `qemu/memtier/{memtier,scan,worker,fault,slot-alloc}.c`、`qapi/memtier.json` | `memtier_scan_range_cold` / `memtier_worker` / `memtier_evict_run` |
| lictor | `lictor/lictor/vm/guest.py`、`controller/vm.py` | `_memtier_enable_on_start` / `memtier_query` / `memtier_set_policy` / `memtier_disable` |
| hyper-agent | `hyper_agent/vm/guest.py`、`support/memtier/controller.py`、`controller/vm.py`、`shell.py` | `_write_memtier_sidecar` / `controller.tick` / `run_loop` |
| ferrovirt | `src/service/vm_create.rs`、`vm_update.rs`、`service/write.rs`、`controller/vcenter/vm.rs` | `complete`/`persist_vm`/`regen_inner` 的 memtier 分支、`write::memtier_query` |
| 前端 | `xvirt-client/src/components/wizards/NewVmWizard.jsx`、`components/vm/VMMonitor.jsx`、`api/vcenter.js` | `AdvancedFields` memtier 控件、`MemtierCard`、`vm.memtier` |

---

## 11. 已知限制 / 后续

- 总控仅按「优先级 + 主机压力」做粗粒度仲裁（模型 A：总控调边界 + worker 在边界内执行）；
  更细的中心化逐页配额（模型 B）未做。
- 后端容量/带宽未纳入放置调度（cluster scheduler 暂不感知分层余量）。
- concilium 迁移前 `disable` 触点待随 migrate RA 落地。
- 内存快照与分层的整合（只导出常驻页、快照带分层文件、恢复多场景）为设计稿，见 [`memtier-snapshot-design.md`](./memtier-snapshot-design.md)。
- DAMON 作为粗粒度 oracle（cell_heat）未接；当前总控用 page_idle 细扫 + 主机 PSI。
- 主机内存预算/阈值尚未在 ferrovirt/UI 暴露（目前是 hyper-agent 常量 + 环境变量）。
- 前端向导暂未暴露**优先级**下拉（可经 API `memtier.priority` 设置；缺省按 policy 回退）。
