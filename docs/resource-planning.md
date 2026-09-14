# xvirt 资源规划：CPU / 内存隔离与预留方案（v1 草案）

> 为 xvirt 设计主机级 CPU、内存的隔离与预留能力：把操作系统（内核 + 系统服务）、
> 虚拟化服务、SPDK、虚拟机划分为可视、可拖拽调整的资源池；覆盖 NUMA 拓扑、
> 内存大页与内存分层的展示与管控。
>
> 2026-09-01 · 生产内核 6.7（cgroup v2 / cpuset isolated 分区可用）
> · 涉及管理服务 / 宿主机代理 / 虚拟机监控服务 / Web 控制台
> · 可视化版本（含页面 mockup）：https://claude.ai/code/artifact/4fcb9c64-0910-462c-9601-5bae73f9fff6
> · 可交互页面原型（独立 demo，不依赖 Web 控制台）：`demo/resource-plan/index.html`（见同目录 README）
> · 依据：平台组件说明、内存分层架构与 QMP 用法文档、
>   `vm-direct-metrics-design.md`、`monitoring/README.md`

---

## §0 摘要 · 选型决策

一句话：**以 cgroup v2 cpuset 分区为唯一基础机制做运行时可调的 CPU/内存池化，
每台 VM 落入独立 systemd scope；隔离、预留、大页、NUMA、SPDK、内存分层统一收进
一份 per-host「资源规划（resource plan）」，由宿主机代理幂等对账落地，前端在
主机详情新增「资源规划」tab 做图形化展示与拖拽编辑。**

| 问题 | 决策 | 理由 |
|---|---|---|
| CPU 隔离用什么 | **cgroup v2 cpuset 分区**（`cpuset.cpus.partition`），不以 `isolcpus` 为基础机制 | isolcpus 需重启、静态、全有全无；内核 6.7 的 `isolated` 分区在运行时提供等效隔离（脱离调度域、无负载均衡），正好支撑「拖拽即生效」 |
| SPDK 轮询核 | cpuset `isolated` 分区 + 可迁移 IRQ 驱离 + unbound workqueue cpumask 排除；managed IRQ（NVMe/mlx5）只能靠 `isolcpus=managed_irq`，与 `nohz_full` 一起归入「低抖动启动档」（标「需重启」） | 轮询线程要独占核、免时钟/中断干扰；isolated 分区覆盖 90% 需求，极致低抖动才需要 boot profile |
| per-VM 资源边界 | **systemd transient scope**：`vm-<uuid>.scope` 挂 `machine.slice`，QEMU 经 `systemd-run --scope` **在 exec 之前**就落入 scope | 目前全部 QEMU 挤在虚拟机监控服务 unit 的 cgroup 里（KillMode=process 的副作用），无法逐 VM 计量/限额；先起进程再搬会让 prealloc 的内存与大页永久计费到旧 cgroup（§3.3）；scope 顺带让 QEMU 生命周期独立于虚拟机监控服务重启 |
| vCPU 绑定 | scope 级 `cpuset` 定池 + QMP `query-cpus-fast` 拿线程 id 后 `sched_setaffinity` 做 1:1 独占绑定 | 池归属用 cgroup（可整体挪），精确 pin 用线程亲和性；两层正交 |
| 内存预留 | `memory.min` 保护 OS / 虚拟化服务 /（P4）VM 热集；`machine.slice` 用 `memory.high` 做 VM 池软预算，再以 `memory.max` 兜底 + 每 scope `memory.oom.group=1` | min 只保护已驻留页、不预扣内存；VM 匿名页在无 swap 主机上不可回收，`high` 只能节流不能阻止超限，没有 `max` 时预留会被吃穿、最后是全局 OOM 随机杀。`max` 取"MemTotal − 各项预留 − 大页 − 余量"，只在主机本来就要 OOM 的点上把代价收敛成"杀一台 VM" |
| 大页 | per-NUMA sysfs 运行时分配（2M 常态可行）；1G 走启动参数/开机早期预留，UI 标「需重启生效」；`hugetlb` cgroup 圈 SPDK 与 VM 的用量 | 1G 页在内存碎片化后运行时分配基本必失败 |
| VM NUMA 落位 | 三态：**浮动**（默认，不绑核不绑 NUMA，跑在未预留资源上）/ 绑定单节点（只用 scope 的 `cpuset.mems`，QEMU 不带 `host-nodes`）/ 显式 vNUMA（`-numa node,memdev`，per-memdev `host-nodes`） | 与现状兼容（今天所有 VM 都是浮动）；需要亲和的 VM 再显式绑 |
| 动态自动绑定 / 在线调整 | 主机级放置控制器（§3.10）：启动时按余量给 A 独占 / B 节点共享 / C 浮动；运行中靠 scope 的 `AllowedCPUs`（即时）+ `sched_setaffinity`（即时）+ `AllowedMemoryNodes`（cgroup v2 写 `cpuset.mems` 触发内核异步迁页）在线搬 vCPU 与内存 | 不需 QEMU/guest 配合、6.7 原生；与 libvirt `numatune --live` 同机制；用户显式放置永不被动 |
| 内存分层关系 | 仅「匿名内存」后端模式的 VM 可开启（现状约束）；主机总控的水位线/预算改从 resource plan 读取并在 UI 暴露 | 内存分层强制 4K 匿名页，与大页、vhost-user（分布式存储/SPDK）共享内存互斥；总控现在的阈值是环境变量常量，应归入规划统一管理 |
| 前端形态 | 主机详情新 tab「资源规划」：查看模式实时叠加用量，编辑模式草稿 → 校验 → 异步任务应用 | 沿用既有 TaskManager / WS / i18n 约定 |
| 数据链路 | 配置走新 REST（plan API + 本地缓存对账）；用量走既有宿主机代理 exporter → 时序库 → 管理服务指标代理 | 监控链路已建成，补 collector 即可；顺带落地 `vm-direct-metrics-design.md` 里搁置的 per-VM cgroup 直采 |

---

## §1 目标与非目标

### 目标

- **隔离**：OS（内核 + 系统服务）、虚拟化服务（管理服务/宿主机代理/虚拟机监控服务/分布式存储/数据库/Web 网关等）、SPDK、虚拟机四类负载互不侵占 CPU；SPDK 核完全独占。
- **预留**：OS 与虚拟化服务的内存有硬保障（不被 VM 压力回收挤死）；大页按 NUMA 节点、按页大小（2M/1G）定量预留；SPDK 的核与大页单独圈占。
- **可视**：主机 NUMA 拓扑、每个逻辑 CPU 的池归属与实时利用率、每 NUMA 节点内存的分段构成（内核/预留/大页/VM/空闲）、每台 VM 的占用与放置、内存分层常驻/换出/后端盘用量，全部图形化。
- **可拖拽**：核心 → 池、VM → 核（独占绑定）、内存段边界手柄，都可拖拽调整；所有操作有非拖拽等价入口（右键菜单/输入框）。
- **可对账**：规划持久化于 DB + 主机本地缓存，重启后由宿主机代理先于 VM 启动幂等重放。

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
| **virt**（虚拟化服务） | `virt.slice`：管理服务 / 宿主机代理 / 虚拟机监控服务 / 集群服务 / 分布式存储守护 / 数据库 / Web 网关 / 控制台代理 / 虚拟交换机用户态 | 软边界 | 2–4C，跑分布式存储 OSD 的节点加大 |
| **spdk** | `spdk.slice`（reactor 轮询线程） | **isolated 分区**：脱离调度域 + IRQ 驱离 | 按 vhost 目标带宽定，1–4C 起步；未部署 SPDK 时池可为空 |
| **vm-shared**（VM 共享 = **未预留资源**） | `machine.slice` 下未独占的 VM scope。它是**残差池**：system / virt / spdk / dedicated 之外的全部核自动归此，不需要（也不能）单独"分配"——把核从别的池释放即回到这里 | 软边界 + per-VM `cpu.weight` | 其余全部 |
| **vm-dedicated**（VM 独占） | 被某台 VM 1:1 绑定的核（vCPU 线程 pin），从 vm-shared 中划出 | 逻辑独占（同池内不共享）；可选升级为 isolated 子分区（前提：`machine.slice` 自身先写 `cpuset.cpus.partition=root` 成为有效分区根，子 scope 才能写 `isolated`） | 按需 |

> **浮动 VM（floating）**：不绑核、不绑 NUMA 的虚拟机——即今天系统里所有 VM 的形态。
> 它们与做了预留/绑定的 VM **长期共存**：调度范围 = vm-shared 全部线程（两节点），
> 内存来自各节点扣除内核·预留·大页后的 VM 池余量，落点由内核 first-touch 决定、
> 可能跨节点。规划页面必须把这一类单独呈现（台数 / Σ vCPU / 共享池线程 / 超分比、
> 内存条上的实测跨节点分布），缩共享池时按它们的规格和超分比校验（§4.4）。

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
| 虚拟化服务预留 | virt.slice 保障额（数据库是大头） | 同上 | `memory.min` |
| 大页 2M / 1G | 预留的 HugeTLB 池（含 SPDK 圈占部分，单列） | `node*/hugepages/*/nr_hugepages,free_hugepages` | sysfs per-node 写入；hugetlb cgroup 圈用量 |
| VM 池 | machine.slice 预算：已用（各 VM `memory.current` 之和）+ 余量 | cgroup + 规划值 | `memory.high`（软） |
| 内存分层视图 | VM 池内叠加显示：Σ常驻 vs Σ已换出（换出部分**不占**本节点内存，是「借出去」的量）+ 后端盘水位 | 虚拟机监控服务的内存分层查询 聚合 | 总控水位线（§3.7） |
| 空闲 / 页缓存 | MemAvailable 口径其余部分 | `/proc/meminfo` | — |

### 2.3 VM 内存后端三模式（关键兼容矩阵）

这是整个方案里最硬的约束来源：内存分层只吃匿名 4K 内存，vhost-user（分布式存储现在、
SPDK 将来）要共享内存，大页要 hugetlbfs/memfd-hugetlb。三者不可兼得，必须做成
VM 级显式属性 `mem_backend`，建 VM 时选定、互斥项在 UI 直接置灰（管理服务校验 400）：

| 能力 | A · 匿名（默认，`-m`） | B · 共享 memfd（`share=on`） | C · 大页（memfd `hugetlb=on` + `share=on`，2M/1G） |
|---|---|---|---|
| 内存分层 | ✓ 唯一支持 | ✗ | ✗（内存分层强制 MADV_NOHUGEPAGE） |
| 分布式存储 vhost-user-blk | ✗ | ✓ 现状 | ✓ |
| SPDK vhost-user | ✗ | △ 可用但不建议 | ✓ 推荐（DMA 走大页） |
| EPT/TLB 大页收益 | ✗（THP 视情况） | ✗ | ✓ |
| 内存超分能力 | ✓（内存分层冷页下刷） | 仅 balloon | ✗（prealloc 全预留） |
| 在线迁移 | ✓（先 disable 内存分层收敛） | ✓（分布式存储盘已验证） | ✓（两端需同额大页） |
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
主机初始化写 grub drop-in，UI 标「需重启」）。

### 3.2 cgroup / systemd 层级

```
/sys/fs/cgroup/
├── init.scope                AllowedCPUs=<system 池>
├── system.slice              AllowedCPUs=<system 池>   memory.min=<OS 预留>
├── user.slice                AllowedCPUs=<system 池>
├── virt.slice               AllowedCPUs=<virt 池>     memory.min=<虚拟化服务预留>
│   ├── 管理服务 / 宿主机代理 / 虚拟机监控服务的 unit
│   ├── 集群服务 / 数据库 / Web 网关 的 unit   ← 用 drop-in Slice=virt.slice 归入
│   └── 存储守护 *.scope           ← 分布式存储守护（宿主机代理托管时以 scope 拉起）
├── spdk.slice                AllowedCPUs=<spdk 池>
│   │                         cpuset.cpus.partition=isolated   ← agent 直写 cgroupfs
│   └── spdk-vhost.service（后续）
└── machine.slice       AllowedCPUs=<vm-shared ∪ vm-dedicated>
    │                         memory.high=<VM 池预算>  memory.max=<VM 池硬顶，§3.4>
    │                         memory.min=Σ 子 scope 的 memory.min（层级分配，否则子项 min 无效）
    ├── machine-shared.slice  不设 AllowedCPUs 之外的属性；C 级（浮动）scope 挂这里，
    │   │                     AllowedCPUs=<vm-shared>，缩共享池只改这一处
    │   └── vm-<uuidA>.scope  继承 cpuset      cpu.weight=100     memory.oom.group=1
    └── vm-<uuidB>.scope    cpuset=独占核集合    cpu.weight=10000   memory.oom.group=1
                              (A/B 级逐 scope 显式设 AllowedCPUs / AllowedMemoryNodes；memory.min 可选)
```

- **isolated 分区的前提**：内核要求分区的 CPU 不与**已配置** `cpuset.cpus` 的兄弟
  cgroup 重叠（未设值的兄弟不构成冲突）。给所有顶层 slice（含 `init.scope`/`user.slice`）
  显式设 `AllowedCPUs` 是充分条件而非内核要求，「全量池化」天然满足；反过来对账时
  要扫根下**全部**子 cgroup（含临时 scope、容器运行时建的组），不能只查已知的几个 slice。
  写入失败不返回 EINVAL：写成功但回读 `cpuset.cpus.partition` 为 `isolated invalid (<原因>)`，
  原因字符串可直接上报（如 `Cpu list in cpuset.cpus not exclusive`）。
- **共享池变更的扇出**：浮动（C 级）scope 挂 `machine-shared.slice` 且不设自己的
  AllowedCPUs，直接继承，缩共享池一次写入即可；A/B 级 scope 仍逐个显式设。
- **memory.min 的层级约束**：cgroup v2 的 min 保护按层级分配——子 scope 的
  `memory.min` 只有在父 `machine.slice` 自身设了 ≥ Σ 子项的 `memory.min` 时才有效。
  控制器在增删任一 scope 的 min 时同步改写 `machine.slice` 的 min（§3.7 P4 依赖此）。
- **归并系统服务**：数据库/Web 网关等属于虚拟化平台的服务用 drop-in
  `Slice=virt.slice` 归入 virt 池——「虚拟化系统占用」在页面上才是一条完整、
  诚实的曲线。
- **下发方式**：`systemctl set-property`（运行时 + 自动落 drop-in 持久化）设
  AllowedCPUs / memory.\*；`cpuset.cpus.partition`、hugetlb 上限等 systemd 不覆盖
  的属性由 agent 直写 cgroupfs，并由开机对账单元重放（§3.9）。

### 3.3 per-VM scope：QEMU 从虚拟机监控服务的 cgroup 里搬出来

**现状**：虚拟机监控服务拉起 QEMU（subprocess），QEMU 全部落在 `虚拟机监控服务 unit` 的
cgroup；服务重启靠 `KillMode=process` 保命。后果：无法逐 VM 计量
（memory.current/cpu.stat/PSI 全是糊的）、无法逐 VM 限额、也无法把 VM 整体挪池。

**改造**（虚拟机监控服务 `launch_vm`）：

1. **在 exec 之前就落入 scope**：`systemd-run --scope --unit=vm-<uuid>
   --slice=machine.slice（C 级为 machine-shared.slice） -p Delegate=yes
   -p CPUWeight=… [-p AllowedCPUs=… -p AllowedMemoryNodes=… -p MemoryMin=…]
   -p ManagedOOMPreference=none -- setsid --fork qemu … -S`。
   不能「先起进程再 `StartTransientUnit` 搬 pid」：模式 B/C 的 `prealloc=on` 在
   `-S` 暂停点之前就完成全部内存分配，搬 pid 不搬已计费的页，`memory.current`
   会永久偏低、`hugetlb.*.max` 对这台 VM 完全失效——这不是一次性的存量问题，
   每次开机都会发生。虚拟机监控服务今天已是 `setsid --fork` 双 fork 拉起，
   父子关系本就断开，`systemd-run` 包一层不改变任何既有语义；
2. 虚拟机监控服务拿到的 Popen 句柄是 `setsid`/`systemd-run` 自身，**真实 QEMU pid
   从 pidfile（`-pidfile`）或 QMP 取**，后续 numa_maps / 线程 id 一律用该 pid；
3. 独占 VM 经 QMP `query-cpus-fast` 取 vCPU 线程 id，逐线程 `sched_setaffinity`
   1:1 绑核；scope 上再写 `memory.oom.group=1`（systemd 不覆盖，直写 cgroupfs）；
4. QMP `cont` 放行。
5. **模式 B 强制 `prealloc=on`**：共享 memfd 的 shmem 页计费到首次触碰者，若分布式
   存储后端先于 QEMU 触碰，页会记到 `virt.slice`、吃掉虚拟化预留。

- **顺带收益**：QEMU 不再住 service cgroup，虚拟机监控服务/宿主机代理重启天然不波及 VM，
  `KillMode=process` 从「保命 hack」降级为双保险。
- **存量迁移**：升级后对账把运行中 QEMU 的 pid 写入新建 scope 即可（cgroup v2 允许
  迁移）；注意已计费的匿名页仍挂在旧 cgroup，`memory.current` 要到页面重新 fault
  后才逐步归位——统计初期偏低是预期行为，标注即可，或选择「下次 VM 重启时生效」。
- **emulator/iothread 分离**（P2 可选）：scope 开 `Delegate=yes` 后建 threaded 子组
  `vcpu/` 与 `emulator/`，独占 VM 的模拟器线程可留在共享池，进一步降低独占核抖动。

### 3.4 内存预留与 VM 池预算

```
# 预算不等式（per host，规划器校验；内核不强制，见下）：
OS 预留(memory.min@system.slice)
 + 虚拟化服务预留(memory.min@virt.slice)
 + Σ 大页池(2M+1G，含 SPDK 圈占)
 + VM 池预算(memory.high@machine.slice)
 ≤ MemTotal − 内核残差观测峰值 − 安全余量(≥ 4% MemTotal)

memory.max@machine.slice = MemTotal − OS 预留 − 虚拟化服务预留 − Σ 大页池 − 安全余量
```

- `memory.min` 是「预留」的落地：VM 压力触发全局回收时，受保护 slice 的页不被
  扫描。**但 min 不预扣内存，只保护已驻留页**：系统服务事后增长的部分要从别处回收，
  而模式 A（未开分层）与模式 B 的匿名/shmem 页在无 swap 主机上**不可回收**，
  `memory.high` 此时只能节流、不能阻止超限，最后是全局 OOM 随机杀进程——所以上面的
  预算式只是校验器约定，内核并不替你守住它。前提：根上 `subtree_control` 启用 memory
  控制器（systemd 默认满足，agent 对账时校验）。
- 内核残差是实测且波动的量，校验器按「Σ 规划项 ≤ MemTotal − 观测残差峰值 − 余量」
  实现（残差峰值取 exporter 近 7 天最大值，首次取当前值 × 1.2）。
- VM 池：`memory.high` 做软预算（顶到 high 触发直接回收 + 节流，给内存分层总控留
  下刷窗口），再以 `memory.max` **兜底**——取「MemTotal − 各项预留 − 大页 − 余量」，
  它只在主机本来就要 OOM 的那个点上生效，配合每个 scope `memory.oom.group=1`，把
  最坏情况从「随机杀进程」收敛成「整台杀一个 VM」（杀谁由 `memory.oom.group` +
  oom_score_adj 按优先级映射决定）。
- **hugetlb 不计入 `memory.current`**：6.7 上 `memory_hugetlb_accounting` 挂载选项
  默认关闭，模式 C 的主内存走 `hugetlb.*` 控制器、不进 memory 控制器——VM 池校验与
  `memory.high/max` 只算模式 C VM 的 QEMU overhead，其主内存在大页池段核算。
- VM 池预算校验规则：`Σ(运行中非内存分层 VM 内存 × (1+overhead)，模式 C 只算 overhead) ≤ 预算`；
  内存分层 VM 按 `min-resident` 折算下限、按全额折算上限，两条线都在 UI 显示
  （「保证线 / 超分线」）。QEMU per-VM overhead 取 `3% + 300MB` 经验值，进规划器常量。

### 3.5 大页管理

- **分配**：写 `/sys/devices/system/node/node<N>/hugepages/hugepages-{2048kB,1048576kB}/nr_hugepages`，
  按节点精确投放。2M 运行时通常可满足（失败时 agent 先 `compact_memory` 重试一次）；
  **1G 运行时分配在碎片化后基本必失败**——策略：规划里记目标值，agent 尝试运行时
  分配，不足额则该项标「需重启」，由开机早期单元（`resource-plan.service`，
  §3.9）在 boot 早期补齐，兜底方案才是 grub `hugepagesz=1G hugepages=N`。
- **圈占**：SPDK 与 VM 共用 HugeTLB 池但通过 `hugetlb.2MB.max` / `hugetlb.1GB.max`
  在 `spdk.slice` 与 `machine.slice` 上互相设限，防一方吃光另一方的预留。SPDK 走
  mmap 预留，`.max` 与 `.rsvd.max` 要**一起设**（前者管 fault 时的计费，后者管
  mmap 时的预留计费）。
- **控制器要手工开**：systemd 不会在根 `cgroup.subtree_control` 里启用 `hugetlb`，
  开机对账单元（§3.9）负责写 `+hugetlb`（根与 `machine.slice`/`spdk.slice` 逐层），
  agent 对账时校验。
- **消费**：模式 C VM 用 `memory-backend-memfd,hugetlb=on,hugetlbsize=2M|1G,share=on,prealloc=on`
  （免 hugetlbfs 挂载点管理）；SPDK 用其自带 `--huge-dir/--mem-size`。

### 3.6 NUMA 落位

- **拓扑采集**：宿主机代理新增 `scan_cpu_topology`（并入 `scan_device` 结果树）：
  `/sys/devices/system/cpu/cpu*/topology/*`（socket/core/thread 配对）+
  `/sys/devices/system/node/node*/`（cpulist、meminfo、hugepages、`distance` 矩阵）。
  落管理服务新表（§4.1），供拓扑图渲染。
- **浮动（默认，`numa_mode=none`）**：不绑 NUMA。scope `AllowedCPUs=<vm-shared>`、
  `AllowedMemoryNodes=<全部节点>`，QEMU 不带 `host-nodes`；内存 first-touch，实际分布
  由 `numastat -p <qemu pid>`（或 `/proc/<pid>/numa_maps` 聚合）采集后展示。与现状零差异，
  升级后老 VM 自动落入此态。浮动 ↔ 绑定单节点可**热调**，但注意语义差别：只改
  `AllowedCPUs` 不迁页；改 `AllowedMemoryNodes`（= `cpuset.mems`）在 cgroup v2 下
  **必然触发全量异步迁页**（`memory_migrate` 恒开，§3.10.4），几十 GB 的 VM 会在
  后台搬几十秒到几分钟——所以「绑定单节点」的热调必须走 §3.10.5 的标准流程并在
  UI 显示进度，不能当成零成本操作。
- **绑定单节点（`numa_mode=node`）**：放置器选「绑定池核数与空闲内存都放得下」的最空节点；
  **只用 scope 的 `AllowedMemoryNodes=N`（`cpuset.mems`）**，QEMU **不带** `host-nodes`——
  两套机制叠加是重复约束，换节点时还要依赖内核对 MPOL_BIND 做 nodes_remap 才能跟上，
  多一层验证负担；`host-nodes` 只留给 vNUMA 的 per-memdev。vCPU 限该节点池内核（共享）
  或 1:1 绑定（独占）。
  独占绑核而内存仍浮动是允许但要警告的组合（跨节点访存）。
- **跨节点 VM（显式 vNUMA，`numa_mode=vnuma`）**：`vm_config.numa` 声明 vnode 列表（每 vnode：vcpus、
  内存、host-node），拼 `-numa node,nodeid=i,cpus=…,memdev=mi`，guest 内看到与宿主
  对齐的拓扑。页面 VM 详情画 vNUMA→host node 的映射关系。
- **降级**：单 NUMA 主机（现有 lab 的 E5-26xx v3 单路）一切照常，页面只渲染一个节点组。

### 3.7 内存分层融合

- **准入**：`mem_backend=anon` 才可开内存分层（既有分布式存储盘互斥校验推广为三模式
  矩阵校验）。
- **预算联动（P4）**：总控内存分层总控模块 现在用
  三个环境变量常量（空闲低/高水位、PSI 阈值）算主机压力。改为
  从本地 plan 缓存读：压力 = `max(PSI 项, 预算项)`，预算项 = VM 池 `memory.current`
  相对 `memory.high` 的挤占度。下刷/回血两条水位线成为 plan 字段，UI 可调
  （内存分层面板上的两个刻度，§5.3）。
- **热集保护（P4 可选）**：总控把每台 内存分层 VM 的 scope `memory.min` 同步为
  `min-resident-mb + overhead`，防 VM 池整体回收把热集也挤掉。**同时**必须把
  `machine.slice` 的 `memory.min` 维持为 Σ 子 scope min（§3.2），否则子项的 min
  在层级分配下为 0、保护无效。
- **展示数据**：全部来自既有 `内存分层查询接口`（经虚拟机监控服务）：
  `resident-bytes / swapped-bytes / backing-total|used / evict-mbps / swapin-faults /
  psi-some-avg10`，聚合进 exporter（§6）。

### 3.8 SPDK 预留

- **本期交付**：plan 中的 `spdk` 块（核列表 + per-node 大页额度）→ agent 建
  `spdk.slice`、写 isolated 分区、按额度校验大页池、驱离 IRQ 与 unbound workqueue：
  - 现存可迁移 IRQ 逐个改写 `smp_affinity_list`；irqbalance **未必安装**，agent 不依赖
    它，装了才补 `IRQBALANCE_BANNED_CPULIST` drop-in 防它改回来；
  - 6.7 的 isolated 分区**不会**把 CPU 从 unbound workqueue 的 cpumask 里摘掉
    （6.9 才有），agent 同时写 `/sys/devices/virtual/workqueue/cpumask` 排除 spdk 核；
  - NVMe、mlx5 等 **managed IRQ** 无法通过 `smp_affinity_list` 改绑（返回 EIO），
    只能靠 `isolcpus=managed_irq,<spdk cpus>` 启动参数——归入「低抖动启动档」。
- **后续接线**：SPDK vhost target 以 `spdk-vhost.service`（`Slice=spdk.slice`）部署，
  reactor mask = 池核；被服务 VM 走模式 C；reactor 是 100% 轮询，利用率图对 spdk 池
  改显示「占用/预留」而非 busy%（接 SPDK 自身 stats 后再显示有效利用率）。

**SPDK 隔离方式的结论（cgroup 分区，而非 isolcpus）**：

- **SPDK 不需要"绝对隔离"**。reactor 是用户态轮询线程，真正怕的是三件事：别的任务
  抢核、IRQ 打断、HT 兄弟核干扰。isolated 分区（脱离调度域、独占）+ 整核成对划入 +
  IRQ 驱离已覆盖这三件。残余只有周期 tick 与 per-CPU 内核线程：每核每秒数百次、每次
  约 1–2 µs，占用 < 0.2%、抖动微秒级，对几十微秒量级的 IO 路径没有实际影响。
- **isolcpus 做不到在线增减**：它是启动参数，改一次重启一次，且内核文档已标为
  Deprecated、建议改用 cpuset；在"调度域隔离"这一层 isolated 分区与
  `isolcpus=domain` 是同一件事，差别只剩 `nohz_full` / `rcu_nocbs` / `managed_irq`
  这些只能启动时设的项——归入可选的「低抖动启动档」，并且只声明 **SPDK 候选核上限**
  （如最多 8 核），之后实际给多少核仍由 cgroup 在上限内动态决定；候选核暂时跑 VM 时
  多付少量 context-tracking 开销，故上限不宜划大。
- **在线给 SPDK 加核（cgroup 层可行）**，固定顺序：从 `machine.slice` 的 cpuset 拿掉这对
  HT（VM 线程自动迁走）→ 写进 `spdk.slice` 的 cpuset（分区保持 isolated）→ 回读
  `cpuset.cpus.partition` 确认不是 `isolated invalid` → 改写现存 IRQ 亲和性 + irqbalance
  禁用列表 + workqueue cpumask。managed IRQ 只在本核发起 IO 时才触发，核上只有 SPDK 时
  不会响；SPDK 自己接管的 NVMe 走 vfio，没有内核 IRQ。
- **真正的限制在 SPDK 进程层**：reactor 核集由启动时的 core mask 固定，运行中不能新增
  reactor。两种做法：① 推荐——SPDK 启动时 reactor mask 覆盖候选核上限，开 dynamic
  调度器让线程只落在需要的 reactor 上，cgroup 只把当前实际需要的核给 `spdk.slice`，
  "加核" = cgroup 加核 + 调度器放开对应 reactor（需验证所用版本下空闲 reactor 是否真正
  停止轮询，否则被内核挪到允许核上空转）；② 兜底——改 core mask 重启 vhost target，
  依赖 inflight-shmfd 重连 + QEMU chardev `reconnect`，客户机不重启、仅短暂 IO 停顿。

**IRQ 与内核线程的诚实口径**：

- SPDK/独占池核通过 irqbalance banned + 显式 affinity 驱离可迁移 IRQ；per-CPU
  kthread（ksoftirqd 等）与 timer tick 无法用 cgroup 移走——这是 isolated 分区与
  isolcpus 的真实差距，UI 的池 tooltip 里注明「残余内核活动 < ~1%」。
- vhost-net 等内核线程不入任何 cgroup，其 CPU 计入「内核/其它」段；后续可按 VM 把
  vhost 线程 affinity 跟随其池（增强项）。

### 3.9 开机对账与应用顺序

```
resource-plan.service (oneshot, DefaultDependencies=no, After=local-fs.target,
                       Before=sysinit.target 虚拟机监控服务 unit 分布式存储守护 unit)
  读 /etc/virt-platform/resource-plan.json（DB 之外的本地缓存，PUT plan 时同步写）
  ① 1G/2M 大页早期投放（越早内存越干净，成功率最高——因此要早于 sysinit.target）
  ② 根 / machine.slice / spdk.slice 的 subtree_control +hugetlb；sysctl kernel.numa_balancing=0
  ③ spdk.slice 建立 + partition=isolated + hugetlb .max/.rsvd.max 上限 + IRQ / workqueue 驱离
  ④ 标记就绪 → 分布式存储守护、虚拟机监控服务启动 → 逐 VM scope 放置（launch 时逐台施加）
  注：顶层 slice 的 AllowedCPUs / AllowedMemoryNodes / memory.min|high|max 由
  `set-property` 落的 drop-in 在开机时由 systemd 自己应用，本单元不重复写，只回读校验
运行时 apply_resource_plan（异步任务）= 同一套幂等 reconcile，逐项返回
  applied / pending-reboot / failed(原因)
```

---

### 3.10 动态自动绑定与在线调整（放置控制器）

目标：**用户在界面手动指定了 CPU 绑定 / NUMA / 内存预留的 VM 是「固定」的，永远不被
自动改动；其余全部 VM 由控制器按主机当前负载在线求最优绑定——负载低时每台都拿到
独占绑核 + 绑节点，VM 变多时在线降级、跨节点再平衡，VM 减少时再升回去。** 所有
调整都在线完成（vCPU 即时、内存后台迁页，§3.10.4），不重启 VM。

#### 3.10.1 两类 VM：固定与自动

| 类别 | 谁决定 | 内容 | 控制器怎么对待 |
|---|---|---|---|
| **固定**（`placement_source=manual`） | 用户在 VM 视图显式设置，或点「锁定」 | CPU：独占核集合 / 绑定节点 / 浮动；内存：绑定节点或 vNUMA、`mem_reserved_mb`（scope `memory.min`）；后端模式 | **不可移动的约束**：它占的独占核、绑定节点上的内存与预留先从可用资源中扣除，求解只在剩余资源上进行 |
| **自动**（`placement_source=auto`，默认） | 控制器 | 同上三个维度，由求解器在线赋值 | 每拍重新求最优，变更走在线原语 |

强制固定（不给自动）的三类：显式 vNUMA（缺 per-memdev 迁移原语）、vfio 直通（页被
pin 不可迁）、1G 大页后端（迁移慢且依赖目标节点空闲 1G 页）。用户改动自动 VM 的任一
放置字段并保存 = 转为固定；「交回自动」则下一拍重新求解。固定 VM 的内存预留
`memory.min` 同时进入 §3.4 的预算恒等式，UI 在内存条上以实色标出。

自动 VM 的可能形态就是三个等级：

| 等级 | 含义 | 对应字段（§4.1） |
|---|---|---|
| **A · 独占** | vCPU 1:1 绑物理线程 + 内存绑同一节点 | `cpu_placement=dedicated, numa_mode=node` |
| **B · 节点共享** | 不绑核，但限定在一个节点的共享池内 + 内存绑该节点，`cpu.weight` 按优先级 | `cpu_placement=shared, numa_mode=node` |
| **C · 浮动** | 不绑核不绑 NUMA（§3.6 默认态） | `cpu_placement=shared, numa_mode=none` |

#### 3.10.2 「最优」的定义：目标函数 + 负载档位

**目标函数**（对自动 VM 集合求放置 P）：

```
Cost(P) = Σ_vm [ w_remote · remote_mem_ratio(vm)          // 跨节点访存比例（numastat 实测 / 目标为 0）
              + w_cont   · contention(vm) ]                 // 所在共享域的超分比 oc（独占 = 0）
        + w_bal · |load(N0) − load(N1)|                     // 节点间 vCPU 与内存用量不均衡
        + w_move · Σ_{vm 变更} move_cost(vm)                // 常驻内存 / 迁页带宽 + 冷却惩罚（滞回来源）
约束：固定 VM 不动；每节点 Σ内存 ≤ VM 池预算；独占核两两不交且 HT 成对；
      绑定后节点剩余共享线程 ≥ min_shared_threads；池边界 = plan 的共享池
```

`move_cost` 是避免抖动的核心：一个变更只有在「预计收益 ≥ 搬动成本 + 冷却惩罚」时才
执行，因此负载短暂波动不会引发来回搬。权重有默认值，不在 UI 暴露；UI 暴露的是下面
的档位阈值。

**负载档位**让「最优」可预测、可解释。主机负载指数
`L = max( 共享池 oc / bind_max_oc, VM 池内存用量比, PSI 归一化 )`，分三档，
进入 / 退出阈值不同（滞回）+ 稳定期：

| 档位 | 判定 | 自动 VM 的目标形态 |
|---|---|---|
| **宽裕** | 所有自动 VM 都能独占放下：Σ⌈vcpus/threads_per_core⌉ ≤ 空闲独占核，且每节点内存放得下 | **全部 A**：按节点装箱（vCPU 降序 first-fit，内存随 vCPU 同节点），两节点均衡 |
| **适中** | 能全部绑节点、不能全部独占 | **A/B 混合**：优先级高、CPU 利用率高的保留 / 升为 A；其余 B（`cpu.weight` 按优先级）；节点间按内存 + vCPU 均衡 |
| **紧张** | 内存或 vCPU 超分（`oc ≥ release_oc` 或内存用量 ≥ 预算） | **B/C**：独占核全部释放回共享池；vCPU 超分在两节点均衡；某节点内存放不下的 VM 允许 C（浮动、跨节点）；内存分层 VM 优先降级（它们对远端访存最不敏感） |

档位只决定「目标形态」，具体哪台 VM 落哪个节点、谁保留独占，仍由目标函数决定。

#### 3.10.3 求解与执行（每拍）

触发：VM 启停 / 迁入迁出 / plan 应用完成 / 每 30s 一拍。

```
① 采样   拓扑 + plan；每 VM：vCPU 利用率（scope cpu.stat）、常驻内存与节点分布
         （numa_maps）、优先级；每节点：oc、空闲独占核、空闲内存、PSI
② 定档   计算 L，按滞回决定档位（进入更高档立即，退出需稳定 promote_stable_s）
③ 求解   以当前放置为初解，目标形态为模板：贪心装箱（固定 VM 先占位 → 自动 VM 按
         优先级、vCPU 降序 first-fit 到 Cost 增量最小的节点/等级）→ 局部搜索（两两交换、
         单台迁移）直到 Cost 不再下降。VM 数在百量级，毫秒级完成
④ 差分   目标 vs 当前 → 动作列表，按成本排序：解绑核（免费）→ 绑核（免费）→
         改 cpu.weight（免费）→ 收编 C→B（几乎免费，只写 cpuset）→ 换节点（有成本）
⑤ 执行   免费动作本拍全部执行；换节点每拍最多 1 台、主机同一时刻最多 1 个内存迁移；
         每台 VM 冷却 cooldown_s；VM 启动后 settle_after_boot_s 内不动
⑥ 记录   placement_event（动作、from/to、触发指标快照、收益估计）+ WS 推送
```

**启动时准入**是同一求解器的增量调用：新 VM 按当前档位的目标形态入场（宽裕 → A，
适中 → 按优先级 A 或 B，紧张 → B 或 C），**不为它挤动别人**；下一拍再做全局优化。
准入在虚拟机监控服务 `launch_vm` 前由宿主机代理决定（同一台主机上串行，避免两台
VM 同时抢同一批核），结果随启动参数一起下发；启动后 `placement_tier` 落库并推 WS。

**oc 的口径**：节点共享池超分比 = Σ落在该节点的 B/C 级 vCPU ÷ 扣除独占后的共享线程
数；B 级 vCPU 全算到其绑定节点，**C 级（浮动）VM 的 vCPU 按各节点共享线程数占比
分摊**到每个节点，避免浮动 VM 在 oc 里被漏算或重复算。节点 CPU 压力：`cpu.pressure`
不分 NUMA 节点，取该节点上所有 VM scope 的 `cpu.pressure some avg60` 按 vCPU 数加权
聚合，或用 per-cpu `/proc/stat` 折算该节点共享线程的 busy%。

**守卫**：plan 应用中、主机参与在线迁移中、内存分层处于下刷档时全部暂停；
`mode=advise` 只记录建议不执行（页面显示「建议」，供上线初期观察）；
`dedicate_max_oc < release_oc` 且 `promote_stable_s > 0` 是硬校验，保证 A/B 不会
反复升降。所有决策写 `placement_event`（§4.1）可审计。

**一个例子**（2 节点 × 8 核 16 线程，共享池每节点 6 核）：早上只有 3 台自动 VM
（4/4/8 vCPU）→ 宽裕档，全部 A，N0 放 4+4、N1 放 8。中午再启 6 台 → 求解发现独占核
不够 → 适中档：优先级 high 的 8 vCPU 保留 A，其余转 B，两节点各 4 到 5 台；转换只是
解绑 + 改 cpuset，秒级完成。傍晚内存用量顶到预算 → 紧张档：剩余 A 释放，两台内存分层
VM 转 C 允许跨节点；一台 12G 的 B 级 VM 因 N0 内存吃紧被搬到 N1（先切 vCPU，内存
后台迁页约 10s）。晚上 VM 陆续关机 → 稳定 5 分钟后逐拍升回 A，每拍最多一次换节点。

#### 3.10.4 在线调整的方法（确定项）

前置：VM 已在独立 scope（§3.3）。不需要 QEMU 配合，也不需要 guest 配合。

| 调整项 | 方法 | 生效 | 说明 |
|---|---|---|---|
| vCPU 可用范围 / 整 VM 换节点 | systemd `SetUnitProperties(scope, runtime=true, AllowedCPUs=…)` = 写 `cpuset.cpus` | **即时**（内核在下一个调度点迁移线程，µs–ms 级） | 一次覆盖 vCPU + 模拟器 + iothread 全部线程 |
| vCPU 1:1 绑核 / 解绑 | 虚拟机监控服务：QMP `query-cpus-fast` 取每 vCPU `thread-id` → `sched_setaffinity(tid, {cpu})`；解绑 = affinity 设回 scope 全集 | **即时** | 「顺序不敏感」只对 cpuset 的收缩/扩张成立（6.7 保存用户 affinity `user_cpus_ptr`，v6.2+：cpuset 变更后取交集，交集为空回退 cpuset 全集）；**`sched_setaffinity` 本身在 mask 与当前 cpuset 交集为空时仍返回 EINVAL**。所以硬性顺序：**扩池 / 换节点先改 cpuset 再 pin，缩池先解 pin（affinity 设回全集）再缩 cpuset** |
| CPU 份额 | `CPUWeight=` = `cpu.weight` | 即时 | A 级 10000，B/C 级 100（可按优先级映射） |
| **整 VM 内存换节点** | systemd `AllowedMemoryNodes=` = 写 `cpuset.mems`。cgroup v2 语义：变更 `cpuset.mems` 时内核对 cpuset 内每个进程的 mm 调 `cpuset_migrate_mm()` → `do_migrate_pages(…, MPOL_MF_MOVE_ALL)`，在内核 workqueue 里**异步迁页**；新分配立刻落到新节点 | 异步，吞吐按主机实测（经验 1–3 GB/s/节点，首次上线用 `migrate_bw_mbps` 标定） | 匿名页 / memfd(shmem) / 2M hugetlb 都可迁；与 libvirt `numatune --live`（strict 模式）同一机制。进度用 `/proc/<qemu pid>/numa_maps` 聚合（或 `numastat -p`）轮询 |
| 按 memdev 换节点（vNUMA VM） | 我方 QEMU（内存分层分支）加 QMP `x-memdev-rebind {id, host-nodes}`：对该 backend 的映射 `mbind(MPOL_BIND, MPOL_MF_MOVE\|MPOL_MF_MOVE_ALL)` | 同步（可分片） | v1 不做，vNUMA VM 固定 manual；做了之后 vnode→host 映射可整体平移 |
| 外部兜底 | 宿主机代理直接 `migrate_pages(2)`（pid, from, to）/ `move_pages(2)`（pid, 页地址表），需 `CAP_SYS_NICE` | 同步 | cpuset 路径失效时的备用；`move_pages` 可做 VMA 粒度（配 `/proc/pid/maps`） |

**KVM 侧发生什么**：页迁移经 mmu_notifier 使 EPT 项失效，vCPU 下次访问该页
触发 EPT violation 重建映射——guest 只感到该页一次短暂停顿，看不到任何拓扑变化
（非 vNUMA 的 guest 拓扑本来就是单节点）。vCPU 线程换核对 guest 完全透明。

**为什么不选其它路**：QEMU 没有运行时改 `host-nodes` 的 QMP（memory-backend 属性
启动后只读）；guest 内 balloon/热插拔改的是容量不是位置；内核自动 NUMA 平衡
（`kernel.numa_balancing`）是全局开关、按页扫描制造 EPT 抖动且与本控制器互相
搬动——**主机上关掉它**（`=0`），由控制器做粗粒度、可审计的迁移。

#### 3.10.5 换节点标准流程（vCPU + 内存一起在线搬）

```
1 准入   非 vfio / 非 1G 大页 / 非 vNUMA；目标节点 free_mem ≥ resident×1.1（A 级再要
         ⌈vcpus / threads_per_core⌉ 空闲核）；主机无其它进行中的迁移；内存分层 VM → 通知总控对该 VM
         quiesce（复用在线迁移已有的 quiesce 路径）
2 vCPU   A 级先逐 vCPU 解 pin（affinity 设回当前 scope 全集），再 AllowedCPUs=<目标节点共享池>
         ← 即时；目标为 A 级则随后逐 vCPU 在新节点 pin（遵守 §3.10.4「先解 pin 再动 cpuset」）
3 内存   AllowedMemoryNodes=<目标节点>          ← 新页立刻落新节点，旧页后台迁移
4 等待   轮询 numa_maps，目标节点占比 ≥ 95% 或超时（resident/带宽 × 3）即结束；
         超时不回滚（残余远端页无害），页面标「部分迁移」，下拍继续
5 落库   vm_config.numa_node / placement_tier / placement_source=auto；写 placement_event；
         推 WS vm.placement_changed
6 恢复   内存分层总控 resume
```

先搬 vCPU 再搬内存：vCPU 到位后新分配全部本地，迁移窗口内的远端访问被迁页吞吐
上界住；反过来会让 vCPU 在旧节点上访问已搬走的页。

#### 3.10.6 限制与上线验证项（6.7）

- **vfio 直通**：全部 guest 内存被 pin，`migrate_pages` 返回 EBUSY——强制 manual。
- **1G 大页**：hugetlb 迁移要求目标节点有空闲 1G 页、且逐页拷贝慢——排除在自动之外；
  2M 大页允许，准入时校验目标节点 `free_hugepages`。
- **内存分层 VM**：uffd-wp 页（PTE marker，5.19+）在页迁移下的行为要在我方分支
  （swap-cache 设计）上实测——验证项 #1；已换出页不在内存，不受影响。
- **分布式存储 vhost 后端共享 memfd**：后端进程映射同一内存，`MOVE_ALL` 覆盖共享页；IO 进行中
  的页被临时 pin 时内核重试——验证项 #2：迁移期间持续 fio 无错误。
- **迁移吞吐与 EPT 抖动**：验证项 #3：8G/32G VM 迁移耗时、guest 内延迟毛刺分布，据此
  定 `max_move_gb`（默认 64）与是否需要分片（`move_pages` 路径）。
- **cpuset 与 affinity 交互**：验证项 #4：在 6.7 上确认缩 cpuset 后再扩回，vCPU 的
  1:1 pin 自动恢复（`user_cpus_ptr` 语义），否则控制器在扩池后重下发 pin。
- **迁页与内存策略/共享映射**：验证项 #5：绑定单节点 VM 改 `cpuset.mems` 后，若 QEMU
  仍带 `host-nodes`（vNUMA memdev）MPOL_BIND 的 nodes_remap 是否跟上；memfd 共享映射在
  分布式存储后端侧（另一个进程、另一个 cgroup）的迁页表现——页归谁计费、是否被后端
  的 mbind 顶回。

#### 3.10.7 组件落点

| 组件 | 内容 |
|---|---|
| 宿主机代理 | 放置控制器模块（与内存分层总控同形态的主机级循环：准入 + 再平衡决策 + 守卫）；执行层 = systemd D-Bus `SetUnitProperties` 改 scope 的 AllowedCPUs / AllowedMemoryNodes / CPUWeight，读 `numa_maps` 跟进度；action `placement_status` / `placement_rebalance` / `placement_advise`；publish `vm.placement_changed` |
| 虚拟机监控服务 | `vm_pin_vcpus` / `vm_unpin_vcpus`（query-cpus-fast + setaffinity）；`launch_vm` 接收 agent 决定的等级与节点 |
| 管理服务 | plan JSON `auto_placement{}`（§4.1）；`vm_config` 增 `placement_source / placement_tier`；`placement_event` 表；REST（§4.2）；事件摄取层接收 `vm.placement_changed` 落库 + WS |
| 前端 | 主机视图新增「自动放置」卡：开关 / 建议模式 / 阈值 / 最近决策与进行中迁移进度；VM 列表 tier 徽标（A/B/C）+ 来源（自动 / 🔒手动）；VM 视图「由控制器自动放置」提示 + 「锁定为手动」切换 |

---

## §4 数据模型与 API

### 4.1 DB（管理服务，数据库）

| 表 / 列 | 内容 |
|---|---|
| `host_resource_plan`（新表） | `host_id`(uniq)、`version`(乐观锁)、`plan`(JSON，见下)、`pending_reboot`(JSON 数组)、`updated_at / updated_by`(JWT sub) |
| `host_cpu_topology`（新表，扫描落库） | 逐逻辑 CPU 一行：`host_id, cpu, core, socket, node, online`；节点级信息（内存、distance）入既有 `device` 树或 JSON 列 |
| `vm_config` 增列 | `cpu_placement`(shared\|dedicated，默认 shared)、`cpu_pins`(JSON "vcpu→pcpu")、`numa_mode`(none\|node\|vnuma，默认 none=浮动)、`numa_node`(node 模式的目标节点)、`numa`(JSON vnode 列表)、`placement_source`(auto\|manual，默认 auto)、`placement_tier`(A\|B\|C，控制器回写)、`mem_backend`(anon\|memfd\|hugepage-2m\|hugepage-1g)、`mem_reserved_mb`(memory.min，可空)。均 `ALTER TABLE … ADD COLUMN IF NOT EXISTS`（沿用 ensure_schema 约定） |
| `placement_event`（新表，审计） | 控制器每个决策一行：`host_id, vm_uuid, ts, action`(admit\|promote\|demote\|rebalance\|bind\|advise)、`from_tier/to_tier`、`from_node/to_node`、`reason`(触发条件与当时指标快照 JSON)、`status`(applied\|advised\|failed\|partial)、`moved_bytes`、`duration_ms` |

```jsonc
// host_resource_plan.plan —— 单一 JSON 文档（池成员表述为 cpu 列表，物理核对齐由校验器保证）
{ "schema": 1,
  "pools": { "system": [0,1,32,33], "virt": [2,3,34,35],
             "spdk": { "cpus": [4,5,36,37], "isolated": true, "boot_profile": false },
             "vm":   { "cpus": [6, "...", 63] } },   // 独占核集合不入 plan：由 vm_config.cpu_pins 派生（单一真相），GET 时聚合返回
  "memory": { "os_reserve_mb": 4096, "virt_reserve_mb": 8192,
              "vm_budget_mb": 214016, "headroom_pct": 4 },
  "hugepages": { "node0": { "2m": 8192, "1g": 16 }, "node1": { "2m": 8192, "1g": 16 },
                 "spdk_quota": { "2m": 2048, "1g": 4 } },
  "memory_tiering": { "evict_watermark_pct": 10, "swapback_watermark_pct": 50, "psi_high": 10 },
  "auto_placement": { "enabled": true, "mode": "apply",            // apply | advise
      "dedicate_max_oc": 1.0, "release_oc": 1.5, "bind_max_oc": 2.0,
      "min_shared_threads": 4, "numa_imbalance_pct": 25, "psi_cpu_high": 20,
      "promote_stable_s": 300, "cooldown_s": 600, "settle_after_boot_s": 120,
      "max_move_gb": 64, "migrate_bw_mbps": 1500 } }
```

### 4.2 REST（管理服务，均挂 JWT）

| 端点 | 语义 |
|---|---|
| `GET /api/vcenter/host/{uuid}/resource-topology` | NUMA/socket/core/thread 拓扑 + 每 CPU 当前池归属 + 每节点内存/大页现势（拓扑图数据源，变更少，进页拉一次） |
| `GET /api/vcenter/host/{uuid}/resource-plan` | plan + version + pending_reboot + 各项实际落地状态（plan vs 实况 diff，供「未生效」标记） |
| `PUT /api/vcenter/host/{uuid}/resource-plan` | 带 `version` 乐观锁（409 = 需重载合并）；**异步任务**（kind=`applyResourcePlan`，走 TaskManager 约定），agent 幂等 reconcile，task.message 汇总逐项结果 |
| `POST …/resource-plan?action=validate` | 服务端 dry-run：完整校验（§4.4）+ 生效性标注，不落地。编辑器「校验」按钮与提交前置检查共用 |
| `PATCH /api/vcenter/vm/{vm}`（扩展） | `UpdateVmRequest` 增 `placement` 稀疏段（cpu_placement/cpu_pins/numa/mem_backend/mem_reserved_mb/source）；运行中仅 pins/weight/numa_node（走 §3.10.5 迁移流程）可热调，其余标「下次启动生效」；`source=manual` = 锁定（控制器不再动它），改回 `auto` 时控制器下拍重新准入 |
| `GET /api/vcenter/host/{uuid}/memory-tiering`（新聚合） | 各 内存分层 VM 的 分层状态信息 + 总控状态（当前压力、水位线、每 VM 当前档位）——面板一次拉全 |
| `GET /api/vcenter/host/{uuid}/placement` | 控制器状态：每节点 oc / free_ded_cores / free_mem、进行中的迁移（VM、目标节点、已迁百分比）、最近 N 条 `placement_event` |
| `POST /api/vcenter/host/{uuid}/placement?action=rebalance\|advise` | 手动触发一拍（advise 只出建议）；异步任务 |

### 4.3 agent actions（ZMQ）

| 组件 | action | 职责 |
|---|---|---|
| 宿主机代理 | `scan_cpu_topology` | 拓扑 + 每 CPU 实际 cgroup 归属 + 大页现势（挂进 hardware-scan 综合扫描） |
| 宿主机代理 | `apply_resource_plan` / `validate_resource_plan` | §3.9 的幂等 reconcile / 纯校验；写 `/etc/virt-platform/resource-plan.json` |
| 宿主机代理 | `tiering_host_status` | 总控快照：压力合成值、水位线、每 VM 下发档位（供 §4.2 聚合端点） |
| 虚拟机监控服务 | `launch_vm` 扩展 + `vm_pin_vcpus` / `vm_unpin_vcpus` | scope 放置（§3.3）；运行中改独占绑定（query-cpus-fast + setaffinity + scope cpuset 更新） |
| 宿主机代理 | `placement_status` / `placement_rebalance` / `placement_advise` | §3.10 控制器：状态快照 / 手动触发一拍 / 只出建议；执行层走 systemd D-Bus 改 scope 属性（AllowedCPUs / AllowedMemoryNodes / CPUWeight），进度读 `numa_maps` |

### 4.4 校验规则（validate 与 apply 共用）

- 池两两不交、并集 = 全部在线 CPU；HT 兄弟同池；system ≥ 2C、virt ≥ 2C。
- 缩池冲突：目标池变更后，受影响核上有运行中 VM 的独占绑定 → 列出阻塞 VM 清单拒绝
  （需先解绑/迁移/关机）。
- spdk 池变更时校验 isolated 前提：扫根下全部子 cgroup（含临时 scope / 容器运行时的组）
  的 `cpuset.cpus` 与 spdk 池无重叠；apply 后回读 `cpuset.cpus.partition`，非 `isolated`
  即失败并原样上报原因字符串。
- 大页：目标值 < 当前 `nr - free`（在用）→ 拒绝；1G 增量运行时分配失败 → 降级为
  pending-reboot 而非失败。
- 内存预算不等式（§3.4）成立且安全余量 ≥ 4%（残差按观测峰值）；`machine.slice`
  的 `memory.max` 由校验器推导、不可手填；VM 池预算低于运行中 VM 保证线 → 拒绝，
  低于超分线 → 警告；模式 C VM 只以 overhead 计入 VM 池、主内存计入大页池。
- VM 放置：pins ⊆ 所属池；模式互斥矩阵（§2.3）；模式 C 的节点大页余量足额。
- 浮动 VM 与共享池：共享池可用线程（扣独占）为 0 而有运行中浮动 VM → 拒绝；
  可用线程 < 任一浮动 VM 的 vCPU 数 → 拒绝；Σ浮动 vCPU / 可用线程 > 4 → 警告；
  某节点共享池核为 0 → 警告（该节点上的浮动内存将跨节点访问）；
  绑定单节点的共享 VM 所在节点共享池核为 0 → 拒绝。
- 自动放置（§3.10）：控制器只动 `placement_source=auto` 的 VM；vfio / 1G 大页 / vNUMA
  VM 强制 manual；换节点前目标节点 `free_mem ≥ resident×1.1`（2M 大页 VM 另查目标节点
  `free_hugepages`）；主机同一时刻最多一个内存迁移。plan 编辑缩池时，控制器先暂停，
  应用完成后下一拍按新池重新平衡（不由 validate 拒绝）。

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
│    逐 VM 常驻/换出条 + 优先级；后端盘水位     │   内存分层徽标                │
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
- 图例下方一条「浮动 VM」汇总条：台数 · Σ vCPU ↔ 共享池可用线程（共享核数 − 独占核数）
  · 超分比（≤2 绿 / ≤4 黄 / >4 红）· VM 名单。浮动 VM 没有可指向的核，只能以这条
  汇总回答「没做预留的 VM 跑在哪、挤不挤」。右栏 VM 列表用虚线标签区分「浮动」，
  NUMA 列显示「不绑 NUMA」。
- 右栏「自动放置」卡（§3.10）：总开关 + 模式（应用 / 仅建议）+ 关键阈值（独占超分
  上限、释放超分、NUMA 失衡 %）+ 当前每节点 oc / 空闲独占核 / 空闲内存 + 进行中迁移
  的进度条（VM、目标节点、已迁 %）+ 最近决策列表（时间 · VM · A→B / N0→N1 · 原因）。
  VM 列表每行加 tier 徽标（A/B/C）与来源（自动 / 🔒手动）；VM 视图顶部提示「由控制器
  自动放置为 X 级」+「锁定为手动」开关，手动改任何放置字段自动变为 manual。

### 5.3 内存与大页板 · 内存分层板

**内存条**（每 NUMA 节点一条，编辑模式段间竖条手柄可拖，段宽即额度）：

```
[内核·其它 6G][OS 4G‖][虚拟化 8G‖][大页2M 16G‖][大页1G 16G ⟳‖][VM池 已用41G/预算58G][空闲·缓存 20G]
  ‖ = 可拖拽手柄     ⟳ = 该段变更需重启生效
  附注行：大页在用 9.2G（VM×3）· SPDK 圈占 1G×4 · 安全余量 ≥ 4%（校验器强制）
```

**内存分层板**（数据源：内存分层查询接口 聚合 + 总控状态）：

```
主机内存压力 0.32 · 当前档位：下刷中
[■■■■■■■─────────|──────────────────|───────]   ← 压力计；两 tick = 下刷线10% / 回血线50%（编辑态可拖）
ai-train-02  [low]    [常驻▓▓░░░░░░░░换出(斜纹)]   常驻 0.8G · 换出 3.2G
oa-file-01   [normal] [常驻▓▓▓▓▓░░░░换出]          常驻 4.4G · 换出 3.6G
db-core-01   [high]   [常驻▓▓▓▓▓▓▓▓▓▓]            常驻 15.4G · 换出 0.6G
后端盘 nvme1n1p3 [▓▓▓░░░░░░░]                      7.4G / 24G · 换出 210 MB/s
```

- 「换出」段代表**不占宿主 RAM** 的部分——VM 池分段条上以斜纹叠加显示
  「内存分层借出」，让内存恒等式在视觉上闭合。
- 每行点开 = VM 内存分层详情（现有 TieringCard 数据 + swapin-faults/evict-aborts
  曲线）；优先级 chip 直接下拉改（走既有 内存分层优先级 API）。

### 5.4 操作逻辑

编辑态应用栏（工具栏右侧常驻）：`3 项变更 · [2 即时生效][1 需重启] · 放弃 | 校验 | 应用`

| 拖拽 | 手势 | 效果与约束 |
|---|---|---|
| 核 → 池 | 拖物理核（HT 成对）到图例池 chip；或框选多核后点池 chip | 草稿中重着色为斜纹待定态；缩池撞独占绑定时高亮阻塞 VM 并拒绝落点 |
| VM → 核 | 从右栏 VM 行拖到 CPU 板 | 悬停时同 NUMA 节点的合法核亮起幽灵框；落下 = 该 VM 转独占 + 绑定这些核；浮动 VM 被拖上来时内存随之改绑该节点（标「下次启动」）；跨节点落点显示警告（破坏 NUMA 亲和） |
| 内存段手柄 | 横向拖竖条 grip | 吸附步进：预留 256M、2M 大页 2M、1G 大页 1G；右栏输入框同步（拖拽给趋势、输入给精确值）；1G 段即时标 ⟳ |
| 水位线刻度 | 拖压力计上的两个 tick | 下刷线恒 < 回血线（互相钳制） |

1. **查看模式**（默认）：5s 轮询指标刷新利用率叠加；一切只读。
2. **进入编辑**：快照当前 plan 为草稿（本地 state）；实时数据继续刷新但不覆盖草稿；
   其它人改动（version 变化）顶部横幅提示。
3. **编辑**：拖拽/输入 → 右栏变更清单逐条累积，每条带生效性徽标
   「即时 / 需 VM 重启 / 需主机重启」与撤销按钮；客户端实时跑轻量校验
   （池覆盖/预算不等式/互斥矩阵）。
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
  虚拟化系统 ▸ 展开：管理服务 1.2% · 宿主机代理 0.8% · 虚拟机监控 0.4% · 数据库 3.1% · Web 网关 0.6% · 分布式存储 2.9%
  虚拟机     ▸ 展开：逐 VM scope（cpu.stat / memory.current / PSI）
```

| 展示项 | 计算口径 |
|---|---|
| 内核 | `/proc/stat` 全机 − Σ 顶层 slice `cpu.stat`（残差法）；内存 = meminfo 减各 slice current 与大页后的残差 |
| 系统服务 | `system.slice` + `user.slice` + `init.scope` 的 cpu.stat / memory.current |
| 虚拟化系统 | `virt.slice` 汇总，展开到 service 级 |
| 虚拟机 | `machine.slice` 汇总，展开到 per-VM scope（这正是 §3.3 改造后才可能拿到的口径） |

### 5.6 前端实现要点

- **零新依赖**：拓扑网格/分段条/压力计全部 CSS + 少量 SVG，无图表库；拖拽用原生
  Pointer Events（控制台浮窗已有同款手写拖拽先例），不引 dnd 库。
- **新组件**：「资源规划」tab 组件挂 host 详情页；子组件
  `CpuTopologyBoard / MemoryBoard / TieringBoard / SpdkCard / PlanInspector /
  VmPlacementList`；草稿与 diff 逻辑仿 `diffVmConfig` 模式（orig 快照 + working
  state + 提交时差异）。
- **i18n**：全部文案走 `useI18n` 键（zh/en 字典同步补齐），不再新增硬编码中文。
- **降级**：无 plan（老 agent / 未初始化）→ 只读拓扑 + 「初始化资源规划」引导按钮
  （生成默认池：全部 vm-shared + 最小 OS/virt 池建议值）。

---

## §6 指标采集链路

复用既有链路：`宿主机代理 exporter → 采集代理 → 时序库 →
管理服务 /api/vcenter/metrics（PromQL 白名单）→ 前端`。本方案只增 collector 与
白名单前缀；plan/topology 等配置态数据走 §4.2 的 REST，不进时序库。

| 新 collector（宿主机代理） | 指标（示例） | 源 |
|---|---|---|
| `cpu_pool` collector | `xvirt_cpu_busy_ratio{cpu,pool,node}` · `xvirt_pool_cpu_ratio{pool}` | `/proc/stat` 逐 CPU + plan 归属映射 |
| `slice` collector | `xvirt_slice_cpu_seconds_total / memory_bytes / psi{slice,service}` | 顶层 slice 与 service 的 `cpu.stat / memory.current / *.pressure` |
| `numa` collector | `xvirt_node_mem_bytes{node,kind}` · `xvirt_hugepages{node,size,state}` | node meminfo + hugepages sysfs |
| `vm_direct` collector | `xvirt_vm_cpu_usage_ratio / memory_used_bytes / cpu_pressure_*{vm}` | per-VM scope cgroup（即 `vm-direct-metrics-design.md` 的落地，§3.3 是它的前置） |
| `tiering` collector | `xvirt_vm_tiering_{resident,swapped,backing_used}_bytes / evict_mbps{vm}` | 虚拟机监控服务的内存分层查询（总控每拍已在查，复用同一份缓存避免双倍 QMP） |

管理服务指标代理的 L1 白名单补 `xvirt_cpu_ / xvirt_pool_ / xvirt_slice_ /
xvirt_node_ / xvirt_vm_tiering_` 前缀。页面查询频率 5s，与既有 VMMonitor 一致。

---

## §7 分期实施

| 阶段 | 交付 | 动到的组件 |
|---|---|---|
| **P0 · 看得见** | 拓扑扫描落库；cpu_pool/slice/numa/tiering collectors；「资源规划」tab 只读版（拓扑板 + 内存板 + 内存分层板 + 归户条，池归属显示为「未规划=全部共享」） | 宿主机代理（scan + collectors）、管理服务（topology API + 白名单）、Web 控制台 |
| **P1 · 池化落地** | plan 模型/REST/本地缓存/开机对账；slice AllowedCPUs + memory.min/high；**per-VM scope 改造**；编辑模式（核→池拖拽、内存段手柄、2M 大页）+ validate + 异步应用 | 管理服务（plan/validate/task）、宿主机代理（apply/reconcile + 单元）、虚拟机监控服务（scope 启动）、Web 控制台（编辑器） |
| **P2 · VM 落位** | vm_config 放置列 + PATCH 扩展；独占绑定（VM→核拖拽 + query-cpus-fast pin）；单节点自动放置 + 显式 vNUMA；模式 C 大页后端；vm_direct collector | 管理服务、虚拟机监控服务（backend/pin）、宿主机代理（config 传递）、前端（VM 放置 UI + vNUMA 图） |
| **P2.5 · 自动放置** | §3.10 放置控制器：启动准入 A/B/C + 再平衡（解绑/绑核/换节点/收编）+ 守卫；scope 属性热改 + `numa_maps` 进度；`placement_event` 审计；先以 `mode=advise` 上线观察再切 `apply`；验证项 #1–#4 | 宿主机代理（controller + D-Bus 执行层）、虚拟机监控服务（pin/unpin）、管理服务（plan 字段 / 事件 / REST）、前端（自动放置卡 + tier 徽标 + 锁定） |
| **P3 · SPDK** | spdk.slice + isolated 分区 + IRQ 驱离 + hugetlb 圈占 + SPDK 板；可选低抖动启动档（主机初始化 grub drop-in） | 宿主机代理、主机初始化、前端 |
| **P4 · 内存分层融合** | 总控读 plan 水位线/预算（替换环境变量常量）；VM 池预算项进压力合成；可选热集 memory.min 同步；水位线 UI 编辑 | 宿主机代理（总控模块）、管理服务、前端 |

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
  cpuset；6.7 内核（v6.2+ `user_cpus_ptr`）在 cpuset 变更时会自动取交集、交集为空
  回退 cpuset 全集，但 `sched_setaffinity` 调用本身遇到与当前 cpuset 交集为空的 mask
  仍返回 EINVAL——换节点必须「解 pin → 改 cpuset → 重 pin」（§3.10.5）；扩回池后 pin
  是否自动恢复要实测（§3.10.6 验证项 #4）。
- **新 VM 必须在 exec 前进入 scope**：先起 QEMU 再搬 pid 会让 prealloc 的内存与
  大页永久计费到虚拟机监控服务的 cgroup（§3.3）——`hugetlb.*.max` 对这台 VM 形同虚设。
  代码评审把这条当硬性检查项。
- **isolated 分区写入失败的表现**：6.7 上约束不满足时写入**不返回 EINVAL**，写成功
  但状态变为 `isolated invalid (<原因>)`，原因字符串直接可读（如 `Cpu list in
  cpuset.cpus not exclusive`）——agent 回读解析上报即可，不必逐 slice 猜冲突。冲突
  只看兄弟 cgroup 已配置的 `cpuset.cpus`，对账时扫根下全部子 cgroup（§3.2）。
- **1G 大页运行时分配**：开机越久成功率越低；已按「pending-reboot 降级 + 开机早期
  投放」设计，不要在 apply 里无限重试 compact。
- **memory.min 过配**：Σ min 逼近 MemTotal 会把全局回收逼到无页可扫、触发 OOM——
  预算不等式的 4% 安全余量是硬校验，不给 UI 绕过入口。
- **VM 池 memory.high 节流的连带效应**：顶到 high 后 QEMU 线程会被 reclaim 节流，
  vCPU 出现毛刺——内存分层水位线应设在 high 之前触发（预算项进压力合成正是为此，
  §3.7）。
- **vhost-net/内核线程不可归户**：其 CPU 落「内核/其它」段；重 IO 的 VM 会让该段
  偏大，口径说明要写进 tooltip，避免被当成泄漏排查。
- **systemd 与手写 cgroupfs 的共管边界**：AllowedCPUs 走 systemd（防 daemon-reload
  覆盖）；partition/hugetlb 上限走直写 + 开机单元重放；不要两边写同一属性。
- **单 NUMA 开发机 vs 多 NUMA 生产机**：所有 per-node 逻辑必须在 node 数=1 时退化
  正确；autotest 用例要在两种拓扑各跑一轮。
- **分布式存储 OSD 的突发 CPU**：recovery/rebalance 时 virt 池可能被打满，影响管理面
  响应——virt 池规模建议随「是否跑 OSD」联动，页面在 virt 池 PSI 高时给提示。
- **自动放置抖动（§3.10）**：阈值太近会让 VM 在 A/B 间反复升降、或两节点来回搬——
  升降各自独立阈值（1.0 / 1.5）+ 稳定期 + 每 VM 冷却 + 每拍一件事是硬性守卫；上线
  先 `advise` 模式看一周决策日志再切 `apply`。
- **cpuset.mems 迁页是异步且不可取消的**：写下去内核 workqueue 就开始搬，中途改主意
  只能再写回原节点（等于再搬一次）——控制器对「进行中」的 VM 不再下发任何新决策，
  超时也只标「部分迁移」不回滚。
- **迁页期间的 guest 毛刺**：每页一次 EPT 重建 + 迁页占用内存带宽；对延迟敏感 VM
  应设 `placement_source=manual` 锁死；`max_move_gb` 限制单次搬动规模。
- **内核自动 NUMA 平衡与控制器打架**：主机 `kernel.numa_balancing=0`（主机初始化
  sysctl drop-in），否则内核会按页扫描把控制器刚搬好的页再挪走并制造 EPT 抖动。
- **vfio / 1G 大页 / vNUMA VM 不可自动搬**：pin 页不能迁、1G 需目标有空闲页且慢、
  vNUMA 缺 per-memdev 迁移原语（待 `x-memdev-rebind`）——统一强制 manual，UI 标明原因。

---

## §9 实现落点（2026-09-11，P0 + P1 主体，含 P2/P3/P4 的基础部分）

| 组件 | 文件 | 内容 |
|---|---|---|
| 宿主机代理 | `hyper-agent/hyper_agent/support/resource_plan/topology.py` | sysfs/cgroupfs 扫描：逐 CPU 拓扑、HT 配对、NUMA 内存与大页、各 slice 实际 cpuset/memory.*、分区状态、根下全部子 cgroup cpuset、内核残差 |
| 宿主机代理 | `.../resource_plan/plan.py` | plan 默认生成（小机器退化）、normalize、§4.4 校验规则（纯函数）、预算不等式、diff |
| 宿主机代理 | `.../resource_plan/apply.py` | 幂等 reconcile：`systemctl set-property` 下 AllowedCPUs/AllowedMemoryNodes/MemoryMin\|High\|Max；大页 per-node sysfs（2M compact 重试、1G 降级 pending_reboot）；`+hugetlb` 与 hugetlb.*.max 圈占；spdk.slice isolated 分区 + IRQ/workqueue 驱离 + irqbalance drop-in；`kernel.numa_balancing=0`；平台服务 `Slice=virt.slice` drop-in（pending_restart）；本地缓存 `/etc/xvirt/resource-plan.json`；drift 对比 |
| 宿主机代理 | `.../resource_plan/cli.py` + `etc/systemd/xvirt-resource-plan.service` | 开机早期单元（`Before=sysinit.target`）重放缓存，大页优先；`xvirt-resource-plan status/topology/validate/default` |
| 宿主机代理 | `hyper_agent/controller/resource_plan.py` | ZMQ actions：`scan_cpu_topology` / `get_resource_plan` / `validate_resource_plan` / `apply_resource_plan` / `tiering_host_status`；apply 后发 `host.resource_plan_applied` 事件并让内存分层总控重读水位线 |
| 宿主机代理 | `metrics_exporter/collectors/resource_plan.py` | `xvirt_cpu_busy_ratio{cpu,pool,node}`、`xvirt_pool_*`、`xvirt_slice_*`（含 virt.slice 下逐 service）、`xvirt_node_mem_bytes`、`xvirt_hugepages*`、`xvirt_vm_tiering_*`、`xvirt_host_tiering_*` |
| 宿主机代理 | `support/memtier/controller.py` | P4：水位线/PSI 满刻度/VM 池预算从 plan 缓存读；VM 池挤占度并入压力合成；每拍快照供 collector 与 `tiering_host_status` 复用（不双倍 QMP） |
| 宿主机代理 | `vm/guest.py`、`shell.py`、`controller/vm.py`、`setup.cfg` | `placement.json` sidecar（create/update）；controller 注册 + 启动对账线程；`vm_pin_vcpus`/`vm_unpin_vcpus`/`vm_placement_status` 转发；包与 console_script |
| 虚拟机监控服务 | `lictor/support/placement.py` | scope 规格（浮动挂 `machine-shared.slice`，绑定/独占挂 `machine.slice` 并显式 AllowedCPUs/AllowedMemoryNodes/CPUWeight/MemoryMin）；`systemd-run --scope` 前缀；`memory.oom.group=1`；`query-cpus-fast` + `sched_setaffinity` 绑核/解绑（顺序守 §3.10.4）；运行时 `set-property --runtime` 改 scope；存量 QEMU 经 D-Bus `StartTransientUnit(PIDs)` 迁入 scope；`numa_maps` 聚合 |
| 虚拟机监控服务 | `lictor/vm/guest.py`、`utils/process.py`、`operation_server.py`、`controller/vm.py` | launch 在 exec 前入 scope（失败降级直起）；启动后 oom.group + 绑核；live op `placement_set`；启动时 `reconcile_scopes`；三个新 action |
| 虚拟机监控服务 | `lictor/support/qemu_command.py` | `mem_backend`：memfd（prealloc=on）/ hugepage-2m / hugepage-1g（memfd hugetlb）；`numa_mode=vnuma` 逐 vnode memdev + `-numa node` |
| 管理服务 | `ferrovirt/src/service/resource_plan.rs` | 表 `host_resource_plan` / `host_cpu_topology` / `host_numa_node`，`vm_config` 放置列；拓扑落库与 DB 兜底；plan 读（DB 权威 + agent 实况 drift/预算）；validate；PUT → version 乐观锁（409）→ 前置校验 → 任务 `applyResourcePlan`；init；内存分层聚合；VM placement 校验/写列/回放 |
| 管理服务 | `src/controller/vcenter/resource_plan.rs` | `GET/POST …/resource-topology`、`GET/PUT/POST …/resource-plan?action=validate\|init`、`GET …/memory-tiering`、`GET /vm/{vm}/placement` |
| 管理服务 | `vm_update.rs` / `vm_create.rs` / `read/vm.rs` / `storage_net.rs` / `read/metrics.rs` / `error.rs` | PATCH `placement` 稀疏段 + 热调 live op；建 VM 带 placement；VM 详情暴露 placement；hardware-scan 顺带拓扑落库；指标 label 白名单；`Conflict` 409 |
| Web 控制台 | `xvirt-client/src/components/host/resplan/*` | `ResourcePlanTab`（查看/编辑、草稿 diff、校验、应用任务跟踪、横幅）、`CpuTopologyBoard`（核→池拖拽/多选、VM→核独占绑定）、`MemoryBoard`（主机级预算条 + 逐节点大页条，Pointer 手柄）、`TieringBoard`（压力计 + 可拖水位线 + 逐 VM 条）、`VmPlacementList` + 放置对话框（PATCH placement）、`planModel.js`（派生/diff/轻量校验） |
| Web 控制台 | `HostDashboardPage.jsx` / `App.jsx` / `management/Console.jsx` / `api/vcenter.js` / `i18n/{zh,en}.js` / `styles/resplan.css` | 两个壳都挂「资源规划」tab；API；全部文案 i18n |

与设计的已知差异（后续补）：内存板的 OS/虚拟化预留是主机级字段，按主机级条拖动，不做逐节点手柄；浮动 VM 的跨节点内存分布在未接入 `numastat` 采集前按节点数均分估算；§3.10 自动放置控制器（P2.5）与 `placement_event` 审计表未实现，`auto_placement` 字段只存储透传；`x-memdev-rebind` 未做，vNUMA VM 只在启动时落位；打包侧需在 `build-deb-xvirt` 为 hyper-agent 加 `xvirt-resource-plan` 入口与单元安装（见 xvirt 仓）。
