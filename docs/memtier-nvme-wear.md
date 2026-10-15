# 内存分层对 NVMe 寿命的影响：测量与优化方案

> 问题：分层频繁换出带来的小块随机写会快速消耗 NVMe 的写入寿命（TBW）。
> 本文记录测量手段、实测数据与优化路线。QEMU 侧统计项已进 `x-memtier-query`
> （`evict-ios / evict-io-bytes / evict-io-hist / evict-zero-pages / evict-reevict-pages /
> evict-sampled-pages / evict-sampled-clean-pages`，qemu 分支 `memtier-stats`）。
>
> 状态：统计项已实现并在 10.8.1.195 上编译通过；首轮实测进行中（2026-10-15）。

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

## 4. 实测

（首轮 baseline 结果待填）

## 5. 优化方案

### 方案 A：降写入量（P0）

1. **零页不落盘**：换出时 `buffer_is_zero`，置 `SWAPPED_ZERO` 不占 slot，缺页用
   `UFFDIO_ZEROPAGE` 填回。
2. **swap-cache**：按 `memtier-swapcache-design.md` 实现 `RESIDENT_CLEAN`，换入后保留 slot
   并写保护，未改的页再换出零 IO。后端需 ≥ 客户机 RAM（现有预分配已满足）。
3. **主机级写入预算**：hyper-agent 总控按盘 TBW 与目标寿命折算日预算，以
   `data_units_written` 闭环压 `max-evict-mbps`。纯编排改动。

### 方案 B：分块循环日志加打洞（P1）

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
