---
name: petrel-vm-disk-perf-and-oom
description: "real-VM petrel disk perf (raw vol vs OCFS2) — seq saturates, 4k-random collapses (petrel 4MB-object amplification); OCFS2 random-write OOM'd gateway → kernel panic → 123 crashed (recovered)"
metadata: 
  node_type: memory
  type: project
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

2026-06-19, continuing [[petrel-writepath-perf-40g]]. Tested real-VM petrel disk perf two ways
on 123+106 / 40G / pool3 (tier=nvme=osd.4, replica=1). Guest = QEMU/KVM with a custom
initramfs (busybox + STATIC fio built on dev-box `cargo`-toolchain host) booting the HOST
kernel (`/boot/vmlinuz-*`, CONFIG_VIRTIO_BLK=y) — no boot disk/partition (the Debian-qcow2 +
qemu-nbd approach kept corrupting GPTs; initramfs is the robust path). Mode1 = petrel vol via
`petrel-vhost` vhost-user-blk. Mode2 = `petrel-ublk` → `mkfs.ocfs2 -M local` → mount → RAW file
→ QEMU `-drive ...cache=none,aio=native`.

**Results (MiB/s, guest fio direct=1)**: seq write — vhost 158-453 / ocfs2 252 / host 210;
seq read — vhost **945-969** / ocfs2 239 / host 567; 4k randwrite — vhost **~1.7** / ocfs2 0.86;
4k randread — vhost **~0.7** / ocfs2 0.17. **Seq saturates** (vhost ≥ host = VM+vhost-user zero
overhead; OCFS2 halves seq-read). **4k random COLLAPSES in BOTH** — the vhost "good random"
(64/134) was a THIN-volume artifact (read/write fresh zero objects); on real data raw-vol
collapses too. Root cause = petrel **4MB object size** → 4k random touches a whole 4MB object →
~1000× amplification (+ no-PLP sync write). NOT OCFS2/VM-specific.

**⚠ CRASH (real bug)**: host 4k-randwrite stress on the OCFS2 file CRASHED node 123 (hard down,
user power-cycled). Kernel log: `petrel-ublk` write-back cache ballooned to **6.5GB anon-rss →
OOM-killed**; then ublk device I/O-errors; then **OCFS2 `VFS_BUG_ON_INODE` → kernel BUG at
fs/namei.c:682 → panic**. Root cause of gateway OOM (petrel-client CachingObjectAccess): allocate-
before-check (write_object :355-400); sparse `data.resize(off+4k,0)` :366 → ~4MB buffer/obj;
`evict` :267 only reclaims CLEAN (random writes keep objs dirty → can't evict); random-dirty
flush to petrel is KB/s (dirty grows >> drains); `bytes`=Σ data.len() UNDERCOUNTS real RSS
(misses Vec capacity slack + per-obj dirty-extent Vecs + flush snapshot copy :506) → the
`2×max_bytes` hard-fail backstop fires against a 3×-undercounted value → OOM.
**FIXED (petrel-client, main `fad27df`, gateways deployed 123+106)**: `write_object` → atomic
check+alloc + `backpressure()`. Same-lock fit-test `c.bytes ≤ max_bytes` (accept one write,
≤1-obj overshoot) else release lock + loop flush(dirty→clean)+evict until ≤max_bytes (fail
after 3 no-progress rounds). GOTCHA hit+fixed: fit-test must be `c.bytes ≤ max_bytes` NOT
`c.bytes+added ≤ max_bytes` (latter → infinite spin in the `(max-added,max]` gap → hung tests).
Regression `backpressure_caps_memory_near_max_bytes_when_flush_stuck` (per-step ≤max_bytes+1obj).
VALIDATED on 123 with the SAME crash-load (4k randwrite id64×16): `--cache-total-mb 256` → RSS
flat ~1.2GB whole run, node survived (was 6.5GB climbing→crash). Memory now BOUNDED, scales
linearly w/ max_bytes; **RSS ≈ ~4× cache-total-mb** (Σdata.len() under-counts: capacity 2× +
per-obj 4k extent vecs + glibc frag — worst case 4k-random; seq ≈1×). **RSS tightened (main `9fdc73f`)**: ① c.bytes accounting len→`data.capacity()` (all 5
add/sub sites: write_object/read_object Some+None/evict/remove); ② `malloc_trim(0)` after a
successful `backpressure()` drain (relaxed petrel-client `forbid(unsafe_code)`→`deny`+local
allow for the one libc call; linux-gnu only). Re-validated same load/256MB cache: RSS ~1.2GB
(~4.7×) → **peak ~620MB / mean ~450MB (~2×)**, stable sawtooth, node survived. **jemalloc (main `405b77a`, gateways deployed)**: petrel-ublk+petrel-vhost `#[global_allocator]`
= tikv-jemallocator + BAKED config `_rjem_malloc_conf = background_thread:true,dirty_decay_ms:0,
muzzy_decay_ms:0`. GOTCHA: jemalloc DEFAULT is WORSE (~5.5×, retains dirty pages 10s + per-thread
arenas) — MUST set bg-thread + immediate decay to win (narenas:1 not needed; bg-thread purges off
hot path → no throughput regression: seqw 184/seqr 631 MiB/s). Re-validated same load/256MB cache:
RSS peak ~280MB = **~1.1×**. FINAL RSS journey: glibc ~4.7× → +capacity+trim ~2.4× → jemalloc
default ~5.5× → **jemalloc tuned ~1.1×**. So default 1024 → ~1.1GB. petrel-client capacity-acct +
malloc_trim kept (help non-jemalloc lib consumers; malloc_trim no-ops under jemalloc). DEPLOY: size
--cache-total-mb by RSS≈1.1×; multi-gateway → cgroup MemoryMax. Further (chunked flush snapshot)=
follow-up.
**Deploy rule: VM disks = raw petrel vol (vhost-user-blk); do NOT put OCFS2 RAW
disks on a gateway that can OOM** (OCFS2 panics kernel on backing-device failure).

**Recovery after reboot**: hyper-agent auto-reconciled mon+osd.0-4 + center VM; cluster 10/10
HEALTH_OK on 40G; persistent 40G IP came back. Test volumes deleted. **Committed to petrel
main `58d80e9`**: `docs/vm-disk-perf-and-gateway-oom.md` + roadmap-next P0(gateway OOM) +
P2(4MB-object random short-coming). (Commit straight to main per [[git-commit-workflow]].)
**P2 random-IO attempt (main `a60e53b`)**: hypothesis was random collapse = 4MB-object read
amplification (4k read → gateway fetched whole 4MB object). Fixed: `read_object` ADAPTIVE fetch
— small read (len < PETREL_READ_ALIGN, default 64KiB) fetches one aligned window (~16×); large
read (seq/bulk) fetches whole-object readahead (pure-windowing regressed cold seqread 113→82
MiB/s, adaptive restored it). `fill_unfetched_gaps` fills only not-locally-valid holes (preserves
in-flight invariant). Test `caching_read_fetches_aligned_window_not_whole_object`. **BUT random
IO STILL not "usable" — bottleneck MOVED, wasn't really amplification**: 4k randread 240→363 IOPS
(only 1.5×); 4k AND 64k randread give ~same ~360 IOPS = **OSD cold-read REQUEST-RATE bound**
(~360/s, ~2 effective concurrency despite 256 outstanding, ~0.7s latency) — a read-path RPC
latency/concurrency issue, NOT object granularity. Cold seqread ~113 MiB/s same root. Random
WRITE ~350 IOPS = no-PLP fsync (hardware). **Real remaining levers for usable random IO: OSD read
RPC concurrency/latency (next) + PLP NVMe; smaller objects help amplification/PG-parallelism but
useless until the request-rate is fixed.** Related: [[petrel-writepath-perf-40g]]
[[petrel-40g-storage-net]] [[ocfs2-on-petrel]].

**Read-path ~290-IOPS wall DUG layer-by-layer (2026-06-19, petrel main fd9325a + db5c41c, pushed,
deployed 123+106).** User: "挖 OSD 读路径 ~2 并发瓶颈". Method: ublk gateway + osd.4 (pool3
replica=1, single nvme osd) cold-read fio + per-segment timing instrumentation (gateway in-flight
gauge + OSD read split) + perf + iostat. Ruled out EACH layer by measurement:
- ublk queue-thread: OLD serialized to nr_queues (block_on per IO); **FIXED** → async in-flight
  (spawn_blocking + eventfd wake to COMMIT) → gateway in-flight ~4→~200. Didn't raise throughput
  (downstream-bound) but is the literal "~2 并发" fix + needed once OSD fixed.
- gateway CPU/codec: perf showed **22% in serde `Vec<u8>` visit_seq** (bincode decodes Vec<u8>
  byte-by-byte; 64KiB payload = 65536 deserialize_u8). **FIXED** → `#[serde(with="serde_bytes")]`
  on ObjReq::Write.data + ObjResp::Data. **Wire-COMPATIBLE** (bincode both paths = `[u64 len][raw
  bytes]`; md5 3-way write-new/read-new, write-new/read-old, write-old/read-new all match). perf
  confirmed visit_seq gone. Gateway only ~1.75 cores = NOT CPU-saturated anyway.
- disk: raw nvme1n1 QD1=119µs / QD128=30.9k IOPS → NOT the wall.
- QoS (mclock): avg_acq_us=**0** (capacity 10000, 290 nowhere near limit) → not it.
- redb metadata (on Optane nvme2n1): iostat db-disk r/s≈0 (all RAM-cached) → not it.
- **store.read ≈3.4ms** initially looked like the wall, BUT in-segment timing (995999a) **REFUTED
  the redb hypothesis**: `begin_read=4µs / open HEADERS+get=12µs / open EXTENTS=3µs / extents.get=31µs
  / deferred=9µs` = **redb only ~59µs** (all RAM-cached); the cost is the **data `pread`≈6ms**
  (buffered FileBlockDevice page-cache/readahead amplifies+serializes concurrent cold reads vs raw
  `--direct=1` QD1=119µs). **FIXED data pread → O_DIRECT** (block-dev only, separate read fd, 4096-
  aligned bounce buffer; files stay buffered; md5 write-then-cold-read OK; coherent because O_DIRECT
  read forces page-cache writeback+invalidate + writes fsync'd before ack). Cut single pread 6→~2.7ms
  + no host-page-cache pollution. **BUT throughput still ~290** — store.read in-flight gauge shows
  **max_inflight=79 (NOT serialized)** but **avg in-flight only ~1.7** (290×0.006). So the real
  throughput wall is **gateway concurrency-DELIVERY**: fio offers 256, gateway delivers only ~1.7 avg
  to OSD store.read. consumer NVMe needs sustained QD-depth for IOPS (raw QD128=30.9k). Tried truly-async ublk gateway
  (e8fcaf7: BlockHandler `*_async` + `rt.spawn`, self:Arc<Self> futures, like petrel-vhost) →
  **gateway CPU 189%→150% but throughput UNCHANGED** (gateway-delivery was a red herring too).
- **ACTUAL KILLER = PREFETCH AMPLIFICATION (b469c11, FIXED, +36% rand / 10.6× seq)**: iostat caught
  it — flood does **data-disk 202 MiB/s (3238×64KiB) serving only 1.1 MiB/s guest = ~180×
  amplification**, disk only 25% util. `prefetch_hint`'s `first_obj<=last_end+1` check: under
  concurrent reads, threads race the shared `last_end` → ~half of random reads misjudged "sequential"
  → each fires 16-object (tens-of-MB) readahead. FIX: monotonic frontier + consecutive-seq-run
  counter (prefetch only after RA_SEQ_MIN=3 in-order steps; behind-frontier out-of-order reads
  neutral; forward gap resets). Random never reaches threshold; true seq (even concurrent) warms up.
  Measured (123, async gw): **4k randread 302→412 IOPS; seqread 55→584 MiB/s** (racy prefetch was
  ALSO wrecking 8-stream concurrent seqread). Tests: prefetch-after-warmup + new skips-random case.
- **Remaining ~412 randread wall** = consumer no-PLP NVMe latency at the OSD's actual QD (avg~1.7
  peak~79); needs PLP drive / fuller pipeline / smaller objects. Seq+bulk unaffected (584 MiB/s).
- **OBJECT GRANULARITY is a HUGE random-IO lever (7999042, measured)**: swept `--object-size`
  64KB/256KB/1MB/4MB on osd.4 single-nvme 2GB vol. **4k randread: 64KB=4177 vs 4MB=544 IOPS (7.7×)**;
  randwrite 64KB=2430 vs 4MB=491 (5×); holds COLD (64MB cache: 3741 vs 347, 10.8× → NOT cache-capacity).
  Mechanism: **gateway per-object cache-entry cost grows with object size** — a 4MB object hit by
  scattered 4k reads bloats its sparse `data` toward 4MB + thousands of `valid` segments → per-read
  resize/fill_unfetched_gaps/slice/merge O(entry): 4MB=2.3ms gw-CPU/read vs 64KB=0.6ms. Disk-independent
  (~80 r/s all sizes). TRADEOFF: small obj wins random (7.7×), large obj wins SEQUENTIAL (4MB=514 vs
  64KB=364 MiB/s, +41%). ACTIONABLE NOW (no code): random-heavy vols (VM/DB) → `--object-size 256KB`
  (near-64KB random + mild seq loss); seq/bulk → keep 4MB. object-size is already per-volume.
- **64KiB-window cache REFACTOR ATTEMPTED → DATA-LOSS BUG → REVERTED (e13c382, code reverted to
  7999042)**: rewrote CachingObjectAccess to chunked (oid→chunk_idx→ChunkEntry ≤64KiB + per-object LRU
  + batch-evict-to-low-water). **36 unit tests pass (cross-chunk/cold-read/concurrent-stress) + direct-IO
  & cold-restart verify pass**, BUT **256MB buffered dd corrupts (objects 4-63 read zeros)**. Localized:
  cmp first-diff @16MiB; passes with big-cache OR big-dirty alone, fails only at default 1024/256 (both
  eviction+inline-flush). Root: chunked `c.bytes` accounting DRIFTS high → spurious eviction at 256MB
  (real RSS only 295MB) → exposes a latent race (inline-flush takes dirty→obj clean→evicted mid-flush→
  flush-fail re-merge `get_mut` misses evicted obj→silent loss). Old per-object cache never evicted at
  256MB so never hit it. LESSON: data-path cache rewrites MUST be E2E-tested with large buffered-dd +
  eviction pressure (unit tests + direct-IO verify insufficient). **2nd attempt (also REVERTED,
  9131f22)**: added flush-in-flight evict protection (ObjChunks.flushing flag) + bytes audit
  (debug_assert c.bytes==Σcap). 37 unit tests pass incl a slow-backend evict-during-flush regression;
  **audit never fires in-process → drift NOT reproducible in-process**. On cluster: 256MB buffered
  still broke (pathologically slow ~min / small-vol total write-failure + zeros = release-only panic or
  `c.bytes-=freed` underflow). REAL BLOCKER: the drift only manifests through the Volume layer (split
  per-object) + readahead prefetch + async backend + eviction pressure — direct write_object/read_object
  unit tests can't trigger it. NEXT: write a unit repro via Volume::write/read + prefetch + latency-backend
  to force the drift in-process BEFORE retrying on cluster; make audit cheap (sampled, not O(N)/op). This
  refactor is deeper than expected — keeps hitting cluster-only failures invisible to unit/direct-IO tests.
  **3rd attempt LANDED (41e0292, deployed 123+106)**: wrote the Volume-layer repro (Volume::write/read
  split + prefetch + latency backend + small-cache evict-during-flush + concurrency) under debug audit —
  **audit NEVER fires → accounting is always correct → THERE IS NO BYTES DRIFT.** The real bug was the
  flush-in-flight eviction race all along (flush takes dirty→obj clean→evicted mid-flush→re-merge can't
  find it→silent loss); the `flushing` flag fixes it. Also added flush COALESCING (contiguous dirty across
  chunks → 1 write per fully-dirty obj; was 64 → why 256MB took minutes) + the audit. Cluster confirm:
  **256MB buffered dd IDENTICAL-OK** (the exact failing test), write 1s, no panic; 4MB-obj 4k randread
  1322→293k IOPS (cacheable working set: chunk cache packs densely → ~100% hit vs per-object sparse 4MB
  entries wasting cache). 36 unit tests + Volume repro guard it. `--object-size 256KB` no longer needed —
  4MB default now fine for random too. LESSON CONFIRMED: cluster-only data-path bugs need a Volume-layer +
  prefetch + latency-backend + eviction-pressure repro to surface in-process; direct write/read_object
  tests are insufficient. HARNESS: hot-swap
  /usr/local/bin/petrel-osd (wire+behavior compat; unlink-then-copy, "Text file busy" else), osd logs
  /var/lib/petrel/log/osd.N.log; killed osd NOT auto-respawned — `systemctl restart xvirt-hyper-agent`
  reconciles it (KillMode=process keeps other OSDs); manual start needs `--daemon </dev/null` (else
  interactive REPL exits on stdin EOF). GOTCHA: heredoc `awk "{print $1}"` → bash eats `$1`; use `'`.
