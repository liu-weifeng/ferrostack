# drive-mirror 跳过空洞与厚制备盘未写区域：现状核实与方案

状态：设计稿，2026-10-21。基于 qemu `stable-11.1-dev` 血统（`block/mirror.c`、
`block/file-posix.c`、`blockdev.c`）阅读 + 10.8.1.196 上 XFS 实测 + 内核 `fs/ocfs2` 源码核对。
实验脚本：`/root/mirror-test.sh`、`/root/mirror-test2.sh`（196 上）。

## 0. 结论

1. **现有 QEMU（11.1 血统）已经能跳过源盘的文件空洞和 OCFS2 厚制备 raw 的 unwritten
   extent，不需要改 QEMU 代码。** 判定来自 `file-posix` 的 `lseek(SEEK_DATA/SEEK_HOLE)`；
   OCFS2 内核把 unwritten extent 和空洞一样报成 hole（`fs/ocfs2/extent_map.c`
   `ocfs2_seek_data_hole_offset`：`is_data = (e_flags & OCFS2_EXT_UNWRITTEN) ? 0 : 1`）。
2. **跳过能否变成"零 IO"取决于目标盘是否被 QEMU 认定为"已经全零"。** 认定了（10.1 起的
   zero bitmap 机制）：零区一个字节不读不写，目标盘的 unwritten extent 原样保留，厚盘仍是
   厚盘；没认定：零区虽然不读，但每 16MB 发一次 write_zeroes 到目标盘，在 OCFS2 上退化为
   `PUNCH_HOLE + fallocate` 两次元数据操作（OCFS2 不支持 `FALLOC_FL_ZERO_RANGE`），若目标
   带 `discard=unmap` 则只打洞——厚盘被打成薄盘。
3. **方案（接口影响最小）**：只改 lictor 的 `storage_mirror_start`：厚制备目标时自己用
   `fallocate` 预建目标文件，调 `drive-mirror` 时 `mode=existing`、`unmap=false`，其余照旧。
   QMP 接口、ferrovirt → lictor 的 action 参数只多一个可选字段。QEMU 可选地加 2 行让
   `drive-mirror` 也接受 `target-is-zero`（省掉开工时对目标的全零探测），非必需。
4. 如果产品线 QEMU 是 `stable-10.0-dev` 血统：没有 zero bitmap，零区每 16MB 一次
   write_zeroes 不可避免，需要回合 10.1 的 4 个 mirror 提交（§5）。

## 1. 现状：mirror 怎么处理零区（11.1 血统）

### 1.1 源盘的判定

`mirror_iteration()` 对每段脏区调 `bdrv_co_block_status_above(source, …, BDRV_WANT_PRECISE)`，
到 `file-posix` 的 `raw_co_block_status()`：`find_allocation()` 用 `lseek(SEEK_DATA)` /
`lseek(SEEK_HOLE)` 判定，data → `BDRV_BLOCK_DATA`，hole → `BDRV_BLOCK_ZERO`；一个 extent
一次 lseek，`pnum` 给到 extent 末尾。随后：

| 判定 | mirror 动作 |
|---|---|
| `DATA` | `MIRROR_METHOD_COPY`：读源、写目标（≤1MB 一片） |
| `ZERO` | `MIRROR_METHOD_ZERO`：**不读源**；目标已知为零则跳过，否则 `blk_co_pwrite_zeroes(target)` |
| 既非 DATA 也非 ZERO | `MIRROR_METHOD_DISCARD`（只在有 backing 链时出现） |

ZERO 段不受 1MB 片限制，但受 mirror 一次处理窗口 `buf-size`（默认 16MB）限制，所以零区
按 16MB 一段处理。

内核侧：哪些区域会被报成 hole——

| 文件系统 | 真空洞 | fallocate 出来的 unwritten extent | 备注 |
|---|---|---|---|
| OCFS2 | hole | **hole**（只看 extent 标志） | `ocfs2_seek_data_hole_offset()` 不查 page cache |
| XFS / ext4 | hole | hole，**但 page cache 里有该范围的页时报 data** | `iomap_seek_data` 查 page cache |
| 块设备（LV/LUN） | 不支持 SEEK_HOLE | — | 一律 data，无法跳过 |

XFS 的 page cache 行为在 196 上实测到了（§4 的 B/A2 组）：主机上对源文件做过一次
`cat`/`cmp` 后，unwritten 区域全部被当成 data 拷了 4GB；`drop_caches` 后只拷 192MB。
客户机 `cache=none` 不会往 page cache 放页，但宿主机上的 `cp`、`md5sum`、备份代理会；
这只是多拷零、不影响正确性。OCFS2 没有这个问题。

### 1.2 目标盘的判定（决定零区是"跳过"还是"写零"）

`mirror_dirty_init()`（10.1 起）：

```
target_is_zero = 用户给的 target-is-zero
              || (drive-mirror 且 mode != existing 且 bdrv_has_zero_init(target))   # blockdev.c
if (!target_is_zero) target_is_zero = bdrv_co_is_all_zeroes(target)              # 自动探测
```

`bdrv_co_is_all_zeroes()` 走目标的 block status：整个文件是 hole/unwritten → 全零（允许文件头
≤64KB 的小段 data，会读出来比对）。所以 **用 `fallocate` 预建、一个字节没写过的 OCFS2 厚盘会被
自动认定为全零**，不需要 `target-is-zero`。

认定全零后 `zero_bitmap` 全置位：`mirror_co_zero()` 看到目标该段仍为零就直接返回，不发 IO；
目标上只会落 COPY 的数据段，unwritten extent 全部保留。

没认定全零：若目标能 `write_zeroes_with_unmap`（需要目标打开时带 `discard=unmap`）先全盘
打洞一遍再拷；否则整盘标脏，零区每 16MB 一次 `pwrite_zeroes`。到 `file-posix`：

| 目标文件系统 | `unmap=true` 且 `discard=unmap` | 其它 |
|---|---|---|
| XFS | `PUNCH_HOLE`（厚→薄） | `ZERO_RANGE`，保持 unwritten（厚） |
| OCFS2 | `PUNCH_HOLE`（厚→薄） | `ZERO_RANGE` 不支持 → `PUNCH_HOLE` + `fallocate(0)`，回到 unwritten（厚），两次元数据操作 |

### 1.3 lictor 现在怎么调

`lictor/vm/storage_migration.py::storage_mirror_start`：`mode=absolute-paths`（QEMU 自己建
目标文件，稀疏）、`sync=full`、`copy-mode=background`，不传 `unmap`（默认 true）。
QEMU 自建的 raw 文件 `has_zero_init` 为真 → `target_is_zero` 自动为真 → 现在的薄目标盘
已经是"零区零 IO"的最优路径，空洞和 unwritten 都被跳过。**缺的只是厚制备目标盘这条路**。

## 2. 方案

### 2.1 lictor（必改，接口只加一个可选参数）

`storage_mirror_start` 的 params 增加 `preallocation: "off" | "falloc"`（默认 `off` = 现有行为）：

```
falloc 时：
  1. qemu-img create -f raw -o preallocation=falloc dest size      # fallocate(0) → OCFS2 unwritten extent
     （size 取源 active 盘的虚拟大小；qcow2 目标不适用，仍走 off）
  2. 校验目标确实"读为零"：qemu-img map --output=json dest 只有一段且 zero=true、data=false；
     不满足（如 OCFS2 未开 unwritten 特性导致 qemu-img 退化成写零）→ 报错，不要静默走慢路径
  3. drive-mirror 参数：mode=existing，unmap=false，其余不变
```

- `mode=existing`：告诉 QEMU 别重建文件。此时 blockdev.c 不再从 `has_zero_init` 推断全零，
  改由 `bdrv_co_is_all_zeroes()` 探测，一个 fallocate 出来的文件探测结果为真（§1.2）。
- `unmap=false`：镜像期间客户机的 discard / 写零不要在厚目标上打洞；OCFS2 上会变成
  `PUNCH_HOLE + fallocate` 回到 unwritten，厚度不变。
- 其它参数、READY 等待、pause/complete/cancel 流程全部不动。

### 2.2 ferrovirt（透传一个字段）

`snapshot_migrate.rs` 调 `storage_mirror_start` 时按目的数据存储类型带 `preallocation`：
目的盘要求厚制备（OCFS2 共享存储上的 raw 盘、或 VM 盘本来就是厚盘）→ `falloc`，否则省略。
来源可用现有 datastore 的 `fs_type`（`support/datastore/local.py` 已区分 xfs/ocfs2）或盘的
制备属性，不新增 REST 字段。

### 2.3 QEMU（可选，2 行）

`drive-mirror` 的 QAPI 结构 `DriveMirror` 加 `'*target-is-zero': 'bool'`，`qmp_drive_mirror()`
里 `target_is_zero |= arg->target_is_zero`（`blockdev-mirror` 已有此字段）。作用只是跳过开工时
的全零探测（探测对 OCFS2 是一次 `lseek`，可忽略），以及给"目标盘不是整文件 unwritten 但我们
知道它是零"的场景一个出口。不做也成立，建议先不做。

### 2.4 不建议的做法

- 不要用 `blockdev-add + blockdev-mirror` 替换 `drive-mirror`：目标要先 `blockdev-add`、
  `node-name` 管理、cancel 时还要 `blockdev-del`，lictor 四个 action 都要动，收益只是显式
  `target-is-zero`。
- 不要在 QEMU 里加"跳过 unwritten"的新旋钮：内核已经把 unwritten 报成 hole，QEMU 看不出
  也不需要看出两者的区别。
- 不要靠目标 `detect-zeroes=unmap`：那是把源的零数据在目标上打洞，和厚制备相反。

## 3. 边界与风险

1. **源盘必须是文件系统上的文件**。块设备源（LV、LUN 直通）不支持 SEEK_HOLE，一律当 data，
   全盘拷贝；这类盘要省流量只能靠 `detect-zeroes`（读出来发现是零再跳）——每个字节都要读。
2. **XFS/ext4 的 page cache 效应**（§1.1）：迁移前宿主机上对源盘做过整盘读的，unwritten 区域
   会被当 data 拷一遍，拷的是零、结果正确，只是慢。OCFS2 无此问题。必要时迁移前
   `echo 1 > /proc/sys/vm/drop_caches` 或对源文件 `posix_fadvise(DONTNEED)`。
3. **目标文件头**：预建后任何写入（哪怕 4KB）都会让自动探测失败（`bdrv_co_is_all_zeroes` 只
   容忍 ≤64KB 的头部 data 且读出来必须是零）。§2.1 第 2 步的 `qemu-img map` 校验就是守这条。
4. **粒度**：mirror `granularity` 默认 64KB，小于一个 granularity 的空洞并入相邻数据段拷贝。
5. **进度显示**：`query-block-jobs` 的 `offset/len` 仍按全盘算，零段瞬间推进，READY 会比
   "数据量 / 带宽"估算来得早，属正常。
6. **OCFS2 reflink 快照**：reflink 共享的 extent 带 `OCFS2_EXT_REFCOUNTED`，SEEK 判为 data，
   照常拷贝；本方案只对 active 盘做 mirror，历史快照链走 `snapshot_chain_migrate`，不受影响。

## 4. 实测（10.8.1.196，XFS `-o discard`，qemu 11.1 血统 `17fa8e0482`）

源 `src.raw`：`fallocate 4G` + `truncate 5G`，在 0 / 1G / 3G 各写 64MB 随机数据，其余为
unwritten（3904MB）和尾部真空洞（1GB）。目标均为 `fallocate -l 5G` 的厚盘。统计用 strace 的
系统调用计数（aio=threads，每次 preadv/pwritev 1MB）和 `filefrag -v` 的 extent 状态。

| 组 | 目标状态 | 源 page cache | 读 / 写（1MB 次） | fallocate | 目标结果 |
|---|---|---|---|---|---|
| A3 | 全 unwritten（认定全零） | 已丢弃 | **192 / 192** | **0** | 5120MB 仍全分配，3 段 unwritten 保留，内容一致 |
| B2 | 2G 处写过 1MB（未认定全零） | 已丢弃 | 192 / 192 | **308**（ZERO_RANGE，16MB 一次） | 仍厚，12 段 unwritten |
| A2 | 全 unwritten | **源被 `cat` 过** | 4096 / 4096 | 0 | 仍厚，但 unwritten 被当 data 拷了 4GB |

A3 就是方案的目标形态：只搬 192MB 数据，耗时 0.55s，零区零 IO，目标厚度不变。B2 对应
"目标没被认定为零"：XFS 上退化为 308 次 `ZERO_RANGE`，换成 OCFS2 则是 616 次打洞加回填。

## 5. 若产品 QEMU 是 10.0 血统

`stable-10.0-dev` 的 `block/mirror.c` 没有 `zero_bitmap`：源零区仍不读，但目标上每 16MB 一次
`pwrite_zeroes` 不可避免（OCFS2：`PUNCH_HOLE + fallocate`）。要达到"零 IO"需回合 10.1 的：

```
253b43a290 mirror: Drop redundant zero_target parameter
181a63667a mirror: Skip pre-zeroing destination if it is already zero
7e277545b9 mirror: Skip writing zeroes when target is already zero
aff46b4bf5 mirror: Reduce I/O when destination is detect-zeroes:unmap
4a7b1bd18d block/mirror: check range when setting zero bitmap for sync write   # 后续修复
```

它们只动 `block/mirror.c`、`blockdev.c` 和 QAPI 的 `blockdev-mirror`（新增 `target-is-zero`），
不改 `drive-mirror` 接口，lictor 侧方案不变。
