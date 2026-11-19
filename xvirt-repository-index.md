# xvirt 相关代码库索引

生成时间：2026-07-18  
来源：`192.168.60.112:/root/code/xvirt` 当前嵌套 git 仓库 remote / branch / HEAD，以及已有架构与 memory 整理。  
Git 远端：kairos（`git@kairos.xve-vt.com:local/<repo>.git`）；原内网 `192.168.60.105` 与 GitHub remote 已于 2026-11-09 从本地仓库移除

验证补充：2026-07 的仓库可访问性验证见 `xvirt-repository-address-validation.md`（旧内网/GitHub 地址口径，仅作历史）。

2026-11-09 更新：**远端统一为 kairos**（表中地址列已改为 kairos；本地不再保留 `origin`/`github`，GitHub 后续可能由 kairos 自动同步）；**顶层仓改为 `ferrostack`**（本工作区 `/root/work/ferrostack`，见 §0）；`xvirt` 降为子仓，职责收窄为部署/打包/组件说明的集成仓；`cdp-worker`、`xbackup-tools` 已并入 `cdp`；新增 `xdesk-client`、`spdk-analysis`（只读参考）；`virt-control`、`krait`、`xvirt-libs` 标为 legacy。下文表格中的「当前路径」仍是旧工作区的历史记录。

2026-09-27 更新：新增 `cdp` 仓库；`qemu` 默认分支切换为 `stable-11.1-dev`。当前主工作区为 `192.168.60.103:/root/work/ferrostack`，已从 `192.168.60.105` 全量 clone 下表全部仓库并新增 `cdp`（共 25 个）；表中“当前路径”列仍为旧 `/root/code/xvirt` 工作区的历史记录，仅 `cdp` 记录新工作区路径。

## 0. 顶层仓 ferrostack

`ferrostack` 是整个工作区的顶层仓（2026-07-18 创建），各子仓 clone 在其目录下，由其 `.gitignore` 忽略、各自独立提交。

```text
kairos: git@kairos.xve-vt.com:local/ferrostack.git             (kairos)
```

ferrostack 自己跟踪的内容：

- `docs/`：跨组件设计与说明（memtier 系列、资源规划、CDP 改名说明、smart-numa 等）、`demo/`
- 根目录散放的设计稿与索引：`xvirt-architecture.md`、`qemu-memory-tiering-tech-design.md`、`qemu-virtual-desktop-protocol-design.md`、本索引等

与 `xvirt` 的分工：`ferrostack` = 工作区顶层 + 跨组件文档；`xvirt` = 平台集成仓（`CLAUDE.md` 组件表、`xvirt.sh`/`xvirtctl`、cicd、nginx、monitoring、打包部署）。

### 已合并 / 改名 / 废弃一览

| 目录 | 状态 | 去向 |
|---|---|---|
| `cdp-worker` | 已并入 `cdp` | → `cdp/recorder/`（原 `worker/`，现 `cdp-recorder`，保留 `cdpwork` 软链接）；`docs/io-redirection.md` → `cdp/docs/qemu-cdp.md`。**2026-11-11 已从本工作区移除**（连同 `cdp-worker.tar.gz`）；历史在 GitHub `liu-weifeng/cdp-worker`（已归档，含本地最后提交 `624a01f`） |
| `xbackup-tools` | 已并入 `cdp` | → `cdp/tools/xbackup/`（本工作区无此目录） |
| `xvirt-libs` | 疑似已被 `cdp/common` 吸收 | `cdp/common/` 含同名 bitmap/xxhash 源（内容已分叉，未逐行核对）；**2026-11-11 已从本工作区移除**；完整历史在 GitHub `free-backup/xvirt-libs`（`main` = `59ee30c`，与本地删除前一致）。注意 `liu-weifeng/xvirt-libs` 是另一个已归档的空模板，不含这些代码。停在 2026-01 |
| `virt-control` | 废弃 | 被 `ferrovirt` 取代。**2026-11-11 已从本工作区移除**；历史在 GitHub `liu-weifeng/virt-control`（内容与删除前的旧内网 `origin/main` 一致；本地另有一个未推送的文档提交 `53aff34` 已放弃） |
| `krait` | 旧方案 | 旧 Rocky/Anaconda 向导，新装机走 `lycan` + `fledge`。2026-11-11 已迁到 kairos（`main` = `6d8822c`），本地只保留 `kairos` remote |
| `xdesk-client` | 从 qemu/xvirt 抽出 | 2026-07-19 独立成仓 |
| `qemu-patches/`、`qemu-patches.tar.gz`、`resource-planning.tar.gz` | 2026-11-11 已删除 | 前两个是基于 QEMU v10.0.7 的旧补丁导出（未入库，无其他副本）；后一个是 `docs/`、`demo/` 对应文件的旧打包，仍可从本仓历史取回 |

## 1. 仓库地址规则

所有在用仓库统一托管在 kairos：

```text
git@kairos.xve-vt.com:local/<repo>.git        # remote 名 kairos
```

说明：

- 本地各仓库只保留 `kairos` remote（`qemu` 另有 `upstream` 指向 QEMU 官方上游）；旧内网 `ssh://git@192.168.60.105:2222/liuwfng/<repo>.git` 与 GitHub `liu-weifeng/<repo>` 不再配置。
- legacy 仓库：`krait` 已于 2026-11-11 迁到 kairos；`cdp-worker`、`xvirt-libs`、`virt-control` 已从工作区移除（去向见上表）；`spdk-analysis` 的 `origin` 为上游 SPDK。
- 各表中的「当前路径 / 分支 / 当前 HEAD」仍是 2026-07 旧工作区的历史记录，以工作区实际状态为准。

## 2. 核心平台仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责 | kairos 地址 |
|---|---|---:|---:|---|---|---|
| xvirt | `/root/code/xvirt` | main | `8267653` | 集成仓（子仓，顶层为 ferrostack）：CLAUDE.md 组件表、xvirtctl、cicd、nginx、monitoring、打包部署 | `git@kairos.xve-vt.com:local/xvirt.git` |
| ferrovirt | `/root/code/xvirt/ferrovirt` | main | `b1f3466` | Rust 控制面：REST/WS、任务、DB owner、资源模型、agent 编排 | `git@kairos.xve-vt.com:local/ferrovirt.git` |
| xvirt-client | `/root/code/xvirt/xvirt-client` | main | `47df377` | React/Vite Web 控制台，center/standalone 双角色 UI | `git@kairos.xve-vt.com:local/xvirt-client.git` |
| hyper-agent | `/root/code/xvirt/hyper-agent` | main | `a779104` | Python 宿主机 agent：存储/网络/硬件/文件/petrel/memtier，VM 类 action 转 lictor | `git@kairos.xve-vt.com:local/hyper-agent.git` |
| lictor | `/root/code/xvirt/lictor` | main | `2daa170` | Python QMP owner：VM 生命周期、快照、迁移、热插拔、console、xvirt-tools | `git@kairos.xve-vt.com:local/lictor.git` |
| concilium | `/root/code/xvirt/concilium` | main | `7019d54` | C 集群资源管理，驱动 xvirt-vm RA 到 lictor | `git@kairos.xve-vt.com:local/concilium.git` |
| qemu | `/root/code/xvirt/qemu` | stable-11.1-dev | `f10e49dac5` | 自研 QEMU fork：CBT bitmap、memtier、迁移、vhost/petrel 相关改造 | `git@kairos.xve-vt.com:local/qemu.git` |
| kestrel | `/root/work/ferrostack/kestrel` | main | `912e7c6` | Rust 推送式轻量监控框架（拟替代 VictoriaMetrics + exporter）：kestreld 服务端（epoll 主线程 + 同步、封存、查询辅助线程，只依赖 libc）、可按类别配置保留期、突发片段可永久保留、多对象共用的顺序追加段存储、突发片段、Rust / Python / C 三种 SDK；设计见 docs/kestrel-design.md v1.5；2026-11-19 在 10.8.1.195 上编译、57 个测试、端到端都过；1 万对象规模、长范围查询、冷查询并发实测见设计稿 §12.1–§12.3（按类别保留加入之后没重跑规模测试） | `git@kairos.xve-vt.com:local/kestrel.git` |

## 3. 存储、快照、备份与数据面仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责 | kairos 地址 |
|---|---|---:|---:|---|---|---|
| petrel | `/root/code/xvirt/petrel` | main | `0e28b36` | Rust 分布式块存储，mon/osd/vhost/ublk/deploy CLI，xvirt datastore 后端 | `git@kairos.xve-vt.com:local/petrel.git` |
| fulmar | `/root/work/ferrostack/fulmar` | main | `f76e2fc` | Rust 用户态共享盘文件系统（替代 OCFS2）：本地模式 mkfs/fulmard/libfulmar/CLI/fsck 链路齐全，设计见 docs/fulmar-design.md，未编译 | `git@kairos.xve-vt.com:local/fulmar.git` |
| aegis-sdk | `/root/code/xvirt/aegis-sdk` | main | `ac0c5f0` | C VDDK-like 备份 SDK，`libaegisdisk.so`、`aegis-cbt`、CBT 读盘 | `git@kairos.xve-vt.com:local/aegis-sdk.git` |
| cdp-agent | `/root/code/xvirt/cdp-agent` | main | `498f026` | guest 内 CDP：Rust agent + Linux/Windows capture driver + cdp-sink | `git@kairos.xve-vt.com:local/cdp-agent.git` |
| cdp-worker | `/root/code/xvirt/cdp-worker` | main | `624a01f` | **[已并入 cdp → cdp/recorder/；2026-11-11 已从工作区移除，历史见 GitHub `liu-weifeng/cdp-worker`]** 旧 CDP worker，IO 日志、索引、位图与降级模型 | GitHub `liu-weifeng/cdp-worker`（已归档） |
| cdp | `/root/work/ferrostack/cdp` | main | `1dc78e1` | 现行 CDP 实现（CMake：common/plugins/tools/recorder/tests），由 cdp、cdp-worker、xbackup-tools 合并而来；`cdp-recorder`（原 cdpwork）数据面，format-v2 格式；QEMU 侧 QMP 为 `cdp-*`；日志引擎设计稿 `docs/log-engine.md`，Rust 实现 `cdplog/`（按批自描述的格式 v4、io_uring 异步管线、写完文件才确认、持续合并（含打洞、并入 base、紧急回收）、复制的三种搬运模式、回切、多盘一致的同步点、降级组转精简段、ublk 守护进程 `cdplog-ublkd` 及其崩溃恢复和控制口、搬运服务 `cdplog-shipd`）与 nbdkit 前端 `plugins/nbdkit-cdplog-plugin.c`；到 `6efead6` 为止的代码已在 10.8.1.195 上编译测试通过（2026-11-12），`1dc78e1` 只改文档 | `git@kairos.xve-vt.com:local/cdp.git` |
| snapshot-helper | `/root/code/xvirt/snapshot-helper` | main | `6ab1a4f` | QEMU 内存快照 helper，将追加迁移流改为稀疏页覆写文件 | `git@kairos.xve-vt.com:local/snapshot-helper.git` |
| xvirt-snapmig | `/root/code/xvirt/xvirt-snapmig` | main | `805411e` | 跨存储 snapshot chain 迁移工具/数据路径 | `git@kairos.xve-vt.com:local/xvirt-snapmig.git` |
| xvirt-blockdiff | `/root/code/xvirt/xvirt-blockdiff` | main | `0b4c8da` | 块级 diff / xvdiff 相关工具，用于 petrel/OCFS2 snapshot-chain interop | `git@kairos.xve-vt.com:local/xvirt-blockdiff.git` |
| imgtools | `/root/code/xvirt/imgtools` | main | `548f8c7` | C 磁盘镜像工具库和 CLI：raw/qcow2/vmdk、allocmap、convert | `git@kairos.xve-vt.com:local/imgtools.git` |
| xvirt-libs | `/root/code/xvirt/xvirt-libs` | main | `59ee30c` | **[legacy，疑似已被 cdp/common 吸收；2026-11-11 已从工作区移除]** 底层公共库/dirty bitmap 等辅助能力 | GitHub `free-backup/xvirt-libs` |
| ocfs2 | `/root/code/xvirt/ocfs2` | main | `86e3246` | OCFS2 kmod/tools 自构与集成 | `git@kairos.xve-vt.com:local/ocfs2.git` |

## 4. 部署、安装、测试与打包仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责 | kairos 地址 |
|---|---|---:|---:|---|---|---|
| build-deb | `/root/code/xvirt/build-deb` | main | `f5156a5` | QEMU deb 打包，含 epoch、OVMF descriptor、`/usr/libexec/qemu-kvm` 等修复 | `git@kairos.xve-vt.com:local/build-deb.git` |
| xvirt-autotest | `/root/code/xvirt/xvirt-autotest` | main | `14910a4` | Python stdlib 自动化测试框架：PXE、center、storage、network、VM、hotplug | `git@kairos.xve-vt.com:local/xvirt-autotest.git` |
| fledge | `/root/code/xvirt/fledge` | main | `e8be210` | PXE/iPXE 自动装机服务 2.0：按 MAC 静态 IP/VLAN/磁盘/角色无人值守装机、未知 MAC 交互引导菜单、实时进度回调与防重装解锁 | `git@kairos.xve-vt.com:local/fledge.git` |
| lycan | `/root/code/xvirt/lycan` | master | `0ff22b6` | Ubuntu 26.04 安装器 2.0：ESXi/Subiquity 风格 Curses TUI、网络(VLAN/MTU/Bond)/磁盘选型与可视化分区/账号与 SSH 密钥、PXE 与自动应答 | `git@kairos.xve-vt.com:local/lycan.git` |
| krait | `/root/code/xvirt/krait` | main | `6d8822c` | **[legacy]** 旧 Rocky/Anaconda 安装向导 UI 与 ISO 集成，已被 lycan + fledge 取代 | `git@kairos.xve-vt.com:local/krait.git` |

## 5. Guest、旧控制面与历史仓库

| 仓库 | 当前路径 | 分支 | 当前 HEAD | 职责/状态 | kairos 地址 |
|---|---|---:|---:|---|---|---|
| xvirt-tools | `/root/code/xvirt/xvirt-tools` | main | `787b231` | Go guest tools，安装在 VM 内，通过 virtio-serial `xvirt-tools.0` 与 host 通信 | `git@kairos.xve-vt.com:local/xvirt-tools.git` |
| virt-control | `/root/code/xvirt/virt-control` | main | `6bc631a` | **[废弃，2026-11-11 已从工作区移除，历史见 GitHub `liu-weifeng/virt-control`]** Java/Spring 旧控制面，已被 ferrovirt 替代；保留作历史参考，不再启动 | GitHub `liu-weifeng/virt-control` |

## 5.1 2026-07 之后新增的目录

| 仓库 | 当前路径 | 分支 | 职责 | kairos 地址 |
|---|---|---:|---|---|
| ferrostack | `/root/work/ferrostack` | main | 顶层仓，见 §0 | `git@kairos.xve-vt.com:local/ferrostack.git` |
| xdesk-client | `/root/work/ferrostack/xdesk-client` | main | xdesk 远程桌面客户端（2026-07-19 从 xvirt QEMU fork 抽出） | `git@kairos.xve-vt.com:local/xdesk-client.git` |
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

- `virt-control`：旧 Java 控制面，已废弃，2026-11-11 已从工作区移除。
- `krait`：旧 Rocky/Anaconda 安装路线，仍有历史资料价值。
- `cdp-worker`：已并入 `cdp/recorder/`，2026-11-11 已从工作区移除（GitHub 有归档）。
- `xvirt-libs`：疑似已被 `cdp/common` 吸收，2026-11-11 已从工作区移除（GitHub `free-backup/xvirt-libs`）。

## 7. clone 示例（kairos）

```bash
mkdir -p /root/work && cd /root/work
git clone git@kairos.xve-vt.com:local/ferrostack.git
cd ferrostack
for r in aegis-sdk build-deb cdp cdp-agent concilium ferrovirt fledge hyper-agent \
         imgtools lictor lycan ocfs2 petrel qemu snapshot-helper xdesk-client xvirt \
         xvirt-autotest xvirt-blockdiff xvirt-client xvirt-snapmig xvirt-tools
do
  git clone git@kairos.xve-vt.com:local/$r.git
done
```

子仓被 ferrostack 的 `.gitignore` 忽略，各自独立提交。`qemu` 默认分支为 `stable-11.1-dev`，CDP 改名工作在 `cdp-rename` 分支。

## 8. 旧工作区切 remote 到 kairos

```bash
cd /root/work/ferrostack
for d in . aegis-sdk build-deb cdp cdp-agent concilium ferrovirt fledge hyper-agent \
         imgtools lictor lycan ocfs2 petrel qemu snapshot-helper xdesk-client xvirt \
         xvirt-autotest xvirt-blockdiff xvirt-client xvirt-snapmig xvirt-tools
do
  repo=$(basename "$(cd "$d" && pwd)"); [ "$d" = . ] && repo=ferrostack
  git -C "$d" remote add kairos "git@kairos.xve-vt.com:local/${repo}.git" 2>/dev/null \
    || git -C "$d" remote set-url kairos "git@kairos.xve-vt.com:local/${repo}.git"
  git -C "$d" remote remove origin 2>/dev/null; git -C "$d" remote remove github 2>/dev/null
done
```

注意：kairos 上多数仓库的历史是重新导入的，与旧 origin 无共同祖先；切换后先 `git fetch kairos`，以 kairos 为准重置，再 cherry-pick 本地独有提交，不要强推。

## 9. 需要后续核对的点

- kairos → GitHub 的自动同步是否启用、同步范围与方向。
- 哪些仓库是私有、哪些公开，交接时需要标注权限。
- `build-deb-xvirt` 当前是否也独立成 git 仓库；当前扫描结果未发现其 `.git`，它可能属于集成仓 `xvirt`。
- `monitoring/`、`nginx/`、`lab-env/` 等目录当前没有单独 `.git`，应视为集成仓 `xvirt` 的内容，除非后续拆仓。
- `xvirt-libs` 是否确认已被 `cdp/common` 吸收；`krait` 是否仍有使用。
