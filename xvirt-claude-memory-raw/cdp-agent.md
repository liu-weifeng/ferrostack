---
name: cdp-agent
description: "In-guest CDP agent (cdp-agent/) — intercepts guest write IO and ships it over TCP, independent of QEMU (works on VMware/bare-metal too)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 1ea055a7-bd83-4806-9f78-1540f1774d1e
---

`cdp-agent/` is a NEW subsystem (built 2026-06-23). **Own repo** (like fledge/aegis-sdk/
xvirt-autotest): remote `ssh://git@192.168.2.107:2222/liuwfng/cdp-agent.git`, branch `main`
(first commit dae82f3); lives as an UNTRACKED nested dir in the xvirt monorepo (not tracked
by it). It is an **in-guest** Continuous Data Protection agent — the QEMU-independent counterpart to the existing `qemu/block/cdp.c`
(IO redirection) + `cdp-worker`. Intercepts write IO INSIDE the guest OS and streams
records over TCP, so it covers VMware / Hyper-V / bare-metal, not just KVM.

Layout:
- `proto/cdp_proto.h` — shared C contract (relay record + wire frames), single source of truth for all 3 codebases.
- `agent/` — **Rust, std-only (zero deps)**, the heart. `core.rs` = degradation state machine; `spool.rs` = segmented WAL journal; `bitmap.rs` = dirty bitmap; `transport.rs` = TCP w/ HELLO/ACK/reconnect; `capture/` = relay reader (linux file / windows DeviceIoControl FFI) + synthetic test source. Builds with `cd agent && cargo build`.
- `linux-km/` — device-mapper target `dm-cdp` (C): clones write bios on completion → kfifo → `/dev/cdp-relay`.
- `windows-driver/` — WDM disk upper filter `cdpflt` (C): IRP_MJ_WRITE completion → ring → inverted-call IOCTL `\\.\CdpFlt`.
- `tools/cdp-collector.py` — reference/test collector (framed + compat).

Key design decisions:
- **Three modes = cdp-worker's HIGH/MIDDLE/LOW** = 实时/增量/位图: realtime stream → incremental WAL drain → bitmap-only (drop payload, keep changed-extent tracking). Auto by journal backlog; LOW forced by spool-full or kernel-ring overflow; hysteresis on LOW exit.
- **Wire-compatible with cdp-worker**: `--wire compat` emits the exact `struct iodata`(iotime,offset,length,ioindex)+payload stream from `cdp-worker/cdpwork/cdp_socket.c:backup_send_iodata`. `--wire framed` (default) adds HELLO/ACK/LEVEL/BITMAP + at-least-once dedup by seq.
- **Never block guest IO**: 4-stage backpressure (kernel ring overflow→gap → bounded chan → WAL full → bitmap). Overflow gaps fold into dirty bitmap (CDP_F_OVERFLOW).
- durable-first: append to WAL before send; collector ACK reclaims segments.

**cdp-sink receiver service** (`sink/`, Rust std-only, reuses agent proto via `#[path]`) added 2026-06-23: ingests the wire protocol, fans out to combinable sinks — `iolog` (cdp-worker-compatible IO log+index+bitmap triple, byte-exact: log 24B hdr+payload, index 32B+ionum trailer, bitmap 16B extents, names `<name>_<serial>_<sync|fsync>_<ts>[_index|_bitmap]`), `disk` (apply to image = v2v/p2v), `replica` (apply to local block dev = DR). cdp-worker iolog `iooffset` = record-header-start (raw.c retoffset), time in SECONDS, log starts at byte 0.

VALIDATED on real hardware + real VM (lab node 123, Ubuntu 26.04 / kernel 7.0.0-22-generic — same as dev box 112 so a locally-built .ko loads on nodes/guests):
- `dm-cdp.ko` builds + loads (fixed: `<linux/types.h>` under `__KERNEL__`; `prepare_ioctl` 6.11+ sig; `no_llseek`→`noop_llseek`; vmalloc+kfifo_init for the 64MiB ring).
- Real kernel: dm-cdp captures loop-dev writes → agent → sink(disk) → md5-identical rebuild.
- **Real VM**: matching-kernel custom-initramfs VM (busybox+dm-cdp.ko+agent), in-guest dm-cdp captures virtio-disk writes → in-guest agent over virtio-net (qemu user-net) → host cdp-sink → disk-rebuild + cdp-worker iolog replay + guest disk all md5-identical.
- `tools/e2e_test.py` 5/5 byte-exact (disk/iolog/combo/compat/overflow→bitmap).
- **Windows cdpflt NOW validated on a real VM (2026-06-24, repo f9e6a5b)**: Windows Server 2022 VM created entirely via xvirt API + unattended autounattend install (drove SeaBIOS "press any key" via a hand-written RFB/VNC keystroke since lictor holds QMP; static IP .180; testsigning on; OpenSSH). cdpflt built in-guest with EWDK (16G ISO pulled with `aria2c -x16` to beat per-conn CDN throttle), self-signed sys+cat, installed as boot-start Disk-class UpperFilter (pnputil + `InstallHinfSection DefaultInstall`), reboot → `sc query`=RUNNING, attached to disk. Real disk writes captured (kernel) → agent relay (`\\.\CdpFlt` inverted-call IOCTL) → host cdp-sink → cdp-worker iolog; unique guest-written markers found in host iolog (backlog 109×, live 6×, clean self-enable 48×), 0 dropped. **Two bugs found+fixed by live testing (build-only couldn't)**: agent PULL buffer 1MiB < max record HDR+1MiB → returns 0 → agent mistakes for EOF + stalls (→2MiB); agent never issued SET_VOLUME so depended on external enabler (→self-enables SET_VOLUME(ALL) on open). vcxproj fixes: Universal→Desktop (INF DefaultUninstall), drop redundant _KERNEL_MODE, /WX off. Guest creds: Administrator / `Xvirt@CDP2026`, IP 192.168.2.180, VM uuid 3ebaac17-... on node 123.
Lab access: nodes 123/106 root pwd `weifeng` (sshpass), services active, debian12.qcow2 + busybox + matching vmlinuz on 123. Related: [[xvirt-components-and-xvirtctl]].
