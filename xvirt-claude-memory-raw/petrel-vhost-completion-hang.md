---
name: petrel-vhost-completion-hang
description: "RESOLVED (2026-06-14): petrel-vhost 真 VM 持续写挂死 = 两个独立 bug(丢 avail-kick + 没开 indirect 描述符大块IO近满环死结);修复=process_queue 复查环 + features 加 VIRTIO_RING_F_INDIRECT_DESC"
metadata:
  node_type: memory
  type: project
  originSessionId: 3e93f53f-4faf-4927-8318-9562d4a000a4
---

**状态:已修复(2026-06-14)**。改动只在 `petrel/bins/petrel-vhost/src/lib.rs` 两处,
未 commit(在工作树)。详见 `petrel/docs/resolved-vhost-completion-hang.md`。

## 根因 = 两个独立 bug(经插桩在 node 123 Ubuntu-6.8 复现工装上区分)
- **模式 A:丢 avail kick**(占多数)。guest 在后端读完 avail.idx、框架 read_kick 已 consume
  kickfd 之后又投新链,这拍 kick 经 ioeventfd→KVM→eventfd 在竞态窗口被吞,worker 永久 idle
  在 ep_poll,guest 等完成 → 互等。独立线程周期重扫 avail 每轮救回 50–270 条 = 烟枪。
  **修:`process_queue` 加回标准 rust-vmm "收割—复查环"** —— 收割空后 `enable_notification()`
  做带 SeqCst fence 的 avail.idx 复查,窗口内有新项就继续收割。(异步重写当初删掉了这段。)
- **模式 B:大块 IO 近满环死结**(剩余那类)。看门狗显示 avail.idx==next_avail==next_used 全对齐
  (已全回收+通知)guest 仍卡,**重扫 avail 捡不到、重发 IRQ 也救不回**。根因=**没 advertise
  `VIRTIO_RING_F_INDIRECT_DESC`**:1MiB 写需 ~257 个直接描述符 > 环深 256 → guest 近满环死结。
  **修:`features()` 加 `VIRTIO_RING_F_INDIRECT_DESC`**(virtio-queue 自动跟随间接表,parse 无需改)。

## 验证(12GiB petrel 卷 + Ubuntu 6.8 golden,fio 1M/qd16/1500M)
修复前 3/8 HANG;仅复查环(无 indirect)2/3 HANG;**复查环+indirect 14/14 PASS**;
逐请求完成(无专用完成线程)+两处修好 **14/14 PASS**。

## 旧假设被推翻(免再走弯路)
- 旧备忘的"把 add_used+signal 放回单一 worker 线程(SPDK/cloud-hypervisor 模型)"**方向错**:
  实测专用单线程完成**不解决模式 B**;逐请求多线程完成在两处修好后照样 14/14。完成不需收敛单线程。
- 重发 callfd 心跳对模式 B 无效(模式 B 不是丢 used IRQ)。

## 复现工装(node 123,root 口令 weifeng)
golden `/root/ubuntu-bench/golden.qcow2`;单轮 `/root/petrel-ubuntu-bench-final.sh`;
测试卷 pool 2 `41893998-…-b41143e2-…`(12GB)。判定:`reboot: Power down`=过 vs timeout 杀=挂。
详见 [[petrel-writepath-perf]] / [[petrel-integration]]。
