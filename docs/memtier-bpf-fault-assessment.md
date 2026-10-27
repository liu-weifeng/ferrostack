# bpf_fault 对内存分层换出 / 换回的适用性评估

> 对象：Zussman 等，*It's the Kernel's Fault! Custom Page Fault Handling With bpf_fault*，
> SOSP '26（DOI 10.1145/3830418.3843896），代码 https://github.com/bpf-fault/bpf-fault 。
> 问题：这套"在内核里用 eBPF 处理缺页"的机制，能否改善 memtier（qemu `memtier-stats`
> 分支）的换出和换回。
>
> 结论（2026-10-27）：换出几乎无收益；换回的收益上限随后端盘延迟下降而上升，
> 在 10 µs 级设备上理论可缩短四成，但 bpf_fault 自身不能做 I/O，拿到收益还需要
> 第二处非主线内核改动。现阶段不引入，记录下触发重新评估的条件。

---

## 1. 论文做了什么

- 把 userfaultfd（uffd）的用户态处理线程换成挂在内核缺页路径上的 eBPF `struct_ops`
  程序，在**触发缺页的线程上下文**里直接处理，没有处理线程、没有上下文切换、
  没有用户态与内核之间的额外数据拷贝。
- 支持 uffd 的三类缺页（MISSING、WP、minor），另外支持文件映射的缺页。处理函数
  收到内核刚分配好的页（MISSING）或现有页（minor / WP 只读），返回 0 表示安装
  PTE，非 0 表示向触发线程发 SIGBUS。
- 论文测得单次缺页延迟 10.3–11.6 µs 降到 1.6–3.7 µs（2.8–6.7 倍），比内核原生缺页
  只慢 1.4 倍；512 线程并发缺页 859 ms 降到 25 ms，因为不再有串行化的处理线程。
- uffd 的 10 µs 里：调度与 IPI 占 43%，系统调用穿越 17%，uffd 协议本身 10%，缺页
  陷入与返回 16%，页准备 15%。
- 应用案例：Firecracker 与 QEMU 的后台快照（WP 缺页改成把旧内容写入 ring buffer
  即放行，QEMU 快照期间最差请求延迟 142 ms 降到 7 ms）、延迟重定位的动态链接、
  JVM 并发压缩、electric fence 内存调试。
- 限制：eBPF 程序不能读写文件、不能访问网络，论文明确说 post-copy 迁移这类要从
  外部取数据的场景做不了。基于 Linux 6.17，内核改动约 2500 行，**未进主线**。
  一个 VMA 只能注册 uffd 或 bpf_fault 之一（共用 `vm_userfault_ctx` 字段）。
  bpf_fault 的 WP 复用 uffd 的 WP PTE 标记位。

本机 Ubuntu 26.04 内核 7.0.0-38 的 kallsyms 与 uapi `bpf.h` 中都没有 bpf_fault
相关符号，测试机同为 7.0 内核。

## 2. memtier 的缺页在哪里

memtier 用 uffd MISSING 模式接管客户机 RAM，相关路径（`qemu/memtier/`）：

| 路径 | 缺页类型 | 发生频率 | 代码 |
|---|---|---|---|
| 缺页换回 | MISSING，页为 SWAPPED | 每次客户机访问已换出页 | `fault.c` `handle_fault` → `swapin_submit`，io_uring / SPDK 读 → `UFFDIO_COPY` |
| 零填充 | MISSING，页从未填充或被零页消除 | 客户机首次触碰新内存 | `fault.c` `uffd_zero_page` |
| 换出窗口内被写 | WP，页为 EVICTING | 冷页在写回 NVMe 期间被客户机写 | `fault.c` WP 分支 → `EVICTING_DIRTIED`，解保护放行，worker 回滚 |
| 写老化回退 | WP | 仅 page_idle 失效（THP）时，所有常驻页都做候选 | `scan.c` `writeage_fallback` |
| 主动换回 | 无缺页 | worker 按 `max-swapin-mbps` 读回并 `UFFDIO_COPY` | `worker.c` |

换出的主要成本不在缺页上：idle 扫描触发的 rmap 走查与 EPT `clear_young`、
`MADV_DONTNEED` 的 EPT 回收与 TLB shootdown、NVMe 写本身。这三项 bpf_fault 都不涉及。

## 3. 对换出的评估

**收益可忽略。** 换出路径里唯一走缺页的是"换出窗口内被写"的 WP 缺页。它对 vCPU 是
约 10 µs 停顿加一次 fault 线程唤醒，bpf_fault 可以在内核里改一个共享 bitmap 直接放
行，降到约 1.6 µs 且不需要 fault 线程。但 page_idle 模式下候选页本来就冷，这类缺页
很少：接口文档样本里 95,000 次换入对应 1,200 次 WP 缺页（`wp-faults-evicting`）。

**写老化回退模式受益更多，但问题本身另有解法。** THP 导致 page_idle 失效时，所有
常驻页都做候选，每次写都要过一遍 uffd 往返。bpf_fault 能把它压到几分之一。不过
写热信号有更便宜的来源：KVM 脏页日志（Intel PML），不改内核。根治仍是让区域不被
THP 支撑。

**KVM 下的读写不可分辨问题不变。** 换入后客户机第一次读也会以写意图把页钉住，
之前因此放弃了"换入时写保护判干净"的 swap-cache 设计（见
`memtier-nvme-wear.md` §4.6）。bpf_fault 复用同一个 WP PTE 标记，内核看到的同样
是写缺页，换个处理器分辨不出读写。

## 4. 对换回的评估

### 4.1 现状分解

接口文档样本（消费级 TLC NVMe，测试机 10.8.1.196）：

| 阶段 | 时间 | bpf_fault 能否缩短 |
|---|---|---|
| vCPU 缺页陷入、uffd 入队、fault 线程接到（`swapin-avg-wait-ns`） | 约 9 µs | 能 |
| 提交读到 NVMe 完成（`swapin-avg-device-ns`） | 约 85 µs | 不能，设备延迟 |
| `UFFDIO_COPY` 拷 4K、装 PTE、唤醒 vCPU（`swapin-avg-publish-ns`） | 约 6 µs | 部分能：省 4K 拷贝，唤醒不可避免 |

memtier 已用自适应 busy-poll（`io-poll` / `busy-poll`）去掉了 fault 线程的唤醒，
论文里占 43% 的"调度加 IPI"在 memtier 上已经只剩 vCPU 这一侧的唤醒。

### 4.2 收益随设备延迟变化

uffd 的固定开销不随设备变，设备越快占比越大。理想路径假设：缺页在 vCPU 自己的
线程里处理，处理函数直接发块设备读并轮询完成，不睡眠不唤醒，数据直接 DMA 进内核
刚分配的页。剩下的只有缺页陷入、eBPF 调用和设备本身。

| 设备 4K 读延迟 | 现在 uffd 路径 | bpf_fault 理想路径 | 缩短 |
|---|---|---|---|
| 85 µs（消费级 TLC，现在测试机） | 约 97 µs | 约 88 µs | 一成 |
| 20 µs（企业盘 + SPDK 轮询） | 约 32 µs | 约 23 µs | 三成 |
| 10 µs（Optane、XL-Flash 一类） | 约 22 µs | 约 13 µs | 四成 |

以上为估算，固定开销取论文的 10 µs，未在快盘上实测。

### 4.3 拿不到收益的原因

1. **bpf_fault 不提供 I/O。** MISSING 处理函数只有"填好返回 0 安装"和"返回错误发
   SIGBUS"两个出口，没有"等用户态取回数据再安装"的模式；非睡眠型不能阻塞，睡眠型
   只能等有界的内核操作，不能等用户态事件。换回要读 NVMe，这一步只能留在用户态的
   io_uring / SPDK，所以 MISSING 必须继续走 uffd。
2. **不能部分切换。** VMA 不能同时注册 uffd 和 bpf_fault。MISSING 留在 uffd 上，WP
   与零填充就无法单独切到 bpf_fault，只能整体替换，而整体替换受 1 阻塞。
3. **要两处非主线内核改动。** 要在处理函数里读 NVMe，除 bpf_fault 补丁外还需一个
   内核 kfunc，在睡眠型处理函数里同步提交 bio 并等完成。论文没有，eBPF 社区目前只
   有发 UDP 包的提案，没有块设备读。两处都要自己维护在 7.0 内核上。
4. **主动换回无关。** 没有 vCPU 在等，吞吐由 NVMe 和 `max-swapin-mbps` 决定。

bpf_fault 真正解决的多 vCPU 并发缺页串行化问题（memtier 用 `fault-shards` 最多 8
线程缓解），同样以处理函数能自己完成换回为前提。

## 5. 现有内核上接近同等效果的办法

按收益排序：

1. **更快的盘加 SPDK。** 不依赖内核改动，先把 85 µs 的大头降下来；uffd 开销绝对值
   不变，但总延迟先减半以上。
2. **内核原生 swap 做参照。** 内核 swap 的换入就是在缺页线程上下文里直接发 I/O，
   路径形状与 bpf_fault 相同。在快盘上测一次内核 swap 的缺页延迟，即得"原地处理"
   在这台机器上的真实下限，以及 memtier 与它的差距里有多少是 uffd 造成的。此前的
   对比报告 `/root/memtier-vs-swap.md` 是慢盘数据。
3. **省 4K 拷贝。** 客户机内存改 memfd / shmem 后端，换入时通过第二个映射把数据
   直接读进页缓存页，再 `UFFDIO_CONTINUE` 装 PTE，去掉 `UFFDIO_COPY` 的拷贝。
   约 1 µs，快盘上才有感。
4. **预取。** 一次缺页后按空间局部性把邻页主动换回，把后续缺页换成不等待的主动
   换回，收益与设备延迟无关。

## 6. 值得借鉴的思路

把"写了没有"这类判定留在内核、用共享 bitmap 批量回收，比每页一次用户态往返便宜
一个数量级。若以后要降低换出窗口内的 vCPU 停顿，优先级是用 KVM 脏页日志替代
uffd-WP 作为换出前的脏判定，而不是等 bpf_fault 进主线。

## 7. 重新评估的触发条件

- bpf_fault 进入主线，或发行版内核携带；
- eBPF 获得块设备读能力（kfunc 或等价机制）；
- 生产后端盘换到 20 µs 级以下。

满足前两条时，memtier 的改动面集中在 `fault.c`（缺页路径与 worker、分配器分离），
换掉这一个文件即可；现在不需要为它预留接口。满足第三条时先做 §5.2 的内核 swap
对照，用实测代替 §4.2 的估算。
