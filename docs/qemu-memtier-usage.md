# QEMU 内存分层（memtier）使用文档

> 本文讲**裸 QEMU** 层面的内存分层能力与 QMP 接口用法 —— 即不经 xvirt 编排，直接对一台
> `qemu-system-x86_64` 用 `x-memtier-*` QMP 命令做内存超分。xvirt 如何自动编排这些命令
> （前端开关、主机级总控按优先级实时调度）见 [`memory-tiering.md`](./memory-tiering.md)；
> 本文是它依赖的底层机制与接口参考，也可独立用于裸 QEMU / 自研管理面。
>
> QEMU 分支：`memory-tiering`（xvirt fork）。子系统代码在工作树 `memtier/`，设计概览另见
> 工作树内 `docs/devel/memory-tiering.rst`。

---

## 1. 能力概述

把客户机 RAM 的**热页**留在宿主物理内存、**冷页**换出到后端块设备（NVMe/文件），客户机
再访问冷页时按需换回 —— 从而让虚机内存总和**超过宿主物理内存（oversubscription）**。

- **完全在 QEMU 内实现**：`userfaultfd`（写保护 + 缺页）+ `io_uring`（O_DIRECT 换出/换入）。
  **不改 KVM**；靠内核 mmu_notifier 让 KVM 同步失效 EPT 项。
- **热度真实**：用内核 `page_idle` 跟踪，能抓**读热**（不只是写），mmu_notifier 感知 EPT 访问。
- **数据无损**：换出页被访问 → uffd 缺页 → 读回填充；实测换出后重读 **sha256 逐字节一致**。
- **实测**（4GB 客户机、3GB 工作集、aggressive、强制换出）：流式换出，**~21s 下刷 >50%**，
  最终压到常驻 ~730–790MiB（**~80%+ 下刷**），换入零错误。

### 能力边界（能做 / 不能做）

| 能 | 不能 / 限制 |
|---|---|
| 对**主 guest RAM 块**做冷页换出/换回 | 只支持**匿名内存**（普通 `-m`）；不支持 memfd/file-backed/共享内存（→ 与 vhost-user 设备如 vhost-user-blk 互斥） |
| 运行中动态启停、改策略、查统计 | 目标 region 被强制 **4K 页**（`MADV_NOHUGEPAGE`），暂不支持大页 |
| 按主机 PSI/压力自动调速（在 QEMU 内 worker） | 跨主机不感知（纯单机；迁移前应 disable 收敛） |
| 读热/写热都识别 | 需运行时 **CAP_SYS_ADMIN**（读 pagemap + page_idle 位图） |
| 任意 raw 块设备或预分配 raw 文件作后端 | 命令是**实验性** `x-` 前缀（接口可能变） |

---

## 2. 前置条件

- **内核**：Linux 6.x，匿名内存 userfaultfd 写保护（`UFFD_FEATURE_PAGEFAULT_FLAG_WP`）。
- **构建**：`liburing`（QEMU `--enable-linux-io-uring`）。
- **运行**：`CAP_SYS_ADMIN`（读 `/proc/self/pagemap` + `/sys/kernel/mm/page_idle/bitmap`）——
  以 root 跑 QEMU 即可。
- **后端**：一个 raw 块设备（如 NVMe 分区）或预分配的 raw 文件，建议容量 ≥ 预计换出量
  （要能 100% 换出就 = 客户机内存大小）。O_DIRECT 打开，故文件需在支持 O_DIRECT 的文件系统上。

---

## 3. 构建带 memtier 的 QEMU

memtier 在 `host_os==linux && linux_io_uring.found()` 时编进 `qemu-system-x86_64`
（否则编错误桩 `memtier-stubs.c`，QMP 命令仍可链接但返回不支持）。

**快速迭代构建**（只出 x86_64 系统二进制，跳过 docs/tools/tests）：

```bash
cd qemu
./configure --target-list=x86_64-softmmu --enable-linux-io-uring --enable-kvm \
            --disable-docs --disable-tools --disable-werror
ninja -C build qemu-system-x86_64        # 改 memtier/*.c 后增量重链 ~30s
# 产物：build/qemu-system-x86_64
```

**打包到系统**（xvirt 用）：`build-deb/build_deb.sh ../qemu memory-tiering` 出 deb，`dpkg -i`
安装 `qemu-system-*` 等。

验证二进制含 memtier：`strings qemu-system-x86_64 | grep memtier`（应有
`memtier_worker` / `memtier_evict_run` / `MEMTIER_BACKING_FULL` 等）。

---

## 4. 启动兼容 memtier 的虚机

要点：**匿名 RAM**（普通 `-m`，不要 `memory-backend-file`/`memfd`/`-mem-path`/hugepages），
KVM 加速。memtier 会对目标 region 自动 `MADV_NOHUGEPAGE`。

```bash
qemu-system-x86_64 \
  -machine q35,accel=kvm -cpu host -m 4096 -smp 2 \
  -drive file=guest.qcow2,if=virtio,format=qcow2 \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
  -qmp unix:/tmp/qmp.sock,server,nowait \
  -display none -serial mon:stdio
```

> ⚠️ 不要给这台 VM 接 **vhost-user-blk（petrel）** 之类需 `share=on` memfd 的设备 —— 那会
> 把 guest RAM 变成共享 memfd，memtier 无法分层。两者互斥。

---

## 5. QMP 接口参考

连 QMP（unix socket）后先 `qmp_capabilities`，再发命令。最小 Python 客户端：

```python
import socket, json
def qmp(execute, **args):
    s = socket.socket(socket.AF_UNIX); s.connect("/tmp/qmp.sock")
    f = s.makefile("rwb", buffering=0); f.readline()            # greeting
    f.write(b'{"execute":"qmp_capabilities"}\n'); f.readline()
    m = {"execute": execute};  m.update({"arguments": args} if args else {})
    f.write((json.dumps(m) + "\n").encode())
    while True:
        r = json.loads(f.readline())
        if "event" not in r: return r
```

### 5.1 `x-memtier-enable` —— 开启分层

```json
{ "execute": "x-memtier-enable",
  "arguments": {
    "path": "/dev/nvme1n1p3",        // 必填：后端 raw 设备/文件（O_DIRECT）
    "policy": "balanced",            // 可选：conservative|balanced|aggressive（默认 balanced）
    "id": "pc.ram",                  // 可选：目标 region；默认主 guest RAM 块
    "tunables": { "min-resident-mb": 1024 }   // 可选：逐项覆盖预设（见 §6）
  } }
```

### 5.2 `x-memtier-query` —— 查状态/统计

```json
{ "execute": "x-memtier-query" }       // 省略 id 返回所有 region
// → { "return": [ { ...MemTierInfo... } ] }    见 §7
```

### 5.3 `x-memtier-set-policy` —— 运行时改策略/旋钮

```json
{ "execute": "x-memtier-set-policy",
  "arguments": { "tunables": { "psi-trigger": 0, "min-resident-mb": 512,
                               "max-evict-mbps": 4000 } } }
```

### 5.4 `x-memtier-disable` —— 停止分层

```json
{ "execute": "x-memtier-disable" }
```

先把**所有换出页强制拉回 RAM**，再拆除 uffd/io_uring/后端（耗时随已换出量增长）。
在线迁移、内存快照前应先 disable 收敛 RAM。

### 5.5 事件

- `MEMTIER_BACKING_FULL` —— 后端设备空间耗尽，换出暂停（无正确性影响，换入仍正常）。

---

## 6. 策略预设与旋钮

### 预设

| 旋钮 | conservative | balanced | aggressive |
|---|---|---|---|
| cold-threshold | 0 | 2 | 8 |
| cold-runs | 8 | 5 | 2 |
| psi-trigger (%) | 20 | 10 | 2 |
| max-evict-mbps | 200 | 500 | 1000 |
| 常驻下限 | 75% | 50% | 25% |
| cooldown-ms | 2000 | 1000 | 500 |

### `tunables`（逐项覆盖预设）

| 字段 | 含义 |
|---|---|
| `cold-threshold` | DAMON/idle nr_accesses 低于此算冷（越大越激进） |
| `cold-runs` | 连续判冷多少次才换出（**准确性关键**：越大越不误换偶发热页，越小越快） |
| `psi-trigger` | 主机 PSI memory `some avg10` 超过此（%）才换出。**设 0 = 无视主机压力立刻换出到下限** |
| `max-evict-mbps` | 换出速率上限（滚动窗口限速） |
| `min-resident-mb` | 常驻硬下限（换出永不把 region 压到此以下） |
| `cooldown-ms` | 刚换回的页在此窗口内不再被换出（防抖动） |
| `target-resident-mb` | **主动换回**上限：常驻低于它且主机空闲时，按近度预取最近换出的页回 RAM（消缺页尾延迟）。默认 = `min-resident-mb`（即主动换回**关**）。与 min 夹出工作带：换出不低于 min、主动换回不高于 target |
| `swapin-watermark` | 主动换回触发的 PSI some avg10 上限 %；引擎钳制其 **< `psi-trigger`**，防换回↔换出振荡 |
| `max-swapin-mbps` | 主动换回限速 MB/s |

### 换出触发条件（核心语义）

worker 每拍换出的前提：**`psi_some_avg10 >= psi-trigger` 且 `resident > min-resident`**。
- 想「跟随主机压力自动换」：用预设的 psi-trigger（如 balanced=10）。
- 想「立刻把这台压到下限」：`psi-trigger=0` + 设好 `min-resident-mb`。
- 想「暂停换出」：`psi-trigger=100`（PSI 几乎不会到 100）。
- **热度准确性**：保留 `cold-runs>=2`（预设 aggressive=2）；实测 `cold-runs=1` 会把偶发热页
  误换出再换回，抖动约高 16 倍。

---

## 7. `MemTierInfo` 统计字段

| 字段 | 含义 |
|---|---|
| `id` / `enabled` / `path` / `policy` / `tunables` | region 标识 / 是否启用 / 后端 / 生效策略 / 生效旋钮 |
| `total-bytes` | region 大小 |
| `resident-bytes` | 当前常驻宿主 RAM 的字节 |
| `swapped-bytes` | 当前在后端设备上的字节 |
| `backing-total/used/largest-free-bytes` | 后端容量 / 已用 / 最大连续空闲 |
| `evicted-pages` | 累计成功换出页数 |
| `evict-aborts` | 累计回滚（换出途中被写脏） |
| `swapin-faults` | 累计 MISSING 缺页（从后端换回，guest 触发） |
| `swapin-errors` | 累计致命读回失败 |
| `active-swapins` | 累计**主动换回**预取页数（控制器触发，非 guest 缺页） |
| `evict-mbps` | 近期换出吞吐 MB/s |
| `psi-some-avg10` | 上次采样的主机 PSI memory some avg10 |

观察要点：`resident` 降 + `swapped` 升 = 在压缩；`swapin-faults` 飙升而 `swapped` 已稳 =
抖动（调大 cold-runs / cooldown）；`evict-aborts` 高 = 换出途中写竞争激烈（正常少量）。

---

## 8. 内部机制（QEMU 的能力如何实现）

### 8.1 页状态机（对应 uffd 缺页分类）

`RESIDENT`（在 RAM 可写）→ `EVICTING`（写保护，写触发 WP 缺页）/`EVICTING_DIRTIED`（途中被写→回滚）
→ `SWAPPED`（已 MADV_DONTNEED，访问触发 MISSING 缺页）→ `FAULTING_IN`（读回在途，并发者串行）。

### 8.2 实时热度获取

`page_idle`：标记页 idle → 等一个 **100ms 墙钟窗口** → 回读，仍 idle 即窗口内没被 EPT 访问
过 = 真冷（能抓读热）。**关键**：100ms 窗口对窗口内所有标记页共享，故一次标记**一整批 cell**、
只睡一个窗口、再回读 → 整段**近乎同一时刻的连贯热度快照**。PFN 用 pagemap 批量读，page_idle
位图按 PFN 词跨度做**一次性批量读-改-写**（整批几个 syscall，而非每页 2 个）。

> 这是本 fork 相对早期实现的关键改进：早期逐 cell 扫描（每 2MB 一个独立窗口、round-robin）
> 冷页探测仅 ~6 MiB/s，多 GB region 几分钟才扫一遍；批量化后整 region 秒级出快照，换出
> 由 ~6 MiB/s 提升到流式（>50% 下刷从「4 分钟到不了」变为 ~21 秒）。

### 8.3 换出 / 换入

- 换出（worker 线程，每 region 一个）：按到下限的差额自适应批量扫描 → 把冷页切成最大连续段
  → 每段一次 io_uring O_DIRECT 大块顺序写 → 确认未被写脏后 MADV_DONTNEED 释放宿主页。
  受 `min-resident` 下限 + `cold-runs` 去抖 + `max-evict-mbps` 限速约束。
- 换入（fault 线程）：uffd poll → MISSING 缺页 → io_uring 读回 → UFFDIO_COPY 填回 → 记
  `fault_tick`（cooldown 防抖）。

---

## 9. 调参配方

```jsonc
// A) 激进超分：尽快把这台压到 1GB 常驻（无视主机压力）
{ "execute":"x-memtier-enable", "arguments":{ "path":"/dev/nvme1n1p3",
  "policy":"aggressive", "tunables":{ "psi-trigger":0, "min-resident-mb":1024,
                                      "max-evict-mbps":4000 } } }

// B) 跟随主机压力温和分层（生产常态）
{ "execute":"x-memtier-enable", "arguments":{ "path":"/backing/vm1.raw",
  "policy":"balanced" } }                 // psi-trigger=10，压力高才换

// C) 暂停换出（保留已换出页）
{ "execute":"x-memtier-set-policy", "arguments":{ "tunables":{ "psi-trigger":100 } } }
```

---

## 10. 完整示例（裸 QEMU 超分实测）

```bash
# 1) 后端
fallocate -l 6G /backing/vm1.raw
# 2) 起 VM（4G 匿名 RAM）
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 -smp 2 \
  -drive file=guest.qcow2,if=virtio -qmp unix:/tmp/qmp.sock,server,nowait -display none &
# 3) 客户机内灌内存（填 3G、保 256M 热）后空闲
# 4) 强制换出到 512M 下限
python3 - <<'PY'
# 见 §5 的 qmp(); 然后：
qmp("x-memtier-enable", path="/backing/vm1.raw", policy="aggressive",
    tunables={"psi-trigger":0,"min-resident-mb":512,"max-evict-mbps":4000})
# 轮询 x-memtier-query 观察 resident↓ / swapped↑
PY
```

实测曲线（本 fork）：换出从 ~6s 起流式推进，**~21s 下刷 >50%**，终态常驻 ~730MiB /
换出 ~3368MiB（~82% 下刷）；400MB 换出后重读 sha256 一致；evict-mbps 正常上报。

---

## 11. 运维与限制

- **迁移/快照前 `x-memtier-disable`** 收敛 RAM（否则迁移看不到完整内存）。
- 后端满 → `MEMTIER_BACKING_FULL` 事件 + 换出暂停（无正确性影响）；扩容/换更大后端后恢复。
- 仅 4K 页（强制 NOHUGEPAGE）；仅匿名 RAM；单主机；`x-` 实验性接口。
- 日志：QEMU 进程 stderr/日志里 `memtier:` 前缀的 warn（如 page_idle 不可用 = 缺
  CAP_SYS_ADMIN）。

---

## 12. 与 xvirt 的关系

xvirt 把上面这套 QMP 自动编排起来（用户无需手敲 QMP）：

- **lictor** 持 QMP，VM 启动时按 sidecar 调 `x-memtier-enable`，并暴露
  `memtier_query/set_policy/disable` action。
- **hyper-agent 主机级总控**每 5s 读真实主机内存压力 + 各 VM 统计，按优先级在主机预算内对每台
  下 `x-memtier-set-policy`（谁换出、换到多深）。
- **ferrovirt + 前端**做创建开关、策略/优先级、统计展示。

完整架构、控制面、落库、前端用法、真实压力下自动压缩的整合验证见
[`memory-tiering.md`](./memory-tiering.md)。
