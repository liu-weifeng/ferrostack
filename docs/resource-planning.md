# xvirt 资源规划：CPU / 内存隔离与预留方案（v1 草案）

> 为 xvirt 设计主机级 CPU、内存的隔离与预留能力：把操作系统（内核 + 系统服务）、
> 虚拟化服务、SPDK、虚拟机划分为可视、可拖拽调整的资源池；覆盖 NUMA 拓扑、
> 内存大页与内存分层（memtier）的展示与管控。
>
> 2026-09-01 · 生产内核 6.7（cgroup v2 / cpuset isolated 分区可用）
> · 涉及 ferrovirt / hyper-agent / lictor / xvirt-client
> · 可视化版本（含页面 mockup）：https://claude.ai/code/artifact/4fcb9c64-0910-462c-9601-5bae73f9fff6
> · 可交互页面原型（独立 demo，不依赖 xvirt-client）：`demo/resource-plan/index.html`（见同目录 README）
> · 依据：CLAUDE.md 组件速查、`memory-tiering.md`、`qemu-memtier-usage.md`、
>   `vm-direct-metrics-design.md`、`monitoring/README.md`

---

## §0 摘要 · 选型决策

一句话：**以 cgroup v2 cpuset 分区为唯一基础机制做运行时可调的 CPU/内存池化，
每台 VM 落入独立 systemd scope；隔离、预留、大页、NUMA、SPDK、memtier 统一收进
一份 per-host「资源规划（resource plan）」，由 hyper-agent 幂等对账落地，前端在
主机详情新增「资源规划」tab 做图形化展示与拖拽编辑。**

| 问题 | 决策 | 理由 |
|---|---|---|
| CPU 隔离用什么 | **cgroup v2 cpuset 分区**（`cpuset.cpus.partition`），不以 `isolcpus` 为基础机制 | isolcpus 需重启、静态、全有全无；内核 6.7 的 `isolated` 分区在运行时提供等效隔离（脱离调度域、无负载均衡），正好支撑「拖拽即生效」 |
| SPDK 轮询核 | cpuset `isolated` 分区 + IRQ 驱离；可选 `nohz_full` 启动参数档位（标「需重启」） | 轮询线程要独占核、免时钟/中断干扰；isolated 分区覆盖 90% 需求，极致低抖动才需要 boot profile |
| per-VM 资源边界 | **systemd transient scope**：`xvirt-vm-<uuid>.scope` 挂 `xvirt-machine.slice` | 目前全部 QEMU 挤在 xvirt-lictor.service 的 cgroup 里（KillMode=process 的副作用），无法逐 VM 计量/限额；scope 顺带让 QEMU 生命周期独立于 lictor 重启 |
| vCPU 绑定 | scope 级 `cpuset` 定池 + QMP `query-cpus-fast` 拿线程 id 后 `sched_setaffinity` 做 1:1 独占绑定 | 池归属用 cgroup（可整体挪），精确 pin 用线程亲和性；两层正交 |
| 内存预留 | `memory.min` 保护 OS / 虚拟化服务 /（P4）VM 热集；`xvirt-machine.slice` 用 `memory.high` 做 VM 池软预算 | min 挡住回收保住预留；VM 池不用 `memory.max` 硬顶——顶穿会触发 OOM 杀 QEMU，软限 + memtier 下刷更安全 |
| 大页 | per-NUMA sysfs 运行时分配（2M 常态可行）；1G 走启动参数/开机早期预留，UI 标「需重启生效」；`hugetlb` cgroup 圈 SPDK 与 VM 的用量 | 1G 页在内存碎片化后运行时分配基本必失败 |
| VM NUMA 落位 | `memory-backend-*,host-nodes=N,policy=bind` + `-numa node,memdev` 建 vNUMA；scope 的 `cpuset.mems` 兜底 | QEMU 原生机制；单节点 VM 自动放置，跨节点 VM 显式声明 vNUMA |
| memtier 关系 | 仅「匿名内存」后端模式的 VM 可开启（现状约束）；主机总控的水位线/预算改从 resource plan 读取并在 UI 暴露 | memtier 强制 4K 匿名页，与大页、vhost-user（petrel/SPDK）共享内存互斥；总控现在的阈值是环境变量常量，应归入规划统一管理 |
| 前端形态 | 主机详情新 tab「资源规划」：查看模式实时叠加用量，编辑模式草稿 → 校验 → 异步任务应用 | 沿用既有 TaskManager / WS / i18n 约定 |
| 数据链路 | 配置走新 REST（plan API + 本地缓存对账）；用量走既有 hyper-agent exporter :9101 → VictoriaMetrics → ferrovirt 指标代理 | 监控链路已建成，补 collector 即可；顺带落地 `vm-direct-metrics-design.md` 里搁置的 per-VM cgroup 直采 |

---

## §1 目标与非目标

### 目标

- **隔离**：OS（内核 + 系统服务）、虚拟化服务（ferrovirt/hyper-agent/lictor/petrel/mariadb/nginx 等）、SPDK、虚拟机四类负载互不侵占 CPU；SPDK 核完全独占。
- **预留**：OS 与虚拟化服务的内存有硬保障（不被 VM 压力回收挤死）；大页按 NUMA 节点、按页大小（2M/1G）定量预留；SPDK 的核与大页单独圈占。
- **可视**：主机 NUMA 拓扑、每个逻辑 CPU 的池归属与实时利用率、每 NUMA 节点内存的分段构成（内核/预留/大页/VM/空闲）、每台 VM 的占用与放置、memtier 常驻/换出/后端盘用量，全部图形化。
- **可拖拽**：核心 → 池、VM → 核（独占绑定）、内存段边界手柄，都可拖拽调整；所有操作有非拖拽等价入口（右键菜单/输入框）。
- **可对账**：规划持久化于 DB + 主机本地缓存，重启后由 hyper-agent 先于 VM 启动幂等重放。

### 非目标（本期不做）

- 集群级调度联动（放置时感知各 host 池余量）——plan 是 per-host 对象，center 侧先做逐主机入口，聚合视图后续。
- SPDK 本体的部署编排（bdev/vhost target 配置）——本期只做**资源预留与展示**，把核和大页圈出来。
- CPU 时间片精细 QoS（cpu.max 配额、实时调度类）——先池化，配额留作后续旋钮。

---

## §2 资源模型

### 2.1 CPU：五类池归属

主机的每个逻辑 CPU（含超线程）在任一时刻属于且仅属于一个池。池即 cgroup 顶层
slice 的 `cpuset`（§3.2）：

| 池 | 承载 | 隔离强度 | 默认规模建议 |
|---|---|---|---|
| **system**（OS） | 内核杂务、sshd、systemd、journald 等 `system.slice`/`user.slice`/`init.scope` | 软边界（cpuset 圈定，不隔离调度域） | ≥2C（≥64C 主机取 4C） |
| **virt**（虚拟化服务） | `xvirt.slice`：ferrovirt / hyper-agent / lictor / conciliumd / petrel 守护 / mariadb / nginx / websockify / OVS 用户态 | 软边界 | 2–4C，跑 petrel OSD 的节点加大 |
| **spdk** | `spdk.slice`（reactor 轮询线程） | **isolated 分区**：脱离调度域 + IRQ 驱离 | 按 vhost 目标带宽定，1–4C 起步；未部署 SPDK 时池可为空 |
| **vm-shared**（VM 共享） | `xvirt-machine.slice` 下未独占的 VM scope（vCPU 超分调度于此） | 软边界 + per-VM `cpu.weight` | 其余全部 |
| **vm-dedicated**（VM 独占） | 被某台 VM 1:1 绑定的核（vCPU 线程 pin），从 vm-shared 中划出 | 逻辑独占（同池内不共享）；可选升级为 isolated 分区 | 按需 |

> **超线程配对约束**：分池以**物理核**为最小拖拽单元（一对 HT 同进同出）。跨池
> 共享同一物理核的两个 HT 会让「隔离」名存实亡；SPDK/独占池尤其如此。UI 上 HT
> 成对渲染，整核拖动。

### 2.2 内存：per-NUMA 分段模型

每个 NUMA 节点的物理内存被划分为若干段，段即展示与预留的口径
（求和 = 节点总量，残差归入「内核/其它」）：

| 段 | 含义 | 数据来源 | 控制手段 |
|---|---|---|---|
| 内核 / 其它 | Slab、页表、vmalloc、per-CPU、内核线程等不可归户部分 | `/proc/meminfo` 与 node meminfo 差额 | 不可配置，仅展示 |
| OS 预留 | system.slice + user.slice 的保障额 | 规划值 vs `memory.current` | `memory.min` |
| 虚拟化服务预留 | xvirt.slice 保障额（mariadb 是大头） | 同上 | `memory.min` |
| 大页 2M / 1G | 预留的 HugeTLB 池（含 SPDK 圈占部分，单列） | `node*/hugepages/*/nr_hugepages,free_hugepages` | sysfs per-node 写入；hugetlb cgroup 圈用量 |
| VM 池 | machine.slice 预算：已用（各 VM `memory.current` 之和）+ 余量 | cgroup + 规划值 | `memory.high`（软） |
| memtier 视图 | VM 池内叠加显示：Σ常驻 vs Σ已换出（换出部分**不占**本节点内存，是「借出去」的量）+ 后端盘水位 | lictor `memtier_query` 聚合 | 总控水位线（§3.7） |
| 空闲 / 页缓存 | MemAvailable 口径其余部分 | `/proc/meminfo` | — |

### 2.3 VM 内存后端三模式（关键兼容矩阵）

这是整个方案里最硬的约束来源：memtier 只吃匿名 4K 内存，vhost-user（petrel 现在、
SPDK 将来）要共享内存，大页要 hugetlbfs/memfd-hugetlb。三者不可兼得，必须做成
VM 级显式属性 `mem_backend`，建 VM 时选定、互斥项在 UI 直接置灰（ferrovirt 校验 400）：

| 能力 | A · 匿名（默认，`-m`） | B · 共享 memfd（`share=on`） | C · 大页（memfd `hugetlb=on` + `share=on`，2M/1G） |
|---|---|---|---|
| 内存分层 memtier | ✓ 唯一支持 | ✗ | ✗（memtier 强制 MADV_NOHUGEPAGE） |
| petrel vhost-user-blk | ✗ | ✓ 现状 | ✓ |
| SPDK vhost-user | ✗ | △ 可用但不建议 | ✓ 推荐（DMA 走大页） |
| EPT/TLB 大页收益 | ✗（THP 视情况） | ✗ | ✓ |
| 内存超分能力 | ✓（memtier 冷页下刷） | 仅 balloon | ✗（prealloc 全预留） |
| 在线迁移 | ✓（先 disable memtier 收敛） | ✓（petrel 已验证） | ✓（两端需同额大页） |
| 大页池占用 | — | — | 启动即全额扣减 |

> **规划耦合**：一台模式 C 的 VM 开机 = 从其绑定 NUMA 节点的大页池按整机内存扣减；
> 页面在编辑大页预留时必须显示「已被 VM 占用 / SPDK 圈占 / 空闲」三段，收缩到低于
> 占用量直接拒绝。

---

## §3 后台实现

### 3.1 选型：cgroup v2 分区，而不是 isolcpus

| | isolcpus / nohz_full（启动参数） | cgroup v2 cpuset 分区（本方案） |
|---|---|---|
| 调整 | 改 grub + 重启 | 运行时写 cgroupfs，秒级生效 → 支撑拖拽 |
| 隔离强度 | 彻底（连 unbound kthread 都躲开，nohz_full 免 tick） | `isolated` 分区 = 脱离调度域、无负载均衡（6.1+；6.7 完整）；per-cpu kthread 仍在，tick 仍在 |
| 粒度 | 整机一份静态清单 | 逐 slice/scope，层级化，可与 weight/quota 组合 |
| 与 systemd | 无关 | `AllowedCPUs=`/`AllowedMemoryNodes=` 原生映射 cpuset |

**结论**：基础机制只用 cgroup v2。SPDK 池用 `isolated` 分区；对极致低抖动需求提供
可选「低抖动启动档」（`nohz_full=<spdk cpus> rcu_nocbs=… irqaffinity=…`，由
xvirt-init 写 grub drop-in，UI 标「需重启」）。

### 3.2 cgroup / systemd 层级

```
/sys/fs/cgroup/
├── init.scope                AllowedCPUs=<system 池>
├── system.slice              AllowedCPUs=<system 池>   memory.min=<OS 预留>
├── user.slice                AllowedCPUs=<system 池>
├── xvirt.slice               AllowedCPUs=<virt 池>     memory.min=<虚拟化服务预留>
│   ├── xvirt-ferrovirt.service / xvirt-hyper-agent.service / xvirt-lictor.service
│   ├── conciliumd.service / mariadb.service / nginx.service   ← 用 drop-in Slice=xvirt.slice 归入
│   └── xvirt-petrel-*.scope        ← petrel 守护（hyper-agent 托管时以 scope 拉起）
├── spdk.slice                AllowedCPUs=<spdk 池>
│   │                         cpuset.cpus.partition=isolated   ← agent 直写 cgroupfs
│   └── spdk-vhost.service（后续）
└── xvirt-machine.slice       AllowedCPUs=<vm-shared ∪ vm-dedicated>
    │                         memory.high=<VM 池预算>
    ├── xvirt-vm-<uuidA>.scope    cpuset=共享池        cpu.weight=100
    └── xvirt-vm-<uuidB>.scope    cpuset=独占核集合    cpu.weight=10000  (memory.min 可选)
```

- **isolated 分区的前提**：内核要求分区的 CPU 不与兄弟 cgroup 重叠——所以 agent
  必须给**所有**顶层 slice（含 `init.scope`/`user.slice`）显式设 `AllowedCPUs`，
  这正是「全量池化」天然满足的条件。
- **归并系统服务**：mariadb/nginx 等属于虚拟化平台的服务用 drop-in
  `Slice=xvirt.slice` 归入 virt 池——「虚拟化系统占用」在页面上才是一条完整、
  诚实的曲线。
- **下发方式**：`systemctl set-property`（运行时 + 自动落 drop-in 持久化）设
  AllowedCPUs / memory.\*；`cpuset.cpus.partition`、hugetlb 上限等 systemd 不覆盖
  的属性由 agent 直写 cgroupfs，并由开机对账单元重放（§3.9）。

### 3.3 per-VM scope：QEMU 从 lictor 的 cgroup 里搬出来

**现状**：lictor 拉起 QEMU（subprocess），QEMU 全部落在 `xvirt-lictor.service` 的
cgroup；服务重启靠 `KillMode=process` 保命。后果：无法逐 VM 计量
（memory.current/cpu.stat/PSI 全是糊的）、无法逐 VM 限额、也无法把 VM 整体挪池。

**改造**（lictor `launch_vm`）：

1. QEMU 以 `-S`（暂停）拉起；
2. 经 D-Bus `StartTransientUnit("xvirt-vm-<uuid>.scope", pids=[qemu_pid],
   Slice="xvirt-machine.slice", Delegate=true, CPUWeight/MemoryMin=…)` 把进程移入
   独立 scope（libvirt 同款做法；不用 `systemd-run --scope` 包一层，避免改变
   父子关系）；
3. 按放置配置设 scope 的 `AllowedCPUs`/`AllowedMemoryNodes`；独占 VM 再经 QMP
   `query-cpus-fast` 取 vCPU 线程 id，逐线程 `sched_setaffinity` 1:1 绑核；
4. QMP `cont` 放行。

- **顺带收益**：QEMU 不再住 service cgroup，lictor/hyper-agent 重启天然不波及 VM，
  `KillMode=process` 从「保命 hack」降级为双保险。
- **存量迁移**：升级后对账把运行中 QEMU 的 pid 写入新建 scope 即可（cgroup v2 允许
  迁移）；注意已计费的匿名页仍挂在旧 cgroup，`memory.current` 要到页面重新 fault
  后才逐步归位——统计初期偏低是预期行为，标注即可，或选择「下次 VM 重启时生效」。
- **emulator/iothread 分离**（P2 可选）：scope 开 `Delegate=yes` 后建 threaded 子组
  `vcpu/` 与 `emulator/`，独占 VM 的模拟器线程可留在共享池，进一步降低独占核抖动。

### 3.4 内存预留与 VM 池预算

```
# 预算恒等式（per host，规划器强制）：
MemTotal = 内核残差(实测)
         + OS 预留(memory.min@system.slice)
         + 虚拟化服务预留(memory.min@xvirt.slice)
         + Σ 大页池(2M+1G，含 SPDK 圈占)
         + VM 池预算(memory.high@xvirt-machine.slice)
         + 安全余量(≥ 4% MemTotal)
```

- `memory.min` 是「预留」的落地：VM 压力触发全局回收时，受保护 slice 的页不被
  扫描——OS 与虚拟化服务由此拿到硬保障。前提：根上 `subtree_control` 启用 memory
  控制器（systemd 默认满足，agent 对账时校验）。
- VM 池用 `memory.high` 不用 `memory.max`：顶到 high 触发直接回收 + 节流，给
  memtier 总控留出下刷窗口；hard max 顶穿会 OOM-kill QEMU，代价不对称。
- VM 池预算校验规则：`Σ(运行中非 memtier VM 内存 × (1+overhead)) ≤ 预算`；
  memtier VM 按 `min-resident` 折算下限、按全额折算上限，两条线都在 UI 显示
  （「保证线 / 超分线」）。QEMU per-VM overhead 取 `3% + 300MB` 经验值，进规划器常量。

### 3.5 大页管理

- **分配**：写 `/sys/devices/system/node/node<N>/hugepages/hugepages-{2048kB,1048576kB}/nr_hugepages`，
  按节点精确投放。2M 运行时通常可满足（失败时 agent 先 `compact_memory` 重试一次）；
  **1G 运行时分配在碎片化后基本必失败**——策略：规划里记目标值，agent 尝试运行时
  分配，不足额则该项标「需重启」，由开机早期单元（`xvirt-resource-plan.service`，
  §3.9）在 boot 早期补齐，兜底方案才是 grub `hugepagesz=1G hugepages=N`。
- **圈占**：SPDK 与 VM 共用 HugeTLB 池但通过 `hugetlb.2MB.max` / `hugetlb.1GB.max`
  在 `spdk.slice` 与 `xvirt-machine.slice` 上互相设限，防一方吃光另一方的预留。
- **消费**：模式 C VM 用 `memory-backend-memfd,hugetlb=on,hugetlbsize=2M|1G,share=on,prealloc=on`
  （免 hugetlbfs 挂载点管理）；SPDK 用其自带 `--huge-dir/--mem-size`。

### 3.6 NUMA 落位

- **拓扑采集**：hyper-agent 新增 `scan_cpu_topology`（并入 `scan_device` 结果树）：
  `/sys/devices/system/cpu/cpu*/topology/*`（socket/core/thread 配对）+
  `/sys/devices/system/node/node*/`（cpulist、meminfo、hugepages、`distance` 矩阵）。
  落 ferrovirt 新表（§4.1），供拓扑图渲染。
- **单节点 VM（默认，自动）**：放置器选「绑定池核数与空闲内存都放得下」的最空节点；
  QEMU 加 `-object memory-backend-ram,host-nodes=N,policy=bind`（模式 B/C 用对应
  backend）+ scope `AllowedMemoryNodes=N`，vCPU 绑该节点池内核。
- **跨节点 VM（显式 vNUMA）**：`vm_config.numa` 声明 vnode 列表（每 vnode：vcpus、
  内存、host-node），拼 `-numa node,nodeid=i,cpus=…,memdev=mi`，guest 内看到与宿主
  对齐的拓扑。页面 VM 详情画 vNUMA→host node 的映射关系。
- **降级**：单 NUMA 主机（现有 lab 的 E5-26xx v3 单路）一切照常，页面只渲染一个节点组。

### 3.7 memtier 融合

- **准入**：`mem_backend=anon` 才可开 memtier（既有 petrel 互斥校验推广为三模式
  矩阵校验）。
- **预算联动（P4）**：总控 `support/memtier/controller.py` 现在用
  `XVIRT_MEMTIER_FREE_LOW_PCT/FREE_HIGH_PCT/PSI_HIGH` 环境变量常量算主机压力。改为
  从本地 plan 缓存读：压力 = `max(PSI 项, 预算项)`，预算项 = VM 池 `memory.current`
  相对 `memory.high` 的挤占度。下刷/回血两条水位线成为 plan 字段，UI 可调
  （memtier 面板上的两个刻度，§5.3）。
- **热集保护（P4 可选）**：总控把每台 memtier VM 的 scope `memory.min` 同步为
  `min-resident-mb + overhead`，防 VM 池整体回收把热集也挤掉。
- **展示数据**：全部来自既有 `x-memtier-query`（经 lictor）：
  `resident-bytes / swapped-bytes / backing-total|used / evict-mbps / swapin-faults /
  psi-some-avg10`，聚合进 exporter（§6）。

### 3.8 SPDK 预留

- **本期交付**：plan 中的 `spdk` 块（核列表 + per-node 大页额度）→ agent 建
  `spdk.slice`、写 isolated 分区、按额度校验大页池、写 irqbalance banned 清单
  （`IRQBALANCE_BANNED_CPULIST` drop-in + 现存 IRQ 的 `smp_affinity_list` 改写驱离）。
- **后续接线**：SPDK vhost target 以 `spdk-vhost.service`（`Slice=spdk.slice`）部署，
  reactor mask = 池核；被服务 VM 走模式 C；reactor 是 100% 轮询，利用率图对 spdk 池
  改显示「占用/预留」而非 busy%（接 SPDK 自身 stats 后再显示有效利用率）。

**IRQ 与内核线程的诚实口径**：

- SPDK/独占池核通过 irqbalance banned + 显式 affinity 驱离可迁移 IRQ；per-CPU
  kthread（ksoftirqd 等）与 timer tick 无法用 cgroup 移走——这是 isolated 分区与
  isolcpus 的真实差距，UI 的池 tooltip 里注明「残余内核活动 < ~1%」。
- vhost-net 等内核线程不入任何 cgroup，其 CPU 计入「内核/其它」段；后续可按 VM 把
  vhost 线程 affinity 跟随其池（增强项）。

### 3.9 开机对账与应用顺序

```
xvirt-resource-plan.service (oneshot, Before=xvirt-lictor.service)
  读 /etc/xvirt/resource-plan.json（DB 之外的本地缓存，PUT plan 时同步写）
  ① 1G/2M 大页早期投放（内存干净，成功率最高）
  ② 顶层 slice AllowedCPUs / AllowedMemoryNodes / memory.min|high
  ③ spdk.slice 建立 + partition=isolated + hugetlb 上限 + IRQ 驱离
  ④ 标记就绪 → lictor 启动 → 逐 VM scope 放置（launch 时逐台施加）
运行时 apply_resource_plan（异步任务）= 同一套幂等 reconcile，逐项返回
  applied / pending-reboot / failed(原因)
```

---

## §4 数据模型与 API

### 4.1 DB（ferrovirt，MariaDB `mx`）

| 表 / 列 | 内容 |
|---|---|
| `host_resource_plan`（新表） | `host_id`(uniq)、`version`(乐观锁)、`plan`(JSON，见下)、`pending_reboot`(JSON 数组)、`updated_at / updated_by`(JWT sub) |
| `host_cpu_topology`（新表，扫描落库） | 逐逻辑 CPU 一行：`host_id, cpu, core, socket, node, online`；节点级信息（内存、distance）入既有 `device` 树或 JSON 列 |
| `vm_config` 增列 | `cpu_placement`(shared\|dedicated)、`cpu_pins`(JSON "vcpu→pcpu")、`numa`(JSON vnode 列表)、`mem_backend`(anon\|memfd\|hugepage-2m\|hugepage-1g)、`mem_reserved_mb`(memory.min，可空)。均 `ALTER TABLE … ADD COLUMN IF NOT EXISTS`（沿用 ensure_schema 约定） |

```jsonc
// host_resource_plan.plan —— 单一 JSON 文档（池成员表述为 cpu 列表，物理核对齐由校验器保证）
{ "schema": 1,
  "pools": { "system": [0,1,32,33], "virt": [2,3,34,35],
             "spdk": { "cpus": [4,5,36,37], "isolated": true, "boot_profile": false },
             "vm":   { "cpus": [6, "...", 63], "dedicated": { "<vm-uuid>": [12,13,44,45] } } },
  "memory": { "os_reserve_mb": 4096, "virt_reserve_mb": 8192,
              "vm_budget_mb": 214016, "headroom_pct": 4 },
  "hugepages": { "node0": { "2m": 8192, "1g": 16 }, "node1": { "2m": 8192, "1g": 16 },
                 "spdk_quota": { "2m": 2048, "1g": 4 } },
  "memtier": { "evict_watermark_pct": 10, "swapback_watermark_pct": 50, "psi_high": 10 } }
```

### 4.2 REST（ferrovirt，均挂 JWT）

| 端点 | 语义 |
|---|---|
| `GET /api/vcenter/host/{uuid}/resource-topology` | NUMA/socket/core/thread 拓扑 + 每 CPU 当前池归属 + 每节点内存/大页现势（拓扑图数据源，变更少，进页拉一次） |
| `GET /api/vcenter/host/{uuid}/resource-plan` | plan + version + pending_reboot + 各项实际落地状态（plan vs 实况 diff，供「未生效」标记） |
| `PUT /api/vcenter/host/{uuid}/resource-plan` | 带 `version` 乐观锁（409 = 需重载合并）；**异步任务**（kind=`applyResourcePlan`，走 TaskManager 约定），agent 幂等 reconcile，task.message 汇总逐项结果 |
| `POST …/resource-plan?action=validate` | 服务端 dry-run：完整校验（§4.4）+ 生效性标注，不落地。编辑器「校验」按钮与提交前置检查共用 |
| `PATCH /api/vcenter/vm/{vm}`（扩展） | `UpdateVmRequest` 增 `placement` 稀疏段（cpu_placement/cpu_pins/numa/mem_backend/mem_reserved_mb）；运行中仅 pins/weight 可热调，其余标「下次启动生效」 |
| `GET /api/vcenter/host/{uuid}/memtier`（新聚合） | 各 memtier VM 的 MemTierInfo + 总控状态（当前压力、水位线、每 VM 当前档位）——面板一次拉全 |

### 4.3 agent actions（ZMQ）

| 组件 | action | 职责 |
|---|---|---|
| hyper-agent | `scan_cpu_topology` | 拓扑 + 每 CPU 实际 cgroup 归属 + 大页现势（挂进 hardware-scan 综合扫描） |
| hyper-agent | `apply_resource_plan` / `validate_resource_plan` | §3.9 的幂等 reconcile / 纯校验；写 `/etc/xvirt/resource-plan.json` |
| hyper-agent | `memtier_host_status` | 总控快照：压力合成值、水位线、每 VM 下发档位（供 §4.2 聚合端点） |
| lictor | `launch_vm` 扩展 + `vm_pin_vcpus` | scope 放置（§3.3）；运行中改独占绑定（query-cpus-fast + setaffinity + scope cpuset 更新） |

### 4.4 校验规则（validate 与 apply 共用）

- 池两两不交、并集 = 全部在线 CPU；HT 兄弟同池；system ≥ 2C、virt ≥ 2C。
- 缩池冲突：目标池变更后，受影响核上有运行中 VM 的独占绑定 → 列出阻塞 VM 清单拒绝
  （需先解绑/迁移/关机）。
- spdk 池变更时校验 isolated 前提（所有顶层 slice 均已显式 AllowedCPUs——由本方案
  自动满足）。
- 大页：目标值 < 当前 `nr - free`（在用）→ 拒绝；1G 增量运行时分配失败 → 降级为
  pending-reboot 而非失败。
- 内存恒等式（§3.4）成立且安全余量 ≥ 4%；VM 池预算低于运行中 VM 保证线 → 拒绝，
  低于超分线 → 警告。
- VM 放置：pins ⊆ 所属池；模式互斥矩阵（§2.3）；模式 C 的节点大页余量足额。

---

## §5 页面设计与交互

### 5.1 信息架构与整体布局

入口：**主机详情新增第 5 个 tab「资源规划」**（standalone 与 center 的 host 详情
共用；center 后续可加聚合视图）。页面 = 左侧四块面板纵排 + 右侧检查器栏：

```
┌─ 工具栏：查看/编辑切换 · 汇总 chips（64 CPU · 已隔离 4 · 大页 48G · 预留 12G）─┐
│            编辑态追加：变更计数 + 校验 + 应用 + 放弃                           │
├───────────────────────────────────────────────┬───────────────────────────────┤
│ ① CPU 拓扑板                                  │ 检查器（右栏 320px）          │
│    socket → NUMA 节点 → 物理核（HT 成对）网格 │   随选中对象切换：核/池/      │
│    颜色=池归属，底部填充=实时利用率           │   内存段/VM 的详情与精确输入  │
│    池图例兼拖放目标                           │   编辑态显示变更清单          │
│ ② 内存与大页板                                │   （逐项撤销）                │
│    每 NUMA 节点一条分段横条 + 拖拽手柄        ├───────────────────────────────┤
│    大页明细行（2M/1G × 总量/在用/SPDK 圈占）  │ 虚拟机放置列表                │
│ ③ 内存分层板                                  │   每 VM 一行：池 · 绑定 ·     │
│    主机压力计 + 两条水位线刻度                │   NUMA · 内存模式 ·           │
│    逐 VM 常驻/换出条 + 优先级；后端盘水位     │   memtier 徽标                │
│ ④ SPDK 板                                     │   行可拖到 CPU 板做独占绑定   │
│    预留核 · 大页圈占 · 低抖动启动档开关       │                               │
└───────────────────────────────────────────────┴───────────────────────────────┘
```

### 5.2 CPU 拓扑板

每个 NUMA 节点一个分组（标题：`Socket N · NUMA N · 8C/16T · 内存 128G ·
大页 2M×8192 · 1G×16 · 池内利用率`），组内物理核为单元格、HT 两线程上下成对；
单元格颜色 = 池，底部填充高度 = 5s 实时利用率，SPDK 池虚线边框（isolated），
编辑草稿中的变更用斜纹待定态渲染，独占核上方显示所属 VM 名 tag。

- SPDK 核显示 100% 填充是轮询的诚实呈现，chip 上注「轮询占用」；接入 SPDK stats
  后改显示有效利用率。
- hover tooltip：cpu id、池、utilization、驻留的 vCPU 线程（vm 名 + vcpu 序号）、
  残余内核活动说明。
- 点击独占核 tag 联动右栏选中该 VM。

### 5.3 内存与大页板 · 内存分层板

**内存条**（每 NUMA 节点一条，编辑模式段间竖条手柄可拖，段宽即额度）：

```
[内核·其它 6G][OS 4G‖][虚拟化 8G‖][大页2M 16G‖][大页1G 16G ⟳‖][VM池 已用41G/预算58G][空闲·缓存 20G]
  ‖ = 可拖拽手柄     ⟳ = 该段变更需重启生效
  附注行：大页在用 9.2G（VM×3）· SPDK 圈占 1G×4 · 安全余量 ≥ 4%（校验器强制）
```

**内存分层板**（数据源：x-memtier-query 聚合 + 总控状态）：

```
主机内存压力 0.32 · 当前档位：下刷中
[■■■■■■■─────────|──────────────────|───────]   ← 压力计；两 tick = 下刷线10% / 回血线50%（编辑态可拖）
ai-train-02  [low]    [常驻▓▓░░░░░░░░换出(斜纹)]   常驻 0.8G · 换出 3.2G
oa-file-01   [normal] [常驻▓▓▓▓▓░░░░换出]          常驻 4.4G · 换出 3.6G
db-core-01   [high]   [常驻▓▓▓▓▓▓▓▓▓▓]            常驻 15.4G · 换出 0.6G
后端盘 nvme1n1p3 [▓▓▓░░░░░░░]                      7.4G / 24G · 换出 210 MB/s
```

- 「换出」段代表**不占宿主 RAM** 的部分——VM 池分段条上以斜纹叠加显示
  「memtier 借出」，让内存恒等式在视觉上闭合。
- 每行点开 = VM memtier 详情（现有 MemtierCard 数据 + swapin-faults/evict-aborts
  曲线）；优先级 chip 直接下拉改（走既有 `memtier.priority` API）。

### 5.4 操作逻辑

编辑态应用栏（工具栏右侧常驻）：`3 项变更 · [2 即时生效][1 需重启] · 放弃 | 校验 | 应用`

| 拖拽 | 手势 | 效果与约束 |
|---|---|---|
| 核 → 池 | 拖物理核（HT 成对）到图例池 chip；或框选多核后点池 chip | 草稿中重着色为斜纹待定态；缩池撞独占绑定时高亮阻塞 VM 并拒绝落点 |
| VM → 核 | 从右栏 VM 行拖到 CPU 板 | 悬停时同 NUMA 节点的合法核亮起幽灵框；落下 = 该 VM 转独占 + 绑定这些核；跨节点落点显示警告（破坏 NUMA 亲和） |
| 内存段手柄 | 横向拖竖条 grip | 吸附步进：预留 256M、2M 大页 2M、1G 大页 1G；右栏输入框同步（拖拽给趋势、输入给精确值）；1G 段即时标 ⟳ |
| 水位线刻度 | 拖压力计上的两个 tick | 下刷线恒 < 回血线（互相钳制） |

1. **查看模式**（默认）：5s 轮询指标刷新利用率叠加；一切只读。
2. **进入编辑**：快照当前 plan 为草稿（本地 state）；实时数据继续刷新但不覆盖草稿；
   其它人改动（version 变化）顶部横幅提示。
3. **编辑**：拖拽/输入 → 右栏变更清单逐条累积，每条带生效性徽标
   「即时 / 需 VM 重启 / 需主机重启」与撤销按钮；客户端实时跑轻量校验
   （池覆盖/恒等式/互斥矩阵）。
4. **校验**：`POST ?action=validate` 服务端 dry-run，结果逐条对应到变更清单
   （阻塞项红、警告黄）。
5. **应用**：`PUT resource-plan`（带 version）→ 异步任务，走既有任务面板/WS 跟踪；
   成功后刷新 plan + topology，pending-reboot 项保留横幅「N 项待重启生效」；
   409 → 提示重载合并。
6. **可达性**：所有拖拽操作均有右键菜单 + 检查器输入等价路径；权限沿用管理员 JWT，
   `updated_by` 落审计。

### 5.5 「操作系统资源占用」的展示口径

页面顶部（或概览 tab 联动）一条主机级归户条，让「OS / 虚拟化系统 / VM」三问有
直接答案，且各段可展开：

```
[内核 5%][系统服务 4%][虚拟化系统 9%][SPDK 6%][虚拟机 38%][空闲 38%]
  虚拟化系统 ▸ 展开：ferrovirt 1.2% · hyper-agent 0.8% · lictor 0.4% · mariadb 3.1% · nginx 0.6% · petrel 2.9%
  虚拟机     ▸ 展开：逐 VM scope（cpu.stat / memory.current / PSI）
```

| 展示项 | 计算口径 |
|---|---|
| 内核 | `/proc/stat` 全机 − Σ 顶层 slice `cpu.stat`（残差法）；内存 = meminfo 减各 slice current 与大页后的残差 |
| 系统服务 | `system.slice` + `user.slice` + `init.scope` 的 cpu.stat / memory.current |
| 虚拟化系统 | `xvirt.slice` 汇总，展开到 service 级 |
| 虚拟机 | `xvirt-machine.slice` 汇总，展开到 per-VM scope（这正是 §3.3 改造后才可能拿到的口径） |

### 5.6 前端实现要点

- **零新依赖**：拓扑网格/分段条/压力计全部 CSS + 少量 SVG，无图表库；拖拽用原生
  Pointer Events（控制台浮窗已有同款手写拖拽先例），不引 dnd 库。
- **新组件**：`components/host/ResourcePlanTab.jsx` 挂 host 详情 tab；子组件
  `CpuTopologyBoard / MemoryBoard / MemtierBoard / SpdkCard / PlanInspector /
  VmPlacementList`；草稿与 diff 逻辑仿 `diffVmConfig` 模式（orig 快照 + working
  state + 提交时差异）。
- **i18n**：全部文案走 `useI18n` 键（zh/en 字典同步补齐），不再新增硬编码中文。
- **降级**：无 plan（老 agent / 未初始化）→ 只读拓扑 + 「初始化资源规划」引导按钮
  （生成默认池：全部 vm-shared + 最小 OS/virt 池建议值）。

---

## §6 指标采集链路

复用既有链路：`hyper-agent exporter(:9101) → vmagent → VictoriaMetrics(:8428) →
ferrovirt /api/vcenter/metrics(PromQL 白名单) → 前端`。本方案只增 collector 与
白名单前缀；plan/topology 等配置态数据走 §4.2 的 REST，不进时序库。

| 新 collector（hyper-agent） | 指标（示例） | 源 |
|---|---|---|
| `collectors/cpu_pool.py` | `xvirt_cpu_busy_ratio{cpu,pool,node}` · `xvirt_pool_cpu_ratio{pool}` | `/proc/stat` 逐 CPU + plan 归属映射 |
| `collectors/slice.py` | `xvirt_slice_cpu_seconds_total / memory_bytes / psi{slice,service}` | 顶层 slice 与 service 的 `cpu.stat / memory.current / *.pressure` |
| `collectors/numa.py` | `xvirt_node_mem_bytes{node,kind}` · `xvirt_hugepages{node,size,state}` | node meminfo + hugepages sysfs |
| `collectors/vm_direct.py` | `xvirt_vm_cpu_usage_ratio / memory_used_bytes / cpu_pressure_*{vm}` | per-VM scope cgroup（即 `vm-direct-metrics-design.md` 的落地，§3.3 是它的前置） |
| `collectors/memtier.py` | `xvirt_vm_memtier_{resident,swapped,backing_used}_bytes / evict_mbps{vm}` | lictor `memtier_query`（总控每拍已在查，复用同一份缓存避免双倍 QMP） |

ferrovirt `metrics.rs` L1 白名单补 `xvirt_cpu_ / xvirt_pool_ / xvirt_slice_ /
xvirt_node_ / xvirt_vm_memtier_` 前缀。页面查询频率 5s，与既有 VMMonitor 一致。

---

## §7 分期实施

| 阶段 | 交付 | 动到的组件 |
|---|---|---|
| **P0 · 看得见** | 拓扑扫描落库；cpu_pool/slice/numa/memtier collectors；「资源规划」tab 只读版（拓扑板 + 内存板 + memtier 板 + 归户条，池归属显示为「未规划=全部共享」） | hyper-agent（scan + collectors）、ferrovirt（topology API + 白名单）、xvirt-client |
| **P1 · 池化落地** | plan 模型/REST/本地缓存/开机对账；slice AllowedCPUs + memory.min/high；**per-VM scope 改造**；编辑模式（核→池拖拽、内存段手柄、2M 大页）+ validate + 异步应用 | ferrovirt（plan/validate/task）、hyper-agent（apply/reconcile + 单元）、lictor（scope 启动）、xvirt-client（编辑器） |
| **P2 · VM 落位** | vm_config 放置列 + PATCH 扩展；独占绑定（VM→核拖拽 + query-cpus-fast pin）；单节点自动放置 + 显式 vNUMA；模式 C 大页后端；vm_direct collector | ferrovirt、lictor（backend/pin）、hyper-agent（config 传递）、前端（VM 放置 UI + vNUMA 图） |
| **P3 · SPDK** | spdk.slice + isolated 分区 + IRQ 驱离 + hugetlb 圈占 + SPDK 板；可选低抖动启动档（xvirt-init grub drop-in） | hyper-agent、xvirt-init、前端 |
| **P4 · memtier 融合** | 总控读 plan 水位线/预算（替换环境变量常量）；VM 池预算项进压力合成；可选热集 memory.min 同步；水位线 UI 编辑 | hyper-agent（controller.py）、ferrovirt、前端 |

P0/P1 是主体（P1 里 per-VM scope 是全方案的地基，也同时解锁 per-VM 指标直采）；
P2–P4 相互独立，可按需要排序。每阶段都端到端可用、可在 123/106 真机验证，并可给
xvirt-autotest 加 stage（plan validate/apply 幂等性、缩池阻塞、互斥矩阵 400 等
都是好用例）。

---

## §8 风险与已知坑

- **存量 QEMU 迁入 scope 的计费偏差**：已 fault 的匿名页留在旧 cgroup 计费，
  `memory.current` 初期偏低（§3.3）。选择「重启渐进生效」或接受初期统计偏差并在
  UI 标注。
- **缩 cpuset 与线程亲和的顺序**：先清/改线程 `sched_setaffinity` 再缩 scope
  cpuset，否则 affinity ∩ cpuset = ∅ 会报 EINVAL；reconcile 内固定「先松绑后缩池」
  顺序。
- **isolated 分区写入失败**：任何一个顶层 cgroup 没设 AllowedCPUs 或与 spdk 池
  重叠，写 partition 返回 EINVAL 且**无 dmesg 提示**——agent 必须回读
  `cpuset.cpus.partition` 确认（值可能是 `isolated invalid`），失败时逐项报告是
  哪个 slice 冲突。
- **1G 大页运行时分配**：开机越久成功率越低；已按「pending-reboot 降级 + 开机早期
  投放」设计，不要在 apply 里无限重试 compact。
- **memory.min 过配**：Σ min 逼近 MemTotal 会把全局回收逼到无页可扫、触发 OOM——
  恒等式的 4% 安全余量是硬校验，不给 UI 绕过入口。
- **VM 池 memory.high 节流的连带效应**：顶到 high 后 QEMU 线程会被 reclaim 节流，
  vCPU 出现毛刺——memtier 水位线应设在 high 之前触发（预算项进压力合成正是为此，
  §3.7）。
- **vhost-net/内核线程不可归户**：其 CPU 落「内核/其它」段；重 IO 的 VM 会让该段
  偏大，口径说明要写进 tooltip，避免被当成泄漏排查。
- **systemd 与手写 cgroupfs 的共管边界**：AllowedCPUs 走 systemd（防 daemon-reload
  覆盖）；partition/hugetlb 上限走直写 + 开机单元重放；不要两边写同一属性。
- **单 NUMA 开发机 vs 多 NUMA 生产机**：所有 per-node 逻辑必须在 node 数=1 时退化
  正确；autotest 用例要在两种拓扑各跑一轮。
- **petrel OSD 的突发 CPU**：recovery/rebalance 时 virt 池可能被打满，影响管理面
  响应——virt 池规模建议随「是否跑 OSD」联动，页面在 virt 池 PSI 高时给提示。
