# Fulmar：面向虚拟化的用户态共享磁盘文件系统设计方案（替代 OCFS2）

> 生成时间：2026-11-09
> 状态：设计稿 v1.1，未动代码。v1.1 修订（2026-11-09）：块设备后端改为**内核裸盘 io_uring 为默认**，SPDK 降为可选加速档；memtier 直通改以块设备 fd 传递为主（§4.3、§8.3、§十六）
> 适用范围：`hyper-agent`（datastore 后端 / 控制面）、`qemu`（块驱动 + memtier 后端）、`xvirt-blockdiff` / `xvirt-snapmig` / `cdp`（快照链与差异）、`petrel`（可共享 io_uring 与 vhost-user-blk 代码）
> 关联文档：`docs/qemu-memtier-interface.md`、`docs/memtier-swapcache-design.md`、`docs/drive-mirror-sparse-thick-design.md`、`docs/resource-planning.md`、`xvirt-blockdiff/README.md`
> 名称说明：Fulmar（暴风鹱，与 petrel 同科）。daemon `fulmard`、客户端库 `libfulmar`、QEMU 驱动与 URL 前缀 `fulmar:`、`mkfs.fulmar`、datastore `fs_type=fulmar`。

---

## 目录

1. [背景、目标与非目标](#一背景目标与非目标)
2. [OCFS2 的问题根因与 Fulmar 的对应解法](#二ocfs2-的问题根因与-fulmar-的对应解法)
3. [总体架构](#三总体架构)
4. [数据路径：libfulmar、共享内存环与块设备后端](#四数据路径libfulmar共享内存环与块设备后端)
5. [盘上格式与元数据](#五盘上格式与元数据)
6. [COW / reflink / 快照：O(1) reflink 与无读放大的 ROW](#六cow--reflink--快照o1-reflink-与无读放大的-row)
7. [动态块大小与碎片治理](#七动态块大小与碎片治理)
8. [与 QEMU 内存分层（memtier）的配合](#八与-qemu-内存分层memtier的配合)
9. [集群模式：成员、心跳、ATS 锁、锁管理与 fence](#九集群模式成员心跳ats-锁锁管理与-fence)
10. [按磁盘限速（QoS）](#十按磁盘限速qos)
11. [一致性、持久化与崩溃恢复](#十一一致性持久化与崩溃恢复)
12. [接口：控制面、数据面、POSIX 网关与工具](#十二接口控制面数据面posix-网关与工具)
13. [与现有组件的衔接与迁移路径](#十三与现有组件的衔接与迁移路径)
14. [其他需要关注的点（清单）](#十四其他需要关注的点清单)
15. [性能目标与验证方法](#十五性能目标与验证方法)
16. [分阶段实施计划与风险](#十六分阶段实施计划与风险)
17. [待拍板事项](#十七待拍板事项)

---

## 一、背景、目标与非目标

### 1.1 现状

- 共享存储 datastore 目前用 **OCFS2（内核态）**：`hyper-agent/support/datastore/local.py` 以 o2cb 单节点集群 `ocfs2sn` 格式化、挂载；VM 磁盘是 raw 文件，磁盘快照靠 **reflink**（`cp --reflink`），快照差异靠 **FIEMAP** 比对两文件的物理映射（`xvirt-blockdiff`），petrel 侧 `petrel-snapdiff` 产同格式 `.xvdiff`。
- OCFS2 内核模块需要**按内核版本自编译打包**（`ocfs2/README.md`，Rocky 10.1 不带 `ocfs2.ko`），每次内核升级都是一次风险。
- QEMU 内存分层（memtier）的换出后端已支持裸块设备 / 预分配 raw 文件 / `spdk:` NVMe 命名空间，`O_DIRECT + io_uring`，4K slot、2MB chunk；**与 vhost-user 共享内存互斥**（memtier 强制 4K 匿名页，`docs/resource-planning.md` §表）。
- petrel 已有 rust-vmm 纯 Rust 的 vhost-user-blk 出口；资源规划已把 SPDK 的核与大页圈成独立池（`spdk.slice`、`hugepages.spdk_quota`）。

### 1.2 目标

| # | 目标 | 度量 |
|---|---|---|
| G1 | **用户态**实现，不依赖内核模块；内核升级零改动 | 只需 io_uring / O_DIRECT 块设备 / userfaultfd 等稳定内核接口（SPDK 档另需 vfio） |
| G2 | **本地模式**与**集群模式**同一套盘上格式与代码，可在线从本地切换到集群 | 单节点部署无心跳 / 无锁开销 |
| G3 | **COW + reflink** 做磁盘快照；reflink 成本与文件碎片程度**无关** | 1TB、百万 extent 的盘 reflink ≤ 10ms |
| G4 | **快照后首写无读放大**：不先拷旧块再写 | 快照后 4K 随机写延迟 ≈ 裸盘 4K 写 + ≤10µs |
| G5 | **低延迟高带宽**：数据路径不经内核文件系统与页缓存，io_uring 直写裸块设备；SPDK 为可选加速 | 4K 随机读相对裸盘附加延迟 ≤ 8µs（io_uring 轮询）/ ≤ 5µs（SPDK 档）；大块顺序吞吐 ≥ 95% 裸盘 |
| G6 | **动态块大小**：寻址粒度固定 4K，分配 / COW 粒度按介质与负载自适应 | 见 §七 |
| G7 | **memtier 换入低延迟**：换出后端文件放在 Fulmar 上，换入路径不经 daemon 或只经一次忙轮询环 | 换入 4K 读延迟 ≈ 裸盘 |
| G8 | 集群：**网络 + 磁盘双通道**、**ATS（Compare-and-Write）盘上锁**、**快速 fence**（停掉故障侧数据路径，而非 panic 整机） | 故障检测 ≤ 7s，接管 ≤ 10s |
| G9 | **按磁盘限速**（IOPS / 带宽 / 突发，读写分开），在线调整 | 控制 API 一次调用生效 |
| G10 | 快照链工具链平滑：差异提取、跨 datastore 迁移、CDP 继续可用，且比 FIEMAP 全扫更快 | `fulmar diff` O(变化量) |

### 1.3 非目标（本期不做）

- 通用 POSIX 文件系统（硬链接、权限模型、mmap、xattr 泛化、海量小文件）。Fulmar 的对象是**少量大文件**（VM 磁盘、memtier 后端、内存快照、ISO）。
- 数据压缩、去重、加密（可留格式位）。
- 多 LUN 拼接（spanning）与分层介质池（先单设备 + 在线扩容；格式预留）。
- 替代 petrel：petrel 是副本 / EC 的分布式块存储，Fulmar 是**共享磁盘（SAN / 多路 NVMe-oF / 本地盘）之上的文件系统**，两者并存，由 datastore 类型选择。

---

## 二、OCFS2 的问题根因与 Fulmar 的对应解法

| 问题 | OCFS2 根因 | Fulmar 解法 | 详见 |
|---|---|---|---|
| **碎片化后 reflink 变慢** | reflink 要**遍历源文件整棵 extent 树并逐 extent 复制 + 逐 extent 更新 refcount 树**，复杂度 O(extent 数)，每步都进 journal；碎片越多越慢 | reflink = 新 inode 指向**同一棵 extent B+ 树的根**并给根加引用，O(1)；数据 extent 的引用计数**延迟到该叶子第一次被 COW 时成批补**（btrfs 式惰性传播） | §6.2 |
| **快照后写放大** | COW 以 **cluster**（通常 1MB）为单位：4K 写要先读 1MB 旧数据、写 1MB 新位置（RMW），再更新 refcount / extent 树 | **ROW（Redirect-on-Write）**：只为被写的 4K 对齐范围分配新 extent，不读不拷旧数据；旧数据仍归快照引用。粒度按策略自适应（§七） | §6.3 |
| COW 热路径要查 refcount 树 | 每次写都要判断 extent 是否共享 → refcount 树查找 | extent 记录带 **birth generation**，`birth > inode.last_snapshot_gen` 即可确定未共享（ZFS 做法），热路径零元数据查找 | §6.3 |
| 内核路径长、延迟抖动 | VFS → OCFS2 → DLM → 块层 → 驱动；cluster 锁在内核 | 用户态、无页缓存、io_uring 直达裸块设备（可选 SPDK）；锁被**所有权分区**消掉，热路径无锁 | §三、§九 |
| 内核模块与版本绑死 | 发行版不带模块，每次内核升级重编 | 纯用户态 | — |
| 故障处理 = 整机 panic / reboot | o2cb 心跳超时触发 panic | **自 fence 停掉数据路径**（QEMU 收 EIO 暂停），幸存者用 **PR 抢占**兜底，不重启主机 | §9.5 |
| 心跳慢（默认 31×2s≈62s） | 盘心跳固定保守 | 盘 + 网双心跳，默认 1s 间隔、6 次判死（≈7s） | §9.2 |
| 零区 / 厚盘语义别扭（`ZERO_RANGE` 不支持，unwritten 报 hole） | 内核 fallocate 语义限制 | 块状态 API 直接给 QEMU（allocated / zero / unwritten / shared），`write_zeroes` 是元数据操作 | §12.2、`drive-mirror-sparse-thick-design.md` |
| 快照差异要 FIEMAP 全盘扫 | 没有树级差异接口 | 两快照共享子树指针相同 → 跳过，`fulmar diff` O(变化量) | §12.4 |

---

## 三、总体架构

```
┌─────────────────────────── 宿主机 N ───────────────────────────┐
│  hyper-agent ──控制 API(unix socket, JSON)──► fulmard (每主机一个) │
│                                                │               │
│  QEMU(VM a) ─libfulmar─┐  共享内存环(SQ/CQ, hugepage)│  ┌─────────┐ │
│   ├ block/fulmar.c     ├───────────────────────────┼─►│ IO 线程 │ │
│   └ memtier fulmar 后端┘  (或 直通租约: 见 §8.3)     │  │ (轮询)  │ │
│                                                │  └────┬────┘ │
│  QEMU(VM b) ─vhost-user-blk(零拷贝, 无 memtier)──┤       │      │
│                                                │  块后端抽象   │
│  qemu-img / cp / 备份 ─ FUSE 网关(管理用途) ─────┤  io_uring 裸盘(默认) | SPDK(可选)
│                                                └───────┼───────┘
└────────────────────────────────────────────────────────┼─────────┘
                       共享 LUN / NVMe-oF ns / 本地 NVMe  │
             ┌───────────────────────────────────────────▼──────────┐
             │ 超级块 │ 集群区(心跳槽/ATS 锁记录/成员表) │ 节点 journal×N │
             │ AG0 │ AG1 │ … │ AGk  (每 AG: 空闲树 + refcount 树 + 数据)  │
             └──────────────────────────────────────────────────────┘
   集群网络：成员 / 租约 / 轻量 DLM / 延迟释放 RPC（TCP，RDMA 可选）
```

### 3.1 组件

| 组件 | 职责 | 语言 / 依赖 |
|---|---|---|
| `fulmard` | 每主机一个常驻进程：挂载 datastore、以 io_uring 打开裸块设备（可选 SPDK）、运行 IO 线程轮询数据环、元数据与 journal、集群成员 / 心跳 / 锁、QoS、FUSE 网关 | Rust 或 C；见 §17 |
| `libfulmar` | 客户端库：打开文件、异步读写 / flush / discard / write_zeroes / block_status、extent 租约（memtier 直通） | C ABI，QEMU 直接链接 |
| QEMU `block/fulmar.c` | 块驱动，协议 `fulmar:<datastore>/<path>`；接 `bdrv_co_preadv/pwritev/flush/pdiscard/pwrite_zeroes/block_status` | QEMU fork |
| QEMU `memtier/backing-fulmar.c` | memtier 换出后端，`path: "fulmar:…"` | QEMU fork |
| `fulmar` CLI | mkfs / mount / snap / reflink / diff / fiemap / stat / qos / cluster | 走控制 API |
| `fulmar-fsck` / `fulmar-scrub` | 离线修复、在线校验 | 独立 |
| FUSE 网关 | 把 datastore 以只读或低速读写 POSIX 目录暴露，供 qemu-img、浏览、上传 | libfuse3 passthrough |

### 3.2 关键设计取向

1. **一台主机一个 FS 实例**（`fulmard`），所有客户端都附着到它；集群中每个节点就是 OCFS2 意义上的一个"挂载"。
2. **所有权分区**：每个文件（VM 盘）任一时刻只属于一个节点（运行该 VM 的节点）；空间分配以 **AG（allocation group）为单位租给节点**。于是文件 extent 树是**单写者**，热路径没有分布式锁，锁只剩慢路径（命名空间、AG 租约、跨节点释放）。
3. **快照不可变**：快照 inode 只读，可被任意节点同时打开读（备份、跨存储迁移、CDP 回放），不需要任何锁。
4. **数据绕过内核文件系统**：数据面只经过内核块层（io_uring + O_DIRECT 直写裸块设备），没有页缓存、没有 VFS；需要再省几微秒时可换 SPDK 用户态驱动（vfio），上层代码不变。
5. **格式即协议**：本地模式与集群模式同格式，集群区始终存在，本地模式只是不启用心跳与锁。

---

## 四、数据路径：libfulmar、共享内存环与块设备后端

### 4.1 三种附着方式

| 方式 | 适用 | 数据拷贝 | 与 memtier 兼容 | 相对裸盘附加延迟（目标） |
|---|---|---|---|---|
| **A. 共享内存环（默认）** | 所有 VM 盘 | 1 次 memcpy（libfulmar ↔ 环内 bounce 缓冲，hugepage） | **兼容**（客户机内存保持匿名页） | 环本身：忙轮询 ≤ 5µs，eventfd 唤醒 ≤ 15µs；后端开销另计（§4.3） |
| **B. vhost-user-blk 零拷贝** | 不开 memtier、追求极限吞吐的 VM | 0 | 不兼容（需 `memory-backend-memfd,share=on`） | ≤ 3µs |
| **C. extent 租约直通** | memtier 换出后端；（二期）VM 盘只读热路径 | 0（客户端对块设备 fd 自己发 io_uring） | 兼容 | ≈ 0（与现有裸分区后端同路径） |

A 是基线：一次 4K memcpy < 1µs，1MB 约 50–100µs 但大块 IO 本身耗时更长且可并行，吞吐仍能到介质上限；换取的是**客户机内存不必共享**，这正是 memtier 的前提。B 复用 petrel 已验证的 rust-vmm vhost-user-blk 栈（后端选 SPDK 时也可用 SPDK vhost-blk target），把后端换成 Fulmar 文件即可。C 见 §8.3。

### 4.2 共享内存环

- 每个客户端进程连接 `fulmard` 时协商一块 hugepage 共享内存：**每 vCPU 线程或每 QEMU iothread 一对 SQ/CQ**（无锁 SPSC），加一片 bounce 缓冲池（按 4K / 64K / 1M 三档 slab）。
- 请求：`{op, fh, offset, len, iov(环内偏移), flags(FUA/…), tag}`；完成：`{tag, status, bytes}`。
- 通知：默认 `fulmard` reactor 忙轮询环；客户端侧忙轮询可选（`poll-us` 自适应：先轮询 N µs 再退到 eventfd），和 memtier 的 `busy-poll` / `io-poll` 旋钮口径一致。
- NUMA：环与 bounce 缓冲分配在客户端所在 NUMA 节点；`fulmard` 每 NUMA 节点至少一个 IO 线程；轮询模式下独占核，从资源规划的 `spdk` 池划拨（建议改称存储池），eventfd 模式不独占核。
- 安全：unix socket 传 memfd，`fulmard` 校验每个 iov 落在该客户端自己的环区间内。

### 4.3 块设备后端：内核裸盘为默认，SPDK 为可选

`fulmard` 经一层薄的块后端抽象（提交 / 完成、flush、discard、write_zeroes、CAW、PR、能力探测）访问整块 LUN 或分区。两种实现，盘上格式与上层代码完全相同，可按主机切换：

| 后端 | 打开方式 | 适用 |
|---|---|---|
| **`uring`（默认）** | `O_RDWR\|O_DIRECT` 打开 `/dev/nvmeXnY`、`/dev/mapper/mpathX`、`/dev/sdX`；io_uring 固定文件 + 固定缓冲（环的 bounce 区直接注册）；可选 SQPOLL、IOPOLL（需 NVMe poll 队列） | 本地 NVMe、FC / iSCSI SAN、NVMe-oF（内核 initiator）、开发机；**所有阶段的基线** |
| **`spdk`（可选）** | vfio 绑定控制器，SPDK nvme / nvmf bdev，reactor 忙轮询 | 低延迟介质（Optane / SLC）、NVMe-oF 且主机已为存储预留独占核、需要 SPDK vhost-blk target 的零拷贝档 |

选内核裸盘为默认的依据：

| 维度 | 内核裸盘（io_uring） | SPDK |
|---|---|---|
| 单次 IO 软件开销 | 5–10µs；IOPOLL + SQPOLL + 固定缓冲可到 3–5µs | 1–3µs |
| 单核 IOPS 上限 | 100–200 万 | 500 万以上 |
| 对 TLC NVMe 4K 读（80–100µs）的影响 | 附加 < 10% | 附加 2–3% |
| 对低延迟介质 / NVMe-oF（10–20µs） | 附加 30–50% | 附加约 10% |
| 带宽 | 几个核即可打满 PCIe 4/5 单盘 | 同 |
| CPU | 按需；可中断可轮询 | 独占核 100% 忙等 |
| 盘的共享 | 同一块 NVMe 可分区分别给 Fulmar、XFS、memtier 裸分区 | 整个控制器被 vfio 独占 |
| ATS / PR | NVMe passthrough / `SG_IO` 发 CAW；PR 用块层统一 ioctl，SCSI 与 NVMe 一套接口 | API 直接；NVMe-oF 目标端支持参差 |
| 多路径 | dm-multipath 成熟 | SPDK multipath 较新 |
| FC SAN | 支持 | 不支持 |
| 运维与排障 | blktrace / iostat 现成；进程崩溃不影响他人 | 大页、vfio、多进程共享内存；二级进程崩溃可能拖累主进程 |

结论：当前主力介质是 TLC NVMe 与 SAN，SPDK 的延迟收益是个位数百分比，代价却是虚拟化主机最稀缺的独占核与整块控制器；集群所需的 ATS、PR、多路径、FC 在内核路径上反而更省事。因此 **P0–P3 全部以 `uring` 后端交付**，`spdk` 后端在 P2 实测评估后按主机开关启用（§十六）。ATS / PR 能力按后端探测（§9.3）。

### 4.4 IO 大小、队列与并发

- 单请求上限 1MB（与 QEMU `max_transfer` 对齐），大 IO 在 `fulmard` 内按 extent 边界切分并发下发。
- 读写 extent 查找走每文件的**叶子缓存**（§5.4），命中时零元数据 IO。
- 写路径见 §6.3；FLUSH 见 §十一。

---

## 五、盘上格式与元数据

### 5.1 布局

```
偏移      区域                       说明
0         超级块（主）                 4K；副本在设备中部与末尾
4K        集群区                      成员表、节点 epoch、心跳槽 ×64（每槽 4K）、
                                     ATS 锁记录 ×M（每条 512B/4K，按设备原子写粒度）
+16MB     节点 journal ×64            每槽默认 512MB，可 mkfs 指定；intent log + 检查点指针
+…        AG 表                       每 AG：owner、租约 epoch、空闲 / 已用计数
AG 0..k   每 AG = 头 + 空闲 extent 树(按偏移/按长度两索引) + refcount 树 + 数据块
全局树    inode 树、目录树（根指针在超级块，COW 更新，命名空间锁保护）
```

- **寻址单位 4K**（匹配 4Kn NVMe、memtier 4K slot、客户机页）；块号 48 位；extent 最长 2^32 块但分配上限 1GB。
- **AG 大小**默认 32–64GB（按设备容量取 256–1024 个 AG），足够每个节点同时租若干个做并行分配。
- 元数据树节点大小 mkfs 可选 4K–64K，默认 **16K**（一叶约 600 条 extent 记录；1TB、平均 256K extent 的盘约 7000 叶、3 层）。
- 所有元数据块带 **校验和 + fsid + 位置 + 代号**，检测误写与陈旧块。

### 5.2 inode 与 extent 记录

```
inode(512B):  ino, type(disk|pinned|snapshot|regular), size, flags,
              extent 树根 (blk, level, gen), last_snapshot_gen,
              class 策略: extsize(分配提示), cow_chunk(自动/定值), thick(保留配额), qos,
              owner 节点/租约 epoch, 时间戳, 父快照 ino(链路可视化用), 内联小 extent 列表(≤16 条)
extent 记录(24B): logical(u64), len(u32), phys(u48), flags(u16: UNWRITTEN|ZERO|SHARED_HINT), birth_gen(u32)
```

`birth_gen` 是关键：全局单调的**快照代号**（每次对该文件 reflink / 快照时 `last_snapshot_gen` 增加），写时比较即可判定共享，热路径不碰 refcount 树（§6.3）。

### 5.3 refcount 树（每 AG 一棵）

键为物理 extent 范围，值为引用数。只有**共享过**的范围才有记录（无记录 = 引用 1），这与 XFS / OCFS2 同；差别在**何时更新**（§6.2）。范围会因不同部分的引用数分化而分裂，后台合并相邻等值记录。

### 5.4 内存中的元数据

- 树节点以 16K 块为单位做 LRU 缓存（类似页缓存但只缓存元数据）；配额按 datastore 规模配置（默认 256MB–2GB）。
- 每打开文件一棵"热叶索引"：逻辑区间 → 已缓存叶子，常驻工作集（一块 1TB 盘几 MB 级）。
- 不缓存数据。

---

## 六、COW / reflink / 快照：O(1) reflink 与无读放大的 ROW

### 6.1 模型

- 每个文件的 extent 树是 **COW B+ 树**；树节点与数据 extent 都有引用计数（节点引用只有共享时才写入 refcount 树，惰性）。
- **快照 = reflink**：对 VM 盘 `D` 做快照即 `reflink(D) → S_n`，`S_n` 标记为 `snapshot` 不可变，`D` 继续是活盘（与现有 `cp --reflink` 流程语义一致，`SNAPSHOT_REFLINK_DESIGN.md` 不变）。
- 任意两文件（同节点或跨节点）可 reflink，只要源在当前节点可见且源处于一致点（活盘需 owner 节点执行；快照任意节点可执行）。

### 6.2 reflink O(1)

```
reflink(src → dst):
  1. 命名空间锁下分配 dst inode（一次 inode 树 COW）
  2. dst.root = src.root；src.root 节点引用 +1（refcount 树一条记录）
  3. src.last_snapshot_gen = dst.last_snapshot_gen = ++global_gen
  4. 一条 journal 事务提交
```

成本与源文件 extent 数量、碎片程度**无关**。引用的惰性传播规则：

- **COW 一个共享的内部节点**时：把它的所有孩子引用 +1，旧节点引用 -1（孩子只有十几到几百个）。
- **COW 一个共享的叶子**时：叶内所有数据 extent 引用 +1（按 AG 分组批量写各 AG 的 refcount 树，一个事务），旧叶引用 -1。
- 以上发生在写路径的慢分支（§6.3 第 2 步），一叶最多发生一次，之后该叶归活盘独有。

> 与 OCFS2 对比：OCFS2 把"所有 extent 引用 +1"放在 reflink 时刻一次做完（O(N) 且阻塞快照），Fulmar 把它摊到后续第一次写入每片叶子时，且被写得少的区域永远不付这笔账。

### 6.3 写路径（ROW）

```
write(D, off, len):
  1. 查叶缓存得覆盖 [off, off+len) 的 extent 记录
     若全部 birth_gen > D.last_snapshot_gen 且非 UNWRITTEN/ZERO → 原地写，结束（零元数据）
  2. 否则（共享或从未分配）：
     a. 若所在叶子共享（叶 ref>1 或祖先共享）→ 先 COW 路径并做 6.2 的惰性引用传播
     b. 以 cow_chunk 对齐确定 ROW 范围 R ⊇ [off,off+len)（§7.2；默认即 [off,off+len) 的 4K 对齐）
     c. 在本节点租用的 AG 中分配新 extent（顺序游标，见 §7.3）
     d. 若 R 比请求大（cow_chunk>4K 策略）：读旧数据补齐 R 两端 —— 这是**唯一**可能出现的读放大，且由策略上限约束
     e. 写数据到新 extent
     f. 更新叶子：拆分旧记录、插入新记录（birth_gen=当前 gen）；旧物理范围 ref-1 进"延迟释放"队列
     g. 写 journal 记录（组提交）
  3. 完成回调
```

- 旧数据**不读不拷**（默认策略），所以"快照后写"的开销 = 一次正常写 + 批量 journal。
- 延迟释放队列（btrfs delayed refs 思路）持久化在 journal 中，后台线程批量执行 refcount 树更新与真正释放，崩溃可重放。
- 快照删除：根引用 -1；若归零则遍历，只进入引用归零的子树，叶内数据 extent 引用 -1；全部后台、可中断续作。

### 6.4 快照链操作对照

| 操作 | 现 OCFS2 流程 | Fulmar |
|---|---|---|
| 建快照 | `cp --reflink D S` | `fulmar snap D S`（O(1)） |
| 删中间快照 | `rm S_k` | `fulmar rm S_k`，后台回收，链路对上层仍连续 |
| 回滚 | `cp --reflink S_k D'` 换盘 | `fulmar reflink S_k D'` + 原子重命名 |
| 差异 | `xvirt-blockdiff diff S_k S_k+1`（FIEMAP 全扫） | `fulmar diff S_k S_k+1 -o .xvdiff`（树级比对，同格式输出） |
| 跨存储迁移 | reflink + merge 重建链 | 同，`fulmar` 侧提供 `apply/splat` 等价能力或继续用 blockdiff |

---

## 七、动态块大小与碎片治理

### 7.1 三个"块大小"分开定义

| 概念 | 取值 | 作用 |
|---|---|---|
| 寻址 / 最小分配单位 | 固定 4K | 与介质扇区、客户机页、memtier slot 对齐；不可变 |
| 分配提示 `extsize` | 每文件，默认 VM 盘 1MB，pinned 文件 2MB，ISO 4MB | 首次写入 / 顺序扩展时按此粒度分配（可 UNWRITTEN 预留），减少 extent 数 |
| COW 粒度 `cow_chunk` | 每文件，`auto` 或 4K/16K/64K/256K | 快照后首写时 ROW 的最小范围；越大 extent 越少但读放大越大 |

### 7.2 `cow_chunk = auto` 策略

- 介质类：NVMe / SSD（随机读便宜）→ 初值 **4K**；HDD 阵列（随机读贵、顺序吞吐高）→ 初值 **64K**。
- 运行期每文件维护近期写入大小直方图与"相邻 extent 可合并率"：
  - 若 ≥ 70% 写入 ≥ 16K，`cow_chunk` 升到 16K（补齐开销 < 1 次 4K 读，收益是 extent 数降 4 倍）。
  - 若 extent 数超过预算（§7.4）且补齐读开销在当前介质上低于后台合并开销 → 升一档。
  - 反之降档。变化只影响之后的新写，不改历史。
- 读放大上界 = `cow_chunk / 4K - 1` 个 4K 读，用户态可设硬上限（`cow-chunk-max`）。

### 7.3 顺序化分配（把随机写变顺序写）

每个活盘在它所在节点租用的 AG 内持有一个**分配游标**；ROW 的新 extent 从游标顺序取（日志式分配）。随机 4K 客户机写在盘上落成顺序写，这对 NVMe 的写放大、SAN 的条带效率都有利；代价是逻辑 → 物理映射更碎，由 §7.4 兜底。多个盘共用一个 AG 时各自游标分段（每盘先领 64MB 分配区）。

### 7.4 后台合并（coalesce）与 extent 预算

- 每文件 extent 预算：默认 `size / 256K`（1TB → 4M 条）；超预算或叶子中平均 extent < 32K 时触发。
- 合并器挑选"小 extent 密集"的逻辑区间，读出、按 `extsize` 写到新位置、以 ROW 方式替换映射（对正在运行的 VM 透明，写写冲突由同一叶锁串行化；并发写入过的范围放弃本轮）。
- 只在节点 IO 利用率低于阈值时工作，且受 QoS 的"后台类"配额限制（§十）。
- 快照不可变 → 合并只针对活盘；快照引用的旧 extent 不动（保证快照链的差异语义不变）。

### 7.5 refcount 树碎片

共享范围局部被覆盖写会让 refcount 记录分裂。缓解：ROW 范围对齐 `cow_chunk`；延迟释放批处理时顺带合并相邻等值记录；快照删除后整段归零即删记录。

---

## 八、与 QEMU 内存分层（memtier）的配合

### 8.1 现状约束（来自 `docs/qemu-memtier-interface.md`）

- 后端要求：`O_RDWR|O_DIRECT`、无空洞、无 unwritten extent（有则告警）、容量 ≥ 客户机 RAM、支持 discard（块设备默认开）；slot 4K、chunk 2MB 日志式分配 + 压实；换入是**同步 4K 读**，延迟直接决定客户机缺页停顿。
- memtier 与 vhost-user 共享内存互斥；`spdk:` 路径说明 QEMU 已能作为 SPDK 应用初始化（Fulmar 不依赖这一点）。
- 迁移能力 `memtier-shared-backing`：源目的能看到同一后端时，换出页不传输。

### 8.2 Fulmar 为 memtier 提供的文件类：`pinned`

| 属性 | 行为 |
|---|---|
| 创建即全量分配并**物理写零或 WRITE ZEROES** | 没有空洞、没有 unwritten；满足 memtier FIEMAP 检查的精神（通过 Fulmar block_status 直接返回 allocated） |
| **不参与快照 / reflink**，不可 COW | extent 映射创建后不变 → 可以安全地把映射租给客户端 |
| `extsize` = 2MB，与 memtier chunk 对齐 | 一个 chunk 落在一个 extent 内，discard 一个 chunk = 一次 TRIM |
| discard 直接透传到设备 | 等价于块设备上的 `BLKDISCARD`，不走元数据（不会出现"文件上开 discard 拖慢 40%"的 XFS 现象） |
| 不计入 VM 磁盘快照 | hyper-agent 建 VM 快照时自动排除 |
| 厚配额 | 创建时占满配额，避免运行中 ENOSPC 触发 `MEMTIER_BACKING_FULL` |

### 8.3 换入路径：extent 租约直通（方式 C）

```
QEMU memtier/backing-fulmar.c:
  open("fulmar:ds1/vm-123.swap", LEASE_EXTENTS)
    → libfulmar 向 fulmard 申请租约，取回完整 extent 表（pinned 文件 2MB 一条，64GB 仅 32K 条）
       以及设备访问句柄：
         默认：块设备 fd（SCM_RIGHTS 传递，O_DIRECT），QEMU 用自己的 io_uring 实例，
               与现有裸分区后端是同一条代码路径
         可选：SPDK 多进程，QEMU 作为二级进程拿同一 NVMe 控制器自己的 qpair（仅当 fulmard 后端为 spdk）
  换入 4K：slot → 文件偏移 → 查内存 extent 表 → 直接对 fd 发 io_uring 读。路径上没有 fulmard。
  换出写、discard：同样直通。
  租约失效（文件被删除 / 节点 fence）→ fulmard 单向通知，libfulmar 立即停止下发并回 EIO。
```

- 延迟：等于现在裸块设备后端的延迟（同一套 io_uring 代码，`io-poll` / `busy-poll` 旋钮原样生效）；比起直接用裸分区，多的只是**一次性**的租约获取。
- 收益不是"比裸盘更快"，而是：换出文件进入 datastore 命名空间与容量管理；**集群模式下 `memtier-shared-backing` 可用**（目的节点打开同一文件，换出页不传输，迁移 downtime 与带宽显著下降）；一台主机多个 VM 的换出文件共享一个设备而不用预先切分区。
- 直通不可用时（如 fd 传递被安全策略禁止），退回方式 A 的忙轮询环：附加 ≤ 5µs，相对 NVMe 4K 读（~80–100µs TLC）影响 < 5%。
- 资源：io_uring 直通不需要 hugepage，`fault-shards` 每片一个 io_uring 实例（与现状一致）。若选 SPDK 直通，QEMU 作为二级进程需几十 MB hugepage，从 `spdk_quota` 划拨；memtier 的 4K 匿名页约束只针对客户机 RAM，不受影响。

### 8.4 其他配合点

- `backing-discard`：pinned 文件默认 true（与块设备一致）。
- `io-poll` / `busy-poll`：直通路径维持现语义（IOPOLL 需 NVMe poll 队列）；SPDK 直通恒为轮询。
- 统计：`backing-*` 字段来源不变；Fulmar 另给出该文件所在设备的队列深度与延迟直方图供监控合并（`docs/memtier-monitoring-design.md`）。
- 内存快照（`memtier-snapshot-design.md` 的两文件产物）可放 Fulmar 普通文件或 FUSE 网关路径，非热路径。

---

## 九、集群模式：成员、心跳、ATS 锁、锁管理与 fence

### 9.1 成员与 epoch

- 节点槽 1..64，配置写在集群区成员表（节点 id、名称、网络地址列表）。加入 / 退出通过控制 API，由持有**集群配置锁**（ATS）的节点改写，`cluster_epoch` 自增。
- 每节点启动挂载时：读成员表 → 抢自己槽位的**节点锁**（ATS，防同 id 双实例）→ 开始心跳 → 加入网络组。

### 9.2 双通道心跳

| 通道 | 周期 | 内容 | 用途 |
|---|---|---|---|
| **磁盘心跳** | 1s（`hb-interval`） | 本槽写 `{seq, 单调时钟, epoch, 我看到的存活位图}`；每周期读全部槽 | **存活性的最终裁决**（能写盘 = 活着并能破坏数据，必须被纳入） |
| **网络心跳** | 500ms | TCP 保活 + 视图交换 | 快速发现、锁消息、租约续约 |

判死：磁盘心跳连续 `hb-dead`（默认 6）周期未更新 **且** 网络不可达 → 死；仅网络不可达但盘心跳正常 → **网络分区**，走仲裁；仅盘心跳停但网络通 → 该节点自报"失去存储"并自 fence（它已不能安全 IO）。

### 9.3 ATS 盘上锁

- 实现：SCSI `COMPARE AND WRITE`(0x89) / NVMe fused `Compare + Write`。`uring` 后端：NVMe 经 `/dev/ngXnY` 的 io_uring passthrough（或 `NVME_IOCTL_IO_CMD`）发 fused 命令，SCSI 经 `SG_IO`；`spdk` 后端：`spdk_bdev_comparev_and_writev_blocks`。记录一个原子写单元（512B 或 4K，按设备）。
- 锁记录：`{magic, lock_id, owner_node, owner_epoch, lease_seq, type, payload_crc}`。
- 获取：读 → 若空闲或 owner 已被判死且其 epoch 已被回收 → CAW(期望旧值, 新值=我)。失败重读重试（有界退避）。
- 续约：**不逐锁续约**，锁的有效性绑定 owner 的磁盘心跳；owner 心跳活着则它的所有 ATS 锁有效。
- 释放：CAW 写回空闲。
- 用途（慢变化、必须跨网络分区可见的所有权）：集群配置锁、节点槽锁、**AG 租约**、**inode 所有权**（活盘归属）、journal 恢复锁。
- 探测：mkfs/mount 时对锁区做一次 CAW 自测；不支持 → 集群模式要求网络 DLM 必达且 PR 可用，否则拒绝集群挂载（本地模式不受影响）。

### 9.4 网络锁管理（轻量 DLM）

- 锁资源少（命名空间、inode 树根、各 AG 的 refcount 树、延迟释放 RPC），用**按资源哈希选主**的简单 DLM（o2dlm 式）即可，不做 POSIX 范围锁。
- 跨节点元数据操作典型只有两类：
  1. 命名空间（创建 / 删除 / 重命名）：拿命名空间锁，COW 全局目录树，提交，释放。
  2. **释放 / 引用别的节点租用的 AG 里的 extent**（删快照时常见）：不直接改，发 RPC 给 AG 的 owner，由其写入自己的 journal；owner 不在线则等它被恢复后由恢复者处理。
- 网络不可达锁主 → 节点进入"失联"状态，1 个 `hb-dead` 周期内未恢复则按分区仲裁。

### 9.5 仲裁与 fence

- 仲裁：以磁盘心跳位图为准，分区中节点数多者赢；平手取含最小 id 者。输方**自 fence**。
- **自 fence（"卸载故障侧"）**：
  1. `fulmard` 立即把块后端置为拒绝提交，**在途 IO 等待完成或超时**（NVMe abort），所有客户端环回 `EIO/ESHUTDOWN`，租约直通客户端收到撤销通知并停发；
  2. QEMU `werror=stop,rerror=stop` → VM 暂停（不崩），hyper-agent 收到 `fulmard` 事件 `FENCED`；
  3. `fulmard` 停止心跳写、释放不了的 ATS 锁留给幸存者回收；
  4. **不 panic、不重启主机**。恢复连通后 `fulmard` 以新 epoch 重新加入，VM 由 hyper-agent 决定在原地恢复（若尚未被别处拉起）或已被 HA 接管则销毁。
- **幸存者侧 fence（兜底，解决"僵尸 IO"）**：存储支持 SCSI-3 PR / NVMe Reservation 时（`uring` 后端用块层统一的 `IOC_PR_REGISTER` / `IOC_PR_PREEMPT` ioctl，SCSI 与 NVMe 同一接口），每节点注册 key；判死后幸存者 `PREEMPT` 其 key，存储层拒绝它一切后续写。不支持 PR 时，依赖时序保证：自 fence 的触发点（自己 `hb-dead/2` 周期写不进心跳）严格早于幸存者接管点（`hb-dead` 周期 + 恢复锁），并把 NVMe 命令超时设短于该差值。
- **恢复**：拿到"恢复锁(ATS)"的幸存者重放死节点 journal → 回收其 AG 租约 / inode 所有权 → 执行其未完成的延迟释放 → 标记槽位可用。journal ≤ 512MB，目标 ≤ 2s。
- 时间目标：检测 ≈ 6–7s，接管 ≤ 10s；可按场景调 `hb-interval`/`hb-dead`。

### 9.6 在线迁移时的所有权交接

QEMU 迁移中源与目的同时打开同一活盘：目的以 `inactive` 打开（只建映射，不写），源在 `bdrv_inactivate` 时通过 libfulmar 发起**所有权移交**（源节点 `fulmard` 释放 inode 所有权 ATS 锁并把叶缓存脏数据提交，目的节点抢锁并激活）。失败则迁移回滚，源重新激活。这是 OCFS2 用 DLM 隐式完成的事，Fulmar 显式化为协议，`lictor`/`hyper-agent` 的迁移流程只需在现有 `inactivate → 切换` 点调用。

---

## 十、按磁盘限速（QoS）

- 每打开文件句柄一组 **令牌桶**：读 IOPS、写 IOPS、读 BW、写 BW、总 IOPS/BW，各带突发值（`*_max`、`*_max_length`，与 QEMU throttle 参数对齐，便于从 libvirt/QEMU 限速迁移）。
- 层级：句柄 → 文件（跨节点读快照时各自独立） → VM（多盘共享一个池）→ datastore → 设备。设备饱和时用 **mClock**（reservation / limit / weight）在 VM 间做加权公平，后台类（合并、快照删除、scrub、迁移拷贝）单独一类低权重。
- 节流点在 **出环（dequeue）**：不从 SQ 取请求即形成背压，完成永不延迟；忙轮询 reactor 在节流期间转去服务其它环。
- 策略持久化在 inode（跟盘走，迁移后自动生效），运行时由控制 API 改，立即生效。
- 可观测：每文件延迟直方图、被节流时间占比、令牌欠账。

---

## 十一、一致性、持久化与崩溃恢复

- **无数据缓存**：客户机写直达设备，`FUA` 透传。
- **元数据**：内存中 COW 树修改 + 每节点 **intent log（journal）组提交**（默认 ≤ 200µs 或凑满 64K 即落盘）；**检查点**每 5s 或 journal 用量 50% 时把 COW 树根写入超级块（原子）。
- **写完成语义**：Fulmar 向 QEMU 暴露"带易失写缓存"的设备（virtio-blk `WCE=1`），即**数据落盘但映射记录可能还在组提交批里**时就可完成写；客户机 `FLUSH` → 等待 journal 落盘 + 设备 flush。这与 qcow2 `lazy_refcounts`、裸盘写缓存语义等价，是低延迟的关键；可用 `sync-meta=always` 强制每写等 journal。
- **ROW 保证不暴露垃圾**：新映射只在新数据落盘后发布；崩溃丢的只是未 FLUSH 的最新写，读回的是快照版旧数据，符合块设备语义。
- **ENOSPC**：元数据全局保留区（≥ 1% 或 2GB）保证删除 / 释放永远能执行；活盘写 ENOSPC 时 QEMU `werror=enospc` 暂停 VM，上层扩容后恢复。
- **崩溃恢复**：超级块最新检查点 + 重放本节点 journal；延迟释放队列持久化，幂等重放。
- **fsck**：离线检查引用计数（扫全部树重算）、空闲树一致性、校验和；在线 scrub 只读校验元数据校验和。

---

## 十二、接口：控制面、数据面、POSIX 网关与工具

### 12.1 控制 API（unix socket，JSON，hyper-agent 调用）

`fs.mkfs / fs.mount / fs.umount / fs.grow`、`cluster.{status,add,remove,enable,fence}`、`file.{create,delete,rename,resize,stat,list,set-policy}`、`snap.{create,delete,diff}`、`reflink`、`qos.set`、`events.subscribe`（`FENCED / DEGRADED / ENOSPC / LEASE_REVOKED / RECOVERED`）。

### 12.2 数据 API（libfulmar，C ABI）

`fulmar_open/close`、`fulmar_preadv/pwritev/flush/discard/write_zeroes`（异步 + 完成回调或轮询）、`fulmar_block_status(off, len) → {allocated|zero|unwritten|hole, shared, phys?}`、`fulmar_lease_extents`（pinned 文件）、`fulmar_set_inactive/activate`（迁移交接）。

QEMU `block/fulmar.c` 把 `block_status` 直接映射给 `bdrv_co_block_status`，从而 drive-mirror 的零区跳过与厚薄盘判定（`drive-mirror-sparse-thick-design.md`）天然成立：厚盘 = `thick` 配额 + UNWRITTEN，报 `ZERO`；无 `ZERO_RANGE` 退化问题。

### 12.3 POSIX 网关（FUSE）

只服务管理用途：qemu-img 转换导入、ISO 上传、浏览、备份工具读取快照、`snapshot-helper` 写内存快照文件。读写经 `fulmard`，性能够用即可；活盘**禁止**经 FUSE 写（打开即拒绝，防绕过所有权）。

### 12.4 工具

- `fulmar diff S_a S_b`：自两根向下比对，子树指针相同即跳过，输出 `.xvdiff`（与 `xvirt-blockdiff`、`petrel-snapdiff` 同格式，上下游无需改）。
- `fulmar fiemap F`：等价 FIEMAP 输出（逻辑、物理、长度、shared/unwritten 标志），保留现有工具兼容路径。
- `fulmar stat F`：extent 数、共享比例、`cow_chunk` 当前值、碎片度、QoS 统计。

---

## 十三、与现有组件的衔接与迁移路径

| 组件 | 改动 |
|---|---|
| `hyper-agent` datastore | 新 backend `fulmar.py`（`SUPPORTED_FS_TYPES` 加 `fulmar`）：mkfs / mount / 快照 / reflink / diff / QoS 全走控制 API；VM 配置文件仍可放 Fulmar（FUSE 路径）或本地 datastore |
| `hyper-agent` qemu_command | 盘类型 `fulmar` 生成 `-blockdev driver=fulmar,…` 或（零拷贝档）vhost-user-blk；memtier `path` 支持 `fulmar:` |
| `qemu` | 新增 `block/fulmar.c`、`memtier/backing-fulmar.c`；迁移 `inactivate` 钩子调所有权交接 |
| `lictor` | 快照 / 迁移流程调用 datastore 接口，不直接碰文件系统（现状若有 `cp --reflink` 直调则替换） |
| `xvirt-blockdiff` / `xvirt-snapmig` / `cdp` | 可不改（`fulmar fiemap` 兼容）；二期切 `fulmar diff` 提速 |
| 资源规划 | `fulmard` 轮询 IO 线程的核与环用 hugepage 从 `spdk` 池划拨（建议改称存储池）；eventfd 模式不占池核。启用 SPDK 档时再加 vfio 绑定与 SPDK 大页 |
| OCFS2 → Fulmar 迁移 | 同机加一块 Fulmar datastore，用现有跨 datastore 快照链迁移（根全量 + 逐层 `merge`）或 drive-mirror 在线迁移；OCFS2 只读保留直至清空 |

---

## 十四、其他需要关注的点（清单）

按"没想到就会出事"的优先级排列。

1. **僵尸 IO 与 fence 完备性**：无 PR 的阵列上只能靠时序（自 fence 早于接管）；要把 NVMe/SCSI 命令超时、多路径故障切换时间都算进 `hb-dead` 的下限，否则一条卡在 HBA 里 30s 的写可能在接管后落盘。部署文档要给出按存储类型的参数表。
2. **ATS / PR 能力矩阵**：NVMe-oF 目标（SPDK/Linux target）对 fused Compare+Write 与 Reservation 的支持不一；FC/iSCSI 阵列的 CAW 块大小（512 vs 4K）与 VAAI 开关；mount 时探测并把结果写进 datastore 元数据与 UI。
3. **在线迁移的双开窗口**：§9.6 的所有权交接必须与 QEMU 的 `inactivate/activate` 严格配对；迁移失败回滚、目的端崩溃、源端崩溃三种路径都要测。
4. **多读者**：备份 / CDP 读快照与活盘并行；快照不可变使之简单，但"删除正在被别的节点读的快照"需要引用计数（打开计数走 DLM）或延迟到读者关闭。
5. **ENOSPC 下的 COW 死锁**：删除需要分配（COW 树节点）；全局保留区与"删除优先"分配路径必须从第一天就有。
6. **空间会计与超分**：ROW 让"快照占用"随活盘写入增长而增长；按文件报 `exclusive / shared / referenced` 三个量，datastore 给出近满告警（thin 超分）与 `MEMTIER_BACKING_FULL` 前置预警。
7. **extent 数与内存**：无上限的 ROW 会让元数据缓存膨胀，§7.4 预算与合并器是必需品而非优化；合并器与客户机写的竞争要有退让策略。
8. **refcount 树热点**：多个活盘共享一个 AG 的 refcount 树时，延迟释放批处理是单点；按 AG 分片 + owner 本地执行能缓解，仍需压测删快照风暴（一次删除几十个快照）。
9. **元数据校验与误写检测**：共享存储上旧节点、错误 LUN 映射、多路径错配都可能造成"错位写"；块内 fsid + 位置 + 代号校验是防线，fsck 要能定位。
10. **4Kn / 512e、对齐与 WRITE ZEROES / TRIM 语义差异**：UNMAP 后读到零不保证（`DRAT/RZAT`），厚盘 ZERO 语义不能依赖设备 TRIM；`write_zeroes` 大范围优先做元数据 ZERO 标记而非下发设备命令。
11. **Flush 成本**：带易失缓存的 NVMe-oF 后端 `FLUSH` 可能毫秒级；客户机频繁 fsync 的负载要评估"组提交 + 设备 flush 合并"效果；可选 `fua-as-flush` 策略。
12. **格式版本与在线升级**：超级块 `compat/ro_compat/incompat` 三组特性位；集群内混合版本运行期间禁止启用新 incompat 特性。
13. **时钟**：心跳只用单调计数与序号，不依赖节点间时间同步；日志中带墙钟仅供排障。
14. **大页与 NUMA 预算**：`fulmard` 环、bounce 缓冲（SPDK 档还有队列）都吃 hugepage；每 VM 每 iothread 的环大小 × VM 数要纳入资源规划，否则会与 1G 大页 VM 抢内存。
15. **可观测性**：Prometheus 指标（每文件 / 每节点延迟直方图、节流、心跳抖动、锁等待、journal 占用、延迟释放积压）、结构化事件、`fulmar trace` 按文件采样请求级 tracing；心跳抖动本身就是存储健康的先行指标。
16. **安全边界**：控制 socket 仅 root / 专用组；环内 iov 范围校验；FUSE 网关默认只读；多租户不是目标，但要防误操作（活盘禁止 FUSE 写、删除活盘要求先无所有者）。
17. **测试基础设施**：崩溃一致性（`dm-flakey` / `dm-dust` 模拟断电与乱序落盘，SPDK 档用故障注入 bdev）、集群故障注入（网络分区、盘心跳延迟、僵尸节点）、xfstests 子集经 FUSE、fio / vdbench 性能回归、与 OCFS2 同机对比基线。
18. **开发与运维工具链**：`fulmar-fsck` 必须与格式同步交付；`fulmar dump`（导出元数据树供离线分析）；只读挂载模式；"紧急只读"降级（元数据校验失败时）。
19. **SPDK 档的代价**：vfio 独占整个 NVMe 控制器（同盘不能再分区给 XFS 或裸分区 memtier）、轮询核 100% 占用、多进程模式脆弱（二级进程崩溃可能拖累主进程共享内存）、无 FC 支持。只在低延迟介质 / NVMe-oF / 已预留 SPDK 核的主机上启用；内核 io_uring 路径是永远可退回的基线。
20. **软件组件替换的组织成本**：QEMU fork 增两个驱动的长期维护；FUSE 路径让已有脚本"看起来还是文件"但性能与语义不同（无 `mmap`、无 `O_DIRECT` 保证），运维脚本需逐一检查。

---

## 十五、性能目标与验证方法

> 以下为目标与估算，未实测。基线用同一台机、同一 NVMe 上的裸盘与 OCFS2（raw 文件）对比。

| 场景 | OCFS2（内核，raw） | Fulmar 目标 | 验证 |
|---|---|---|---|
| 4K 随机读延迟（QD1） | 裸盘 + 10–20µs | 裸盘 + ≤ 8µs（环 + io_uring 轮询）/ ≈ 裸盘（直通）/ 裸盘 + ≤ 5µs（SPDK 档） | fio in-guest，`cache=none` vs `fulmar` |
| 4K 随机写延迟，快照后首写 | 1–3ms（1MB cluster RMW + refcount 更新） | 裸盘 4K 写 + ≤ 10µs | 建快照后立即 fio randwrite |
| 顺序读写吞吐（1MB） | ≈ 裸盘 | ≥ 95% 裸盘 | fio seq |
| reflink 1TB、1M extent | 数十秒到分钟 | ≤ 10ms | `fulmar snap` 计时 |
| 删快照（1TB） | 秒到分钟级前台 | 前台 ≤ 10ms，后台回收不影响前台 P99 | 删除期间 fio P99 |
| memtier 换入 4K | 走裸分区 ≈ 裸盘 | ≈ 裸盘（直通）/ +≤5µs（环） | 现有 `tests/` redis 基准 + `evict-reevict` |
| 节点故障到接管 | ≈ 62s（默认 o2cb） | ≤ 10s | 拔网线 / 拉黑 LUN |
| 快照差异（1TB，1% 变化） | FIEMAP 全扫（分钟级） | 秒级 | `fulmar diff` vs `xvirt-blockdiff diff` |

---

## 十六、分阶段实施计划与风险

| 阶段 | 内容 | 退出标准 |
|---|---|---|
| **P0 原型（本地模式）** | 盘上格式 v0、COW 树、ROW 写、reflink O(1)、`uring` 裸盘后端、`fulmard` + 环 + `libfulmar` + QEMU `block/fulmar.c`、`fulmar` CLI 基本命令、FUSE 只读 | 单机 VM 能启动、快照、回滚；崩溃一致性测试通过；延迟目标达 80% |
| **P1 memtier + 工具链** | `pinned` 类、`backing-fulmar.c`（fd 直通 + 环模式回退）、`fulmar diff/fiemap`、hyper-agent `fulmar` datastore、QoS 令牌桶、合并器 | memtier 基准与裸分区后端持平（同一 io_uring 路径）；blockdiff 流程不改即可跑；OCFS2 → Fulmar 迁移演练 |
| **P2 性能档（可选）** | vhost-user-blk 零拷贝档（rust-vmm）、io_uring IOPOLL / SQPOLL / 固定缓冲调优、`spdk` 后端（nvme / nvmf bdev）作为可开关特性、SPDK 二级进程直通评估 | `uring` 档延迟目标达成；`spdk` 档在低延迟介质 / NVMe-oF 上给出实测收益表，决定是否纳入默认部署 |
| **P3 集群** | 成员 / 双心跳 / ATS 锁 / DLM / AG 租约 / inode 所有权 / 迁移交接 / 自 fence / PR 抢占 / journal 恢复 | 故障注入矩阵通过；接管 ≤ 10s；`memtier-shared-backing` 跨节点迁移验证 |
| **P4 运维成熟** | fsck / scrub / 在线扩容 / 指标 / 版本升级 / 文档 | 生产灰度 |

主要风险与对策：

| 风险 | 对策 |
|---|---|
| COW 文件系统的引用计数与 ENOSPC 正确性 | 模型级单测（随机操作序列 + 全量重算校验）、持续模糊测试、fsck 从 P0 就有 |
| SPDK 档带来运维复杂度与稳定性风险 | 默认不启用；内核 io_uring 路径永远可用，SPDK 作为按主机开关的特性 |
| 阵列 ATS / PR 支持缺失 | 能力探测 + 时序兜底 + 明确的"不支持则拒绝集群挂载"策略 |
| QEMU fork 维护面扩大 | 驱动保持薄（协议在 libfulmar 内），对上游 QEMU 版本变化敏感面小 |
| 实现语言与团队栈 | 见 §17 |

---

## 十七、待拍板事项

1. **实现语言**：Rust（与 petrel 同栈，直接复用其 io_uring / vhost-user-blk / rust-vmm 代码）vs C（与 QEMU、SPDK 同栈）。建议：`fulmard` 用 Rust，块后端抽象先只实现 `uring`；`spdk` 档通过薄 FFI（bdev 提交 / 完成、CAW、reservation 几个接口）后接；`libfulmar` 提供 C ABI 给 QEMU。
2. **默认 `cow_chunk`**：NVMe 上 4K 还是 16K 起步（后者 extent 数少 4 倍，代价是最多 3 个 4K 补齐读）。建议 `auto`，初值按介质。
3. **是否保留 FUSE 可写**：建议仅非活盘、非 pinned 文件可写。
4. **集群规模上限**：64 槽（心跳区 256KB）是否够；OCFS2 为 255。
5. **memtier 直通是否作为 P1 默认**：fd 直通与现有裸分区后端同一条路径、风险低，建议 P1 即默认，环模式作回退。
6. **SPDK 档是否进入本期范围**：建议只做后端抽象预留与 P2 实测评估，不作为交付承诺。
