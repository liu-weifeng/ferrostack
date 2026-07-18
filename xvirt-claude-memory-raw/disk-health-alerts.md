---
name: disk-health-alerts
description: "alert_monitor now does disk SMART monitoring; node 123 petrel pool has TWO failing WD HDDs (sdb, sdd)"
metadata: 
  node_type: memory
  type: project
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

**alert_monitor.py（hyper-agent）现监控物理磁盘 SMART 健康**（commit hyper-agent `3f2eb5b`）。之前只盯网卡链路，磁盘坏扇区静默不报。现每 ~60s `smartctl -H -A -i` 扫物理盘：整体健康 FAILED / NVMe critical-warning → critical；Current_Pending_Sector / Offline_Uncorrectable / NVMe 介质错误 >0 → warning。边沿触发 `alert.raised`/`alert.cleared`（category=storage, resource=disk，带型号/SN/坏扇区数），经既有事件链 → ferrovirt `alert` 表 → `/api/vcenter/alerts`。告警子系统三源（VM宕机/lictor、NIC/alert_monitor、datastore挂载/ferrovirt对账）现+磁盘 SMART。

**node 123 petrel HDD 池有两块坏盘**（消费级 WDC WD5000AAKX，需更换）：
- `/dev/sdd` SN WD-WCC2E0DCVCL0：**待处理坏扇区=72**
- `/dev/sdb` SN WD-WCAYUN738913：待处理=3、不可恢复=2

后果：petrel 读到落在坏扇区的对象 → `Input/output error (os error 5)`，且 **petrel 读路径在 medium error 时不自动切副本**（replica=2 本应能读出，是 petrel 读路径可改进点）。这正是挡住 [[snapshot-storage-migration]] 分支树迁移**全量字节读回校验**的原因（迁移本身成功，机制已由 clone_test2/engine_test 独立证明 byte-exact）。要完整验证需先换盘 / 标坏盘 OSD `out` 走副本。
