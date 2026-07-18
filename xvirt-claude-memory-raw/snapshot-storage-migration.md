---
name: snapshot-storage-migration
description: Snapshot-carrying cross-storage migration is now a first-class ferrovirt API + agent proxy (distinct from petrel-shared cross-host migration)
metadata: 
  node_type: memory
  type: project
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

带快照树的**跨数据存储迁移**已接成平台一等 API（区别于 [[vm-live-migration-cpu-mode]] 那条 petrel 共享盘的跨主机 RAM 迁移）。搬的是磁盘数据本身 + 整棵快照树，目的端 reflink 复刻链上空间共享。

**API**: `POST /api/vcenter/vm/{vm}/migrate`
- body `{dest_host}` → 老的 petrel 共享盘跨主机在线迁移（只迁 RAM/设备态）。
- body `{dest_datastore, online?}` → 新的带快照跨存储迁移。

**链路**（commits: ferrovirt `37fb489` + hyper-agent `5712fdc`，已 push + 部署 123/106 active）:
- ferrovirt `service/snapshot_migrate.rs` `migrate_storage_task`：读 DB `vm_snapshot` 树 + `vm_snapshot_disk.snapshot_artifact_path` 逐盘构建 xvirt-snapmig tree spec。
- **离线**（VM 关机，§5.5 forward）：agent `snapshot_chain_migrate`（hyper-agent 本机跑 `xvirt-snapmig offline-tree`）在目的端重建整棵 reflink 树（含 active 叶）→ 重指 `vm_disk.file_path` + `vm_snapshot_disk.*_path` → `vm_update::regenerate_agent_config`（已改 pub(crate)）重写 config.xml。**要求目的 datastore 与源 VM 同主机**（同机读源+写目的）。
- **在线**（VM 运行，§5.4 backward）：逐盘 `storage_mirror_start`（drive-mirror 持续=可回滚）→ 镜像跑时重建冻结历史子树 → `storage_mirror_pause_snapshot` 取两端一致基准 → `storage_mirror_complete` cutover；失败任意环节 → `storage_mirror_cancel` 回滚。这 4 个是 lictor `vm/storage_migration.py` action，hyper-agent `controller/vm.py` 瘦代理转 lictor。

**底层工具**（独立 GitLab 仓，部署 `/usr/local/bin` on 123+106）: `xvirt-blockdiff`（FIEMAP 块 diff，.xvdiff 格式）/ `xvirt-snapmig`（offline-tree 重建）/ `petrel-snapdiff`（petrel 对象级 diff，与 OCFS2 链互迁）。

**目的端落平台原生布局**（关键修复，否则 VM 在目的端不可用）：active=`<dest_home>/<disk_uuid>.<fmt>`、快照副本=`<active>.snap.<snap_uuid>`（= snapshot.rs 命名）、config.xml 拷到目的家目录让 update_vm 重写、源 config.xml 归档成 `.migrated`（保证 `glob /datastore/*/<uuid>/config.xml` 唯一，monitor 用）。xvirt-snapmig 加 per-node `dst`；hyper-agent 加 `copy_path`/`rename_path` 原语。commits: snapmig `805411e` + hyper-agent `d909ebe` + ferrovirt `30590a9`（均 push）。

**离线完整 VM E2E 已验（DONE，2026-06-20，node 123）**：真 cirros VM（raw 盘 on ocfs2-a）→ API 建快照树 `S0→{S1,S2}`（含中途 revert）→ `POST /migrate {dest_datastore:ocfs2-b, online:false}` → 4 镜像 byte-exact、整树 reflink 共享 cirros base、config.xml 重指 ocfs2-b + glob 唯一、**VM 从目的盘启动到 cirros login**、revert 到迁移后 S1/S2 字节精确还原。测试 VM uuid `1856496a-...`（留在 123，shutoff）；ocfs2-a/ocfs2-b 已注册为 ferrovirt datastore（host 4）。

**在线（drive-mirror）完整 VM E2E 已验（DONE，2026-06-20，node 123）**：运行中 cirros VM（on ocfs2-a）→ `POST /migrate {online:true}` → **同一 QEMU 进程不重启**活迁 cutover、活迁后 QEMU 实开 ocfs2-b 盘（重启从目的盘引导只开 ocfs2-b，证明真切换）、两端基准 `.migbaseline` 字节一致（drive-mirror 同步证明）、历史快照 byte-exact、config 重指 + glob 唯一、控制台续跑同会话、revert 到迁移后 S1/S2 字节精确。修了在线两 bug（lictor `6203936`+ferrovirt `3c74782`）：① BLOCK_JOB predicate 读错字段（完整 msg vs data）+ register-after-event 竞态 → 改 **query-block-jobs 轮询**（race-immune）；② drive-mirror 需先建目标目录（mkdir）。**revert 要求 VM 完全 shutoff**（stop 后等状态真到 shutoff 再 revert，否则 fail）。
**petrel 裸卷 ↔ OCFS2 快照链互迁已验（DONE，2026-06-20，node 123）**：`.xvdiff`/blob 格式混用 `petrel-snapdiff`（对象级 diff）+ `xvirt-blockdiff`（FIEMAP reflink diff）做双向链迁移，两向 byte-exact、增量只传变更对象（25%/步）、OCFS2 侧保留 reflink 共享。回归脚本 `petrel/bins/petrel-snapdiff/interop_test.sh`（commit `e222f06`）。要点：`xvirt-blockdiff splat blob TARGET` 写自描述 blob 到文件或 `/dev/ublkbN`（petrel-snapdiff 无 splat，OCFS2→petrel 写到 served 块设备再 online snapshot）；petrel-ublk serve 后台跑 + `--control-file` + `--pid` 在线快照（会 flush 网关缓存）。
**petrel 盘已接进完整 VM 迁移 API（DONE，2026-06-20，node 123）**：`POST /migrate {dest_datastore, online:false}` —— dest 是 petrel 数据存储 → OCFS2 盘+链搬到 petrel 卷+COW 快照；源盘 petrel → 反向搬回 OCFS2 reflink。commits: hyper-agent `4fc30a6`（support/petrel_chain.py 引擎 + snapshot_chain_migrate_petrel/find_vm_config 动作）+ ferrovirt `71b00ba`（migrate_petrel_task 编排 + disk_dto pool + regen _home_datastore 回退）。实测 cirros VM 线性链 S0→S1→S2：**OCFS2→petrel** 3 快照 byte-exact + 盘/快照重指（vm_disk.protocol='petrel'+source_name+petrel_pool_id；快照 V2_RBD_NATIVE/RBD_SNAPSHOT 存 snap 名）+ config.xml→petrel-disks.json sidecar + **从 petrel 卷 vhost-user-blk 启动到 cirros login**；**petrel→OCFS2** 3 快照 byte-exact + reflink + config.xml 搬到目的+glob 唯一 + **从 OCFS2 启动到 cirros login**。关键点：① petrel 仅**线性链**（单卷线性 COW，分支树拒绝；分支需 clone 待后续）；② petrel 盘 DB 标记=`protocol='petrel'`+`source_name`(vol)+`petrel_pool_id`；③ petrel 快照复用 `V2_RBD_NATIVE`/`RBD_SNAPSHOT` 枚举（artifact_path 存 snap 名=snap uuid）；④ to_petrel 家目录留原 OCFS2（盘无段→`_home_datastore` 提示），from_petrel 家目录随盘搬 dest；⑤ 注册了 `petrel-ds` 测试 datastore（type=petrel，mount 占位，pool 走 resolve_home_pool）。
**petrel 分支快照树迁移已实现（用 clone，2026-06-21）**：去掉线性限制 —— 树分解为「活动路径=主卷 + 每个 off-path 节点=从 主卷@最近主路径祖先 `rbd clone`(thin) 出的分支子卷」。commits: petrel `5b0d258`（**read_snapshot 补瘦克隆读穿透**——分支卷快照里未 copy-up 的对象要读穿透父基快照，否则 petrel-snapdiff 读分支快照丢数据，是个真 bug）+ hyper-agent `8644659`（petrel_chain 多卷树引擎）+ ferrovirt `e710441`（migrate_petrel_task 树分解；DB artifact 存 `<vol>@<snap>`）。实测分支树 `S0→{S1→S2, S3}`(current=S3) OCFS2→petrel **结构全对**（主卷 d12[@S0,@S3] + 分支卷 d12-b5[S1]/d12-b6[S2] 均 clone 主卷@S0）。**全量字节读回校验被 petrel 池两块坏盘挡住**（见 [[disk-health-alerts]]：sdb/sdd 坏扇区→petrel I/O error，且 petrel 读路径 medium-error 不切副本）；机制 byte-exact 由 clone_test2/engine_test/早前线性整机独立证明，换盘后可补。要点：分支只一层 clone（始终自主卷，避嵌套 clone 读穿透）；petrel `clone --mode thin` = rbd clone，恢复快照(rollback)=对象级零拷贝 clone_object。

**未验（deferred）**：petrel 在线迁移（drive-mirror 到 vhost 难）；petrel↔petrel 跨池（用 petrel-ublk migrate-pool）；在线多盘暂停取基准非跨盘原子；分支树全量字节读回（待坏盘更换）。
