# xvirt Claude Memory 工作交接整理

生成时间：2026-07-18  
来源：`192.168.60.112:/root/.claude/projects/-root-code-xvirt/memory/`，已排除 `dev_credentials.md`。  
配套总架构文档：`xvirt-architecture.md`

## 1. 给后续 AI 的快速上下文

你接手的是 `/root/code/xvirt` 这套服务器虚拟化平台。当前权威入口是顶层 `CLAUDE.md` 和 Claude memory 目录。不要从零重新探索组件结构，先读：

- `/root/code/xvirt/CLAUDE.md`
- `/root/.claude/projects/-root-code-xvirt/memory/MEMORY.md`
- `/root/.claude/projects/-root-code-xvirt/memory/xvirt-components-and-xvirtctl.md`
- `/root/.claude/projects/-root-code-xvirt/memory/no-local-testing.md`
- `/root/.claude/projects/-root-code-xvirt/memory/git-commit-workflow.md`

核心架构状态：

- `ferrovirt` 是当前 Rust 控制面，已取代 Java `virt-control`。
- `hyper-agent` 是宿主机底层服务，但运行中 QEMU/QMP 类能力已经瘦代理到 `lictor`。
- `lictor` 持有 QMP，负责 VM 生命周期、热插拔、快照、迁移、console、xvirt-tools。
- `concilium` 是集群资源管理层，主要驱动 VM start/stop/status/migrate。
- `petrel` 是 Rust 分布式存储，VM 盘通过 vhost-user-blk 使用，守护进程由 hyper-agent 托管。
- `xvirtctl` 是当前首选服务管理工具，改哪个组件只 `./xvirtctl deploy <comp>`，不要无谓重建全栈。

重要工作习惯：

- 用户明确说过“不用在本地测试”：默认不要从本地控制机跑 autotest suite/dashboard/validation，除非用户明确要求。
- petrel + xvirt lab repos 的提交习惯：需要提交时直接 commit 到 main，不走 feature branch；但仍然只在用户要求时 commit，只在用户要求时 push。
- 遇到旧文档时注意代际差异：`README.md`、部分打包文档仍残留 Java `virt-control` / RPM 旧形态。

## 2. 环境与工具现状

### 2.1 实验环境

- 主要测试节点：`192.168.2.123`、`192.168.2.106`，root 密码记录在被排除的 credentials memory 中。
- center：`192.168.2.121`，当前已通过 API/自动化框架作为受管 VM 管理，而不是旧的手工 `xvirt-center-vm` systemd QEMU。
- gateway / PXE：`192.168.2.1`，运行 fledge PXE 服务。
- artifact repo：`192.168.2.107:8085`。

### 2.2 xvirtctl

`/root/code/xvirt/xvirtctl` 是 systemd 服务管理主工具：

- `./xvirtctl list`
- `./xvirtctl status [comp]`
- `./xvirtctl build <comp>`
- `./xvirtctl deploy <comp|all>`
- `./xvirtctl logs <comp> [-f]`
- `./xvirtctl role [center|standalone]`

服务顺序：`lictor -> hyper-agent -> ferrovirt -> concilium -> nginx`。

### 2.3 打包现状

- `build-deb/`：QEMU deb。
- `build-deb-xvirt/`：组件 deb / autoinstall ISO。
- 注意：memory 记录中 `build-deb-xvirt` 曾仍打 Java `virt-control`，新栈完全 deb 化还需要核对是否已补 `ferrovirt / petrel / concilium / aegis` 包。
- QEMU deb 已处理 epoch，避免 Ubuntu stock QEMU `1:10.2.1` 覆盖自研 QEMU。

## 3. 已完成的重要任务

### 3.1 组件重构与运行管理

已形成清晰的新栈：

- `ferrovirt`：Rust 控制面，DB owner。
- `hyper-agent`：宿主机 agent。
- `lictor`：QMP owner，必须先于 hyper-agent 启动。
- `concilium`：集群资源管理。
- `petrel`：分布式存储。
- `xvirt-client`：center/standalone 双角色前端。
- `aegis-sdk`：VDDK-like C SDK。

`xvirtctl` 已作为当前系统服务管理器使用，支持按组件精准 build/deploy/restart。

### 3.2 自动化测试框架 xvirt-autotest

已完成独立仓库 `xvirt-autotest/`：

- 纯 Python 标准库，无三方依赖。
- 覆盖 packaging -> ISO -> PXE -> center -> storage -> network -> VM。
- 默认 run 包括 bootstrap、center、host、inventory、enrollment、storage、network、vm、hotplug。
- Web dashboard `serve` 端口 :8910。
- 增加测试只需要在 `cases/` 下加一个 `@testcase` 函数。

关键进展：

- Stage 25 `bootstrap.*`：center 已改成“经 node standalone API 创建的受管 VM”，不是旧手工裸 QEMU。
- 真实 destructive cutover 已验证：新 center VM 安装、上线、纳管 123/106。
- Stage 65 `hotplug.*`：覆盖运行中磁盘、网卡、CD-ROM、CPU、内存热插拔，并检查 `"live applied N/N op(s)"`。
- 已知正常 env-skip：petrel-disk VM 的旧 V1 qcow2-chain snapshot、TSC/CPU 条件不满足的旧迁移路径、AHCI/SATA 运行时热插。

### 3.3 fledge PXE 自动部署

`fledge/` 已拆为独立仓：

- 运行在 gateway `192.168.2.1`。
- 上传 ISO 后可按 MAC 设置静态 IP、系统盘、角色，然后 netboot + autoinstall。
- dnsmasq PXE 以附加配置方式接入。
- 完整链路已在 106 上的 VM 验证。
- 后续目标是用于物理 106/123 重装。

### 3.4 lycan Ubuntu 26.04 自动安装 ISO

`lycan/` 是 Ubuntu 26.04 autoinstall installer：

- standalone / management ISO 已在 nested QEMU 中验证。
- 解决了 layered-casper、BIOS boot `ef02`、apt openssh、casper reboot bypass、串口/SSH harness 等坑。
- 后续仍需要更完整地打进 ferrovirt/lictor/petrel bundle，完成 join 与 VM perf test。

### 3.5 Ubuntu port / QEMU deb / 包管理修复

已完成或记录的关键修复：

- QEMU deb 加 epoch：`2:10.0.7+git...`，避免被 Ubuntu stock `1:10.2.1` 替换。
- 自研 QEMU 与 OVMF file conflict 已修：qemu-system-data 不再抢 `ovmf-generic` 的 edk2 descriptor JSON。
- `xvirt-meta` 版本 pin 已修：去掉对 7 个组件的精确 `= @VERSION@` 依赖，避免单组件 deploy 后 apt wedge。
- `/usr/libexec/qemu-kvm` symlink 已由打包处理。
- ferrovirt OVMF 路径已从 Rocky `/usr/share/edk2/ovmf/...` 改到 Ubuntu `/usr/share/OVMF/*_4M.fd`，UEFI VM 可启动。

### 3.6 VM 在线迁移与 CPU mode

已完成真实 VM 双向在线迁移验证：

- 123 -> 106：Debian 12 guest，shared petrel-dist vhost-user-blk，在线迁移成功，guest session 连续。
- 106 -> 123：反向同样成功。
- 关键组合：`cpu_mode=custom Haswell-noTSX` + `kvm-apic.vapic=off` + 两端自研 QEMU 版本一致。
- 旧结论“TSC 是硬件死墙”已推翻。TSC mismatch 对 QEMU 是 warning，Linux guest 通过 kvm-clock 吸收频率差。

已实现的保护：

- ferrovirt 在迁移前检查 host-passthrough VM 的源/目的 CPU model。
- 如果 CPU model 不同则提前 400 拒绝，并提示改用 custom CPU。
- custom-mode VM 不被 TSC mismatch 阻断。

### 3.7 快照携带的跨存储迁移

已实现新的 migrate API：

```text
POST /vm/{vm}/migrate { dest_datastore, online? }
```

能力：

- 迁移时携带完整 reflink snapshot tree。
- offline forward 与 online drive-mirror 两条路径。
- 目标端生成平台原生布局：active disk、`.snap.<uuid>` artifacts、config.xml 移动，源侧归档 `.migrated`。
- OCFS2 A -> B 的 offline + online E2E 已验证。
- petrel-vol <-> OCFS2 snapshot-chain interop 已接入 full VM migrate API。
- petrel 目前只支持 LINEAR chains，branch/clone 树仍有限制。

重要已知坑：

- online 迁移 block job 用 `query-block-jobs` 轮询，不依赖 BLOCK_JOB 事件。
- revert 需要 VM fully shutoff。
- petrel branch-clone snapshot 迁移仍有 copy-up 对象丢失问题，见后续待办。

### 3.8 cdp-agent in-guest CDP

已完成新的 `cdp-agent/`：

- QEMU-independent，运行在 guest 内。
- Rust std-only 用户态 agent。
- Linux `dm-cdp` 和 Windows `cdpflt` capture drivers。
- 三档模式：实时 / 增量 / 位图，对应 cdp-worker HIGH / MIDDLE / LOW。
- wire-compatible with cdp-worker `struct iodata` stream。
- 配套 `cdp-sink`：`iolog`、`disk`、`replica` 可组合。
- 已在真机 + Linux VM 端到端验证 md5 三方一致。
- 已在 Windows Server 2022 VM 上验证 boot-start filter build/sign/load/capture。

### 3.9 Petrel 分布式存储集成

Petrel 作为 xvirt datastore 已集成：

- petrel REST API。
- lvm/petrel datastore type。
- VM disks via vhost-user-blk。
- 删除 VM 可回收 volume。
- `petrel-ublk` list / offline snapshot。
- “浏览存储池”功能已完成。
- Phase 0-4 已完成，Phase 5 cluster management UI 延后。

已验证的相关能力：

- shared OCFS2 over petrel volume 完成。
- fstrim/UNMAP discard 支持。
- targeted reclaim 删除脚本包含 reflink refcount 处理。
- 40G storage network 从 1G mgmt 迁到 10.40.0.x direct link，MTU9000，流量已证明走 40G；但性能主要受磁盘限制。

### 3.10 Petrel 性能与 bug 修复

已完成/验证：

- vhost completion hang 已修：丢 avail-kick + 未启用 INDIRECT_DESC 两个独立 bug，14/14 PASS。
- cross-pool isolation 已修：OSD/wire 从 single-pool 修到 multi-pool data path，同名跨 pool 隔离通过。
- seqread regression 根因定位：dirty-extent cache flush/overlay race panic 已修；剩余 replica=2 collapse 是 consumer HDD 限制。
- VM disk 读路径主要问题定位并修复：
  - async pipeline。
  - serde_bytes 降 CPU。
  - data pread O_DIRECT。
  - prefetch amplification 约 180x 问题修为 monotonic-frontier + consecutive-seq-run。
- gateway OOM 通过 jemalloc/缓存控制改善。
- medium-error 读 failover 已实现并在坏盘池验证：可从 replica 读取。

仍保留的重要结论：

- durable write 只有 raw NVMe 的约 6-11%，瓶颈是 OSD per-object redb txn + fsync 串行。
- 需要 group commit / pipelined replication / batching，tiering 前先解决写路径。
- VM disk 推荐 raw vol，不推荐 OCFS2 叠层。
- gateway async cache 测试时写完要 sleep 8-12s 再停，否则会误判数据损坏。

### 3.11 Petrel 磁盘替换与恢复

已完成：

- disk replacement safety：superblock validation、by-id、recover-interval。
- 验证 3 次。

仍待查：

- backfill 不收敛，约 96% stall；数据仍在 replica 上安全，但 recovery 本身有 bug。

### 3.12 QEMU memory-tiering / memtier

已完成：

- QEMU fork `memory-tiering` branch。
- guest RAM tiering：`uffd + io_uring`。
- QMP 接口 `x-memtier-*`。
- dev host 运行过对应 build，byte-perfect 验证。
- 设计过 memtier 与 migration 的集成方案：跳过 swapped pages、shared petrel backing、capability negotiation、savevm meta。

关键性能结论：

- 在 E5-2673 v3 上 page_idle 不可用，退化为 write-aging。
- 对 Redis workload，传统 host NVMe swap 比当前 memtier 快约 2-5 倍。
- petrel backing 对 Redis 不可用，swap-in ms latency 会导致吞吐崩。
- memtier 要赢，需要 page_idle-capable CPU（Skylake+）、稳定 hot set、本地 NVMe。
- memtier swap-in 是 1-page/sync/single-thread/no-readahead，是主要瓶颈。
- naive cluster readahead 回归，因为 memtier 会为 speculative pages double-pay。
- swap-cache 模型前两步通过，第三步 clean free-eviction 触发 guest soft-lockup，已回退到 known-good。

### 3.13 状态 reconcile

ferrovirt 已增加 real VM/datastore state reconcile：

- startup + 60s 周期。
- 修复 power-loss reboot 后 VM 仍显示 running 的问题。

### 3.14 管理控制台设计

管理控制台设计来自 claude.ai/design `xvirt-mgmt` 项目：

- 已通过 DesignSync MCP 只读引用。
- 2026-06-19 已 port Monitor / Configure / Permissions 两栏 tabs 到 `ResourceTabs.jsx`。

### 3.15 磁盘健康告警

`alert_monitor.py` 已做 disk SMART monitoring：

- pending / uncorrectable sectors -> warning。
- health FAILED -> critical。
- category = storage。

当前环境重要事实：

- node 123 petrel pool 有两块 WD HDD 异常：`sdb` 和 `sdd`。
- 坏盘导致 petrel read I/O errors，曾阻塞 branch-tree petrel migration full-readback verification。
- read failover 已改善主线读取，但 branch clone 迁移 bug 仍需修。

## 4. 进行中 / 未完成 / 有风险任务

### 4.1 Petrel branch-clone snapshot migration 错误

状态：待修。

现象：

- bad-disk 测试暴露 branch-clone snapshots 在 tree petrel migration 中丢失 copy-up objects。
- 读取时会读成 parent 内容。
- main / linear chain 正确，branch tree 不正确。

建议下一步：

- 先构造最小 branch clone snapshot tree 复现。
- 关注 petrel COW / snapdiff / xvirt-blockdiff 迁移路径是否只处理 linear parent chain。
- 修复后必须在坏盘 read failover 环境和健康环境都做 byte-exact full-readback。

### 4.2 Petrel backfill 不收敛

状态：待查。

现象：

- 磁盘替换安全修复已验证。
- backfill 会卡在约 96%，但数据仍有 replica。

建议下一步：

- 重点查 recovery/peering multi-pool、对象状态机、缺失对象扫描、progress accounting。
- 不要只看百分比，确认具体卡住的 PG/object set。

### 4.3 Petrel durable write 性能

状态：设计/优化待做。

瓶颈：

- per-object redb txn + fsync 串行。
- 并发不扩展。
- replica=2 每次写被慢盘/慢 NVMe floor。

方向：

- group commit。
- pipelined replication。
- batching / large-write coalescing 产品化。
- OSD class / placement policy 修复，避免慢盘拖累快池。

### 4.4 Memtier 产品化仍未到位

状态：核心功能可跑，生产收益受硬件/算法限制。

问题：

- E5 v3 无 page_idle，write-aging 会误驱逐 read hot set。
- swap-in 单页同步瓶颈明显。
- petrel backing latency 不适合 Redis 这种延迟敏感工作负载。
- swap-cache clean eviction 方案曾触发 soft-lockup，已回退。

建议：

- 在 Skylake+ / page_idle 可用机器上重测。
- 先实现 read heat / LRU / demand-paced reclaim。
- 每一步都跑 pmemsave round-trip + redis deterministic integrity test。

### 4.5 新栈 deb 化仍需核对

状态：需复查 build-deb-xvirt 当前实际内容。

memory 记录中仍提示：

- `build-deb-xvirt` 可能仍打 Java `virt-control`。
- 缺 `xvirt-ferrovirt`、`xvirt-petrel`、`xvirt-concilium`、`aegis-sdk` deb。
- meta 依赖需要从 virt-control 切到 ferrovirt。

### 4.6 PXE 重装物理节点

状态：fledge 方案验证过，物理 106/123 重装仍是下一步。

注意：

- PXE destructive 操作必须 gated。
- 需要确认 MAC、目标盘、角色、ISO、静态 IP。
- gateway dnsmasq enable-tftp 需要 restart，不是 reload。

### 4.7 前端管理控制台继续补齐

状态：部分 port 完成。

已完成：

- Monitor / Configure / Permissions two-pane tabs。

待继续：

- center/management 视图更多资源页和细节交互。
- i18n 深层表单硬编码中文仍需逐步迁移。

### 4.8 旧路径与旧文档清理

状态：待清理。

风险：

- `virt-control` 仍在树中但已废弃。
- README/PACKAGING 中仍有旧 RPM / Java 控制面内容。
- 部分 memory 是已关闭问题，后续 AI 容易误认为仍未完成。

建议：

- 在顶层 docs 增加“当前生产路径 / 历史路径 / 实验路径”三列表。
- 把 `CLAUDE.md` 中的“工作进展”按时间迁移到 `docs/HISTORY.md` 或专门 changelog。

## 5. 关键坑位清单

- 不要默认跑本地 autotest，除非用户明确要求。
- 改组件后只 deploy 该组件，不要重启全栈。
- lictor 必须先于 hyper-agent。
- petrel gateway cache 是单写语义，多 gateway shared vol 要 `--no-cache`。
- petrel-ublk 预写镜像后要 sleep 8-12s 再停 gateway，避免异步缓存未落盘。
- petrel VM volume 名不是传入的 `vol`，而是 `<vm_uuid>-<disk_uuid>`。
- petrel pool 参数很重要，不带 `--pool` 可能在 default pool 创建 stray vol。
- 迁移相关：host-passthrough 跨不同 CPU host 应拒绝，custom CPU 才是主线。
- 已启动的旧 VM 若未带 vapic-off，迁移前需要重启。
- Ubuntu OVMF 路径是 `/usr/share/OVMF/*_4M.fd`。
- `xvirt-meta` 不应 pin 精确 git sha 版本。
- QEMU deb 必须保持 epoch 高于 Ubuntu stock。
- memtier 与 vhost-user/petrel 在内存模型上互斥或受限。
- branch-tree snapshot migration 不要假设已正确，linear OK 不代表 branch OK。

## 6. 建议的下一批工作优先级

1. 修 Petrel branch-clone snapshot migration copy-up 丢失问题。
2. 查 Petrel backfill 96% stall。
3. 复查并完成新栈 deb 化：ferrovirt / lictor / concilium / petrel / aegis-sdk。
4. 把 fledge + lycan + xvirt-autotest 串成可重复的物理节点重装验证流程。
5. Petrel 写路径 group commit / pipelined replication 设计与实现。
6. memtier 在 page_idle-capable 硬件上重新验证，并推进 read-heat / demand-paced reclaim。
7. 清理文档代际差异，把旧 Java/RPM 路径明确标为历史。
8. 继续管理控制台与 i18n 深层页面迁移。

## 7. 可直接给其他 AI 的接手 Prompt

```text
你现在接手 /root/code/xvirt。先读 /root/code/xvirt/CLAUDE.md 和 /root/.claude/projects/-root-code-xvirt/memory/MEMORY.md，不要重新探索全仓结构。当前新栈是 ferrovirt(Rust 控制面, DB owner) + hyper-agent(Python 宿主机 agent) + lictor(Python QMP owner) + concilium(C 集群层) + petrel(Rust 分布式存储) + xvirt-client(React)。Java virt-control 已废弃。

工作习惯：默认不要从本地控制机跑 autotest，除非用户明确要求。改组件后用 /root/code/xvirt/xvirtctl deploy <comp>，只部署被改的组件。用户要求 commit 时，petrel/xvirt lab repos 直接 commit 到 main；只在用户要求时 push。

当前已完成的大块：xvirtctl 新栈运行管理、xvirt-autotest、fledge PXE、lycan Ubuntu autoinstall ISO、QEMU deb epoch/OVMF/apt wedge 修复、真实 VM 双向在线迁移、CPU mode precheck、跨存储携带 snapshot tree 的迁移、cdp-agent、petrel datastore/vhost-user-blk 集成、petrel 多个 read/vhost/cache bug 修复、QEMU memtier 原型与性能分析、状态 reconcile、disk SMART 告警。

当前优先待办：1) petrel branch-clone snapshot migration 会丢 copy-up objects，linear/main OK 但 branch tree 不正确；2) petrel disk replacement 后 backfill 卡约 96%；3) petrel durable write 性能受 per-object redb txn+fsync 串行限制，需要 group commit/pipeline/batching；4) build-deb-xvirt 可能仍未完整切到 ferrovirt 新栈，需要复查 deb 化；5) memtier 产品化需要 page_idle/read-heat/demand-paced reclaim；6) 文档仍混有旧 Java/RPM 路径，需要清理。

常见坑：petrel VM vol 名是 <vm_uuid>-<disk_uuid>，不是 create 请求传入 vol；petrel-ublk 写完镜像要 sleep 8-12s 再停；petrel shared OCFS2 多 gateway 要 --no-cache；跨不同 CPU host 的 host-passthrough VM 应拒绝，custom CPU 才可迁；QEMU deb 必须高 epoch 防止 Ubuntu stock 覆盖；OVMF 路径用 /usr/share/OVMF/*_4M.fd。
```

