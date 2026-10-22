# 内存分层虚拟机的内存快照：只保存常驻页，两种恢复方式（技术方案）

> 范围：只讲 QEMU（`memtier/` + `migration/`）与 snapshot-helper 两层的机制、数据格式与恢复路径。
> 编排（谁来拷文件、谁来选模式、API 字段）不在本文，见 [`memtier-snapshot-design.md`](./memtier-snapshot-design.md)。
>
> 代码现状：QEMU `memtier-stats` / `stable-11.1-dev` 血统已含 `memtier/migrate.c` 的全部机制
> （quiesce 延长到 `cont`：`e123c8a468`）；snapshot-helper `fa76dd3` 已实现本文两种恢复；
> 未做真机验证（2026-10-22）。

---

## 0. 一句话

开了分层的虚机做内存快照时，**只把仍在 DRAM 里的页写进快照文件**；已经换出到后端的页
不读回、不传输，只记录「哪个 gfn 在后端第几个 slot」。快照因此由两部分组成：

```
快照文件   <snap>.memory        常驻页 + 设备状态 + "memtier" 段（gfn→slot 映射）
内存交换文件 <snap>.memtier.raw  快照时刻后端文件的副本，slot 偏移与活动后端一致
```

恢复有两种方式：

| 方式 | 做法 | 恢复后 | 宿主需要的 RAM | 目标 QEMU |
|---|---|---|---|---|
| **一：两部分都导入** | helper 把交换文件里的换出页按映射读出，和常驻页一起当普通 RAM 页喂给 QEMU；映射段剪掉 | 全部页常驻，**不分层** | 全量客户机 RAM | 普通构建即可 |
| **二：只导入快照文件** | 交换文件放到映射里记的路径；QEMU 加载映射段，把换出页登记为 SWAPPED 并指向交换文件 | 与快照时刻完全一致，**启动即分层**，换出页按需从交换文件缺页读回 | 常驻量 | 必须带 memtier |

---

## 1. 页状态与它们在快照里的去向

memtier 的页状态（`memtier-int.h`）：

| 状态 | 含义 | 快照时 |
|---|---|---|
| `RESIDENT` | 在 DRAM，无 slot | 作为普通 RAM 页进快照文件 |
| `RESIDENT`（未映射） | 零页消除：换出时全零，直接 `MADV_DONTNEED`，不占 slot；首次访问零填 | 迁移读到的是零 → `RAM_SAVE_FLAG_ZERO`，快照文件里只置位图，不占空间 |
| `RESIDENT_CLEAN` | swap-cache：在 DRAM，但仍持有上次换出的 slot（内容未变） | 当普通 RAM 页进快照文件；它的 slot **不进映射**，恢复后该 slot 空闲 |
| `SWAPPED` | 不在 DRAM，内容在后端 `slot × 4K` | 只记 `(gfn, slot)` 进映射段；数据留在交换文件 |
| `EVICTING` / `EVICTING_DIRTIED` | 换出写在途 | quiesce 时回滚成 `RESIDENT`（撤 WP），按常驻页处理 |
| `SWAPPED_MOVING` | 压实搬运中 | quiesce 前完成；quiesce 期间不再压实 |

所以快照文件的大小 ≈ 常驻量，交换文件的引用集合 = 快照时刻所有 `SWAPPED` 页。

---

## 2. 保存流程（QEMU 内部）

触发条件：迁移能力 `memtier-shared-backing` 打开，然后对一个 fd/pipe 做 `migrate`。
这条路径本来是给「两端可见的共享后端」热迁移用的，内存快照原样复用，差别只是对端是文件。

```
migrate
  └ ram_init_bitmaps
      ├ memtier_migration_src_begin()        每个 region 置 migrating，等 worker 确认 quiesce
      │    worker: drain 在途换出；EVICTING* 回滚为 RESIDENT；migrate_quiesced=1
      │    此后 worker 每拍直接 continue：不换出、不主动换回、不打洞、不压实
      └ 对所有 SWAPPED 页清脏位                     ← 它们不会被发送
  precopy 轮次：只发脏的常驻页（feature-4：首轮先发冷页，热页留到后面）
      客户机缺页换入的页由 fault 线程标脏 → 仍会发送（状态 SWAPPED→RESIDENT_CLEAN）
  stop-and-copy（vCPU 已停）
      ├ ram SECTION_END：剩余脏页
      ├ 设备段 SECTION_FULL …
      └ "memtier" 段 SECTION_FULL（memtier_mig_save）   ← 在 ram 之后、EOF 之前
  完成：runstate=postmigrate
      memtier_migration_src_end()：VM 未在运行 → 释放推迟（release_pending）
  [调用方在这里拷贝后端文件：reflink / 导出]                ← 后端此时稳定
  cont → vm_change_state 回调 → memtier_mig_release()：worker 恢复工作
```

两个不变量由 QEMU 保证：

1. **`src_begin` 之后到 `cont` 之前，后端上任何被引用的 slot 不会被写、不会被释放、不会被搬。**
   换出被 quiesce；主动换回被 quiesce；dead chunk 的 discard 与压实都在 worker 主循环的
   `migrating` 分支之后，同样被跳过；swap-cache 的换入保留 slot，不 `slot_free`。
2. **映射段在 vCPU 停止后生成**，读到的 `meta[].state/slot` 不会再变。

快照文件与交换文件的一致性只依赖「拷贝后端发生在 postmigrate 窗口内」这一条。

### 2.1 "memtier" 段格式（版本 1）

```
段头   QEMU_VM_SECTION_FULL(0x04) | be32 section_id | u8 7 "memtier" | be32 instance_id=0 | be32 version_id=1
blob   be32 n_regions
       per region:
         be32 len + id            region id（"" = 默认 guest RAM 块）
         be32 len + path          后端路径（活动路径，不是副本路径）
         u8   policy
         be64 nr_pages
         be64 n_swapped
         n_swapped × { be64 gfn, be32 slot }
尾部   QEMU_VM_SECTION_FOOTER(0x7e) | be32 section_id
```

每个换出页 12 字节：8GB 客户机换出 50% 约 12.5MB，可接受。段里**不带 tunables**，只带 policy。

### 2.2 helper 保存侧

snapshot-helper 只解析 ram 段，ram 之后的字节作为 epilogue 原样落盘，所以映射段天然在
epilogue 里。保存结束后 helper 扫一遍 epilogue 找 `"\x07memtier"` 段头并完整解析 blob
（region 数、字符串长度、`nr_pages` 对得上某个 block、FOOTER 的 section_id 与段头一致），
四项都过才在索引里记：

```c
SnapHelpIndex.flags            |= SNAPHLP_F_MEMTIER
SnapHelpIndex.memtier_sect_off  = 段头的绝对文件偏移（落在 epilogue 内）
SnapHelpIndex.memtier_sect_len  = 段头 + blob + 尾部
SnapHelpIndex.memtier_pages     = 各 region n_swapped 之和
```

文件版本不变：老 helper 读新文件忽略这些字段，等价于方式二；新 helper 读老文件 `flags=0`。

### 2.3 交换文件

后端文件（`fallocate` 预建，无头、无元数据，`slot × 4K` 即偏移）在 postmigrate 窗口内整份
拷贝。支持 reflink 的文件系统（XFS reflink=1、OCFS2、btrfs）上 `cp --reflink=always` 即时完成、
初始 0 占用，之后活动后端被改写的 chunk 才 CoW；分块日志分配器把写集中在 2MB chunk 内，
CoW 粒度与之吻合。拷出来的文件里不被映射引用的区域（空闲 chunk、`RESIDENT_CLEAN` 的 slot、
已死但未 discard 的 chunk）是无用数据，不影响正确性。

后端是裸分区 / ublk 块设备时不能 reflink，见 §6.3。

---

## 3. 恢复方式一：两部分都导入（merge）

目标：恢复成一台**不分层**的普通虚机，所有页在 DRAM。目标 QEMU 不需要 memtier。

```
snapshot-helper <snap>.memory 0 --memtier=merge --memtier-backing=<snap>.memtier.raw
  │
  ├ 读索引、位图；校验 SNAPHLP_F_MEMTIER，解析映射段；交换文件大小必须 ≥ (max slot+1)×4K
  ├ 发 prologue（原样）
  ├ 发一个 ram SECTION_END：
  │     每个 block：先按位图发常驻页（全零→ZERO，否则 PAGE）
  │                 再按映射对每个 (gfn, slot)：pread(交换文件, slot×4K) → 全零→ZERO，否则 PAGE
  │     RAM_SAVE_FLAG_EOS + FOOTER
  └ 发 epilogue：[0, sect_off) 与 [sect_off+len, end) 两段，映射段剪掉
qemu -S -incoming …     普通 ram_load_precopy；不含 memtier 段，什么都不用知道
```

要点：

- 一个 gfn 不会同时出现在位图和映射里（快照时它要么 RESIDENT 要么 SWAPPED），不需要去重。
- `QEMU_VM_VMDESCRIPTION`（JSON）里仍列着 memtier，加载端对 vmdesc 只做提示不校验，无影响。
- 恢复后若再 `x-memtier-enable`，分层从零开始：后端重新分配，与交换文件无关。
- 读交换文件的 I/O 模式：按 gfn 顺序读 slot 是随机 4K 读。分块日志分配器下同一次换出的
  一段 gfn 落在连续 slot 上，局部性尚可；进一步优化是把映射按 slot 排序后顺序 `pread`，
  再按 gfn 发——helper 一次只持一页缓冲，排序只改读取顺序，流格式不变（可选，未做）。
- 时间 ≈（常驻量 + 换出量）/ 发送带宽；换出量部分受交换文件读带宽限制。
- 校验失败即拒绝：无映射段却给了 `merge`、页大小不是 4096、交换文件装不下最大 slot。

---

## 4. 恢复方式二：只导入快照文件（keep）

目标：恢复到**与快照时刻完全一致的分层状态**，启动即分层，宿主只付常驻量的 RAM。

```
交换文件放到映射段记的活动路径（同宿主同路径直接放回；不同路径见 4.2）
snapshot-helper <snap>.memory 0            （--memtier=keep 是默认，等价于不认识 memtier 的 helper）
  ├ prologue 原样
  ├ ram SECTION_END：只有常驻页（位图）
  └ epilogue 原样，含映射段
qemu -S -incoming …
  ├ ram_load_precopy：常驻页写进 RAM；没发的页保持未映射（读为零）
  └ 加载 "memtier" 段 → memtier_mig_load → memtier_dst_restore(id, path, policy, pairs)
        1. tier_check_support；按 id 找 RAMBlock；该 region 不得已启用
        2. tier_init：打开 path（O_DIRECT），建 uffd、io_uring、slot 分配器（nr_slots = 文件大小/4K）
        3. 整体校验：gfn < nr_pages；gfn 不重复；slot_reserve(slot) 成功（slot < nr_slots、未占用）
           任一失败 → error_setg、teardown、加载失败，不会带着错映射起机
        4. 每对：state=SWAPPED, meta.slot=slot
        5. 每对：MADV_DONTNEED(hva)        ← 丢掉 RAM 里的零页，首次访问走 MISSING 缺页
        6. resident_pages = nr_pages - n；enabled=1；起 fault 线程与 worker 线程
cont → 客户机第一次碰到换出页 → fault 线程从交换文件 slot×4K 读回 → UFFDIO_COPY
```

### 4.1 恢复后的状态

- 常驻集合、换出集合、每页 slot 与快照时刻一致；swap-cache 的 `RESIDENT_CLEAN` 页恢复为普通
  `RESIDENT`（slot 已在快照时放弃，再换出要重写一次）。
- 零页消除过的页：快照里是 ZERO，恢复后 RAM 中是零页（已映射）；再换出时再次被判零、丢弃。
- 分块日志分配器：被引用的 chunk 置 `CH_USED`（部分 live），其余 `CH_FREE`；没有 `CH_DEAD`
  需要 discard。后续换出从空闲 chunk 开始写日志，压实在空闲 chunk 降到保留线时才会搬动这些
  部分 live 的 chunk。交换文件里未被引用区域的旧数据会被自然覆盖。
- policy 来自映射段；tunables 不在流里，恢复后由调用方 `x-memtier-set-policy` 回放。
- 恢复后 QEMU 对交换文件是**读写**的（新的换出写进去），所以它必须是可写的独立副本，
  不能直接指向快照目录里那份（否则快照被破坏，不能再次恢复）。

### 4.2 交换文件在别的路径

映射段里的 `path` 是快照时的活动路径。恢复到别的宿主或别的目录时，helper 重写段内路径：

```
snapshot-helper <snap>.memory 0 --memtier=keep --memtier-path=/new/path/backing.raw
```

helper 用解析好的 blob 重新序列化该段（只改 `path` 字符串与长度前缀，pairs 原样拷贝），
epilogue 其余字节不动。QEMU 侧无改动。

### 4.3 前提与失败模式

| 条件 | 不满足时 |
|---|---|
| 目标 QEMU 带 memtier（非 `memtier-stubs` 构建） | stubs 不注册 `"memtier"` 段 → 加载报未知段失败。只能走方式一 |
| 交换文件 ≥ (max slot+1)×4K | `slot_reserve` 失败 → 加载拒绝（而不是运行期读越界） |
| 交换文件与快照文件来自同一次快照 | 内容对不上无法检测（后端无元数据）；靠调用方成对保管。可选：段版本 2 带后端大小 + 校验值，见 §6.1 |
| 宿主有 uffd / page_idle 权限（CAP_SYS_ADMIN） | 与 `x-memtier-enable` 同样的要求 |
| 后端为普通文件或块设备 | SPDK 后端的 path 语义不同，暂不支持方式二 |

### 4.4 可选的预热

方式二恢复后，所有换出页都靠缺页读回，刚起来的一段时间尾延迟高。两个现成手段，不需要新机制：

- `x-memtier-set-policy target-resident-mb=N`（配合 `swapback-watermark`）：worker 在宿主
  空闲时主动按「最近换出」顺序拉回，直到常驻量到 N。
- 调用方按 `evict-tick` 顺序无法预知，但分块日志的 slot 顺序 ≈ 换出时间顺序：映射按 slot 倒序
  即「最近换出在前」，可作为预热顺序的近似（需在 QEMU 侧加一个「按 slot 顺序换回」选项，未做）。

---

## 5. 两种方式对比

| | 方式一：两部分都导入 | 方式二：只导入快照文件 |
|---|---|---|
| 恢复后分层 | 否（可再 enable，从零开始） | 是，状态与快照时刻一致 |
| 宿主 RAM | 全量客户机 RAM | 常驻量 |
| 读交换文件 | 恢复时全部读完（随机 4K） | 按需，客户机访问才读 |
| 恢复耗时 | ∝ 常驻量 + 换出量 | ∝ 常驻量 |
| 启动后性能 | 立即全速 | 冷页首次访问有缺页延迟 |
| 目标 QEMU | 任意 | 带 memtier |
| 交换文件之后 | 不再需要 | 成为活动后端，被读写 |
| 跨宿主 | 直接 | 需交换文件可达 + 必要时改路径 |
| 实现位置 | 全在 helper | QEMU `memtier_dst_restore` + helper 原样透传 |

两种方式对同一份快照产物都适用，快照侧不做选择。

---

## 6. 边界与扩展

### 6.1 映射段版本 2（可选）

在版本 1 之后追加 `be64 backing_bytes`、`be64 backing_check`（例如被引用 slot 中前若干个的
内容哈希）。加载端核对后端大小与哈希，能抓住「交换文件拿错了」这类错误，而不是起机后读到错数据。
版本 1 的 helper 与 QEMU 互相兼容：QEMU 按 `version_id` 判断，helper 的解析器按版本取字段。

### 6.2 "全部换出"恢复（可选的第三种方式）

把方式二推到极端：helper 把常驻页也写进交换文件的空闲 chunk，映射段改成「全部页 SWAPPED」，
ram 段为空；恢复瞬间宿主几乎不占 RAM，客户机边跑边把工作集缺页读回。适合超分严重的宿主做
「先起来再说」。代价是恢复时要把常驻量写一遍后端（NVMe 磨损）、起机后一段时间延迟高。
QEMU 侧不需要改：`dst_restore` 对 n_swapped = nr_pages 同样成立。helper 侧需要在交换文件里
选 chunk（映射未引用的 chunk 即空闲），并读写交换文件。未实现。

### 6.3 后端是块设备（裸分区 / ublk 卷）

块设备不能 reflink，交换文件要靠「导出被引用 slot」生成：postmigrate 窗口内按映射对每个
slot `pread` 设备、`pwrite` 到同偏移的稀疏文件。两种实现位置：

- QEMU 命令 `x-memtier-export-backing {id, path}`：postmigrate 下才接受；SPDK 后端走
  `mt_spdk_read_sync`，是唯一能覆盖 SPDK 的办法。
- helper 子命令：读快照文件里的映射段，直接读设备写稀疏文件。不覆盖 SPDK。

恢复方式一不受影响（helper 的 merge 已接受块设备作 `--memtier-backing`）。方式二要把稀疏文件
按偏移写回设备（或改路径指向文件）。未实现。

### 6.4 reflink 下的 O_DIRECT 写

活动后端被 QEMU 以 O_DIRECT 写，reflink 之后首次写共享 extent 触发 CoW。XFS / OCFS2 / btrfs
的 directio CoW 路径正确但有额外开销（分配 + 元数据），分块日志让 CoW 按 2MB 粒度发生，
放大可控。需真机测量，尤其 OCFS2。

### 6.5 多 region

映射段按 region 记录，每个 region 一个后端路径、一份交换文件。helper 的 merge 当前只收一个
`--memtier-backing`，多 region 快照只能走方式二；扩展为按 region id 指定即可。

---

## 7. 验证方法

两种方式都用确定性值做字节级校验（复用 `memtier/tests/redis_integrity.sh` 与
`redis_integrity_guest.py` 的思路：`rig.py populate <tag> <n>` 写入可由 key 推出的值，
`verify <tag> <n> <sample>` 抽样比对）：

1. 客户机 8GB，aggressive 策略压到常驻 ≤ 30%，`x-memtier-query` 记 `swapped-bytes`。
2. 打快照：核对快照文件页区大小 ≈ `resident-bytes`，`--info` 的 `memtier_pages` ≈ `swapped-bytes/4K`。
3. 方式一恢复：`x-memtier-query` 无 region；`verify` 全部通过；恢复耗时与读交换文件字节数。
4. 方式二恢复：`x-memtier-query` 的 `swapped-bytes` 与快照时一致；`verify` 全部通过
   （触发大量缺页读回）；`swapin-errors = 0`；之后继续压测，确认新换出不破坏已引用 slot。
5. 方式二 + 改路径：交换文件放到另一目录恢复，同上。
6. 负面用例：交换文件截短 → 加载拒绝；无映射段的快照给 `merge` → helper 拒绝。
7. 快照后 `cont` 之前的窗口：故意延迟拷贝 30s，确认期间 `evict-ios` / `backing-discards` /
   `backing-compactions` 不增长。

selftest 已覆盖 helper 侧三种流的字节级往返（keep / keep+改路径 / merge）；QEMU 侧与端到端
在真机上未跑。
