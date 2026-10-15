# 内存分层下的内存快照设计（memtier × snapshot-helper）

> 目标：虚机开启内存分层（memtier）后，含内存快照**只导出仍在 DRAM 中的常驻页**，
> 快照产物 = 内存快照文件 + 内存分层文件；恢复时既能**直接回到分层状态**，也能
> **把分层文件中的页全部导入**后不分层运行。
>
> 现状问题：不做任何处理时，QMP `migrate` 读 SWAPPED 页会触发 uffd 缺页，把**全部换出页
> 拉回宿主 RAM**，快照耗时 ∝ 换出量，超分主机上可能直接 OOM。文档此前的建议是快照前
> `x-memtier-disable` 收敛，代价相同。
>
> 相关文档：[`memory-tiering.md`](./memory-tiering.md)（架构）、
> [`../qemu-memory-tiering-tech-design.md`](../qemu-memory-tiering-tech-design.md) §12（共享后备热迁移）、
> `../snapshot-helper/src/README.md`（快照文件格式）、`../virt-control/docs/snapshot.md`（快照总体流程）。
>
> 状态：设计稿（2026-10-15）。进度：§6.1 QEMU 两处必改已提交到 qemu `stable-11.1-dev`（待 112 构建机编译与真机验证），其余未动。

---

## 0. 结论先行

| 问题 | 结论 |
|---|---|
| 内存快照格式要改吗 | **不改版本**。QEMU 迁移流复用已有的 `"memtier"` savevm 段；snapshot-helper 文件保持 v1，只在 `flags` 占一位、在 `reserved[]` 里记 memtier 段的位置（向后兼容，老 helper 读新文件仍可按「继续分层」恢复）。 |
| snapshot-helper 要改吗 | **「继续分层」场景不用改**：常驻页自然落到页区，memtier 段随 epilogue 原样透传。**「全部导入」与「改路径」场景要改**：save 时定位 memtier 段并记入索引；restore 新增 `--memtier=keep\|merge`、`--memtier-backing=`、`--memtier-path=`。 |
| QEMU 要改吗 | 两处小改（必做）：换出 quiesce 延长到 `cont`；`memtier_dst_restore` 校验 slot 范围与重复。可选：导出命令（裸设备/petrel 后端）、段版本 2 带后端大小。 |
| 编排要改吗 | lictor 快照创建加「设能力 → reflink 后端 → 复位能力」；回滚加「后端放回 + 恢复模式选择」；`_memtier_enable_on_start` 幂等。控制面快照计划加分层文件项。 |

---

## 1. 可复用的现有机制

QEMU 分支（`stable-11.1-dev`，`memtier/migrate.c` + `migration/ram.c`）已经为共享后备热迁移实现了下面三件事，快照直接复用：

1. **只发常驻页**：迁移能力 `memtier-shared-backing` 打开后，`ram_init_bitmaps` 先调
   `memtier_migration_src_begin()` 让 worker quiesce（在途换出回滚，每页稳定为 RESIDENT 或
   SWAPPED），再把所有 SWAPPED 页的脏位清掉。迁移期间客户机缺页换入的页会自己重脏，仍会被发送。
2. **`"memtier"` savevm 段**：停机阶段、ram `SECTION_END` 之后，以 `SECTION_FULL` 发出
   每个 region 的 `{id, path, policy, nr_pages, n_swapped, [gfn:be64, slot:be32]…}`（版本 1）。
3. **目的端恢复**：`memtier_mig_load → memtier_dst_restore()`：打开 `path`，建 uffd 与 ring，
   对每对 (gfn, slot) 置 SWAPPED、`slot_reserve`、`MADV_DONTNEED`，起 fault/worker 线程。
   该段在 `migration_object_init` 无条件注册，目的端**不需要也不能**预先 enable。

snapshot-helper 侧：ITERATE 只处理 ram 的 `SECTION_PART/END`，之后所有字节（设备 `SECTION_FULL`、
`QEMU_VM_EOF`）作为 epilogue 原样捕获、restore 时原样回放。memtier 段恰好落在 epilogue 里。

后端文件（`<vm_dir>/memtier-backing.raw`，hyper-agent 按客户机内存大小 `fallocate`）本身无头、无
元数据，slot 号 × 4K = 文件偏移；gfn→slot 映射只存在于 QEMU 内存，即 memtier 段。

---

## 2. 快照产物

```
<snapshot_dir>/
  memory/<snap>.state          snapshot-helper v1：页区只含常驻页；epilogue 内含 memtier 段
  memory/<snap>.memtier.raw    快照时刻后端文件的副本，slot 偏移与活动后端完全一致
  disks/ config/ …             不变
```

| 文件 | 实际占用 |
|---|---|
| `.state` | ≈ 常驻量（页区稀疏，位图只标常驻页） |
| `.memtier.raw` | reflink 副本：初始 0，活动后端随后被改写的 slot 才 CoW；上限 = 后端大小 |

**核心不变量：后端副本保持与活动后端相同的 slot 偏移，memtier 段里的 `path` 仍是活动路径。**
恢复时把副本拷回活动路径即可，与磁盘的 reflink 回写同构，QEMU 不需要任何路径改写。

否决的替代方案：把换出页合并进 `.state` 页区做成单文件。能做（按 gfn→slot 从后端读入页区），
但快照拷贝量 ∝ 换出量，恢复时所有页常驻需要全量 RAM，违背分层目的。不过这个「合并」动作
放到**恢复时**做就是 §4 的场景 B，所以合并能力保留，只是时机后移。

---

## 3. 快照创建流程（lictor `_live_memory_snapshot`）

在现有 9 步流程上增加 3 处（★）：

```
 1. 写 libvirt 头 + VM 配置 + padding 到 memory_file_path
★1a. 读 sidecar / x-memtier-query：开了分层 →
     migrate-set-capabilities memtier-shared-backing=true
 2. pipe + fork snapshot-helper（stdin=pipe-r, stdout=memory file）
 3. getfd 送 pipe-w 进 QEMU
 4. migrate uri=fd:snap-mem-<sid>
 5. 等 MIGRATION/completed，run-state=postmigrate；等 helper exit 0
 6. 各盘 cp --reflink=always
★6a. cp --reflink=always <vm_dir>/memtier-backing.raw <snap>/memory/<snap>.memtier.raw
     （VM 已停、换出已 quiesce、后端稳定）
 7. flip magic → LibvirtQemudSave
 8. cont
★8a. migrate-set-capabilities memtier-shared-backing=false（finally 里保证执行）
```

要点：

- **能力必须复位**。迁移能力是 QEMU 进程级持久的；不复位，之后向其它宿主热迁移会跳过换出页，
  目的端若后端不可达则数据丢失。
- **quiesce 必须持续到 `cont`**（QEMU 改动，见 §6.1）。否则 worker 在 postmigrate 期间可能
  主动换回（`slot_free`）再换出复用该 slot，在 6a 之前覆盖快照引用的数据。
- feature-4 的「首轮只发冷页」随能力一并生效，对快照只影响发送顺序，无害。
- 后端不可 reflink（裸设备、SPDK、petrel 卷 `backing_kind=petrel`）时，v1 回退到既有的
  「先 `x-memtier-disable` 再快照」路径并在 API 返回中标明；v2 用 §6.1 的导出命令。
- paused 状态的 VM 走同一路径（QEMU 把 paused 上的 migrate 视为无脏页直接 stop-and-copy）。

---

## 4. 恢复场景矩阵

快照产物同时带内存文件和分层文件，因此**「继续分层」还是「全部导入」在恢复时再选**，
快照侧无分支。

| 场景 | 恢复后状态 | 实现 | 前提 |
|---|---|---|---|
| **A. 继续分层（默认）** | 与快照时刻一致：常驻页在 RAM，换出页按需从分层文件读回 | 后端副本放回活动路径 → `-incoming exec:snapshot-helper FILE 0` → QEMU 加载 memtier 段自行 `memtier_dst_restore` | 目标 QEMU 带 memtier（非 `memtier-stubs` 构建）；CAP_SYS_ADMIN |
| **B. 全部导入，不分层** | 所有页常驻，VM 不带分层 | helper restore `--memtier=merge --memtier-backing=<副本>`：按 gfn→slot 从分层文件读页，作为普通 RAM 页一起发出；memtier 段从 epilogue 剔除 | 宿主空闲内存 ≥ 客户机 RAM；目标 QEMU 可以是普通构建 |
| **B'. 全部导入后重新分层** | 先全常驻，worker 随后按冷热重新换出 | 同 B；lictor 启动后按 sidecar `x-memtier-enable`，现有代码已覆盖 | 同 B |
| **C. 继续分层，分层文件在别处** | 同 A | helper restore `--memtier=keep --memtier-path=NEW` 改写段内 `path` | 跨宿主 / 跨 datastore 导入 |
| **D. 继续分层并预热** | 同 A，近期换出页主动拉回 | A 之后用现有 `target-resident-mb` / `swapin-watermark` 主动换回 | 无新机制 |

B 的零改动兜底：先按 A 恢复再 `x-memtier-disable` 强制收敛。只适合「同宿主、带 memtier、
想退出分层」，不能替代 helper 合并（目标 QEMU 无 memtier 时走不通）。

### 4.1 恢复流程（lictor `snapshot_revert`，restore-memory 步）

```
 1. 磁盘 reflink 回写（不变）
★1a. 按 memtier_restore 决定模式（见 4.2）
     keep  → cp --reflink=always <snap>/memory/<snap>.memtier.raw <vm_dir>/memtier-backing.raw
             （先 unlink 旧活动后端，与磁盘回写同一段逻辑）
     merge → 不放回；helper 直接读快照目录里的副本
 2. 起 QEMU -S -incoming exec:snapshot-helper FILE 0 [--memtier=… …]
 3. 等 MIGRATION/completed；按 was_running 决定 cont
★3a. _memtier_enable_on_start 改为幂等：
     先 x-memtier-query；已启用（场景 A/C）→ 只 x-memtier-set-policy 回放 sidecar tunables
     （memtier 段只带 policy，不带 tunables）；未启用（场景 B/B'）→ 照旧 enable 或跳过
```

### 4.2 模式选择

revert 参数新增 `memtier_restore = auto | keep | merge`，默认 `auto`：

```
目标 QEMU 带 memtier？ ──否──▶ 宿主空闲内存 ≥ RAM？ ──是──▶ merge
        │是                              └──否──▶ 拒绝（提示）
        ▼
分层文件能放回 sidecar 路径？ ──是──▶ keep（A）
        └──否──▶ 路径可改写？ ──是──▶ keep + --memtier-path（C）
                      └──否──▶ 同上「空闲内存」分支 → merge / 拒绝
```

API 层要标明两种模式的资源差异：A 恢复时宿主只需常驻量，B 需要全量 RAM，超分主机上贸然选
merge 会把宿主打爆。

---

## 5. snapshot-helper 改动

### 5.1 文件格式扩展（版本号保持 1）

```c
#define SNAPHLP_F_MEMTIER   0x1   /* flags: epilogue 内含 memtier 段，快照依赖外部分层文件 */

/* 占用 SnapHelpIndex.reserved[60] 的前 24 字节（老文件为 0）： */
uint64_t memtier_sect_off;   /* epilogue 内偏移（相对 epilogue_off），含段头，含 FOOTER */
uint64_t memtier_sect_len;
uint64_t memtier_pages;      /* 所有 region n_swapped 之和，供 --info / 空间估算 */
```

兼容性：老 helper 读新文件忽略 flags 与 reserved，按 A 场景恢复；新 helper 读老文件
`flags=0` → 无 memtier 依赖，行为不变。

### 5.2 save：定位 memtier 段

epilogue 是原样字节，不解析设备段。但 memtier 段头部模式固定、blob 自描述，可在 epilogue
捕获完成后一次扫描定位：

```
段头   0x04 | be32 section_id | u8 len=7 | "memtier" | be32 instance_id | be32 version_id=1
blob   be32 n_regions
       per region: be32 len + id | be32 len + path | u8 policy | be64 nr_pages
                   | be64 n_swapped | n_swapped × { be64 gfn, be32 slot }
尾部   0x7e | be32 section_id          （send_section_footer 默认开启）
```

匹配 `\x07memtier` 后按 blob 格式向前解析：`n_regions` 很小、字符串长度合理、`nr_pages` 能
对上某个 block 的 `used_length / page_size`、尾部 FOOTER 的 section_id 与段头一致，四项都过才
认定。误匹配概率可忽略，且校验失败只是不设 flag，不影响 A 场景。

### 5.3 restore：新增参数

```
snapshot-helper FILE 0 [--memtier=keep|merge] [--memtier-backing=PATH] [--memtier-path=NEW]
```

- `keep`（默认，与今天行为一致）：epilogue 原样回放。给了 `--memtier-path` 时重写段内 `path`
  字符串（长度前缀，直接重组该段再发）。
- `merge`：
  1. 常驻页按位图回放（不变）。
  2. 对每个 region 的 pairs，从 `--memtier-backing` 以 `slot × 4K` `pread`；全零 →
     `RAM_SAVE_FLAG_ZERO`，否则 `RAM_SAVE_FLAG_PAGE`，与常驻页一起收进同一个 ram `SECTION_END`。
  3. epilogue 发 `[0, off)` 与 `[off+len, end)` 两段，剔除 memtier 段。QEMU 的
     `QEMU_VM_VMDESCRIPTION`（JSON）里仍列有 memtier，加载端对 vmdesc 只做提示不校验，无影响。
  4. `flags` 没有 `SNAPHLP_F_MEMTIER` 却指定 `merge` → 报错退出。
- `--info`（可选）：打印索引，含 memtier 依赖与页数。

`--size-hint` 不变（仍以 RAM 为上界）。selftest 增加一组带 memtier 段的流做 keep / merge /
path 改写三种往返校验。

---

## 6. QEMU 改动

### 6.1 必做（已实现，待编译验证）

1. **quiesce 延长到 `cont`。** 现在 `ram_save_cleanup → memtier_migration_src_end()` 在迁移
   完成时立刻放开 worker。改为：`src_end` 时若 runstate 不是 RUNNING，把 `migrating` 的清除
   推迟到 vm change state handler（RUNNING 时释放）；迁移失败路径保持立即释放。
   编排侧兜底（不推荐单独依赖）：快照前 `x-memtier-set-policy psi-trigger=100,
   target-resident-mb=min`，完成后恢复；但会与 hyper-agent 总控 5s 一拍的 set-policy 竞争。
2. **`memtier_dst_restore` 加校验**：`slot < nr_slots`（即后端大小 ≥ 最大 slot）、slot 不重复、
   gfn 不越界不重复；先整体校验再 `MADV_DONTNEED`，不满足直接 `error_setg` 拒绝加载并 teardown，
   而不是运行期缺页读越界。`slot_reserve()` 改为返回 bool。

### 6.2 可选

- `x-memtier-export-backing {id, path}`：postmigrate 下只把被引用的 slot 按原偏移写到稀疏
  文件（SPDK 后端走 `mt_spdk_read_sync`）。用于裸设备 / petrel 卷后端；恢复时按偏移写回。
- memtier 段版本 2 追加 `be64 backing_bytes`，加载端核对后端大小。
- 入站路径覆盖（`-incoming defer` + QMP）。C 场景由 helper 改写路径已覆盖，本期不做。

---

## 7. 控制面与编排

- **快照计划**：增加 `memtier_backing_artifact_path`（`<snap>/memory/<snap>.memtier.raw`）；
  快照 JSON 记录 `memtier: {enabled, backing_artifact, pages}` 作为「此快照依赖分层文件」的
  权威来源（helper 的 flag 只是辅助）。DB 先按 `memory_file_path` 约定派生，后续再加列。
- **空间估算**：`.state` 实际 ≈ 常驻量，后端副本初始 0、上限 = 后端大小；配额按上限预留。
- **删除**：`snapshot_delete` 一并 unlink 后端副本。
- **互斥与回退**：分层开启但后端不可 reflink → v1 回退「disable 再快照」并标明；或拒绝。
- **hyper-agent 总控**：恢复后 region 已启用，总控下一拍即纳管，无需改动。
- hyper-agent 的 `vm/snapshot.py` 与 lictor 同源，改动同步。

---

## 8. 风险与待验证

| 项 | 说明 |
|---|---|
| O_DIRECT 写共享 extent 的 CoW | QEMU 对活动后端是 `O_DIRECT` 写；reflink 后 XFS / OCFS2 / btrfs 的 directio CoW 需真机验证，尤其 OCFS2 |
| swapcache 落地后 | `RESIDENT_CLEAN` 保留 slot；quiesce 已规划转回 RESIDENT，兼容。但 quiesce 持续到 `cont` 期间一律不得 `slot_free` |
| 恢复宿主 QEMU 构建 | A/C 场景要求带 memtier；stubs 构建不注册该段，加载报未知段。B 场景不要求 |
| merge 的内存峰值 | 恢复瞬间全量 RAM 常驻；`auto` 必须先查宿主余量 |
| 能力复位失败 | `finally` 保证；lictor 重启后对运行中 VM 做一次 `query-migrate-capabilities` 纠偏 |
| 段定位误匹配 | 四项校验（见 5.2）；失败只影响 B/C，不影响 A |

---

## 9. 落地顺序

1. QEMU：§6.1 两处必改 + `tests/` 增加「快照后 postmigrate 期间 slot 不变」用例。
2. lictor：创建流程（§3）→ 回滚 keep 模式（§4.1 场景 A）→ `_memtier_enable_on_start` 幂等。
3. 控制面：快照计划字段、空间估算、API 标注。
4. snapshot-helper：§5.1 索引扩展 + §5.2 定位 + §5.3 `merge` / `--memtier-path` + selftest。
5. lictor：`memtier_restore=auto|keep|merge` 判定（§4.2）。
6. 真机：XFS / OCFS2 上 reflink + O_DIRECT CoW；4GB 客户机 80% 下刷后快照耗时与占用；
   A / B / C 三种恢复的 sha256 完整性（复用 `memtier/tests/redis_integrity.sh` 思路）。
7. 可选：§6.2 导出命令，覆盖裸设备 / petrel 后端。
