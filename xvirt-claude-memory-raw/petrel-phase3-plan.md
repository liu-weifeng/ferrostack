---
name: petrel-phase3-plan
description: Petrel integration Phases 0-2 DONE (commits+build env); Phase 3 plan (VM disks on petrel vhost-user-blk + config on LVM)
metadata: 
  node_type: memory
  type: project
  originSessionId: b5803e41-db51-453d-ad92-34f7963f863c
---

Companion to [[petrel-integration]]. Phases 0-2 of petrel storage integration are DONE; this holds the verified status + the Phase 3 plan (the main petrel-integration.md kept getting truncated on append during a degraded-shell session).

## DONE (committed + pushed) as of 2026-05-31
- **hyper-agent** repo main: `7324e51` (Phase1 petrel mgmt API: support/petrel/{paths,supervisor,cluster,volume,vhost}.py + controller/petrel.py 20 actions + shell.py register/reconcile) + `46e24f1` (fixes, hardware-validated) + `db0c818` (Phase2: support/datastore/{lvm,petrel}.py + registry).
- **ferrovirt** repo main: `eb83791` (Phase2: lvm/petrel types in service/storage_net.rs create_datastore_task + 10 petrel_* cluster-mgmt service fns + NEW controller/vcenter/petrel.rs REST + vcenter.rs merge). `cargo check` clean.
- **petrel cluster LIVE** on host: mon@127.0.0.1:6790 + osd.0 + osd.1, fsid 1c92fd5e-5b9e-4675-8788-542ca884299b, pool nas_data id=1, /var/lib/petrel/cluster.json. test-vol-0 (1GiB) in pool. Don't re-bootstrap.

## ferrovirt BUILD ENV (one-time host setup, NOT in repo — redo if env resets)
1. `ln -sf /usr/lib/x86_64-linux-gnu/libclang-21.so.1 /usr/lib/x86_64-linux-gnu/libclang.so`
2. PAM headers (apt is broken by xvirt's qemu deps → extract deb manually):
   `cd /tmp && apt-get download libpam0g-dev && dpkg-deb -x libpam0g-dev_*.deb /tmp/pamdev && cp -r /tmp/pamdev/usr/include/security /usr/include/`
3. build: `cd ferrovirt && PATH=/root/.cargo/bin:$PATH LIBCLANG_PATH=/usr/lib/x86_64-linux-gnu BINDGEN_EXTRA_CLANG_ARGS=-I/usr/lib/gcc/x86_64-linux-gnu/15/include cargo check`
Rust gotchas: `TaskObj::exclusive(a,b,c)` 2nd arg is `impl Into<String>` → pass `&str` directly, NOT `.into()` (else E0283). `VcenterResponse::of(x)` works fine.

## petrel REST API now live (ferrovirt :8080)
- `/api/vcenter/storage/petrel/cluster` POST{meta_disk,data_disks[],replica?,pg_num?,pool?,host?} / GET?host= / DELETE{wipe?}
- `/api/vcenter/storage/petrel/hosts` POST{bucket_id,name,parent?} / DELETE{bucket_id,osd_ids[]}
- `/api/vcenter/storage/petrel/osds` POST{osd_id,host_id,weight}; `.../osds/{osd}/out` POST; `.../osds/{osd}` DELETE
- `/api/vcenter/storage/petrel/pools` GET?host= / POST{name,size?,min_size?,pg_num?,ec_k?,ec_m?}
- `POST /api/vcenter/datastore` with type=lvm{vg,size_bytes,fs_type=xfs | devices,vg_name} OR type=petrel{pool_name?,create_pool?,size?,min_size?,pg_num?,ec_k?,ec_m?}

## PHASE 3 PLAN — VM create on petrel block disks + LVM config home (IN PROGRESS, nothing written yet)
Goal: a VM whose virtual disk(s) live on petrel (raw block via vhost-user-blk), config.xml/nvram on an lvm/local datastore (petrel = block only, no FS).

**hyper-agent vm/guest.py create_vm** (verified current code):
- line 66 `vm_datastore` = HOME datastore segment → config.xml at /datastore/<vm_datastore>/<uuid>/config.xml; line 70 vm_datastore_type. ALREADY decoupled from per-disk datastore. For petrel VMs vm_datastore must be lvm/local (not petrel).
- lines 84-111 per-disk loop does image.create_img(d["path"]). NEED: skip for petrel disks; instead petrel volume.create + vhost.start → /run/petrel/<vol>.sock; stash sock for the launcher.
- line 114 config_builder.build_guest → config.xml.

**hyper-agent support/qemu_command.py** (verified): QemuLauncher; `_build_disks` line 678 → `_build_disk` ~683 (-blockdev + -device per gc.LibvirtConfigGuestDisk); `_build_memory` line 521; `_build_machine` 441.
- NEED vhost-user-blk branch: `-chardev socket,id=chr-<n>,path=<sock>` + `-device vhost-user-blk-pci,chardev=chr-<n>,id=<dev>[,num-queues=N]`.
- NEED shared mem when ANY vhost-user disk present: `-object memory-backend-memfd,id=mem,size=<MB>M,share=on` + `-numa node,memdev=mem` (or -machine memory-backend=mem). Inspect _build_memory output first (line 521).

**guest_config.py**: LibvirtConfigGuestDisk has no vhost-user source. SIMPLEST approach: side-channel param `param["vhost_disks"]=[{sock,target_dev,bus,queues}]` that QemuLauncher reads directly (bypass libvirt XML for these disks) rather than extending the XML model. Decide during impl.

**start_vm/stop_vm**: vhost socket must exist before qemu launch → spawn petrel-vhost per petrel disk before, stop after. (PetrelController.petrel_vhost_start exists; VM lifecycle should drive it.)

**ferrovirt vm_create.rs**: disk path calc ~line 312, datastore resolve ~1331. petrel disk → type=block, no qcow2 path, alloc petrel vol; ensure VM home datastore is lvm/local. CreateVmRequest already has per-disk datastore + top-level datastore (home).

**Sub-tasks**: (3a) qemu_command vhost-user-blk + shared-mem ✅ DONE; (3b) guest.py create_vm petrel-disk branch + start/stop_vm vhost mgmt; (3c) ferrovirt vm_create petrel-disk DB/param wiring; (3d) E2E real VM booting off a petrel disk.

### 3a DONE (hyper-agent commit 075880e, PUSHED, unit-tested)
qemu_command.py QemuLauncher now: `self.vhost_disks=[]` attr (caller fills per petrel disk: {socket(req),target_dev?,bus?,num_queues?,bootindex?,alias?,addr?(dict{bus,slot,function} or PCI obj)}); `_build_vhost_disks()`/`_build_vhost_disk()` emit `-chardev socket,id=chr-vhost-diskN,path=<sock>` + `-device vhost-user-blk-pci,chardev=...,id=...[,num-queues,bootindex,PCI addr]` (called in build() right after _build_disks); `_build_memory()` when vhost_disks non-empty emits `-object memory-backend-memfd,id=ram-node0,size=<bytes>,share=on` + `-numa node,memdev=ram-node0` (else unchanged). Unit test /tmp/vh_test.py (vhost present → chardev/device/memfd/numa all True) + /tmp/vh_test2.py (no vhost → NO_MEMFD/NO_NUMA, plain -m) both PASS. NOTE: first commit was broken (method defs lost to a failed Edit anchor → AttributeError); amended with defs (now 65 insertions). Lesson: after Edit, ALWAYS run the unit test before committing.

### 3b NEXT (hyper-agent vm/guest.py)
Need to read: start_vm + stop_vm in guest.py (how QemuLauncher is instantiated/spawned, how tap NICs are set up as the model for per-disk side-effects), and how disks are passed. Plan:
- create_vm (line 84-111 loop): for a petrel disk (identify by d.get("backend")=="petrel" or d.get("vhost") or a datastore-type marker the controller sets), SKIP image.create_img (no qcow2). The petrel VOLUME itself should be created here (volume.create(vol, size)) OR at start — decide: create at create_vm (persistent), map (vhost) at start_vm.
- start_vm: before spawning qemu, for each petrel disk call petrel vhost.start(vol,size)→sock, and append {socket:sock, target_dev, bus, num_queues, bootindex, addr} to launcher.vhost_disks. Then build()/spawn.
- stop_vm: after qemu down, vhost.stop(vol) for each petrel disk.
- Import: `from hyper_agent.support.petrel import volume as petrel_volume, vhost as petrel_vhost`.
- vol naming convention: suggest `<vm_uuid>-<disk_id_or_index>`; must match what ferrovirt vm_create allocates (3c) — keep deterministic so create/start/stop/delete agree. Petrel disk size: d.get("size") GiB.

## ENV WARNING (this session)
Severe Bash/Read instability: stdout dropped/garbled/delayed, parallel batches cancelled, Writes/heredoc-appends silently truncated, `echo`/exit-144 from the safety-classifier model being temporarily unavailable. Mitigations: ONE critical tool call per turn; re-verify every Write/append landed (wc -l); prefer Write to a fresh file over `cat >>` append; NEVER `pkill -f petrel` (matches the Bash tool's own snapshot argv → kills the command). Kill petrel daemons by explicit PID.

### 3a VERIFIED + PUSHED (final)
hyper-agent commit `230977b` (main, pushed, in sync). Unit tests PASS when run with `PYTHONPATH=/root/code/xvirt/hyper-agent python3 /tmp/vh_test.py` (the Bash tool cwd resets to /root/code/xvirt, NOT hyper-agent — always set PYTHONPATH=/root/code/xvirt/hyper-agent to import hyper_agent). Verified argv for a vhost disk: `-object {"qom-type":"memory-backend-memfd","id":"ram-node0","size":<bytes>,"share":true}` + `-numa node,memdev=ram-node0` + `-chardev socket,id=chr-vhost-disk0,path=<sock>` + `-device {"driver":"vhost-user-blk-pci","chardev":"chr-vhost-disk0","id":<alias>,"num-queues":N,"bootindex":B,"bus":"pcie.0","addr":"0xa"}`. No-vhost VM → no memfd/numa, plain -m (unchanged). Test scripts saved: /tmp/vh_test.py (vhost), /tmp/vh_test2.py (no-vhost).

### 3b STARTING NOW — guest.py create_vm/start_vm/stop_vm
Was about to grep guest.py for: def start_vm / def stop_vm / def create_vm / def delete_vm / QemuLauncher / .build() / spawn / Popen — to learn how QemuLauncher is instantiated & how to populate launcher.vhost_disks before build(), and where to start/stop petrel-vhost per disk. NEXT SESSION: do that grep first (with PYTHONPATH note above irrelevant for grep). Then:
- create_vm petrel-disk branch (~line 84-111 loop): identify petrel disk via d.get("backend")=="petrel" (controller must set this) → skip image.create_img; create the petrel volume via `from hyper_agent.support.petrel import volume as pv; pv.create(vol, size_bytes)`. vol name deterministic = `<vm_uuid>-disk-<id>` (must match ferrovirt 3c).
- start_vm: before spawn, for each petrel disk `sock = petrel_vhost.start(vol, size_bytes); launcher.vhost_disks.append({...})`. (Model the per-disk side-effect on how priv.taps NICs are handled.)
- stop_vm: petrel_vhost.stop(vol) per petrel disk after qemu down.
Petrel cluster LIVE (mon+osd.0+osd.1 on nvme1n1/sda/sdb). Don't re-bootstrap.

### 3b DONE + PUSHED (hyper-agent commit 4f9e2a1)
- config_builder._disks: skips disks with backend=='petrel' (out of libvirt XML).
- guest.create_vm: petrel-disk branch — no qemu-img; petrel volume.create(d['vol'], size); collects specs → writes <vm_dir>/petrel-disks.json sidecar (cleared if none).
- guest._launch (shared entry, ~line 1131): before build(), _setup_petrel_vhost(launcher, config_path) reads sidecar, petrel vhost.start per vol → sock, fills launcher.vhost_disks; rollback _cleanup_petrel_vhost_for_vm on failure. New helpers in guest.py: _petrel_sidecar_path/_load_petrel_disks/_setup_petrel_vhost/_cleanup_petrel_vhost_for_vm/_cleanup_petrel_vhost_by_uuid (all near _setup_taps ~line 1230).
- monitor/manager.py _on_conn_close (~line 312): _cleanup_petrel_vhost_by_uuid(uuid) after tap cleanup (VM stop/crash → stop vhost).
- Tested: helper imports, sidecar path (=/datastore/seg/uuid/petrel-disks.json), JSON round-trip — all PASS; 4 files py_compile OK. Run python with PYTHONPATH=/root/code/xvirt/hyper-agent (cwd resets to xvirt parent).
- Disk dict fields agent expects for petrel: {backend:"petrel", vol:"<vm_uuid>-disk-<id>", size:<GiB>, create:bool, target_dev?, bus?, num_queues?, boot_order?, alias?, address?}. ferrovirt (3c) MUST emit these in create_vm's disk param.

### 3c NEXT — ferrovirt vm_create.rs petrel-disk wiring
In ferrovirt/src/service/vm_create.rs: disk path calc ~line 312 (`/datastore/{ds}/{vm_uuid}/{disk_uuid}.{fmt}`), datastore resolve ~line 1331. For each disk: look up its datastore type. If type=="petrel": set disk param backend="petrel", vol=`<vm_uuid>-disk-<id>` (deterministic; matches agent + future delete), size=<GiB>, NO file path/qcow2, type="block"; do NOT compute /datastore path. Also: the VM HOME datastore (top-level `datastore`, where config.xml lands) must be lvm/local — if it resolves to type=petrel, reject with clear error (petrel has no FS). The agent create_vm reads param["disks"][].backend. Build ferrovirt with the libclang/PAM/BINDGEN env (see above). Then 3d E2E: create lvm config datastore + create VM with a petrel disk via REST, start it, confirm vhost-user-blk + guest sees the disk.

### 3c DONE + PUSHED — REAL commits: 254237f (partial: home-reject only) + 97a571c (completion). cargo check clean, synced. (IGNORE any other hashes below — buffer-corrupted.)
vm_create.rs create_vm_task_labeled (~line 1340-1398): (1) home datastore type==petrel → reject (no FS for config.xml, ~line 1345); (2) per-disk loop (~1361-1398) already resolves each disk's (dseg, dtype) via inline `SELECT uuid,mount_path,type FROM datastore` — added: a `vm_uuid` pre-bound from req.uuid (written back to req so complete()/fill_disks use the same uuid), and for non-cdrom petrel disk → set backend="petrel",type="block",vol="<vm_uuid>-<disk_uuid>",create=true,datastore_type="petrel", `continue` (skip segment). NOTE: NO separate resolve_ds_type helper — the existing inline query returns dtype, reused directly. (3) fill_disks (~line 310-316): skip `/datastore/...` path computation when `os_def(&x,"backend")==Some("petrel")`. Agent create_vm gets config.disks[] with backend/vol/type/create. vm_disk persists path=NULL + block_dev=NULL for petrel (vhost sock computed at start). Agent unwraps params.get("config") → guest.create_vm.
KNOWN GAP for Phase 4: vm_disk row for a petrel disk has type=block, file_path NULL, but no explicit petrel marker/vol stored — identify petrel disks later by JOIN vm_disk→datastore WHERE type='petrel', and recompute vol=<vm_uuid>-<disk_uuid> (disk_uuid must be persisted; verify vm_disk stores disk uuid). 

### 3d ✅ DONE + E2E PASS (2026-06-01)
VM `e2e-petrel-vm` (uuid 07980293-...) boots off a petrel block volume via vhost-user-blk, config.xml on LVM datastore. Verified live: QEMU cmdline has `vhost-user-blk-pci` + `memory-backend-memfd,share=on` + `-numa node,memdev=ram-node0` + `-chardev socket,...path=/run/petrel/<vol>.sock`; `petrel-vhost serve` daemon serving the 4GiB vol (pulled map epoch=3 pool=1); petrel pool shows 16 objects; config.xml has 0 `<disk>` for the petrel vol (correctly excluded). datastores: cfg-pool(lvm, 20G LV in ubuntu-vg, home) + petrel-pool(petrel, nas_data).

**CRITICAL ARCHITECTURE DISCOVERY**: hyper-agent **proxies start_vm/launch to LICTOR** (`hyper-agent/.../controller/vm.py` → lictor_client → :5560). **lictor runs the actual QEMU process** and has its OWN parallel copy of `support/qemu_command.py`, `support/config_builder.py`, `vm/guest.py` (NOT shared with hyper-agent). So Phase 3a/3b changes I'd put in hyper-agent had NO effect on launch — had to port them ALL into lictor. Lesson for any VM-runtime change: **patch lictor, not (or as well as) hyper-agent.** create_vm (disk alloc, sidecar write, config.xml) DOES run in hyper-agent; but start/stop/QMP/launch run in lictor.

**Bugs fixed during 3d E2E** (all committed-pending — see below):
1. `datastore.type` MariaDB ENUM lacked `petrel` (JPA-generated from Java DatastoreType which didn't have it) → `ALTER TABLE datastore MODIFY type enum(...,'petrel')` + added `petrel` to virt-control `DatastoreType.java` for future bootstrap.
2. hyper-agent `vm/guest.py`: `json` not imported (sidecar write crashed `name 'json' is not defined`) → added `import json`. Also config_builder petrel-skip never landed in Phase 3b (silent Edit fail) → re-added.
3. **lictor** got the full port: `support/qemu_command.py` (vhost_disks attr + _build_vhost_disk + memfd in _build_memory + call in build()), `support/config_builder.py` (skip backend==petrel), `vm/guest.py` (json import + _setup_petrel_vhost/_load_petrel_disks/_cleanup_petrel_vhost_* helpers that spawn `petrel-vhost serve` reading /var/lib/petrel/cluster.json for mon/pool + sidecar; called in launch_vm before build()), `monitor/manager.py` (_cleanup_petrel_vhost_by_uuid on QMP EOF).
4. Custom QEMU env fixups (non-petrel, see [[dev-box-bringup-state]]): seabios firmware symlinks `/usr/share/qemu/*.bin → /usr/share/seabios/*` (custom qemu-system-data deb doesn't ship them — DEB BUG, should add to its .install); reinstalled custom `qemu-system-modules-opengl/spice` 10.0.7 (apt -f had reverted them to archive 10.2.1 → ABI mismatch `undefined symbol qemu_dmabuf_get_strides` → `qxl-vga not valid`).

**COMMITTED** (2026-06-01, all 3 own repos): lictor `e1691cc` (vhost-user-blk launch port); hyper-agent `b257dc8` (json import + config_builder skip); virt-control `6bc631a` (DatastoreType+petrel). NOT pushed yet (push them next). The datastore.type ALTER was applied to the LIVE mx DB directly (idempotent; re-run on fresh DB: `ALTER TABLE datastore MODIFY type enum('btrfs','ext4','gfs2','lvm','nfs','ocfs2','raw','xfs','zfs','petrel') NULL`).

**TODO carry-over**: (a) custom qemu-system-data deb should ship the seabios `/usr/share/qemu/*.bin` symlinks (build-deb `*.install` gap — currently created by hand, lost on reinstall); (b) custom qemu-system-modules-opengl/spice keep getting reverted to archive 10.2.1 by `apt -f install` → ABI mismatch; pin/hold them or fix deb versioning.

## ALL COMMITS (petrel integration — AUTHORITATIVE, verified via git log, 2026-06-01):
hyper-agent main (HEAD 1e19f39, pushed+synced): 7324e51(P1 mgmt API) 46e24f1(fixes, hw-validated) db0c818(P2 lvm/petrel datastore backends) 230977b(3a qemu vhost-user-blk+memfd, unit-tested) 1e19f39(3b guest.py petrel-disk lifecycle+sidecar).
ferrovirt main (HEAD 97a571c, pushed+synced): eb83791(P2 lvm/petrel datastore types + petrel cluster REST) 254237f(3c partial: home-reject) 97a571c(3c completion: per-disk petrel branch + fill_disks skip + vm_uuid pre-bind). cargo check clean.
petrel cluster LIVE: mon@127.0.0.1:6790 + osd.0 + osd.1 on nvme1n1(meta)/sda/sdb(data), pool nas_data id=1, fsid 1c92fd5e-5b9e-4675-8788-542ca884299b, /var/lib/petrel/cluster.json. Don't re-bootstrap.
Remaining: 3d E2E (task #6: live VM boot off petrel disk), Phase 4 lifecycle (delete/edit/resize/snapshot — see KNOWN GAP above re: identifying petrel disks via JOIN datastore type='petrel' + recompute vol), Phase 5 UI (deferred per user).
ENV WARNING persisted whole session: Bash/Read intermittently return empty (safety-classifier model overload, also "exit 144"); Edit anchors silently failed several times (3c, 3b had to be re-applied). Lesson baked in: after EVERY Edit re-grep to confirm it landed, and cargo check / py_compile before committing. Never `pkill -f petrel` (matches the Bash tool's own argv). Build python tests with PYTHONPATH=/root/code/xvirt/hyper-agent (cwd resets to parent).

## PHASE 4 (VM delete reclaims petrel volume) — DONE 2026-06-01, committed+pushed
Commits: petrel `325a945` (Volume::delete + petrel-ublk `delete` subcmd + NetworkObjectAccess robustness), hyper-agent `2f89ec9` (volume.delete real impl + delete_vm petrel reclaim + glob-locate vm_datastore), ferrovirt `4773151` (delete_vm allows null ds_segment for all-petrel VMs + agent timeout 120→600s).

What works (verified): petrel-ublk `delete --vol [--ignore-missing] --mon --pool` reclaims all data+snapshot+head objects, idempotent. Small petrel-disk VM (≤16 objects) full REST DELETE?purge=true reclaims vm_dir + DB row + petrel volume + vhost daemon. delete_vm reads petrel-disks.json sidecar before rmtree, stops vhost then deletes vol.

Robustness fixes in petrel client (bins/petrel-osd/src/client_data.rs NetworkObjectAccess): call() has 15s RPC timeout (wedged OSD no longer hangs forever); retry() does bounded refresh-retry ONLY on stale-epoch/connection errors (NotFound/business errors return immediately); remove_once idempotent (missing object = success). Volume::delete uses remove_objects_wide with DELETE_CONCURRENCY=8 (fault-tolerant join_all, not try_join_all).

KNOWN LIMIT (hardware, not code): on this box's 2× slow WD-HDD OSDs, deleting a LARGE volume (256+ objects) is very slow (~1-2s per remove even of thin-volume holes, because petrel OSD remove = RocksDB delete + replica fan-out regardless of existence) — a 1GiB/256-obj delete takes >5min. DELETE_CONCURRENCY was tuned 256→64→8: 64+ knocked the slow OSDs Down (heartbeat miss under burst → "acting set empty"); 8 is stable (OSDs survive) but slow. ROOT CAUSE = petrel OSD remove throughput + OSD heartbeat-flapping under load on slow HDDs, NOT the lifecycle code. Future petrel fix: skip-remove-nonexistent (track allocated objects) so thin-volume delete only touches real objects; and/or raise OSD heartbeat tolerance under IO load.

OSD flapping mitigation applied this session: mon restarted with `--osd-down-timeout-secs 60 --slow-osd-latency-us 5000000` (was default 6s) → stops idle-cluster epoch churn. To restart a single OSD: it must join while the OTHER osd is already Up (osd `demo --replicate` waits for `--wait-osds`=2; restarting both at once → both time out waiting for each other). cluster.json at /var/lib/petrel; osd configs deploy-out/osd.N.toml.

## "浏览存储池" (browse petrel pool: list volumes + snapshots) — DONE 2026-06-01, committed
Commits: petrel `846e748`, hyper-agent `9228430`, ferrovirt `031d355`, xvirt-client `6625b54` (PUSHED 2026-06-01).
- **petrel-ublk `list [--json]`** (`bins/petrel-ublk` list_volumes): scans all Up OSDs' `<vol>.head` (suffix-filtered server-side), reads each head meta → {name,size,object_size,thick,snapshots[{name,size,created_unix}],parent}. New `PgManager::list_oids(prefix,suffix,limit)` in petrel-pg + `ObjReq::List`/`ObjResp::Oids` in petrel-osd dispatch (wrapped in `tokio::task::spawn_blocking` — list_oids is sync, scans 64 PG collections, would block the async worker and time out the RPC's own reply otherwise). `NetworkObjectAccess::list_objects` fans out + merges.
- **CRITICAL BUG fixed**: `list_oids` pagination was an infinite loop — `store.list_objects(coll,start,BATCH)` range is `[start,..)` INCLUSIVE; the loop set start=last-of-page and never broke on a short page → re-fetched the last element forever for ANY non-empty collection. Fix = skip the boundary element on non-first pages + break when page len < BATCH.
- **petrel-ublk `snapshot` now has an OFFLINE branch** (no --pid/--control-file, just --vol --mon): opens the unmapped volume and writes the COW snapshot to head directly. Lets the control plane snapshot offline (powered-off) VM disks. hyper-agent `volume.snapshot()` + `petrel_volume_snapshot` action fall back to offline when the vol isn't vhost-served (previously raised).
- **UI wiring**: hyper-agent action `petrel_pool_browse {path}` → `volume.browse()` returns `{path,abs_path,entries[]}` (same shape as `browse_datastore`) where `entries[].type ∈ volume|snapshot`; root path lists volumes (with snapshot_count, drillable), path=`<vol>` lists that vol's snapshots. ferrovirt `browse_datastore_files` branches on `datastore.type=='petrel'` → calls `petrel_pool_browse` instead of `browse_datastore` (petrel has no FS mount). xvirt-client `DatastoreBrowser` reuses the dir/file navigation: NAV_TYPES={dir,volume} drillable, snapshot=leaf (shown `@name`, camera icon), volume shows "(N 快照)", isPetrel hides upload/download/delete. E2E PASS via web API (login→ferrovirt→agent→petrel-ublk): 6 vols at root, drill-in shows snapshot typed correctly.

## RELIABLE OSD-recovery procedure (when the 2 HDD OSDs flap Down — happens on idle too, ~90s after register; heartbeat loop `osd_call_leader` has no per-call timeout so a hung mon-conn stalls it silently)
1. List OSD pids by EXACT comm (avoids the `pgrep -f petrel` self-match trap that returns the Bash tool's own argv → exit 144): `ps -eo pid,comm | awk '$2=="petrel-osd"{print $1}'`.
2. `kill` those pids; sleep 6; `kill -9` survivors; sleep 4 (redb holds its lock until the proc dies — racing a restart → "Database already open. Cannot acquire lock").
3. Start BOTH fresh with `--wait-osds 1` (default 0 = wait for pool size 2 → if peer not up yet it waits forever and exits): `nohup /usr/local/bin/petrel-osd demo --mon 127.0.0.1:6790 --osd-config /var/lib/petrel/deploy-out/osd.N.toml --replicate --transport binary --wait-osds 1 --daemon >> /var/lib/petrel/log/osd.N.log 2>&1 &` (osd.0 first, sleep 8, then osd.1, sleep 14). Verify `petrel-mon osds --mon 127.0.0.1:6790` shows `2/2 up`.
4. Restarting **hyper-agent is SAFE** for running OSDs: `cluster.start_all()`→`supervisor.spawn` is idempotent (pgrep match → returns existing pid, no restart). It won't disturb manually-started OSDs.
