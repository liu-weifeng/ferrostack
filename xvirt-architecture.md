# xvirt 架构梳理

生成时间：2026-07-18  
来源：192.168.60.112:/root/code/xvirt，重点参考 `CLAUDE.md`、`README.md`、`PACKAGING.md`、各组件 README/DESIGN 与源码目录。  
更新：2026-11-20，加入两个新组件——推送式监控框架 **kestrel**（§14.2，拟替代 VictoriaMetrics + exporter）和用户态共享盘文件系统 **fulmar**（§8.1，拟替代 OCFS2）。两者都还没接进 hyper-agent / ferrovirt / QEMU，本文按设计稿描述目标形态，并标出当前实现状态。

## 1. 项目定位

xvirt 是一套自研服务器虚拟化平台，整体目标对标 vSphere / Proxmox。项目不是单一服务，而是一个多仓/多组件集成工作区：上层提供 Web 管理控制台、REST/WS API、资源模型和任务系统；下层直接驱动 QEMU、宿主机存储/网络/硬件扫描、QMP、迁移、快照、备份和监控。

当前权威状态以顶层 `CLAUDE.md` 为准：Rust 控制面 `ferrovirt` 已取代旧 Java `virt-control`；`hyper-agent` 不再直接持有全部 VM 运行时原语，运行中 QEMU/QMP 能力拆到 `lictor`；`petrel`、`aegis-sdk`、`cdp-agent`、内存分层、自动化测试等形成围绕核心虚拟化平台的扩展生态。

## 2. 总体拓扑

```text
浏览器
  |
  | HTTP / WebSocket
  v
nginx :80/:443
  |-- 静态 xvirt-client SPA
  |-- /api/*       -> ferrovirt :8080
  |-- /websocket/* -> ferrovirt :8080
  |-- /vnc/<port>/ -> websockify / noVNC
  v
ferrovirt
  |  REST / WS / TaskManager / MariaDB(mx) owner
  |  ZMQ DEALER -> hyper-agent ROUTER :5556
  |  ZMQ SUB    <- hyper-agent PUB    :5558
  v
hyper-agent
  |  宿主机准备工作：存储、网络、硬件扫描、文件传输、petrel 托管
  |  VM/QMP/xvirt-tools 类 action 瘦代理到 lictor
  v
lictor
  |  QEMU/QMP 唯一持有者：启动、停止、快照、迁移、热插拔、console、xvirt-tools
  |  PUB :5561 -> hyper-agent relay -> PUB :5558
  v
QEMU + host kernel
  |
  | virtio-serial channel "xvirt-tools.0"
  v
guest 内 xvirt-tools

旁路/扩展：
concilium 集群资源管理 -> lictor CLI
petrel 分布式块存储，由 hyper-agent 托管守护进程
fulmar 用户态共享盘文件系统（规划中，替代 OCFS2）：每主机一个 fulmard，
       hyper-agent 经 unix socket 控制，QEMU 经 libfulmar 共享内存环读写
aegis-sdk / cdp-agent / cdp-worker 备份与 CDP
VictoriaMetrics / vmagent / ebpf_exporter / Grafana 监控（现行）
kestrel 推送式监控（规划中，替代上一行）：各服务内嵌 SDK 推到 kestreld :7410，
        ferrovirt / 界面经 HTTP :7411 查询
lycan/krait/fledge/build-deb-xvirt 打包部署与自动装机
```

## 3. 核心组件

| 组件 | 技术栈 | 角色 | 主要入口 |
|---|---|---|---|
| xvirt-client | React 18 / Vite | Web 控制台，支持 center 多主机管理与 standalone 单机界面 | nginx 静态托管，dev 为 :5173 |
| ferrovirt | Rust / axum / sqlx / ZMQ | 管理控制面、REST API、WebSocket、任务系统、DB 唯一 owner | :8080，`ferrovirt/src/main.rs` |
| hyper-agent | Python asyncio / pyzmq | 每台宿主机的底层 agent，做主机存储/网络/硬件/文件/petrel 等操作 | ROUTER :5556，PUB :5558，file :5557 |
| lictor | Python asyncio / pyzmq | QEMU/QMP 持有者，承接 VM 生命周期、快照、迁移、热插拔、xvirt-tools | ROUTER :5560，PUB :5561 |
| concilium | C daemon | 集群资源管理，驱动 `xvirt-vm` RA 到 lictor；单节点可 bootstrap quorum | `/run/conciliumd.sock` |
| petrel | Rust | 分布式存储，块卷经 vhost-user-blk 给 VM；由 hyper-agent reconcile | mon :6790 等 |
| fulmar | Rust（libfulmar 为 C ABI） | 用户态共享盘文件系统，替代 OCFS2 做共享 datastore；COW + O(1) reflink 快照，本地/集群同格式 | `fulmard` 控制 socket，`fulmar:` 块驱动；见 §8.1 |
| QEMU | C，自研改造 | 虚拟化底座，含 CBT 位图导出/导入、memtier、迁移相关改造 | qemu 工作树 |
| xvirt-tools | Go | guest VM 内 daemon，经 virtio-serial 与 host 通信 | guest 内服务 |
| aegis-sdk | C library + CLI | 类 VDDK 的备份 SDK，提供磁盘读写/CBT 能力 | `libaegisdisk.so`、`aegis-cbt` |
| cdp-agent | Rust + C 内核驱动 | guest 内 CDP，拦截写 IO 并流出到 sink | agent -> sink TCP |
| monitoring | VictoriaMetrics 生态 | 宿主机/VM/数据存储指标、告警、Grafana（现行） | vmagent :8429，vmsingle :8428 |
| kestrel | Rust 服务端 + Rust/Python/C SDK | 推送式轻量监控，拟替代上一行；对象 + 指标集模型，分层精简，保留突发 | 推送 :7410，查询 :7411；见 §14.2 |

## 4. ferrovirt 控制面

`ferrovirt` 是当前主控制面，已替代旧 Java `virt-control`。它拥有 MariaDB `mx` 数据库，agent 不直接碰 DB。源码结构显示其模块按 controller / service / entity / repository 分层：

- `controller/*`：REST 入口，覆盖 auth、account、host、vm、datastore、network、task、alert、aegis、vcenter 等。
- `service/task_manager.rs`：异步任务管理。写操作、扫描、VM 电源、创建/编辑、快照等通过任务化执行。
- `service/vm_create.rs`、`vm_update.rs`、`snapshot.rs`、`vm_migrate.rs`：VM 创建、编辑、快照、迁移编排。
- `service/host_sync.rs`、`storage_net.rs`、`host_manager.rs`：宿主机扫描、存储/网络同步、主机纳管。
- `node_zmq.rs`、`zmq.rs`、`compute_client.rs`：与 hyper-agent 的 ZMQ 通信。
- `websocket.rs`、`service/event.rs`：WebSocket 推送与 agent 事件处理。
- `auth.rs`、`jwt.rs`、`service/auth.rs`：PAM 登录 + JWT。

关键设计点：

- DB ownership 清晰：`ferrovirt` 是资源状态、任务、用户偏好、快照元数据的唯一写入者。
- API 返回与路径变量统一使用 UUID，`name` 只用于展示。
- 写操作多走 TaskManager，任务状态经 WebSocket 推送，前端也有轮询兜底。
- agent 事件经 ZMQ PUB/SUB 单向推送到 ferrovirt，用于实时更新 VM 状态和 UI 事件。
- REST 面向 vCenter 风格资源：host、VM、datastore、network、task、snapshot、metrics 等。

## 5. hyper-agent 宿主机 agent

`hyper-agent` 是每个宿主机上的底层服务。它监听 ZMQ ROUTER :5556，接收 `ferrovirt` 发来的 action 请求，主循环基于 `zmq.asyncio`，阻塞型 handler 放到 ThreadPoolExecutor 中执行。

当前职责边界：

- 保留：存储池、数据存储、网络、设备树扫描、单盘 wipe/rescan、文件传输、监控 exporter、petrel 守护进程托管、Aegis NBD 本地导出、memtier 主机级总控。
- 转发：VM 生命周期、运行中 QMP、快照、迁移、console、xvirt-tools 等瘦代理到 `lictor`。
- 不做：不调用 libvirt；不读写 MariaDB；不做上层资源决策。

源码上主要模块：

- `controller/{host,vm,storage,network,device,petrel,memtier,aegis,snapshot,xvirt_tools}.py`
- `support/datastore/*`：local、shared fs、zfs、nfs、iscsi、lvm、petrel 等数据存储适配。
- `support/{device_scanner,nic_scanner,storage_device_scanner,storage_adapter_scanner}.py`
- `support/petrel/*`：petrel cluster / supervisor / vhost / volume 管理。
- `metrics_exporter/collectors/*`：host、VM、datastore、zfs 指标。
- `event_publisher.py`：向 ferrovirt 侧发布 agent 事件。

## 6. lictor QEMU/QMP 层

`lictor` 的目标是把“直接管理 QEMU 虚拟机”的职责从 `hyper-agent` 中独立出来。它使用与 hyper-agent 类似的 ZMQ 服务框架，但语义上更像轻量 libvirt：持有 QMP、QEMU 运行时、xvirt-tools socket，并提供原子 VM 原语和复合操作。

核心职责：

- VM start / stop / reset / status / power 操作。
- 在线快照、reflink 快照、CBT 在线导出、快照删除/回滚。
- 在线迁移与存储迁移。
- 运行中设备热插拔：磁盘、网卡、CD-ROM、CPU、内存、网卡 link。
- QMP 异步事件监听并发布到 :5561，由 hyper-agent relay 到 :5558。
- 管理 virtio-serial channel 下的 xvirt-tools。

设计边界：

- ferrovirt 仍只连 hyper-agent，协议面不直接感知 lictor。
- concilium 可通过 lictor CLI 直接驱动 start/stop/status/migrate。
- lictor 是 QMP 唯一持有者，避免多个组件同时控制同一个 QEMU monitor。

## 7. concilium 集群层

`concilium` 是 C 写的集群资源管理守护进程，角色是做集群决策与资源动作触发。它驱动 `xvirt-vm` RA，最终通过 lictor CLI 执行 VM 生命周期与迁移动作。单节点可通过 `--bootstrap` 自成 quorum，保证单机也能走统一路径。

职责范围在文档中刻意收窄：启动、关闭、状态检查、跨主机迁移。编辑 VM、快照、console、xvirt-tools 等不经 concilium，避免集群层变成大而全的数据路径。

## 8. 存储体系

xvirt 存储体系同时支持传统本地/共享数据存储与自研分布式存储：

- local / xfs / ext4 / btrfs：由 hyper-agent 创建目录与文件，VM 磁盘通常是 qcow2/raw。
- zfs：agent 调用 zfs 工具创建 dataset。
- nfs / ocfs2 / gfs2 / iscsi / lvm：作为数据存储或块设备能力纳入扫描/管理。
- petrel：Rust 分布式存储，提供 mon/osd/vhost 等 CLI/守护进程，VM 可通过 vhost-user-blk 使用块卷。
- fulmar（规划中）：用户态共享盘文件系统，`fs_type=fulmar`，用来替代 OCFS2，见 §8.1。

一个重要历史约束：`data_store.uuid` 和 `/datastore/<segment>` 中的路径段不一定相同。VM 创建时 agent 需要的 `vm_datastore` 是 mount path 的最后一段，而不是 entity uuid。当前 ferrovirt/服务层会做适配。

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

与现有组件的衔接（设计稿 §十三）：hyper-agent 新增 `support/datastore/fulmar.py` 和 `-blockdev driver=fulmar` 生成；QEMU 新增两个驱动文件，迁移时调所有权交接；lictor 的快照 / 迁移改走 datastore 接口，不直调 `cp --reflink`；`fulmard` 的轮询核与环用大页从资源规划的存储池（现 `spdk` 池）划拨。OCFS2 迁移走现有跨 datastore 快照链迁移或 drive-mirror，OCFS2 只读保留到清空为止。

**当前状态（2026-11-20）**：子仓 `fulmar` main `f76e2fc`，11 个 crate + 4 个可执行 + libfulmar。本地模式的 mkfs / fulmard / 共享内存环 / libfulmar C ABI / CLI / fsck 链路代码已写完，**还没编译、没测试**；集群模式（P3）只有骨架；hyper-agent、QEMU、lictor 一侧都没动。分期：P0 本地原型 → P1 memtier + 工具链 + hyper-agent datastore → P2 性能档（vhost-user-blk、SPDK）→ P3 集群 → P4 运维。实现语言（骨架已按 Rust 写）、默认 `cow_chunk`、FUSE 是否可写、集群槽数、memtier 直通是否默认、SPDK 是否进本期，共六项待拍板（设计稿 §十七）。

## 9. 网络体系

网络资源模型以 `virtual_switch` 为核心，支持：

- `LINUX_BRIDGE`
- `OVS`
- `SRIOV`

agent 的 `nic_scanner.py` 通过 sysfs、`ovs-vsctl` 等探测网络状态。虚拟交换机记录 backend、uplink、SR-IOV PF/VF 信息。VM 创建时前端或控制面提交的 network 会被解析为实际 `source_dev`，避免 agent 端生成 XML 时出现空 bridge。

## 10. VM 生命周期关键链路

### 10.1 创建 VM

```text
xvirt-client -> ferrovirt POST /api/vcenter/vm
  -> TaskManager 创建异步任务
  -> ferrovirt 补全默认配置、分配 alias / PCI / channel / datastore path
  -> ZMQ 调 hyper-agent create_vm
  -> hyper-agent 做离线准备：目录、磁盘、config.xml、存储槽位
  -> ferrovirt 落库 vm / vm_config / vm_disk / vm_network_interface / vm_video
```

agent 端不补默认值，默认值由控制面负责。落库失败但 host 文件已创建时会返回失败任务，交由人工查看日志与目录清理。

### 10.2 启动 VM

```text
ferrovirt power?action=start
  -> hyper-agent
  -> concilium 或直连 lictor
  -> lictor 解析 config.xml / 生成 QEMU argv / Popen QEMU
  -> lictor attach QMP
  -> lictor 发布 vm.state_changed
  -> hyper-agent relay
  -> ferrovirt 写 DB state + WS 推前端
```

### 10.3 编辑 VM / 热插拔

VM PATCH 是稀疏 patch。设备列表使用 `add/remove/update` 三段。控制面先落 DB，再重新生成 agent config；运行中 VM 会额外下发 live ops 给 lictor。

已覆盖的 live ops 包括：

- `disk_attach` / `disk_detach`
- `nic_attach` / `nic_detach`
- `cdrom_change`
- `cpu_set`
- `mem_add`
- `nic_set_link`

限制主要在 QEMU/guest ACPI 配合、SATA 热插支持、预留 PCI root port、max_vcpu/max_memory 等启动时条件。

## 11. 快照、备份与 CBT

xvirt 的备份数据面围绕“创建快照/freeze -> 导出 dirty bitmap -> 按位图读静止镜像”展开，明确不用 QMP `drive-backup`，因为离线 VM 没有 QEMU 进程。

核心能力：

- QEMU 自研 `block-dirty-bitmap-export/import`，导出 QEMUBMP 位图文件。
- 在线 VM：lictor 执行 `stop -> export -> clear -> cont`，同窗口 reflink 出静止副本。
- 离线 VM：hyper-agent 通过 `qemu-storage-daemon` / `qemu-img` 导出位图。
- change-point 持久化到 ferrovirt DB 的 `vm_snapshot` / `vm_snapshot_disk`。
- aegis-sdk 提供 VDDK 风格 C API，支持控制面 NBD ticket 模式和本地模式。
- `aegis-cbt` CLI 可 parse/verify/blank/merge QEMUBMP。

离线 CBT 复位使用延迟模型：导出位图后生成 `<disk>.aegis-blank` 空白标记，VM 下次启动由 lictor 导入空白位图完成复位，避免 freeze 阶段直接修改源盘。

## 12. 内存分层 memtier

memtier 是跨组件能力：QEMU 内 per-VM worker 将客户机 RAM 冷页换出到块设备，用于内存超分。技术特点包括：

- 基于 `uffd-wp` + `io_uring`，不改 KVM。
- QEMU 提供 `x-memtier-*` QMP 接口。
- lictor 持有 QMP，hyper-agent 做主机级总控，根据优先级和压力调度。
- ferrovirt 和前端负责配置与编排。
- 与 petrel/vhost-user 场景存在互斥约束：memtier 需要匿名内存，petrel/vhost-user 通常需要共享 memfd。

已有实测结论显示，性能损耗主要由冷页访问率决定，而不仅是超分比例；在热集可驻留物理内存时可实现较大超分。

## 13. guest 与 CDP 生态

`xvirt-tools` 是安装在 guest VM 内的 daemon，host 通过 QEMU virtio-serial channel `xvirt-tools.0` 与其 NDJSON 通信，用于 guest 内操作、状态与扩展能力。

`cdp-agent` 是更独立的 guest 内 CDP 方案，Rust 用户态 + Linux `dm-cdp` / Windows `cdpflt` 过滤驱动，拦截写 IO 并通过 TCP 流出。它不依赖 QEMU，覆盖 VMware、裸机、KVM 等场景。接收端 `cdp-sink` 支持 iolog、disk、replica 等模式。旧 `cdp-worker` 仍作为 QEMU 侧/备份 worker 路径存在，具备 IO 日志、索引、位图和降级模型。

## 14. 监控与告警

当前监控栈已从 Prometheus + node_exporter 迁移到 VictoriaMetrics 体系：

```text
hyper-agent exporter :9101
ebpf_exporter :9435
  -> vmagent :8429
  -> remote_write
  -> vmsingle :8428
  -> ferrovirt /api/vcenter/metrics
  -> nginx
  -> xvirt-client
```

配套组件包括 Grafana、vmalert、Alertmanager、飞书 adapter。host 通用 OS 指标由 hyper-agent 自身读 `/proc` 暴露，VM/datastore/zpool 语义指标也由 hyper-agent exporter 提供。

### 14.1 现行方案的问题

拉模式要为每类数据配 exporter 和抓取目标；VM、盘、OSD 随时增减，要靠服务发现跟；降采样后突发只剩平均值；长期容量随保留期线性增长。这些是引入 kestrel 的原因。

### 14.2 kestrel 推送式监控框架（拟替代 VictoriaMetrics + exporter）

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
| vmalert + Alertmanager 告警 | 本版只有事件；告警放 kestreld 还是 ferrovirt 待定 |

**当前状态（2026-11-20）**：子仓 `kestrel` main `912e7c6`（含 v1.5 的按类别保留与片段永久保留）。2026-11-19 在测试机 10.8.1.195 上编译通过，57 个测试和 24 项端到端都过；1 万对象规模、长范围查询、冷查询并发都实测过，但这些是 v1.5 之前跑的，按类别保留加入后没重跑。还**没有任何生产组件接入 SDK**，现行监控栈不变。待拍板六项（设计稿 §13）：名字、默认分层与保留期、min/max 是否用 f32、与 VictoriaMetrics 并行还是直接切、告警放哪、数据目录放什么盘。

## 15. 打包与部署

当前部署以 deb 包、systemd unit 和 `xvirtctl` 为主。顶层 `PACKAGING.md` 仍保留 Rocky/RPM/ISO 的大量历史说明，但顶部备注指出新栈形态已经是 `ferrovirt + lictor + concilium + hyper-agent + xvirt-client + monitoring`。

开发/部署首选脚本：

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
lictor -> hyper-agent -> ferrovirt -> concilium -> nginx
```

关键端口：

| 端口 | 组件 |
|---|---|
| 80 / 443 | nginx |
| 8080 | ferrovirt |
| 5556 | hyper-agent ROUTER |
| 5557 | hyper-agent file |
| 5558 | hyper-agent event PUB |
| 5560 | lictor ROUTER |
| 5561 | lictor event PUB |
| 6790 | petrel mon |
| 8428 / 8429 | VictoriaMetrics / vmagent |
| 9101 / 9435 | hyper-agent exporter / ebpf_exporter |
| 7410 / 7411 | kestreld 推送 / 查询（规划中） |

fulmard 不占 TCP 端口：控制 API 走本机 unix socket，数据走共享内存环；集群模式另有节点间 TCP（成员、租约、DLM），端口待定。

## 16. 技术特点总结

1. 控制面与执行面分离：ferrovirt 只做资源/任务/DB/编排，agent/底层组件执行宿主机动作。
2. 不依赖 libvirt 管理 VM：底层直接拼 QEMU 命令行、直接持 QMP，但路径和 XML/argv 尽量兼容 libvirt 习惯。
3. QMP 所有权集中：lictor 成为 QEMU/QMP 唯一持有者，降低多组件并发控制风险。
4. ZMQ 作为内部 RPC/事件总线：请求使用 DEALER/ROUTER，事件使用 PUB/SUB。
5. 资源句柄 UUID 化：前端对象引用统一使用 uuid，name 只显示。
6. 任务模型显式：写操作异步化，WebSocket + 轮询兜底。
7. 快照/备份深度自研：QEMU bitmap export/import、Aegis SDK、本地 NBD、CBT change-point DB 持久化。
8. 具备高级虚拟化能力：在线迁移、热插拔、reflink 快照、分布式存储、内存分层超分、SR-IOV 模型。
9. 部署链完整：从组件 deb、离线 bundle、ISO、PXE、自动化测试到监控告警都有配套项目。
10. 多语言与角色化控制台：前端支持 center/standalone，两套管理视角，并逐步国际化。
11. 去内核模块、去外部依赖的方向（规划中）：共享存储从内核 OCFS2 转向用户态 fulmar，监控从 VictoriaMetrics + exporter 转向服务自推的 kestrel，两者都只用稳定的内核接口与 libc。

## 17. 主要风险与待办

- 文档存在代际差异：`README.md` 和 `PACKAGING.md` 中部分内容仍是旧 Java `virt-control` / RPM 形态，需要以后统一刷新。
- `build-deb-xvirt` 文档提示当前可能仍打 Java virt-control，未完全切 ferrovirt，打包前必须核对。
- network 的 virtual_switch 已有 uuid 字段，但 controller 还未完全切到 uuid。
- 周期性全量 VM state 对账尚未完成；事件链路异常时状态可能停在旧值。
- SR-IOV VF 分配模型已建，但接入 VM 创建路径仍是待办。
- 部分前端深层表单/向导正文仍有硬编码中文，i18n 尚未完全迁移。
- 旧组件/旁路组件较多，如 virt-control、旧 hyper-agent 残留、cdp-worker、krait/lycan/fledge 等，需要明确哪些是生产路径，哪些是历史/实验路径。
- memtier、petrel、热插拔、在线迁移都依赖底层 QEMU/内核/硬件条件，生产部署需要检查兼容矩阵。
- fulmar 代码未编译未测试，COW 引用计数与 ENOSPC 的正确性是主要风险（设计稿要求模型级单测 + 模糊测试 + 从 P0 起就有 fsck）；集群模式依赖阵列的 ATS / SCSI PR 支持，要做能力探测，不支持就拒绝集群挂载。
- kestrel 切换期要改 ferrovirt `resource_metrics.rs`（PromQL → kestrel 查询接口），并决定告警归属；在此之前现行监控栈不能拆。按类别保留的规模表现还没测。

## 18. 建议的后续文档拆分

建议以本文为总览，后续拆成几份更稳定的工程文档：

- `ARCHITECTURE.md`：总体架构与组件边界。
- `RUNTIME-FLOWS.md`：VM 创建、启动、编辑、快照、迁移、备份、事件同步链路。
- `DEPLOYMENT.md`：deb/ISO/xvirtctl/systemd/端口/角色。
- `DATA-MODEL.md`：DB 表、UUID 约定、datastore segment、任务模型。
- `OPERATIONS.md`：日志、调试命令、常见故障、监控告警。
