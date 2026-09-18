# 内存分层(memtier)瓶颈分析

> 基于 `qemu/memtier/`(memory-tiering 分支)源码 + 123 上 redis-benchmark 实测
> (见 `docs/memtier-redis-benchmark.md`:E5-2673 v3 上传统主机 NVMe swap 比 memtier
> 快 2–5×,深超分比差距更大)。本文定位 memtier 慢在哪、为什么、怎么改。

---

## 0. 结论速览

按对随机访问吞吐的影响排序的瓶颈:

| # | 瓶颈 | 位置 | 影响 |
|---|---|---|---|
| 1 | **换入一次 1 页、同步阻塞、无预读** | `fault.c:memtier_swapin_blocking` | **压倒性主因**;每个 miss 暴露全部延迟,内核 swap 靠簇预读一次带回 ~8 页 |
| 2 | **单 fault 线程串行所有 vCPU 缺页** | `fault.c:memtier_fault_loop`(每 region 1 线程) | 多 vCPU 缺页排队,无每-CPU 并行 |
| 3 | **page_idle 在 Haswell-EP 失效 → write-aging,每次写加 WP 缺页** | `worker.c` + `scan.c` | 硬件相关税,与 #1 叠加 |
| 4 | **换出串行不流水**(drain-after-each-run) | `worker.c:evict_cell_runs` | 次要(后台);深压力下追不上换入 |
| 5 | **主动换回默认关 + 按 recency 非热度 + 满载被 PSI 挡掉** | `worker.c:memtier_swapback_tick` | 尾延迟优化,对均匀随机无效 |

**最高杠杆的单点改进:给换入加簇预读**——利用"连续冷 run → 连续 slot"的既有盘上布局,
缺页时一次大读把相邻已换出页一并带回。

---

## 1. 换入(swap-in)路径 —— 头号瓶颈

`fault.c:memtier_swapin_blocking`(`fault.c:26`):

```c
io_uring_prep_read_fixed(sqe, 0, bounce, TIER_PAGE_SIZE,           /* 只读 1 个 4KB 页 */
                         (uint64_t)slot * TIER_PAGE_SIZE, idx);
io_uring_submit(&r->swapin_ring);
io_uring_wait_cqe(&r->swapin_ring, &cqe);                          /* 阻塞等它完成 */
uffd_copy_page(r->uffd, HVA_OF(r, gfn), bounce, TIER_PAGE_SIZE, false);
```

问题:

- **一次只读一页**(`TIER_PAGE_SIZE`),无预读、无相邻页批量。
- **同步阻塞**:`io_uring_submit` 后立刻 `io_uring_wait_cqe`,完全没用上 `SWAPIN_QD=64` 的队列深度和 64 个 bounce 缓冲——提交一个就等一个。
- **批内串行**:`memtier_fault_loop` 一次 `poll` 取最多 16 个 uffd 事件(`fault.c:217`),`handle_fault` 在 for 循环里**逐个**调上面的阻塞读(`fault.c:218-220`)→ 16 个缺页 = 16 次串行往返。
- **单线程**:每 region 只 1 个 fault 线程(`memtier.c:423`)。N 个 vCPU 同时缺页全部串行排队。

### 对比内核 swap

内核缺页走 **swap 簇预读**(`swapin_readahead`,`page_cluster` 默认 3 → 一次读回 `2^3=8` 个页),
且块层 + 多 CPU 可有多个异步读在飞。redis 的"随机"访问其实有微局部性(同一 4KB 页里多个
value、相邻哈希桶),内核预读顺手把邻页带回命中;memtier 每次只读 1 页,把全部延迟暴露在
每个 miss 上。这正是 benchmark 中内核 swap 快 2–5× 的主因,**深超分比 miss 率越高、差距越大**。

---

## 2. 换出(eviction)路径 —— 合并连续但串行

`worker.c:memtier_evict_run`(`worker.c:103`)本身是合理的:

- 把 coldmap 里**连续冷页段**切成最多 `EVICT_MAX_RUN=128` 页的 extent;
- 一次 `uffd_change_protection` 写保护整段(`worker.c:131`);
- **一个大顺序 O_DIRECT 写**落连续 slot(`worker.c:143`,零拷贝)。

**它不刻意等待凑页**,而是把当前找到的最大连续冷段一次写出。两个问题:

- **每个 run 写完即阻塞等完成**:`evict_cell_runs` 在每次 `memtier_evict_run` 后立刻
  `memtier_drain_evict_completions(r, true)`(`worker.c:196`,wait_all=true)→ `EVICT_QD=256`
  的深度没用于流水,换出串行。
- **碎片化冷页 → 大量小写**:冷页不连续时每段就一两页,退化成小 IO。

换出是后台动作,延迟不致命;但深比率下**追不上换入速度 + 争 vCPU 的 CPU**,导致 resident
漂移、换入压力更大。次要瓶颈。

---

## 3. 主动换回(proactive swap-back)—— 有机制但默认关、按 recency 非热度

机制在 `worker.c:memtier_swapback_tick`(`worker.c:224`)+ `fault.c:memtier_active_swapback`,但:

- **默认关闭**:`target_resident == min_resident`,第一道 gate(`worker.c:227`)直接返回。
- **按 recency 排序**:走 recall 环 newest-first(`worker.c:247`),是"最近换出"而非"最热"。
- **一次一页串行**:`SWAPBACK_QD=8` 但 `io_uring_wait_cqe` 逐个等(`fault.c:122`)。
- **满载被 PSI 挡掉**:`swapin_watermark` 被钳到 `< psi_trigger`(`worker.c:230`),benchmark
  满载时主机 PSI 高 → 主动换回根本不触发。

对**均匀随机访问**:下一个访问哪页无法预测,任何预取都救不了(本测试最坏情况)。
对**有稳定热集**的真实负载:主动换回能把热页留驻、削缺页尾延迟,但要真有用需改成
**按 per-page 访问频度排序预取最热的换出页**,而非现在的"最近换出"。它是尾延迟优化,
不是 benchmark 的解药。

---

## 4. 内核 swap 逻辑的指导意义

| 内核机制 | memtier 现状 | 借鉴改进(按价值排序) |
|---|---|---|
| **swap 簇预读**(一次读邻近 N 页) | 一次 1 页 | **★最高价值**:缺页时连带预读相邻 slot。换出本就写连续 run→连续 slot,邻页在盘上顺序,一次大读几乎零额外成本填多页 |
| **多 IO 在飞 / 每 CPU 并行缺页** | 单 fault 线程、submit 即 wait | 多 fault 线程,或用满 64 深 ring 异步提交、完成时再 COPY |
| **swap 写出聚簇**(为预读铺垫) | 已写连续 run→连续 slot | 已具备底子,缺读侧利用 |
| **大 folio 换入换出**(多页粒度) | 强制 4K(`MADV_NOHUGEPAGE`) | 冷区用多页粒度,摊薄每页 uffd/syscall 开销 |
| **refault 距离 / workingset 检测** | page_idle / write-aging | refault 驱动更鲁棒,**且在 Haswell-EP 上照常工作**(page_idle 失效那台正缺这个) |
| **eviction 流水** | drain-after-each-run | 用 EVICT_QD 深度做多 run 在飞 |

---

## 5. 改进路线(本仓后续落地顺序)

1. **换入簇预读**(本轮实现):缺页 gfn 落在某个已换出的连续段内时,检测其在盘上相邻的、
   仍 SWAPPED 且 slot 连续的页,一次 `io_uring` 大读把整簇读回、逐页 `UFFDIO_COPY` 填回并置
   RESIDENT。预读窗口上限(如 16 页 = 64KB)对齐 bounce 池。**只读已 SWAPPED 且 slot 连续的页**
   (避免读到无关数据/越界)。
2. **异步多缺页**(后续):fault 线程把一批 uffd 事件的读一次性提交满 ring,完成时再逐个 COPY,
   消除 submit-after-each-wait 的串行。
3. **eviction 流水**(后续):去掉 run 间的 wait_all,用 EVICT_QD 让多 run 在飞,完成时回收。
4. **热度感知主动换回**(后续):recall 环改按访问频度排序;放宽 PSI gate 让其在有热集时也能预取。

> 关键洞察:memtier 的盘上布局(连续冷 run → 连续 slot,`worker.c:memtier_evict_run`)天然适合
> 簇预读——读侧只要顺着把相邻 slot 一起读回即可。

---

## 6. 实验:换入簇预读 —— **实测反而变慢,暴露更深的架构问题**

按 §5.1 实现了换入簇预读(`MT_RA_WINDOW=8`:缺页时把盘上相邻、仍 SWAPPED 且 slot 连续的
forward 邻页一次大读回来、一次多页 `UFFDIO_COPY` 填回),在 123 复测 NVMe 1.5×:

| | GET rps | GET p99 | 累计缺页 | resident/swapped |
|---|---|---|---|---|
| 原版(无预读) | **17 584** | 9.9 ms | 230 K | 4103 / 2040 MB |
| 加簇预读(window 8) | **4 459** | 48.6 ms | **454 K** | 4187 / 1956 MB |

**预读让吞吐掉了 4 倍、缺页数翻倍。已回退。**

### 根因:memtier 对投机页"双重付费",内核 swap 不会

- **内核**的预读页放进 **swap cache**:没被访问就**直接丢弃**(页是干净的、副本已在 swap 上,
  丢弃零成本)。
- **memtier**的预读页被 `UFFDIO_COPY` 变成**完全 RESIDENT**:在 resident floor 顶着时,worker
  为维持 floor 必须把这些没用到的投机页**再换出(WP + 写回 backing + DONTNEED)**——
  **刚从盘上读进来、又写回盘**,纯浪费一次读 + 一次写,还触发 eviction churn(每个真实缺页
  诱发约 7 个投机页的换出写)。miss 率越低(浅比率)污染占比越高,所以 1.5× 退化最惨。

> 即:**簇预读单独上是错的**。它只在"投机页未用时能零成本丢弃"的前提下才成立,而 memtier
> 当前缺这个前提。

### 真正的修复:swap-cache 模型(slot 保留 + 干净页免写回换出)

要让预读(以及一般换入)真正变快,必须照搬内核的 swap-cache 语义:

1. **换入不释放 slot**:页换入后保留其 backing slot(数据仍有效副本),并对页加 **uffd-WP**;
   新增页状态 `RESIDENT_CLEAN`(驻留、只读、slot 有效)。
2. **干净页免写回换出**:worker 换出 `RESIDENT_CLEAN` 冷页时,**只 `MADV_DONTNEED` + 转 SWAPPED
   复用原 slot**,**不写盘、不分配新 slot**——零 IO。这正是内核丢弃干净 swap-cache 页的等价物。
3. **写时转脏**:guest 写 `RESIDENT_CLEAN` 页 → uffd-WP 缺页 → 释放 slot(盘上副本作废)、解 WP、
   转普通 `RESIDENT`(脏),此后换出需正常写回。
4. 配合后,**簇预读的投机页未用时再换出是零成本的**(只 DONTNEED,不写盘),预读净收益转正;
   读多写少(GET 为主)的负载收益最大。

**代价与风险**:① backing 需 ≥ 客户机 RAM(干净驻留页也占 slot,而非只占已换出页);
② WP-on-swapin 给每个换入页的"首次写"加一次 WP 缺页(SET 为主的负载略有回归);
③ slot 紧张时需回收干净驻留页的 slot(把它们标脏);④ 与迁移 quiesce、eviction 的 WP/锁
交互都要逐一处理。这是 concurrency 敏感的较大改动,**一个 bug = 客户机内存损坏**,必须配
`memtier/tests/` 的 sha256 换出/换入数据完整性用例逐步验证,**不能在长会话里仓促上线**。

### 本轮结论

- 簇预读已实现、实测、**因架构原因回退**;123 已恢复 known-good 二进制(baseline ~60k 复核一致)。
- **下一步(验证过方向、待谨慎实现)**:swap-cache 模型(slot 保留 + 干净页免写回换出 + WP 转脏),
  再在其上叠加簇预读 + 异步多缺页。预期这才能让 memtier 在 NVMe 随机访问上逼近/追平内核 swap。

> 教训:把内核 swap 的"一次读多页"照搬到 memtier 之前,得先有内核 swap 的"干净页免成本丢弃"。
> 两者是配套的;只搬前者会被换出写回放大抵消。这也修正了 §5 里"最小改动逼近内核 swap"的判断——
> 真正的最小**有效**改动是 swap-cache,而非单独的预读。
