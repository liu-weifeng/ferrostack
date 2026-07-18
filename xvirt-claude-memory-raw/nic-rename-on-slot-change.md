---
name: nic-rename-on-slot-change
description: "TODO + gotcha — PCI slot changes rename NICs (enpXsY), breaking netplan/br-ex uplink; pin names by MAC"
metadata: 
  node_type: memory
  type: project
  originSessionId: b5803e41-db51-453d-ad92-34f7963f863c
---

On real node 192.168.2.123 (xvirt-standalone), adding/moving a PCIe card shifts PCI
bus enumeration and **renames the NIC** (was `enp5s0` at install → `enp7s0` after an
NVMe slot change). This breaks the install-time netplan + OVS br-ex, whose uplink
points at the old name → br-ex loses its uplink, NIC falls back to dracut-default DHCP.

Symptom: `ovs-vsctl show` → `could not open network device enp5s0 (No such device)`;
IP lands on the bare NIC via DHCP instead of br-ex.

Manual fix applied 2026-06-05: rewrote `/etc/netplan/01-netcfg.yaml` to point br-ex at
the new NIC name + static IP; updated `/etc/xvirt/default-bridge.conf`.

**TODO (not yet done):** make NIC naming immune to slot changes — emit a systemd
`.link` file (or net.ifnames=0 + udev rule) that pins the interface name by MAC at
install time (lycan `install.py` / xvirt-init), so br-ex uplink survives hardware
moves. Until then, any PCI slot change needs a manual netplan uplink-name fixup.

Related: [[lycan-autoinstall-iso]] (installer writes the netplan + br-ex).
