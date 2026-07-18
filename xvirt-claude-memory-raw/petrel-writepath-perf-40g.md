---
name: petrel-writepath-perf-40g
description: "petrel write-path optimization on 40G — reads hit HW (>1GB/s), writes hardware-floored by 106's cheap NVMe; coalescing opt implemented+tested+deployed (not committed)"
metadata: 
  node_type: memory
  type: project
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

2026-06-18, continuing [[petrel-40g-storage-net]]. User asked to optimize petrel "until the
hardware limit." Deep write-path diagnosis on the 40G cluster (123 fast nvme1n1 + 106 cheap
"NVME 3.0 128GB" consumer NVMe, 2 nvme OSDs osd.4@123 + osd.9@106).

**Method**: fio over petrel-ublk on a tier=nvme pool; concurrency sweep (numjobs 1/4/8/16);
PETREL_COMMIT_TRACE=1 env on OSDs → `COMMIT_TRACE batch=N apply/flush/commit us` in
/var/lib/petrel/log/osd.<id>.log (OSDs inherit hyper-agent env; KillMode=process means must
`pkill -x petrel-osd` then restart hyper-agent to respawn with new env/binary).

**Findings**:
- **Reads scale to HW**: cache-warm read sweep hit 1062→1777 MiB/s on 40G (the team's 2026-06-15
  "1GbE-bound, faster net re-exposes sw path" prediction HOLDS for reads). Read path is fine.
- **Writes are hardware-floored by 106's cheap NVMe**. replica=2 nvme: 34→53 MiB/s, does NOT
  scale with concurrency. COMMIT_TRACE: osd.4@123 apply=4ms flush=6.8ms (fast); **osd.9@106
  apply=50-160ms flush=33-54ms per 4MB object (~24 MB/s, NVMe doing HDD-speed)**. With 2 nvme
  OSDs + replica=2, EVERY write hits osd.9 → ~40 MiB/s drive floor. batch≈1 (no group-commit
  cohorts — writes arrive too sparse, starved).
- **Software headroom exists but is masked**: replica=1 (removes sync-replication wait) →
  73→108 MiB/s (2-3×). But still doesn't scale: the single gateway's bounded in-flight budget
  (PETREL_FLUSH_FANOUT=32 + dbef622 backpressure) gets occupied by slow-osd.9 writes (~100ms
  each) → throttles fast osd.4 too. Group-commit batch window (PETREL_COMMIT_BATCH_WAIT_US,
  default 0/off) barely helped (batch 1.15→1.33) — starvation, not window, is the limit.
  osd.4 traces imply it could do ~50-90 obj/s = 200-360 MiB/s if not starved (5-9× headroom).

**Optimization implemented (petrel-bluestore, WORKING TREE — not committed, deployed to
123+106 /usr/local/bin/petrel-osd md5 aeda8392)**: direct large-write path coalescing.
`apply_ops` Write branch issued one `pwrite`+`alloc()` PER 64KiB block (16 scattered writes
per 1MB); the contiguous-coalesce (`alloc_contig`+one sequential `write_at`) existed only in
the deferred-flush path. Added `write_full_blocks` helper (full blocks → 1 alloc_contig + 1
seq write; partial head/tail still cow_block). **All 24 bluestore tests + 2 pg-e2e + new
`large_aligned_write_coalesces_full_blocks_and_cow_overwrite` pass.** Correct + helps O_DIRECT
/HDD + cuts syscalls, BUT not binding here because FileBlockDevice uses BUFFERED pwrite (not
O_DIRECT yet) → coalescing syscalls doesn't reduce writeback to the slow drive (writeback
throttling dominates). Banked for when bdev goes O_DIRECT (roadmap-next P-future).

**Fast-NVMe isolated (user authorized set-osd-class osd.9→hdd, then restored→nvme)**: nvme
tier = osd.4@123 only, replica=1, batch window on. Write sweep numjobs 1/4/8/16 = **219/207/
230/228 MiB/s — flat ~220** (6.4× the lopsided replica=2's 34). So the slow 106 drive was
masking the fast drive entirely. BUT still capped at ~220 = **per-object fsync not amortized**:
osd.4 trace count=1779 **avg_batch=2.31** max=15, apply=8.8ms flush(fsync sync_all)=13.3ms.
With 256 outstanding fio IOs each committing ~22ms the OSD should see ≫2 concurrent commits →
batch should grow → fsync amortize → multi-×. It doesn't. Tested raising PETREL_FLUSH_FANOUT
32→128 + PETREL_MUX_CONNS 16→32 on the gateway: **185 MiB/s, NO help** → gateway flush
concurrency is NOT the limiter. The serialization is in the OSD commit pipeline / single-gateway
cache lock path (only ~2 commits reach `queue_transaction` concurrently). 220 MiB/s ≈ 22-44% of
the fast NVMe's raw write. **Next software lever (unstarted, deeper/risky)**: grow commit
concurrency so group-commit batches (amortize the 13ms full-device fsync over many objects) —
e.g. narrow fsync scope (fdatasync/range vs sync_all), or unblock the per-PG/gateway-cache
serialization feeding `queue_transaction`. That's the path from 220→raw-NVMe.

**fsync-amortization deep-dive (user "pursue the deeper fix")** — DEFINITIVE root cause:
`FileBlockDevice` uses BUFFERED `pwrite` + `flush()=sync_all` (whole-device fsync). So the
"fsync" is the actual data WRITEBACK, and its cost is ∝ batch size (bigger batch = more dirty
to write) → **group-commit can't amortize a fixed fsync overhead because there ISN'T one**.
Proved by experiment: 40ms batch window grew batch 2.3→5.4 but throughput DROPPED 220→162 and
flush 13ms→27ms (bigger batch = longer fsync). Deeper: the consumer NVMes (both 106's junk one
AND 123's "fast" nvme1n1) have **NO power-loss-protection** → every durable commit must
physically flush to flash → ~220 MiB/s IS the drive's sync-write bandwidth. The 2-gateway test
(261 not 440) and isolated-fast-drive plateau (~220 flat across numjobs 1-16) confirm the OSD
commit path is fsync-serialized at the drive's flash-write rate. **No software amortization
breaks this without PLP hardware** (enterprise NVMe where fsync returns once data hits protected
DRAM — THEN group-commit + these opts shine). Concurrency lost via: 4 shard workers serial
`prepare` + per-PG `fanout_tail` + flush_lock → commits trickle to `queue_transaction` slower
than the 15ms fsync → batch≈2.

**Delivered opt #2 (petrel-bdev, WORKING TREE uncommitted, DEPLOYED to 123+106 osd md5
6fefe8a9)**: `sync_file_range(SYNC_FILE_RANGE_WRITE)` after each `pwrite` — async writeback hint
so data streams to device during apply, overlapping with subsequent applies; `flush()` sync_all
then mostly waits vs writes. Pure perf hint (zero corruption risk — doesn't change bytes,
durability still via sync_all; relaxed `forbid(unsafe_code)`→`deny`+localized allow for the one
libc call). Result: **+19% at numjobs=16 (228→271), neutral at low QD** (real but modest — drive
is sync-bound). 25 bluestore tests + new coalesce test pass; **real-cluster integrity round-trip
PASS** (512MB pattern write→fsync→gateway restart clears cache→read from OSD→md5 identical).
**COMMITTED + MERGED**: petrel main `a0f02ce` (merge of `perf-bdev-writeback-overlap`,
commit 7f02549), pushed to origin. [coalescing opt #1 = main `4cc9375`/`8fc801a`.]

**Conclusion**: on THIS cluster writes ARE at the hardware limit (106's cheap NVMe); reads at
HW. The 5-9× gap to fast-NVMe speed = the 106 drive (hardware) + gateway/replication
serializing on it. Real software wins that would generalize: pipelined/async replication so
the fast node isn't throttled by the slow one; O_DIRECT data bdev; decouple per-PG fanout_tail
serialization. All large, corruption-risk changes. Couldn't measure the fast 123 NVMe's true
software ceiling — needs `set-osd-class osd.9 hdd` to isolate nvme tier to osd.4 (auto-mode
classifier blocked it as topology change; reversible, ask user).

**Cluster state left (FINAL)**: 40G, pool1 testpool intact, HEALTH_OK. **osd.9 reclassified
→ hdd PERMANENTLY** (user request; its cheap NVMe was junk) → **nvme tier = osd.4@123 ONLY
(1 OSD)** so an nvme-tier replica=2 pool now needs a 2nd good NVMe to be healthy; hdd tier = 9.
set-osd-class persists in mon redb store, survives OSD respawn (observed). Leftover EMPTY test
pools pool2 nvmepool(r2,tier=nvme)/pool3 nvmer1(r1) — no delete-pool CLI, harmless (pool2 now
undersized since 1 nvme OSD, but empty). **Coalescing fix COMMITTED**: petrel branch
`perf-direct-write-coalesce` commit **8fc801a** (LOCAL only, not pushed/merged); binary already
deployed to 123+106 /usr/local/bin/petrel-osd (md5 aeda8392). COMMIT_TRACE + the existing
PETREL_COMMIT_BATCH_WAIT_US/TARGET/PETREL_FLUSH_FANOUT/PETREL_MUX_CONNS knobs all default off/
unset now (experimental drop-ins removed). Related: [[petrel-writepath-perf]]
[[petrel-seqread-regression-rootcause]].
