---
name: dev-box-bringup-state
description: "Dev box (Ubuntu 26.04) bring-up via xvirtctl source mode — apt is wedged by a half-finished transaction; manual dep fixes applied; what's up vs blocked"
metadata: 
  node_type: memory
  type: project
  originSessionId: b5803e41-db51-453d-ad92-34f7963f863c
---

Bringing up the xvirt stack on the dev box in **xvirtctl source mode** (no deb install of components — see [[xvirt-components-and-xvirtctl]]). Started 2026-06-01.

## ✅ BRING-UP COMPLETE (2026-06-01 ~03:52). All 5 services up + enabled at boot.
- ✅ **xvirt-lictor** — active, :5560/:5561. Source unit (WorkingDirectory=lictor, python3 -m lictor.shell).
- ✅ **xvirt-hyper-agent** — active, :5556/:5557/:5558. Source unit (python3 -m hyper_agent.shell). Reconciles live petrel.
- ✅ **xvirt-ferrovirt** — active, :8080 + node-zmq :5570/:5571. role=Standalone. Connected to hyper-agent + event sub. Source unit (target/release/ferrovirt, WorkingDirectory=ferrovirt for .env).
- ✅ **nginx** — active, :80/:443. Serves PROD conf (build-deb-xvirt/assets/xvirt-nginx.conf → /etc/nginx/conf.d/xvirt.conf): static SPA from /usr/share/xvirt-client/dist + /api→:8080 + /websocket→:8080 + /vnc/<port>. Self-signed cert at /etc/pki/tls/{certs/xvirt.crt,private/xvirt.key} (gen by nginx/setup-https.sh), symlinked to /etc/ssl/{certs,private}/xvirt.{crt,key} for the prod conf. Removed Ubuntu stock /etc/nginx/sites-enabled/default (default_server clash).
- ✅ **mariadb** 11.8.6 — active, :3306, enabled. DB `mx` + user mx/test created. 53 JPA tables bootstrapped by running the Java virt-control jar ONCE then killing it.
- ⚪ **concilium** — inactive (not built/started; optional cluster layer — start later if needed).
- petrel cluster LIVE (mon@:6790 + osd.0 + osd.1, 2/2 up).
- E2E verified: nginx→ferrovirt→PAM chain works (login endpoint reaches PAM; SPA root HTTP 200 title "xvirt — 主机管理控制台"; /api/auth/login hits ferrovirt PAM correctly). NOTE: dev_credentials.md memory referenced in CLAUDE.md does NOT exist here — couldn't test a successful login, but PAM rejection proves the wiring.

## Daily ops now: `./xvirtctl status` / `restart <comp>` / `logs <comp> -f` / `deploy <comp>` (rebuild+restart only that comp).

## (historical) original status before bring-up:

## Manual dependency fixes applied this session (apt is wedged — extracted debs by hand, no apt transaction):
1. **Cryptodome** (hyper-agent hostid.py needs `Crypto`/`Cryptodome`): `apt-get download python3-pycryptodome` → `dpkg-deb -x` → `cp -a .../dist-packages/Cryptodome /usr/lib/python3/dist-packages/`. Fixed crashloop.
2. **libzmq dev** (ferrovirt zmq-sys build needs pkg-config libzmq; only runtime libzmq.so.5 was present): `apt-get download libzmq3-dev` → extract → copy `libzmq.so` symlink + `libzmq.pc` + headers to /usr/lib/x86_64-linux-gnu + /usr/include. THEN **patched /usr/lib/x86_64-linux-gnu/pkgconfig/libzmq.pc**: stripped `Requires.private: krb5-gssapi libsodium openpgm-5.3 norm libbsd` → `Requires.private:` and `Libs.private: -lstdc++` (those are static-link-only deps not installed; dynamic link doesn't need them). After that `pkg-config --modversion libzmq`=4.3.5 and ferrovirt built.
3. ferrovirt build env (recap): `PATH=/root/.cargo/bin:$PATH LIBCLANG_PATH=/usr/lib/x86_64-linux-gnu BINDGEN_EXTRA_CLANG_ARGS=-I/usr/lib/gcc/x86_64-linux-gnu/15/include` + libclang.so symlink + PAM headers (see [[petrel-phase3-plan]]).

## ROOT BLOCKER: apt globally wedged
The box is mid-way through an **interrupted apt transaction**: ~100 packages in `iU` (unpacked-not-configured), incl. the whole Ubuntu-archive qemu set (qemu-system-x86 1:10.2.1+ds-1ubuntu3 etc). The running `/usr/bin/qemu-system-x86_64` is ALREADY archive 10.2.1. The user's **custom QEMU build** (qemu-system-common/gui 10.0.7+git17b4742) is stuck in `iF` (failed-config). Any `apt install` / `apt --fix-broken` is refused until this resolves. `/usr/libexec/qemu-kvm` (which hyper-agent's `_DEFAULT_EMULATOR` at qemu_command.py:37 expects) is currently MISSING.

## USER DECISION (2026-06-01): rebuild + keep the CUSTOM QEMU (10.0.7), do NOT accept archive 10.2.1.
- Custom QEMU fork is cloned at **/root/code/xvirt/qemu** (branch `stable-10.0-dev`, HEAD 17b47425f1 = matches the iF'd custom debs). Remote `ssh://git@192.168.2.107:2222/liuwfng/qemu.git`.
- deb packaging in **/root/code/xvirt/build-deb** (`build_deb.sh [QEMU_SRC] [COMMIT]`, FORK_BRANCH=stable-10.0-dev). 9 prebuilt custom debs already in `build-deb/out/*_10.0.7+git17b4742_*.deb`. Build tools + dev libs (spice-server 0.16.0, virgl, epoxy/gbm/drm) already installed — build does NOT need apt.
- PLAN: rebuild custom qemu debs (`./build_deb.sh /root/code/xvirt/qemu HEAD`, ~15min) → `dpkg -i build-deb/out/*_10.0.7+git17b4742_*.deb` to overwrite the iF/archive qemu with custom 10.0.7 → that completes/unwedges the transaction → then `apt --fix-broken install` (now consistent) → install mariadb-server + nginx → start ferrovirt + nginx via xvirtctl → add /usr/libexec/qemu-kvm symlink for hyper-agent.
- IN PROGRESS: qemu deb rebuild running (log /tmp/qemu-deb-build.log).

## Next steps after qemu rebuilt+installed:
1. `dpkg -i /root/code/xvirt/build-deb/out/qemu-*_10.0.7+git17b4742_*.deb` (+ qemu-system-data _all.deb).
2. ensure `/usr/libexec/qemu-kvm` → `/usr/bin/qemu-system-x86_64` symlink exists (hyper-agent needs it).
3. `apt-get -f install` to settle the rest of the iU packages.
4. install mariadb-server + nginx (now apt works). **DB schema bootstrap (IMPORTANT)**: ferrovirt does NOT create the core JPA tables — it only `CREATE TABLE IF NOT EXISTS` for its own task_object/host_enrollment/host_membership/storage_net additions (main.rs:64-72). The ~52 core tables (host/vm/datastore/vm_config/vm_disk/...) come from the legacy Java virt-control JPA `ddl-auto=update`. migrations/README says: on a fresh box, run Java virt-control once to build the schema, OR import an mx dump. **Java jar IS built** at virt-control/build/libs/virt-control-0.0.1-SNAPSHOT.jar + Java 25 present. So: create DB `mx` + user mx/test (db-init.sql template at build-deb-xvirt/assets/db-init.sql), then run the jar ONCE (`java -jar ... --spring.config...` pointing at mx) to create all tables, stop it, THEN start ferrovirt. (No mx dump exists on disk.)
5. `./xvirtctl deploy ferrovirt` (or install-units + start) and `./xvirtctl start nginx`.
6. `./xvirtctl status` should show all 5 active.
