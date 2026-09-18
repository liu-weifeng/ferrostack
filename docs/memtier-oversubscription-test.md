# 内存分层超分 + Petrel 部署测试报告

> 真机环境：xvirt 节点 **192.168.2.123**（standalone 角色），2026-06-05。
> 全程通过 xvirt REST API 创建/管理虚拟机；内存分层换出后端在 NVMe；存储池为 Petrel 分布式块存储。

---

## 1. 测试环境

| 项 | 配置 |
|---|---|
| CPU | Intel Xeon E5-2673 v3 (1 socket, 12C/24T) |
| 物理内存 | 16 GB（`MemTotal` 15312 MiB） |
| 系统盘 | nvme0n1 (K-RICARD 128G)：`/` `/var` `/datastore`(local-pool, XFS) |
| Petrel 数据盘 | sda/sdb/sdc/sdd — 4× WDC WD5000AAKX 500G HDD |
| Petrel 元数据盘 | nvme2n1 — Intel Optane M10 16G（低延迟，承载各 OSD RocksDB/WAL） |
| 内存分层后端 | nvme1n1 — Kingchuxing 128G → XFS 挂 `/memtier`（117G 可用，支持 O_DIRECT） |
| QEMU | 自研 `memory-tiering` 分支 10.0.7+gitf11a9d3b9f（4 个 `x-memtier-*` QMP 命令） |
| 控制面 | ferrovirt + hyper-agent(主机级总控) + lictor(持 QMP) |

> 目标盘（sda–sdd / nvme1n1 / nvme2n1）测试前已清空重建；系统盘 nvme0n1 未触碰。

---

## 2. Petrel 分布式存储部署（全程 API）

```
POST /api/vcenter/storage/petrel/cluster
  { "meta_disk":"nvme2n1", "data_disks":["sda","sdb","sdc","sdd"], "replica":2, "pool":"nas_data" }
POST /api/vcenter/datastore
  { "type":"petrel", "name":"petrel-pool", "pool_name":"nas_data", "create_pool":false, "host":"<uuid>" }
```

结果：

- **4/4 OSD up**（osd.0–3 落 4 块 HDD），mon @127.0.0.1:6790，pool `nas_data`（size=2, pg_num=256）。
- Optane(nvme2n1) 承载全部 OSD 的元数据/WAL。
- 数据存储 `petrel-pool` 注册成功，前端可见、可放卷。
- 内存分层测试结束后复查：集群仍 **4/4 up**，未受影响。

---

## 3. 内存分层测试方法

- **VM 全部经 xvirt API 创建**（`POST /api/vcenter/vm`，`memtier:{enabled,policy,priority,path}`）；lictor 启动时 `x-memtier-enable`，hyper-agent 主机级总控每 5s 按主机压力+优先级实时下发换出策略。
- 换出后端：每 VM 一个 `/memtier/<vm>.raw`（在 nvme1n1）。
- VM 磁盘用普通 qcow2（**memtier 与 petrel/vhost-user-blk 互斥**：前者需匿名 RAM，后者需共享 memfd）。
- Guest 内运行自写 `membench`（cache-line 步长随机指针追逐，报告平均访存延迟 ns/访问；源码 `.run/membench.c`）。两种负载画像：
  - **均匀随机**：整 1.5GB 工作集持续随机访问 = 分层的**最坏情况**（无冷页可分层）。
  - **局部性**：256MB 热集持续访问 + 0.1% 访问散到冷区 = **真实业务**画像（冷页可廉价换出）。
- **基线**（无压力全驻留）：**~114 ns/访问**（即 DRAM 随机访问真实延迟）。
- 超分比 = Σ(VM 内存) / 物理内存(15312 MiB)。

---

## 4. 结果

### 4.1 最坏情况（均匀随机访存，2GB VM）

| 超分比 | VM | 换出到 nvme1n1 | 平均延迟 | 相对基线 |
|---|---|---|---|---|
| 0.26x | 2 | 0 | 117 ns | 1.0x |
| 0.53x | 4 | 0 | 126 ns | 1.1x |
| 0.80x | 6 | 2.4 GB | 621 ns | 5.3x |
| 1.07x | 8 | 6.4 GB | 12905 ns | **110x** |
| 1.33x | 10 | 9.7 GB | 45439 ns | **390x** |
| 1.60x | 12 | 13.0 GB | 53732 ns | **461x** |

→ 工作集一旦溢出物理内存便**崩塌**：均匀随机访问没有冷页，被换出的页立刻又被访问，缺页风暴。这是分层的理论下界。

### 4.2 真实情况（局部性：256MB 热集 + 0.1% 冷访问，2GB VM）

| 超分比 | VM | 换出到 nvme1n1 | 平均延迟 | 相对基线 | 存活 VM |
|---|---|---|---|---|---|
| 0.26x | 2 | 0 | 114 ns | 1.0x | 2/2 |
| 0.53x | 4 | 2.8 GB | 373 ns | 3.3x | 4/4 |
| 1.07x | 8 | 8.6 GB | 1138 ns | 10x | 8/8 |
| 1.60x | 12 | 14.7 GB | 1931 ns | 17x | 12/12 |
| 2.14x | 16 | 21.0 GB | 2583 ns | 23x | 16/16 |
| **2.67x** | **20** | **27.7 GB** | 4649 ns | 41x | **20/20** |

→ 同 1.07x：局部性 **10x** vs 均匀 **110x**（差 11 倍）；同 1.60x：**17x** vs **461x**（差 27 倍）。
→ **16GB 主机稳定运行 40GB 虚拟机内存（2.67x），20 台全部存活响应**；主机驻留被总控压到 ~13GB，27.7GB 冷页落 nvme1n1。

### 4.3 把 nvme1n1 压满（14× 8GB 低优先级 VM，commit 后即冷）

| VM | 总 committed | 主机驻留 | 真实换出到 nvme1n1 | nvme1n1 df |
|---|---|---|---|---|
| 3 | 24 GB | 4.8 GB | 19.8 GB | 27 G |
| 6 | 48 GB | 6.5 GB | 42.7 GB | 51 G |
| 9 | 72 GB | 7.6 GB | 66.1 GB | 75 G |
| 12 | 96 GB | 8.2 GB | 90.1 GB | 99 G |
| **14** | **112 GB** | **8.5 GB** | **~104 GB（106164 MB）** | **115 G / 120 G (96%)** |

→ **16GB 物理内存的主机跑 14 台 8GB VM = 112GB 虚拟机内存（7.5x 超分）**，~104GB 真实客户机页换出到 nvme1n1（把盘压到 96% 满），主机驻留仅 8.5GB，14 台全活、membench 持续响应（热集 ~150ns），全程无 OOM。

---

## 5. 关键结论

1. **超分能力**：内存分层让 16GB 主机安全超分到 **7.5×（112GB 虚拟机内存）**，把 128GB NVMe 用真实冷页填到 **96% 满**。
2. **性能损耗由"冷页访问率（局部性）"主导，而非超分比本身**：只要热集放得进物理内存，超分到 2.67x 仍只有 ~40x 延迟代价（绝对值几 µs，对非访存密集业务可接受）；反之均匀随机访存（无局部性）刚过 1.0x 就 110x 崩塌。**业务内存局部性是超分成败的决定因素。**
3. **主机级总控有效**：hyper-agent 按实时压力把每台 VM 自动压到常驻下限（低优先级最深 ~5% RAM），无需人工干预。

---

## 6. 工程注意点（坑）

- **同时 commit 会 OOM**：多台 VM 首次提交工作集（memset 全量）会瞬时冲过换出速度 → 内核 OOM 杀 qemu。必须**节流 VM 启动**（等上一台被压缩、主机内存回落再起下一台），或在调度器加内存准入控制。
- **常驻下限公式**（`hyper_agent/support/memtier/controller.py`）：`floor = base × (1 − 0.8×pressure)`，base = {high 0.75, normal 0.5, low 0.25}，最深约 5% RAM，绝对下限 `MIN_FLOOR_MB=64`。深度填盘用 `priority=low` + 调高 `XVIRT_MEMTIER_FREE_LOW_PCT`（顶高空闲、逼总控持续换出）。
- **memtier 与 petrel/vhost-user 互斥**：分层 VM 必须用 lvm/local/file 普通盘（匿名 RAM），不能挂 petrel 块盘（共享 memfd）。
- **后端容量**：每 VM backing 文件按全内存 `fallocate`，故 nvme1n1 可承载的「VM 内存总量」≈ 盘容量；满后发 `MEMTIER_BACKING_FULL` 暂停换出（无正确性影响）。

---

## 7. 数据与脚本位置

- 原始数据：节点 `/var/lib/xvirt-test/results-uniform.csv`、`results-locality.csv`、`fill.log`。
- 基准源码：dev box `/root/code/xvirt/.run/membench.c`。
- 相关设计：`docs/memory-tiering.md`、`docs/qemu-memtier-usage.md`。
