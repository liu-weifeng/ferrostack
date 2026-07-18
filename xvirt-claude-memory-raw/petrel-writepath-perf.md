---
name: petrel-writepath-perf
description: "petrel 数据路径吞吐瓶颈 — 持久写只发挥 NVMe 缓存盘 ~6-11% 性能(OSD 写路径串行:per-object redb 事务+fsync,无批处理/流水线/并发扩展)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 3e93f53f-4faf-4927-8318-9562d4a000a4
---

**实测(2026-06-14,123+106 真盘,1MiB 顺序写 O_DIRECT+fsync 持久写)**:
- 裸 NVMe(120G 缓存盘 nvme1n1):seqW 691–973 MB/s,seqR 1.1–1.2 GB/s,4k 150–280 MB/s。裸 HDD seqW ~120 MB/s。
- petrel 卷 @ nvme tier **size=2**(跨双节点复制):seqW **43 MB/s**(≈裸 6%),seqR 84 MB/s。
- petrel 卷 @ nvme tier **size=1**(无复制):seqW **76 MB/s**(≈裸 11%)。
- size=1 **4 并发写**:71 MB/s 聚合(≈ 单写 76)——**并发不扩展**。
- petrel 卷 @ hdd tier size=2:seqW 35 MB/s(与 nvme 43 接近 → 盘速没体现)。

**结论:petrel 数据路径不发挥缓存盘性能**。持久吞吐 ~43-76 MB/s,与盘速(691+)几乎无关,nvme tier 仅略快于 hdd tier。复制约腰斩(76→43),但根因是**OSD 写路径串行**:每对象一个 redb 事务 + fsync,无批处理/组提交/流水线复制,**并发不扩展**(单 OSD 写串行)。网关 write-back 缓存只能吸收异步突发,**持续持久写**受路径限。

**对分层的影响**:tiering 正确把数据路由到 NVMe,但 petrel 写路径封顶吞吐 → 缓存盘速度被浪费。要"充分发挥缓存盘"必须先优化 petrel OSD 写路径:① 组提交(group commit,多 op 一次 fsync)② 流水线/异步复制(不每对象同步等远端)③ 批量对象写 ④ 跨 PG 并行提交 ⑤ 大 IO 直写绕 deferred。与既有 memory `[[memtier-bottleneck-analysis]]`(swap-in 1-page/sync/single-thread)同源——petrel 的同步单线程 IO 路径。

**优化进展(2026-06-14,petrel main 4322a54)**:
- 加 env 门控提交计时(`PETREL_COMMIT_TRACE`):实测 **batch 恒≈1**,每对象 apply~3.5ms + flush~5.6ms(消费级 NVMe fdatasync 慢,无 PLP)+ commit~0.5ms。group-commit 机制在(leader 排空队列成批),但并发 flush 的对象到 OSD 仍**逐个**落 → 每对象付一次 ~6ms fsync = 吞吐瓶颈。
- petrel-ublk serve 写回缓存窗口可调 `--cache-dirty-mb`(默认 16→256)/`--cache-total-mb`(默认 1024)。depth-1 顺序写 **76→103 MB/s(+35%)**。
- group-commit **成批窗口**(env 门控默认关):`PETREL_COMMIT_BATCH_WAIT_US`(默认0)+`PETREL_COMMIT_BATCH_TARGET`(默认32),leader 队列未达 target 等极短窗口攒 cohort。**实测仅微增(175→179 MB/s,4 网关;batch 仍多为 1)** —— 证明根因在上游:**窗口攒不到不存在的并发**。
- **✅ batch=1 根因已确认 = mux 单连接串行化**:并发 flush(join_all 64 对象,32 落 osd4)在网络上被**串行上线**。`mux` 每 OSD **只有一条连接**,writer 锁在发完整 4M payload 期间全程持有 → 同一 OSD 的 32 个并发对象逐个发送(每个 ~1ms 传 4M 回环),OSD 每 ~1ms 才收到一笔 < 单次 commit(~10ms)→ leader 每轮基本只看到 1 笔。即便开 2ms 成批窗口,1ms/笔的到达率也只能凑成 batch 2-15 偶发、仍以 1 为主。远端副本(跨节点)放大。**4 网关聚合 175-179 MB/s 也因此封顶**(并发未真正抵达 OSD)。
- **真正修法 = mux 每 OSD 多连接(连接池),让并发对象写在网络上并行抵达 OSD** → group-commit/cohort 窗口才有料可批。这是下一个高价值改动(在 `bins/petrel-osd/src/mux.rs`:MuxPool 每 addr N 连接轮询;影响读写两路 + 复制 fan-out,数据路径须谨慎+完整性校验)。次选:writer 锁不在持锁期间发完整大 payload(帧分片,但单 TCP 流仍无法真并行)。
- **下一步**:① 实现多连接 mux(高价值,但网络层重构,建议独立专注会话+每步完整性测);② 或先真 VM/fio 高 QD 测真实吞吐(dd depth-1 是最坏情形,真 VM 多队列 IO 可能天然带更多并发抵达 OSD,先量真实差距再决定是否做 mux 重构)。

**当前集群(已部署 tiered)**:123 meta=nvme2n1 + ssd-cache=nvme1n1 + hdd=sda-sdd;106 meta=nvme0n1 + ssd-cache=nvme1n1 + hdd=sdb-sde。10/10 OSD(hdd×8 + nvme×2 auto-detect),pool: testpool(1,默认全盘)/tier-nvme(2,size2)/tier-hdd(3)。**nvme 类 OSD = osd.4@123 + osd.9@106**(`petrel-mon usage` 看容量 128G vs HDD 500G 区分)。基线存 /tmp/nvme_baseline.txt(节点本地,易失)。

---
## 真 VM 集成测试(2026-06-14):分层放置 OK,但发现真 VM 写路径**严重挂起 bug**
经 xvirt API 建两台真 VM(vmnvme=critical / vmhdd=normal),boot 镜像 = 自建 Alpine+fio qcow2
(`/root/benchimg/bench.qcow2` @123,直接 `-kernel vmlinuz-virt -initrd initramfs-virt` 启动避开
syslinux;启动跑 `/etc/local.d/bench.start` fio 打 /dev/vdb 再 poweroff),数据盘 = petrel 卷经
**vhost-user-blk**(真 lictor 数据路径:`petrel-vhost serve` + `-object memory-backend-memfd,share=on`
`-device vhost-user-blk-pci`)。launcher `/root/petrel-vm-bench.sh @123`。

**✅ 分层放置端到端验证**(已修 ferrovirt 3 bug,见 commit 7663d95):critical→pool2 tier-nvme,
normal→pool3 tier-hdd;`petrel-mon usage` 证实数据物理落 nvme OSD(osd.4+osd.9,size=2),8 个 hdd OSD 0 字节。

**真 VM IO 实测(nvme tier,size=2,vhost-user-blk)**:
- 4k 随机读 qd32:**461 MiB/s / 118k IOPS**(缓存命中,极好)。
- 4k 随机写 qd32:**18 IOPS / 83 KiB/s**(灾难——每对象同步 fsync + size2 跨节点复制 + mux 串行,与盘速无关)。
- 1M 顺序写 qd1:**65 MB/s**(≈裸 NVMe 691 的 ~10%,与 dd 一致;低深度才不挂)。
- 缓存内 qd16 写(200MB<缓存):45 MB/s,完成。

**🔴 严重 bug:持续写超过网关缓存总量 → guest IO 永久挂起**(非性能,是可用性/正确性)。
- 复现确定:200MB qd16 写**完成**;1500MB qd16 写 / 2G qd16 顺序写 / 大块顺序读**永久挂死**(fio 卡 D 态,
  连自己的 time_based 计时都无法结束 = 某 IO 永不返回)。低深度(qd1)2G 写不挂(流入≈排空,缓存不积压)。
- **线程转储定性(决定性证据)**:挂起时 petrel-vhost **完全空闲**——所有 tokio-rt-worker 在 futex(park)、
  唯一 vring_worker 卡 `ep_poll`(等 guest kick)、控制线程等 unix socket。即后端没卡在锁/计算/网络,
  而是 **vring 通知 missed-kick / lost-wakeup**:guest 加了描述符+(EVENT_IDX 下)抑制了 kick,后端却
  回睡没重扫 avail ring → guest 等完成、后端等 kick,死锁。
- 代码位置 `bins/petrel-vhost/src/lib.rs` `handle_event`(348-364 有 disable/process/enable+重扫循环,
  形似正确但仍漏)+ `process_queue`(190,**同步 block_on 处理,大写时把通知禁用窗口拉到数秒**,极大放大竞态)。
  网络 RPC 有 15s 超时(client_data.rs:225)故**非网络挂起**;ublk 用内核 ublk 通知机制,**不踩此 bug**。
- **修复方向(未做,需独立专注 + 完整性验证,勿仓促进活 IO 路径)**:① 修 vring 通知 missed-kick(可能 enable_notification
  返回值语义/索引竞态,或换处理模型);② 把同步 block_on 处理改异步/缩短通知禁用窗口;③ 加 fio/真 VM 持续写回归测。
- 已修可提交项:petrel-vhost 缓存可调(commit ef0258c,16MB→256MB,缓解但**不修**挂起,因任何 >缓存的持续写都触发)。

**结论(回答"是否充分发挥缓存盘性能")**:否。读优秀(缓存命中 461MB/s),但写:顺序仅 ~10% 盘速、
4k 随机仅 18 IOPS(≈盘小 IO 能力的 0.03%),且**持续大写会挂死 guest**。真 VM 写负载(装系统/大文件/DB)
当前不可用,须先修 vring missed-kick 挂起 + 写路径性能(mux 多连接 / group-commit)。

---
## vhost 挂起修复尝试(2026-06-14):部分修复,**非确定性残留,未提交**
深挖 petrel-vhost(`bins/petrel-vhost/src/lib.rs`)持续 qd16 写挂死,逐层定位(env 门控 trace
`XVIRT_VHOST_TRACE` + 线程转储 + osd-bytes 时序):
- **改 1(真 bug,已写未提交):去掉 `VIRTIO_RING_F_EVENT_IDX`**。EVENT_IDX 下后端同步处理
  (handle_chain 里 block_on 阻塞数秒)把 disable→process→enable 通知窗口拉长,与 guest kick 抑制
  竞态 → 丢 kick → 后端永久 idle 在 ep_poll(线程时序证实 t+42s 起永久 idle)。去掉后 guest 每批必 kick、
  eventfd 不丢唤醒。**单次 1500MB qd16 写从"必挂"变成"偶尔过"(49 MB/s)**。
- **改 2(正确健壮性,已写未提交):process_queue 每条链必 add_used**。原 `iter().collect()` 取走全部
  avail 链后,中途 `?` 提前返回会丢掉已取出链 → guest 永等。改为出错也以 0 长度回收 + 不中断批 + 必 signal。
- **试过但无效/已回滚**:① used-ring 心跳(每 3ms 重发 callfd + 重扫 avail)——证明残留挂起**既非丢 used
  中断也非丢 kick**(心跳重发/重扫都救不回,guest 进 D 态不再下发);② else 分支 drain-loop
  `while process_queue {}`——反而更易挂(已回滚)。
- **关键证据(残留挂起)**:flush() 完全健康(后台 flusher 每 50ms 跑,snapshot 空=无脏=写根本没到
  cache.write_object);HC start==done(后端把收到的都处理完了);osd bytes 冻结;guest D 态不再下发。
  ⇒ 残留是**更深的非确定性竞态**(virtio-queue/vhost-user-backend 框架交互 或 mux 单连接响应解复用 或
  block_on×多线程运行时),改 1/2 只**降低概率未消除**。
- **可靠性实测(改1+改2,单 1500MB qd16 写 ×3):1 过 2 挂**。⇒ **未达可提交标准,改动留在 123 working tree
  未 commit/push**(123/106 上**已部署**该 petrel-vhost 二进制——比基线好[基线 EVENT_IDX 必挂+pre-multipool
  线协议不兼容],但仍偶发挂)。**真正的修法 = vhost 数据面异步化重写**:不要让单 vring_worker 每个 IO
  block_on 串行阻塞,改为每请求 spawn 异步任务、完成时 add_used+signal(SPDK/cloud-hypervisor 式)——
  同时解决挂起(worker 不被阻塞)+ 性能(真并发)。是独立专注大改 + 完整性/可靠性回归。
- 复现/诊断工装在 123:`/root/petrel-vm-bench.sh`(serve vhost + 直 kernel 启 `/root/benchimg/bench.qcow2`
  跑 fio 打 /dev/vdb)、`/root/benchimg/`(boot 镜像 + vmlinuz/initramfs)。trace 用 `XVIRT_VHOST_TRACE=1`。

### vhost 数据面**异步化重写**(2026-06-14):大幅改善但仍残留 ~20-30% 偶发 freeze,**未提交**
按"不要让单 vring worker 每个 IO block_on 阻塞"重写 petrel-vhost 数据面(working tree 未 commit):
- **改法**:`AsyncBlock` trait(async read/write/flush)+ `impl for Volume<A>`;`VhostBlkBackend` 持
  `Arc<dyn AsyncBlock>` + `tokio::runtime::Handle`;`process_queue` 只**解析**每条链成 `ParsedReq`
  (拥有 write_data + WO 段 (gpa,len),不借用 guest 内存/链,可 Send)然后 **spawn 异步任务**立即返回;
  任务 `.await` 后端 IO → `build_response` 拼回设备可写区 → `write_slice` 散落(经 BitmapMmapRegion 标脏)
  → `add_used` + `signal_used_queue`。不开 EVENT_IDX。main.rs 两路都 `Arc::new(volume) as Arc<dyn AsyncBlock>`。
  单测 5 个过(build_response 布局/短读/写/错误 + config)。
- **效果**:可靠性 **sync 1/3 → async ~7-8/10**;**overwrite(W2)彻底修好**(原来必挂,现 W1+W2+R1+randrw
  全跑通);真异步=真并发。
- **残留**:~20-30% 偶发整机 freeze(num-queues=1 8/10、=4 7/10,**与队列数无关**)。**关键证据:挂起时
  `spawned==done`(T+/T- 计数,后端把收到的每条请求都 add_used+signal 了)、无 panic、无 kernel oops、
  guest 静默停在 fio 不再下发**。即**后端可证无错**;丢 kick(重扫 avail 无效)/丢 used 中断(重发 callfd 无效)
  /丢描述符(恒 add_used)/卡任务(spawned==done)/多队列(=1 仍挂)全部排除。⇒ 残留在 **QEMU vhost-user
  中断投递 / guest virtio_blk / 跨 tokio-线程 add_used 的内存可见性**,或**极简测试 guest(手搓 Alpine + 直 kernel
  boot)自身的脆性**——需 gdb(guest)/QEMU trace 级工具,本环境够不到。center VM(真发行版)跑 petrel-vhost
  正常,佐证可能是测试 guest артефакт。
- **坑(测试可靠性)**:连续 loop 测时,上一轮被 timeout 杀的 qemu 偶尔残留持有 `boot-<label>.qcow2` 写锁
  → 下一轮 `Failed to get "write" lock` 假"挂起"。**每轮用唯一 label(唯一 boot 镜像/socket)+ 轮间
  `while pgrep -f qemu...; do sleep 1; done` 等彻底退出**才得干净统计。
- **已 commit + push**(petrel `32e7c0e`,origin/main):异步重写。123/106 部署该干净二进制。

### 真发行版 VM 测试(2026-06-14)——**结论翻转:残留 freeze 是真 bug,非测试 guest артефакт**
按"区分测试 guest 脆性 vs 真 bug"做了 Ubuntu 24.04 cloud image(kernel 6.8,UEFI 启动,完整 systemd)
真发行版 VM 测试(工装:`/root/ubuntu-bench/golden.qcow2` = chroot apt 装 fio + bench.service auto-run;
launcher `/root/petrel-ubuntu-bench.sh`,OVMF `edk2-x86_64-code.fd` UEFI 启动,petrel vol 经 vhost-user-blk):
- **Ubuntu 100% 挂(0/8+0/6+0/6+0/4+0/4 多批全挂)**,vs Alpine ~75% 过。**同一后端、同一 fio 命令、同一卷**
  → **残留 freeze 是真 bug,Ubuntu 6.8 virtio_blk 必现**,非极简 guest артефакт。早先"多半是测试 guest"判断**推翻**。
- **逐一排除(Ubuntu 确定性复现)**:① 并发——`XVIRT_VHOST_INFLIGHT=1`(后端串行)仍挂;② 多队列——device
  `--num-queues 1` 仍挂;③ 丢 used 完成中断——纯重发 callfd 心跳(每 10ms signal_used_queue,不碰队列)仍挂;
  ④ end_fsync/FLUSH——去掉 `--end_fsync` 与 Alpine 完全一致仍挂;⑤ 后端正确性——trace 证 `spawned==io_done==
  completed==1746`(≈整轮 1500MB 写,每条都 add_used+signal 了)。
- **症状定性(决定性)**:挂起时 **guest 内核健康**(无 hung-task watchdog、无 oops,getty/各 service 照常起到登录),
  **只有 fio 卡在 io_getevents** 等后端其实已投递的完成。⇒ 是 **vhost-user-blk ↔ guest virtio_blk 完成投递握手**
  的不兼容,**与 guest 内核版本相关**(6.8 必现 / 6.12 偶现),且**异步后端从 tokio 线程做 add_used+signal**
  这一结构差异是最大嫌疑(sync 版在 vring-worker 线程做完成,但 sync 版有别的 Alpine bug,未在 Ubuntu 对照)。
- **结论**:**petrel-vhost 对真 VM 持续写负载尚不可用**(async 比 sync 改善,但 Ubuntu 必挂)。`32e7c0e` 仍是
  相对改善(overwrite 修好、架构正确、Alpine 75%、不再首写必挂),保留在 main,但**生产写重 VM 不可依赖**。
- **真正修法(需独立深挖,要 guest-gdb / QEMU `-trace vhost_*/virtio_*`)**:很可能需把 **add_used+signal 放回
  vhost 框架的 vring-worker 线程**(异步 IO 完成经 channel + 完成 eventfd 回送 worker 处理,即 SPDK/cloud-hypervisor
  的标准异步 vhost 模型;vhost-user-backend 0.22 无现成 hook,需自建完成 fd 或换模型),或定位 used-ring 字节/
  内存序与 6.8 virtio_blk 的具体不符点。工装全留在 123(`/root/ubuntu-bench/`、`/root/petrel-ubuntu-bench.sh`,
  trace `XVIRT_VHOST_TRACE=1`,确定性复现便于后续 gdb)。
