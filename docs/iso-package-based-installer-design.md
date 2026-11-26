# xvirt 安装 ISO 重构：按包安装，同一 ISO 兼作升级源（设计稿）

> 状态：**草案，待定事项已部分拍板**（2026-11-25，见 §13）。
> 涉及仓库：`lycan`（安装器）、`build-deb` / `xvirt/build-deb-xvirt`（打包）、`fledge`（PXE）、`cicd`（流水线）、新建 `xvirt-distro`（发行版清单）。
> 一句话：仿照 RHEL 的 Anaconda 模式——ISO 本身就是一个**带签名的完整 apt 仓库**，安装器用 `mmdebstrap + apt` 从这个仓库把系统装到硬盘上（内核也是一个 deb）；装好的节点挂上同一个 ISO（或经 HTTP 访问同一仓库）即可 `apt full-upgrade` 升级。

---

## 1. 背景与现状

### 1.1 现在的 ISO 是怎么装系统的

现有 lycan ISO 基于 `ubuntu-26.04.1-live-server-amd64.iso` 改造（`lycan/scripts/build-iso.sh`）：

1. 引导 Ubuntu 的 casper Live 环境，lycan 注入在 `.installer` squashfs 层里，并屏蔽 subiquity。
2. lycan 把 `ubuntu-server-minimal` + `ubuntu-server` 两层 squashfs 用 overlay 叠起来，**整棵目录树复制到目标盘**（`_extract_base_system`）。
3. 用 ISO 里的离线 bundle（`build-offline-bundle.sh` 收集的依赖闭包，约 290 个 deb）通过 apt 安装 xvirt / qemu 等。
4. 重新生成 initramfs、安装 grub、写入 `/etc/xvirt/role.conf` 等首次启动配置。

### 1.2 存在的问题

| 问题 | 说明 |
|---|---|
| **不能作为升级源** | 基础系统和内核以镜像形式存在，ISO 里没有对应的 deb，`apt` 无法从 ISO 升级它们 |
| **依赖 Ubuntu Live ISO 的内部结构** | 分层 casper、overlay whiteout、subiquity/snapd 抢 tty1、casper 拦截重启、cloud-init、live initrd 导致的 kernel panic 等一系列问题，都来自"借用别人的 Live 盘" |
| **目标系统内容不受控** | 装进去的是 Ubuntu 预先做好的 server 镜像（含 snapd、cloud-init 等不需要的东西），再逐项禁用 |
| **按角色出多个 ISO** | standalone / management / compute 各一个 ISO（每个约 3.0G），bundle 按角色过滤 |
| **版本号不能比较大小** | xvirt deb 用 `0.0.1+git<hash>`，哈希不单调，apt 无法正确判断新旧 |

### 1.3 RHEL 的做法（参照对象）

- Anaconda 同样先引导 `vmlinuz + initrd`，再挂载 `LiveOS/squashfs.img`（install.img）作为安装环境运行。**"Live 启动安装环境"这一点两者没有区别。**
- 区别在于**目标根文件系统怎么来**：Anaconda 用 dnf 从 ISO 上的 BaseOS/AppStream 仓库安装 RPM（内核也是 RPM），因此 DVD 本身就是一个完整仓库，可以直接用于 `dnf upgrade`。
- Debian 系的对应物是 debian-installer（debootstrap + apt），Ubuntu 20.04 之前的 alternate ISO 也是这样；20.04 以后 Ubuntu 才改成 subiquity/curtin 复制镜像。

---

## 2. 目标与非目标

### 2.1 目标

1. **一个 ISO 覆盖全部角色**：standalone / management / compute 在安装时选择（或由应答文件指定）。
2. **ISO 是完整、自洽、带签名的 apt 仓库**：目标系统上每个已安装的包（含内核、grub、shim）都能在 ISO 的 `pool/` 中找到。
3. **按包安装**：目标系统由 `mmdebstrap` + `apt` 从 ISO 仓库装出，dpkg 状态从一开始就是干净一致的。
4. **同一 ISO 可作为升级源**：单机挂载 ISO 升级；集群由管理节点通过 HTTP 发布同一仓库，计算节点在线升级。
5. **完全离线**：安装和升级都不需要访问外网；**ISO 构建也不访问外网**，只读本地镜像源（§4.1）。
6. **可复现**：每个 ISO 对应 `xvirt-distro` 仓库的一个 commit + 一个 Ubuntu 快照时间点 + 一组 xvirt 包版本。
7. **保留 lycan 现有能力**：TUI 向导、应答文件、网络/磁盘/账号配置、PXE（fledge）流程不变。

### 2.2 非目标（本期不做）

- 不自建完整的 Ubuntu 镜像站，也不重新编译 Ubuntu 的包。
- **不做升级回滚**：不做 A/B 分区、ostree、LVM/btrfs 快照等回滚机制（§10.5）。
- **不支持已部署的老节点**：用"复制镜像"方式装出的现有节点不迁移、不能用新 ISO 升级，只支持新方案安装的节点。
- 不改变 xvirt 各组件本身的功能。

---

## 3. 总体方案

```mermaid
flowchart LR
  U["Ubuntu 官方源<br/>resolute / -updates / -security"] -->|aptly mirror + snapshot| A["Ubuntu 快照<br/>（时间点固定）"]
  G["各组件 git 仓库<br/>ferrovirt / lictor / petrel / qemu / lycan ..."] -->|CI 构建 deb| X["xvirt 自有源<br/>（aptly local repo）"]
  D["xvirt-distro<br/>元包 / 种子清单 / 构建脚本"] -->|构建元包、keyring、release 包| X
  A --> M["合并快照 + 计算依赖闭包"]
  X --> M
  D -->|种子清单| M
  M --> P["ISO 仓库<br/>dists/ + pool/<br/>xvirt GPG 签名"]
  P --> R["mmdebstrap 构建安装环境<br/>live squashfs（含 lycan）"]
  P --> I["xorriso 打包 ISO<br/>BIOS + UEFI(Secure Boot)"]
  R --> I
  P -.同一份仓库.-> H["HTTP 发布<br/>（fledge / 管理节点）"]
```

关键点：

- **只有两类上游**：Ubuntu 快照（基础包）和 xvirt 自有源（自研和第三方包）。
- **git 里只放源码和清单，不放 deb**。deb 是 CI 产物，进 apt 仓库。
- **安装环境和目标系统用同一个 pool 构建**，版本天然一致。
- ISO 仓库发布到 HTTP 后，就是 PXE 安装源和在线升级源。

---

## 4. 软件源体系

### 4.1 本地 Ubuntu 镜像源与快照（构建不依赖网络，镜像尽可能小）

**原则一：同步和构建分开。** 只有"镜像同步"这一步访问外网；ISO 构建、CI 打包、闭包计算、mmdebstrap 全部只读本地镜像，构建机可以完全断网。

**原则二：镜像只包含需要的包（已拍板）。** 不做全量镜像，只同步 `xvirt-distro` 种子及其依赖闭包。

```mermaid
flowchart LR
  N["Ubuntu 官方源 / 国内镜像站"] -->|"mirror-sync.sh<br/>（唯一联网步骤，按需触发）"| L["本地镜像（按种子过滤）<br/>/srv/xvirt-mirror/aptly"]
  L -->|"aptly snapshot create<br/>（秒级，不复制文件）"| S["Ubuntu 快照<br/>ubuntu-20261125"]
  S --> C["compose-repo / build-iso<br/>（离线）"]
  L -->|nginx 只读发布| W["http://&lt;mirror&gt;/ubuntu<br/>供 CI 容器、开发机使用"]
```

- **工具**：aptly。它把下载的 deb 统一存放在自己的 pool 里（按内容去重），快照和发布只是元数据和硬链接，打快照、合并、过滤、发布都在本地完成，速度快。
- **镜像范围**：`resolute`、`resolute-updates`、`resolute-security` 三个 pocket，`main`、`restricted`、`universe` 三个组件，只要 amd64、不要源码包。
- **按种子过滤**：过滤条件 = `xvirt-distro` 中**所有种子的并集**：
  - 运行时：`base` + 三个角色 + `installer-env`（决定 ISO 内容）；
  - 构建期：`build-deps`（CI 编译各组件所需的构建依赖，只进镜像，不进 ISO）。
  - **不跟随 Recommends / Suggests**；`a | b` 形式的可选依赖只跟随首选项（aptly 默认行为），与目标系统的安装策略一致（§7.6）。
- **按 pocket 分别过滤的问题与处理**：aptly 的 `-filter-with-deps` 只在**当前 mirror 内**解析依赖，`-updates` / `-security` 中的包往往依赖 release pocket 里的包，单独过滤会漏。因此由 `xvirt-distro/mirror/mirror-sync.sh` 做**不动点迭代**：
  1. release pocket：`-filter=<种子并集> -filter-with-deps`；
  2. 把第 1 步得到的完整包名列表作为 `-updates` / `-security` 的过滤条件（按包名，不再展开依赖）；
  3. 三个 pocket 合并成临时快照，用 `aptly snapshot filter -with-deps` 重新计算闭包；
  4. 若闭包中出现镜像里没有的包（更新引入了新依赖），把它们加入过滤列表，回到第 1 步，直到没有新增；
  5. 打带日期的快照（如 `ubuntu-20261125`）。
- **代价**：种子里新增一个包时，需要先跑一次 `mirror-sync.sh`（联网）再构建。compose-repo 发现缺包时直接报错并提示"请先同步镜像"，不会悄悄联网。
- **上游地址可配置**：默认用国内镜像站（如 `mirrors.aliyun.com/ubuntu`）加快同步，仍校验 Ubuntu 官方签名（aptly mirror 校验上游 `Release.gpg`，构建机需导入 Ubuntu archive keyring）。
- **同步时机**：
  - 出现需要处理的 Ubuntu 安全公告时（§7.5）；
  - `xvirt-distro` 种子变更时；
  - 准备发布新功能版之前。
  不设每天定时全量同步。构建只引用快照名，同一快照多次构建结果一致。
- **快照保留**：正式发布引用过的快照永久保留（写进 `build-manifest.json`，便于重建和追溯）；其余快照保留最近 N 个，定期 `aptly db cleanup` 回收空间。
- **本地发布**：本地镜像同时用 nginx 只读发布，CI 构建容器（`ferrostack-ci-builder`）和开发机的 apt 源都指向它，替代直接访问公网，也加快 `package-deb` 阶段安装构建依赖的速度。
- **体积**：按种子过滤后预计只有几 GB（主要是内核、固件、各 pocket 中同一包的多个版本），首次同步时实测。存放在 `/srv/xvirt-mirror`（aptly rootDir），与构建临时目录分开。

### 4.2 xvirt 自有源

- aptly local repo，分两个通道：
  - `xvirt-dev`：CI 每次合入后构建的包，用于日常 ISO / 冒烟测试；
  - `xvirt-<major>`（如 `xvirt-1`）：正式发布的包，只追加不覆盖。
- 内容：xvirt 平台包（`xvirt-ferrovirt`、`xvirt-lictor`、`xvirt-hyper-agent`、`xvirt-petrel`、`xvirt-concilium`、`xvirt-client`、`xvirt-init` …）、定制 QEMU（`qemu-kvm-xvirt`，build-deb 产出）、`lycan`、元包、`xvirt-archive-keyring`、`xvirt-release`，以及 Ubuntu 中没有、需要自行打包的第三方软件。
- 现有的 `cicd/var/local-apt-repo`（CI 内置 APT 源）演进为该仓库的一个发布端点。

### 4.3 ISO 仓库（合并后的产物）

- **发布 distribution 按大版本（已拍板）**：`xvirt-1` 覆盖 1.x 的全部功能版、补丁版和 Ubuntu 更新版。同一大版本内任何一个更新的 ISO 都可以直接作为升级源；跨大版本（如 1.x → 2.x）时由 `xvirt-upgrade` 把节点的 suite 从 `xvirt-1` 切换到 `xvirt-2`。
- `aptly snapshot merge` 合并 Ubuntu 快照与 xvirt 快照；xvirt 包与 Ubuntu 包同名时，以 xvirt 为准（merge 时 xvirt 快照放在后面，或用版本号/epoch 保证更高）。
- `aptly snapshot filter -with-deps` 按种子计算闭包，生成 ISO 快照（如 `iso-1.0.0`）。**只放需要的依赖包**：只包含 `Priority: required`、运行时种子及其 Depends/Pre-Depends 闭包，不含 Recommends、不含 `Priority: important/standard` 的包（§7.6）。
- `aptly publish snapshot -skip-signing` 输出 `dists/` + `pool/`，然后把 `Release` 送到签名服务签名（§4.4），再放进 ISO 根目录。
- 发布参数：`-distribution=xvirt-1 -component=main -architectures=amd64`，`-label=xvirt -origin=xvirt`。

> 依赖闭包示意（种子以 `xvirt-distro` 为准）：
>
> ```bash
> aptly snapshot merge -latest all-1.0.0 ubuntu-20261125 xvirt-1-20261125
> aptly snapshot filter -with-deps all-1.0.0 iso-1.0.0 \
>   'Priority (required) | xvirt-standalone | xvirt-management | xvirt-compute | xvirt-installer-env'
> aptly publish snapshot -skip-signing -distribution=xvirt-1 iso-1.0.0 iso
> # 再调用签名服务生成 InRelease / Release.gpg（§4.4）
> ```

### 4.4 签名密钥与签名服务（cairn.xve-vt.com）

**已拍板：签名私钥放在 `cairn.xve-vt.com` 上。** 构建机上不放私钥，通过签名服务远程签名。

- **两把密钥**：`xvirt-release`（正式通道）和 `xvirt-dev`（dev 通道），分开生成、分开使用。建议 ed25519，或兼容性更好的 RSA4096。
- **签名服务 `xvirt-signer`**（部署在 cairn.xve-vt.com）：
  - 私钥由该主机上的 gpg-agent 持有，私钥文件和口令不出这台机器；口令通过 systemd credential 提供，不写进配置文件；
  - HTTPS 接口：`POST /sign`，提交 apt `Release` 文件，返回 `InRelease`（clearsign）和 `Release.gpg`（分离签名）；
  - 认证：CI 使用专用 token（或 mTLS 客户端证书），按通道授权——dev token 只能用 dev 密钥；**正式密钥只允许发布流水线调用，打 tag 的构建自动放行（已拍板）**，不需要人工逐次确认：
    - 发布 tag 打在 `xvirt-distro` 上，格式 `v<X.Y.Z>` 或 `v<X.Y.Z>-u<YYYYMMDD>`（Ubuntu 更新版）；git 服务器上把 `v*` 设为**受保护 tag**，只有发布负责人能创建，且不能移动或删除；
    - 流水线确认当前 commit 正是该 tag 指向的 commit、tag 中的版本与 `VERSION` 一致，才用正式 token 调用签名服务；签名服务再核对请求里的 `Suite` 与 tag 的大版本一致（`v1.*` 只能签 `xvirt-1`）；
    - 每次正式签名自动在 Cairn 记录一条通知（tag、构建 ID、Release sha256），事后可追溯；
  - 校验输入：只接受格式正确的 apt Release 文件，且 `Origin: xvirt`、`Suite` 必须在允许列表内，拒绝签任意内容；
  - 审计：记录每次签名的时间、调用方、构建 ID、Release 文件 sha256、使用的密钥；
  - 备份：私钥离线加密备份两份，存放在 cairn.xve-vt.com 以外的地方；主机上的密钥目录单独用户、最小权限。后续可迁到 YubiKey / HSM，接口不变。
- **构建侧流程**：`aptly publish -skip-signing` → 调用 `xvirt-signer` → 写入 `InRelease` / `Release.gpg` → 构建机用公钥 `gpgv` 本地验证 → 再继续构建安装环境和 ISO。签名这一步只访问 cairn.xve-vt.com，不访问任何软件源，因此仍符合"构建不依赖网络源"。在 CI 中，构建容器以 `--network none` 运行，签名步骤在容器外由流水线执行。
- 公钥打包为 `xvirt-archive-keyring`，安装到 `/usr/share/keyrings/xvirt-archive-keyring.pgp`；dev 构建的 ISO 中 keyring 包同时含 dev 公钥，正式 ISO 只含正式公钥。目标系统的 apt 源用 `Signed-By:` 指向该文件。
- 密钥轮换：keyring 包可同时携带新旧两把公钥，过渡一个大版本。

---

## 5. `xvirt-distro` 清单仓库

新建 git 仓库 `xvirt-distro`，描述"一个 xvirt 发行版由什么组成"，作用类似 Fedora 的 comps + kickstart、Ubuntu 的 germinate seeds。

```text
xvirt-distro/
├── VERSION                     # 产品版本，如 1.0.0
├── distro.yaml                 # Ubuntu 快照名、xvirt 通道、发布 distribution 名、签名密钥 ID
├── seeds/
│   ├── base.txt                # 所有角色共有的基础包：linux-image-generic、grub、shim、openssh-server、chrony …
│   ├── standalone.txt          # → 生成 xvirt-standalone 元包
│   ├── management.txt          # → 生成 xvirt-management 元包
│   ├── compute.txt             # → 生成 xvirt-compute 元包
│   ├── installer-env.txt       # 安装环境：lycan、parted、mdadm、lvm2、ethtool、xfsprogs …
│   ├── build-deps.txt          # CI 编译各组件的构建依赖：只进本地镜像，不进 ISO
│   └── blacklist.txt           # 禁止进入系统的包：snapd、cloud-init、ubuntu-advantage-tools …
├── debian/                     # 元包 / xvirt-release / xvirt-archive-keyring 的打包描述
├── keys/                       # 发布公钥（私钥在 cairn.xve-vt.com，不进仓库）
├── mirror/
│   └── mirror-sync.sh          # 本地镜像同步（按种子过滤 + 不动点迭代，§4.1）
├── iso/
│   ├── grub/                   # grub.cfg 模板（BIOS + UEFI 共用）
│   ├── isolinux 或 eltorito/   # BIOS 引导相关
│   └── build-iso.sh            # ISO 组装脚本（§7）
├── installer-env/
│   └── hooks/                  # 安装环境 squashfs 的定制钩子（自动登录 tty1 启动 lycan 等）
└── tests/                      # 依赖闭包检查、版本单调检查
```

### 5.1 元包

| 元包 | 依赖（示意） |
|---|---|
| `xvirt-base` | `linux-image-generic`、`grub-efi-amd64-signed`、`shim-signed`、`grub-pc-bin`、`openssh-server`、`chrony`、`xvirt-archive-keyring`、`xvirt-release`、`xvirt-init`、常用运维工具；`Conflicts: snapd, cloud-init` |
| `xvirt-standalone` | `xvirt-base`、`xvirt-ferrovirt`、`xvirt-lictor`、`xvirt-hyper-agent`、`xvirt-petrel`、`xvirt-concilium`、`qemu-kvm-xvirt`、`openvswitch-switch`、`nginx`、`mariadb-server` … |
| `xvirt-management` | `xvirt-base`、`xvirt-ferrovirt`、`xvirt-client`、`nginx`、`mariadb-server`、`avahi-daemon`、`libnss-mdns` … |
| `xvirt-compute` | `xvirt-base`、`xvirt-lictor`、`xvirt-hyper-agent`、`xvirt-petrel`、`qemu-kvm-xvirt`、`openvswitch-switch` … |
| `xvirt-installer-env` | 仅用于构建安装环境，不装到目标系统 |

- 三个角色元包之间互相 `Conflicts`，一台机器只能装一个角色。
- **元包依赖要写版本下限**（例如 `xvirt-ferrovirt (>= 1.0.0)`），保证升级时组件一起升。
- 以后新增依赖只需要改元包，升级时 `apt full-upgrade` 会自动装上新依赖。

### 5.2 `xvirt-release` 包

- 安装 `/etc/xvirt-release`（产品名、版本、构建 ID、Ubuntu 快照名），以及 `/etc/os-release` 的 xvirt 扩展字段（`VARIANT=` / `VARIANT_ID=`），**不替换 Ubuntu 的 os-release**。
- 安装 apt 源模板（§9.1）和 apt pin（§9.2）。

---

## 6. 包与版本规范

### 6.1 版本号

| 场景 | 格式 | 示例 |
|---|---|---|
| 正式发布 | `<X.Y.Z>-<rev>` | `1.0.0-1` |
| 正式发布前的开发构建 | `<X.Y.Z>~dev<YYYYMMDDHHMM>.g<hash>-1` | `1.0.0~dev202611252230.g3d9d55d-1` |
| 发布后的补丁构建 | `<X.Y.Z>-<rev>` 中 rev 递增，或升 Z | `1.0.0-2` / `1.0.1-1` |

- `~` 在 dpkg 中排序低于任何字符，所以 `1.0.0~dev… < 1.0.0-1`，开发版天然低于正式版。
- **时间戳放在哈希前面**，保证按构建顺序单调递增。
- 取消现有的 `0.0.1+git<hash>`，新方案从 `1.0.0` 起。老节点不在支持范围内（§2.2），无需用 epoch 兼容旧版本号。
- 定制 QEMU 继续沿用 `qemu-kvm-xvirt` 的 epoch 2，但 Debian revision 部分同样改为带时间戳的递增格式。
- CI 增加**版本单调检查**：同名包的新版本必须 `dpkg --compare-versions new gt old`。

### 6.2 maintainer 脚本规范

升级能否顺利，主要取决于各 deb 的 maintainer 脚本。要求：

1. `postinst configure` 必须**幂等**，能在首次安装和升级两种情况下重复执行。
2. 配置文件：用户可修改的放 `conffiles`（dpkg 会提示/保留用户修改）；由程序管理的配置在 postinst 里做迁移，不放 conffiles。
3. 数据库 schema 迁移：沿用 ferrovirt 启动时的 `schema_fix` 机制（ferrovirt_fixup 表记录已做过的修正），postinst 只负责重启服务；迁移必须向前兼容，不允许破坏性 DDL 在同一版本内既改又删。
4. 服务：用 `dh_installsystemd`，升级时 `--restart-after-upgrade`；对正在运行 VM 的服务（lictor / hyper-agent / qemu 相关）**不能因为升级而杀掉虚拟机**——需要在设计上保证重启 agent 不影响已运行的 qemu 进程，或在升级流程中先进入维护模式（§10.3）。
5. 在 chroot 中执行时（安装阶段）不得启动服务；靠 `policy-rc.d` 返回 101 + 检测 `systemd` 是否在运行来判断。
6. 首次启动才能做的事（读硬件、建网桥、初始化数据库等）继续交给 `xvirt-init.service`，不要放进 postinst。

---

## 7. ISO 结构与构建

### 7.1 ISO 目录结构

```text
/
├── .disk/info                       # 产品名 + 版本 + 构建 ID
├── xvirt/
│   ├── build-manifest.json          # xvirt-distro commit、Ubuntu 快照名、全部包及版本、sha256
│   ├── answers/                     # 内置应答模板（standalone / management / compute）
│   └── xvirt-archive-keyring.pgp    # 公钥（供人工校验 ISO 仓库签名）
├── dists/xvirt-1/                 # apt 仓库元数据（InRelease / Release.gpg / Packages）
├── pool/main/...                    # 全部 deb：内核、基础系统、xvirt 组件
├── live/
│   ├── vmlinuz                      # 安装环境内核（与 pool 中 linux-image 同一版本）
│   ├── initrd.img
│   └── filesystem.squashfs          # 安装环境根文件系统（含 lycan）
├── boot/grub/                       # grub.cfg、BIOS 用 grub 模块
└── EFI/boot/                        # bootx64.efi(shim)、grubx64.efi(签名 grub)、efiboot.img
```

体积目标：**ISO ≤ 1 GB**（三个角色合一；现在单角色 ISO 约 3.0G）。P2 实测后校准，体积控制措施见 §7.6。

### 7.2 安装环境（Live runtime）

- 用 `mmdebstrap` 从**同一个 ISO 仓库**构建：`mmdebstrap --variant=minbase --include=xvirt-installer-env resolute ...`（`xvirt-installer-env` 依赖 `live-boot`、`linux-image-generic`、lycan 及分区工具），压成 `live/filesystem.squashfs`。
- 引导方式（**已拍板**）：Ubuntu 内核 + `initramfs-tools` + **Debian 的 `live-boot`**（内核参数 `boot=live`），不再使用 casper。支持 `toram`（整个安装环境读入内存，安装过程中可以拔掉 U 盘/虚拟光驱）；PXE 时用 `fetch=http://…/live/filesystem.squashfs`。
- `live-boot` 在 Ubuntu 的 universe 中提供，包含在本地镜像内；P2 阶段首先验证它与 resolute 的内核、initramfs-tools 能正常配合，如有问题以补丁包形式放进 xvirt 自有源。
- 安装环境中 tty1 自动登录并启动 lycan；`systemd.mask` subiquity 之类的补丁全部不再需要。
- 救援 SSH、"Rescue & Hardware Diagnostics Shell" 引导项沿用现有设计。

### 7.3 引导与 Secure Boot

- **UEFI**：使用 pool 中 Ubuntu 签名的 `shim-signed` 和 `grub-efi-amd64-signed`，内核也是 Canonical 签名的，因此 Secure Boot 开启时无需自有签名密钥。注意 Ubuntu 签名 grub 的配置查找路径（`/EFI/ubuntu/` 及 ISO 内 `/boot/grub/`）要和现有 lycan ISO 的做法保持一致。
- **BIOS**：`grub-pc-bin` 生成 El Torito 引导映像，xorriso 构建 hybrid ISO（可直接 `dd` 到 U 盘）。
- **定制 QEMU、自研内核模块**：都是用户态包或 DKMS 包时不影响 Secure Boot；如以后需要自编内核，再考虑 MOK 签名。

### 7.4 构建步骤（`xvirt-distro/iso/build-iso.sh`）

0. **离线前提**：构建全程只访问本地 aptly（Ubuntu 快照 + xvirt 自有源），不配置任何网络源；CI 中构建容器以 `--network none` 运行作为强制检查。
1. 读取 `distro.yaml` 与 `VERSION`，确定 Ubuntu 快照、xvirt 通道。
2. 构建 xvirt-distro 自己的包（元包、`xvirt-release`、`xvirt-archive-keyring`），加入 xvirt 自有源。
3. aptly：合并快照 → 按种子过滤闭包 → `publish -skip-signing` 到 `staging/iso-root/` → 调用 cairn.xve-vt.com 上的 `xvirt-signer` 签名（dev 或正式密钥，§4.4）→ 本地 `gpgv` 验证。
4. **闭包自检**：在干净 chroot 中，只使用 `file:` 指向 staging 仓库，分别 `apt-get install -s` 三个角色元包和 `xvirt-installer-env`，任何缺包即失败；检查 `blacklist.txt` 中的包没有进入闭包；**体积检查**：输出最大的 20 个包和 ISO 总大小，超出预算即失败（§7.6）。
5. mmdebstrap 构建安装环境 squashfs，提取同版本 `vmlinuz` / `initrd.img`。
6. 生成 grub 配置（交互安装、自动应答 × 三种角色、救援 shell）。
7. 生成 `xvirt/build-manifest.json` 与 `.disk/info`。
8. xorriso 打包 hybrid ISO（BIOS + UEFI），输出 `xvirt-<version>-amd64.iso` 及 sha256。
9. 同时把 `staging/iso-root/{dists,pool}` 发布到内部 HTTP 仓库（供 PXE 和在线升级）。

整个构建可以在现有 `ferrostack-ci-builder:26.04.1` 容器中运行（需要额外安装 aptly、mmdebstrap、live-boot 相关工具，aptly rootDir 以只读方式挂入），不再依赖 `ubuntu-26.04.1-live-server-amd64.iso`。

### 7.5 ISO 类型与版本

| 类型 | 内容变化 | ISO 版本示例 | 发布 distribution |
|---|---|---|---|
| 功能版 | xvirt 包更新（可同时更新 Ubuntu 快照） | `1.1.0` | `xvirt-1` |
| 补丁版 | 个别 xvirt 包修复 | `1.1.1` | `xvirt-1` |
| **Ubuntu 更新版**（已拍板） | **只换 Ubuntu 快照**，xvirt 包与所基于版本完全相同 | `1.1.1-u20261215` | `xvirt-1` |

Ubuntu 更新版的出法：

**触发条件（已拍板）：有安全公告时再出**，不按固定周期出。

1. **公告监测**：`usn-watch` 定时任务（只拉取元数据，很小）读取 Ubuntu 安全公告（USN）/ OVAL 数据中 resolute 的条目，与**仍在支持期内的已发布 ISO** 的 `build-manifest.json` 做比对：公告涉及的包在我们的闭包里，才算命中。
2. **命中后通知**：在 Cairn 中自动建 issue（含 USN 编号、CVE、涉及的包及版本、受影响的 ISO 版本），由负责人判断是否出更新版（例如内核、openssh、openssl、glibc 等高危公告一般要出）。
3. **出更新版**：先 `mirror-sync.sh` 同步本地镜像并打新快照；`xvirt-distro` 中只修改 `distro.yaml` 的 Ubuntu 快照名，xvirt 通道仍指向原正式版的 xvirt 快照，不重新构建任何 xvirt 包。
4. **出给哪个版本**：每个大版本只给**最新功能版**出 Ubuntu 更新版；运行旧功能版的节点可以直接用它升级（同一 distribution，且跨版本升级受支持，§10.3）。
5. 走同样的 compose-repo / build-iso / 冒烟测试 / 升级测试流程。
6. `xvirt-release` 的 `/etc/xvirt-release` 增加 `UBUNTU_SNAPSHOT=` 字段，便于在界面和 `xvirt-upgrade --check` 中看出节点当前所在的快照。
7. 因为 distribution 相同，它可以直接作为同一大版本节点的升级源，升级效果等同于 `apt full-upgrade` 拿到 Ubuntu 的安全更新（含内核）。

### 7.6 体积控制（ISO 只包含需要的依赖包）

**已拍板：ISO 只包含需要的依赖包。** 措施：

| 措施 | 说明 |
|---|---|
| 不跟随 Recommends / Suggests | 镜像、ISO 闭包、目标系统三处一致：闭包只跟 Depends / Pre-Depends；lycan 装包加 `--no-install-recommends`；`xvirt-release` 安装 `/etc/apt/apt.conf.d/90xvirt-no-recommends`（`APT::Install-Recommends "false"`），保证以后升级时也不会去找 ISO 里没有的推荐包 |
| 不纳入 important / standard 优先级 | 只包含 `Priority: required`（mmdebstrap `minbase`）+ 种子里显式列出的包；需要的工具（如 `less`、`vim-tiny`、`iproute2`、`pciutils`）在 `base.txt` 中逐个列出 |
| 内核只装 `linux-image-generic` | 不装 `linux-generic` 带来的 headers；没有 DKMS 需求时不需要编译环境 |
| 黑名单 | `blacklist.txt`：snapd、cloud-init、ubuntu-advantage-tools、ubuntu-pro-client、landscape、motd-news、popularity-contest、apport、文档类包等；闭包里出现即构建失败，并打印是谁依赖进来的 |
| 安装环境瘦身 | squashfs 用 xz 压缩；`dpkg path-exclude` 去掉 `/usr/share/doc`、`man`、`info`，locale 只保留 `zh_CN`、`en_US`；不保留 apt 缓存和列表 |
| **固件不重复存放（已拍板，必须做）** | `linux-firmware` 是体积最大的单项之一，ISO 里只保留 pool 中的那一份 deb，安装环境 squashfs 不含固件。具体做法见下文 |
| 体积预算检查 | CI 每次构建输出最大的 20 个包、各部分体积（pool / squashfs / 引导），超出预算（初定 1 GB）即失败，防止体积悄悄变大 |

**固件去重的做法**

1. **构建时**：安装环境照常安装 `linux-firmware`（保证 dpkg 依赖满足），但用 `dpkg path-exclude=/usr/lib/firmware/*` 不落地固件文件，squashfs 中只剩包登记信息。
2. **启动时**：安装环境里的 `xvirt-firmware-unpack.service` 排在 `systemd-udev-trigger.service` **之前**运行：
   - 从 live 介质（`/run/live/medium`）的 pool 中找到 `linux-firmware_*.deb`（版本由构建时写入的清单指定，与 squashfs 中登记的版本一致）；
   - 把其中的 `/usr/lib/firmware` 解包到 tmpfs，并挂到安装环境的 `/usr/lib/firmware`；
   - 之后 udev 冷插拔加载网卡、HBA 等驱动时就能找到固件。
   - PXE 启动时 live 介质不在本地，改为从 `lycan.repo=` 指向的 HTTP 仓库下载同一个 deb。
3. **initramfs 阶段的驱动**：在切换到真正的根之前加载的驱动（读 ISO 的存储驱动、PXE 时取 squashfs 用的网卡驱动）用不上 tmpfs 里的固件。由 `initramfs-tools` 按 `MODULES=netboot` 自动把**这些驱动用到的少量固件**打进 initrd，这部分重复只有几 MB，可以接受。
4. **兜底**：解包失败时不阻止启动，lycan 在网卡/磁盘列表中标出"固件缺失"的设备，并在日志中给出原因。
5. **内存开销**：解包后的固件放在内存里，大约几百 MB；安装环境本来就要求 ≥4 GB 内存，可以接受。
6. **目标系统不受影响**：目标系统按正常方式安装 `linux-firmware` deb，固件落在硬盘上。

> 同样的思路可以用于 `linux-modules-extra`（内核扩展模块，在 pool 与 squashfs 中也是重复的），但模块在 initramfs 和 udev 阶段都要用，处理更复杂。先做固件，模块是否去重看 P2 的实测体积再定。

> 注意：不装 Recommends 后，有些包的常用功能可能缺失（例如某些服务依赖的辅助工具只在 Recommends 里）。各角色需要的这类包要显式加进种子，靠离线冒烟测试和物理机测试发现遗漏。

---

## 8. lycan 改动

交互界面、应答文件、网络、磁盘、账号相关代码**基本不动**，主要改动在 `steps/install.py` 的"装系统"部分。

### 8.1 新的安装流程

| 阶段 | 内容 |
|---|---|
| 1. 准备磁盘 | 不变：`_release_disk`、分区（ef02 / ESP / swap / root …）、mkfs、挂载到 `/target` |
| 2. 定位安装源 | 光盘/U 盘：挂载 ISO 到 `/run/xvirt/media`；PXE：使用 `lycan.repo=http://…` 或 fledge 下发的 URL |
| 3. 装基础系统 | `mmdebstrap --variant=minbase --include=xvirt-base --aptopt='Dir::Etc::TrustedParts "/run/xvirt/keyrings"' resolute /target "deb [signed-by=…] file:/run/xvirt/media xvirt-1 main"` |
| 4. 装角色 | chroot 中 `apt-get install -y xvirt-<role>`（角色来自向导或应答文件） |
| 5. 系统配置 | 不变：fstab、hostname、netplan、账号/SSH 密钥、时区、键盘、`/etc/xvirt/role.conf`、`default-bridge.conf` |
| 6. 引导 | `update-initramfs -c -k all`（目标系统用的是 `linux-image-generic` 自带的普通 initramfs，不再有 live initrd 混入的问题，`MODULES=most` 可保留为保险）、`grub-install`（BIOS + UEFI）、`update-grub` |
| 7. apt 源落盘 | 写入目标系统的 `xvirt.sources`（§9.1），默认指向本地 ISO 挂载点 / 管理节点仓库，不指向 Ubuntu 公网 |
| 8. 收尾 | 卸载、同步、按应答 `reboot` |

### 8.2 安装速度

- 用 `dpkg --force-unsafe-io`（安装阶段可接受；结束时统一 `sync`）和 `eatmydata` 减少 fsync。
- 预计在 SSD 上 3~6 分钟完成装包，比复制镜像慢几分钟，可以接受。
- 进度：apt 用 `-o APT::Status-Fd=<fd>` 输出机器可读进度，lycan 映射到现有 8 阶段进度条和 fledge 回调（`callback_url`）。

### 8.3 可以删掉的 workaround

以下针对 casper / Ubuntu Live 盘的特殊处理在新方案下不再需要：

- overlay 挂载分层 squashfs 再复制（`_target_layer_chain` / `_extract_base_system`）；
- 屏蔽 subiquity（build 期 symlink + `systemd.mask=` 内核参数）；
- 绕过 casper 的"请移除安装介质"（强制 `reboot -f`）；
- 禁用 cloud-init（不再安装，blacklist 中禁止）；
- 删除 `file:/cdrom` 旧源、修 chroot `resolv.conf`；
- live initrd 混入目标系统导致的 kernel panic。

### 8.4 兼容

- 应答文件格式不变；新增可选字段 `repo:`（覆盖安装源 URL）。
- 内核参数新增 `lycan.repo=`；其余 `lycan.*` 参数保持不变。
- 旧的 per-role ISO 构建脚本保留一个版本周期后移除。

---

## 9. 目标系统上的 apt 配置

### 9.1 源

`/etc/apt/sources.list.d/xvirt.sources`（deb822 格式，由 lycan 写入，模板来自 `xvirt-release`）：

```text
Types: deb
URIs: file:/run/xvirt/upgrade-media
Suites: xvirt-1
Components: main
Signed-By: /usr/share/keyrings/xvirt-archive-keyring.pgp
Enabled: no
```

- 平时**不启用**任何源（离线节点不应该在 `apt update` 时报错）；升级工具临时启用（§10）。
- 计算节点可配置为指向管理节点发布的 HTTP 仓库。
- **默认不允许接入 Ubuntu 公网源（已拍板）**：lycan 不写入 `ubuntu.sources`，安装界面和管理界面都不提供该选项；Ubuntu 的安全更新只通过 xvirt 的 Ubuntu 更新版 ISO 获得（§7.5）。
- 管理员如果手工加了其他源，`xvirt-upgrade --check` 会列出所有非 xvirt 的源并给出警告（这种节点不在支持范围内），升级时只启用 xvirt 源。

### 9.2 Pin

`xvirt-release` 安装 `/etc/apt/preferences.d/xvirt`，让 `Origin: xvirt` 的源优先级高于其他源，避免节点误接 Ubuntu 公网源后 Ubuntu 的同名包覆盖 xvirt 包（例如 qemu）。

---

## 10. 升级

### 10.1 单节点：`xvirt-upgrade`

由 `xvirt-init` 或单独的 `xvirt-upgrade` 包提供，命令：

```bash
xvirt-upgrade --iso /path/to/xvirt-1.1.0-amd64.iso     # 本地 ISO 文件
xvirt-upgrade --media /dev/sr0                          # 光驱 / U 盘
xvirt-upgrade --repo http://<mgmt>/xvirt/repo           # 管理节点发布的仓库
xvirt-upgrade --check                                   # 只做预检和模拟
```

流程：

1. **挂载与校验**：挂载 ISO（只读 loop）；用本机 keyring 校验 `InRelease` 签名；读取 `xvirt/build-manifest.json`。
2. **预检**：
   - 目标版本 ≥ 当前版本（不允许跨大版本跳级时，在此拒绝）；
   - 角色元包在仓库中存在；
   - `/`、`/boot`、`/var` 剩余空间足够；
   - `apt-get -s full-upgrade` 模拟成功，不会删除角色元包或关键包；
   - 集群角色检查（§10.3）。
3. **执行**：临时启用源 → `apt-get update` → `apt-get full-upgrade -y`（`DEBIAN_FRONTEND=noninteractive`，`--force-confold`）→ `apt-get autoremove` 保留最近 2 个内核。
4. **收尾**：如内核或 qemu/libvirt 相关包有更新，提示需要重启（或按参数自动重启）；记录升级日志到 `/var/log/xvirt/upgrade-<时间>.log` 并上报 ferrovirt。
5. 卸载 ISO、禁用临时源。

### 10.2 从 Web 界面升级

后续在管理中心/单机界面提供"上传 ISO → 预检 → 升级"，后端调用 `xvirt-upgrade`。本期只做命令行。

### 10.3 集群滚动升级

1. **先升管理节点**：管理节点升级后，把 ISO 仓库发布到自己的 nginx（如 `/xvirt/repo/xvirt-1/`）。
2. **再逐台升计算节点**：
   - 进入维护模式，把运行中的虚拟机迁移走（或按策略关机）；
   - 节点上执行 `xvirt-upgrade --repo http://<mgmt>/xvirt/repo`；
   - 需要时重启，自检通过后退出维护模式。
3. **版本兼容策略（已拍板：兼容多个版本）**：
   - 管理节点必须能同时管理**多个旧版本**的计算节点，不仅是 N-1。集群允许长期处于混合版本状态（例如分批、跨多个维护窗口升级）。
   - 计算节点可以**跨版本直接升级**（例如 1.0 → 1.3），不要求逐个版本升级；apt 会直接装到目标版本，各组件的数据/配置迁移必须能从任意受支持的旧版本一步迁移到新版本。
   - 兼容窗口在每个版本的发布说明中明确（**已拍板：至少兼容 4 个功能版**，即管理节点为 1.5 时，至少能管理 1.1~1.5 的计算节点），窗口外的计算节点在管理中心中标记为"版本过旧，需先升级"。
   - 实现要求：
     - 计算节点（hyper-agent / lictor）向管理节点上报**产品版本 + 能力列表**；管理节点按能力决定是否下发新功能，不按版本号硬编码判断；
     - 管理节点 ↔ 计算节点的 API / ZMQ 消息只做**向后兼容的增量修改**（加字段、加可选参数、加新接口），删除或改语义必须跨过兼容窗口后才能进行；
     - 共享存储（Petrel / Fulmar）的磁盘格式、集群元数据同样要求旧版本节点可读写；
     - CI 的 `test-upgrade` 覆盖"新管理节点 + 窗口内每个旧版本计算节点"的混合集群用例。
   - 计算节点版本不能高于管理节点（升级顺序永远是先管理节点）；`xvirt-upgrade` 在计算节点上预检这一点。
4. 管理中心后续可以编排整个流程；本期先提供命令行和操作手册。

### 10.4 PXE（fledge）

- fledge 发布与 ISO 完全相同的 `dists/` + `pool/` + `live/`：iPXE 加载 `live/vmlinuz` + `initrd.img`，`fetch=` 拉 `filesystem.squashfs`，`lycan.repo=` 指向 HTTP 仓库。
- PXE 安装出的节点与光盘安装出的节点，包组合完全一致。

### 10.5 回滚

**已拍板：本期不考虑回滚。** 不做 apt 降级流程、LVM/btrfs 快照或 A/B 分区，分区布局保持现状。

仍然保留的最低限度保护（不算回滚功能，只是常规做法）：

- 升级后保留上一个内核，新内核无法启动时可在 grub 菜单中选择旧内核；
- `xvirt-upgrade` 升级前自动备份 `/etc/xvirt` 和管理数据库到 `/var/backups/xvirt/`，用于人工排障。

---

## 11. CI 流水线改造

现有 `iso` profile：`sync → unit-test → package-deb → offline-bundle → build-iso → test-nested-kvm`。调整为：

| 阶段 | 变化 |
|---|---|
| `package-deb` | 版本号改为 §6.1 格式；产物发布到 aptly 的 `xvirt-dev` 仓库（替代直接写 `cicd/var/local-apt-repo`） |
| `offline-bundle` | **删除**，由 `compose-repo` 取代 |
| `mirror-sync`（新，独立任务，不在流水线内） | 唯一联网的步骤：`mirror-sync.sh` 按种子过滤同步 + 打日期快照；由安全公告、种子变更或发布前手工触发；失败不影响构建（继续用上一个快照） |
| `usn-watch`（新，定时任务） | 比对 Ubuntu 安全公告与已发布 ISO 的包清单，命中时在 Cairn 建 issue（§7.5） |
| `compose-repo`（新） | 只读本地 aptly：合并 Ubuntu 快照 + xvirt 快照 → 按种子过滤 → 签名发布；执行闭包自检和版本单调检查 |
| `build-iso` | 改为调用 `xvirt-distro/iso/build-iso.sh`，只出一个 ISO；`pipeline.jsonc` 中 `iso.roles`、`base_ubuntu_iso`、`bundle_filter`、`extra_offline_pkgs` 等配置项删除，改由 `xvirt-distro` 管理 |
| `test-nested-kvm` | 同一 ISO 分别以三种角色自动安装，仍用 `restrict=on` 验证完全离线 |
| `test-upgrade`（新） | 用兼容窗口内的旧版 ISO 分别安装 → 用本次 ISO 执行 `xvirt-upgrade --iso`（覆盖跨版本直接升级）→ 校验服务、数据、版本；另测管理节点 HTTP 仓库方式升级计算节点，以及"新管理节点 + 各旧版本计算节点"的混合集群 |
| `test-pxe-destructive` | fledge 改为发布新的仓库结构 |

---

## 12. 实施阶段

| 阶段 | 内容 | 产出 / 验收 |
|---|---|---|
| P0 基础设施 | 搭 aptly；整理种子（含 `build-deps`），用 `mirror-sync.sh` 按种子同步本地镜像并打快照；nginx 发布本地镜像，CI 容器改用本地源；在 cairn.xve-vt.com 生成正式 / dev 两把密钥并部署 `xvirt-signer`；建 `xvirt-dev` 仓库 | 断网状态下能 compose 出合并仓库，并经签名服务签名、本地验证通过 |
| P1 包规范 | 各组件版本号改为 §6.1；检查 maintainer 脚本是否符合 §6.2（尤其是服务重启不杀 VM）；建 `xvirt-distro`，产出元包、`xvirt-release`、`xvirt-archive-keyring` | 在干净 chroot 中只用合并仓库就能装出三种角色 |
| P2 ISO 与安装环境 | mmdebstrap 构建安装环境；`build-iso.sh`；BIOS + UEFI（含 Secure Boot）引导 | ISO 在 nested KVM 中能启动 lycan |
| P3 lycan 改造 | §8 的安装流程；删除 casper workaround；进度上报 | 三种角色离线自动安装通过 nested 冒烟，以及至少一台物理机（NVMe）安装 |
| P4 升级 | `xvirt-upgrade`；管理节点发布仓库；升级测试进 CI | 跨版本升级与混合版本集群 E2E 通过（单机 + 集群） |
| P5 PXE 与收尾 | fledge 适配；CI profile 切换；旧 per-role ISO 下线；文档 | `release-e2e` profile 全绿 |

---

## 13. 决策记录与待定事项

### 13.1 已拍板（2026-11-25）

| # | 事项 | 结论 | 见 |
|---|---|---|---|
| 1 | 安装环境引导框架 | 改用 **Debian live-boot**，不再用 casper | §7.2 |
| 2 | 升级回滚 | **本期不考虑** | §2.2、§10.5 |
| 3 | 已部署的老节点 | **只支持新方案安装的节点**，不考虑老环境迁移 | §2.2 |
| 4 | 集群版本兼容 | **兼容多个版本**（不止 N-1），支持跨版本直接升级 | §10.3 |
| 5 | Ubuntu 快照更新 | 出**只更新 Ubuntu 快照的 ISO**，xvirt 包不变 | §7.5 |
| 6 | 构建用的软件源 | **不依赖网络源**：先把网络源同步为本地镜像，构建只读本地镜像 | §4.1 |
| 7 | 发布 distribution | **按大版本**：`xvirt-1`、`xvirt-2` … | §4.3 |
| 8 | 兼容窗口 | 管理节点**至少兼容 4 个功能版**的计算节点 | §10.3 |
| 9 | Ubuntu 更新版节奏 | **有安全公告时再出**，不按固定周期 | §7.5 |
| 10 | 本地镜像范围 | **尽可能小**，只同步种子及其依赖闭包 | §4.1 |
| 11 | ISO 内容 | **只包含需要的依赖包**，不含 Recommends 等 | §7.6 |
| 12 | 签名私钥 | 放在 **cairn.xve-vt.com**，以签名服务方式提供 | §4.4 |
| 13 | Ubuntu 公网源 | **默认不允许**，界面不提供该选项 | §9.1 |
| 14 | 固件重复 | **必须优化掉**：ISO 只保留 pool 中的 `linux-firmware`，安装环境启动时解包到内存 | §7.6 |
| 15 | 正式密钥签名 | 受保护 tag 触发的发布构建**自动放行** | §4.4 |

### 13.2 仍待定

1. **体积预算**：初定 ISO ≤ 1 GB，P2 实测后确认。
2. **内核扩展模块是否去重**：`linux-modules-extra` 是否像固件一样去重，看 P2 实测体积再定（§7.6）。
3. **签名服务的隔离方式**：cairn.xve-vt.com 上 `xvirt-signer` 与 Cairn 服务本身如何隔离（独立系统用户，还是独立容器），实施 P0 时确定。

---

## 14. 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| maintainer 脚本不规范，升级时重启服务杀掉虚拟机或配置被覆盖 | 升级导致业务中断 | §6.2 规范 + `test-upgrade` 进 CI；升级前进入维护模式 |
| 依赖闭包遗漏（例如虚拟包、可选依赖、硬件固件） | 离线安装或升级失败 | 闭包自检（§7.4 第 4 步）；`restrict=on` 离线冒烟；物理机验证 `linux-firmware` 等 |
| 同名包版本号无法正确比较 | 升级时 apt 认为新包更旧 | §6.1 版本规范 + CI 版本单调检查 |
| Secure Boot 下引导失败 | 部分物理机无法启动 ISO | 只使用 Ubuntu 签名的 shim / grub / kernel；物理机矩阵测试 |
| 装包速度慢于复制镜像 | 安装时间多几分钟 | unsafe-io / eatmydata；接受 |
| 镜像按种子过滤后，加包要先联网同步 | 新增依赖时多一步操作 | compose-repo 缺包时明确报错并提示同步；`mirror-sync.sh` 不动点迭代保证 updates/security 的新依赖不遗漏 |
| 安全公告漏看或响应慢 | 节点长期带着已知漏洞 | `usn-watch` 自动比对并在 Cairn 建 issue；同步失败告警；构建继续使用上一个快照，不阻塞出包 |
| 不装 Recommends 导致功能缺失 | 某些服务的辅助功能不可用 | 需要的包显式写进种子；离线冒烟与物理机测试 |
| 签名私钥所在主机被攻破，或有人滥用 tag 自动放行 | 攻击者可签发恶意仓库 | 私钥只在 cairn.xve-vt.com、最小权限、审计日志、输入校验；`v*` 受保护 tag 只有发布负责人可建；每次正式签名在 Cairn 留记录；离线备份以便吊销轮换；后续迁 HSM |
| 固件解包失败或 initrd 缺固件 | 安装环境中个别网卡/HBA 不可用 | 解包失败不阻断启动并在界面标出；initrd 自动带上早期驱动的固件；物理机矩阵测试 |
| 多版本兼容导致协议/格式包袱 | 开发约束增多，测试矩阵变大 | 能力协商代替版本判断；只做增量协议修改；兼容窗口外才清理旧逻辑；混合集群用例进 CI |
| live-boot 与 Ubuntu resolute 的配合问题 | 安装环境无法启动 | P2 首先验证；必要时在 xvirt 自有源中提供补丁版 live-boot |
