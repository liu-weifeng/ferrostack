---
name: petrel-read-failover
description: petrel read path now fails over to replica on medium-error; bad-disk testing also surfaced a branch-clone snapshot bug
metadata: 
  node_type: memory
  type: project
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

**petrel 读路径加了 acting-set 副本失败转移**（commit petrel `dae7b3e`，部署 123+106）。之前 `client_data.rs::read_once` 只路由 primary，primary 盘坏扇区/介质错误（OSD 回 `Input/output error`）→ 直接报错、不切副本（`retry()` 只治 stale-epoch）。加 `route_all()`（acting set 全成员 addr，primary 在前）+ read_once 失败转移：primary 读 Err/RPC 失败 → 依次试副本，返回首个成功。不存在对象回零数据(Data 非 Err) 不误触发；写路径/object_exists 未加（exists 元数据在 nvme redb，一般不打 HDD 坏扇区）。

**实测（123 池 sdb/sdd 双坏盘，[[disk-health-alerts]]）**：之前 5/5 重试全卡坏扇区的快照读，现自动从 `123:6802`（坏盘）切到 `106:6816` 副本读成功，主卷 S0/S3 读回 **byte-exact**（C0/C3）。坏盘下读稳定性达成、且失败转移服务的是正确副本数据。

**"分支克隆快照丢数据"已定位 = 坏盘间歇性数据损坏，不是 clone 代码 bug（2026-06-21 排查结论）**。一度怀疑分支 clone 快照丢 copy-up（读 `d12-b5@S1` 得父内容），但系统隔离排查**证伪了 clone 特定性**：
- 写持久化**正常**（写卷→新网关读 OSD = byte-exact）。
- 普通卷 写→快照→snapdiff 读 = **byte-exact 正常**（stest）。
- 但**同样步骤的另一普通卷 rmain@S0 却读出全零**（repro3）—— **普通卷也会错，间歇性**。
- `clone_test2`（瘦克隆写+快照+读）此前 PASS。
⇒ 不是 clone/分支路径的代码 bug，是**两块坏盘（sdb/sdd）造成的间歇性数据损坏**：某些对象落在坏扇区/某副本读不出时，读偶尔返回 `Data(零)`（不是 Err，failover 抓不到）→ 静默错数据。换盘前在这池上**没法干净验证任何快照内容**。**分支树迁移代码结构正确**（clone_test2 证明机制行），全量字节验证待坏盘更换。
**额外健壮性缺口**：部分坏盘失败模式下读回 `Data(零)` 而非 Err → 读 failover 兜不住（只兜 Err）。要彻底兜需 OSD 对"对象存在但数据读失败"返 Err（而非零），让 failover 切副本。
