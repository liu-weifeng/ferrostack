---
name: petrel-integration
description: Long task — integrate petrel distributed block storage into xvirt (datastore type + VM disks via vhost-user-blk + LVM config pool)
metadata: 
  node_type: memory
  type: project
  originSessionId: b5803e41-db51-453d-ad92-34f7963f863c
---

Integrating the new `petrel/` component (Rust distributed block storage, sibling repo in xvirt tree) into xvirt. Started 2026-05-31.

**Goal**: petrel as a new datastore type for VM virtual disks (raw block via vhost-user-blk); since petrel only serves blocks (no FS/dirs), VM config files (config.xml) live on a new LVM-backed local datastore.

**Control plane is ferrovirt (Rust)** — user confirmed "virt-control 已启用，已替换为 ferrovirt". All new datastore/VM code goes in `ferrovirt/src/{controller/vcenter,service}/`. virt-control (Java) is legacy/dead. ferrovirt listens :8080 (same as virt-control, can't run both). Live start script: `xvirt-ferrovirt.sh` (components: lictor, hyper-agent, ferrovirt, xvirt-client, nginx). lictor (:5560 ROUTER, :5561 PUB) holds QMP and is proxied by hyper-agent for vm/snapshot/xvt actions.

**BUILD/TOOLCHAIN GOTCHA**: `cargo` is NOT on the non-interactive shell PATH. It lives at `/root/.cargo/bin/cargo` (symlink → rustup, toolchain stable-x86_64). Bash tool shells don't source `~/.bashrc`/`~/.cargo/env`, so plain `cargo build` fails with "command not found" SILENTLY when backgrounded (empty log). Always build with `export PATH="/root/.cargo/bin:$PATH"` or call `/root/.cargo/bin/cargo` directly. This also applies to building ferrovirt. The petrel workspace is large (tokio + vhost-user + bluestore/redb) — release build takes several minutes.

**Petrel single-node bring-up** (CLI flags below were read from --help during corrupted-buffer period — RE-VERIFY each binary's actual --help once built; loop-device smoke test was NEVER actually run, cargo was missing):
1. `petrel-deploy --meta-disk <m> --data-disks <csv> --replica 2 --pool nas_data --pg-num <n> --output <dir> --apply --assume-yes` (wipes disks; meta disk → GPT partitions for RocksDB+WAL, data disks → raw + 4KiB superblock, NO mkfs; writes deploy-out/{cluster-map.json,petrel.toml,osd.N.toml})
2. `petrel-mon init --store <redb> --from <dir>/cluster-map.json`
3. `petrel-mon serve --store <redb> --listen 127.0.0.1:6790` (daemon)
4. per osd.N.toml: `petrel-osd demo --mon 127.0.0.1:6790 --osd-config <dir>/osd.N.toml --replicate --transport binary --daemon` (osd.toml has data_bdev/db_bdev/wal_bdev + pool id=1 + mon addr; BlueStore on raw partitions, NO mount needed)
5. `petrel-mon wait-up --mon 127.0.0.1:6790 --count <N> --timeout 30`
Status/health: `petrel-mon {health,pools,usage,list-volumes} --mon 127.0.0.1:6790 --json`. pool nas_data deterministically gets id=1.
Volume: `petrel-ublk create --vol <n> --vol-size <bytes> --mon 127.0.0.1:6790 --pool 1` (offline/RBD-like, no daemon, no --backend when --mon+--pool given). Other ublk subcmds: serve/resize/snapshot/rollback/rmsnap/clone/list. **NO volume-delete subcommand exists in CLI yet** — gap for VM delete (orphaned volumes); may add `petrel-ublk delete` later (petrel is modifiable).
VM disk export: `petrel-vhost --socket <sock> --vol <n> --vol-size <bytes> --mon 127.0.0.1:6790 --pool 1 --num-queues N` (NO subcommand; runs foreground; vhost-user-blk). QEMU side: `-object memory-backend-memfd,share=on` (shared mem REQUIRED) + `-chardev socket,path=<sock>` + `-device vhost-user-blk-pci,chardev=...`.

**REAL disk layout (verified via lsblk/blkid 2026-05-31, env had delayed results so re-verify before destructive ops)**:
- nvme0n1 119G = SYSTEM DISK (nvme0n1p1 /boot/efi, p2 /boot, p3 → ubuntu-vg LVM, root = ubuntu-vg/ubuntu-lv ext4). **DO NOT WIPE.** ubuntu-vg has ~58G FREE.
- nvme1n1 13.4G = Intel Optane (currently xfs) → petrel **metadata disk** (Optane ideal for RocksDB/WAL). User OK'd wipe.
- sda 465G WDC HDD (currently xfs) → petrel data OSD.0. User OK'd wipe.
- sdb 465G WDC HDD (currently xfs) → petrel data OSD.1. User OK'd wipe.
- sdc = blank (no signature).
Petrel layout decision: meta=nvme1n1, data=sda,sdb, replica=2.
**Config pool decision**: user picked "nvme0n1" but that's the system disk — DO NOT pvcreate/vgcreate it. Instead create an LV in the EXISTING ubuntu-vg (58G free) → mkfs.xfs → mount as the LVM/local config datastore for config.xml. LvmBackend must support "create LV in existing VG", not just "make new VG from disk".

**Supervision: hyper-agent + pidfiles** (user choice, NOT systemd). hyper-agent spawns petrel daemons (mon, per-OSD, per-volume vhost), tracks PIDs under /var/lib/petrel/*.pid, reconciles/re-adopts on startup (same pattern as `hyper_agent/support/nbd_export.py` which manages qemu-nbd subprocesses).

**hyper-agent integration facts**:
- Controllers: `hyper_agent/controller/*.py` use `metaclass=ControllerMetaclass` + `@action("name")`. Registered by instantiation in `hyper_agent/shell.py` (~line 31: VmController, HostController, NetworkController, StorageController, DeviceController, XvirtToolsController, SnapshotController, AegisController). Add PetrelController there. (operation_server.py also lists controllers ~line 54.)
- Datastore backends: `hyper_agent/support/datastore/` — base.py (DatastoreBackend ABC: create/destroy/scan/import_pool, make_metadata helper, type_name), registry.py (_BACKENDS dict, add new type here), local.py (LocalBackend, reusable helpers: _mkfs_xfs/_get_fs_uuid/_mount/_persist_fstab/_query_capacity/_ensure_device_available/_wipefs). LvmBackend can subclass/reuse LocalBackend helpers.
- create_datastore action (`controller/storage.py:18`) dispatches by params["type"] → registry.get_backend(type).create(params, cb). delete_datastore → backend.destroy.
- subprocess helper: `hyper_agent.support.host_util.run_command` and `hyper_agent.utils.process.run_command(cmd, shell=False) -> (stdout, stderr, code)`.
- VM create agent side: `hyper_agent/vm/guest.py:create_vm` (builds /datastore/<seg>/<vm-uuid>, qcow2 via image.create_img, writes config.xml), `_reconcile_disk_files` ~line 699. QEMU cmdline built in `hyper_agent/support/qemu_command.py` (NOT a vm/launcher.py — that file doesn't exist).

**ferrovirt integration facts**:
- controllers: `ferrovirt/src/controller/vcenter/{datastore,vm,host,...}.rs`; also `controller/datastore.rs`. services: `ferrovirt/src/service/{datastore? (none yet — uses storage_net.rs),vm_create.rs(1797 lines),vm_update.rs,write.rs,...}`. NOTE: no service/datastore.rs — datastore logic likely in storage_net.rs / controller. vcenter/datastore.rs is ~90 lines.
- task model: async TaskManager (service/task_manager.rs).

**STATUS 2026-05-31 (end of session)**: Phase 1 hyper-agent code COMMITTED as `7324e51` on hyper-agent repo `main` (hyper-agent is its OWN git repo, root /root/code/xvirt/hyper-agent, branch main). NOT pushed yet. petrel bins BUILT + installed to /usr/local/bin (petrel-{deploy,mon,osd,ublk,vhost}, built from petrel release via `/root/.cargo/bin/cargo`). Single-node REAL-disk bootstrap NOT run yet (deferred: session had severe shell-output instability — stdout dropped/delayed/out-of-order — unsafe to wipe real disks without observing each step). Dry-run deploy (`petrel-deploy ... --output /tmp/pdry` WITHOUT --apply) succeeded and is SAFE (writes config only, disks intact) — use it to inspect cluster-map.json/osd.N.toml format. NEXT SESSION: re-verify disks (lsblk), then run bootstrap via `cd /root/code/xvirt/hyper-agent && python3 -c "from hyper_agent.support.petrel import cluster; ..."` (must set PYTHONPATH/cwd=hyper-agent or ModuleNotFoundError) OR via live agent ZMQ action petrel_cluster_create.

**REAL CLI corrections applied to wrappers (verified from --help)**: petrel-mon subcmds = init/serve/set-pg-num/add-osd/add-bucket/out-osd/remove-osd/add-mon/remove-mon/set-pg-temp/create-pool/set-autoscale/usage/scrub-status/scrub-trend/pools/osds/pg-temps — NO wait-up, NO health, NO list-volumes, NO --json. petrel-ublk subcmds = serve/create/resize/snapshot/rollback/rmsnap/clone — NO list, NO delete; snapshot/rollback/rmsnap require --control-file+--pid (online only). petrel-ublk resize offline = `--size`(not --new-size) `--vol --mon --pool`. petrel-vhost is `serve` subcommand: `petrel-vhost serve --socket --vol --vol-size --object-size --mon --pool --control-file` — NO --num-queues. petrel-osd `demo --mon --osd-config --replicate --transport binary --daemon`. Default pool id=1. cluster.py reads fsid/pool-id from generated cluster-map.json (defensive multi-format parse) since mon has no json. _count_up_osds parses `petrel-mon osds` text for 'up'.

**STATUS as of 2026-05-31**: Phase 1 hyper-agent code WRITTEN but UNTESTED. Created: `hyper_agent/support/petrel/{__init__,paths,supervisor,cluster,volume,vhost}.py` + `hyper_agent/controller/petrel.py` (PetrelController, 20 actions, all import-OK + py_compile-OK). Registered in `hyper_agent/shell.py` _build_controllers() (line ~40) + reconcile hook (background thread calls petrel_cluster.start_all on agent startup). NOTE: operation_server.py does NOT have its own controller list — only shell.py does; a stray import was added then removed. petrel bins build was RE-LAUNCHED with correct cargo path (was never built before). Not yet committed.

**Phasing** (commit+push per phase): (0) build+install petrel bins — IN PROGRESS (cargo PATH fixed, building); (1) hyper-agent petrel mgmt API (PetrelController + support/petrel/) w/ pidfile supervision + ferrovirt REST to bootstrap single-node cluster from API; (2) lvm datastore type (LV in existing VG) + petrel pool-as-datastore type; (3) VM create on petrel disks (vhost-user-blk) + config.xml on LVM; (4) edit/resize/snapshot/delete lifecycle; (5) UI (deferred — user said not yet).

**ENV WARNING**: this session had severe tool-result delivery delays (results arrived many turns late, batched, out of order). Re-verify disk state with fresh single commands before any petrel-deploy --apply. Smoke test ran only on loop devices (safe). No real disk wiped yet as of memory write.

---
## AUTHORITATIVE STATUS (supersedes any conflicting hashes above) — 2026-05-31

hyper-agent repo (own git, /root/code/xvirt/hyper-agent, branch main, PUSHED to origin ssh://192.168.2.107:2222/liuwfng/hyper-agent.git):
- `7324e51` Phase 1: petrel mgmt API (support/petrel/{__init__,paths,supervisor,cluster,volume,vhost}.py + controller/petrel.py 20 actions + shell.py register & reconcile hook)
- `46e24f1` fixes validated on hardware (supervisor trusts Popen.pid; _pgrep adds "--"; _count_up_osds skips header; vhost.start drops unsupported --control-file, adds --num-queues)
- `9bf94f3` Phase 2 hyper-agent half: support/datastore/{lvm,petrel}.py + registry (6 backends: local/lvm/zfs/nfs/shared_fs/petrel)

VALIDATED ON REAL HW: single-node petrel cluster LIVE (mon@127.0.0.1:6790 + osd.0 + osd.1, fsid 1c92fd5e-5b9e-4675-8788-542ca884299b, pool nas_data id=1, /var/lib/petrel/cluster.json, test-vol-0 1GiB exists). status() returns osds_up=2, osd_status both True, HEALTH_OK. vhost-user-blk export/stop PASS. LvmBackend create/destroy on ubuntu-vg PASS (VG free 58→38→58 clean).

DONE: Phase 0 (bins), Phase 1 (hyper-agent mgmt+validated), Phase 2 hyper-agent half (datastore backends).
TODO next (in order):
1. Phase 2 ferrovirt half — controller/service Rust. storage_net.rs:160 create_datastore_task(state,user,req:Value): add type=lvm & type=petrel to validation + agent param assembly (lvm: vg|devices,size_bytes,fs_type; petrel: pool_name,create_pool) + persist datastore row. Add controller/vcenter/petrel.rs REST routes for cluster create/status/destroy + host/osd add/remove + pool, wired in controller/vcenter.rs routes(), via TaskManager + dispatch::agent_on(state,&target,action,params,secs) (target = dispatch::resolve_explicit_or_self). datastore table cols: uuid,name,type,mount_path,total_bytes,used_bytes,free_bytes,shared,status,create_time(+zpool_id,partition_id,cluster_id). Build ferrovirt with PATH=/root/.cargo/bin:$PATH (cargo NOT on default PATH!).
2. Phase 3 — VM create on petrel disks. vm_create.rs:312 disk path calc, ~1331 datastore resolve. petrel disk → type=block, block_dev=vhost sock, no qcow2; VM home(config.xml) must be lvm/local datastore. hyper-agent guest.py create_vm skip qcow2 for petrel + call petrel_vhost_start; qemu_command.py add vhost-user-blk branch + shared memfd (-object memory-backend-memfd,share=on REQUIRED for vhost-user).
3. Phase 4 lifecycle, Phase 5 UI (deferred).

ENV WARNING for next session: this session had severe Bash/Read output instability (stdout dropped/garbled/delayed, parallel batches cancelled, Writes silently lost). Work ONE tool call per turn for critical ops; re-verify every Write landed (ls/wc); avoid `pkill -f petrel` (matches the Bash tool's own snapshot argv → kills the command, exit 144) — kill petrel by explicit PID instead.

## STATUS 2026-05-31 (Phases 0-2 DONE, committed+pushed)
hyper-agent repo main: 7324e51 (P1 mgmt API) + 46e24f1 (fixes, hw-validated) + db0c818 (P2 lvm/petrel datastore backends). PUSHED.
ferrovirt repo main: eb83791 (P2: lvm/petrel datastore types in storage_net.rs create_datastore_task + 10 petrel_* cluster-mgmt service fns + controller/vcenter/petrel.rs REST routes). PUSHED. cargo check clean.
petrel cluster LIVE on host: mon@127.0.0.1:6790 + osd.0 + osd.1, fsid 1c92fd5e-5b9e-4675-8788-542ca884299b, pool nas_data id=1, /var/lib/petrel/cluster.json. test-vol-0 1GiB in pool.

### ferrovirt BUILD ENV (one-time host setup, NOT in repo — redo if env resets):
1. `ln -sf /usr/lib/x86_64-linux-gnu/libclang-21.so.1 /usr/lib/x86_64-linux-gnu/libclang.so`
2. PAM headers (apt broken by qemu deps → extract manually): `cd /tmp && apt-get download libpam0g-dev && dpkg-deb -x libpam0g-dev_*.deb /tmp/pamdev && cp -r /tmp/pamdev/usr/include/security /usr/include/`
3. build: `cd ferrovirt && PATH=/root/.cargo/bin:$PATH LIBCLANG_PATH=/usr/lib/x86_64-linux-gnu BINDGEN_EXTRA_CLANG_ARGS=-I/usr/lib/gcc/x86_64-linux-gnu/15/include cargo check`
Rust gotcha: TaskObj::exclusive(a,b,c) 2nd arg impl Into<String> — pass &str not .into() (E0283). VcenterResponse::of(x) works fine.

### petrel REST API (ferrovirt :8080) now available:
- /api/vcenter/storage/petrel/cluster POST{meta_disk,data_disks[],replica?,pg_num?,pool?,host?} GET?host= DELETE{wipe?}
- .../petrel/hosts POST{bucket_id,name,parent?} DELETE{bucket_id,osd_ids[]}
- .../petrel/osds POST{osd_id,host_id,weight}; .../osds/{osd}/out POST; .../osds/{osd} DELETE
- .../petrel/pools GET?host= POST{name,size?,min_size?,pg_num?,ec_k?,ec_m?}
- POST /api/vcenter/datastore type=lvm{vg,size_bytes,fs_type=xfs | devices,vg_name} or type=petrel{pool_name?,create_pool?,...}

## PHASE 3 PLAN (NEXT — VM create on petrel block disks + LVM config home)
Goal: VM virtual disk(s) on petrel (raw block via vhost-user-blk); config.xml on lvm/local datastore (petrel has no FS).
hyper-agent guest.py create_vm (CONFIRMED current structure, line refs):
- line 66: `vm_datastore`=param.get("vm_datastore") = the HOME datastore segment (config.xml lands at /datastore/<vm_datastore>/<uuid>/config.xml). line 70: vm_datastore_type. **This already decouples VM home from per-disk datastore** — good. For petrel VMs, vm_datastore must be an lvm/local datastore (NOT petrel).
- line 84-111: per-disk loop creates qcow2 via image.create_img using d["path"]. NEED: skip this for petrel disks (d has e.g. "backend":"petrel" or "block_dev"/no "path"); instead call petrel volume create + petrel_vhost_start → get /run/petrel/<vol>.sock, stash sock on the disk for config_builder.
- line 114: config_builder.build_guest(param) → to_xml → config.xml. The guest XML disk needs a vhost-user-blk representation.
qemu_command.py (CONFIRMED): QemuLauncher class. _build_disks line 678 → _build_disk line ~683 (-blockdev + -device per gc.LibvirtConfigGuestDisk). _build_memory line 521. _build_machine 441.
- NEED: vhost-user-blk branch — when a disk is petrel/vhost type, emit `-chardev socket,id=chr-<n>,path=<sock>` + `-device vhost-user-blk-pci,chardev=chr-<n>,id=<dev>[,num-queues=N]` instead of -blockdev/-device. AND _build_memory must use shared memfd when ANY vhost-user disk present: `-object memory-backend-memfd,id=mem,size=<MB>M,share=on` + `-numa node,memdev=mem` (or -machine memory-backend=mem). Check _build_memory current output first.
- guest_config.py: LibvirtConfigGuestDisk likely needs a new source type or a passthrough field for vhost socket; OR handle petrel disks outside the libvirt-xml model (simplest: a side list param["vhost_disks"]=[{sock,target_dev,bus,queues}] that QemuLauncher reads directly, bypassing guest XML). DECIDE during impl.
- start_vm/stop_vm: spawn/stop petrel-vhost per petrel disk around qemu launch (vhost socket must exist before qemu starts; vhost daemon already managed by PetrelController.petrel_vhost_start, but VM lifecycle should drive it).
ferrovirt vm_create.rs: disk path calc ~line 312, datastore resolve ~1331. petrel disk → type=block, no qcow2 path, alloc petrel vol; ensure VM home datastore is lvm/local. CreateVmRequest already has per-disk datastore + top-level datastore (home).
Break Phase 3 into: (3a) agent qemu_command vhost-user-blk + shared-mem support (unit-testable via build_command_line); (3b) agent guest.py create_vm petrel-disk branch + start/stop_vm vhost mgmt; (3c) ferrovirt vm_create petrel-disk DB/param wiring; (3d) E2E real VM on petrel disk.
