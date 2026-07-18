---
name: petrel-40g-storage-net
description: petrel cluster storage net moved from 1G mgmt to 40G direct (10.40.0.x); throughput is disk-bound so 40G = headroom not speedup; 106 NVMe-rename fix + networkd persistence
metadata: 
  node_type: memory
  type: project
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

2026-06-18: switched the 123+106 petrel distributed-storage cluster network from the
1G mgmt net (192.168.2.x) onto the **40G direct-connect link** (ConnectX-3 / mlx4_en,
MTU 9000), and benchmarked.

**Topology**: 40G is a direct cable between the two nodes' Mellanox dual-port cards,
2nd ports: `enp4s0d1`=10.40.0.123 (MAC 24:8a:07:d9:3b:11), `ens4d1`=10.40.0.106
(MAC 50:6b:4b:57:35:c1), subnet 10.40.0.0/24. ufw already had a blanket
`ALLOW IN 10.40.0.0/24` on both, so no firewall work.

**How the rebind was done** (in-place, no data loss — cluster had ~0 data): edit
`/var/lib/petrel/cluster.json` on both nodes (123: listen/mon_addr/addr_ip + hosts[].ip
→ 10.40.0.x; 106: mon_addr→10.40.0.123:6790, addr_ip→10.40.0.106), `pkill -x petrel-mon
petrel-osd`, then `systemctl restart xvirt-hyper-agent` (123 mon first, then 106). On
restart hyper-agent `start_all()` reads cluster.json → mon binds `--listen`, OSDs
`--mon`/`--addr-ip`; OSD `register()` re-advertises public/cluster addr → mon map
updates to 10.40.0.x automatically. **KillMode=process kept the running `center` VM
(pid 3855, disk on local XFS) untouched** through the 123 hyper-agent restart.

**Reboot persistence**: the 10.40.0.x IPs were runtime-only (`ip addr add`, not in any
config). Made durable via networkd-native drop-in `/etc/systemd/network/30-storage40g.network`
(match by MAC + Address + MTUBytes=9000) — deliberately NOT netplan, because their
`/etc/netplan/01-netcfg.yaml` has a pre-existing typo `nameservers.addresses:
[223.5.5.5 8.8.8.8]` (missing comma) that makes `netplan generate` HARD-FAIL. **Flagged
but did not fix** that typo (mgmt net via OVS br-ex currently works; risk-averse). cluster.json
backups left at `*.bak-1g` on each node.

**Also fixed (prereq)**: 106's 5 OSDs (osd.5-9) were DOWN — its two NVMe drives swapped
names on a reboot (13.4G Optane meta now `nvme1n1`, 119G data-OSD now `nvme0n1`), so the
osd tomls' `db_bdev=/dev/nvme0n1pX` didn't exist. Rewrote `/var/lib/petrel/deploy-out/
osd.{5..9}.toml` to **stable paths**: db→`/dev/disk/by-partlabel/petrel-osd{N}-dbwal`,
osd.9 data→`/dev/disk/by-id/nvme-NVME_3.0_128GB_202603210005`. Reboot-proof now. (123
was fine.) Same device-rename class as [[nic-rename-on-slot-change]].

**Benchmark result (fio on /dev/ublkb0, ublk gateway on 123, replica=2 cross-host)**:
seqwrite 32 MiB/s, seqread 76 MiB/s, randwrite 226 IOPS (0.9 MiB/s), randread collapsed
(~10 IOPS transient HDD thrash). **KEY**: petrel throughput here is DISK-bound (consumer
WD-HDD pool + per-object redb txn+fsync replication), far below even the OLD 1G link's
~118 MB/s. So **40G is headroom, not a speedup** — the network was never the bottleneck
for this HDD pool. NIC counters PROVED the move though: ~8.5 GB storage+replication traffic
on 40G during the bench, ~1 MiB on the 1G mgmt link. Raw 40G iperf3 ≈ 8.3 Gb/s usable
(ConnectX-3 underperforms its 40G rating — mlx4_en tuning — but still >> any disk).
**NVMe-only pool test (2026-06-18, same 40G, replica=2 across the 2 nvme OSDs osd.4@123
+ osd.9@106)** — created `petrel-mon create-pool --tier nvme` (pool 2 `nvmepool`, kept
empty after; no delete-pool CLI). Result vs the HDD pool: seqwrite **34** MiB/s (HDD 32 —
**identical**), seqread 106 (HDD 76, 1.4x), randwrite 691 IOPS (HDD 226, 3x), randread
24 IOPS (HDD 10 — both collapsed). **CONCLUSION: neither 40G nor NVMe raises throughput —
the bottleneck is petrel's SOFTWARE data path, not media or network.** seqwrite ~34 MiB/s
on NVMe = ~5-6% of raw NVMe = serialized per-object redb txn+fsync commit (no group-commit/
pipelining). Reads are single-stream object-fetch bound; 4k-random-read against 4MB objects
+ 1GB gateway cache = huge read amplification → randread collapses on any media. Fix is in
petrel software (group-commit, pipelined replication, batching, read parallelism/readahead),
see [[petrel-writepath-perf]]. Related: [[petrel-phase3-plan]] [[ocfs2-on-petrel]]
[[state-reconcile-and-lab-env]].
