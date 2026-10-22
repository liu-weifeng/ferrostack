# 内存分层监控数据设计：底层推什么、平台记什么

> 前提：hyper-agent 周期性调 `x-memtier-query`（总控每 5s 一拍已经在调），把数据推到监控平台
> （现有 Prometheus 风格 exporter，前缀 `xvirt_`，VM 标签 `vm_uuid`）。本文定义推送的指标集、
> 命名、标签、采样周期，以及平台侧的派生规则、保留策略、看板与告警。字段语义以
> [`qemu-memtier-interface.md`](./qemu-memtier-interface.md) §4 为准。
>
> 2026-10-22。现状：exporter 已推 7 个 VM 级指标 + 3 个主机级总控指标（`resource_plan.py::_tiering`）。

---

## 0. 原则

1. **推原始值，不推派生值。** 累计量按 counter 原样推，速率、比例、近期平均都在平台用
   `rate()` / recording rule 算。这样 reset（enable 归零）和采样抖动由平台统一处理，底层不做状态。
2. **一次 query 多处用。** 总控每拍的 `x-memtier-query` 结果已经放在 controller snapshot 里，
   exporter 从 snapshot 取，不再单独调 QMP（query 会向 QEMU stderr 打诊断行，少调为好）。
3. **三层粒度：VM、宿主、后端盘。** VM 看效果与健康，宿主看总控决策，盘看磨损与容量。
4. **分两档推。** 核心档默认推、进默认看板和告警；诊断档默认也推但只进深度看板，量不大
   （每 VM 约 40 条序列），保留期可以短。
5. **标签少而稳。** 全部 VM 级指标只带 `vm_uuid`；`priority`、`policy` 这类会变的属性单独
   作为 info 型指标，不挂到每条序列上，避免换档时序列断裂。

---

## 1. 数据来源与周期

| 来源 | 周期 | 内容 |
|---|---|---|
| `x-memtier-query`（经总控 snapshot） | 5s | 每 VM 的 `MemTierInfo` 全部字段 |
| 总控 snapshot | 5s | 宿主合成压力、余量、状态、各 VM 档位 |
| `write_budget` | 5s | 每后端盘当日写入量、预算、当前分配的 `max-evict-mbps` 总和 |
| `nvme smart-log` | 60s | 每盘 `data_units_written`、`percentage_used`、温度 |
| `/proc/pressure/memory` | 5s | 宿主 PSI（总控已读，顺手导出） |
| QMP 事件 | 实时 | `MEMTIER_BACKING_FULL`、`MEMTIER_SWAPIN_ERROR` |
| 编排动作 | 实时 | enable / disable / set-policy / 快照恢复进入分层态 |

exporter 被 scrape 的周期建议与采样一致（5s）；平台存 5s 原始点，降采样见 §5.2。

---

## 2. VM 级指标

标签：`vm_uuid`。多 region 时求和（当前只有一个 region）。
类型：G = gauge，C = counter。「源」是 `MemTierInfo` 字段。

### 2.1 核心档

| 指标名 | 型 | 源 | 用途 |
|---|---|---|---|
| `xvirt_vm_tiering_total_bytes` | G | `total-bytes` | 分母 |
| `xvirt_vm_tiering_resident_bytes` | G | `resident-bytes` | 已有 |
| `xvirt_vm_tiering_swapped_bytes` | G | `swapped-bytes` | 已有；下刷比例分子 |
| `xvirt_vm_tiering_clean_resident_bytes` | G | `clean-resident-bytes` | swap-cache 占比 |
| `xvirt_vm_tiering_backing_total_bytes` | G | `backing-total-bytes` | 已有 |
| `xvirt_vm_tiering_backing_used_bytes` | G | `backing-used-bytes` | 已有 |
| `xvirt_vm_tiering_backing_free_chunks` | G | `backing-free-chunks` | 容量告警的真实依据（≤4 开始压实，0 停换出） |
| `xvirt_vm_tiering_evict_mbps` | G | `evict-mbps` | 已有；总控限速是否落地 |
| `xvirt_vm_tiering_evicted_pages_total` | C | `evicted-pages` | 换出速率 |
| `xvirt_vm_tiering_evict_aborts_total` | C | `evict-aborts` | 已有 |
| `xvirt_vm_tiering_evict_reevict_pages_total` | C | `evict-reevict-pages` | 抖动率分子 |
| `xvirt_vm_tiering_evict_zero_pages_total` | C | `evict-zero-pages` | 免写比例 |
| `xvirt_vm_tiering_evict_clean_pages_total` | C | `evict-clean-pages` | 免写比例 |
| `xvirt_vm_tiering_swapin_faults_total` | C | `swapin-faults` | 已有；客户机受影响程度 |
| `xvirt_vm_tiering_zero_fills_total` | C | `zero-fills` | 区分「读后端」与「零填」的缺页 |
| `xvirt_vm_tiering_active_swapins_total` | C | `active-swapins` | 主动换回量 |
| `xvirt_vm_tiering_swapin_errors_total` | C | `swapin-errors` | critical 告警 |
| `xvirt_vm_tiering_swapin_latency_samples_total` | C | `swapin-lat-samples` | 延迟分解分母 |
| `xvirt_vm_tiering_swapin_latency_ns_total{stage="wait"\|"device"\|"publish"}` | C | `avg × samples` | 底层把平均值乘回样本数得累计和，平台 `rate(sum)/rate(samples)` 得近期平均 |
| `xvirt_vm_tiering_swapin_max_ns` | G | `swapin-max-ns` | 自 enable 以来最差值；跳变即告警 |
| `xvirt_vm_tiering_evict_io_bytes_total` | C | `evict-io-bytes` | 盘写入量归因到 VM；磨损 |
| `xvirt_vm_tiering_evict_ios_total` | C | `evict-ios` | 平均写大小 |
| `xvirt_vm_tiering_backing_compact_moved_pages_total` | C | `backing-compact-moved-pages` | 写放大的第二来源 |
| `xvirt_vm_tiering_psi_some_ratio` | G | `psi-some-avg10 / 100` | QEMU 看到的宿主压力（与宿主 PSI 对账） |
| `xvirt_vm_tiering_tunable{name=…}` | G | `tunables` 中 `min-resident-mb`、`target-resident-mb`、`psi-trigger`、`max-evict-mbps`、`max-swapin-mbps`、`cold-runs` | 生效值；与总控下发值对比确认落地；画在水位图上做参考线 |
| `xvirt_vm_tiering_info{policy=…, backing_dev=…, fault_shards=…, backing_discard=…}` | G=1 | `policy`、`path`→盘名、enable-time 旋钮 | info 型；用 `backing_dev` 与盘级指标 join |
| `xvirt_vm_tiering_tier{priority=…}` | G | 总控 | 已有：该 VM 当前档位 |

### 2.2 诊断档

| 指标名 | 型 | 源 | 用途 |
|---|---|---|---|
| `xvirt_vm_tiering_evict_io_size_total{le_kib="4"…"1024"}` | C | `evict-io-hist` | 写大小分布（按桶累计，不做成 Prometheus histogram，桶是精确大小而非 ≤） |
| `xvirt_vm_tiering_evict_io_seq_total` / `_fwd_total` | C | `evict-io-seq` / `evict-io-fwd` | 日志顺序率 |
| `xvirt_vm_tiering_backing_discards_total` / `_compactions_total` | C | 对应字段 | 分配器行为 |
| `xvirt_vm_tiering_backing_dead_chunks` | G | `backing-dead-chunks` | discard 是否跟得上 |
| `xvirt_vm_tiering_evict_sampled_pages_total` / `_sampled_clean_pages_total` | C | 对应字段 | swap-cache 命中率 |
| `xvirt_vm_tiering_evict_verify_reads_total` | C | `evict-verify-reads` | verify 关着时应为 0 |
| `xvirt_vm_tiering_clean_slot_reclaims_total` | C | `clean-slot-reclaims` | 后端偏小的信号 |
| `xvirt_vm_tiering_wp_faults_total{kind="evicting"\|"other"}` | C | `wp-faults-*` | other ≠ 0 是实现问题 |

不推：`backing-largest-free-bytes`（历史字段，恒 ≤ 2MB）、`enabled`（恒 true，用序列存在性表示）、
`backing-chunk-bytes` / `backing-chunks`（常量，放 info 标签即可）。

---

## 3. 宿主级指标

无 VM 标签。

| 指标名 | 型 | 源 | 用途 |
|---|---|---|---|
| `xvirt_host_tiering_pressure` / `_headroom` / `_state` | G | 总控 | 已有 |
| `xvirt_host_tiering_psi_some_ratio` | G | `/proc/pressure/memory` some avg10 | 总控输入 |
| `xvirt_host_tiering_vms` | G | snapshot | 分层中的 VM 数 |
| `xvirt_host_tiering_resident_bytes` / `_swapped_bytes` | G | 各 VM 求和 | 宿主级超分视图：swapped 就是「省下来的物理内存」 |
| `xvirt_host_tiering_budget_pressure` | G | 总控 `budget_pressure` | 写预算对决策的影响 |

分层 VM 数、求和值也可以在平台 `sum by (host)` 得到；底层直接推是为了让单机看板不依赖 join。

---

## 4. 后端盘级指标

标签：`backing_dev`（如 `nvme1n1`，由 `path` 解析到所属整盘，分区归到盘）。

| 指标名 | 型 | 源 | 周期 | 用途 |
|---|---|---|---|---|
| `xvirt_backing_tiering_written_bytes_total` | C | `write_budget` 按盘累计（来自各 VM `evict-io-bytes` 差分） | 5s | 分层写入归因 |
| `xvirt_backing_tiering_budget_bytes_per_day` | G | `write_budget` | 5s | 当日预算 |
| `xvirt_backing_tiering_written_today_bytes` | G | `write_budget` | 5s | 当日已用 |
| `xvirt_backing_tiering_allowance_mbps` | G | `write_budget` | 5s | 当前分给同盘 VM 的总限速 |
| `xvirt_backing_nvme_data_units_written_total` | C | smart-log `data_units_written × 512000` | 60s | 盘实际写入（含非分层写入），与上面对账得写放大 |
| `xvirt_backing_nvme_percentage_used` | G | smart-log | 60s | 寿命 |
| `xvirt_backing_nvme_temperature_celsius` | G | smart-log | 60s | |
| `xvirt_backing_tiering_vms` | G | 按 `path` 归并 | 5s | 该盘上分层 VM 数 |

---

## 5. 事件

| 事件 | 推送形式 |
|---|---|
| `MEMTIER_BACKING_FULL` | 计数 `xvirt_vm_tiering_events_total{type="backing_full"}`（QEMU 持续满时约 200ms 一次，底层按 VM 去重为「进入满状态」一次）+ 事件日志（平台事件流 / 告警注解） |
| `MEMTIER_SWAPIN_ERROR` | `xvirt_vm_tiering_events_total{type="swapin_error"}` + 事件日志，附 `gfn`；随后 VM 进程退出，由现有 VM 状态监控接手 |
| enable / disable / set-policy / 快照恢复进入分层态 | 事件日志（`xvirt_vm_tiering_events_total{type="enable"\|"disable"\|"set_policy"\|"restored"}`），set-policy 附下发的旋钮；用于在看板上标注「为什么这时候水位变了」 |

---

## 6. 平台侧

### 6.1 Recording rules（派生指标）

| 名称 | 表达式（PromQL 风格） |
|---|---|
| 下刷比例 | `swapped_bytes / total_bytes` |
| 换出速率 B/s | `rate(evicted_pages_total[1m]) * 4096` |
| 缺页换入速率 /s | `rate(swapin_faults_total[1m])` |
| 零填比例 | `rate(zero_fills_total[1m]) / (rate(swapin_faults_total[1m]) + rate(zero_fills_total[1m]))` |
| 抖动率 | `rate(evict_reevict_pages_total[5m]) / rate(evicted_pages_total[5m])` |
| 免写比例 | `(rate(evict_zero_pages_total[5m]) + rate(evict_clean_pages_total[5m])) / rate(evicted_pages_total[5m])` |
| 后端写放大 | `(rate(evict_io_bytes_total[5m]) + rate(backing_compact_moved_pages_total[5m]) * 4096) / (rate(evicted_pages_total[5m]) * 4096)` |
| 平均写大小 | `rate(evict_io_bytes_total[5m]) / rate(evict_ios_total[5m])` |
| 日志顺序率 | `rate(evict_io_fwd_total[5m]) / rate(evict_ios_total[5m])` |
| 近期缺页延迟（分段） | `rate(swapin_latency_ns_total{stage}[1m]) / rate(swapin_latency_samples_total[1m])` |
| 盘级写放大 | `rate(nvme_data_units_written_total[1h]) / sum by (backing_dev) (rate(evict_io_bytes_total[1h]))` |
| 寿命预测（天） | `(1 - percentage_used/100) * TBW / (rate(nvme_data_units_written_total[7d]) * 86400)` |
| 宿主省下的内存 | `sum by (host) (swapped_bytes)` |

### 6.2 保留

| 数据 | 原始 | 降采样 |
|---|---|---|
| 核心档 gauge / counter | 5s × 7 天 | 1m × 90 天，1h × 2 年（容量规划） |
| 诊断档 | 5s × 3 天 | 5m × 30 天 |
| 盘级 smart-log | 60s × 90 天 | 1h × 5 年（寿命） |
| 事件 | 全量 × 1 年 | |

### 6.3 看板

**VM 面板**（一台 VM）
- 水位：total / resident / swapped / clean-resident 堆叠，叠 `min-resident-mb` 与 `target-resident-mb` 参考线，事件标注（set-policy、恢复）。
- 流量：换出 B/s（叠 `max-evict-mbps` 参考线）、缺页换入 /s、主动换回 /s、零填 /s。
- 延迟：三段近期平均叠加 + `swapin_max_ns`。
- 质量：抖动率、免写比例、后端写放大、evict-aborts 速率。
- 后端：used / total、free chunks、dead chunks、压实与 discard 速率。

**宿主面板**
- 宿主 PSI 与总控 pressure / headroom / state 同图；各 VM 档位时间线。
- 各 VM swapped 堆叠（= 超分收益）；各 VM 换出速率堆叠叠盘级 allowance。

**盘面板**
- 当日分层写入 vs 预算；smart-log 写入速率 vs 分层写入速率（盘级写放大）；percentage_used 趋势与寿命预测。

### 6.4 告警

| 级别 | 规则 | 说明 |
|---|---|---|
| critical | `increase(swapin_errors_total[5m]) > 0` 或事件 `swapin_error` | QEMU 已退出 |
| critical | VM 进程存在而 `xvirt_vm_tiering_*` 序列消失 > 30s，且编排没有 disable 事件 | query 失败或 region 异常消失 |
| warning | 事件 `backing_full`，或 `backing_free_chunks <= 4` 持续 5m | 换出暂停 / 即将暂停 |
| warning | 抖动率 > 10% 持续 10m | 旋钮过激进 |
| warning | 近期 device 段平均 > 500µs 或 `swapin_max_ns` 跳变 > 10ms | 后端盘变慢 |
| warning | `written_today_bytes > budget_bytes_per_day` | 写预算超支 |
| warning | `percentage_used > 80` | 盘寿命 |
| warning | `tunable{name="max-evict-mbps"}` 与总控下发值不一致持续 1m | set-policy 没落地 |
| info | `rate(wp_faults_total{kind="other"}[5m]) > 0`，或 verify 关着而 `evict_verify_reads_total` 增长 | 实现诊断 |
| info | `clean_slot_reclaims_total` 增长 | 后端容量小于 RAM |

---

## 7. 底层改动点（hyper-agent exporter）

- `_tiering()` 从 snapshot 的 regions 映射新增字段；多 region 求和；`swapin_latency_ns_total`
  由 `avg × samples` 乘回；`evict-io-hist` 展开为 9 条带 `le_kib` 标签的 counter。
- `tunable{name}` 与 `info{...}` 两条 info 型指标；`backing_dev` 由 `path` 经 `/sys/dev/block`
  归并到整盘（`write_budget.py` 已有该解析）。
- 盘级：`write_budget` 已有当日记账，补 `allowance_mbps`；新增 60s 一次的 smart-log 采样线程
  （`write_budget.py` 已调 `nvme smart-log -o json`，复用）。
- 事件：lictor 收到 QMP 事件后转发给 hyper-agent，exporter 计数；编排动作在动作处计数。
- 序列生命周期：VM disable 或消失后 `_evict_stale` 清理，已有。
- 顺带：去掉 QEMU `tier_query_one` 里的 stderr 诊断输出，否则 5s 一次写日志。
