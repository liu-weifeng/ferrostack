# 热度记录与主动换回设计

主动换回的唯一收益是**消灭缺页尾延迟**——在 guest 踩到换出的热页之前，趁主机空闲把它备回 RAM，
把 p99/p999 尖峰提前摊销到空闲时段。本设计给出热度快照的记录与按热度的主动换回，
并保证它在任何情况下都不会损害正确性（被动缺页永远兜底）。

并入完整设计：作为 §6.5（紧接三层监控），meta 扩展进 §4，QMP 旋钮进 §11。

---

## 0. 与当前实现的对齐（必读）

本设计原按「完整设计（含 DAMON + 三层漏斗 + 异步 swap-in）」写。落到当前 `memtier/` 代码
（分支 `memory-tiering`）有几处必须对齐，否则无法实现。**核心机制（epoch 桶 + 主动换回 +
滞回带 + 被动兜底）经核实可行**；以下为差异与改法：

1. **热度源尚不存在**：`cell_heat` 字段已分配但**从未写入**（`g_malloc0` 后无人更新），也没有
   DAMON / 三层漏斗 EWMA。故 §2 的 `evict_heat = cell heat 快照` 现在只会全记 0。
   → **v1 不依赖 `evict_heat`，纯靠 `evict_epoch`（换出近度）召回**（桶内不再排序）。`evict_heat`
   留作可选二级键，待真正把 page_idle 冷扫结果累成 cell 级 EWMA 后再启用。
2. **swap-in 当前是同步阻塞**：`memtier_swapin_blocking()` 在单一 fault 线程上 `io_uring_wait_cqe`
   + `uffd_copy_page`，**没有** §5 假设的 `submit_swapin`/`swapin_complete` 异步回调。
   → 主动换回**单开一个 swap-in ring + bounce 缓冲**（fault 线程的 `swapin_ring` 不可跨线程共用），
   控制器在自己的 ring 上做（阻塞或异步皆可），天然不与被动缺页争用、不饿死被动。
3. **失败兜底（新引入的正确性边角）**：控制器置 `SWAPPED→FAULTING_IN` 后若填充**失败**，那个看到
   FAULTING_IN 而 park 的 guest fault 会一直挂住（无 COPY 唤醒）。→ `active_swapback` 失败**必须把页
   恢复 `SWAPPED`** 并触发一次 `UFFDIO_WAKE`/让被动重试。见 §6。
4. **命名/驱动**：结构体是 `MemTierRegion`（非 `tier_region_t`）；无独立 monitor 线程/DAMON 聚合周期
   → `epoch++` 挂在 worker 循环或一个 100ms 定时检查上。`page_meta_t` 4B→8B 前需确认所有 `meta[]`
   访问都在 `LOCK_OF` 内（现状 `handle_fault`/`evict_complete_one` 都在锁内）。
5. **桶插入避免锁内 malloc**：换出已改为**大批量**，`evict_complete_one` 一拍插很多页；`GArray`
   append 在桶 spinlock 内可能 realloc。→ 预分配/批量插入，别在自旋锁内 malloc。
6. **定位**：主动换回是**延迟优化（消缺页尾延迟），非正确性/容量特性**——被动换回已兜底。默认
   `target==min`（关），作为 **Phase 2** 排在换出路径之后，低风险。

> **v1 已落地并实测**（QEMU `f11a9d3b9f`）：实现了 `evict_epoch` 快照 + 按近度召回 + 独立
> swap-back ring + 失败兜底 + 三个 QMP 旋钮（引擎钳制 `swapin-watermark < psi-trigger`）+
> `active-swapins` 统计 + **hyper-agent 总控回血接入**。
>
> **召回结构改为单一 ring（不是 §4 的 per-epoch 桶）**：批量换出器一个 ~100ms epoch 就能换出
> 数 GB，**per-epoch 桶（cap 8192）会溢出**——实测只召回了 1.7GB 中的 ~64MB。改为一个按 region
> 大小的 recall ring（append 换出 gfn、newest-first 走、`SWAPPED && epoch 龄 ≤
> RECALL_WINDOW_EPOCHS` 判有效），容量与单 epoch 换出速率解耦。**召回窗口加到 25.6s**
> （`RECALL_WINDOW_EPOCHS=256 × 100ms`）：6.4s 比外部总控的 ~5s 节拍还短，压力换出的页在总控
> 看到富余、下达回血前就老化掉了。
>
> 独立实测（4GB 换出到 ~750MB 后回血 target=3000）：`active-swapins` 流式升 160→453836
> （~1.77GB 召回回血，对比桶版 ~16K），常驻回升、换出下降，无抖动、滞回带生效、完整性 PASS。
> **总控回血整合实测**（dev 主机）：加压→总控换出（常驻 2322→1825）；释放→总控把 target
> 1832→3739、`active-swapins` 上升、常驻回升 = 总控驱动回血。`evict_heat`/cell EWMA 仍待实现
> （v1 纯近度）。窗口可经 `RECALL_WINDOW_EPOCHS` 调。

---

## 1. 设计原则

1. **热度是换出瞬间的死快照，不是持续量。** 页进 `SWAPPED` 后不在 RAM 里，DAMON A-bit / page_idle 全失效，
   没有访问事件可更新——任何「换出后衰减」都是凭空猜测。换出瞬间打一个快照存下，之后不动。
2. **真正反映「现在还热不热」的唯一可靠信号是 guest 来踩它**（被动换回）。主动换回只是用快照做有根据的预测，
   猜错由被动兜底。所以**快照不必精确，够区分新旧/冷温即可**，别为它的精度过度投入。
3. **近度 > 热度。** 最近被迫换出的页最可能是「其实还在用、被压力误伤」的，优先级最高。排序主键是换出近度，
   次键才是换出前热度。
4. **复用，不新造。** 热度源用 §6 三层漏斗里现成的粗 cell heat；执行路径用现成的 swap-in；
   并发用现成的 `FAULTING_IN` 状态 + 分片锁。本设计只新增「快照字段 + 召回索引 + 控制器」三样。
5. **换回是全局决策。** 单 VM 看「主机空了」是局部视角，那块空闲可能要给别的 VM。QEMU 只暴露机制
   （换回到 `target-resident`），是否回血、回多少由编排层（mx-manager）按全局空闲 + 优先级拍板。

---

## 2. 热度快照：记什么、何时记、来源

- **记什么**：`evict_heat`（换出前热度）+ `evict_epoch`（换出近度）。
- **何时记**：`evict_complete` 把页置 `SWAPPED` 的那一刻，一次写入，之后不变。
- **来源**：`evict_heat` = 该页所在 2MB 粗 cell 的 `heat`（EWMA）。粒度是 cell 级（同 cell 的页
  heat 相同），无妨——更细的区分交给 `evict_epoch`。
  > ⚠️ 现状（见 §0.1）：cell heat EWMA **尚未实现**（`cell_heat` 字段全 0）。**v1 跳过 `evict_heat`，
  > 只记 `evict_epoch`**，召回纯按近度；待把 page_idle 冷扫结果累成 cell EWMA 后再启用此二级键。
- **失效**：页被换回（主动或被动）置 `RESIDENT` 时，快照随之作废（不清零也行，靠状态判断）。

---

## 3. meta 扩展与 epoch 计数器

`page_meta_t` 从 4 字节扩到 8 字节：

```c
typedef struct {
    uint32_t state : 3;
    uint32_t slot  : 29;
    uint16_t evict_heat;     /* 换出瞬间的 cell heat 快照 */
    uint16_t evict_epoch;    /* 换出时的 epoch（粗时间戳，非 wall clock） */
} page_meta_t;               /* 8B/页 → 32GB guest = 64MB 元数据 */
```

> 内存敏感时改用侧表：meta 保持 4 字节，另用「只对 SWAPPED 页存在的、按 slot 索引的数组」存
> `{evict_heat, evict_epoch}`，内存只随实际换出量增长，代价是多一次查表。先用 8 字节，够了再换。

**epoch 计数器**：全局单调递增，由 monitor 线程**每个 DAMON 聚合周期（~100ms）+1**。
只表达相对新旧，不用绝对时间。16 位轮回不是问题——见 §4，召回顺序由桶位置决定，不靠比较原始 epoch 数值。

---

## 4. epoch 分桶召回索引

不维护全局有序堆（每次换出/换回都要 O(log n) 维护、还和分片锁打架）。主动换回是**低频、批量、不在关键路径**
的后台动作，「近似有序」足够。用 epoch 环形桶：

```c
#define NR_BUCKETS 64        /* 64 × 100ms ≈ 6.4s 的「近期换出」窗口 */
typedef struct {
    GArray   *gfns;          /* 该 epoch 换出的页 id 列表 */
    QemuSpin  lock;
} bucket_t;
/* tier_region_t 增加： bucket_t buckets[NR_BUCKETS]; uint64_t epoch; */

#define BUCKET_OF(r, ep)  (&(r)->buckets[(ep) % NR_BUCKETS])
```

- **换出时插入**（O(1)）：`evict_complete` 置 SWAPPED 后，把 gfn 追加到当前 epoch 桶。
- **召回时从最新 epoch 往回取** → 天然实现「换出近度优先」。
- **桶是软状态**：不一致不影响正确性（最坏换回顺序次优，被动兜底），故用独立桶锁、不和迁出共享严格锁。
- **老化即回收**：epoch 推进覆盖到某桶时，清空该桶（残留页留在 SSD，只能被动召回）——
  正是想要的：>6.4s 没被访问的页是真冷，不值得预取。窗口大小 = `NR_BUCKETS × 聚合周期`，可调。
- **陈旧条目**：一页可能换出（epoch 5）→ 换回 → 又换出（epoch 9），于是同时存在于桶 5 和桶 9。
  桶 5 里那条是陈旧的。召回时按下面规则过滤，不必主动去重。

**条目有效性**（召回时判定）：
```
有效 ⟺ meta[gfn].state == SWAPPED  &&  meta[gfn].evict_epoch == 本桶 epoch
```
`evict_epoch` 字段正是为这个去重而留——它让「重新换出」场景下旧桶条目被自动识别为陈旧而跳过+剔除。

桶内是否再按 `evict_heat` 排序：可选。通常 epoch 维度已够，桶内不排也行；要排就对小桶做一次插入排序。

---

## 5. 主动换回控制器

并进 §5 的 controller 线程，周期性低频调用。

### 双触发门

```c
void swapback_tick(tier_region_t *r)
{
    if (resident_bytes(r) >= r->cfg.target_resident) return;        /* target 门：够了 */
    if (host_psi_some_avg10() >= r->cfg.swapin_watermark) return;   /* PSI 门：别加压 */
    ...
}
```

- **target 门**：换回到 `target-resident-mb` 为止，不超。它与 `min-resident-mb` 夹出工作带——
  迁出不低于 min，主动换回不高于 target，中间只走被动。
- **PSI 门（滞回带核心）**：只在 `psi < swapin-watermark` 时换回，而 `swapin-watermark` 明显低于
  迁出的 `psi-trigger`。中间的死区谁都不动，**杜绝换回↔迁出在水位线附近振荡**（这是主动换回最危险的反噬）。

### 候选选择 + 执行 + 限速

```c
    uint64_t budget = r->cfg.max_swapin_bytes_per_tick;             /* 限速 */
    for (uint64_t ep = r->epoch; ep + NR_BUCKETS > r->epoch && budget; ep--) {
        bucket_t *b = BUCKET_OF(r, ep);
        for (each gfn in b->gfns) {
            if (!budget) break;
            if (passive_swapin_pending(r)) return;   /* 让路给被动缺页：它延迟敏感，优先 */

            qemu_spin_lock(LOCK_OF(r, gfn));
            bool valid = (r->meta[gfn].state == SWAPPED &&
                          r->meta[gfn].evict_epoch == (uint16_t)ep);
            qemu_spin_unlock(LOCK_OF(r, gfn));
            if (!valid) { bucket_mark_dead(b, gfn); continue; }     /* 剔除陈旧条目 */

            active_swapback(r, gfn);
            budget -= PAGE_SIZE;
        }
    }
}

/* 选中一页主动换回：复用 swap-in I/O，只是由控制器发起、无 guest fault 驱动 */
static void active_swapback(tier_region_t *r, uint64_t gfn)
{
    QemuSpin *L = LOCK_OF(r, gfn);
    qemu_spin_lock(L);
    if (r->meta[gfn].state != SWAPPED) { qemu_spin_unlock(L); return; }  /* 已被换回/正在换 */
    r->meta[gfn].state = FAULTING_IN;            /* 占位，挡并发迁出/重复换回 */
    uint32_t slot = r->meta[gfn].slot;
    qemu_spin_unlock(L);
    submit_swapin(r, gfn, slot);                 /* 同被动：io_uring 读 → 回调 swapin_complete */
}
```

`UFFDIO_COPY` 对一个无人等待的缺页只是「预填」——正是 prefetch 语义，合法（已核实 fault handler
行为）。

> ⚠️ 现状（见 §0.2/§0.3）：当前 swap-in 是**同步阻塞**的 `memtier_swapin_blocking()`，没有
> `submit_swapin`/`swapin_complete` 异步拆分。落地时：(a) 主动换回**单开一个 swap-in ring + bounce**，
> 不复用 fault 线程的 `swapin_ring`（非多线程安全）；(b) 在该 ring 上调阻塞填充即可（低频 best-effort）；
> (c) **填充失败必须 `meta=SWAPPED` 复位 + `UFFDIO_WAKE`**，避免 park 的 guest fault 永挂。

### 停止条件

`budget` 用尽 / 触及 `target-resident` / PSI 升回水位 / 召回窗口扫完。下个 tick 再来。

### 优先级：绝不饿死被动缺页

主动换回是 best-effort，被动缺页是延迟关键。两条措施：
- `passive_swapin_pending()` 检查：有待处理的真实缺页就立即让路。
- 主动换回的在飞深度压在 swap-in ring 容量之下（或单开一个低优先级 ring），保证被动缺页随时有 SQE。

---

## 6. 并发：主动换回 × 被动缺页撞同一页

不需要新机制，现有 `FAULTING_IN` + 分片锁就够：

- 控制器在 `LOCK_OF(gfn)` 内把 `SWAPPED → FAULTING_IN`；若已非 SWAPPED 直接跳过。
- 若 guest 在控制器换回中途缺页同一页：fault handler 见 `FAULTING_IN` → 走「等填充方唤醒」分支。
  控制器的 `swapin_complete` 里那次 `UFFDIO_COPY` 落在同一地址，**会唤醒那个被 park 的 guest fault**——
  一次 COPY 同时完成了主动换回和被动缺页，无需特判。
- 反向（被动先把页置 FAULTING_IN）：控制器 `active_swapback` 见非 SWAPPED → 跳过。互不重复。
- **失败兜底（必须）**：主动换回置 FAULTING_IN 后，期间可能有 guest fault 看到 FAULTING_IN 而 park
  等唤醒。若主动填充**失败/取消**，**必须把页复位 `SWAPPED` 并 `UFFDIO_WAKE` 该地址**——让 park 的
  guest fault 重新走一次被动缺页，否则它永挂。（被动路径同线程同步填充无此问题，是主动换回新引入的。）

---

## 7. QMP 旋钮（加进 §11 的 MemTierTunables）

```qapi
##
# 追加到 MemTierTunables：
# @swapin-watermark:  主动换回触发的 PSI some avg10 上限 %。
#                     必须 < psi-trigger（否则与迁出振荡）；命令层校验并拒绝违反者。
# @max-swapin-mbps:   主动换回限速 MB/s。
# @target-resident-mb: 主动换回的目标常驻量上限。默认 = min-resident-mb（即不主动换回）。
##
```

- 默认 `target-resident-mb == min-resident-mb` → 主动换回默认关闭，纯被动，零反噬。
- **编排层「回血」动作 = `set-policy` 调高某 VM 的 `target-resident-mb`**：主机空闲 + 该 VM 优先级高时，
  把 target 抬高，控制器据此主动换回；全局吃紧时把 target 压回 min，主动换回自动停。
- 滞回带在 schema/命令层固化（`swapin-watermark < psi-trigger`），从接口上就堵死振荡配置。

预设建议：conservative/balanced 默认 `target == min`（不主动换回，省事省盘）；
延迟敏感档可设 `swapin-watermark = psi-trigger/4`、`max-swapin-mbps` 取被动峰值的一半。

---

## 8. 明确不做的事

- **不在换出后持续更新热度**：SSD 上的页没有访问事件，快照就该是死的。
- **不维护全局有序堆**：epoch 分桶的近似有序足够，且 O(1) 维护、不和分片锁打架。
- **不存 wall-clock 时间戳**：只需相对新旧，epoch 计数 + 桶位置即可。
- **不让单 VM 自行决定回血量**：那是编排层的全局决策。
- **不追求快照精度**：cell 级粗热度 + epoch 足矣，被动换回兜底，过度投入无意义。

---

## 9. 改动落点小结

> 注：结构体实际名为 `MemTierRegion`（非 `tier_region_t`）；下表按实际命名落地。

| 位置 | 改动 |
|---|---|
| `page_meta_t`（memtier-int.h） | 4B → 8B，加 `evict_epoch`（v1）；`evict_heat` 待 cell EWMA 落地再加。改前核实 `meta[]` 全在 `LOCK_OF` 内 |
| `MemTierRegion`（memtier-int.h） | 加 `bucket_t buckets[NR_BUCKETS]`、`uint64_t epoch`、独立 `swapback_ring`+bounce、换回配置 |
| worker/monitor（worker.c） | worker 循环或 100ms 定时 `epoch++`；epoch 推进时回收被覆盖的桶（无 DAMON 聚合周期，由此驱动） |
| `evict_complete_one`（worker.c，SWAPPED 分支） | 写 `evict_epoch` 快照 + `bucket_insert(当前 epoch, gfn)`；插入用预分配/批量，**勿在桶自旋锁内 malloc** |
| controller（worker.c/新线程） | 新增 `swapback_tick`（双门 + 召回 + 限速 + 让路） |
| swap-in 路径（fault.c） | 新增 `active_swapback`：**用独立 ring/bounce**（不复用 fault 线程 `swapin_ring`）；失败复位 `SWAPPED`+`UFFDIO_WAKE` |
| QMP（qapi/memtier.json） | `MemTierTunables` 加 3 个换回旋钮，命令层校验 `swapin-watermark < psi-trigger` |
| hyper-agent 总控（集成） | 模型 A 总控加下发 `target-resident-mb` + `swapin-watermark`，按全局空闲+优先级回血（见 memory-tiering.md §4） |

---

## 10. 一句话

热度只在换出瞬间冻结一次，主动换回按「换出近度（epoch 桶）优先、换出前热度次之」召回，
复用 swap-in 执行，靠 PSI 滞回带防振荡、`target-resident` 防过量、`FAULTING_IN` 防并发冲突，
被动缺页永远是正确性兜底——猜错了无非多走一次被动回读。
