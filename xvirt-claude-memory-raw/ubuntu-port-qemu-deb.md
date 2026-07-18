---
name: ubuntu-port-qemu-deb
description: Ubuntu 26.04 port — QEMU deb packaging lives in build-deb/; configure-flag source of truth + gotchas
metadata: 
  node_type: memory
  type: project
  originSessionId: ab63e8f4-0873-4e1b-b66f-392a858b6266
---

xvirt is being ported from Rocky/RHEL-10 to **Ubuntu 26.04 LTS** (dev box is already 26.04). The QEMU deb packaging is done and lives in the **`build-deb/`** sibling repo (`ssh://git@192.168.2.107:2222/liuwfng/build-deb.git`): `debian/{control,rules,*.install,...}` + `build_deb.sh` (counterpart of the qemu repo's `pkg/build_rpm.sh`) + `README.md`.

**Configure-flag source of truth**: `build-rpm/` (with its modified spice/virgl spec) is GONE — never committed, lost in the move to Ubuntu. The surviving truth is inside the QEMU fork repo (`ssh://git@192.168.2.107:2222/liuwfng/qemu.git`, branch `stable-10.0-dev`, VERSION 10.0.7, cdp already in-tree) at **`pkg/qemu-kvm.spec`** + `pkg/build_rpm.sh`. That committed spec is stock RHEL-10 (only `--enable-spice-protocol`); the deb adds `--enable-spice --enable-virglrenderer` (the variant the last RPM shipped, per user).

**deb decisions** (confirmed w/ user): package split mirrors the Ubuntu archive qemu source (9 pkgs: qemu-system-x86 / -common / -data / -gui / -modules-opengl / -modules-spice / qemu-utils / qemu-block-extra / qemu-guest-agent); only x86_64-softmmu; deps (spice-server 0.16.0 / virglrenderer 1.2.0) from Ubuntu archive, NOT rebuilt from source.

**Gotchas baked into debian/rules**:
- Dropped RHEL-only flags (break non-RHEL build / clash w/ Ubuntu layout): `--without-default-devices`, `--with-devices-x86_64=x86_64-rh-devices` (config file doesn't exist in-tree), `--rhel-version=10`, `--with-suffix=qemu-kvm`, `--interp-prefix`, `--enable-selinux`, `--enable-werror`.
- **`--enable-mpath` → `--disable-mpath`**: Ubuntu's `libmpathpersist-dev` ships only `mpath_persist.h`, not the `mpath_cmd.h` that `qemu-pr-helper.c` includes → build fails otherwise (Debian's qemu drops it too).
- Ubuntu paths: `--prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu --libexecdir=/usr/lib/qemu`, modules under `/usr/lib/<triplet>/qemu/`.

**hyper-agent compat**: `hyper_agent/support/qemu_command.py:37` hard-codes `_DEFAULT_EMULATOR='/usr/libexec/qemu-kvm'` (RHEL path). The `qemu-system-x86` deb ships a `/usr/libexec/qemu-kvm` → `/usr/bin/qemu-system-x86_64` symlink so hyper-agent works unchanged. Verified E2E: VM boots via that path with kvm+virtio-vga-gl+egl-headless+spice.

**Component debs + installer ISO** live in `build-deb-xvirt/` (main repo, separate from the qemu `build-deb/`). `build-xvirt-debs.sh` builds 5 `Architecture: all` debs via staging + dpkg-deb: `xvirt-init` (first-boot deployer = `/usr/libexec/xvirt-init`, the only boot-enabled service; 9 phases per PACKAGING.md §10 but XFS not OCFS2, ufw not firewalld), `xvirt-hyper-agent` (python→dist-packages + hyper-{agent,vm,qmp} + data files), `xvirt-virt-control` (gradle bootJar→/opt/xvirt + config + db-init), `xvirt-client` (npm dist→/usr/share + nginx vhost), `xvirt-meta` (pulls all + qemu, Provides xvirt-conf). `iso/build-iso.sh <ubuntu-26.04-live-server.iso>` remasters an autoinstall ISO: bakes `/xvirt-bundle` apt repo (all 14 debs + dpkg-scanpackages index) + `/nocloud` seed (`iso/autoinstall/user-data`), grub default = `autoinstall ds=nocloud;s=/cdrom/nocloud/`. Full stack dep-closure resolves rc=0 vs Ubuntu archive (apt simulate). **blivet fix**: `hyper-agent/.../support/storage_util.py` made blivet a lazy `@property` (python3-blivet not on Ubuntu) — agent now starts without it. Default ISO login xvirt/xvirt.

Build: `cd build-deb && ./build_deb.sh /tmp/qemu-inspect HEAD` (or a qemu checkout / git URL). `KEEP_WORK=1` preserves the temp build tree for inspecting `debian/tmp` / re-running `dpkg-buildpackage -nc`. Artifacts → `build-deb/out/*.deb`. Build-deps: see README. usbredir dev pkgs on Ubuntu = `libusbredirhost-dev` + `libusbredirparser-dev`; spice-protocol = `libspice-protocol-dev`.
