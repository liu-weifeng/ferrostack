# 内存分层(memtier)redis-benchmark 性能测试报告

> 测试节点 **192.168.2.123**,2026-06-12。对比:**无分层基准** vs **memtier 内存分层**
> (NVMe / petrel 两种后端,超分 1.5–9×)vs **传统主机 swap**(NVMe,相同 floor)。
>
> ⚠️ **重要前提**:测试 CPU 为 **Xeon E5-2673 v3(Haswell-EP)**,该家族 KVM 下 `page_idle`
> 空闲跟踪失效(见 `[[qemu-memory-tiering]]` / CLAUDE.md),memtier 在此机器上**只能走降级的
> uffd-WP 写老化(write-aging)路径**——只识别写热度、每页挂写保护、换出/换入走 uffd+io_uring
> 用户态路径。**本报告的 memtier 数字代表降级模式下的下限**;在 `page_idle` 有效的 CPU(如 dev
> 机 114)上,memtier 有真实读热度跟踪,表现会显著更好。

---

## 1. 测试环境

| 项 | 值 |
|---|---|
| 宿主机 | 192.168.2.123,Xeon **E5-2673 v3** @2.40GHz(12C/24T),15 GiB RAM |
| QEMU | `qemu-memtier`(xvirt fork,memory-tiering 分支 `20910cc5e4`,含 write-aging 修复) |
| 客户机 | Ubuntu 24.04 cloud image,**6144 MiB 匿名 RAM**,4 vCPU,**无 guest 内部 swap** |
| Redis | redis-server 8.0.5,`maxmemory 0`(不淘汰)、`save ""`、`appendonly no` |
| 数据集 | **~3.4 GiB**:redis-benchmark SET 灌入 ~334 万键 × 1000 B(键 `key:NNN…`,与压测键一致) |
| memtier 后端①| 本地 NVMe:`/memtier/backing.raw`(8 GiB,XFS on Kingchuxing 128 GB NVMe) |
| memtier 后端②| petrel 分布式卷:`mtswap-bench` 8 GiB → `petrel-ublk serve` → `/dev/ublkb5`(2 节点集群 123+106) |
| 传统 swap | 主机 NVMe swapfile `/memtier/hostswap`(8 GiB,同一块 nvme1n1) |

---

## 2. 测试方法

- **压测工具**:`redis-benchmark -t get,set -n 300000 -c 50 -d 1000 -r 3400000 -P 1 --csv`
  —— **在客户机内部跑**(localhost),避免 QEMU user-net(slirp)成为瓶颈掩盖后端差异。
  `-r 3400000` 让 GET/SET 随机命中**整个 3.4 GiB 数据集**(均匀随机 = **最坏局部性**,无热点)。
- **超分比定义**:客户机 RAM 固定 6144 MiB,比率 R ⇒ 常驻下限 floor = 6144/R。
  - memtier:`x-memtier-enable path=<后端> policy=aggressive tunables={psi-trigger:0, min-resident-mb:floor}`,
    等 resident 收敛到 floor(E5-26xx v3 上首次需等 ~90s 让 write-aging 引擎启动)后压测。
  - 传统 swap:把 QEMU 进程放进 host cgroup,`systemctl set-property MemoryHigh=floor`,内核把
    客户机 RAM(QEMU 匿名内存)换出到 NVMe swap;`MemoryMax=infinity` + 充足 swap **防 OOM 杀进程**。
- **比率梯度**:1.5× / 2× / 3× / 5× / 7× / 9×。
- **采集**:GET/SET 的 rps、p50/p99/avg 延迟;后端的 resident/swapped/swapin-faults。

---

## 3. 基准分数(无内存分层,数据集全驻留)

| 操作 | rps | p50 (ms) | p99 (ms) | avg (ms) |
|---|---|---|---|---|
| GET | **63 251** | 0.41 | 0.88 | 0.42 |
| SET | **60 852** | 0.45 | 1.10 | 0.49 |

---

## 4. memtier 内存分层 × 本地 NVMe

| 超分比 | floor (MiB) | 实测 resident / swapped (MiB) | GET rps | GET p99 (ms) | SET rps | SET p99 (ms) | 累计换入缺页 |
|---|---|---|---|---|---|---|---|
| 基准 | 6144 | 全驻留 | 63 251 | 0.88 | 60 852 | 1.10 | — |
| 1.5× | 4096 | 4103 / 2040 | 17 584 | 9.9 | 14 584 | 13.3 | 230 K |
| 2×   | 3072 | 3082 / 3061 | 8 472 | 28.1 | 8 343 | 24.6 | 398 K |
| 3×   | 2048 | 2074 / 4069 | 4 283 | 49.7 | 4 134 | 54.5 | 674 K |
| 5×   | 1229 | 1418 / 4725 | 1 328 | 158.8 | 1 440 | 137.7 | 1.65 M |
| 7×   | 878  | 1281 / 4862 | 865 | 254.0 | 822 | 274.4 | 2.38 M |
| 9×   | 683  | 1241 / 4902 | 779 | 291.8 | 703 | 347.4 | 2.70 M |

- resident 能真正压到 floor(write-aging 引擎启动后),swapped 升到 ~4.9 GiB,功能正确、数据无损。
- 均匀随机访问(无局部性)下退化剧烈:9× 时 GET 仅剩基准的 **1.2%**,p99 飙到 ~292 ms。
- **退化由冷页访问率主导**:每个 redis 操作会触碰多个冷页(数据结构本身也被换出),缺页数远超请求数。

---

## 5. memtier 内存分层 × petrel 分布式卷 —— **不适用于 redis**

- **换出正常**:resident 压到 floor、swapped 2 GiB 落入分布式 petrel 池(池用量增长,跨 123+106 副本)。
- **换入(swap-in)致命**:压测时 redis **吞吐坍塌到 <100 ops/s**——20000 次 GET 在 **200 s 内跑不完**,
  缺页解析速率仅 **~140 faults/s**。
- **根因**:petrel 分布式卷每次 swap-in 要走**网络 + 远端 OSD 读**(ms 级延迟),而 NVMe 是 µs 级。
  redis 单线程在每个缺页上串行阻塞,均匀随机访问下缺页风暴使吞吐塌到接近零。压测期间还观察到
  guest vCPU `soft lockup`(等不来的 swap-in)。
- **结论**:分布式存储作内存分层交换盘**只适合"热集稳定驻留、极少缺页"的场景**;对 redis 这种
  随机访问 + 对延迟敏感的负载**不可用**。按用户判断,该项已停止。

---

## 6. 传统主机 swap 对比(NVMe,相同 floor)

把 QEMU 进程(= 客户机 RAM)用 host cgroup `MemoryHigh` 压到与 memtier 相同的 floor,内核换出到 NVMe swap:

| 超分比 | floor (MiB) | QEMU resident / host-swap (MiB) | GET rps | GET p99 (ms) | SET rps | SET p99 (ms) |
|---|---|---|---|---|---|---|
| 1.5× | 4096 | 4002 / 551 | 48 325 | 1.4 | 40 069 | 2.5 |
| 2×   | 3072 | 3071 / 1405 | 14 679 | 13.5 | 14 053 | 10.9 |
| 3×   | 2048 | 2010 / 2731 | 7 986 | 39.2 | 7 588 | 40.7 |
| 5×   | 1229 | 870 / 3683 | 6 018 | 42.4 | 4 960 | 46.2 |
| 7×   | 878  | 806 / 3952 | 4 477 | 53.2 | 4 426 | 47.8 |
| 9×   | 683  | 680 / 4084 | 3 900 | 51.3 | 4 077 | 50.9 |

- **全程无 OOM**(`MemoryHigh` 软限只 throttle+reclaim 不杀,`MemoryMax=infinity`,swap 充足)。
- 注意 floor 浅时(1.5×)host-swap 只换出 551 MiB(数据集基本驻留),而 memtier 主动把整个 6 GiB
  region(含未触碰的零页)压到 floor、换出 2 GiB——这是两者机制差异,但**在相同 floor 下对比**仍成立。

---

## 7. 综合对比与分析

**GET 吞吐(rps)三方对比**(基准 63 251):

| 超分比 | memtier-NVMe | 传统 host-swap | host-swap / memtier | memtier-petrel |
|---|---|---|---|---|
| 1.5× | 17 584 | **48 325** | 2.7× | 坍塌 |
| 2×   | 8 472 | **14 679** | 1.7× | 坍塌 |
| 3×   | 4 283 | **7 986** | 1.9× | 坍塌 |
| 5×   | 1 328 | **6 018** | 4.5× | 坍塌 |
| 7×   | 865 | **4 477** | 5.2× | 坍塌 |
| 9×   | 779 | **3 900** | 5.0× | 坍塌 |

### 关键发现

1. **在这台 E5-2673 v3 上,传统主机 NVMe swap 全面快于 memtier,且深比率差距拉大到 ~5×。**
   原因是 memtier 在该 CPU 上**只能走降级的 write-aging 模式**:每页挂 uffd-WP 写保护、每次写触发
   WP 缺页回滚判定、换出/换入走用户态 uffd+io_uring,还有持续的冷页扫描线程争 CPU。这些开销在
   写敏感、随机访问的 redis 上被放大。内核原生 swap 路径(direct reclaim + 主缺页 + NVMe 预读批量)
   per-fault 更省、无用户态往返。

2. **petrel 分布式卷作交换盘对 redis 不可用**——swap-in 走网络+OSD(ms 级),随机访问缺页风暴下
   吞吐坍塌到 <100 ops/s。分布式后端只适合"热集驻留、极少换入"的工作负载。

3. **退化由局部性(冷页访问率)主导,而非超分比本身**。本测试用均匀随机(`-r` 覆盖全集)是
   **最坏情况**;真实业务有热点时,三种方案的缺页率都会大幅下降、分数显著回升(参见
   `docs/memtier-oversubscription-test.md` 在 dev 机上的"热集驻留 + 0.1% 冷访问"结论:同样超分比下
   损耗比均匀随机低 11–28 倍)。

4. **OOM 防护有效**:host cgroup `MemoryHigh`(软限)+ `MemoryMax=infinity` + 充足 NVMe swap,
   可在主机层把 VM 内存安全超分到 9×、全程不触发 OOM 杀进程。

### memtier 的适用边界(结合本测试)

- memtier 的价值**不在于裸吞吐**,而在于:① 对客户机**完全透明**(无需 guest 配 swap);
  ② 在 `page_idle` 有效的 CPU 上能识别**读热度**;③ 主机级按优先级实时调度多 VM。
- **在 page_idle 失效的 E5-26xx v3 上,若只追求单 VM 吞吐,传统主机 swap 是更优选择**。
- 要发挥 memtier 优势,应:① 跑在 page_idle 有效的较新 CPU(Skylake+);② 用于有稳定热集的负载
  (而非均匀随机);③ 配本地 NVMe / 高速盘(不要分布式卷)。

---

## 8. 复现要点

- 客户机镜像 `/memtier/redis.qcow2`(cloud-init 装 redis + 开 `enable-debug-command`),seed `/memtier/seed.iso`;
  启动脚本 `/memtier/boot.sh`,QMP `/memtier/qmp.sock`,压测经 hostfwd ssh 进 guest(ubuntu/redis123)。
- 驱动:`/memtier/driver.py`(memtier:QMP 控分层 + 等 floor 收敛)、`/memtier/hostswap.py`
  (cgroup MemoryHigh 控传统 swap);结果累积到 `/memtier/results.csv`。
- **E5-26xx v3 注意**:memtier enable 后 resident 会先快速换出 ~650 MiB 即停滞 ~90 s,待 write-aging
  积累 30 窗口才深度换出到 floor——settle 必须等穿这段停滞,否则会误测在浅压缩状态。
