# 内存分层对 NVMe 寿命的影响：测量与优化方案

> 问题：分层频繁换出带来的小块随机写会快速消耗 NVMe 的写入寿命（TBW）。
> 本文记录测量手段、实测数据与优化路线。QEMU 侧统计项已进 `x-memtier-query`
> （`evict-ios / evict-io-bytes / evict-io-hist / evict-zero-pages / evict-reevict-pages /
> evict-sampled-pages / evict-sampled-clean-pages`，qemu 分支 `memtier-stats`）。
>
> 状态：统计项、零页不落盘、swap-cache、分块循环日志加打洞均已实现（qemu `memtier-stats` 分支），对比实测见 §4：写量较 baseline 少 24%，98% 的写落在前一次写的紧后面；主机级写入预算已在 hyper-agent 实现（单测）（2026-10-15）。

---

## 1. 先纠正前提

**磨损 = 主机写入字节数 × 写放大系数（WAF）**。

- 改顺序写、改日志结构只能压 WAF（FTL 内部搬运倍数）。随机 4K 写在消费级盘上 WAF 常见
  3 到 5，企业盘 1.5 到 3；大块顺序写约 1.0 到 1.2。
- 写入量由换出量决定：换出多少页就写多少字节。零页、换入后未改又换出的干净页，是
  纯粹可以省掉的写。

两项都要做，写入量这一项的杠杆通常更大。

## 2. 现状（代码事实，`qemu/memtier`）

| 项 | 现状 | 影响 |
|---|---|---|
| 单次换出写 | `EVICT_MAX_RUN=8`，32KB，QD1 | 小写。刻意为保护换入尾延迟从 512KB/QD16 降下来（慢盘上一次 512KB 写让碰撞的读等 3.5ms） |
| slot 分配 | best-fit 且最低地址优先（`slot-alloc.c`） | 换入即 `slot_free`，空洞被小段填补，LBA 越跑越散 |
| 零页 | 换出不识别全零页 | 空闲内存多的客户机白写零 |
| 干净页重写 | 换入即释放 slot，再换出必重写（swap-cache 设计未实现） | 来回的页每次都写 |
| TRIM | 无打洞、无 discard；后端 `fallocate` 全量预分配 | FTL 不知道哪些块已死 |
| 写入上限 | `max-evict-mbps`，按性能设 | 唯一现成的写量闸门 |

## 3. 测量方法

三路数据同时抓，互相印证：

1. **QEMU**：`x-memtier-query` 新增字段。`evict-io-hist` 看写有多碎；`evict-zero-pages` 和
   `evict-sampled-clean-pages / evict-sampled-pages` 直接给出方案 A 两项各能省多少。
2. **内核块层**：`/sys/block/nvme0n1/stat` 的写请求数与扇区数，看块层合并后设备收到的
   请求大小。
3. **NVMe 控制器**：`nvme smart-log` 的 `data_units_written`（主机写入）、`percentage_used`。
   消费级盘不暴露 NAND 写入量，WAF 只能按 IO 大小分布推断；企业盘可用厂商日志页直接算。

脚本：测试机 `/root/work/memtier-test/iopoll/wear-run.sh <tag> [backing] [tunables]`，
包一层 `run-redis.sh`（8 vCPU / 8GB 客户机，redis 预灌约 4GB，驻留下限 4096MB，
200 万次均匀随机 GET + 100 万次 SET），前后抓三路计数并打印汇总。

## 4. 实测：baseline（2026-10-15，10.8.1.195）

条件：Xeon Gold 6240，Great Wall GW3500 256GB 消费级 NVMe + XFS，后端 `backing.img` 16GB
O_DIRECT；客户机 8 vCPU / 8GB，redis 预灌 4.9GB（384 万键 × 1KB）；`aggressive`，
`psi-trigger=0`，驻留下限 4096MB，`max-evict-mbps=4000`，`cooldown-ms=200`；默认 io_uring
路径（未开 io-poll）。流程：enable → 20s 内下刷到 4095MB → 200 万次均匀随机 GET → 100 万次
SET → disable。全程 169s。

### 4.1 三路计数对照

| 来源 | 写次数 | 写字节 | 平均每次 |
|---|---|---|---|
| QEMU `evict-ios / evict-io-bytes` | 487,897 | 5.72 GiB | 12.3 KiB |
| 块层 `/sys/block/nvme0n1/stat` | 485,399（合并 5,051） | 5.73 GiB | 12.4 KiB |
| NVMe `host_write_commands / data_units_written` | 485,399 | 5.73 GiB | 12.4 KiB |

三路一致到 1%：XFS 上 O_DIRECT 没有额外元数据写，块层几乎不合并，QEMU 发出什么设备就
收到什么。换出 1,499,453 页（5.72 GiB），平均写入速率 35MB/s，下刷阶段峰值约 200MB/s。

### 4.2 写大小分布（`evict-io-hist`）

| 大小 | 次数 | 占写次数 | 占字节 |
|---|---|---|---|
| 4K | 231,628 | 47% | 15% |
| 8K | 119,131 | 24% | 16% |
| 16K | 32,396 | 7% | 9% |
| 32K | 104,742 | 21% | 56% |

近一半的写命令是单页 4K，四分之三不超过 8K。`EVICT_MAX_RUN=8` 把上限钉在 32K，而冷页
在客户机地址空间里本来就是碎的（热页夹在中间），连续冷段往往只有一两页；再叠加
best-fit 分配把这些小写撒到被换入腾出的零散空洞里，FTL 看到的是典型的随机小写。

### 4.3 可避免的写（方案 A 的收益）

| 项 | 数值 | 含义 |
|---|---|---|
| `evict-zero-pages` | 108,060（7.2%） | redis 灌满的客户机仍有 7% 零页；一般客户机会高得多 |
| `evict-reevict-pages` | 135,524（9.0%） | 换入后又被换出的页 |
| `evict-sampled-clean-pages / sampled` | 4,237 / 4,274（**99.1%**） | 再换出的页几乎全部未被修改 |

即 swap-cache 能在这个负载下省掉约 9% 的写，零页省略再省 7%，合计约 16%。GET 为主的
负载里再换出比例不高；换入换出来回更频繁的负载（工作集略大于驻留下限、抖动）这个比例
会显著上升，而 99% 的"未修改"说明只要页回来过，几乎都值得留 slot。

### 4.4 结论

1. **写量本身不大但很碎**：35MB/s 平均、12KB 平均一次，半数是 4K 随机写。这是消费盘
   WAF 最差的形态，寿命问题主要来自 WAF 而不是写量。这块盘不暴露 NAND 写入量，WAF
   无法直接测；需要企业盘（厂商日志页）才能给出实数。
2. **方案 B（分块顺序日志）正对症**：它不改变单次 IO 大小，但把这 48 万次小写变成 LBA
   连续的流，FTL 能整块回收。
3. **方案 A 在此负载省约 16%**，在抖动负载下更多；零页省略对普通客户机收益更大。
4. 现有换出路径没有任何文件系统放大，分块日志不会引入新的放大源。

脚本与原始数据：`/root/work/memtier-test/iopoll/wear-run.sh`，`results/wear-base/`。

### 4.5 零页不落盘（方案 A-1）实测

实现：qemu `memtier-stats` 分支 `60f6b39657`。换出 run 写保护后逐页 `buffer_is_zero`，
全零页不写后端，和邻页一起 `MADV_DONTNEED` 后状态保持 RESIDENT、无 slot，等价于
"从未填充"的页：冷扫描跳过非在场页，converge / 迁移把它当普通读为零的内存，客户机
下次访问走已有的 `UFFDIO_ZEROPAGE` 路径。写范围裁剪到 run 内非零页的跨度；整段全零的
run 不发 IO。没有新增页状态，memtier 段与快照格式不变。

同一流程、同一客户机镜像重跑（233s，含重新预灌）：

| 指标 | baseline | 零页不落盘 | 变化 |
|---|---|---|---|
| 换出页数 | 1,499,453 | 1,396,330 | 负载抖动 |
| 后端写字节（QEMU = 块层 = NVMe） | 5.72 GiB | 4.95 GiB（换出页 5.33 GiB） | **写量比换出量少 7.0%** |
| 零页 | 108,060（7.2%） | 105,290（7.5%，全部未写） | |
| `zero-fills` | 无此字段 | 5,906 | 被省掉的页里 5.6% 又被访问，零填充免读盘 |
| GET rps / p99 | 28.6k / 3.50ms | 28.6k / 3.62ms | 持平 |
| SET rps / p99 | 34.1k / 3.14ms | 37.0k / 3.01ms | 持平 |
| `swapin-errors` / `evict-aborts` | 0 / 1 | 0 / 0 | |

零页 105,290 页 = 0.40 GiB，实际少写 0.38 GiB：裁剪只去掉 run 两端的零页，夹在中间的
仍会写（随后丢弃），损失约 5%，与"run 很短"的预期一致。redis 灌满的客户机零页只有
7.5%，收益即为 7%；普通客户机（空闲内存多、页回收后清零）零页比例高得多，收益随之放大。

### 4.6 swap-cache（方案 A-2）实测与两次改道

实现：qemu `memtier-stats` 分支 "memtier: swap-cache"。换入的页保留 slot（状态
`RESIDENT_CLEAN`），再换出时若内容未变则不写，直接在原 slot 上 `SWAPPED`。

**改道一：uffd 写保护在 KVM 下判不了"干净"。** 原设计靠换入时写保护、靠 WP 缺页判脏。实测
纯只读遍历 39 万个换出页产生 39 万次 WP 缺页：KVM 处理 EPT 缺页时无论客户机读写都以写意图
把页钉住，换入后的第一次读就把写保护打掉了。所以换入时不再写保护，改为**换出时比对内容**。

**改道二：按读回比对太慢。** 先做了"换出时读回 slot 逐字节比对"的精确版本，正确性测试全过，
但干净页的换出从零 IO 变成一次读（约 70µs），redis SET 阶段吞吐掉了约 25%（34k → 26.5k rps）。
把比对改成 **128 位 MurmurHash3**：换入时记页哈希（16B/页），换出时重算比对；相等即视为未变
（碰撞概率 2^-128），不同则写回原 slot。保留 `swapcache-verify` 旋钮可额外读回比对。
指纹不同的页直接写回自己的 slot，不再申请新 slot。

同一流程重跑（哈希模式）：

| 指标 | baseline | 零页不落盘 | 零页 + swap-cache |
|---|---|---|---|
| 换出页数 | 1,499,453 | 1,396,330 | 1,386,968 |
| 后端写字节 | 5.72 GiB | 4.95 GiB | **4.44 GiB** |
| 写次数 | 487,897 | 448,248 | **373,559** |
| 写量 / 换出量 | 100% | 93% | **84%** |
| 再换出页 / 其中未变（免写） | 9.0% / 无 | 9.3% / 无 | 107,982 / **107,052（99.1%）** |
| 驻留中持有 slot 的干净页 | 无 | 无 | 2.5 GiB |
| GET rps / p99 | 28.6k / 3.50ms | 28.6k / 3.62ms | **30.1k / 3.40ms** |
| SET rps / p99 | 34.1k / 3.14ms | 37.0k / 3.01ms | **43.8k / 2.33ms** |
| swapin-errors / evict-aborts | 0 / 1 | 0 / 0 | 0 / 0 |

写量比 baseline 少 22%，写次数少 23%；吞吐反而更好，因为干净页的换出不再与缺页读抢 NVMe。
完整性测试（`iopoll/integrity-run.sh`：确定性值 100 万键，强制换出 → 校验 → 压低下限
强制干净页再换出 → 再校验 → 覆写 → 再换出 → 校验 → disable 收敛 → 校验）全部通过。

代价与约束：每客户机页 16B 哈希（32GB 客户机 128MB）；换入与干净页换出各约 1.5µs 哈希
计算；干净页占用 slot，后端需 ≥ 客户机 RAM，满时按游标批量降级干净页释放 slot
（`clean-slot-reclaims`）；SPDK 后端暂不走此路径。

### 4.7 分块循环日志加打洞（方案 B）实测

实现：qemu `memtier-stats` 分支 "memtier: chunked circular-log backing allocator"。
`slot-alloc.c` 重写为 2MB chunk 的日志：打开一个 chunk 顺序 bump 分配，写满换下一个
（空闲 FIFO，最早释放的先用）；释放只计数，chunk 内最后一个活 slot 释放时整块进死队列，
由 worker 打洞（文件 `FALLOC_FL_PUNCH_HOLE`，分区 `BLKDISCARD`）后回空闲 FIFO。热路径
不做 GC；空闲 chunk 只剩 `CHUNK_RESERVE`（2）+2 个时，或分配失败时，worker 压实活率最低的
chunk：其中 swap-cache 副本直接丢弃（页在 RAM），SWAPPED 页以 `SWAPPED_MOVING` 占住、读出
写进保留 chunk（缺页到达则驻留等待，搬完 `UFFDIO_WAKE` 重试）。slot 编号语义不变，快照、
迁移、snapshot-helper 全部不受影响。

**写入模式**（16GB 后端、redis 流程、与上一版本背靠背 A/B）：

| 指标 | swap-cache 版（best-fit 分配） | 分块日志 |
|---|---|---|
| 后端写字节 | 4.43 GiB | **4.34 GiB** |
| 写次数 | 359,640 | 353,897 |
| 与前一次写 LBA 连续的写（`evict-io-seq`） | 无此统计 | **98.2%** |
| 块层合并次数 | 约 6k | 约 23k |
| GET rps / p99 | 30.1k / 3.32ms | 30.7k / 3.30ms |
| SET rps / p99 | 45.4k / 2.09ms | **49.0k / 1.91ms** |

单次写仍是 4K 到 32K，但 98% 落在前一次写的紧后面：FTL 看到的是顺序流，这正是方案 B
要的效果，且性能不降反升。
（`evict-io-seq` 只算"紧跟前一次写"，run 两端被跳过的零页 / 未变页会留洞而不计入；
§4.8 新增 `evict-io-fwd` 度量日志序，两台机器都是 100%。）

**压实压力测试**（1.5GB 后端、1GB 确定性数据、下限 300MB，后端 100% 占满）：日志持续滚动，
压实 896 次、搬 276,738 页、打洞 896 次，换出持续进行，六轮校验全部通过，99.6% 顺序写。
这个极端场景下主机侧写放大约 1.6（搬页量 / 换出量）；16GB 后端的正常运行中压实为 0。

**打洞的两个发现**，决定了默认值：

1. 文件后端打洞后，重新写入该 chunk 的每次写都要付 XFS 的 extent 分配或 unwritten 转换，
   XFS 还会让它与同一文件上并发的 O_DIRECT 换入读串行化；实测打洞开启时 SET 吞吐 27k rps，
   关闭 47k。打洞后立刻 `fallocate(KEEP_SIZE)` 预分配回来也一样慢（unwritten 转换）。
   所以 **`backing-discard` 对文件后端默认关，对裸分区默认开**（`BLKDISCARD` 没有文件系统
   开销）。对磨损敏感的部署应当用裸分区或 namespace 做后端。
2. XFS 不挂 `-o discard` 时打洞不会向盘发 TRIM（本测试机就是），只有周期性 `fstrim` 能补；
   而不打洞的文件 `fstrim` 也无能为力。文件后端要获得 TRIM 收益只能开打洞并接受上述代价。

另一个过程中的教训：前几轮测得分块日志 SET 只有 27k，追了打洞、unwritten、文件碎片
（19,804 个 extent）三个方向都不是，最后背靠背 A/B 证明是机器状态的波动，新旧版本在同一
时段测分别是 45.4k 与 49.0k。单次测量的 SET 吞吐波动可到 40%，结论必须用背靠背 A/B 下。

**修过的两个 bug**：后端满时 `memtier_evict_run` 直接 `return` 跳过了本轮已排队 IO 的
`io_uring_submit`，drain 永远等不到完成（旧代码就有，16GB 后端从不触发）；压实需要空闲
空间才能搬页，没有保留区时后端一满压实就空转，引入 `CHUNK_RESERVE`。

### 4.8 第二台机器复测（2026-10-21，10.8.1.196）

10.8.1.195 失联，harness 与客户机镜像在 10.8.1.196 重建（同型 CPU / 内存；后端盘换成
Great Wall GW3000 512GB，`nvme1n1`：p1 1.5GiB 裸分区、p3 30.5GiB 裸分区、p2 XFS 挂
`-o discard`，内有 16GB `backing.img`）。三个版本（baseline 只含统计项、swap-cache、分块
日志）各自编译，同一台机、同一时段、每组都是新起客户机，顺序交错以排除时段漂移。

**文件后端**（16GB，XFS）：

| 指标 | baseline | swap-cache | 分块日志 | 分块日志（修复后复测） |
|---|---|---|---|---|
| 后端写字节 | 5.13 GiB | 4.66 GiB | 4.64 GiB | 4.56 GiB |
| 写次数 | 328,050 | 318,687 | 319,318 | 317,104 |
| 严格连续写（`evict-io-seq`） | — | — | 87.3% | 87.4% |
| 日志序写（`evict-io-fwd`，新增） | — | — | — | **100.0%** |
| GET rps / p99 | 17.9k / 12.3ms | 19.8k / 11.8ms | 19.8k / 11.7ms | 20.2k / 11.6ms |
| SET rps / p99 | 20.2k / 10.1ms | 22.4k / 9.5ms | 22.8k / 9.3ms | 23.0k / 9.2ms |

**裸分区后端**（`/dev/nvme1n1p3`，`backing-discard` 默认开）：

| 指标 | baseline | swap-cache | 分块日志（两次） | 分块日志（修复后复测） |
|---|---|---|---|---|
| 后端写字节 | 5.17 GiB | 4.64 / 4.66 GiB | 3.94 / 4.61 GiB | 4.61 GiB |
| 写次数 | 329,354 | 319,504 / 318,545 | 353,643 / 318,352 | 319,874 |
| 块层合并后的写请求 | 276,157 | 266,256 | 312,610 / 263,425 | 265,080 |
| 严格连续写 | — | — | 97.1% / 86.8% | 87.1% |
| 日志序写 | — | — | — | **100.0%** |
| BLKDISCARD 次数（QEMU 计数 = 块层计数） | — | — | 155 / 53 | 57 |
| GET rps / p99 | 17.9k / 12.3ms | 20.0k / 11.7ms | 21.1k / 20.4k | 20.6k / 11.4ms |
| SET rps / p99 | 20.1k / 10.3ms | 22.3k / 9.4ms | 30.8k / 22.6k | 22.6k / 9.4ms |

三路计数（QEMU、块层字节、NVMe `data_units_written`）在每一组都 1:1 吻合。

结论与 §4.5–4.7 一致，并补上两点：

1. **写量**：swap-cache 与分块日志的字节数相同（分配器不改变写什么，只改变写到哪），都比
   baseline 少 10–12%；这台机器零页占 3.3%、重复换出 12–14% 且 99.9% 未变，收益来源与
   195 一致。吞吐两者相当，比 baseline 高约 10%，GET/SET p99 各低约 1ms。第一组裸分区分块
   日志 SET 30.8k 是单次波动（换出量也明显少），反序复测 22.6k 与 swap-cache 持平，再次印证
   §4.7 的"必须背靠背 A/B"。
2. **顺序性**：严格"紧跟前一次写"只有 87%，比 195 的 98% 低，追下来是一个度量问题：每个换出
   run 的 slot 是整段分配的，run 两端的零页和未变 swap-cache 页不写，下一次写就从段尾开始，
   LBA 中间留一个几页的洞。这台机器 swap-cache 命中多（17–19 万页），洞就多。新增
   `evict-io-fwd`（前进且落在一个 chunk 之内）度量日志序，修复后两种后端都是 **100.0%**：
   FTL 看到的是单调前进、偶有小洞的流，不是随机写。

**裸分区 BLKDISCARD 路径首次真机验证**：30.5GiB 分区上完整性六轮校验通过；1.5GiB 分区
压实压力测试（后端 100% 占满、日志持续滚动）压实 755 次、搬 92,899 页、`BLKDISCARD` 712 次，
与块层 `/sys/block/nvme1n1/stat` 的 discard 计数增量（1337 → 2049）完全一致，六轮校验通过，
disable 收敛 20s。文件后端同样的压力测试（打洞默认关）压实 105 次、搬 36,887 页、
校验通过。

**顺带修掉的一个 bug**（`verify` 关着却出现 `evict-verify-reads`，195 上就见过 91 次）：
一个 run 里只要有一个"换入后被改过"的 swap-cache 页，其余未变的 swap-cache 页也会被排一次
校验读；`swapcache-verify` 关时没有 bounce 缓冲，读以 `-EFAULT` 失败，完成路径把页当作
"已改"原地重写——每页一次多余的随机 4K 写。修复后计数归零（本节"修复后复测"两列）。

## 5. 优化方案

### 方案 A：降写入量（P0）

1. **零页不落盘**（已实现，见 §4.5）：换出时 `buffer_is_zero`，零页不写、不占 slot，
   状态保持 RESIDENT 等价"从未填充"，缺页走已有的 `UFFDIO_ZEROPAGE` 路径。
2. **swap-cache**（已实现，见 §4.6）：换入后保留 slot，记 128 位哈希，再换出时哈希未变则
   零 IO。写保护在 KVM 下不可用（读也会打掉），读回比对又太慢，故用哈希。后端需 ≥ 客户机 RAM。
3. **主机级写入预算**（已实现）：hyper-agent 总控 `support/memtier/write_budget.py`。
   按后端**盘**记账（文件按所在设备、分区折到整盘、dm/ublk 就是自己），用
   `/sys/block/<dev>/stat` 的累计写扇区数，不依赖 nvme-cli，同盘其它写入者也算在内；
   当天起点与日期持久化在 `/var/lib/xvirt/memtier-write-budget.json`。每拍
   允许速率 = 当天剩余预算 / 当天剩余秒数（下限 `XVIRT_MEMTIER_BUDGET_MIN_MBPS`=8），
   按优先级权重（256:512:1024）分给同盘 VM 作 `max-evict-mbps` 上限；主机压力 ≥ 0.9 时越过
   预算（宁可磨损不 OOM）。配置：plan `memory_tiering.write_budget_gb_per_day`（优先）或
   环境变量 `XVIRT_MEMTIER_WRITE_BUDGET_GB_PER_DAY`，0 = 不限；可用
   `budget_bytes_per_day(tbw_tb, life_years)` 从 TBW 折算。`tiering_host_status` 多
   `write_budget` 字段（每盘预算、当天已用、写速率、允许速率、nvme smart 寿命百分比），
   每 VM 多 `backing_device / budget_cap_mbps / budget_overridden`。未真机验证。

### 方案 B：分块循环日志加打洞（P1）— 已实现，见 §4.7

- 后端按 2MB chunk（对齐冷扫描 cell）顺序推进分配，写满换下一块，尾部回绕。
- 每 chunk 活页计数，`slot_free` 递减，归零整块打洞 / TRIM（文件 `FALLOC_FL_PUNCH_HOLE`，
  分区 `BLKDISCARD`，SPDK deallocate）并进空闲链表，按最早释放复用。
- **slot 编号不变**（仍是 4K 页索引，chunk = slot >> 9），快照 memtier 段、snapshot-helper
  merge、共享后备热迁移都不受影响。
- **单次写可以仍是 32KB**：收益来自 LBA 连续而不是 IO 大小，不推翻为尾延迟做的决策；
  快盘上 run 大小做成按介质设置的旋钮。
- **不做 GC**：半活 chunk 原地保留，只有全死才回收。可选"活率 < 25% 且空闲不足时压实"
  的慢路径兜底空间碎片。
- 取舍：打洞放弃 `fallocate` 的空间保证。寿命敏感部署建议专用分区或 namespace 做后端。

### 不建议

- 攒超过 1 到 2MB 再写：无额外收益，推迟内存释放，写保护期间回滚率上升。
- 带 GC 的严格日志结构：主机侧 WAF 吃掉 FTL 侧节省。
- 压缩：作为 P2，需与分块日志一起设计。

## 6. 量化参考

1TB 企业盘、1 DWPD、5 年，总写入约 1.8PB：

| 场景 | 主机写入 | WAF | 寿命 |
|---|---|---|---|
| 持续 100MB/s 换出，随机小写 | 8.6TB/天 | 3 | 约 70 天 |
| 同上，顺序写 | 8.6TB/天 | 1.1 | 约 190 天 |
| 写量压到 30MB/s，顺序写 | 2.6TB/天 | 1.1 | 约 640 天 |
