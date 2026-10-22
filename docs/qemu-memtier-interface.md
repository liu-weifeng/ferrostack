# QEMU 内存分层（memtier）接口参考：配置接口与查询接口

> 面向**上层功能与监控设计**的接口规格。以 qemu `memtier-stats` 分支（HEAD `17fa8e0482`，
> 11.1 血统）的 `qapi/memtier.json` 与 `memtier/*.c` 为准，每个字段都对照了实现。
> 机制与调参背景见 [`qemu-memtier-usage.md`](./qemu-memtier-usage.md)，xvirt 编排见
> [`memory-tiering.md`](./memory-tiering.md)。
>
> 2026-10-22。所有命令都是实验性 `x-` 前缀，字段可能变动；本文列出的就是当前全集。

---

## 1. 总览

### 1.1 对象模型

- **region**：分层的基本单位，对应一个 guest RAM 块（`RAMBlock`，如 `pc.ram`）。一个 region
  一个后端、一套策略、一个 worker 线程、1–8 个 fault 线程。当前实际只用一个 region（主 RAM 块）。
- **后端（backing）**：raw 块设备、预分配 raw 文件或 SPDK NVMe 命名空间。无头、无元数据，
  按 4K slot 寻址，以 2MB chunk 做循环日志分配。
- **页状态**：`RESIDENT`（在 RAM）/ `RESIDENT_CLEAN`（在 RAM 且仍持有后端 slot，swap-cache）/
  `SWAPPED`（只在后端）/ 过渡态 `EVICTING`、`EVICTING_DIRTIED`、`FAULTING_IN`、`SWAPPED_MOVING`。

### 1.2 生命周期

```
未分层 ──x-memtier-enable──▶ 分层中 ──x-memtier-disable──▶ 未分层
   ▲                            │  ▲
   │                            │  └── x-memtier-set-policy（运行中改策略/旋钮，不断流）
   │                            └──── x-memtier-query（任何时候）
   └── 带 "memtier" 段的 -incoming 流加载成功后直接进入「分层中」（快照恢复 / 共享后端热迁移）
```

- 同一 region 不能重复 enable（报错），没有「重新配置」命令，改参数走 set-policy。
- disable 是同步的：拉回全部换出页后拆除，期间 QMP 不响应（§3.3）。
- QEMU 退出不做任何收敛：后端内容随进程作废（除非上层做了快照）。

### 1.3 接口一览

| 类型 | 名称 | 作用 |
|---|---|---|
| 命令 | `x-memtier-enable` | 对一个 region 开启分层 |
| 命令 | `x-memtier-set-policy` | 运行中改预设 / 旋钮 |
| 命令 | `x-memtier-disable` | 收敛并关闭分层 |
| 命令 | `x-memtier-query` | 状态与累计统计（`MemTierInfo` 列表） |
| 事件 | `MEMTIER_BACKING_FULL` | 后端满，换出暂停 |
| 事件 | `MEMTIER_SWAPIN_ERROR` | 换入失败，QEMU 进程随即退出 |
| 迁移能力 | `memtier-shared-backing` | 迁移 / 快照只发常驻页，换出页以映射段携带 |
| 环境变量 | `XVIRT_MEMTIER_TRACE` | enable 时读取；置位后 stderr 输出逐事件 trace |

---

## 2. 前置条件与构建判定

| 条件 | 不满足时 |
|---|---|
| 构建时 Linux + liburing（`--enable-linux-io-uring`） | 编进 `memtier-stubs.c`：四条命令仍存在，全部返回 `memory tiering requires Linux userfaultfd-wp and io_uring support`；`"memtier"` 迁移段不注册 |
| 宿主页 4K | enable 报 `memtier requires a 4096-byte host page size` |
| 内核 userfaultfd 且支持写保护（6.x） | `userfaultfd not available` / `kernel lacks userfaultfd write-protect (need 6.x)` |
| 运行时 CAP_SYS_ADMIN（pagemap + page_idle） | enable 成功但 worker 告警 `page_idle/pagemap unavailable ... eviction disabled`：只能换入不能换出 |
| guest RAM 为匿名内存（普通 `-m`） | memfd / file-backed / `share=on`（vhost-user 设备）的 RAM 无法分层 |
| 后端支持 O_DIRECT | 打开失败 `cannot open backing device` |

上层判断「这个 QEMU 二进制带不带 memtier」：不能看 `query-commands`（stubs 也有命令名）；
可行的办法是对二进制找字符串 `memtier-worker`（线程名，只有真实现有），或起机后试
`x-memtier-query`（stubs 返回错误，真实现返回空列表）。

---

## 3. 配置接口

### 3.1 `x-memtier-enable`

```json
{ "execute": "x-memtier-enable",
  "arguments": { "id": "pc.ram",
                 "path": "/dev/nvme1n1p3",
                 "policy": "balanced",
                 "tunables": { "min-resident-mb": 2048, "fault-shards": 4 } } }
→ { "return": {} }
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | str | 否 | RAM 块名。缺省 = 有宿主映射的最大 RAM 块（q35/pc 下即 `pc.ram`）。写错报 `no RAM block named 'X'` |
| `path` | str | 是 | 后端。三种形式：块设备 `/dev/...`；普通文件；`spdk:0000:BB:DD.F[,ns=N]`（SPDK 用户态 NVMe，需 hugepage 与 vfio 绑定） |
| `policy` | enum | 否 | `conservative` / `balanced`（默认）/ `aggressive` / `custom`（= balanced 的基线，语义上表示「全部由 tunables 指定」） |
| `tunables` | object | 否 | 逐项覆盖，字段见 §3.4 |

执行时发生的事（上层要知道的副作用）：

1. 对整个 region `MADV_NOHUGEPAGE`，并把已有 THP 拆成 4K（否则 page_idle 看不见访问）。
2. 读 pagemap 统计实际已映射页数作为初始 `resident-bytes`（未被客户机碰过的 RAM 不算常驻）。
3. 按 `fault-shards` 把 region 切成 2MB 对齐的连续段，每段一个 uffd + `mt-fault<N>` 线程；
   起 `memtier-worker` 线程。
4. 打开后端：文件 / 块设备以 `O_RDWR|O_DIRECT`；容量 = 文件大小或 `BLKGETSIZE64`，
   slot 数 = 容量 / 4K（不足一个 slot 报 `backing device too small`）。普通文件会做一次 FIEMAP：
   有空洞告警 `backing file ... is sparse`，有 fallocate 未写 extent 告警 `has N MiB of unwritten
   (fallocate'd) extents; zero-fill it once`。两种都能用，只是首次写该区域慢。
5. `io-poll=true` 时做一次 polled 读探测，内核不支持则 enable 失败
   `io-poll: polled read probe on '...' failed`。

错误（`error.desc` 原文）：`memtier already enabled for region 'X'`、`no suitable guest RAM block
found`、`cannot open backing device 'P': ...`、`backing device too small`、`io_uring_queue_init
failed`、`failed to create userfaultfd (shard N)`、`failed to register shard N with userfaultfd`、
`spdk: ...`。失败时不留任何残留状态，可以修正后重试。

后端容量建议：≥ 客户机 RAM。swap-cache 让换入过的页继续占 slot，后端小于 RAM 时会先触发
`clean-slot-reclaims`（放弃 swap-cache 优势）再报 `MEMTIER_BACKING_FULL`。

### 3.2 `x-memtier-set-policy`

```json
{ "execute": "x-memtier-set-policy",
  "arguments": { "id": "pc.ram", "policy": "aggressive",
                 "tunables": { "psi-trigger": 0, "max-evict-mbps": 300 } } }
→ { "return": {} }
```

语义是**合并**，这一点对上层总控最重要：

- region 内部保存一份 override 集合。set-policy 给的每个字段写入 override；没给的字段不变。
- 生效值 = 预设值，再逐项被 override 覆盖。换 `policy` 只换预设，**已有 override 继续生效**。
  例：先设过 `min-resident-mb=2048`，再切 `conservative`，常驻下限仍是 2048MB 而不是 75%。
- **没有清除 override 的接口。** 要回到预设行为，上层必须显式把该字段设回预设对应的值
  （预设表见 §3.5）。上层应自己维护「期望旋钮全集」并整体下发，而不是增量改。
- `fault-shards`、`io-poll`、`busy-poll` 只在 enable 时生效，set-policy 里给了会被接受但无效。
- 钳制在每次 resolve 时做（§3.4 的「范围」列）；`x-memtier-query` 返回的是钳制后的生效值。
- 立即生效，不中断换出 / 换入；worker 下一拍（≤ 200ms）按新值工作。

错误：`memtier not enabled for that region`。

### 3.3 `x-memtier-disable`

```json
{ "execute": "x-memtier-disable", "arguments": { "id": "pc.ram" } }
→ { "return": {} }
```

顺序：停 worker 与 fault 线程（join）→ 在 QMP 调用线程里**同步**把每个 `SWAPPED` 页读回
（4K 同步读）、释放 `RESIDENT_CLEAN` 的 slot → 关 uffd、io_uring、后端 → 从 region 列表删除。

上层要点：

- 耗时 ∝ `swapped-bytes` / 后端读带宽，单线程同步 4K 读，数 GB 换出量是几十秒量级；
  期间整个 QMP 不响应，客户端超时要按 `swapped-bytes` 估算（xvirt 现用 120s，不够时先用
  set-policy `target-resident-mb=total` + `psi-trigger=100` 让 worker 异步换回，等 `swapped-bytes`
  接近 0 再 disable）。
- 宿主必须有 `swapped-bytes` 的空闲内存，否则收敛本身会把宿主打爆。
- 成功后 `x-memtier-query` 不再列出该 region；所有累计计数随之丢失（再 enable 从 0 开始）。
- 后端文件内容不清零、不打洞。

错误：`memtier not enabled for that region`。

### 3.4 旋钮（`MemTierTunables`）全表

「时机」：E = 仅 enable 时生效；R = enable 与 set-policy 都可。

| 字段 | 类型 / 单位 | 默认（balanced） | 范围 / 钳制 | 时机 | 含义 |
|---|---|---|---|---|---|
| `cold-threshold` | uint16 | 2 | — | R | 粗扫描 cell 的访问计数低于此判冷（越大越激进） |
| `cold-runs` | uint16 | 5 | — | R | cell 连续判冷多少轮才进入细扫描（去抖；≥2 推荐） |
| `psi-trigger` | number，% | 10 | — | R | 宿主 PSI memory `some avg10` ≥ 此值才换出。0 = 无视压力一直换到下限；100 = 实际上停止换出 |
| `max-evict-mbps` | uint32，MB/s | 500 | 0 = 不限 | R | 换出写入限速（滚动 1s 窗口） |
| `min-resident-mb` | uint64，MB | 50% RAM | ≤ total | R | 常驻硬下限，换出永不压到它以下 |
| `cooldown-ms` | uint32，ms | 1000 | — | R | 刚换回的页在此窗口内不换出 |
| `hysteresis-mb` | uint64，MB | 0 | ≤ total − min | R | 换出迟滞带：常驻 > min+band 才起动，压到 min+band/2 停。0 = 超过 min 即换 |
| `idle-streak` | uint16 | 2 | 0 视为 1 | R | 一页要连续多少轮扫描都 idle 才算冷。2 是为了过滤 pfn 复用带来的假 idle，设 1 会出现换出/换回踩踏 |
| `target-resident-mb` | uint64，MB | = min | [min, total] | R | 主动换回上限：常驻 < target 且宿主空闲时按「最近换出优先」预取。= min 即主动换回关 |
| `swapin-watermark` | number，% | psi-trigger / 2 | ≥ psi-trigger 时钳到 0.9× | R | PSI 低于此才允许主动换回 |
| `max-swapin-mbps` | uint32，MB/s | 256 | — | R | 主动换回限速 |
| `fault-shards` | uint16 | 1 | [1, 8] | E | 缺页处理分片数；每片一个线程与 uffd。SPDK / busy-poll 下每片占一个核 |
| `swapcache-verify` | bool | false | — | R | 哈希相等的再换出页额外读回逐字节比对再免写 |
| `backing-discard` | bool | 块设备 true，文件 false | SPDK 无效 | R | 死 chunk 打洞 / BLKDISCARD 后再复用。文件上打开会显著拖慢换出写（XFS 实测 −40% 客户机写吞吐） |
| `io-poll` | bool | false | SPDK 无效 | E | io_uring IOPOLL 换入；需要 NVMe poll 队列，enable 时探测 |
| `busy-poll` | bool | false | SPDK 恒 on | E | fault 线程不睡眠忙等（每片一个核） |

预设只定义前 6 项 + 常驻比例；其余 10 项的默认与预设无关。

### 3.5 预设值

| | conservative | balanced / custom | aggressive |
|---|---|---|---|
| cold-threshold | 0 | 2 | 8 |
| cold-runs | 8 | 5 | 2 |
| psi-trigger | 20 | 10 | 2 |
| max-evict-mbps | 200 | 500 | 1000 |
| min-resident-mb | 75% RAM | 50% RAM | 25% RAM |
| cooldown-ms | 2000 | 1000 | 500 |

### 3.6 决策语义（上层要能预测 QEMU 什么时候动）

worker 每拍（活跃时连续、空闲时 200ms 一拍）：

```
换出条件：psi-some-avg10 ≥ psi-trigger  且  resident > min-resident (+ hysteresis 起动点)
          且 page_idle 可用  且  后端有空闲 chunk（否则先 discard 死 chunk → 压实 → 回收 swap-cache slot）
          速率 ≤ max-evict-mbps
换出对象：连续 idle-streak 轮扫描都 idle、且换入后已过 cooldown-ms 的页；全零页不写后端；
          swap-cache 命中（内容未变）的页不写后端
主动换回：resident < target-resident  且  psi-some-avg10 < swapin-watermark，速率 ≤ max-swapin-mbps
          对象 = 最近 25.6s 内换出的页，最近优先
缺页换入：任何时候，客户机访问 SWAPPED 页即读回；不受任何旋钮限制
```

PSI 是**宿主级**信号（`/proc/pressure/memory`），所有 VM 看到同一个值。想让某一台在压力下
先让出内存，靠的是它的 `psi-trigger` 更低、`min-resident-mb` 更小，这就是主机级总控的杠杆。

### 3.7 迁移能力 `memtier-shared-backing`

```json
{ "execute": "migrate-set-capabilities",
  "arguments": { "capabilities": [ { "capability": "memtier-shared-backing", "state": true } ] } }
```

打开后 `migrate` 对分层 region：只发常驻页；换出页不读回，映射以 `"memtier"` savevm 段
（版本 1，每页 12 字节）在停机阶段随设备状态发出；目的端加载该段即进入分层态。首轮 precopy
先发冷页、热页延后。细节见 [`memtier-snapshot-restore-design.md`](./memtier-snapshot-restore-design.md)。

上层约束：

- **能力是 QEMU 进程级持久的**，用完必须关回去，否则后续普通迁移会丢换出页。
- 两端都要设（能力协商）。目的端加载段时要求后端路径可打开且容量 ≥ 最大 slot，否则加载失败。
- 从 `migrate` 开始到 VM 再次 `cont` 之间，换出 / 主动换回 / 打洞 / 压实全部暂停，映射与后端
  稳定；上层拷贝后端要在这个窗口内做。
- 目的端恢复的 region 带 `policy`，**不带 tunables**（override 为空，全部回到预设与派生默认），
  上层恢复后要重放一次 set-policy。

---

## 4. 查询接口

### 4.1 `x-memtier-query`

```json
{ "execute": "x-memtier-query" }                       // 所有 region
{ "execute": "x-memtier-query", "arguments": { "id": "pc.ram" } }
→ { "return": [ MemTierInfo, ... ] }
```

- 不带 `id`：返回所有**已启用** region，未分层时返回 `[]`（这是判断「是否分层」的权威方法，
  不要依赖配置文件）。带 `id` 而该 region 未启用：`memtier not enabled for region 'X'`。
- 返回的 `enabled` 当前恒为 true（disable 后 region 即从列表删除）。
- 开销很小（读计数器，不扫页表），可以每秒级轮询。
- 已知副作用：每次 query 向 QEMU 的 stderr 打两行 `memtier-dbg:` / `memtier-lat:` 诊断
（线上日志会按采样频率增长；待去掉）。
- 所有 `*-pages` / `*-ios` / `*-bytes` 累计量是单调计数器，`enable` 时归零，`disable` 后消失；
  水位量是瞬时值。换出页统一 4K，页数 × 4096 = 字节。

### 4.2 `MemTierInfo` 字段

#### 标识与配置

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | str | region 名 |
| `enabled` | bool | 恒 true |
| `path` | str | 后端路径（`spdk:` 前缀表示 SPDK） |
| `policy` | enum | 当前预设 |
| `tunables` | object | 16 个旋钮的**生效值**（钳制后；含 enable-time 旋钮），上层用它核对配置是否落地 |

#### 内存水位（gauge，字节）

| 字段 | 说明 | 关系 |
|---|---|---|
| `total-bytes` | region 大小 | |
| `resident-bytes` | 在宿主 RAM 的页（含 `RESIDENT_CLEAN`，不含从未映射过的页） | resident + swapped ≤ total，差值 = 从未映射或零页消除过的页 |
| `swapped-bytes` | 只在后端的页 | 下刷比例 = swapped / total |
| `clean-resident-bytes` | 在 RAM 且仍持有后端 slot 的页（swap-cache） | ≤ resident |

#### 后端空间（gauge）

| 字段 | 说明 |
|---|---|
| `backing-total-bytes` | 容量（slot 数 × 4K） |
| `backing-used-bytes` | 已离开空闲集合的 slot，**含**死 chunk 与已关闭 chunk 内的空洞；≥ swapped + clean-resident |
| `backing-largest-free-bytes` | 历史字段，日志分配器下恒 ≤ 2MB（一个 chunk）；上层用 `backing-free-chunks` |
| `backing-chunk-bytes` | chunk 大小，2MB |
| `backing-chunks` | chunk 总数 |
| `backing-free-chunks` | 可写的空闲 chunk。≤ 4（2 个压实保留 + 2）时 worker 开始压实；为 0 时换出暂停 |
| `backing-dead-chunks` | 无存活 slot、等待 discard 的 chunk |

#### 换出（counter）

| 字段 | 说明 |
|---|---|
| `evicted-pages` | 成功换出的页（含零页消除与 swap-cache 免写的页） |
| `evict-aborts` | 换出途中被客户机写而回滚的页 |
| `evict-zero-pages` | 换出时全零、未写后端的页 |
| `evict-clean-pages` | swap-cache 命中、未写后端的再换出页 |
| `evict-reevict-pages` | 换入后再次换出的页（抖动的直接度量） |
| `evict-sampled-pages` / `evict-sampled-clean-pages` | 再换出页中做过哈希比对的数 / 其中内容未变的数 |
| `evict-verify-reads` | `swapcache-verify` 触发的读回比对次数（关着时应为 0） |
| `clean-slot-reclaims` | 后端满时被迫放弃 slot 的 swap-cache 页 |
| `evict-mbps` | 近 1s 换出吞吐（gauge） |

#### 换入（counter，除注明外）

| 字段 | 说明 |
|---|---|
| `swapin-faults` | 客户机缺页触发的读回 |
| `zero-fills` | 用零页满足的缺页（从未映射页 + 零页消除过的页），不读后端 |
| `active-swapins` | 主动换回预取的页 |
| `swapin-errors` | 致命读回失败。非 0 意味着进程已经 / 即将 `exit(1)` |
| `swapin-lat-samples` | 延迟分解的样本数 |
| `swapin-avg-wait-ns` | 缺页被 fault 线程接到 → 提交读 的平均 ns（排队） |
| `swapin-avg-device-ns` | 提交 → 完成被收割 的平均 ns（设备 + 完成感知） |
| `swapin-avg-publish-ns` | 完成 → UFFDIO_COPY 完成、vCPU 被唤醒 的平均 ns |
| `swapin-max-ns` | 最差一次端到端 ns |

三段平均值是**自 enable 以来的累计平均**，不是滑动窗口；上层要看近期值需自己对
`samples × avg` 做差分。

#### 后端写 I/O（counter，NVMe 磨损监控）

| 字段 | 说明 |
|---|---|
| `evict-ios` | 换出写次数 |
| `evict-io-bytes` | 换出写字节（后端设备看到的宿主写量，不含压实） |
| `evict-io-hist` | 按写大小分桶的次数：`[{size-kib: 4, count}, {8}, … {1024}]` 共 9 桶，末桶含 ≥1M |
| `evict-io-seq` | 与上一次写尾部紧连的写次数 |
| `evict-io-fwd` | 在上一次写之后、一个 chunk 以内前进的写次数（日志序，含零页 / 免写留下的洞），含 seq |
| `backing-discards` | 打洞 / BLKDISCARD 的 chunk 数 |
| `backing-compactions` | 压实次数 |
| `backing-compact-moved-pages` | 压实搬运的页数（宿主侧写放大） |

#### 诊断

| 字段 | 说明 |
|---|---|
| `psi-some-avg10` | 上一拍采到的宿主 PSI memory some avg10（%） |
| `wp-faults-evicting` | 换出在途页上的写保护缺页（正常，等于回滚数的来源） |
| `wp-faults-other` | 其它状态页上的写保护缺页，应为 0；非 0 是实现问题 |

### 4.3 监控派生指标与告警建议

记 Δ 为两次采样之差，T 为采样间隔。

| 指标 | 公式 | 用途 |
|---|---|---|
| 下刷比例 | `swapped / total` | 分层效果 |
| 换出速率 | `Δevicted-pages × 4K / T` 或直接 `evict-mbps` | 总控限速是否生效 |
| 缺页换入速率 | `Δswapin-faults / T` | 客户机受影响程度 |
| 抖动率 | `Δevict-reevict-pages / Δevicted-pages` | 高则调大 cold-runs / idle-streak / cooldown |
| 免写比例 | `(Δevict-zero-pages + Δevict-clean-pages) / Δevicted-pages` | swap-cache 与零页消除的收益 |
| 后端写放大 | `(Δevict-io-bytes + Δcompact-moved-pages × 4K) / (Δevicted-pages × 4K)` | 磨损 |
| 日志顺序率 | `Δevict-io-fwd / Δevict-ios` | 正常 ≈ 100%；下降说明分配器退化 |
| 近期缺页延迟 | `Δ(samples × avg) / Δsamples` 三段分别算 | 尾延迟来源定位（排队 / 设备 / 发布） |
| 后端占用 | `backing-used / backing-total`，`backing-free-chunks` | 容量告警 |

建议告警：

| 级别 | 条件 |
|---|---|
| critical | `swapin-errors > 0`，或收到 `MEMTIER_SWAPIN_ERROR`，或 QEMU 进程退出 |
| warning | `MEMTIER_BACKING_FULL`；`backing-free-chunks ≤ 4` 持续；`swapin-max-ns` 跨过阈值；抖动率 > 10% |
| info | `wp-faults-other > 0`（实现诊断）；`evict-verify-reads > 0` 而 verify 关着（实现诊断）；`zero-fills` 占缺页比例异常高（客户机大量碰新内存） |

### 4.4 事件

| 事件 | 字段 | 频率 / 后续 |
|---|---|---|
| `MEMTIER_BACKING_FULL` | `id`, `used-bytes`, `total-bytes` | 后端没有空闲 chunk 且压实 / 回收都没腾出空间。每个 worker 拍最多一次，持续满时约每 200ms 一次，上层要去重。换出暂停，换入正常，无正确性影响；空间释放（客户机释放页、压实）后自动恢复 |
| `MEMTIER_SWAPIN_ERROR` | `id`, `gfn` | 读回失败或 UFFDIO_COPY 失败。页内容不可恢复，QEMU 紧接着 `exit(1)`：上层按 VM 崩溃处理，不要尝试在该事件后发 QMP |

事件只在 QMP 连接上启用了事件接收时送达（默认开）。

### 4.5 迁移流中的 `"memtier"` 段

`memtier-shared-backing` 打开时，停机阶段在 ram 之后发一个 `SECTION_FULL`：

```
be32 n_regions
per region: str id | str path | u8 policy | be64 nr_pages | be64 n_swapped | n_swapped × {be64 gfn, be32 slot}
```

snapshot-helper 能定位、重写路径或把它剪掉并合并换出页；见
[`memtier-snapshot-restore-design.md`](./memtier-snapshot-restore-design.md) §2.1 与 §3–4。

---

## 5. 给上层设计的约束清单

1. **真相在 QEMU。** 是否分层、生效旋钮、水位都以 `x-memtier-query` 为准；配置文件只是意图。
2. **enable 不幂等。** 重复 enable 报 `memtier already enabled`；快照恢复 / 热迁移目的端会带着
   分层状态起机，启动流程要先 query 再决定 enable 还是 set-policy。
3. **override 只增不减。** set-policy 是合并语义且没有清除接口；上层应整体下发期望旋钮全集。
4. **三项 enable-time 旋钮**（`fault-shards`、`io-poll`、`busy-poll`）要改只能 disable 再 enable。
5. **disable 阻塞 QMP**，耗时和内存需求都 ∝ `swapped-bytes`；先异步换回再 disable。
6. **迁移能力要复位**，且设置窗口内后端稳定，拷贝后端在 `cont` 之前完成。
7. **恢复后 tunables 丢失**，只保留 policy；上层重放。
8. **计数器在 enable 时归零、disable 后消失。** 监控要处理 region 消失与重置（视为 counter reset）。
9. **PSI 是宿主级的**，同机所有 VM 共享；每 VM 差异化只能靠 `psi-trigger` / `min-resident-mb` /
   `max-evict-mbps`。
10. **后端 ≥ RAM** 才不会触发 swap-cache 降级与 `BACKING_FULL`；文件后端建议预写零（避免
    unwritten 转换），裸分区建议保持 `backing-discard=true`。
11. **一个 region 一个后端**，后端不能在 VM 间共享（无元数据、无锁）。
12. **`MEMTIER_SWAPIN_ERROR` 即进程退出**，按崩溃处理。
13. **query 会向 stderr 打诊断行**，日志采集要能容忍或等待该行为去掉。
14. **stubs 构建的识别**不能靠命令是否存在。

---

## 6. 交互样例

```jsonc
// 开启：4 个缺页分片，常驻下限 2GB，跟随宿主压力
{ "execute": "x-memtier-enable",
  "arguments": { "path": "/dev/nvme1n1p3", "policy": "balanced",
                 "tunables": { "min-resident-mb": 2048, "fault-shards": 4 } } }

// 总控下发：压力下先让这台出让内存，限速 300MB/s
{ "execute": "x-memtier-set-policy",
  "arguments": { "tunables": { "psi-trigger": 2, "min-resident-mb": 1024, "max-evict-mbps": 300 } } }

// 暂停换出但保留分层（例如热迁移前）
{ "execute": "x-memtier-set-policy", "arguments": { "tunables": { "psi-trigger": 100 } } }

// 异步把全部页换回（为 disable 做准备）
{ "execute": "x-memtier-set-policy",
  "arguments": { "tunables": { "psi-trigger": 100, "target-resident-mb": 8192,
                               "swapin-watermark": 99, "max-swapin-mbps": 2000 } } }

// 查询（返回截取，字段齐全见 §4.2）
{ "execute": "x-memtier-query" }
{ "return": [ {
    "id": "pc.ram", "enabled": true, "path": "/dev/nvme1n1p3", "policy": "balanced",
    "tunables": { "cold-threshold": 2, "cold-runs": 5, "psi-trigger": 10, "max-evict-mbps": 500,
                  "min-resident-mb": 2048, "cooldown-ms": 1000, "hysteresis-mb": 0, "idle-streak": 2,
                  "target-resident-mb": 2048, "swapin-watermark": 5, "max-swapin-mbps": 256,
                  "fault-shards": 4, "swapcache-verify": false, "backing-discard": true,
                  "io-poll": false, "busy-poll": false },
    "total-bytes": 8589934592, "resident-bytes": 2415919104, "swapped-bytes": 5905580032,
    "clean-resident-bytes": 268435456,
    "backing-total-bytes": 8589934592, "backing-used-bytes": 6442450944,
    "backing-largest-free-bytes": 2097152, "backing-chunk-bytes": 2097152,
    "backing-chunks": 4096, "backing-free-chunks": 1024, "backing-dead-chunks": 3,
    "backing-discards": 120, "backing-compactions": 4, "backing-compact-moved-pages": 310,
    "evicted-pages": 1900000, "evict-aborts": 1200, "evict-zero-pages": 130000,
    "evict-clean-pages": 42000, "evict-reevict-pages": 61000,
    "evict-sampled-pages": 61000, "evict-sampled-clean-pages": 42000,
    "evict-verify-reads": 0, "clean-slot-reclaims": 0, "evict-mbps": 180,
    "swapin-faults": 95000, "zero-fills": 8000, "active-swapins": 0, "swapin-errors": 0,
    "swapin-lat-samples": 95000, "swapin-avg-wait-ns": 9000, "swapin-avg-device-ns": 85000,
    "swapin-avg-publish-ns": 6000, "swapin-max-ns": 2100000,
    "evict-ios": 61000, "evict-io-bytes": 7000000000, "evict-io-seq": 53000, "evict-io-fwd": 60900,
    "evict-io-hist": [ {"size-kib": 4, "count": 2000}, {"size-kib": 8, "count": 1500},
                       {"size-kib": 16, "count": 3000}, {"size-kib": 32, "count": 9000},
                       {"size-kib": 64, "count": 20000}, {"size-kib": 128, "count": 25000},
                       {"size-kib": 256, "count": 500}, {"size-kib": 512, "count": 0},
                       {"size-kib": 1024, "count": 0} ],
    "psi-some-avg10": 3.2, "wp-faults-evicting": 1200, "wp-faults-other": 0 } ] }

// 事件
{ "event": "MEMTIER_BACKING_FULL", "data": { "id": "pc.ram", "used-bytes": 8589934592, "total-bytes": 8589934592 } }
{ "event": "MEMTIER_SWAPIN_ERROR", "data": { "id": "pc.ram", "gfn": 123456 } }
```
