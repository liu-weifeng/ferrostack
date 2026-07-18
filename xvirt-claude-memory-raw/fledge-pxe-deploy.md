---
name: fledge-pxe-deploy
description: fledge = new PXE provisioning service (xvirt repo) on gateway 192.168.2.1; upload ISO → netboot+autoinstall nodes with per-MAC static IP/disk/role. Full chain validated in a VM.
metadata: 
  node_type: memory
  type: project
  originSessionId: 2275a03b-6714-4f8e-a88a-b99c286e621e
---

**Task (2026-06-17)**: PXE-deploy the two compute nodes (123/106) + center + distributed storage via a reusable, ISO-driven PXE service on the gateway. User steers: test in a VM first; rebuild ISO for PXE + updated packages; keep each disk's role identical on reinstall.

## fledge — the new service (`/root/code/xvirt/fledge/`)
**Now its own standalone repo** (2026-06-18): pushed to `ssh://git@192.168.2.107:2222/liuwfng/fledge.git` (branch main) and removed from the xvirt monorepo's tracking (parent commit `cd4309e` `git rm --cached`d it). Files stay on disk as a nested repo, like petrel/lycan/aegis-sdk — edit there, push to its own remote.

Single-file stdlib-only Python service (no pip), runs as root on the **gateway 192.168.2.1**. Turns an uploaded xvirt ISO into a netboot source + per-machine (by NIC MAC) config: static IP / root pw / **target system disk** / role / image. Generates iPXE script + lycan answer YAML on the fly; serves whole ISO via os.sendfile (Range).
- `fledge.py` (UI+API+static, port 8068), `install-gateway.sh` (additive dnsmasq PXE + iPXE binaries + systemd), `fledge.service`, `README.md`.
- **Deployed on gateway**: `/opt/fledge/fledge.py` + `fledge.service` (active), `/srv/fledge/{images,answers}`, `/srv/tftp/{undionly.kpxe,ipxe.efi(=snponly.efi from ubuntu ipxe deb)}`. UI http://192.168.2.1:8068/.
- **Gateway dnsmasq is the DHCP server** (`/etc/dnsmasq.d/lan.conf`, range .100-.200). PXE added PURELY ADDITIVELY in `/etc/dnsmasq.d/pxe.conf` (enable-tftp + arch-match iPXE chainload → http://gw:8068/boot.ipxe). `enable-tftp` needs dnsmasq **restart** (not reload — SIGHUP won't open the TFTP socket). `dnsmasq --test` gates it. DNS/DHCP verified intact after. fledge writes per-node `dhcp-host` pins to `/etc/dnsmasq.d/fledge-hosts.conf` + reloads.

## Boot chain (VALIDATED end-to-end in a VM on node 106, BIOS/SeaBIOS path)
firmware PXE → DHCP(gw) → TFTP undionly.kpxe → iPXE → http://gw:8068/boot.ipxe → /ipxe?mac=${mac} → kernel/initrd + `boot=casper ip=dhcp url=http://gw:8068/images/<img>.iso lycan.auto lycan.answers=http://gw:8068/answers/<mac>.yaml` → casper fetches whole ISO to RAM (needs ≥~6GB RAM; gave VM 8G) → lycan reads per-MAC answers over HTTP (lycan ALREADY supports http answers URL, autoinstall.py) → installs to the named disk with static IP. Test VM came up `role=standalone host=xvirt-pxetest disk=/dev/vda (manual) reboot=True`, partitioned /dev/vda. Test harness on 106: tap `tap-pxetest` on br-ex, qemu disk bootindex=1(empty→fallback)/NIC bootindex=2 (self-terminating, no reinstall loop), serial→/datastore/<ds>/pxetest/serial.log, VNC :9.

## Rebuilt ISO `/root/pxe-build/xvirt-pxe-26.04.iso` (3.0G, on gateway as /srv/fledge/images/xvirt-26.04.iso)
- Built with lycan `scripts/build-iso.sh <base26.04> --role standalone --bundle <offline-bundle>`. **One ISO serves BOTH roles** — lycan reads role from the per-MAC answer file, not the baked one.
- Components rebuilt from current source → debs `git189d7e8` (build-deb-xvirt/out, old versions stashed in out/_old). qemu REUSED = `2:10.0.7+git8aaf00193b` (matches what nodes run, build-deb/out, others in _old). Offline bundle = `build-deb-xvirt/offline-bundle` (294 debs, 228M, full dep closure incl openssh/avahi).
- Build box = 112; was 100% full → freed by deleting old ISOs + `rm -rf petrel/target` (8.8G, rebuildable). ISO build needs ~8G.

## Node disk roles (MUST preserve on reinstall) — see lab-env/README.md
- **123** (sys disk **nvme0n1**, MAC enp7s0 `00:e0:26:78:22:69`, static .123): nvme1n1=petrel meta(Optane), sda-sdd+nvme2n1=petrel data(osd0-4), mon host.
- **106** (sys disk **sda**, MAC enp6s0 `22:46:4d:02:83:89`, →static .106): nvme0n1=petrel meta(Optane), sdb-sde+nvme1n1=petrel data(osd5-9).
- petrel pool=testpool replica=2 fsid 395a8c1e-54d3-43bb-91c7-d2f8c5c80826. autoinstall MUST set target_disk explicitly (auto picks largest=465G HDD = WRONG).

## ✅ DONE (2026-06-17) — full deploy complete
- 106(.106)+123(.123) PXE-reinstalled via fledge (legacy BIOS/undionly.kpxe), correct system disks (sda/nvme0n1), petrel data disks preserved, full stack active.
- center = NEW management VM on 123 (systemd `xvirt-center-vm`, **8GB RAM** — 6GB failed: casper OOM'd downloading 3GB ISO into tmpfs), static .121, **root pwd now KNOWN = weifeng**. MAC 52:54:00:ce:01:21.
- both nodes enrolled to center (CONNECTED): 123=7c8ca16a.. 106=5f44536d..
- petrel rebuilt fresh (new fsid 9abf3492..), mon@123, replica=2, **10/10 OSD HEALTH_OK** (hdd 8/8 + nvme 2/2), registered datastore `petrel-pool`. Same disk roles (Optane meta, 4HDD+1SSD data/node).

## Key gotchas learned
- NVMe `/dev` names REORDER across reinstall → map petrel disks by MODEL (Optane=INTEL MEMPEK=meta), re-check each deploy.
- fledge one-shot is consumed by ANY `/ipxe` fetch (incl. verification curls) → RE-ARM (re-POST profile) right before booting a node; verify via `/answers/<mac>.yaml` (safe), never `/ipxe`.
- dnsmasq: `enable-tftp` AND main-config `dhcp-host` need RESTART not reload; per-node IP pins must use a `dhcp-hostsfile` (reload re-reads it). install-gateway.sh uses `systemctl restart fledge` (enable --now won't restart a running svc).
- casper netboot (`url=<iso>`) loads whole 3GB ISO into RAM → VM needs ≥8GB RAM.
- pkill -f "name <vm>" matches its own ssh shell → self-kills; match qemu by comm or PID.
Related: [[lycan-autoinstall-iso]] [[state-reconcile-and-lab-env]] [[petrel-phase3-plan]].
