# memtier swap-cache 模型设计

> 目标:消除"刚换入的干净页被换出时重复写盘"的浪费,让换入簇预读真正有收益、并整体逼近
> 内核 swap 吞吐。背景见 `docs/memtier-bottleneck-analysis.md` §6(簇预读单独上反而慢 4×,
> 根因是 memtier 对投机/换入页"双重付费":读进来→变 RESIDENT→为守 floor 又写回)。
>
> **核心思想(照搬内核 swap-cache)**:页换入后**保留其 backing slot**,只要页未被写过,盘上副本
> 就一直有效;再次换出该干净页时**只 `MADV_DONTNEED` 复用原 slot,不写盘**(零 IO,等价于内核
> 丢弃干净 swap-cache 页)。页被写过才作废 slot、转脏、按原路写回。

---

## 1. 状态机

现有状态(`memtier-int.h:pstate_t`):`RESIDENT / EVICTING / EVICTING_DIRTIED / SWAPPED / FAULTING_IN`。

**新增** `RESIDENT_CLEAN`(枚举值 5,3-bit state 容得下):

| 状态 | 含义 | 页保护 | slot |
|---|---|---|---|
| `RESIDENT` | 驻留、可写、无有效盘上副本(脏或从未换出) | 可写 | 无 |
| `RESIDENT_CLEAN` | 驻留、**只读(uffd-WP)**、盘上 slot 仍是有效副本 | **WP** | 有效(保留) |
| `EVICTING` | 换出写盘中(WP) | WP | 已分配(写入中) |
| `EVICTING_DIRTIED` | 换出途中被写,将回滚 | 解 WP 中 | 将释放 |
| `SWAPPED` | 已 DONTNEED,盘上 slot 有效 | 不在场 | 有效 |
| `FAULTING_IN` | 换入读盘中 | 不在场 | 读取中 |

### 关键转移

```
SWAPPED --(guest MISSING fault / pmemsave 读)--> FAULTING_IN --读回--> RESIDENT_CLEAN  (保留 slot, 加 WP)
RESIDENT_CLEAN --(guest 写 → uffd WP-fault)--> RESIDENT                                (释放 slot, 解 WP)
RESIDENT_CLEAN --(worker 冷换出: 干净快路径)--> SWAPPED                                 (DONTNEED, 复用原 slot, 0 IO)
RESIDENT       --(worker 冷换出: 脏慢路径)--> EVICTING --写盘--> SWAPPED                 (新 slot, 写回)
```

**对比原版**:原版换入即 `slot_free` → 转 `RESIDENT`(可写,无盘副本),再换出必走写盘。
新版换入转 `RESIDENT_CLEAN`(WP,保留 slot),再换出若仍干净则免写盘。

---

## 2. 各路径改动

### 2.1 换入(`fault.c:memtier_swapin_blocking`)
- 读回页后 `UFFDIO_COPY` 时**带 `UFFDIO_COPY_MODE_WP`**(一步把页装成写保护),或 COPY 后再
  `uffd_change_protection(WP=true)`。
- **不** `slot_free`;`meta.state = RESIDENT_CLEAN`,`meta.slot` 保持指向原 slot。
- 统计:`swapped_pages--`、`resident_pages++`(页确实回到 RAM);新增 `clean_resident_pages++`
  (观测用)。`free_slots` 不变(slot 仍占用)。

### 2.2 写时转脏(`fault.c:handle_fault` WP 分支)
- WP-fault 且 `state == RESIDENT_CLEAN`:`slot_free(meta.slot, 1)`(盘上副本作废)、解 WP、
  `state = RESIDENT`、`clean_resident_pages--`。让 vCPU 继续(它现在能写了)。
- WP-fault 且 `state == EVICTING`:维持原逻辑(→ EVICTING_DIRTIED)。

### 2.3 干净页免写回换出(`worker.c`)
- `page_evictable`:候选条件从 `state==RESIDENT` 扩到 `RESIDENT || RESIDENT_CLEAN`(都遵守 cooldown)。
- 换出执行:对 `RESIDENT_CLEAN` 页走**干净快路径**——在 shard 锁内确认仍是 `RESIDENT_CLEAN`
  (页一直 WP,guest 不可能在锁外把它写脏而不经 WP-fault)→ `state = SWAPPED`(slot 不变)→ 锁外
  `MADV_DONTNEED`。**不 WP(已 WP)、不写盘、不分配 slot**。`evicted_pages++`、`clean_resident_pages--`、
  `resident_pages--`、`swapped_pages++`。
- 对 `RESIDENT`(脏)页:维持原 `memtier_evict_run`(分配新 slot、WP 整段、一次大写、完成判脏)。

### 2.4 换出 run 的碎片问题
原 `memtier_evict_run` 对连续冷段一次大写。现在冷段里会**混有干净页(免写)和脏页(要写)**。
方案:`evict_cell_runs` carve 连续候选段时,**先把段内 `RESIDENT_CLEAN` 页逐个/批量走干净快路径
(DONTNEED)剔除**,剩下的脏页子段再走 `memtier_evict_run` 大写。即"干净页先零成本回收,脏页再
按原连续写"。实现上:遍历候选段,把它切成「干净子段(DONTNEED 批)」与「脏子段(evict_run)」交替处理。

---

## 3. 并发与正确性(逐项)

| 竞态 | 处理 |
|---|---|
| guest 写 `RESIDENT_CLEAN` ↔ worker 干净换出 | 页恒 WP ⇒ guest 写必先 WP-fault(取 shard 锁转 RESIDENT)。worker 在 shard 锁内见 `RESIDENT_CLEAN` 才 DONTNEED;两者锁互斥。worker 转 SWAPPED 后,guest 再访问走 MISSING-fault 正常换入。**关键不变量:DONTNEED 只在页 WP 时做,故无未落的写丢失。** |
| DONTNEED 后、guest 立刻访问 | 页转 SWAPPED(锁内)后才 DONTNEED(锁外)。这窗口内页仍 present+WP:读不缺页(读到旧内容——但内容==盘上副本,正确);写 → WP-fault,handler 见 state 已 `SWAPPED` → 需处理:WP-fault on SWAPPED 视作"已被换出",转 MISSING 语义不成立。**收口**:DONTNEED 必须在 state→SWAPPED 的**同一持锁段**完成,或换出路径对干净页采用「先 DONTNEED 再置 SWAPPED」并持锁覆盖。实现时干净换出整体持 shard 锁完成 DONTNEED(单页 4K,DONTNEED 很快),消除窗口。 |
| 换入 COPY+WP 的原子性 | `UFFDIO_COPY_MODE_WP` 一次装成 WP,无"先可写后补 WP"的窗口。若内核不支持该 mode 则 COPY(DONTWAKE)+change_protection(WP)+WAKE,确保唤醒前已 WP。 |
| 迁移 quiesce(`worker.c:worker_quiesce_for_migration`) | 现只认 EVICTING/EVICTING_DIRTIED。新增:`RESIDENT_CLEAN` → `slot_free` + 解 WP + `RESIDENT`(让该页按普通脏页参与迁移传输,盘上副本作废)。保证 quiesce 后只剩 RESIDENT/SWAPPED。 |
| backing 共享迁移 re-dirty | 干净页换出不产生新 slot、不上线;换入(fault.c)对 migrating 仍 re-dirty(已有逻辑)。无需改。 |

---

## 4. slot 容量与回收

- **代价**:干净驻留页也占 slot ⇒ 同时占 slot 的页数 ≤ 全部曾换出过的页。**backing 需 ≥ 客户机
  RAM**(原来只需 ≥ 最大换出量)。文档与编排需提示这条。
- **slot 紧张兜底**:`slot_alloc_extent` 返回 0(脏页换出拿不到 slot)时,先**回收一批
  `RESIDENT_CLEAN` 页的 slot**:把它们标脏(`state=RESIDENT` + `slot_free`,内容已在 RAM,不丢),
  腾出 slot 给脏页写回。简单实现:扫描 meta 找 N 个 RESIDENT_CLEAN free 掉。v1 可先只在 backing-full
  事件里做一轮;benchmark 场景 backing(8G)> 客户机 RAM(6G)不会触发。

---

## 5. 统计

`tier_stats_t` 新增 `clean_resident_pages`(当前 RESIDENT_CLEAN 数)。`MemTierInfo` 增字段便于观测
"免写回换出"占比。`evicted_pages` 干净/脏都计;可加 `evict_clean_pages` 区分零 IO 换出量。

---

## 6. 实现步骤(每步独立编译 + 完整性测试)

1. **状态 + 数据结构**:加 `RESIDENT_CLEAN`、`clean_resident_pages` 统计;不改行为(swap-in 仍
   free slot + RESIDENT)。编译过。
2. **换入转 RESIDENT_CLEAN + WP + 保留 slot**;WP-fault 加 `RESIDENT_CLEAN→RESIDENT` 转脏分支。
   此时换出尚未识别 RESIDENT_CLEAN(它不在 `page_evictable`)→ 干净页**不会被再换出**(只读留驻),
   功能正确但 floor 可能压不动干净页。**先验证完整性**(pmemsave 往返 + redis 校验)。
3. **干净页免写回换出**:`page_evictable` + `evict_cell_runs` 干净快路径。验证 floor 能压下去 +
   完整性(干净页 DONTNEED 后 swap-in 内容一致)。
4. **迁移 quiesce 处理 RESIDENT_CLEAN** + slot-full 回收干净 slot。
5. **(可选)叠加换入簇预读**:此时投机页未用→再换出零成本,预读净收益转正。
6. 每步后 `ninja` 编译 + 跑测试;全绿才进下一步。

---

## 7. 完整性测试(数据无损是第一要务)

### 7.1 QEMU 微测(pmemsave 往返,无需 guest 负载)
仿 `tests/stagec_shared_backing_migrate.py`:
1. `pmemsave` 捕获区域 REF。
2. enable memtier(aggressive, psi-trigger=0, 低 floor)→ 等区域换出(SWAPPED)。
3. `pmemsave` 区域(触发 swap-in → RESIDENT_CLEAN)→ md5 == REF。**验换入正确**。
4. 再压低 floor / 等一轮 → RESIDENT_CLEAN 干净换出(DONTNEED 复用 slot)。
5. `pmemsave` 再读(swap-in)→ md5 == REF。**验干净再换出无损**。
   新增 `tests/swapcache_integrity.py`。

### 7.2 redis 端到端数据校验(覆盖脏路径)
确定性值:`value(key) = sha256(key)` 截断到 1000B。脚本(host 经 redis-cli / guest 内):
1. SET N 个键,值 = `h1(key)`。
2. 压 memtier 低 floor 强制换出。
3. GET 全部,校验 == `h1(key)`。**验 clean swap-in**。
4. SET 全部覆盖,值 = `h2(key)`(写 → WP-fault → 转脏)。
5. 再强制换出(脏写回)。
6. GET 全部,校验 == `h2(key)`。**验脏写回 + 再 swap-in**。
   任一不符即数据损坏 → 立即停。新增 `tests/redis_integrity.sh`(在 123 redis VM 上跑)。

### 7.3 复测性能
完整性全绿后,在 123 重跑 `docs/memtier-redis-benchmark.md` 的 NVMe 超分扫描(1.5–9×),对比:
原版 memtier / swap-cache / swap-cache+预读 / 传统 host-swap。预期 swap-cache(尤其 +预读)显著
缩小与 host-swap 的差距。

---

## 8. 风险与回退

- 一个并发 bug = 客户机内存损坏。**每步必须先过 7.1 + 7.2 完整性测试再继续**。
- 若某步完整性不过且短时定位不了,**回退该步**,保留已验证的前序步骤。
- backing < 客户机 RAM 的部署需走 §4 slot 回收;v1 先在 backing≥RAM 下验证。

---

## 9. 实测结果(2026-06-12,123 / E5-2673 v3)—— 步骤 1-2 验证通过,步骤 3 触硬件墙

按 §6 分步实现 + 每步 redis 确定性值完整性测试(`tests/redis_integrity.sh` + `redis_integrity_guest.py`:
SET 确定性值 → 强制换出 → GET 校验 → 覆写转脏 → 再换出 → GET 校验)。

| 步骤 | 内容 | 结果 |
|---|---|---|
| 1 | 加 `RESIDENT_CLEAN` 状态 + 统计(不改行为) | 编译通过 |
| 2 | 换入 COPY+WP+保留 slot→`RESIDENT_CLEAN`;WP-fault 转脏释放 slot;迁移 quiesce 处理 | **完整性全过**(clean swap-in + 脏写回 + 拉回三次校验 OK,换出 2GB) |
| 3 | 干净页免写回换出(`page_evictable`+evict 快路径:DONTNEED 复用 slot 零 IO) | **失败:guest soft-lockup** |

### 步骤 3 失败定位

- **首个 bug(已修)**:`evict_clean_one` 在持 shard 自旋锁时调 `MADV_DONTNEED` → mmu_notifier 进
  KVM,与缺页 vCPU 争锁 → 死锁。修法:照搬脏页路径(`evict_complete_one`),**解锁后**再 DONTNEED
  (靠 guest store 指令 WP-fault 被 park、worker 抢先 DONTNEED、store 经 MISSING 重试落地,无丢写)。
- **根本性的墙(未解)**:修死锁后仍 **guest soft-lockup**(serial:`CPU#0 stuck 53s` + RCU stall,
  redis 楔死且 disable 也救不回)。根因:
  - 123 是 **E5-26xx v3,page_idle 失效 → memtier 走 write-aging(只识别写热度)**。redis 的**读热**
    工作集(被反复 GET 但不写)在 write-aging 眼里是**冷的** → 被换出。
  - swap-cache 让换出**变便宜**(零 IO),于是 worker 在 `psi-trigger=0` 强制 floor 下**疯狂过度换出**
    读工作集(evicted 飙到 3-4M 页),DONTNEED/EPT-失效风暴 → vCPU 饿死 soft-lockup。
  - **加换出节流**(每 128 页 yield 300us,`EVICT_YIELD_*`)能缓解但**不能根治**:floor=450(对
    ~5GB 驻留约 11x)下仍 soft-lockup。
- **对比内核 swap 为何不死**:内核按 **LRU + 需求驱动**回收最冷页、**保住读热工作集**,且回收速率由
  实际分配压力自然节流;memtier 在读热盲的 CPU 上做不到——它按固定 floor + psi=0 无差别地把整个
  region 压到下限,误换读工作集 → 抖动,cheap 换出把抖动放大成 soft-lockup。

### 结论

- **swap-cache 换入(步骤 1-2)正确、数据无损、已验证**,是有价值的基础设施。
- **干净页免写回换出(步骤 3)在本硬件上不可用**:它本身数据语义正确,但与"读热盲(write-aging)
  + psi=0 强制深 floor"叠加会把 guest 抖死。**便宜的换出放大了误换读工作集的根本问题**。
- **真正的前置条件(本轮未具备)**:① **有效的读热度跟踪**(page_idle 在 Skylake+ 才正常,123 的
  Haswell-EP 不行)让 worker 只换真冷页、保住读工作集;② **需求/LRU 驱动 + 压力自适应的回收节流**
  (而非固定 floor + psi=0 的无差别下压)。两者齐备后,swap-cache + 簇预读才能在不抖死 guest 的
  前提下逼近内核 swap。
- **已回退步骤 3**(soft-lockup 不可用);源码回到 known-good。步骤 1-2 的设计与步骤 3 的失败教训
  保留在本文档。完整性测试脚本 `tests/redis_integrity*.{sh,py}` 保留供将来在合适硬件上继续。

> 教训叠加:之前发现"簇预读单独上反而慢(双重付费)";本轮发现"修了双重付费(swap-cache)后,
> cheap 换出在读热盲 CPU 上又把抖动放大成 soft-lockup"。**memtier 在 E5-26xx v3 上对读密集随机
> 负载做深超分的根本障碍是读热度跟踪缺失**,这是任何换出/换入优化都绕不过的前提。

---

## 10. 后续方向(按优先级)

本轮把障碍定位到**读热度跟踪缺失**(不是换出/换入机制)。要让 swap-cache + 簇预读真正落地、
不抖死 guest,按以下顺序推进:

1. **先解决读热度(根本前提,最高优先级)**。三条路任选/并行:
   - **换 page_idle 正常的 CPU 复跑**:Skylake+ 上 `page_idle` 的 EPT 空闲跟踪有效,memtier 能识别
     读热、只换真冷页。把本仓**已验证的完整性测试**(`tests/redis_integrity*.{sh,py}`)+ swap-cache
     步骤 1-3 直接在这类机器上重跑——预期不再 soft-lockup,且能量化 swap-cache + 预读相对内核 swap
     的收益。**这是验证 swap-cache 价值最直接的路径**。
   - **page_idle 失效时的替代读热度(让 123/Haswell-EP 也能用)**:① **KVM dirty-log 反推**——
     KVM_GET_DIRTY_LOG 给写热;读热可借 EPT A-bit(若 PML/access-tracking 可用)或周期性
     KVM_CLEAR_DIRTY + 访问采样;② **DAMON**(内核 6.x 的 region 级访问采样,不依赖 page_idle 的
     per-page 位图);③ uffd-WP 写老化已有,缺的是读老化——可对采样子集临时去映射(MADV_COLD/
     access-bit 采样)探测读访问。目标:`scan.c` 在这类 CPU 上也能产出"真冷"coldmap。
2. **回收改需求/LRU 驱动 + 压力自适应**(消除 psi=0 强制 floor 的无差别下压):
   - 不再"resident>floor 就无脑换到 floor",而是**按 PSI/分配压力梯度回收**(像内核 reclaim:压力大
     才多收、压力小就停),并优先换 recall 环/heat 最冷的页。`psi-trigger>0` 的生产模式本就是这个意思,
     但 benchmark 要测固定超分比又需要 psi=0——需要一个"既能压到目标 floor、又不饿死 guest"的折中:
     如**带最小驻留时间的 LRU 队列**(页换入后至少驻留 T 才可换),把抖动频率压到 guest 能存活。
   - 本轮的 `EVICT_YIELD_*` 节流是必要但不充分的安全网,应保留并与上面的 LRU/压力节流叠加。
3. **swap-cache 步骤 4-6(前两项就绪后再做)**:slot-full 回收干净 slot(§4)→ 叠加换入簇预读
   (此时投机页未用→零成本再换出,预读净收益转正,§6.5)→ 复测对比(§7.3)。

### 当前可直接复用的资产
- swap-cache 步骤 1-2 的实现思路(本文档 §1-3)+ 已验证的并发正确性(WP+保留 slot、解锁后 DONTNEED)。
- 完整性测试脚手架 `qemu/memtier/tests/redis_integrity{.sh,_guest.py}`(确定性值 SET/换出/GET 校验,
  覆盖 clean swap-in + 脏写回),硬件无关,可直接在新 CPU 上跑。
- `EVICT_YIELD_*` 换出节流(防 soft-lockup 安全网)的设计。

> 一句话:**不要再优化换出/换入了,先把"哪些页是真冷"搞对**。读热度一旦可靠,swap-cache(免重复
> 写回)+ 簇预读(摊薄读延迟)+ 需求驱动回收(不误换热集)三件套才能合力把 memtier 在随机访问下
> 推到接近内核 swap;缺了读热度,任何机制层优化都会被"误换读工作集"的抖动吃掉。

---

## 11. 换 AMD EPYC 平台能否改善读热度?(评估 + 验证步骤)

**结论速览**:有希望但不能假定——改善与否取决于 **NPT 的 Accessed 位 + KVM/内核读热度链路是否真生效**,
不是 EPYC 品牌本身带来的。务必先实测再下结论。

### 障碍卡在哪一层(再明确)

memtier 读热度靠 `page_idle`:清页 Accessed 位 → 等窗口 → 回读是否被硬件重置。对**客户机 RAM**,访问
发生在**嵌套页表**(Intel EPT / AMD NPT)里,需内核 `mmu_notifier`(`clear_young`/`test_young`)把"清/查
A 位"桥接给 KVM,KVM 再操作 EPT/NPT 的 A 位。这是一条 **嵌套页表 A 位 → KVM → page_idle** 的软硬件
协同链路,**非 Intel 专有**。

### EPYC 评估

- **AMD EPYC 全系(Zen:Naples/Rome/Milan/Genoa/Bergamo)NPT 均支持 A/D 位**,硬件底子与现代 Intel EPT
  A/D 对等 → substrate 在,有希望。
- **但不能断定 EPYC 就修好**:
  1. **123 的 Haswell-EP 失败本身可疑**——EPT A/D 位自 Haswell(2013)就有。E5-26xx v3 上"present 页
     全显热"的根因很可能**不是 CPU 没能力,而是 KVM 配置 / 内核版本 / 该型号 errata** 让 `kvm_age_gfn`
     保守地总报 young。若根因在软件层,**换 AMD 品牌未必解决**。
  2. 真正决定因素:该机器 **NPT A/D 是否启用**(`kvm_amd npt=1` + A/D 支持)、且 **KVM young/age 跟踪
     在该内核上确实反映 NPT 访问**——与 Haswell 失败同一条软件路径。

### 验证步骤(EPYC 上花几分钟先测,别假设)

1. 起 memtier VM,guest 内对一块已知区域**只读**反复访问(热区),另一块**完全不碰**(冷区)。
2. `x-memtier-enable`(aggressive, psi-trigger=0)后看 `XVIRT_MEMTIER_TRACE` 的 `scan ... ncold=N`:
   - **ncold>0 且只换出冷区** → page_idle 读热度有效 → memtier 能保住读工作集 → swap-cache + 簇预读
     能落地、且可能追平/超过内核 swap 的前提成立。
   - **ncold≈0、present 页全显热**(同 123) → 即便 EPYC 这条链路也没生效,根因在 KVM/内核 → 回到 §10
     的替代读热度方案(DAMON / KVM dirty-log 反推)。
3. 若(a)成立:直接把已提交的完整性测试(`tests/redis_integrity*.{sh,py}`)+ swap-cache 步骤 1-3 在
   EPYC 上复跑,量化相对内核 swap 的收益。

### 排查命令速记(在 EPYC host 上)
```bash
# NPT + A/D 是否开
cat /sys/module/kvm_amd/parameters/npt          # 应为 Y/1
# 内核是否支持访问跟踪 / page_idle
ls /sys/kernel/mm/page_idle/bitmap              # 存在=内核开了 idle tracking
grep -i "access.*track\|ad bits\|npt" /var/log/dmesg* 2>/dev/null
# memtier 自带的读热度诊断:enable 后看 trace ncold
XVIRT_MEMTIER_TRACE=1 ...                        # scan ncold>0 => 读热度有效
```

> 一句话:**EPYC 提供了可能(NPT A/D),能否兑现取决于 KVM/内核读热度链路在该机器上是否真生效——
> 这要实测;读热度一旦生效,才是 memtier 在随机读负载上翻盘的关键前提。**
