# xvirt 相关代码库索引

生成时间：2026-07-18  
来源：`192.168.60.112:/root/code/xvirt` 当前嵌套 git 仓库 remote / branch / HEAD，以及已有架构与 memory 整理。  
新内网仓库 IP：`192.168.60.105`  
GitHub 远端：`https://github.com/liu-weifeng`

验证补充：仓库可访问性验证结果见 `xvirt-repository-address-validation.md`。GitHub 私有仓应使用 SSH 地址 `git@github.com:liu-weifeng/<repo>.git` 验证；本索引中的 GitHub HTTPS 地址主要用于阅读/展示。

2026-11-09 更新：**顶层仓改为 `ferrostack`**（本工作区 `/root/work/ferrostack`，见 §0）；`xvirt` 降为子仓，职责收窄为部署/打包/组件说明的集成仓；`cdp-worker`、`xbackup-tools` 已并入 `cdp`；新增 `xdesk-client`、`spdk-analysis`（只读参考）；`virt-control`、`krait`、`xvirt-libs` 标为 legacy。下文表格中的「当前路径」仍是旧工作区的历史记录。

2026-09-27 更新：新增 `cdp` 仓库；`qemu` 默认分支切换为 `stable-11.1-dev`。当前主工作区为 `192.168.60.103:/root/work/ferrostack`，已从 `192.168.60.105` 全量 clone 下表全部仓库并新增 `cdp`（共 25 个）；表中“当前路径”列仍为旧 `/root/code/xvirt` 工作区的历史记录，仅 `cdp` 记录新工作区路径。

## 0. 顶层仓 ferrostack

`ferrostack` 是整个工作区的顶层仓（2026-07-18 创建），各子仓 clone 在其目录下，由其 `.gitignore` 忽略、各自独立提交。

```text
内网:   ssh://git@192.168.60.105:2222/liuwfng/ferrostack.git   (origin)
GitHub: git@github.com:liu-weifeng/ferrostack.git             (github)
kairos: git@kairos.xve-vt.com:local/ferrostack.git             (kairos)
```

ferrostack 自己跟踪的内容：

- `docs/`：跨组件设计与说明（memtier 系列、资源规划、CDP 改名说明、smart-numa 等）、`demo/`
- 根目录散放的设计稿与索引：`xvirt-architecture.md`、`qemu-memory-tiering-tech-design.md`、`qemu-virtual-desktop-protocol-design.md`、本索引等

与 `xvirt` 的分工：`ferrostack` = 工作区顶层 + 跨组件文档；`xvirt` = 平台集成仓（`CLAUDE.md` 组件表、`xvirt.sh`/`xvirtctl`、cicd、nginx、monitoring、打包部署）。

### 已合并 / 改名 / 废弃一览

| 目录 | 状态 | 去向 |
|---|---|---|
| `cdp-worker` | 已并入 `cdp` | → `cdp/recorder/`（原 `worker/`，现 `cdp-recorder`，保留 `cdpwork` 软链接）；`docs/io-redirection.md` → `cdp/docs/qemu-cdp.md`。目录仅作历史，停在 2026-02 |
| `xbackup-tools` | 已并入 `cdp` | → `cdp/tools/xbackup/`（本工作区无此目录） |
| `xvirt-libs` | 疑似已被 `cdp/common` 吸收 | `cdp/common/` 含同名 bitmap/xxhash 源（内容已分叉，未逐行核对）；停在 2026-01 |
| `virt-control` | 废弃 | 被 `ferrovirt` 取代，不再启动，仅作历史参考 |
| `krait` | 旧方案 | 旧 Rocky/Anaconda 向导，新装机走 `lycan` + `fledge` |
| `xdesk-client` | 从 qemu/xvirt 抽出 | 2026-07-19 独立成仓 |
| `qemu-patches/`、`*.tar.gz` | 旧导出物 | 基于 QEMU v10.0.7 的补丁导出与旧打包，当前 qemu 在 11.1，勿当现行源码 |

## 1. 仓库地址规则

当前 `/root/code/xvirt` 工作区里，多数 remote 仍指向旧地址：

```text
ssh://git@192.168.2.107:2222/liuwfng/<repo>.git
```

按你提供的新信息，后续建议统一记录为：

```text
内网 SSH:
ssh://git@192.168.60.105:2222/liuwfng/<repo>.git

GitHub HTTPS:
https://github.com/liu-weifeng/<repo>.git

GitHub SSH:
git@github.com:liu-weifeng/<repo>.git
```

说明：

- `192.168.60.105` 的端口和 namespace 这里按现有旧 remote 约定推断为 `:2222/liuwfng`。如果新 Git 服务改了端口或 namespace，需要批量修正。
- GitHub 仓库统一按 `liu-weifeng/<repo>` 记录；如果部分仓库是私有仓或未镜像到 GitHub，需要以后在 GitHub 侧核对。
- 当前检出信息来自服务器工作区，可能不是远端最新状态。

## 2. 核心平台仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责 | 新内网地址 | GitHub 地址 |
|---|---|---:|---:|---|---|---|
| xvirt | `/root/code/xvirt` | main | `4b09dac` | 集成仓（子仓，顶层为 ferrostack）：CLAUDE.md 组件表、xvirtctl、cicd、nginx、monitoring、打包部署 | `ssh://git@192.168.60.105:2222/liuwfng/xvirt.git` | `https://github.com/liu-weifeng/xvirt.git` |
| ferrovirt | `/root/code/xvirt/ferrovirt` | main | `b1f3466` | Rust 控制面：REST/WS、任务、DB owner、资源模型、agent 编排 | `ssh://git@192.168.60.105:2222/liuwfng/ferrovirt.git` | `https://github.com/liu-weifeng/ferrovirt.git` |
| xvirt-client | `/root/code/xvirt/xvirt-client` | main | `47df377` | React/Vite Web 控制台，center/standalone 双角色 UI | `ssh://git@192.168.60.105:2222/liuwfng/xvirt-client.git` | `https://github.com/liu-weifeng/xvirt-client.git` |
| hyper-agent | `/root/code/xvirt/hyper-agent` | main | `a779104` | Python 宿主机 agent：存储/网络/硬件/文件/petrel/memtier，VM 类 action 转 lictor | `ssh://git@192.168.60.105:2222/liuwfng/hyper-agent.git` | `https://github.com/liu-weifeng/hyper-agent.git` |
| lictor | `/root/code/xvirt/lictor` | main | `2daa170` | Python QMP owner：VM 生命周期、快照、迁移、热插拔、console、xvirt-tools | `ssh://git@192.168.60.105:2222/liuwfng/lictor.git` | `https://github.com/liu-weifeng/lictor.git` |
| concilium | `/root/code/xvirt/concilium` | main | `7019d54` | C 集群资源管理，驱动 xvirt-vm RA 到 lictor | `ssh://git@192.168.60.105:2222/liuwfng/concilium.git` | `https://github.com/liu-weifeng/concilium.git` |
| qemu | `/root/code/xvirt/qemu` | stable-11.1-dev | `46c00088e5` | 自研 QEMU fork：CBT bitmap、memtier、迁移、vhost/petrel 相关改造 | `ssh://git@192.168.60.105:2222/liuwfng/qemu.git` | `https://github.com/liu-weifeng/qemu.git` |

## 3. 存储、快照、备份与数据面仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责 | 新内网地址 | GitHub 地址 |
|---|---|---:|---:|---|---|---|
| petrel | `/root/code/xvirt/petrel` | main | `0e28b36` | Rust 分布式块存储，mon/osd/vhost/ublk/deploy CLI，xvirt datastore 后端 | `ssh://git@192.168.60.105:2222/liuwfng/petrel.git` | `https://github.com/liu-weifeng/petrel.git` |
| aegis-sdk | `/root/code/xvirt/aegis-sdk` | main | `ac0c5f0` | C VDDK-like 备份 SDK，`libaegisdisk.so`、`aegis-cbt`、CBT 读盘 | `ssh://git@192.168.60.105:2222/liuwfng/aegis-sdk.git` | `https://github.com/liu-weifeng/aegis-sdk.git` |
| cdp-agent | `/root/code/xvirt/cdp-agent` | main | `498f026` | guest 内 CDP：Rust agent + Linux/Windows capture driver + cdp-sink | `ssh://git@192.168.60.105:2222/liuwfng/cdp-agent.git` | `https://github.com/liu-weifeng/cdp-agent.git` |
| cdp-worker | `/root/code/xvirt/cdp-worker` | main | `624a01f` | **[已并入 cdp → cdp/recorder/，仅作历史]** 旧 CDP worker，IO 日志、索引、位图与降级模型 | `ssh://git@192.168.60.105:2222/liuwfng/cdp-worker.git` | `https://github.com/liu-weifeng/cdp-worker.git` |
| cdp | `/root/work/ferrostack/cdp` | main | `98dddeb`+ | 现行 CDP 实现（CMake：common/plugins/tools/recorder/tests），由 cdp、cdp-worker、xbackup-tools 合并而来；`cdp-recorder`（原 cdpwork）数据面，format-v2 格式；QEMU 侧 QMP 为 `cdp-*` | `ssh://git@192.168.60.105:2222/liuwfng/cdp.git` | `https://github.com/liu-weifeng/cdp.git` |
| snapshot-helper | `/root/code/xvirt/snapshot-helper` | main | `6ab1a4f` | QEMU 内存快照 helper，将追加迁移流改为稀疏页覆写文件 | `ssh://git@192.168.60.105:2222/liuwfng/snapshot-helper.git` | `https://github.com/liu-weifeng/snapshot-helper.git` |
| xvirt-snapmig | `/root/code/xvirt/xvirt-snapmig` | main | `805411e` | 跨存储 snapshot chain 迁移工具/数据路径 | `ssh://git@192.168.60.105:2222/liuwfng/xvirt-snapmig.git` | `https://github.com/liu-weifeng/xvirt-snapmig.git` |
| xvirt-blockdiff | `/root/code/xvirt/xvirt-blockdiff` | main | `0b4c8da` | 块级 diff / xvdiff 相关工具，用于 petrel/OCFS2 snapshot-chain interop | `ssh://git@192.168.60.105:2222/liuwfng/xvirt-blockdiff.git` | `https://github.com/liu-weifeng/xvirt-blockdiff.git` |
| imgtools | `/root/code/xvirt/imgtools` | main | `548f8c7` | C 磁盘镜像工具库和 CLI：raw/qcow2/vmdk、allocmap、convert | `ssh://git@192.168.60.105:2222/liuwfng/imgtools.git` | `https://github.com/liu-weifeng/imgtools.git` |
| xvirt-libs | `/root/code/xvirt/xvirt-libs` | main | `59ee30c` | **[legacy，疑似已被 cdp/common 吸收]** 底层公共库/dirty bitmap 等辅助能力 | `ssh://git@192.168.60.105:2222/liuwfng/xvirt-libs.git` | `https://github.com/liu-weifeng/xvirt-libs.git` |
| ocfs2 | `/root/code/xvirt/ocfs2` | main | `86e3246` | OCFS2 kmod/tools 自构与集成 | `ssh://git@192.168.60.105:2222/liuwfng/ocfs2.git` | `https://github.com/liu-weifeng/ocfs2.git` |

## 4. 部署、安装、测试与打包仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责 | 新内网地址 | GitHub 地址 |
|---|---|---:|---:|---|---|---|
| build-deb | `/root/code/xvirt/build-deb` | main | `f5156a5` | QEMU deb 打包，含 epoch、OVMF descriptor、`/usr/libexec/qemu-kvm` 等修复 | `ssh://git@192.168.60.105:2222/liuwfng/build-deb.git` | `https://github.com/liu-weifeng/build-deb.git` |
| xvirt-autotest | `/root/code/xvirt/xvirt-autotest` | main | `14910a4` | Python stdlib 自动化测试框架：PXE、center、storage、network、VM、hotplug | `ssh://git@192.168.60.105:2222/liuwfng/xvirt-autotest.git` | `https://github.com/liu-weifeng/xvirt-autotest.git` |
| fledge | `/root/code/xvirt/fledge` | main | `6ed351c` | PXE provisioning 服务，按 MAC/角色/磁盘自动装机 | `ssh://git@192.168.60.105:2222/liuwfng/fledge.git` | `https://github.com/liu-weifeng/fledge.git` |
| lycan | `/root/code/xvirt/lycan` | master | `947749a` | Ubuntu 26.04 autoinstall ISO 构建器，standalone/management 自动应答 | `ssh://git@192.168.60.105:2222/liuwfng/lycan.git` | `https://github.com/liu-weifeng/lycan.git` |
| krait | `/root/code/xvirt/krait` | main | `6d8822c` | **[legacy]** 旧 Rocky/Anaconda 安装向导 UI 与 ISO 集成，已被 lycan + fledge 取代 | `ssh://git@192.168.60.105:2222/liuwfng/krait.git` | `https://github.com/liu-weifeng/krait.git` |

## 5. Guest、旧控制面与历史仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责/状态 | 新内网地址 | GitHub 地址 |
|---|---|---:|---:|---|---|---|
| xvirt-tools | `/root/code/xvirt/xvirt-tools` | main | `787b231` | Go guest tools，安装在 VM 内，通过 virtio-serial `xvirt-tools.0` 与 host 通信 | `ssh://git@192.168.60.105:2222/liuwfng/xvirt-tools.git` | `https://github.com/liu-weifeng/xvirt-tools.git` |
| virt-control | `/root/code/xvirt/virt-control` | main | `6bc631a` | **[废弃]** Java/Spring 旧控制面，已被 ferrovirt 替代；保留作历史参考，不再启动 | `ssh://git@192.168.60.105:2222/liuwfng/virt-control.git` | `https://github.com/liu-weifeng/virt-control.git` |

## 5.1 2026-07 之后新增的目录

| 仓库 | 当前路径 | 分支 | 职责 | 内网地址 |
|---|---|---:|---|---|
| ferrostack | `/root/work/ferrostack` | main | 顶层仓，见 §0 | `ssh://git@192.168.60.105:2222/liuwfng/ferrostack.git` |
| xdesk-client | `/root/work/ferrostack/xdesk-client` | main | xdesk 远程桌面客户端（2026-07-19 从 xvirt QEMU fork 抽出） | `ssh://git@192.168.60.105:2222/liuwfng/xdesk-client.git` |
| spdk-analysis | `/root/work/ferrostack/spdk-analysis` | — | 上游 SPDK clone（`github.com/spdk/spdk`），仅供读代码分析，非自有仓 | 上游 |

## 6. 推荐分组视图

### 6.1 日常开发最常改

- `ferrovirt`
- `xvirt-client`
- `hyper-agent`
- `lictor`
- `petrel`
- `qemu`
- `xvirt-autotest`

### 6.2 存储/备份专题

- `petrel`
- `aegis-sdk`
- `xvirt-snapmig`
- `xvirt-blockdiff`
- `snapshot-helper`
- `imgtools`
- `cdp-agent`
- `cdp`（含原 cdp-worker → `recorder/`、xbackup-tools → `tools/xbackup/`）
- ~~`xvirt-libs`~~（legacy）

### 6.3 安装部署专题

- `build-deb`
- `xvirt`
- `lycan`
- `fledge`
- `xvirt-autotest`
- `krait`
- `ocfs2`

### 6.4 历史/参考

- `virt-control`：旧 Java 控制面，已废弃。
- `krait`：旧 Rocky/Anaconda 安装路线，仍有历史资料价值。
- `cdp-worker`：已并入 `cdp/recorder/`，目录仅作历史。
- `xvirt-libs`：疑似已被 `cdp/common` 吸收。

## 7. clone 示例

### 7.1 从新内网 Git 服务 clone

```bash
mkdir -p /root/code/xvirt-new && cd /root/code/xvirt-new
git clone ssh://git@192.168.60.105:2222/liuwfng/xvirt.git
git clone ssh://git@192.168.60.105:2222/liuwfng/ferrovirt.git
git clone ssh://git@192.168.60.105:2222/liuwfng/xvirt-client.git
git clone ssh://git@192.168.60.105:2222/liuwfng/hyper-agent.git
git clone ssh://git@192.168.60.105:2222/liuwfng/lictor.git
git clone ssh://git@192.168.60.105:2222/liuwfng/concilium.git
git clone ssh://git@192.168.60.105:2222/liuwfng/petrel.git
git clone ssh://git@192.168.60.105:2222/liuwfng/qemu.git
git clone ssh://git@192.168.60.105:2222/liuwfng/build-deb.git
git clone ssh://git@192.168.60.105:2222/liuwfng/xvirt-autotest.git
```

### 7.2 从 GitHub clone

```bash
mkdir -p ~/code/xvirt-github && cd ~/code/xvirt-github
git clone https://github.com/liu-weifeng/xvirt.git
git clone https://github.com/liu-weifeng/ferrovirt.git
git clone https://github.com/liu-weifeng/xvirt-client.git
git clone https://github.com/liu-weifeng/hyper-agent.git
git clone https://github.com/liu-weifeng/lictor.git
git clone https://github.com/liu-weifeng/concilium.git
git clone https://github.com/liu-weifeng/petrel.git
git clone https://github.com/liu-weifeng/qemu.git
git clone https://github.com/liu-weifeng/build-deb.git
git clone https://github.com/liu-weifeng/xvirt-autotest.git
```

## 8. 批量切 remote 示例

在现有 `/root/code/xvirt` 工作区中，可用下面思路将旧内网 remote 切到新内网。执行前建议先 `git status` 和备份。

```bash
cd /root/code/xvirt

for d in . aegis-sdk build-deb cdp-agent cdp-worker concilium ferrovirt fledge \
         hyper-agent imgtools krait lictor lycan ocfs2 petrel qemu snapshot-helper \
         virt-control xvirt-autotest xvirt-blockdiff xvirt-client xvirt-libs \
         xvirt-snapmig xvirt-tools
do
  repo=${d##*/}
  [ "$d" = "." ] && repo=xvirt
  git -C "$d" remote set-url origin "ssh://git@192.168.60.105:2222/liuwfng/${repo}.git"
done
```

如需同时保留 GitHub mirror，可添加第二 remote：

```bash
git remote add github "git@github.com:liu-weifeng/<repo>.git"
```

## 9. 需要后续核对的点

- `192.168.60.105` 是否继续使用 SSH 端口 `2222` 和 namespace `liuwfng`。
- GitHub 上是否所有内网仓库都已镜像到 `liu-weifeng/<repo>`。
- 哪些仓库是私有、哪些公开，交接时需要标注权限。
- `build-deb-xvirt` 当前是否也独立成 git 仓库；当前扫描结果未发现其 `.git`，它可能属于集成仓 `xvirt`。
- `monitoring/`、`nginx/`、`lab-env/` 等目录当前没有单独 `.git`，应视为集成仓 `xvirt` 的内容，除非后续拆仓。
- `xvirt-libs` 是否确认已被 `cdp/common` 吸收；`krait` 是否仍有使用。
- §7、§8 的 clone / 批量切 remote 示例是 2026-07 的旧工作区口径，新工作区应先 clone `ferrostack`，再按 §0 的 `.gitignore` 列表逐个 clone 子仓。
