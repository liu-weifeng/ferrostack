# VM 指标直采设计（不依赖 libvirt）—— 设计稿，暂不落地

> 背景：`mx-manager-host-monitoring-plan.md` §2.4 用 `libvirt-exporter`
> 取 per-VM 指标。但 xvirt 的 hyper-agent **不调用 libvirt**（直接拼
> QEMU 命令行 + 自管 QMP，见 `hyper-agent/CLAUDE.md`），libvirt-exporter
> 无法用。本文设计一条**直采**方案替代它；**本阶段只设计、不实现**
> （第一阶段先把 VictoriaMetrics + ebpf_exporter 跑通，见
> `monitoring/README.md`）。

## 1. 思路

复用 hyper-agent 既有能力，不引新进程：

- **QMP**：每台运行中的 VM 都有 hyper-agent 接管的 QMP（`monitor/manager.py`
  的 `MonitorManager`，`manager_sync.execute(vm_uuid, cmd)`）。直接发
  QMP 查询拿 QEMU 自报的精确统计。
- **cgroup v2**：每个 qemu 进程落在自己的 scope/slice，
  `/sys/fs/cgroup/.../{cpu.stat,memory.current,io.stat,cpu.pressure}`
  给出内核侧 CPU/内存/IO/**per-VM PSI**，与宿主机 PSI 同口径。
- **/proc/<qemu-pid>**：RSS、线程数、vCPU 线程 CPU 时间兜底。
- **tap 设备**：`/sys/class/net/<tap>/statistics/{rx,tx}_bytes` 拿 per-VM
  网络字节（QMP `query-netdev` 不含累计字节，用 tap 计数更稳）。

产出仍走 **hyper-agent 进程内 Prometheus exporter（`:9101`）**——
新增一个 per-VM collector，指标名沿用现有 `xvirt_vm_*` 前缀（ferrovirt
`resource_metrics.rs` 模板已用 `xvirt_vm_cpu_usage_ratio` /
`xvirt_vm_memory_used_bytes` 等，**前端无需改**，直采补齐这些 series 即可）。

```
QEMU(QMP) ┐
cgroup v2 ┼─► hyper_agent.metrics_exporter.collectors.vm_direct ─► :9101 /metrics
/proc     ┘        （vm_uuid → qemu pid → cgroup path 映射）          │
tap stats                                                            ▼  VictoriaMetrics scrape → ferrovirt → 前端
```

## 2. vm_uuid → 资源定位

hyper-agent 已知每台 VM 的：pidfile（`/run/libvirt/qemu/<uuid>.pid` 风格，
`QemuLauncher` 写）、QMP sock（`MonitorManager` 注册表，key=vm uuid）、
磁盘/网卡 alias（config.xml）。由此：

| 需要 | 来源 |
|---|---|
| qemu pid | pidfile / `MonitorManager` 已记录 |
| cgroup 路径 | `/proc/<pid>/cgroup`（v2 单行 `0::/…`）→ 拼 `/sys/fs/cgroup/<path>` |
| tap 名 | config.xml interface target dev / `query-netdev` 的 ifname |
| 块设备 node | config.xml disk alias ↔ QMP `query-blockstats` 的 `qdev`/`node-name` |

## 3. 指标与数据源

| 指标（`xvirt_vm_*`，label `vm=<uuid>` 等） | 源 | 说明 |
|---|---|---|
| `cpu_usage_ratio` / `cpu_seconds_total` | cgroup `cpu.stat:usage_usec`（首选）或 `/proc/<pid>/stat` | 每核归一 |
| `vcpu_seconds_total{vcpu=N}` | QMP `query-cpus-fast` + 线程 CPU 时间 | per-vCPU |
| `cpu_pressure_*` | cgroup `cpu.pressure` | **per-VM PSI**，比利用率更准 |
| `memory_used_bytes` / `memory_total_bytes` | cgroup `memory.current` / config 内存上限；QMP `query-balloon`、`query-memory-size-summary` | balloon 实际占用 |
| `memory_pressure_*` | cgroup `memory.pressure` | reclaim 压力 |
| `disk_read_bytes_total` / `disk_write_bytes_total` / `disk_*_ops_total` / 延迟 | QMP `query-blockstats`（首选，QEMU 精确）；兜底 cgroup `io.stat` | per-disk(alias) |
| `net_rx_bytes_total` / `net_tx_bytes_total` / `_packets_total` | `/sys/class/net/<tap>/statistics/*` | per-iface |
| `state` / `up` | `MonitorManager` 连接态 + QMP `query-status` | 1=running |
| `perf_ipc` / `llc_occupancy_bytes` / `mbm_*`（可选，进阶） | `perf_event_open` per-vCPU-线程；Intel RDT `resctrl` | 需 CPU 支持，按 `cpuid` 探测后开 |

采集周期：QMP 查询较重，5s；cgroup/proc/tap 读文件极轻，可 1s。直方图类
（block 延迟）若需亚秒分辨率，后续可在 ebpf_exporter 侧按 cgroup/dev 维度
聚合，避免 QMP 高频。

## 4. 与现有代码的接入点

- 新增 `hyper_agent/metrics_exporter/collectors/vm_direct.py`，注册进
  `metrics_exporter/registry.py`（与现有 collectors 同模式）。
- 复用 `monitor.manager_sync.execute(vm_uuid, ...)` 发 QMP；复用
  `xvirt_tools_client`（若 guest 装了 xvirt-tools，可补 guest 内视角：
  steal time、guest mem available——host 侧看不到的）。
- 指标前缀/标签与 `ferrovirt/src/service/resource_metrics.rs` 的模板对齐
  （`xvirt_vm_*`，label `vm` / `mount` / `iface`），**ferrovirt 与前端零改**。

## 5. 分期（落地时）

1. cgroup + /proc + tap 直采（无 QMP，最稳）：cpu/mem/PSI/net 基础面。
2. QMP `query-blockstats` / `query-balloon` 接入：精确磁盘 IO + balloon。
3. per-vCPU + perf/RDT（IPC、LLC、MBM）：进阶，按硬件能力开关。
4. guest 内视角（xvirt-tools）：steal/guest-available 交叉校验。

## 6. 风险

- **cardinality**：label 用 `vm=<uuid>`，VM 频繁增删累积 series 暴涨
  （plan §八#1）。落地时用稳定业务 ID 或对已销毁 VM 做 stale 标记 +
  VictoriaMetrics 端 `-search.maxStalenessInterval` / relabel 丢弃。
- QMP 查询阻塞：必须走 `manager_sync`（worker 线程），不能阻塞 asyncio
  主 loop；查询要有超时，单 VM 失败不拖累整体 collector。
- cgroup 路径随 systemd-run/scope 变化：每轮按 `/proc/<pid>/cgroup`
  重解析，不缓存绝对路径。
- QEMU 版本差异：`query-stats` 需 QEMU≥7.1；老版本退回
  `query-blockstats`/cgroup。
