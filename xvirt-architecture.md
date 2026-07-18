# xvirt 架构梳理

生成时间：2026-07-18  
来源：192.168.60.112:/root/code/xvirt，重点参考 `CLAUDE.md`、`README.md`、`PACKAGING.md`、各组件 README/DESIGN 与源码目录。

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
aegis-sdk / cdp-agent / cdp-worker 备份与 CDP
VictoriaMetrics / vmagent / ebpf_exporter / Grafana 监控
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
| QEMU | C，自研改造 | 虚拟化底座，含 CBT 位图导出/导入、memtier、迁移相关改造 | qemu 工作树 |
| xvirt-tools | Go | guest VM 内 daemon，经 virtio-serial 与 host 通信 | guest 内服务 |
| aegis-sdk | C library + CLI | 类 VDDK 的备份 SDK，提供磁盘读写/CBT 能力 | `libaegisdisk.so`、`aegis-cbt` |
| cdp-agent | Rust + C 内核驱动 | guest 内 CDP，拦截写 IO 并流出到 sink | agent -> sink TCP |
| monitoring | VictoriaMetrics 生态 | 宿主机/VM/数据存储指标、告警、Grafana | vmagent :8429，vmsingle :8428 |

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

一个重要历史约束：`data_store.uuid` 和 `/datastore/<segment>` 中的路径段不一定相同。VM 创建时 agent 需要的 `vm_datastore` 是 mount path 的最后一段，而不是 entity uuid。当前 ferrovirt/服务层会做适配。

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

## 17. 主要风险与待办

- 文档存在代际差异：`README.md` 和 `PACKAGING.md` 中部分内容仍是旧 Java `virt-control` / RPM 形态，需要以后统一刷新。
- `build-deb-xvirt` 文档提示当前可能仍打 Java virt-control，未完全切 ferrovirt，打包前必须核对。
- network 的 virtual_switch 已有 uuid 字段，但 controller 还未完全切到 uuid。
- 周期性全量 VM state 对账尚未完成；事件链路异常时状态可能停在旧值。
- SR-IOV VF 分配模型已建，但接入 VM 创建路径仍是待办。
- 部分前端深层表单/向导正文仍有硬编码中文，i18n 尚未完全迁移。
- 旧组件/旁路组件较多，如 virt-control、旧 hyper-agent 残留、cdp-worker、krait/lycan/fledge 等，需要明确哪些是生产路径，哪些是历史/实验路径。
- memtier、petrel、热插拔、在线迁移都依赖底层 QEMU/内核/硬件条件，生产部署需要检查兼容矩阵。

## 18. 建议的后续文档拆分

建议以本文为总览，后续拆成几份更稳定的工程文档：

- `ARCHITECTURE.md`：总体架构与组件边界。
- `RUNTIME-FLOWS.md`：VM 创建、启动、编辑、快照、迁移、备份、事件同步链路。
- `DEPLOYMENT.md`：deb/ISO/xvirtctl/systemd/端口/角色。
- `DATA-MODEL.md`：DB 表、UUID 约定、datastore segment、任务模型。
- `OPERATIONS.md`：日志、调试命令、常见故障、监控告警。
