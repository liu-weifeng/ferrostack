---
name: xvirt-components-and-xvirtctl
description: "xvirt component map (don't re-explore code structure) + xvirtctl systemd service manager — build/deploy only the named component"
metadata: 
  node_type: memory
  type: project
  originSessionId: b5803e41-db51-453d-ad92-34f7963f863c
---

**Read this BEFORE exploring xvirt's code structure — the component map is authoritative, saves re-tracing every session.** Mirrors the 速查表 at top of `/root/code/xvirt/CLAUDE.md`.

## Components (user-confirmed 2026-06-01)
- **xvirt-client** — React/Vite SPA web console. TWO UIs by `role`: **center** (multi-host management console) + **standalone** (single-host management). Served as static `dist/` by nginx (Vite :5173 only in dev).
- **ferrovirt** — Rust management/control-plane service. Upper-layer resource + task management, REST `/api/*` + WS, sole owner of MariaDB `mx`. **Replaced Java virt-control** (same :8080, can't coexist). Reads `.env` from cwd. Build needs LIBCLANG_PATH + PAM headers + BINDGEN_EXTRA_CLANG_ARGS (see [[petrel-phase3-plan]]).
- **hyper-agent** — Python asyncio host-level service. Directly manages VMs/storage-pools/network (builds QEMU cmdline, QMP, sysfs, qemu-img, nmcli, ovs-vsctl). NO libvirt, NO DB. Supervises petrel daemons (`support/petrel/`). ZMQ ROUTER :5556 / event PUB :5558 / file :5557.
- **lictor** — Python; holds QMP + event PUB (:5560/:5561). hyper-agent thin-proxies vm/snapshot/xvt actions to it. MUST start before hyper-agent.
- **concilium** — C daemon, cluster resource management (drives xvirt-vm RA → lictor). `/run/conciliumd.sock`. Single-node `--bootstrap`.
- **petrel** — Rust distributed storage (single/multi-node; block volumes to VMs via vhost-user-blk). CLIs `petrel-{mon,osd,deploy,ublk,vhost}` → /usr/local/bin. Daemons reconciled by hyper-agent — NO own systemd unit.
- **aegis-sdk** — C library, VDDK-like disk read/write SDK for backup. `libaegisdisk.so` + `aegis.h` + `aegis-cbt`. NOT a service.
- **xvirt-tools** — Go binary installed INSIDE guest VMs (virtio-serial `xvirt-tools.0`).
- Other lower-level/side: cdp-worker, krait, lycan, vadk, imgtools, snapshot-helper, xvirt-libs, monitoring.
- **virt-control** (Java/Spring) = DEPRECATED, replaced by ferrovirt; code still in tree but never started.

## xvirtctl — `/root/code/xvirt/xvirtctl` (systemd service manager, the PRIMARY tool)
Goal: services via systemd; **only rebuild/reinstall/restart the component you name — don't touch others** (saves tokens, precise).
- `./xvirtctl list` / `status [comp]` / `start|stop|restart [comp|all]` / `enable|disable` / `logs <comp> [-f]`
- `./xvirtctl build <comp>` — compile only (ferrovirt=cargo release w/ libclang env; petrel=cargo 5 bins; concilium=make; xvirt-client=npm build; aegis-sdk=make; lictor/hyper-agent=python no-build).
- `./xvirtctl deploy <comp|all>` — build → install to system path → write/refresh unit → restart. **After editing a component, run `./xvirtctl deploy <comp>`** (e.g. `deploy ferrovirt` only rebuilds+restarts ferrovirt). petrel deploy → install CLIs + restart hyper-agent. xvirt-client deploy → npm build + sync dist + restart nginx.
- `./xvirtctl install-units [comp]` — write systemd units + daemon-reload only.
- `./xvirtctl role [center|standalone]` — /etc/xvirt/role.conf.
- service names: xvirt-lictor / xvirt-hyper-agent / xvirt-ferrovirt / conciliumd / nginx. Order: lictor→hyper-agent→ferrovirt→concilium→nginx.
- Units generated inline by the script (dev-machine: run from source via WorkingDirectory + python3 -m / cargo target bin). On dev box hyper-agent/lictor have no /usr/bin entry → unit runs `python3 -m hyper_agent.shell` / `lictor.shell` with WorkingDirectory.
- Ports: 80/443 nginx, 8080 ferrovirt, 5556/5557/5558 agent, 5560/5561 lictor, 6790 petrel-mon, 5173 vite(dev).

## deb packaging status
`build-deb-xvirt/build-xvirt-debs.sh` builds component debs (xvirt-init/hyper-agent/virt-control/client/meta) — **STILL packages Java virt-control, NOT ferrovirt; no petrel/concilium/aegis deb yet.** QEMU deb in sibling `build-deb/`. See [[ubuntu-port-qemu-deb]]. To fully deb-ify the new stack: add xvirt-ferrovirt/xvirt-petrel/xvirt-concilium packages + swap meta dep virt-control→ferrovirt (future task). For now `xvirtctl install-units` runs everything from source via systemd on the dev box.

Old launch scripts xvirt.sh (pidfile, has deprecated virt-control) + xvirt-ferrovirt.sh (pidfile, ferrovirt) kept as dev fallback; new work uses xvirtctl.
