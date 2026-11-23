# xvirt 架构梳理

初版：2026-07-18（来源 192.168.60.112:/root/code/xvirt）。  
全面刷新：2026-11-20。2026-11-22 增补 §16.3 kestrel 接入方案（1.0 定 kestrel 为时序库），并按 kairos/main 核对更新 §16.0 与 §19 的监控、compute 角色、xvirt-client 打包条目。  
2026-11-23 §16.3 补 Petrel 覆盖（`petrel` / `petrelpool` / `petrelosd` 类别、Petrel 盘经 petrel-vhost 计数取 IO、Petrel 数据存储容量映射到池、三条 Petrel 预置规则）与 center 的 kestrel 容量估算。  
刷新来源换成 ferrostack 工作区（`/root/work/ferrostack`）下各子仓的代码、README/CLAUDE.md/设计文档，以及 `docs/` 下的跨组件设计稿。本次新增资源规划与 VM 放置（§12）、远程桌面 xdesk（§15）、推送式监控框架 **kestrel**（§16.2，拟替代 VictoriaMetrics + exporter）、用户态共享盘文件系统 **fulmar**（§8.1，拟替代 OCFS2）。

状态用语：**已实测**＝在真机或测试机上跑过；**已实现未验证**＝代码已写，没编译或没真机验证；**仅设计**＝只有设计稿。仓库地址、分支、HEAD 以 `xvirt-repository-index.md` 为准（远端统一为 kairos）。

本次核对的一个限制：子仓的 `git log` 没能逐仓跑，7 月之后的变化是从 reflog、文档日期和代码本身判断的。

## 1. 项目定位

xvirt 是一套自研服务器虚拟化平台，整体目标对标 vSphere / Proxmox。它不是单一服务，而是多仓集成的工作区：上层提供 Web 管理控制台、REST/WS API、资源模型和任务系统；下层直接驱动 QEMU、宿主机存储/网络/硬件扫描、QMP、迁移、快照、备份和监控。

工作区组织（2026-11 起）：

- 顶层仓 **ferrostack**：放跨组件设计文档（`docs/`）和本文这类总览；各组件是 clone 在它下面的独立子仓，各自提交。
- **xvirt** 降为集成仓：组件表 `CLAUDE.md`、`xvirtctl`、nginx、monitoring、`build-deb-xvirt` 打包。
- 所有仓库托管在 kairos（`git@kairos.xve-vt.com:local/<repo>.git`）。

当前形态：

- Rust 控制面 `ferrovirt` 已取代旧 Java `virt-control`，后者已从工作区移除。
- 运行中 QEMU/QMP 的能力集中在 `lictor`；`hyper-agent` 管宿主机准备工作和主机级总控。
- 7 月之后新增了几项跨组件能力：资源规划与自动放置、memtier 一系列增强、内存分层快照、带快照树的跨存储迁移。
- 外围生态还有 `petrel`、`cdp`、`aegis-sdk`、`cdp-agent` 和安装部署链（lycan、fledge、build-deb）。

## 2. 总体拓扑

```text
浏览器
  | HTTP / WebSocket
  v
nginx :80/:443
  |-- 静态 xvirt-client SPA
  |-- /api/*、/websocket/* -> ferrovirt :8080
  |-- /vnc/<port>/          -> websockify / noVNC
  v
ferrovirt（center，role=management）
  |  REST / WS / TaskManager / MariaDB(mx)
  |  C1 DEALER -> 节点 :5570（enrollment / sync 命令）
  |  C2 SUB    <- 节点 :5571（资源状态、任务进度、节点健康事件）
  v
ferrovirt（每台主机，role=standalone，本机 MariaDB）
  |  ZMQ DEALER -> hyper-agent ROUTER :5556
  |  ZMQ SUB    <- hyper-agent PUB    :5558
  |  周期状态对账（默认 60s）
  v
hyper-agent
  |  宿主机准备：存储、网络、硬件扫描、文件传输、petrel 托管
  |  主机级总控：资源规划 apply、自动放置控制器、memtier 水位与写入预算
  |  VM 启停：已被 concilium 纳管 -> concilium，否则直连 lictor
  |  其余 VM/QMP/xvirt-tools 类 action 瘦代理到 lictor
  v
lictor
  |  QEMU/QMP 唯一持有者：启动、停止、快照、迁移、热插拔、console、xvirt-tools
  |  QEMU 经 systemd-run --scope 放进 machine*.slice/vm-<uuid>.scope，按放置绑核
  |  PUB :5561 -> hyper-agent relay -> PUB :5558
  v
QEMU 11.1 fork（stable-11.1-dev）+ host kernel
  |
  | virtio-serial channel "xvirt-tools.0"
  v
guest 内 xvirt-tools

旁路/扩展：
concilium 集群资源管理（gossip :7700）-> lictor CLI（启停；迁移 RA 仍是桩）
petrel 分布式块存储，由 hyper-agent 托管守护进程
fulmar 用户态共享盘文件系统（规划中，替代 OCFS2）：每主机一个 fulmard，
       hyper-agent 经 unix socket 控制，QEMU 经 libfulmar 共享内存环读写
cdp（recorder + cdplog）/ cdp-agent / aegis-sdk 备份与 CDP（控制面尚未编排 CDP）
VictoriaMetrics / vmagent / ebpf_exporter / Grafana 监控（现行；deb 包里仍是 Prometheus，见 §16）
kestrel 推送式监控（规划中，替代上一行）：各服务内嵌 SDK 推到 kestreld :7410，
        ferrovirt / 界面经 HTTP :7411 查询
xdesk 远程桌面（QEMU -display xdesk + 独立客户端，未接入控制面）
lycan / fledge / build-deb / build-deb-xvirt / xvirt-autotest 安装、打包与自动化测试
```

角色由 `/etc/xvirt/role.conf` 决定（`xvirt-init` 首启时读）：

- `standalone`（默认）：管理面加计算面都装。
- `management`：只有 MariaDB、nginx、ferrovirt，即 center。
- `compute`：openvswitch、xvirt-resource-plan、lictor、hyper-agent、conciliumd，**不启 ferrovirt**。

center 纳管主机时连的是主机上的 ferrovirt（5570/5571），所以实际纳管的节点用的是 standalone 角色，纯 compute 角色与这套设计不一致（见 §19）。

## 3. 核心组件

| 组件 | 技术栈 | 角色 | 主要入口 |
|---|---|---|---|
| xvirt-client | React 18 / Vite | Web 控制台，center 多主机管理与 standalone 单机两套界面；主机详情有"资源规划"页 | nginx 静态托管，开发时 :5173 |
| ferrovirt | Rust / axum / sqlx / ZMQ | 管理控制面：REST、WebSocket、任务系统、DB 唯一写入者、资源规划与放置的权威、迁移编排 | :8080；center↔节点 :5570/:5571 |
| hyper-agent | Python asyncio / pyzmq | 每台宿主机的底层 agent：存储、网络、硬件、文件、petrel、资源规划 apply、放置控制器、memtier 总控 | ROUTER :5556，PUB :5558，file :5557 |
| lictor | Python asyncio / pyzmq | QEMU/QMP 持有者：VM 生命周期、快照、迁移、热插拔、xvirt-tools；systemd scope 与绑核 | ROUTER :5560，PUB :5561 |
| concilium | C daemon | 集群资源管理：选主、HA 故障转移、self-fence + IPMI STONITH，驱动 `xvirt-vm` RA 到 lictor | `/run/conciliumd.sock`，gossip :7700 |
| petrel | Rust | 分布式块存储（副本 / EC、按设备类分池），块卷经 vhost-user-blk 或 ublk 给 VM；由 hyper-agent reconcile | mon :6790，osd :6800–6820 |
| fulmar | Rust（libfulmar 为 C ABI） | 用户态共享盘文件系统，替代 OCFS2 做共享 datastore；COW + O(1) reflink 快照，本地/集群同格式 | `fulmard` 控制 socket，`fulmar:` 块驱动；见 §8.1 |
| QEMU | C，自研 fork，基线 11.1.1 | 虚拟化底座：memtier、CDP 采集、CBT 位图导出/导入、热度 pre-copy 迁移、qcow2 在线缩盘、xdesk 显示后端 | 分支 `stable-11.1-dev` |
| xvirt-tools | Go | guest VM 内 daemon，经 virtio-serial 与 host 通信 | guest 内服务 |
| aegis-sdk | C library + CLI | 类 VDDK 的备份 SDK，提供磁盘读写和 CBT | `libaegisdisk.so`、`aegis-cbt` |
| cdp | C + Rust | 现行 CDP：`cdp-recorder` 记录 QEMU 侧采集的写，`cdplog` 日志引擎（ublk / nbdkit / 搬运服务） | 见 §14 |
| cdp-agent | Rust + C 内核驱动 | guest 内 CDP，拦截写 IO 流向 cdp-sink；不依赖 QEMU | agent -> sink TCP |
| monitoring | VictoriaMetrics 生态 | 宿主机/VM/数据存储指标、告警、Grafana（现行） | vmagent :8429，vmsingle :8428 |
| kestrel | Rust 服务端 + Rust/Python/C SDK | 推送式轻量监控，拟替代上一行；对象 + 指标集模型，分层精简，保留突发 | 推送 :7410，查询 :7411；见 §16.2 |

## 4. ferrovirt 控制面

`ferrovirt` 是主控制面，拥有 MariaDB `mx` 数据库，agent 不直接碰 DB。模块按 controller / service / entity / repository 分层：

- `controller/*`：account、aegis、alert、auth、cis、datastore、host、internal、network、sys、task、vm。`controller/vcenter/*` 是 vCenter 风格资源：cluster、datacenter、datastore、enrollment、host、metrics、mgmt、network、petrel、resource_metrics、resource_plan、resource_pool、snapshot、vm。
- `service/task_manager.rs`：异步任务管理。写操作、扫描、电源、创建/编辑、快照、资源规划下发（`applyResourcePlan`）都走任务。
- `service/vm_create.rs`、`vm_update.rs`、`snapshot.rs`、`vm_migrate.rs`、`snapshot_migrate.rs`：VM 创建、编辑、快照（含内存分层快照）、在线迁移、带快照树的跨存储迁移。
- `service/resource_plan.rs`：资源规划与放置（见 §12）。
- `service/host_sync.rs`、`storage_net.rs`、`host_manager.rs`、`state_reconcile.rs`：主机扫描、存储/网络同步、主机纳管、周期状态对账。
- `service/enrollment.rs`、`mgmt.rs`、`src/node_zmq.rs`：center 与节点之间的纳管和同步通道（C1 命令 :5570，C2 事件 :5571）。
- `service/cluster_concilium.rs`、`cluster_admin.rs`、`datacenter_admin.rs`：集群与数据中心管理。concilium 多节点配置由 ferrovirt 下发，经 hyper-agent `concilium_apply` 写 `/etc/concilium/cluster.conf`。
- `service/alert.rs`、`websocket.rs`、`service/event.rs`：告警、WebSocket 推送、agent 事件处理。
- `auth.rs`、`jwt.rs`、`service/auth.rs`：PAM 登录 + JWT。

关键设计点：

- DB 归属清晰：ferrovirt 是资源状态、任务、用户偏好、快照元数据、资源规划的唯一写入者。主机上的 `/etc/xvirt/resource-plan.json` 只是缓存副本。
- API 返回与路径变量统一使用 UUID，`name` 只用于展示。
- 写操作走 TaskManager，任务状态经 WebSocket 推送，前端也有轮询兜底。
- 实时状态靠 agent 事件经 ZMQ PUB/SUB 单向推送，topic 包括 `vm.state_changed`、`vm.placement_changed`、`vm.guest.state`、`alert.raised`、`alert.cleared`。
- 事件丢失时由**周期状态对账**兜底：连上 host 时跑一次，之后按 `local_host.refresh_seconds`（默认 60s，下限 15s）主动查 VM 电源和 datastore 挂载状态并纠正 DB（`service/state_reconcile.rs`）。
- 在线迁移由 ferrovirt 直接编排：`migrate_export → migrate_incoming → migrate_out → migrate_finish/abort`，不经 concilium。
- 指标查询 `/api/vcenter/metrics`、`resource_metrics` 现在走 PromQL（白名单标签），切 kestrel 时要改。

## 5. hyper-agent 宿主机 agent

`hyper-agent` 是每台宿主机上的底层服务，监听 ZMQ ROUTER :5556 接收 ferrovirt 的 action 请求。主循环基于 `zmq.asyncio`，阻塞型 handler 放到 ThreadPoolExecutor 里执行。

职责边界：

- **保留**：存储池、数据存储、网络、设备树扫描、单盘 wipe/rescan、文件传输、监控 exporter、petrel 守护进程托管、Aegis NBD 本地导出、带快照树的跨存储迁移（`snapshot_chain_migrate`）。
- **主机级总控**：资源规划的扫描/校验/apply、开机对账、自动放置控制器、memtier 水位线与 NVMe 写入预算（见 §12、§13）。
- **转发**：VM 启停在已被 concilium 纳管时走 concilium，否则直连 lictor；其余 VM 生命周期、运行中 QMP、快照、迁移、console、xvirt-tools、放置类 action（`vm_pin_vcpus`、`vm_unpin_vcpus`、`vm_apply_placement`、`vm_placement_status`）瘦代理到 lictor。
- **不做**：不调用 libvirt；不读写 MariaDB；不做上层资源决策。

主要模块：

- `controller/{host,vm,storage,network,device,petrel,memtier,aegis,snapshot,xvirt_tools,concilium,resource_plan}.py`
- `support/datastore/*`：local（含 OCFS2）、shared fs、zfs、nfs、iscsi、lvm、petrel 等数据存储适配。
- `support/resource_plan/{topology,plan,apply,bootprofile,rdt,placement,cli}.py`：资源规划与放置。
- `support/memtier/*`（含 `write_budget.py`）、`support/memtier_backing.py`：memtier 总控与共享后端。
- `support/petrel/*`、`support/petrel_chain.py`：petrel 集群、supervisor、vhost、卷管理，以及 petrel 快照链互迁。
- `support/{device_scanner,nic_scanner,storage_device_scanner,storage_adapter_scanner}.py`
- `concilium_client.py`、`support/concilium_cluster.py`、`support/alert_monitor.py`
- `metrics_exporter/collectors/{host,vm,datastore,zfs,resource_plan}.py`
- `event_publisher.py`：向 ferrovirt 发布 agent 事件。

后台线程：`resource-plan-reconcile`（开机兜底的幂等对账）、`placement-controller`（自动放置，默认 30s 一拍，可用 `XVIRT_PLACEMENT_INTERVAL` 调整）。

## 6. lictor QEMU/QMP 层

`lictor` 把"直接管理 QEMU 虚拟机"的职责从 hyper-agent 中独立出来。它用与 hyper-agent 类似的 ZMQ 服务框架，语义上更像轻量 libvirt：持有 QMP、QEMU 运行时和 xvirt-tools socket，提供原子 VM 原语和复合操作。注册的 controller 只有 Vm、XvirtTools、Snapshot、Aegis 四个。

核心职责：

- VM start / stop / reset / status / power 操作。
- 在线快照、reflink 快照、CBT 在线导出、快照删除和回滚；内存分层快照（只导出常驻页，加后端 reflink 副本）及 keep / merge 恢复。
- 在线迁移各步骤，以及存储迁移（`storage_mirror_*`）。
- 运行中设备热插拔：磁盘、网卡、CD-ROM、CPU、内存、网卡 link、放置（`placement_set`）。
- 放置执行（`support/placement.py`）：
  - QEMU 在 exec 前经 `systemd-run --scope` 放进 `vm-<uuid>.scope`。浮动 VM 挂 `machine-shared.slice`，绑定或独占的 VM 挂 `machine.slice`。
  - 独占 VM 用 `query-cpus-fast` + `sched_setaffinity` 做 1:1 绑核。
  - 读 VM 目录下的 `placement.json` sidecar，决定内存后端（anon / memfd / hugepage-2m / hugepage-1g）和 vNUMA。
  - 没有 systemd-run 时降级为不带 scope 直接启动。
- QMP 异步事件监听，发布到 :5561，由 hyper-agent relay 到 :5558。
- 管理 virtio-serial channel 下的 xvirt-tools。

设计边界：

- ferrovirt 只连 hyper-agent，协议面不直接感知 lictor。
- concilium 可通过 lictor CLI 驱动 start / stop / status。
- lictor 是 QMP 唯一持有者，避免多个组件同时控制同一个 QEMU monitor。
- QEMU 进程现在挂在独立的 scope 里，不在 lictor 的 cgroup 下。原来靠 `KillMode=process` 防止"重启 lictor 连带杀 VM"，现在从结构上解决了（推论，未实测）。

## 7. concilium 集群层

`concilium` 是 C 写的集群资源管理守护进程，负责集群决策与资源动作触发：

- 多节点之间用 gossip（:7700）通信，选主后集中调度。
- HA 故障转移，self-fence 加 IPMI STONITH；可选 corosync 后端。
- 单节点可用 `--bootstrap` 自成 quorum，单机也能走统一路径。
- 驱动 `xvirt-vm` RA，最终通过 lictor CLI 执行 VM 启停和状态检查。
- 多节点配置由 ferrovirt `cluster_concilium.rs` 编排、hyper-agent 落盘。

职责刻意收窄。编辑 VM、快照、console、xvirt-tools 都不经 concilium。**跨主机迁移目前也不经 concilium**：RA 的 migrate 调 `lictor migrate`，而 lictor 没有实现 `migrate_vm`，RA 注释写明"P4 前诚实失败"；migrate / promote / demote RA 都是桩。在线迁移实际由 ferrovirt 编排（§4）。

## 8. 存储体系

xvirt 同时支持传统本地/共享数据存储和自研存储：

- **local / xfs / ext4 / btrfs**：hyper-agent 创建目录与文件，VM 磁盘通常是 qcow2/raw。
- **zfs**：agent 调用 zfs 工具创建 dataset。
- **OCFS2**：现行的共享 datastore。hyper-agent 用 o2cb 单节点集群 `ocfs2sn` 格式化挂载（`support/datastore/local.py`）。盘是 raw 文件，快照靠 `cp --reflink`，快照差异靠 FIEMAP 比对（`xvirt-blockdiff`）。内核模块由 `ocfs2` 仓按内核版本自编译，这也是要用 fulmar 替换它的原因之一（§8.1）。
- **nfs / gfs2 / iscsi / lvm**：作为数据存储或块设备能力纳入扫描/管理。
- **petrel**：Rust 分布式块存储，已实测。
  - 二进制：mon、osd、vhost、ublk、deploy、manager、agent、snapdiff、bench。
  - 链路：deploy → mon → osd（BlueStore / Mem，PG，副本或 EC）。VM 出口是 vhost-user-blk，主机侧也可以用 ublk 映射成 `/dev/ublkbN`。
  - 卷能力：精简/厚制备、离线和在线 resize、COW 快照与回滚、克隆与 flatten、多池同时挂载。
  - 按设备类分池（CRUSH device-class 影子树，`petrel-mon create-pool --tier hdd|nvme`）。这不是 Ceph 那种 cache-tier。60.100 单机实测 NVMe 池对比 HDD 池：顺序读 2.6 倍，4K 随机读 5 倍。
  - manager / agent 提供 REST 和 Prometheus 指标。
  - Mon Raft 化进行中（增量 1–5 已完成）；"Mon 内嵌进 OSD"仅设计。
  - `petrel-snapdiff` 按 4MiB 对象比较卷快照，输出 `.xvdiff`，与 xvirt-blockdiff 同格式，用于 OCFS2 ↔ petrel 快照链互迁。
- **fulmar（规划中）**：用户态共享盘文件系统，`fs_type=fulmar`，用来替代 OCFS2，见 §8.1。

一个重要的历史约束：`data_store.uuid` 和 `/datastore/<segment>` 里的路径段不一定相同。VM 创建时 agent 需要的 `vm_datastore` 是挂载路径的最后一段，不是实体 uuid。ferrovirt 的服务层会做适配。

### 8.1 fulmar 用户态共享盘文件系统（替代 OCFS2）

设计稿：`docs/fulmar-design.md` v1.1；代码：独立子仓 `fulmar`。

**为什么要换。** 现在的共享 datastore 是内核态 OCFS2（hyper-agent 用 o2cb 单节点集群格式化挂载，盘是 raw 文件，快照靠 `cp --reflink`，差异靠 FIEMAP 比对）。问题有四个：内核模块要按内核版本自编译打包；碎片多了以后 reflink 变慢（要逐个 extent 复制并更新 refcount）；快照后首写按 cluster（通常 1MB）读改写，写放大大；心跳超时会 panic 整机。

**定位。** fulmar 建在共享 LUN / NVMe-oF / 本地 NVMe 之上，对象是少量大文件（VM 盘、memtier 换出后端、内存快照、ISO），不做通用 POSIX。它和 petrel 并存，不替代 petrel：petrel 是多副本 / EC 的分布式块存储，fulmar 是共享磁盘上的文件系统，用哪个由 datastore 类型决定。

```text
hyper-agent ──控制 API（unix socket，JSON）──► fulmard（每主机一个）
                                                 │ IO 线程轮询
QEMU block/fulmar.c ─┐                           │
QEMU memtier 后端    ├─ libfulmar（C ABI）── 共享内存环（SQ/CQ）
                     ┘   或 extent 租约直通（memtier 换入）
qemu-img / 上传 / 备份 ── FUSE 网关（仅管理用途，活盘禁写）
                                                 ▼
                         块后端：io_uring + O_DIRECT 裸盘（默认）｜ SPDK（可选）
                                                 ▼
          盘上：超级块 │ 集群区（心跳槽 / ATS 锁 / 成员表）│ 每节点 journal │ AG×N
```

组成：

| 部件 | 作用 |
|---|---|
| `fulmard` | 每主机一个常驻进程：挂载、元数据与 journal、IO 线程、集群成员 / 心跳 / 锁、QoS、FUSE 网关 |
| `libfulmar` | C ABI 客户端库，QEMU 直接链接；异步读写、flush、discard、write_zeroes、block_status、extent 租约 |
| QEMU `block/fulmar.c`、`memtier/backing-fulmar.c` | 块驱动（`fulmar:<datastore>/<path>`）与 memtier 换出后端 |
| `fulmar` CLI / `mkfs.fulmar` / `fulmar-fsck` | 管理、格式化、离线修复与在线校验；`fulmar diff` 直接输出 `.xvdiff` |

关键设计：

- **用户态、不经页缓存**：数据走 io_uring 直写裸块设备，内核升级不用动；需要再省几微秒时换 SPDK 后端，上层不变。
- **O(1) reflink**：新 inode 指向同一棵 extent B+ 树的根并给根加引用，数据 extent 的引用计数等第一次 COW 时再成批补；reflink 耗时与碎片程度无关。
- **ROW 写，无读放大**：快照后的写只为写到的范围分配新 extent，不读旧数据。extent 记录带 birth generation，热路径判断"是否共享"不用查 refcount 树。
- **所有权分区**：每个 VM 盘任一时刻只属于运行它的节点，空间按 AG 租给节点，热路径没有分布式锁；快照只读，任意节点可同时读。
- **本地 / 集群同格式**：集群区始终在盘上，本地模式只是不开心跳和锁，可以在线切到集群模式。集群模式用盘 + 网双心跳（约 7 秒判死）、ATS 盘上锁、轻量 DLM；故障时**自 fence 停数据路径**（QEMU 收到 EIO 暂停），幸存节点用 SCSI PR 抢占兜底，不重启主机。
- **配合 memtier**：提供 `pinned` 文件类，换入时把块设备 fd 和 extent 映射租给 QEMU 直读，走的是和裸分区后端相同的 io_uring 路径。这样 memtier 换出文件可以放在共享 datastore 上。
- **按盘限速**：令牌桶，参数与 QEMU throttle 同名，控制 API 在线调整。
- **快照链工具兼容**：`fulmar fiemap` 保持 FIEMAP 兼容，`xvirt-blockdiff` / `xvirt-snapmig` / `cdp` 可以不改；`fulmar diff` 跳过两棵树里相同的子树，开销与变化量成正比。

与现有组件的衔接（设计稿 §十三）：

- hyper-agent 新增 `support/datastore/fulmar.py`，并生成 `-blockdev driver=fulmar`。
- QEMU 新增两个驱动文件，迁移时调用所有权交接。
- lictor 的快照和迁移改走 datastore 接口，不再直调 `cp --reflink`。
- `fulmard` 的轮询核和环用大页从资源规划的存储池（现 `spdk` 池）划拨。
- OCFS2 迁移走现有的跨 datastore 快照链迁移或 drive-mirror，OCFS2 只读保留到清空为止。

**当前状态（2026-11-20）**：子仓 `fulmar` main `f76e2fc`，11 个 crate + 4 个可执行 + libfulmar。

- 本地模式的 mkfs / fulmard / 共享内存环 / libfulmar C ABI / CLI / fsck 链路代码已写完，**还没编译、没测试**。
- 集群模式（P3）只有骨架；hyper-agent、QEMU、lictor 一侧都没动，QEMU 里也还没有 `backing-fulmar.c`。
- 分期：P0 本地原型 → P1 memtier + 工具链 + hyper-agent datastore → P2 性能档（vhost-user-blk、SPDK）→ P3 集群 → P4 运维。
- 待拍板六项（设计稿 §十七）：实现语言（骨架已按 Rust 写）、默认 `cow_chunk`、FUSE 是否可写、集群槽数、memtier 直通是否默认、SPDK 是否进本期。

## 9. 网络体系

网络资源模型以 `virtual_switch` 为核心，支持：

- `LINUX_BRIDGE`
- `OVS`
- `SRIOV`
- 分布式交换机（`switch_type='distributed'`，配 `virtual_switch_host` 表记录各主机成员）

另有 port-group 资源。agent 的 `nic_scanner.py` 通过 sysfs、`ovs-vsctl` 等探测网络状态。虚拟交换机记录 backend、uplink、SR-IOV PF/VF 信息。

- network 句柄已按"uuid 优先，回退 db id"解析，路由为 `/api/vcenter/network/{network}`。
- VM 创建时，提交的 network 会被解析为实际 `source_dev`，避免 agent 生成配置时出现空 bridge。
- SR-IOV VF 目前只做扫描同步和计数，**还没接入 VM 创建**。

## 10. VM 生命周期关键链路

### 10.1 创建 VM

```text
xvirt-client -> ferrovirt POST /api/vcenter/vm
  -> TaskManager 创建异步任务
  -> ferrovirt 补全默认配置、分配 alias / PCI / channel / datastore path
  -> resource_plan::validate_placement 校验放置（固定 / 自动，NUMA、内存后端）
  -> ZMQ 调 hyper-agent create_vm
  -> hyper-agent 做离线准备：目录、磁盘、config.xml、placement.json sidecar
  -> ferrovirt 落库 vm / vm_config（含放置列）/ vm_disk / vm_network_interface / vm_video
```

agent 端不补默认值，默认值由控制面负责。落库失败但主机上文件已创建时，返回失败任务，交由人工查看日志并清理目录。

### 10.2 启动 VM

```text
ferrovirt power?action=start
  -> hyper-agent
  -> concilium（VM 已纳管）或直连 lictor
  -> lictor 解析 config.xml + placement.json，生成 QEMU argv（内存后端、vNUMA）
  -> systemd-run --scope 放进 machine*.slice/vm-<uuid>.scope，再 exec QEMU
  -> lictor attach QMP；post_launch 绑核、设 oom_group
  -> lictor 发布 vm.state_changed
  -> hyper-agent relay
  -> ferrovirt 写 DB state + WS 推前端（事件丢失时由周期对账兜底）
```

### 10.3 编辑 VM / 热插拔

VM PATCH 是稀疏 patch，设备列表用 `add/remove/update` 三段。控制面先落 DB，再重新生成 agent 配置；VM 在运行时额外下发 live ops 给 lictor。

已覆盖的 live ops：

- `disk_attach` / `disk_detach`
- `nic_attach` / `nic_detach`
- `cdrom_change`
- `cpu_set`
- `mem_add`
- `nic_set_link`
- `placement_set`（改绑核 / NUMA 节点；换节点按"解 pin → 改 cpuset → 重 pin"，要迁页，不是零成本）

限制主要来自 QEMU/guest ACPI 配合、SATA 热插支持、预留 PCI root port、max_vcpu/max_memory 等启动时条件。

### 10.4 在线迁移

ferrovirt `vm_migrate.rs` 编排：`migrate_export`（源）→ `migrate_incoming`（目标，QEMU `-incoming` 用 TCP 49152–49200）→ `migrate_out` → `migrate_finish` / `migrate_abort`。QEMU 11.1 fork 带按热度的迭代 pre-copy。memtier VM 可以用 `memtier-shared-backing` 迁移能力：后端共享，不搬冷页（QEMU 侧自标 WIP，跨主机实测未确认）。

## 11. 快照、备份与迁移

### 11.1 CBT 与备份

备份数据面围绕"创建快照/freeze → 导出 dirty bitmap → 按位图读静止镜像"展开，明确不用 QMP `drive-backup`，因为离线 VM 没有 QEMU 进程。

- QEMU 自研 `block-dirty-bitmap-export/import`，导出 QEMUBMP 位图文件（11.1 上为 2026-11 的重写版，见下方说明）。
- 在线 VM：lictor 执行 `stop → export → clear → cont`，同一窗口内 reflink 出静止副本。
- 离线 VM：hyper-agent 通过 `qemu-storage-daemon` / `qemu-img` 导出位图。
- change-point 持久化到 ferrovirt DB 的 `vm_snapshot` / `vm_snapshot_disk`。
- aegis-sdk 提供 VDDK 风格 C API，支持控制面 NBD ticket 模式和本地模式（`Aegis_OpenLocal`）。
- `aegis-cbt` CLI 可 parse / verify / blank / merge QEMUBMP。
- 离线 CBT 复位用延迟模型：导出位图后生成 `<disk>.aegis-blank` 空白标记，VM 下次启动时由 lictor 导入空白位图完成复位，避免 freeze 阶段直接改源盘。

未完成：xvirt 自带的 BackupService 和 `aegis_change_point` 表。

**11.1 上的实现（2026-11-20 重写，`stable-11.1-dev` `b63be76397`）**：

- 来历：原实现是旧工作区 qemu 的提交 `1f5bbc160f` / `07e367520f`（10.0），重新导入 kairos 时丢失，升级到 11.1 时也没带上，从开发机、195 和 GitHub 都没能找回。现按 `xvirt/CBT_BITMAP_PERSISTENCE_SPEC.md` 在 11.1 上重写。命令名和 QEMUBMP v1 格式与 lictor、hyper-agent、aegis-sdk 现有代码一致。
- 代码在 `block/monitor/bitmap-file.c`，qemu-storage-daemon 里同样可用：
  - `block-dirty-bitmap-export {node, name, filename}`：先在内存里复制位图，再写 `<filename>.tmp`（0640），fdatasync 后改名；失败时删掉临时文件。
  - `block-dirty-bitmap-import {node, filename, persistent?}`：先完整校验文件（头、长度、CRC32C），再在节点上建位图。盘大小不符或同名位图已存在都返回错误；导入期间的新写入不会丢。`persistent` 默认按节点能否存持久位图决定（qcow2 存，raw 不存）。
- drive-mirror 完成后，源节点的有名位图会复制到目标节点（不持久化），存储迁移后 CBT 链不断。外部快照不复制，保持上游语义（iotest 260）。
- 没照搬规范的：迁移节点名兜底（用上游 `block-bitmap-mapping`）、带时间戳日志函数（用 `-msg timestamp=on`）、`qemu-img parsebitmap`（由 aegis-cbt 覆盖）。
- 验证（195）：新 iotest `tests/xvirt-cbt-bitmap-file` 8 例、位图/mirror/快照/备份/迁移相关的上游 iotests 35 个、qcow2 auto 组都通过。hyper-agent 用 qemu-storage-daemon 离线导出导入、aegis-cbt 的 verify/blank/merge、lictor `test_qemubmp` 也都对这个版本测过。**还没经 lictor 在真 VM 上做端到端验证**（运行中导出、开机导入空白位图复位）。

### 11.2 内存快照

- `snapshot-helper` 替代 libvirt iohelper，把迁移流改写成按页固定偏移的稀疏覆写文件，文件大小上限约为 RAM 加元数据。
- **内存分层快照**（`docs/memtier-snapshot-design.md`、`memtier-snapshot-restore-design.md`）：
  - 快照产物有两个文件：`.state` 保存仍在 DRAM 的常驻页和 `"memtier"` 段（gfn→slot），`.memtier.raw` 是后端的 reflink 副本，slot 偏移不变。
  - 恢复分 keep（默认，恢复后保持分层）和 merge（helper 合并导入）。
  - 落地情况：snapshot-helper `--memtier=keep|merge`、lictor/hyper-agent 的创建和 keep 恢复、`memtier_restore=auto|keep|merge`、ferrovirt `vm_snapshot.memtier_mode` / `memtier_artifacts` 两列都已接通；QEMU 两处必改已进 `stable-11.1-dev`。
  - 状态：**已实现未验证**。单测通过，QEMU 待编译，没有真机验证；裸设备/petrel 后端的导出未实现。

### 11.3 带快照树的跨存储迁移

- `xvirt-blockdiff`：用 FIEMAP 比较两个 reflink 快照的物理块来定脏区，子命令 diff / merge / apply / extract / splat / show。`.xvdiff` 格式为 `"XVDIFF\x01\x00" | disk_size | nrange | (off,len)*`。
- `xvirt-snapmig offline-tree`：先全量拷根，再按子节点 reflink + merge 重建整棵快照树。
- `petrel-snapdiff`：petrel 侧产出同格式 `.xvdiff`，用于 OCFS2 ↔ petrel 互迁。
- 已接入控制面：ferrovirt `service/snapshot_migrate.rs`（离线走 snapmig，在线走 lictor `storage_mirror_*`）、hyper-agent `snapshot_chain_migrate` 和 `petrel_chain.py`。代码注释称已实测，没找到独立测试报告。

### 11.4 drive-mirror 厚盘/薄盘

`docs/drive-mirror-sparse-thick-design.md`（**仅设计**，现象在 10.8.1.196 的 XFS 上实测过）：

- 问题：目标盘没被认定为全零时，QEMU 会按 16MB 发 write_zeroes，带 `discard=unmap` 时厚盘会被打成薄盘。
- 方案：lictor `storage_mirror_start` 对厚目标先 fallocate，再以 `mode=existing`、`unmap=false` 启动。
- lictor 尚未实现。

## 12. 资源规划与 VM 放置

设计稿 `docs/resource-planning.md`（v1，2026-09；§0 决策表为定论）。P0 + P1 + P2.5 已在 hyper-agent、lictor、ferrovirt、xvirt-client 四个仓实现，状态是**已实现未验证**：单测和编译级验证通过，真机端到端没跑。

三层机制：

1. **启动参数层**：bootprofile 写 `/etc/default/grub.d/99-xvirt-resource-plan.cfg` 并 update-grub，以 `/proc/cmdline` 为准对账（isolcpus、大页等）。
2. **cgroup v2 层**：`virt.slice`（平台服务）、`spdk.slice`（存储轮询核与大页池）、`machine.slice`（绑定/独占 VM）、`machine-shared.slice`（浮动 VM，继承 cpuset）。
   - VM 池用 `memory.high` 做软预算，给 memtier 总控留出下刷窗口；用 `memory.max` 兜底（MemTotal − 各项预留 − 大页 − 余量），每个 scope 设 `memory.oom.group=1`。
   - `numa_mode=node` 只用 cpuset.mems，QEMU 不带 host-nodes；hugetlb 控制器要手工开 `+hugetlb`。
3. **放置控制器**（hyper-agent，30s 一拍）：
   - VM 分固定 / 自动两类，等级为 A 独占 / B 节点共享 / C 浮动。
   - 负载档位分宽裕 / 适中 / 紧张，带滞回和稳定期。
   - 求解器是贪心装箱，差分动作按成本排序，每拍最多换节点一台；支持在线 NUMA 迁移（按 numa_maps 看进度）。
   - SPDK 候选核有上限；memtier 下刷档期间暂停放置。

落点：

| 层 | 内容 |
|---|---|
| ferrovirt | `service/resource_plan.rs`；新表 `host_resource_plan`、`host_cpu_topology`、`host_numa_node`、`placement_event`；`vm_config` 加放置列（cpu_placement、cpu_pins、numa_mode、numa_node、numa_json、placement_source、placement_tier、mem_backend、mem_reserved_mb）。路由 `/api/vcenter/host/{host}/{resource-topology,resource-plan,memory-tiering,placement}`、`/api/vcenter/vm/{vm}/placement`；任务 `applyResourcePlan` |
| hyper-agent | `support/resource_plan/*`；action scan_cpu_topology、get/validate/apply_resource_plan、placement_status/rebalance/advise、tiering_host_status；开机 oneshot `xvirt-resource-plan.service`（在 lictor、hyper-agent 之前执行 `apply-cached`） |
| lictor | `support/placement.py`：systemd scope、绑核、内存后端、vNUMA；live op `placement_set` |
| xvirt-client | 主机详情"资源规划"页：CpuTopologyBoard、MemoryBoard、TieringBoard（可拖水位线）、NodeHealthBoard、AutoPlacementCard、VmPlacementList |
| 观测 | `xvirt_placement_*`、`xvirt_node_health_*`、`xvirt_rdt_state`、`xvirt_cpu_runq_seconds`、`xvirt_slice_*`、`xvirt_hugepages*`、`xvirt_host_tiering_*`；RDT MBM 带宽；ebpf `runqlat` 按 cgroup 归户（未编译验证） |

已知差距（设计稿 §9.2）：

- 求解器只有贪心装箱，没有局部搜索。
- ferrovirt 侧浮动 VM 的内存分布仍按节点均分估算。
- vNUMA 强制固定。
- 分层 VM scope 的 `memory.min` 热集自动同步未做。
- `smart-numa-design.md` 是后续设计稿。

## 13. 内存分层 memtier

memtier 是跨组件能力：QEMU 内的 per-VM worker 把客户机 RAM 的冷页换出到块设备，用于内存超分。QEMU 代码已在 `stable-11.1-dev`（`docs/memory-tiering.md` 里写的 `memory-tiering` 分支已过时）。

**机制**（已实现）：

- userfaultfd 处理 MISSING 缺页和写保护，IO 走 io_uring O_DIRECT，不改 KVM。
- 判冷主路径用 page_idle；page_idle 失效的 CPU（如 Haswell-EP）上自动降级为 uffd-WP write-aging。现在会先探测 idle 位能否保持再决定是否降级，避免误回退。
- 后端：裸块设备、预分配 raw 文件或 SPDK 用户态 NVMe（不走 io_uring）。无元数据，按 4K slot 寻址，按 2MB chunk 做循环日志分配，配合打洞/压实。
- 线程：每个 region 一个 worker 线程，加 1–8 个 fault 线程（`fault-shards`）；换入读有设备优先级。
- 增强：
  - swap-cache：页从后端读回后仍保留 slot，干净页再次换出时不用重写；用哈希比对判断是否干净。
  - 零页不落盘。
  - 主动换回：单一 recall ring，hyper-agent 用两条水位线"回血"。
  - THP 拆分。
  - 限制换出的在途写。

**接口**（`qapi/memtier.json`）：

- 命令：`x-memtier-enable`、`x-memtier-set-policy`、`x-memtier-disable`、`x-memtier-query`、`x-memtier-query-heat`。最后一个返回 9 个 log2 空闲时长桶，每桶给出常驻/已换出字节和累计再访问页数。
- 事件：`MEMTIER_BACKING_FULL`、`MEMTIER_SWAPIN_ERROR`。
- 迁移能力：`memtier-shared-backing`（unstable）。
- 约 18 项 tunable，含 heat-scan-period-ms、heat-sample-percent、target-resident-mb、swapin-watermark、backing-discard、io-poll 等。

**分工**：

- QEMU 执行换出换入。
- lictor 持 QMP，并负责快照和迁移接入。
- hyper-agent 做主机级总控：按优先级和压力调度；水位线、PSI 满刻度、VM 池预算从资源规划缓存读；NVMe 每日写入预算 `write_budget.py` 钳制同盘 VM 的 `max-evict-mbps`。
- ferrovirt 和前端负责配置、编排和水位线编辑（TieringBoard）。
- "VM 实际常驻多少"完全由总控经 QMP 决定，不经 cgroup。cgroup 只做 `memory.high` / `memory.max` 兜底（§12）。
- 热度直方图目前**没有上层调用方**。

**约束**：memtier 要求 4K 匿名内存，与 petrel/vhost-user 需要的共享 memfd 互斥。fulmar 后端 `backing-fulmar.c` **仅设计**。

**实测结论**：

- 超分（123 机）：16GB 主机跑 14 台 8GB VM，即 7.5 倍超分，NVMe 填到 96%，无 OOM。性能损耗由冷页访问率决定：有局部性时 2.67 倍超分约慢 40 倍（绝对值几 µs）；均匀随机访问刚过 1 倍就慢约 110 倍。
- redis（降级的 write-aging 模式）：9 倍超分时 GET 只剩基准的 1.2%；petrel 作后端不适合 redis。
- 瓶颈：头号瓶颈是"换入同步单页、无预读"。之后补了 fault 路径分片和换入读优先级，修完后是否重测 redis 未确认。
- 热度直方图（195 机）：10% 抽样偏差约 ±1.4%，2 秒周期占 worker 约 14.5%。合入后换出器能守住常驻下限，写密集负载下 redis SET 从约 56k 降到约 39k 次/秒（换入增多所致）。
- NVMe 磨损：零页不落盘加分块日志后，写量比基线少 10–24%，98–100% 的写是同一 chunk 内顺序前进。
- bpf_fault（SOSP'26，仅评估）：换出无收益；换回在 10µs 级设备上理论上能缩短约四成，但还需要一处非主线内核改动，**现阶段不引入**。

## 14. guest 工具与 CDP 生态

### 14.1 xvirt-tools

`xvirt-tools` 是装在 guest VM 内的 daemon。host 通过 QEMU virtio-serial channel `xvirt-tools.0` 与它用 NDJSON 通信，用于 guest 内操作、状态与指标（hyper-agent 的 VM 指标走 `metrics.tick`）。

### 14.2 现行 CDP

cdp-worker 和 xbackup-tools 已并入 `cdp` 仓，独立的 cdp-worker 仓已从工作区移除。控制面（ferrovirt / hyper-agent / lictor）**目前没有编排 CDP**，它还是平台外的独立组件。

```text
QEMU 写完成后旁路复制 ──共享内存（每盘一块）──► cdp-recorder（每盘一进程，原 cdpwork）
                                                   │ v2：IO 日志 + 索引 + 位图；跟不上降级为位图
guest 内 cdp-agent ──TCP──► cdp-sink（iolog 模式，同样产出 v2）
                                                   ▼
                     cdplog 引擎（登记 v2 日志组 → 可写日志链，只写 v3）
          ┌──────────────────────┼──────────────────────┐
  nbdkit-cdplog-plugin     cdplog-ublkd             cdplog-shipd
  恢复 / 演练（只读视图、   切换后长期运行            容灾复制（dir / TCP / ssh，
  子链）                   （/dev/ublkbN 给 QEMU）    可续传）
```

- **采集**：QEMU 侧 QMP 为 `cdp-attach` / `cdp-detach` / `cdp-update` / `query-cdp` / `cdp-checkpoint` / `cdp-resume`，旧的 `io-redirection-*` 保留为 deprecated 别名。改名连同写钩子范围、写零按 1MiB 记录、fifo 错误标志对齐、阻塞模式的修复，于 2026-11-20 合入 `stable-11.1-dev`（合并 `3198719878`，另补 `system/runstate.c` 漏改 `7e96c87daa`）；编译、QMP 冒烟和 iotests 通过，**配合 cdp-recorder 与真 guest 的端到端未验证**。discard 不采集。
- **记录**：`cdp/recorder/`，二进制 `cdp-recorder`，保留 `cdpwork` 软链接和包名 Provides/Obsoletes。`cdp-checkpoint` 打同步点，`cdp-resume` 配合存储侧 reflink 快照恢复记录。
- **引擎**：`cdp/cdplog`，Rust `libcdplog` + C 接口 + `cdplog` CLI。能力有：崩溃恢复、持续合并（重写、打洞、并入 base、紧急回收、分级保留）、三种搬运模式（全量 / 按有效索引 / 降级为位图）、子链与回切、多盘一致同步点、降级组转精简段。格式口径：文档叫"v3"，文件头 version 字段为 4，按批自描述。到 `6efead6` 为止的代码在 10.8.1.195 上编译测试通过，**已实测**。未覆盖：跨机 ssh 搬运、真实 VM 降级流程、VMware 即时恢复、NVMe。
- **guest 内 CDP**：`cdp-agent`，Rust 用户态 + Linux `dm-cdp` / Windows `cdpflt` 过滤驱动，两者都做过真 VM 验证。cdp-sink 支持 iolog / disk / replica 三种模式，可组合。
- **VMware 恢复**：`cdp/tools/cdp-vmware-restore.py` 经 iSCSI 把恢复点映射给 ESXi，第一期在 ESXi 8.0U3 上验证过。它用的还是旧 chain 插件；旧的 `nbdkit-cdp(-chain)-plugin` 与新插件并存，退役未做。
- **规划**：fulmar 作为生产端来源直接导出 v3 段，取代"QEMU 采集 + recorder"的双写（仅设计）。

## 15. 远程桌面 xdesk

- QEMU 内置 `-display xdesk` 显示后端（`ui/xdesk.c`，已进 `stable-11.1-dev`）。
  - 视频：软件 surface 捕获 → GStreamer → H.264（优先 VA 硬编，回退 vaapi / x264）→ RTP/UDP。
  - 键鼠：走 TCP 控制通道（默认 6900）。
- 客户端是独立仓 `xdesk-client`，GTK4 + GStreamer 单文件。
- 阶段一端到端已实测：112 的 Arc A380 硬编，Debian 13 客户端在 Wayland 下渲染。
- 实现与设计稿 `qemu-virtual-desktop-protocol-design.md` v2 不同：设计稿是 `-display dbus` + DMA-BUF + 外部 worker + WebRTC、浏览器优先；实际是 QEMU 内编码 + 原生客户端 + 自定义 RTP。
- 控制面没有接入；音频、剪贴板、多显示器、USB 重定向、WebRTC 等阶段二、三内容未开始。

## 16. 监控与告警

### 16.0 现行方案

监控栈已从 Prometheus + node_exporter 迁到 VictoriaMetrics（`xvirt/monitoring/README.md`）：

```text
hyper-agent exporter :9101（host node_* + VM / datastore / zpool / 资源规划 / 分层指标）
ebpf_exporter :9435（biolatency / cfs-throttling / runqlat）
  -> vmagent :8429（1s/5s 抓取 + 30s stream aggregation）
  -> remote_write
  -> vmsingle :8428
  -> ferrovirt /api/vcenter/metrics（PromQL）
  -> nginx
  -> xvirt-client
```

配套：Grafana :3000、vmalert :8880、Alertmanager :9093、飞书 adapter :9094（webhook 仍是占位值，只实测过 mock 链路）。

**口径不一致（2026-11-22 按 kairos/main 核对，比原先判断更严重）**：VictoriaMetrics 栈只在 192.168.2.112 手工部署，没有打进 deb。`xvirt-meta` 只 Recommends prometheus 和 prometheus-node-exporter，而 CI 出离线包用 `--no-install-recommends`（`build-deb/ci/make-bundle.sh`），所以**全新 ISO 里没有任何时序库**；`xvirt-init` 仍去 enable prometheus 并放行 9090/9100。ferrovirt 默认查 VictoriaMetrics :8428（`assets/ferrovirt.env`、`src/config.rs`），查不到时静默返回空序列，界面图表全空。主机查询还依赖 vmagent 流聚合后的指标名（`node_*:30s_rate_avg`），换回原生 Prometheus 也查不到。1.0 的处理见 §16.3。

设计中未落地：

- `docs/vm-direct-metrics-design.md`：hyper-agent 直采 QMP / cgroup / PSI / tap 计数。文中"hyper-agent 持 QMP"已过时。
- `docs/memtier-monitoring-design.md`：memtier 指标分档、看板、告警。现在已推出 7 个 VM 级和 3 个主机级指标。

### 16.1 现行方案的问题

拉模式要为每类数据配 exporter 和抓取目标；VM、盘、OSD 随时增减，要靠服务发现跟；降采样后突发只剩平均值；长期容量随保留期线性增长。这些是引入 kestrel 的原因。

### 16.2 kestrel 推送式监控框架（拟替代 VictoriaMetrics + exporter）

设计稿：`docs/kestrel-design.md` v1.5；代码：独立子仓 `kestrel`。

```text
hyper-agent / lictor（Python SDK）
petrel / fulmar / cdplog / ferrovirt（Rust SDK）     一条长连接，推送协议
qemu / concilium / cdp（C SDK，调用方定期 kst_poll）  ───────────────────►  kestreld :7410
                                                                               │
                                       ferrovirt / 界面 / 脚本 ◄── HTTP JSON :7411
```

- **推而不拉**：底层服务在自己进程里取数，通过 SDK 推给 `kestreld`。没有 exporter、没有 scrape、没有服务发现；对象一帧注册、一帧注销，超时自动判失联。
- **对象 + 指标集**，不是序列 + 标签：一个监控对象（一台 VM、一块盘、一个 OSD）绑定一组有名字、有类型的字段（瞬时值 / 累计量 / 状态 / 事件），每次推送一行；没有标签倒排索引。
- **分层存储、写入时精简**：原始 / 1 分钟 / 5 分钟 / 1 小时 / 1 天五层（默认原始 2 小时，聚合层分别 1 天 / 7 天 / 90 天 / 2 年）。多个对象共用按时间窗口切分的顺序追加文件；聚合在写入时增量完成，窗口到期封存一次，过期整窗删除，没有持续的后台压实。保留期可以按对象类别单独配置。每对象磁盘占用到保留期满即封顶（16 字段约 2.6 MB）。
- **保留突发**：每个聚合桶带 min / max / 峰值时刻；在线检测到突发时把那一段原始分辨率整行另存为片段，按重要度保留，可配置为永久；状态变化和突发都记成事件。
- **服务端**：一个 epoll 主线程收数并答内存查询，同步、封存各一个辅助线程，另有查询线程读文件，只依赖 libc。查询接口为 `/api/v1/{objects,latest,series,bursts,events,stats}`，不做 PromQL；跨对象聚合由知道归属关系的服务另推一个对象（如 hyper-agent 推 `host/<名>`）。
- **资源**：1 万对象实测内存约 129 MB（每对象约 13 KB），机械盘即可承载。

接入后的数据流变化：

| 现行 | kestrel |
|---|---|
| hyper-agent exporter :9101 + ebpf_exporter :9435 被 vmagent 抓 | hyper-agent / lictor 等直接推 :7410 |
| vmagent → vmsingle :8428 | kestreld 自带存储 |
| ferrovirt `resource_metrics.rs` 用 PromQL 取数 | 改调 :7411 的 `series` / `latest` |
| vmalert + Alertmanager 告警 | 告警由 ferrovirt 规则引擎评估（2026-11-22 已定，§16.3.6），kestreld 只提供数据与事件 |

**当前状态（2026-11-20）**：子仓 `kestrel` main `912e7c6`（含 v1.5 的按类别保留与片段永久保留）。

- 2026-11-19 在测试机 10.8.1.195 上编译通过，57 个测试和 24 项端到端都过。
- 1 万对象规模、长范围查询、冷查询并发都实测过，但这些是 v1.5 之前跑的，按类别保留加入后没重跑。
- 还**没有任何生产组件接入 SDK**，现行监控栈不变。
- 待拍板六项（设计稿 §13）中，第 4、5 项已于 2026-11-22 定下：**1.0 直接用 kestrel，不打包 VictoriaMetrics**；**告警放 ferrovirt**。其余四项（名字、默认分层与保留期、min/max 是否用 f32、数据目录放什么盘）仍待定，接入方案按默认值写。

### 16.3 kestrel 接入方案（1.0）

本节是 kestrel 接入 1.0 的实现设计，读者是直接写代码的工程师或 Agent。目标是让 Cairn 1.0 中被监控口径阻塞的五条需求可交付：REQ-MON-001（主机 CPU / 内存 / 网络 / 磁盘实时与历史曲线）、REQ-MON-002（VM CPU / 内存 / 磁盘 IOPS 与延迟 / 网络）、REQ-MON-006（告警规则：阈值、持续时间、级别、对象作用域）、REQ-MON-007（预置告警：主机断连、存储将满、HA 动作、硬件故障）、REQ-PLT-011（资源概览仪表盘含告警汇总）。另外覆盖 REQ-STO-019（Petrel 集群 / OSD 健康）的监控与告警部分。

**Petrel 覆盖（2026-11-23 补充）**：1.0 部署场景里数据存储和 VM 盘默认放在 Petrel 共享池上（xvirt-autotest `physical-dual` 场景的 `fulmar` 池就是 Petrel 池，与 fulmar 文件系统无关）。原方案的 `ds` 只覆盖挂在 `/datastore/<段>` 的文件系统，`vmdisk` 只取 QMP `query-blockstats`，而 Petrel 数据存储没有文件系统挂载、Petrel 盘经 vhost-user-blk 直挂不过 QEMU 块层，两者都会没有数据；「存储将满」预置规则也就对 Petrel 不生效。本节因此增加 `petrel` / `petrelpool` / `petrelosd` 三个类别、`vmdisk` 的 petrel-vhost 数据源、Petrel 数据存储到池的映射与三条 Petrel 预置规则（§16.3.2–§16.3.6）。

已定的决定（2026-11-22）：

1. 1.0 的时序库是 kestrel，不打包 VictoriaMetrics，也不再依赖 Prometheus。
2. 告警由 ferrovirt 评估：ferrovirt 新增规则表与周期评估器，命中后调用现有 `service/alert.rs` 的 `raise` / `resolve`；kestreld 不做告警。
3. 前端接口形状不变：`GET /api/vcenter/{vm|host|datastore}/{id}/metrics` 继续返回 `{key: [[ts, v], ...]}` / `{key: {label: [[ts, v], ...]}}`，前端图表组件不改。
4. 192.168.2.112 上手工部署的 VictoriaMetrics / Grafana / vmalert 不动也不再接入；1.0 不打包 Grafana、Alertmanager、飞书 adapter。

#### 16.3.1 部署拓扑

```text
standalone 节点                         management + compute
┌───────────────────────────┐          ┌──────────── center ────────────┐
│ hyper-agent ─┐            │          │ kestreld :7410 (0.0.0.0, 口令) │◄── compute 节点的
│ lictor ──────┼─► kestreld │          │          :7411 (127.0.0.1)     │    hyper-agent / lictor
│              │   127.0.0.1│          │ ferrovirt ──► :7411            │    经管理网推送
│ ferrovirt ───┘◄── :7411   │          └────────────────────────────────┘
└───────────────────────────┘
```

| 角色 | 是否运行 kestreld | 生产者推向 | 谁查询 |
|---|---|---|---|
| standalone | 是 | 本机 `127.0.0.1:7410` | 本机 ferrovirt |
| management（center） | 是 | 自己没有生产者（无 hyper-agent / lictor） | center ferrovirt |
| compute | 否 | 纳管它的 center 的 `<center 管理 IP>:7410`；未纳管时不推 | — |

- 查询端口 `:7411` 没有鉴权（kestrel README「未实现」），一律只监听 `127.0.0.1`，只给同机 ferrovirt 用；不对外放行。
- 推送端口 `:7410` 在 management / standalone 上监听 `0.0.0.0`，并配置预共享口令（kestrel 设计 §3.5，HELLO 里校验，不加密，走管理网）。
- 数据目录 `/var/lib/kestrel`。center 是一台 VM，系统盘需按 kestrel 设计 §5.2 的容量公式预留。1.0 规模上限（REQ-PLT-016）还没定，先按 **32 台主机 × 每台 50 台 VM**（每台 VM 2 块盘、1 块网卡、3 个客户机文件系统；每台主机 8 个 OSD）、默认分层（原始层 2 小时，聚合 1 分钟 1 天 / 5 分钟 7 天 / 1 小时 90 天 / 1 天 2 年）估算：

  | 类别 | 字段数 F | 间隔 I | 对象数 | 每对象活满 | 合计 |
  |---|---|---|---|---|---|
  | `vm` | 16 | 10 s | 1600 | 2.6 MB | 4.2 GB |
  | `vmdisk` | 7 | 10 s | 3200 | 1.2 MB | 3.9 GB |
  | `guest` | 9 | 10 s | 1600 | 1.5 MB | 2.5 GB |
  | `vmtier` | 7 | 30 s | 1600 | 1.2 MB | 1.9 GB |
  | `vmnic` | 6 | 10 s | 1600 | 1.1 MB | 1.7 GB |
  | `guestfs` | 2 | 60 s | 3200 | 0.4 MB | 1.3 GB |
  | 主机类（`host` `hostfs` `hostnic` `ds` `plan`） | — | — | 352 | — | 0.6 GB |
  | Petrel 类（`petrel` `petrelpool` `petrelosd`） | — | — | 261 | — | 0.3 GB |
  | 合计 | | | 约 1.3 万 | | **约 16 GB** |

  这是数据活满 2 年后的稳定值（之后不再增长）；加上突发片段（每对象上限 256 KiB）、每层多留的一个窗口与已删除 VM 在 `purge_after`（30 天）内的残留，**center 给 `/var/lib/kestrel` 预留 32 GB**。standalone 单机按同样公式只有几百 MB。规模上限定了以后按 `(raw_span / I) × (8 + 8F) + Σ聚合层桶数 × (16 + 24F)` 重算。

#### 16.3.2 对象模型与键

对象键是 `类别/名字`，会变的东西放属性（kestrel 设计 §2.1）。1.0 用以下类别，**类别划分上线后不再改**（改类别的保留覆盖会换数据目录，kestrel README「保留设置」）：

| 类别 | 键 | 推送者 | 间隔 | 属性 | 说明 |
|---|---|---|---|---|---|
| `host` | `host/<machine-id>` | hyper-agent | 10 s | `ip` `hostname` | 主机 CPU / 内存 / 负载 / PSI |
| `hostfs` | `hostfs/<machine-id>/<挂载点转义>` | hyper-agent | 30 s | `host` `mount` `fstype` | 每个真实文件系统一个对象（排除 tmpfs / overlay / squashfs / devtmpfs） |
| `hostnic` | `hostnic/<machine-id>/<设备名>` | hyper-agent | 10 s | `host` `device` | 物理网卡与 br-ex（排除 lo / tap / veth / vnet） |
| `ds` | `ds/<datastore 挂载段>` | hyper-agent | 30 s | `host` `mount` `type` | 数据存储容量与 IO |
| `vm` | `vm/<vm_uuid>` | lictor | 10 s | `host` `name` | **宿主机侧**：cgroup CPU / 内存、tap 网卡合计、块设备合计 |
| `vmdisk` | `vmdisk/<vm_uuid>/<设备 alias>` | lictor | 10 s | `host` `vm` `alias` `bus` | QMP `query-blockstats` 每块盘 |
| `vmnic` | `vmnic/<vm_uuid>/<设备 alias>` | lictor | 10 s | `host` `vm` `alias` `model` | tap 计数，每块网卡 |
| `guest` | `guest/<vm_uuid>` | hyper-agent | 10 s | `host` | **客户机内**（xvirt-tools 上报）：负载、内存使用 / 可用、swap |
| `guestfs` | `guestfs/<vm_uuid>/<挂载点转义>` | hyper-agent | 60 s | `host` `vm` `mount` `fstype` | 客户机内文件系统 |
| `plan` | `plan/<machine-id>` | hyper-agent | 30 s | `host` | 资源规划 / 分层 / 放置的主机级字段（现 `collectors/resource_plan.py` 的主机级指标） |
| `vmtier` | `vmtier/<vm_uuid>` | hyper-agent | 30 s | `host` | memtier 每 VM 字段（现 `xvirt_vm_tiering_*`） |
| `petrel` | `petrel/<fsid>` | hyper-agent（mon 所在主机） | 30 s | `mon_addr` | Petrel 集群整体：健康、mon、OSD 计数 |
| `petrelpool` | `petrelpool/<fsid>/<池名>` | hyper-agent（mon 所在主机） | 30 s | `pool_id` `size` `device_class` | 每个池的容量与用量（Petrel 数据存储的容量曲线与「将满」告警来源） |
| `petrelosd` | `petrelosd/<fsid>/<osd id>` | hyper-agent（mon 所在主机） | 30 s | `host` `addr` `device_class` | 每个 OSD 的上下线、是否在集群内与用量 |

约定：

- **主机标识用 `/etc/machine-id`**，不用 IP 或主机名（两者都会变；standalone 自注册的 `host.ip` 默认还是 `127.0.0.1`，见 `service/enrollment.rs` 的 `local_ip.unwrap_or("127.0.0.1")`）。ferrovirt 的 `host` 表加列 `machine_id VARCHAR(64) NULL`，在 `host_manager::connect_host` 连上 agent 时调用 hyper-agent 新 action `host_identity`（返回 machine-id、hostname、管理网 IP）回填；之后键直接拼成 `host/<machine_id>`，不按 IP 匹配。属性 `ip`、`hostname` 只用于展示与排查。
- **挂载点转义**：`/` → `_`，根目录写成 `root`，其余字符按 kestrel 键规则（不能含 `#`、空格）百分号转义。
- **VM 迁移**：新主机上的 lictor 用同一个键重新注册、更新 `host` 属性即可，历史不断（kestrel 设计 §2.1）；源端停推后对象进入失联、目的端来数据后恢复在线。
- **VM 删除**：lictor 在 VM 删除（不是关机）时对 `vm` / `vmdisk` / `vmnic` 发 GONE；关机只停推，对象自然失联。
- **Petrel 三类只由 mon 所在主机推送**（hyper-agent 读本机 `cluster.json` 的 `role`，含 mon 的那台推；单节点 Petrel 就是本机），避免多台各推一份。数据来自现有 `support/petrel/cluster.py::status()`（`osds` / `usage` / `pools` 文本与 `_build_topology` 的解析结果），不新开查询通道。mon 迁移到别的主机后由新主机用同一组键接着推，历史不断。`device_class` 取 OSD 配置（`osd.<id>.toml`，hyper-agent `a6a99cb` 起按加盘参数写 `hdd` / `ssd` / `nvme`）；池的 `device_class` 取 CRUSH 规则所选的类，规则不限类时为空。
- 宿主机侧与客户机内分开成 `vm` 与 `guest` 两类：宿主机侧不依赖 xvirt-tools，没有装 guest 工具的 VM 也有 CPU / 磁盘 / 网络曲线（REQ-MON-002 必需）；客户机内的内存使用、文件系统只有装了 xvirt-tools 才有。

#### 16.3.3 指标集与字段

累计量推原始累计值，服务端换算速率（kestrel README「接入一个服务」）；每个数值字段设突发下限 `floor`。

| 指标集 | 字段（类型，单位，floor） | 数据来源 |
|---|---|---|
| `host.v1` | `cpu_usage`（gauge，ratio 0–1，0.1）`load1` `load5` `load15`（gauge）`mem_total` `mem_available`（gauge，B）`psi_cpu_some` `psi_mem_some` `psi_io_some`（counter，µs） | `/proc/stat`（agent 按间隔算比率后推 gauge）、`/proc/loadavg`、`/proc/meminfo`、`/proc/pressure/*`；即现 `collectors/host.py` 的数据 |
| `hostfs.v1` | `size` `avail`（gauge，B）`used_ratio`（gauge，0–1，0.05） | `statvfs` |
| `hostnic.v1` | `rx_bytes` `tx_bytes` `rx_packets` `tx_packets` `rx_errors` `tx_errors` `rx_drops` `tx_drops`（counter）`link`（state：0 down / 1 up） | `/sys/class/net/<dev>/statistics/*`、`operstate` |
| `ds.v1` | `size` `used` `avail`（gauge，B）`used_ratio`（gauge）`reads` `writes` `read_bytes` `write_bytes` `read_time_ms` `write_time_ms` `io_time_ms` `weighted_io_time_ms`（counter） | 现 `collectors/datastore.py` |
| `vm.v1` | `cpu_usage`（gauge，ratio，按 vCPU 数归一，0.1）`mem_rss`（gauge，B）`mem_configured`（gauge，B）`net_rx_bytes` `net_tx_bytes` `net_rx_packets` `net_tx_packets`（counter，全部 tap 之和）`disk_rd_ops` `disk_wr_ops` `disk_rd_bytes` `disk_wr_bytes` `disk_rd_time_ns` `disk_wr_time_ns`（counter，全部盘之和）`power`（state：0 关 / 1 运行 / 2 暂停） | cgroup v2 `cpu.stat`、`memory.current`（VM 所在 systemd scope）、tap 计数、QMP `query-blockstats` |
| `vmdisk.v1` | `rd_ops` `wr_ops` `rd_bytes` `wr_bytes` `rd_time_ns` `wr_time_ns` `flush_ops`（counter） | 普通盘：QMP `query-blockstats`（`rd_total_time_ns` 等）。**Petrel 盘**（vhost-user-blk，不在 QEMU 块层，`query-blockstats` 里没有）：petrel-vhost 每卷计数文件（§16.3.4「petrel-vhost」） |
| `vmnic.v1` | `rx_bytes` `tx_bytes` `rx_packets` `tx_packets` `rx_drops` `tx_drops`（counter） | tap 的 `/sys/class/net/<tap>/statistics/*`（方向与 guest 视角相反，推送时交换 rx/tx，使字段表示 guest 视角） |
| `guest.v1` | `load1` `load5` `load15` `mem_total` `mem_used` `mem_available` `swap_total` `swap_used` `cpu_usage`（gauge） | xvirt-tools `metrics.tick`（现 `collectors/vm.py` 缓存的那一帧） |
| `guestfs.v1` | `size` `used`（gauge，B） | 同上 |
| `plan.v1`（`plan/<machine-id>`） | `tiering_pressure` `tiering_headroom` `tiering_state`（state）`placement_load_index` `placement_tier`（state：0 roomy / 1 medium / 2 tight）`placement_pending_actions` `placement_dedicated_cores_need` `placement_dedicated_cores_avail`；四个 CPU 池各一个 `pool_cpu_ratio_<池>` `pool_cpus_<池>`（池固定为 system / virt / spdk / vm）；六个顶层 slice 各一个 `slice_cpu_seconds_<slice>`（counter）`slice_memory_<slice>`（slice 名里的 `.` 换成 `_`） | `collectors/resource_plan.py`（推送线程按需调它的 `collect`） |
| `plannode.v1`（`plan/<machine-id>/node<N>`） | `mem_total` `mem_free` `mem_used` `file_pages` `anon_pages` `slab` `oc` `free_dedicated_cores` `free_mem` `health_score`（gauge）`numa_hit` `numa_miss` `numa_foreign`（counter） | 同上 |
| `vmtier.v1` | `resident_bytes` `swapped_bytes` `backing_used_bytes` `backing_total_bytes` `evict_mbps`（gauge）`swapin_faults` `evict_aborts`（counter）`tier_state`（state）`remote_mem_ratio` `mbm_total` `mbm_local` `remote_bw_ratio` `llc_occupancy`（gauge） | 同上 |

`plan` 类只取上表这些固定字段，不按运行时的 label 值动态拼字段（指标集一变对象就会被退役、历史分段，kestrel 设计 §2.1）；per-CPU busy / runq、per-service、hugepages、NUMA 距离矩阵、vCPU 落点这些高基数或静态的数据 1.0 不推，需要时在资源规划页直接调 agent。
| `petrel.v1` | `health`（state：0 HEALTH_OK / 1 HEALTH_WARN / 2 HEALTH_ERR / 3 未知）`mon_up`（state）`osd_total` `osd_up` `osd_in`（gauge） | `status()` 的 `health`、`mon_up`、OSD 列表（up = `Up`，in = `reweight > 0`） |
| `petrelpool.v1` | `stored_bytes` `objects`（gauge）`max_avail_bytes`（gauge，B）`used_ratio`（gauge，0–1，0.05）`pg_num`（gauge） | `pools` / `usage` 文本；`used_ratio = stored / (stored + max_avail)`。**`max_avail` 是近似值**：全部 up 且在集群内的 OSD 剩余容量之和 ÷ 池副本数——`petrel-mon` 文本里没有 CRUSH 规则，按设备类区分的池（缓存层 / 数据层）会被高估；精确值要 petrel-mon 提供按规则的可用量（K1 之后） |
| `petrelosd.v1` | `up` `in`（state）`used_bytes` `total_bytes`（gauge，B）`used_ratio`（gauge，0–1，0.05） | `osds` / `usage` 文本 |

Petrel 字段以 K1 实现时 `petrel-mon` 的 `osds` / `usage` / `pools` 实际输出为准；文本里拿不到的字段（例如 PG 降级数）不推，不编造。PG 级健康与修复建议（REQ-STO-019 的其余部分）仍由存储页面直接调 `petrel_cluster_status` 展示，不经监控。

`vm.v1` 的 IO 延迟由 ferrovirt 在查询时按点计算：`(Δrd_time_ns / Δrd_ops) / 1e6` 毫秒（见 §16.3.5）；kestrel 存的已是每秒速率，同一对象两字段同一时间戳，可直接相除。

#### 16.3.4 生产者改造（hyper-agent、lictor）

两个进程都用 `kestrel/sdk/python/kestrel.py`（单文件、只用标准库）。打包时把它装成 `/usr/lib/python3/dist-packages/kestrel.py`，由 kestrel 的 deb 提供，hyper-agent / lictor 的 deb 依赖它。

公共部分（新模块 `hyper_agent/support/kestrel_push.py`，lictor 侧对应 `lictor/support/kestrel_push.py`，代码相同）：

- 读 `/etc/xvirt/kestrel.conf`（见 §16.3.7）：`KESTREL_ADDR`、`KESTREL_TOKEN`。文件不存在或 `KESTREL_ADDR` 为空 → 不推送（compute 未纳管时就是这样），每 30 秒重新检查一次文件，变化时重建 `Client`（重新声明、重新注册）。
- `Client(addr, source="hyper-agent"|"lictor", token=...)`，未确认缓冲按 kestrel README 建议设为「一个落盘周期（默认 120 s）+ 同步时间」的数据量；连不上时 SDK 自行退避重连，不阻塞采集线程。
- 采集与推送跑在独立线程，按对象间隔对齐；单个对象采集异常只记日志、跳过这一拍。

hyper-agent：

1. `metrics_exporter/` 保留采集代码（collectors），新增 `metrics_exporter/kestrel_sink.py` 把每个 collector 的快照转换成上表的对象与行；`MetricsExporter.start()` 启动推送线程。
2. Prometheus 文本接口 `:9101` 改为默认关闭（环境变量 `XVIRT_PROM_EXPORTER=1` 才开），只为过渡期给手工监控机用；xvirt-init 不再放行 9101。
3. `guest` / `guestfs` 对象由 `VmMetricsCollector` 收到 `metrics.tick` 时推送；超过 60 s 没有 tick 的 VM 停推（对象自然失联）。
4. 主机标识：启动时读 `/etc/machine-id`；新增 action `host_identity` 返回 `{machine_id, hostname, mgmt_ip}`（`mgmt_ip` 取默认路由所在接口的 IPv4，即 br-ex），供 ferrovirt 回填 `host.machine_id`（§16.3.2）。

lictor（持有 QMP，宿主机侧 VM 数据由它推）：

1. 新模块 `lictor/monitor/vm_metrics.py`：每 10 s 对每台已 attach 的 VM 发一次 `query-blockstats`（走现有 QMP 连接，不新开 socket；不要在 QMP read loop 协程里同步等待，见 xvirt CLAUDE.md「STOP 事件取 reason 的死锁陷阱」），读该 VM systemd scope 的 cgroup `cpu.stat` / `memory.current`，读其 tap 设备计数，推 `vm` / `vmdisk` / `vmnic`。
2. 设备 alias 与 tap 名的对应关系取自 VM 的 config.xml（lictor 启动 VM 时已解析过），盘与网卡热插拔后在下一拍按新设备列表注册 / 注销子对象。
3. **Petrel 盘**：VM 目录下 `petrel-disks.json` 里每块盘（`alias`、`vol`）也注册 `vmdisk` 对象（属性多 `backend=petrel` `bus=vhost-user-blk` `vol`），计数读 petrel-vhost 的每卷计数文件（见下），不发 `query-blockstats`；`vm` 的磁盘合计字段把两种来源加在一起。计数文件不存在（旧版 petrel-vhost）或超过 15 s 没更新（petrel-vhost 被 SIGKILL 后的残留）时这块盘这一拍跳过，不推 0。
4. VM 删除时对三类对象发 GONE。删除动作目前在 hyper-agent 还是经瘦代理到 lictor 需实现时确认；若不经 lictor，由 hyper-agent 在删除成功后用同样的键发 GONE（GONE 只需要键，不要求是注册它的那个连接——实现时对照 kestrel 设计 §3 确认，不允许则改为让 lictor 提供 `metrics_forget` action 给 hyper-agent 调用）。

hyper-agent 的 Petrel 部分：`kestrel_sink` 在本机是 Petrel mon 所在主机时每 30 s 调一次 `support/petrel/cluster.py::status()`，推 `petrel` / `petrelpool` / `petrelosd`；`status()` 里有阻塞调用，跑在推送线程里，不进 ZMQ 处理协程。OSD 被移出集群（CRUSH 里没有了）时发 GONE。

petrel-vhost（petrel 仓 `bins/petrel-vhost`，K1 新增；一个进程服务一个卷，`--stats-dir` 默认 `/run/petrel/vhost`，传空串关闭）：每个卷维护读写次数、字节、累计耗时（ns，从请求解析完到完成，含排队）与 flush 次数（原子计数，只计成功的请求），每 5 s 以 `rename` 原子替换方式写 `/run/petrel/vhost/<转义后的卷名>.stats`（转义：保留 `[A-Za-z0-9_-]` 与非首位的 `.`，其余每字节 `%XX`；uuid 形式的卷名不变）（一行 JSON：`{"rd_ops":…,"wr_ops":…,"rd_bytes":…,"wr_bytes":…,"rd_time_ns":…,"wr_time_ns":…,"flush_ops":…}`，进程启动时清零，卷关闭时删文件）。lictor 按盘的卷名读这个文件；计数器回绕或进程重启导致的回退由 kestrel 的 counter 处理（设计 §2.2）。不走 socket、不新增端口。

#### 16.3.5 ferrovirt 查询改造

1. 配置：`FERROVIRT__METRICS__KESTREL_URL`（默认 `http://127.0.0.1:7411`），替代 `FERROVIRT__METRICS__PROMETHEUS_URL`（旧键读到时打一条弃用告警并忽略）。`assets/ferrovirt.env` 同步改。
2. 新模块 `src/service/kestrel_client.rs`：封装 `objects` / `latest` / `series` / `events` 四个调用（kestrel 设计 §9），复用 `state.http`，超时沿用 `FERROVIRT__METRICS__TIMEOUT_SECONDS`。`series` 的 `points` 取 `(end - start) / step`，上限 1000；聚合层返回的「平均」作为曲线值。
3. 重写 `src/service/resource_metrics.rs` 的三个入口，**输出键名与形状保持不变**：

| 输出键（现有） | 新来源 |
|---|---|
| host `cpu_usage_ratio` / `load_1/5/15` / `memory_used` / `memory_total` | `host/<host.machine_id>`：`cpu_usage`、`load*`、`mem_total - mem_available`、`mem_total`；`machine_id` 为空（agent 还没连上过）时返回空序列并带 `source_error: "host identity unknown"` |
| host `fs_used` / `fs_total`（按 `mountpoint` 分组） | `objects?class=hostfs&attr.host=<id>` 逐个取 `size - avail`、`size`，分组标签用属性 `mount` |
| host `net_rx_bps` / `net_tx_bps`（按 `device` 分组） | `hostnic` 对象的 `rx_bytes` / `tx_bytes`（已是速率） |
| vm `cpu_usage_ratio` | `vm/<uuid>` 的 `cpu_usage`（宿主机侧；不再依赖 guest 工具） |
| vm `memory_used` / `memory_total` / `swap_used` / `load_1` | 有 `guest/<uuid>` 时取客户机内 `mem_used` / `mem_total` / `swap_used` / `load1`；没有时 `memory_used` 退回 `vm` 的 `mem_rss`、`memory_total` 用 `mem_configured`，其余缺省 |
| vm `fs_used` / `fs_total`（按 `mount`） | `guestfs` 对象 |
| vm `net_rx_bps` / `net_tx_bps` / `net_rx_pps` / `net_tx_pps`（按 `iface`） | `vmnic` 对象，分组标签用属性 `alias` |
| vm **新增** `disk_iops_read` / `disk_iops_write` / `disk_bps_read` / `disk_bps_write` / `disk_lat_read_ms` / `disk_lat_write_ms`（按 `disk` 分组） | `vmdisk` 对象；延迟 = `rd_time_ns / rd_ops / 1e6`（`rd_ops` 为 0 的点置空）。前端 VM 监控页加一组磁盘图（REQ-MON-002） |
| datastore 全部现有键 | `ds/<挂载段>`；`latency_read/write` 同上按点相除，`util` = `io_time_ms / 1000` |
| **Petrel 数据存储**（`datastore.type = petrel`）`size_total` / `size_used` / `size_avail` | `petrelpool/<fsid>/<池名>`：`stored_bytes + max_avail_bytes`、`stored_bytes`、`max_avail_bytes`。池名取 `datastore` 表新列 `petrel_pool VARCHAR(64) NULL`（`create_datastore_task` 建 Petrel 数据存储时按请求的 `pool_name` / 集群默认池写入；已有行在 `ensure_schema` 后由一次 `petrel_cluster_status` 回填 `pool_name`）；fsid 从同一次状态查询取并缓存。IO 键（`iops_*` / `bps_*` / `latency_*` / `util`）返回空序列并带 `source_error: "not collected for petrel datastore"`——池级 IO 不在 1.0 采集范围（REQ-MON-003 未承诺），按盘的 IO 在 VM 监控页看 |

4. kestrel 不可达或对象不存在时**不再静默返回空序列**：返回空序列的同时带原因——整体取不到（kestreld 不可达、`host.machine_id` 为空）放顶层 `"source_error": "<原因>"`，个别键没有数据源（对象不存在、Petrel 数据存储的 IO、没装 guest 工具时的 swap / 负载）放 `"series_errors": {"<键>": "<原因>"}`。前端监控页状态栏显示「监控服务不可用」而不是一片空白。数据存储级 IO 只有一组（hyper-agent 已把底层设备加总），分组标签固定为 `all`。
5. 删除 PromQL 透传接口 `/api/vcenter/metrics/query` 与 `/query_range`（`controller/vcenter/metrics.rs`）和 L1 白名单（`read::metrics::validate`）；调用这两个接口的前端代码一并删除。唯一的调用方是资源规划页的每 CPU 利用率（`xvirt_cpu_busy_ratio`，per-CPU 不进 kestrel），改为新接口 `GET /api/vcenter/host/{id}/cpu-busy` → hyper-agent action `cpu_busy`（两次读 `/proc/stat`，间隔 500 ms）。
6. 资源概览（REQ-PLT-011）新接口 `GET /api/vcenter/overview`：主机数 / VM 数（按电源状态）来自 DB；CPU、内存总量与使用率用 `latest?class=host&field=cpu_usage|mem_total|mem_available` 按主机汇总（内存中取，不读盘）；存储总量与使用率用 `latest?class=ds`；告警汇总复用 `alert::summary`。management 角色按 center 纳管的全部主机汇总，standalone 只有本机。

#### 16.3.6 ferrovirt 告警规则引擎（REQ-MON-006 / 007）

表（`service/alert_rule.rs` 的 `ensure_schema` 建）：

```sql
CREATE TABLE IF NOT EXISTS alert_rule (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  uuid          CHAR(36) NOT NULL UNIQUE,
  name          VARCHAR(128) NOT NULL,
  enabled       TINYINT(1) NOT NULL DEFAULT 1,
  builtin       TINYINT(1) NOT NULL DEFAULT 0,      -- 预置规则：可改阈值/级别/启停，不可删除
  object_class  VARCHAR(32)  NOT NULL,              -- host | hostfs | hostnic | ds | vm | vmdisk | guest | guestfs | petrel | petrelpool | petrelosd
  field         VARCHAR(64)  NOT NULL,              -- 该类别指标集里的字段名
  op            VARCHAR(2)   NOT NULL,              -- > >= < <=
  threshold     DOUBLE       NOT NULL,
  duration_sec  INT          NOT NULL DEFAULT 60,   -- 持续超阈这么久才告警
  severity      VARCHAR(16)  NOT NULL,              -- critical | warning | info（与现有 alert 表一致）
  scope_kind    VARCHAR(16)  NOT NULL DEFAULT 'all',-- all | cluster | host | vm | datastore
  scope_id      VARCHAR(64)  NULL,                  -- 对应实体 uuid / cluster id
  title_tpl     VARCHAR(255) NULL,                  -- 可用 {object} {value} {threshold}
  created_by    VARCHAR(64)  NULL,
  created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

评估器（`service/alert_rule.rs::evaluator_loop`，在 `main.rs` 与 `reconcile_loop` 同样方式启动；只在有 kestreld 的角色即 management / standalone 运行）：

1. 每 30 s 一拍。把启用的规则按 `(object_class, field)` 分组，每组调一次 `latest?class=&field=`（从 kestreld 内存取，不读盘），不按规则逐条请求。
2. 作用域过滤：`host` → 对象属性 `host` 等于该主机的 `host.machine_id`；`vm` → 键里的 vm_uuid；`datastore` → `ds` 键，Petrel 数据存储则是它对应的 `petrelpool` 键；`cluster` → 集群成员主机集合。
3. 每个 `(规则, 对象)` 在内存里记「首次超阈时刻」；持续超阈达到 `duration_sec` → `alert::raise`，`alert_key = rule:<rule uuid>:<object key>`（与现有 `raise` 的去重机制一致，重复命中只累加 `occur_count`）；回到阈值内 → `alert::resolve(alert_key)`。对象失联（`latest` 不再返回它）时不 resolve，保持原状态，避免数据中断被误当成恢复。
4. ferrovirt 重启后内存状态丢失：启动时对所有 `alert_key LIKE 'rule:%'` 的活动告警重新评估一次，仍超阈则保持，不超则 resolve。
5. 规则被禁用或删除时，resolve 它名下的全部活动告警。

接口（`controller/alert_rule.rs`，需 JWT）：`GET /api/vcenter/alert-rules`、`POST /api/vcenter/alert-rules`、`PATCH /api/vcenter/alert-rules/{uuid}`、`DELETE /api/vcenter/alert-rules/{uuid}`（预置规则返回 409）、`GET /api/vcenter/alert-rules/fields` 返回每个类别可用字段与单位（供前端下拉，数据来自 `objects?schema=1`）。前端在「告警」页加「规则」子页：列表、启停开关、新建 / 编辑对话框（类别、字段、比较、阈值、持续时间、级别、作用域）。

预置规则（首次 `ensure_schema` 时插入，`builtin=1`）与非指标型预置告警（REQ-MON-007）：

| 告警 | 实现 |
|---|---|
| 数据存储将满 | 规则：`ds.used_ratio > 0.85` 持续 300 s → warning；`> 0.95` 持续 60 s → critical |
| 主机文件系统将满 | 规则：`hostfs.used_ratio > 0.90` 持续 300 s → warning |
| 主机内存不足 | 规则：`host.mem_available` 低于总量 5% —— 用派生字段：hyper-agent 在 `host.v1` 里多推一个 `mem_available_ratio`（gauge），规则 `< 0.05` 持续 120 s → warning |
| 主机 CPU 持续满载 | 规则：`host.cpu_usage > 0.95` 持续 600 s → warning |
| 主机断连 | 不走指标：ferrovirt `host_manager` 在主机连接状态从 connected 变为 disconnected / not_responding 时 `raise`（category=host，critical），恢复 connected 时 `resolve`。management 与 standalone 都适用（standalone 下是本机 agent 断开） |
| 监控数据中断 | 评估器每拍查 `objects?class=host`：DB 里登记了 `machine_id` 的主机，其 `host/<machine-id>` 对象状态为 stale / retired → `raise`（warning，「主机监控数据中断」，键 `monitor_stale:host/<mid>`），回到 live → `resolve`。（实现时改用对象状态而不是拉 `events`：状态是当前值，不用处理事件窗口与重复） |
| HA 动作 | hyper-agent `support/ha_events.py` 订阅本机 conciliumd 事件流（UDS `events.subscribe`，级别 ≥ warning）并 PUB `alert.raised`（category=ha，一次性事件告警，键带事件时间戳，用户确认后进历史）；ferrovirt 已订阅 `alert.*`，无需改 ferrovirt。覆盖 `resource.unexpected_stop` / `resource.start_failed` / `resource.fenced` / `node.fenced` / `node.fence_failed`（critical）与 `resource.moved`（warning）。**`resource.placed` 不报**：执行器每次拉起都发它（`prev_state` 恒为 `Assigned@本节点`），本地分不出正常启动与故障后重拉。concilium 事件只发给本节点订阅者，所以各节点各报自己执行的动作，center 汇总 |
| 硬件故障 | 已有：SMART（hyper-agent `support/alert_monitor.py`）。IPMI SEL 不在 1.0 范围 |
| Petrel 数据存储将满 | 规则：`petrelpool.used_ratio > 0.85` 持续 300 s → warning；`> 0.95` 持续 60 s → critical（与普通数据存储同阈值；Petrel 满了 OSD 会拒写，所以单独成规则而不是靠 `ds`） |
| Petrel 集群异常 | 规则：`petrel.health >= 1` 持续 120 s → warning；`>= 2` 持续 60 s → critical |
| Petrel OSD 离线 | 规则：`petrelosd.up < 1` 持续 120 s → warning（对象粒度，一个 OSD 一条告警，标题带 OSD id 与所在主机） |

#### 16.3.7 打包与部署

1. **kestrel deb**：在 `xvirt/build-deb-xvirt/` 新增包 `xvirt-kestrel`（`packages/xvirt-kestrel/{control.in,postinst,conffiles}`，`build-xvirt-debs.sh` 增加 `build_kestrel`），内容：`/usr/bin/kestreld`、`/lib/systemd/system/kestreld.service`（来自 `kestrel/etc/kestreld.service`）加 drop-in `kestreld.service.d/xvirt.conf`（`User=kestrel`）、`/etc/kestrel/kestreld.conf`（conffile，默认 `http = 127.0.0.1:7411`）、`/usr/lib/python3/dist-packages/kestrel.py`。postinst 建系统用户 `kestrel` 与 `/var/lib/kestrel`（0750），配置文件 0640 root:kestrel（里面有口令）。**postinst 不启动服务**：要不要跑 kestreld 由角色决定，交给 xvirt-init。CI（`build-deb/ci/Jenkinsfile.deb-xvirt`）增加拉取 kestrel 仓。
2. **xvirt-meta**：`Depends` 加 `xvirt-kestrel`（CI 离线包不装 Recommends），`Recommends` 去掉 `prometheus, prometheus-node-exporter`；现有的 `xvirt-archive-keyring`（Depends）与 `xvirt-fledge`（Recommends，网关用，xvirt `521918f` 加入）保持不变。hyper-agent、lictor 的 deb `Depends: xvirt-kestrel`（为了 Python SDK）。
3. **xvirt-init**：
   - `phase_enable_monitoring` 改为：management / standalone 才执行；首次运行生成 32 字节随机口令，写入 `/etc/kestrel/kestreld.conf` 的 `token =`，并把 `http` 改为 `127.0.0.1:7411`；`en kestreld`。standalone 同时写 `/etc/xvirt/kestrel.conf`（`KESTREL_ADDR=127.0.0.1:7410`、`KESTREL_TOKEN=<口令>`，权限 0600）。
   - 不再 enable prometheus / prometheus-node-exporter。
   - `phase_firewall`：management / standalone 放行 `7410/tcp`；删除 `9090/tcp`、`9100/tcp`；不放行 7411、9101。
   - 文件头注释里的服务清单同步更新。
4. **compute 节点拿到 center 的地址与口令**：在纳管流程里下发，不另开通道。
   - center 侧：`host_admin::run_add_host_worker` 发 ENROLL 时在请求体里加 `kestrel: {addr: "<center 管理 IP>:7410", token: "<center kestreld 口令>"}`（center 从本机 `/etc/kestrel/kestreld.conf` 读口令，管理 IP 取 ferrovirt 已有的本机地址配置）。
   - compute 侧：`controller/internal.rs` 的 `op_enroll` 收到后写 `/etc/xvirt/kestrel.conf`（0600，原子替换），ENROLL 响应里带 `monitoring: {status}` 供 center 记日志；`op_unenroll` 后：**standalone 节点**（本机自己跑 kestreld）恢复推给本机 `127.0.0.1:7410`（口令取本机 kestreld.conf），**compute 节点**删除该文件停推。hyper-agent / lictor 30 s 内发现变化并开始 / 停止推送（§16.3.4）。被纳管的 standalone 节点因此在纳管期间推给 center、本机界面的监控曲线会断（数据在 center 上看），解除纳管后恢复。实现见 ferrovirt `service/kestrel_conf.rs`。
   - 已经纳管的 compute 节点：center 升级后执行一次 `POST /api/vcenter/host/{uuid}?action=resync-monitoring`（重发 kestrel 配置，node_zmq 新操作 `monitoring.set`，用 `host_membership.sync_token` 鉴权，不要 root 口令；payload 的 `kestrel` 为 null 时按解除纳管处理）。管理控制台主机右键菜单「重新下发监控配置」调它。
   - 依赖 xvirt `bed0917`（compute 节点运行 node 模式 ferrovirt）。
5. **ferrovirt.env**：`FERROVIRT__METRICS__KESTREL_URL=http://127.0.0.1:7411`，删除 `PROMETHEUS_URL` 行。

#### 16.3.8 实施分期

| 期 | 内容 | 涉及仓 | 完成标准 |
|---|---|---|---|
| K0 前置验证与打包 | Python SDK 对着 kestreld 端到端跑通（声明、注册、推送、断线重连、口令校验、GONE）；`xvirt-kestrel` deb；xvirt-init / xvirt-meta / 防火墙改动；center 系统盘容量估算写进 §16.3.1 | kestrel、xvirt、build-deb | 新 ISO 装出的 standalone 节点 `kestreld` 运行、`/api/v1/stats` 有响应、9090/9100 不再放行 |
| K1 生产者 | hyper-agent `kestrel_push` + `kestrel_sink`（host / hostfs / hostnic / ds / guest / guestfs / plan / vmtier / petrel / petrelpool / petrelosd）；lictor `vm_metrics`（vm / vmdisk / vmnic，含 Petrel 盘）；petrel-vhost 每卷计数文件；9101 默认关闭 | hyper-agent、lictor、petrel | standalone 节点上 `objects` 列出上表全部类别，起停一台 VM 后 `vm` / `vmdisk` / `vmnic` 对象出现并有数据；热插一块盘出现新的 `vmdisk` 对象；Petrel 盘的 VM 里 `dd` 时其 `vmdisk` 有写入速率；有 Petrel 集群的节点上三类 Petrel 对象齐全 |
| K2 ferrovirt 查询 | `kestrel_client`；`resource_metrics` 改写（含 VM 磁盘新键、Petrel 数据存储容量）；`datastore.petrel_pool` 列与回填；`source_error`；删除 PromQL 透传；`/api/vcenter/overview`；前端 VM 磁盘图与概览页接新接口 | ferrovirt、xvirt-client | 主机 / VM / 数据存储三个监控页在全新节点上有曲线；VM 不装 xvirt-tools 也有 CPU、磁盘、网络曲线；停 kestreld 时页面显示「监控服务不可用」 |
| K3 告警 | `alert_rule` 表、评估器、接口、前端规则子页；预置规则；主机断连、监控中断、HA 动作告警 | ferrovirt、hyper-agent、xvirt-client | §16.3.9 的告警用例全部通过 |
| K4 多节点 | ENROLL 下发 kestrel 配置、`op_enroll` / `op_unenroll` 写删 `kestrel.conf`、`resync-monitoring`；center 汇总概览 | ferrovirt | center 纳管一台 compute 节点后，center 界面上该节点与其 VM 有曲线；解除纳管后该节点停止推送 |

K0 → K1 → K2 可以按顺序由不同人接力；K3 依赖 K1（要有数据）；K4 依赖 K2 与 xvirt `bed0917` 的 compute 节点验证。

**进度（2026-11-23）**：K4 代码已写，**未编译、未验证**：ferrovirt `service/kestrel_conf.rs`（center 读本机 kestreld.conf 拼推送目标；节点写 / 复位 `/etc/xvirt/kestrel.conf`）、ENROLL 带 `kestrel`、`op_enroll` / `op_unenroll` 写 / 复位、node_zmq `monitoring.set`、`action=resync-monitoring`；xvirt-client 管理控制台主机右键「重新下发监控配置」。center 概览汇总全部纳管主机在 K2 已实现（kestreld 收的就是所有纳管主机）。K0–K4 代码全部写完，整体待测试机验证（§16.3.9）。

K3：代码已写，**未编译、未验证**：ferrovirt `service/alert_rule.rs`（表 + 10 条预置规则 + CRUD + 每 30 s 评估器 + 主机断连 + 监控数据中断）与 `controller/alert_rule.rs`（`/api/vcenter/alert-rules[/fields|/{uuid}]`）、`host_manager` 连接状态翻转时出 / 消断连告警（只在 connected→disconnected 那一次 raise）；hyper-agent `support/ha_events.py`；xvirt-client 告警页「告警规则」子页（列表、启停、新建 / 编辑 / 删除）；xvirt-autotest stage 35 `metrics.*`（overview / host_series / datastore_series / source_error*）与 stage 36 `alertrule.*`（crud / fire_and_resolve / scope / builtin_ds_full* / host_disconnect*），已在 Cairn 套件 TS-FS-XVIRT-AUTOTEST 登记并按上表 verifies 关联。§16.3.9 里需要 VM 或特定环境的 `metrics.vm_series_no_tools` / `vm_disk_io` / `petrel_*` 与 `alertrule.ha_action`、`alertrule.petrel_osd_down` 还没写用例。

K2：代码已写，**未编译、未验证**：ferrovirt `service/kestrel_client.rs`、`service/resource_metrics.rs` 改写（含 VM 磁盘新键、Petrel 数据存储容量、`source_error` / `series_errors`）、`service/overview.rs` + `GET /api/vcenter/overview`、`host.machine_id`（`connect_host` 后经 `host_identity` 回填）、`datastore.petrel_pool`（建 Petrel 数据存储时写入，已有行按哨兵挂载点 `/petrel/<池>` 回填）、`GET /host/{id}/cpu-busy`、删除 PromQL 透传、配置 `KESTREL_URL`（`PROMETHEUS_URL` 读到时告警并忽略）；xvirt `assets/ferrovirt.env`；xvirt-client VM 监控页加磁盘 IOPS / 吞吐 / 延迟三张图、三个监控页状态栏显示监控不可用、资源概览卡（单机主机概览页顶部 + 管理控制台根节点摘要）、主机概览页的 CPU / 内存 / 网络曲线从占位假数据换成真实数据、资源规划页改用 `cpu-busy`。compute 节点本机不跑 kestreld，其节点模式 ferrovirt 的监控接口会返回 `source_error`（界面在 center 上看）。

K1：代码已写，**未编译、未验证**：hyper-agent `support/kestrel_push.py`（与 lictor 同一份）、`metrics_exporter/kestrel_sink.py`（host / hostfs / hostnic / ds / guest / guestfs / plan / vmtier / petrel / petrelpool / petrelosd）、action `host_identity`、:9101 默认关闭（`XVIRT_PROM_EXPORTER=1` 才开）；lictor `monitor/vm_metrics.py`（vm / vmdisk / vmnic，含 Petrel 盘）、`delete_vm` 后注销；petrel `bins/petrel-vhost/src/stats.rs` 每卷计数文件。VM 删除时 hyper-agent 推的 `guest` / `guestfs` / `vmtier` 不注销，停推后失联、按 `retire_after` 退役。

K0：代码已写，**未编译、未验证**（构建机不稳定，验证等测试环境）：xvirt `packages/xvirt-kestrel`（control / postinst / conffiles）+ `assets/kestreld-xvirt.conf`（`User=kestrel` drop-in）+ `build-xvirt-debs.sh::build_kestrel`；xvirt-meta 与 hyper-agent / lictor 依赖；xvirt-init `phase_enable_monitoring`（角色判断、口令只生成一次、standalone 写 `/etc/xvirt/kestrel.conf`，排在 `enable_compute` 之前）与防火墙（放行 7410，收回 9090 / 9100）；build-deb `Jenkinsfile.deb-xvirt` 拉 kestrel 仓；kestrel `tests/sdk_python_e2e.sh`（Python SDK 端到端，待跑）。`assets/ferrovirt.env` 的 `PROMETHEUS_URL` 留到 K2 与 ferrovirt 代码一起改，K0 不动。

#### 16.3.9 验收用例

在 xvirt-autotest 增加 `stage 35 metrics.*` 与 `stage 36 alertrule.*`（用 `@testcase` 注册，默认运行；需要造故障的标 gated），并在 Cairn 中以 `verifies` 关联到对应 AC：

| 用例 | 断言 | 关联需求 |
|---|---|---|
| `metrics.kestrel_up` | ferrovirt 所在机 `127.0.0.1:7411/api/v1/stats` 可达；外部访问 7411 被拒 | — |
| `metrics.host_series` | `GET /api/vcenter/host/{id}/metrics?range=10m` 的 `cpu_usage_ratio`、`memory_used`、`fs_used`、`net_rx_bps` 均非空，且时间点间隔 ≤ 30 s | REQ-MON-001 |
| `metrics.host_history` | `range=24h`（节点运行满 1 天后）返回聚合层数据，点数 > 0 | REQ-MON-001 |
| `metrics.vm_series_no_tools` | 起一台不装 xvirt-tools 的 VM，5 分钟后 `cpu_usage_ratio`、`net_rx_bps`、`disk_iops_write`、`disk_lat_write_ms` 非空 | REQ-MON-002 |
| `metrics.vm_disk_io` | VM 内 `dd` 写 512 MiB 期间 `disk_bps_write` 峰值 > 10 MB/s | REQ-MON-002 |
| `metrics.datastore_series` | 数据存储监控页 `size_used`、`iops_write` 非空 | REQ-MON-001 |
| `metrics.source_error` | 停 kestreld 后接口返回 `source_error`，恢复后消失（gated） | — |
| `metrics.overview` | `/api/vcenter/overview` 的主机数、VM 数与 DB 一致；CPU / 内存 / 存储使用率在 0–100% 之间；告警数与 `/alerts/summary` 一致 | REQ-PLT-011 |
| `alertrule.crud` | 新建、修改、禁用、删除自定义规则；删除预置规则返回 409 | REQ-MON-006 |
| `alertrule.fire_and_resolve` | 新建规则 `host.cpu_usage > 0`、持续 30 s、作用域为被测主机 → 90 s 内出现活动告警；把阈值改为 `> 2` → 90 s 内告警 resolved 并进历史 | REQ-MON-006 |
| `alertrule.scope` | 作用域为另一台主机的同样规则不在被测主机上产生告警 | REQ-MON-006 |
| `alertrule.builtin_ds_full` | 临时把预置「数据存储将满」warning 阈值改成 0.0 → 告警出现；改回 → resolved | REQ-MON-007 |
| `alertrule.host_disconnect` | 停被测节点 hyper-agent → 主机断连告警出现；恢复 → resolved（gated，management 环境） | REQ-MON-007 |
| `alertrule.ha_action` | 多节点集群中触发一次 HA 重启 → 出现 HA 动作告警（gated，需要 quorum 环境） | REQ-MON-007 |
| `metrics.petrel_objects` | 有 Petrel 集群时 `objects?class=petrel|petrelpool|petrelosd` 非空，OSD 对象数等于 `petrel_cluster_status` 的 OSD 数 | REQ-STO-019 |
| `metrics.petrel_vmdisk_io` | Petrel 盘的 VM 内 `dd` 写 512 MiB 期间该盘 `disk_bps_write` 峰值 > 10 MB/s | REQ-MON-002 |
| `metrics.petrel_datastore_capacity` | Petrel 数据存储监控页 `size_total`、`size_used` 非空，与 `petrel_cluster_status` 池用量误差 < 5% | REQ-MON-001 |
| `alertrule.petrel_osd_down` | 停一个 OSD 进程 → 120 s 后出现 OSD 离线告警；恢复 → resolved（gated，破坏性） | REQ-MON-007、REQ-STO-019 |

#### 16.3.10 不在 1.0 范围

- kestrel 的跨对象聚合查询、直方图类型、传输加密、查询鉴权（kestrel README「未实现」）；1.0 用「生产者另推汇总对象 + ferrovirt 用 `latest` 汇总」与「查询只监听本机」绕开。
- 告警通知（邮件 / Webhook，REQ-MON-008）只依赖 `alert` 表，与本节正交，单独实现。
- petrel / fulmar / cdplog / qemu / concilium 通过 Rust / C SDK 直接推送自身指标；1.0 只接 hyper-agent 与 lictor（Petrel 集群状态由 hyper-agent 代推，Petrel 盘 IO 由 lictor 读 petrel-vhost 计数文件代推）。
- Petrel 池级 IO、PG 降级 / 恢复进度、缓存层命中率；fulmar 文件系统（fulmard）的任何指标；fledge 网关自身状态。
- ebpf_exporter（biolatency / cfs-throttling / runqlat）的数据；后续由 hyper-agent 读 BPF map 推送。
- 历史数据迁移：手工监控机上的 VictoriaMetrics 数据不导入 kestrel。

## 17. 打包、安装与部署

目标系统是 **Ubuntu 26.04**，以 deb 包、systemd unit 和 `xvirt-init` 首启部署为主。`xvirt/PACKAGING.md`、`xvirt/cicd/`、`xvirt/README.md` 仍是 Rocky 10.1 / RPM / virt-control / krait 时代的内容，只能当历史看。

| 环节 | 现状 |
|---|---|
| QEMU 包 | `build-deb` 打 `stable-11.1-dev`，`qemu-kvm-xvirt 11.1.1`，epoch 2（压过 Ubuntu 归档版），9 个 deb，带 `/usr/libexec/qemu-kvm` 软链 |
| 平台包 | `xvirt/build-deb-xvirt/build-xvirt-debs.sh`：xvirt-init、xvirt-lictor、xvirt-hyper-agent（含 `xvirt-resource-plan.service`）、xvirt-ferrovirt、xvirt-petrel、xvirt-concilium、xvirt-client、xvirt-meta。不再打 virt-control；监控栈未打包 |
| 首启 | `xvirt-init` 读 `/etc/xvirt/role.conf`，按角色建网桥、存储池、数据库，enable 服务并配置 ufw |
| CI | Jenkins `http://192.168.60.112:8081`，流水线 `deb-qemu`、`deb-xvirt`、`iso-xvirt`；构建容器 `xvirt-deb-builder:26.04`（112 上离线 apt 镜像）；apt 仓 `http://192.168.60.105:8083/xvirt-apt` |
| ISO | `lycan`：离线 TUI 安装器（替代 Subiquity），`scripts/build-iso.sh`，应答文件 `answers/{standalone,management,compute}.yaml`，bundle 为自产 deb 加离线依赖闭包 |
| PXE | `fledge`：按 MAC 派发 iPXE（:8068），静态 IP / VLAN / 磁盘 / 角色无人值守装机 |
| 自动化测试 | `xvirt-autotest`：package → iso → PXE provision → center / storage / network / vm（含 memtier）；最近一次全量 2026-06-28，71 通过 / 0 失败 / 5 跳过 |
| legacy | `krait`（Rocky/Anaconda 安装向导）、`build-rpm`、`xvirt.sh` |

开发/运维脚本仍是 `xvirtctl`：

```bash
./xvirtctl list
./xvirtctl status [comp]
./xvirtctl start|stop|restart [comp|all]
./xvirtctl build <comp>
./xvirtctl deploy <comp|all>
./xvirtctl logs <comp> [-f]
./xvirtctl role [center|standalone]
```

服务启动顺序：

```text
xvirt-resource-plan（oneshot，sysinit 阶段） -> lictor -> hyper-agent -> ferrovirt -> concilium -> nginx
```

unit 依赖：hyper-agent `After=xvirt-lictor openvswitch-switch`；ferrovirt `After=mariadb xvirt-hyper-agent`。`xvirtctl` 还不认识 `xvirt-resource-plan`。

关键端口：

| 端口 | 组件 |
|---|---|
| 80 / 443 | nginx |
| 8080 | ferrovirt |
| 5570 / 5571 | ferrovirt center↔节点（C1 命令 ROUTER / C2 事件 PUB） |
| 5556 | hyper-agent ROUTER |
| 5557 | hyper-agent file |
| 5558 | hyper-agent event PUB |
| 5560 | lictor ROUTER |
| 5561 | lictor event PUB |
| 7700 | concilium gossip |
| 6790 | petrel mon |
| 6800–6820 | petrel osd |
| 49152–49200 | 在线迁移（QEMU `-incoming`） |
| 6900 | xdesk 控制通道（默认） |
| 8428 / 8429 | VictoriaMetrics / vmagent |
| 9101 / 9435 | hyper-agent exporter / ebpf_exporter |
| 3000 / 8880 / 9093 / 9094 | Grafana / vmalert / Alertmanager / 飞书 adapter |
| 9090 / 9100 | prometheus / node-exporter（旧 deb 口径；K0 起 xvirt-init 不再启用、并收回放行） |
| 7410 / 7411 | kestreld 推送（management / standalone 放行）/ 查询（只听 127.0.0.1）；K0 代码已写未验证 |
| 5173 | Vite（仅开发） |

fulmard 不占 TCP 端口：控制 API 走本机 unix socket，数据走共享内存环；集群模式另有节点间 TCP（成员、租约、DLM），端口待定。

## 18. 技术特点总结

1. 控制面与执行面分离：ferrovirt 只做资源、任务、DB 和编排，agent 和底层组件执行宿主机动作。
2. 不依赖 libvirt 管理 VM：底层直接拼 QEMU 命令行、直接持 QMP，但路径和 XML/argv 尽量兼容 libvirt 习惯。
3. QMP 所有权集中：lictor 是 QEMU/QMP 唯一持有者；QEMU 进程放在独立的 systemd scope 里，与 lictor 的生命周期解耦。
4. ZMQ 作为内部 RPC/事件总线：请求用 DEALER/ROUTER，事件用 PUB/SUB；center 与节点之间同样走 ZMQ。事件为主，周期对账兜底。
5. 资源句柄 UUID 化：前端对象引用统一用 uuid，name 只显示。
6. 任务模型显式：写操作异步化，WebSocket + 轮询兜底。
7. 主机资源显式规划：启动参数、cgroup slice、放置控制器三层，VM 分独占 / 节点共享 / 浮动，自动装箱和在线 NUMA 迁移。
8. 快照/备份深度自研：QEMU bitmap export/import、Aegis SDK、本地 NBD、CBT change-point 入库、内存分层快照、带快照树的跨存储迁移、CDP 日志引擎。
9. 高级虚拟化能力：热度 pre-copy 在线迁移、热插拔、reflink 快照、分布式存储、内存分层超分、SR-IOV 模型、原生远程桌面。
10. 部署链完整：组件 deb、离线 ISO、PXE、自动化测试、监控告警都有配套项目。
11. 多语言与角色化控制台：前端支持 center/standalone 两套管理视角，并逐步国际化。
12. 去内核模块、去外部依赖的方向（规划中）：共享存储从内核 OCFS2 转向用户态 fulmar，监控从 VictoriaMetrics + exporter 转向服务自推的 kestrel，两者都只用稳定的内核接口与 libc。

## 19. 主要风险与待办

7 月版里已解决的几项：
- build-deb-xvirt 已打 ferrovirt，不再打 virt-control。
- 周期性全量 VM state 对账已实现（`state_reconcile.rs`）。
- network 句柄已改为 uuid 优先解析（是否所有引用点都已切换未逐一确认）。
- virt-control 已从工作区移除。

仍存在或新发现的：

- **CBT 位图导出/导入是重写版，缺真 VM 验证**：10.0 的原实现丢失，2026-11-20 在 11.1 上重写并合入 `stable-11.1-dev`（`b63be76397`），iotests 和调用方兼容测试都过了。出包部署前还要经 lictor 在真 VM 上走一遍运行中导出和开机复位（§11.1）。
- **文档代际差异更大了**：`xvirt/README.md`、`PACKAGING.md`、`cicd/`、`PIPELINES.md`、`build-deb-xvirt/README.md` 仍是 virt-control / RPM 口径；`xvirt/CLAUDE.md` 还引用已不存在的 `virt-control/` 代码，"VM state 对账未做""仍打 Java virt-control"这两句也已过时；`build-deb/README.md` 写"无 epoch"与脚本矛盾；`aegis-sdk/README.md` 的已知限制没更新。
- **监控口径不一致，全新 ISO 没有任何时序库**（2026-11-22 核对，§16.0）：1.0 已定改用 kestrel，接入方案见 §16.3；接入完成前 Cairn 1.0 的 REQ-MON-001/002/006/007、REQ-PLT-011 保持阻塞。
- ~~**xvirt-client deb 可能打不出来**~~：已在 xvirt `39edb7d`（2026-11-19）补齐 `packages/xvirt-client/{control.in,postinst,conffiles}`（原判断基于落后的本地 clone）；仍未实际构建验证。另：全部 APT 源都是 `[trusted=yes]`，没有 GPG 签名。
- **纯 compute 角色不启 ferrovirt**（2026-11-22 核对属实）：xvirt-init 对 compute 不启 MariaDB 与 ferrovirt，center 经 5570 ENROLL / PROBE 被拒，纯 compute 节点无法纳管；已在 xvirt `bed0917` 修复（compute 也启 MariaDB + node 模式 ferrovirt，`xvirtctl role` 改写 `ROLE=` 行），待出 ISO 后由 center 纳管一台 compute 节点验证。另：经注册 token 纳管的路径不存在，只有地址 + root 凭据。
- **concilium 迁移路径不通**：RA → `lictor migrate` → `migrate_vm` 未实现；conciliumd unit 没有 `After=xvirt-lictor`，只靠 `xvirtctl` 的启动顺序。
- **`xvirtctl` 不管 `xvirt-resource-plan`**。
- **SR-IOV VF 分配未接入 VM 创建**。
- **前端 i18n 未完成**：表单和向导正文仍有大量硬编码中文。
- **大量能力处于"已实现未验证"**：资源规划与放置、内存分层快照、CDP 改名、`memtier-shared-backing` 跨主机迁移都缺真机端到端验证。
- **兼容矩阵缺失**：memtier、petrel、热插拔、在线迁移、资源规划（cgroup v2 cpuset 分区、RDT、hugetlb）都依赖 QEMU / 内核 / 硬件条件，没有统一的兼容矩阵文档。
- **控制面尚未接入 CDP、xdesk、memtier 热度直方图**。
- fulmar 代码未编译未测试，COW 引用计数与 ENOSPC 的正确性是主要风险（设计稿要求模型级单测 + 模糊测试 + 从 P0 起就有 fsck）；集群模式依赖阵列的 ATS / SCSI PR 支持，要做能力探测，不支持就拒绝集群挂载。
- kestrel 接入（§16.3）要改 ferrovirt `resource_metrics.rs`（PromQL → kestrel 查询接口）、新增 ferrovirt 告警规则引擎、hyper-agent / lictor 改为推送；kestrel 的 Python SDK 还没对着服务端跑过，按类别保留的规模表现也没测，这两项是 K0 阶段的前置验证。

## 20. 相关文档

| 主题 | 文档 |
|---|---|
| 仓库地址与分支 | `xvirt-repository-index.md` |
| 资源规划 | `docs/resource-planning.md`、`docs/smart-numa-design.md` |
| memtier | `docs/memory-tiering.md`、`docs/qemu-memtier-interface.md`、`docs/qemu-memtier-usage.md`、`docs/memtier-*.md`、`qemu-memory-tiering-tech-design.md` |
| 内存快照 | `docs/memtier-snapshot-design.md`、`docs/memtier-snapshot-restore-design.md` |
| 存储迁移 | `docs/drive-mirror-sparse-thick-design.md`、`xvirt-blockdiff/README.md`、`xvirt-snapmig/README.md` |
| CDP | `cdp/README.md`、`cdp/docs/log-engine.md`、`cdp/cdplog/README.md`、`cdp/docs/qemu-cdp.md`、`docs/cdp-rename-changes.md` |
| 监控 | `xvirt/monitoring/README.md`、`docs/kestrel-design.md`、`docs/vm-direct-metrics-design.md`、`docs/memtier-monitoring-design.md` |
| 文件系统 | `docs/fulmar-design.md` |
| 远程桌面 | `qemu-virtual-desktop-protocol-design.md`、`xdesk-client/README.md` |
| center 纳管 | `ferrovirt/docs/host-enrollment.md` |
