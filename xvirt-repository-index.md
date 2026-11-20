# xvirt 相关代码库索引

初版：2026-07-18（来源 `192.168.60.112:/root/code/xvirt` 旧工作区）。  
刷新：2026-11-20。数据直接读自 `/root/work/ferrostack/<repo>/.git`（HEAD、本地分支、kairos 远端跟踪分支、FETCH_HEAD 时间、remote 配置）。

- 表中所有在用仓库的本地分支 HEAD 都与 kairos 远端跟踪分支一致，没有未推送的提交。
- 远端跟踪分支只反映各仓**上次 fetch**时的状态（见"上次 fetch"列，多数是 2026-11-09），kairos 上之后有没有别人推的新提交，要 fetch 才知道。
- 组件架构见 `xvirt-architecture.md`。

## 0. 顶层仓 ferrostack

`ferrostack` 是工作区顶层仓（2026-07-18 创建，工作区 `/root/work/ferrostack`）。各子仓 clone 在它下面，由它的 `.gitignore` 忽略、各自独立提交。

```text
kairos: git@kairos.xve-vt.com:local/ferrostack.git
```

ferrostack 自己跟踪的内容：

- `docs/`：跨组件设计与说明。包括 memtier 系列、资源规划、内存快照、drive-mirror 厚薄盘、smart-numa、直采指标、CDP 改名说明、kestrel 设计稿、fulmar 设计稿。另有 `demo/`。
- 根目录：`xvirt-architecture.md`（平台架构总览）、本索引、`qemu-memory-tiering-tech-design.md`、`qemu-virtual-desktop-protocol-design.md`、`xvirt-claude-memory-*`（交接用的历史记录）、`xvirt-repository-address-validation.md`（2026-07 旧地址验证，仅作历史）。

与 `xvirt` 的分工：`ferrostack` 是工作区顶层，放跨组件文档；`xvirt` 是平台集成仓（`CLAUDE.md` 组件表、`xvirtctl`、nginx、monitoring、`build-deb-xvirt` 打包、旧 cicd）。

### 已合并 / 改名 / 废弃一览

| 目录 | 状态 | 去向 |
|---|---|---|
| `cdp-worker` | 已并入 `cdp` | → `cdp/recorder/`（二进制 `cdp-recorder`，保留 `cdpwork` 软链接）；`docs/io-redirection.md` → `cdp/docs/qemu-cdp.md`。2026-11-11 从工作区移除；历史在 GitHub `liu-weifeng/cdp-worker`（已归档，含本地最后提交 `624a01f`） |
| `xbackup-tools` | 已并入 `cdp` | → `cdp/tools/xbackup/` |
| `xvirt-libs` | 疑似已被 `cdp/common` 吸收 | `cdp/common/` 含同名 bitmap/xxhash 源（内容已分叉，未逐行核对）。2026-11-11 从工作区移除；完整历史在 GitHub `free-backup/xvirt-libs`（`main` = `59ee30c`）。注意 `liu-weifeng/xvirt-libs` 是另一个已归档的空模板 |
| `virt-control` | 废弃 | 被 `ferrovirt` 取代。2026-11-11 从工作区移除；历史在 GitHub `liu-weifeng/virt-control` |
| `krait` | legacy | 旧 Rocky/Anaconda 安装向导，新装机走 `lycan` + `fledge`。仍保留在工作区，已迁到 kairos |
| `xdesk-client` | 从 qemu 抽出 | 2026-07-19 独立成仓 |
| `qemu-patches/`、`qemu-patches.tar.gz`、`resource-planning.tar.gz` | 2026-11-11 已删除 | 前两个是基于 QEMU v10.0.7 的旧补丁导出（未入库，无其他副本；可能含 CBT bitmap export/import，该功能已在 11.1 上重写，见 §2.1）；后一个可从本仓历史取回 |

## 1. 仓库地址规则

所有在用仓库统一托管在 kairos，各仓本地只配 `kairos` 这一个 remote：

```text
git@kairos.xve-vt.com:local/<repo>.git
```

例外：

- `qemu` 另有 `upstream`（`https://gitlab.com/qemu-project/qemu.git`）。
- `spdk-analysis` 只有上游 SPDK（`https://github.com/spdk/spdk.git`），不是自有仓。

旧内网 `ssh://git@192.168.60.105:2222/liuwfng/<repo>.git` 和 GitHub `liu-weifeng/<repo>` 已于 2026-11-09 从本地仓库移除。

## 2. 核心平台仓库

| 仓库 | 分支 | HEAD | 上次 fetch | 职责 / 现状 |
|---|---|---|---|---|
| xvirt | main | `771b4c0` | 11-09 | 集成仓：`CLAUDE.md` 组件表、`xvirtctl`、nginx、monitoring（VictoriaMetrics 栈）、`build-deb-xvirt`（8 个平台 deb + `xvirt-init`）；`README.md`、`PACKAGING.md`、`cicd/` 仍是 RPM/virt-control 旧口径 |
| ferrovirt | main | `b0189fd` | 11-09 | Rust 控制面：REST/WS、任务、DB owner、center↔节点 ZMQ（5570/5571）、周期状态对账、资源规划与放置、在线迁移与跨存储快照链迁移编排、内存分层快照落库 |
| xvirt-client | main | `830f6dc` | 11-09 | React/Vite Web 控制台，center/standalone 双角色；主机详情有资源规划页（拓扑、内存、分层水位线、节点健康、自动放置） |
| hyper-agent | main | `e80ec6d` | 11-09 | Python 宿主机 agent：存储/网络/硬件/文件/petrel；资源规划 apply 与放置控制器；memtier 总控与 NVMe 写入预算；VM 类 action 转 lictor 或 concilium |
| lictor | main | `317ee3d` | 11-09 | Python QMP owner：VM 生命周期、快照（含内存分层快照 keep/merge）、迁移、热插拔、xvirt-tools；systemd scope 与绑核 |
| concilium | main | `4eeea1e` | 11-09 | C 集群资源管理：gossip :7700、选主、HA、self-fence + STONITH，驱动 `xvirt-vm` RA 到 lictor；迁移 RA 仍是桩 |
| qemu | stable-11.1-dev（当前检出 memtier-heat） | kairos `b63be76397` | 11-20 | 自研 QEMU fork，基线 11.1.1。memtier 全套（含热度直方图、swap-cache）、CDP io-redirection、CBT 位图导出/导入、热度 pre-copy 迁移、ram-layout、qcow2 在线缩盘、xdesk。分支见 §2.1 |
| kestrel | main | `912e7c6` | 11-19 | Rust 推送式轻量监控框架（拟替代 VictoriaMetrics + exporter）：kestreld + Rust/Python/C SDK，按类别保留、突发片段永久保留；设计见 `docs/kestrel-design.md` v1.5；2026-11-19 在 10.8.1.195 上编译、57 个测试、端到端都过；尚无生产组件接入 |

### 2.1 qemu 分支

| 分支 | kairos | 内容 | 状态 |
|---|---|---|---|
| `stable-11.1-dev` | `b63be76397` | 默认分支。2026-09-27 从 `upstream/stable-11.1` 建出并 cherry-pick 全部自研改动，2026-11-10 合入 memtier-stats + memtier-heat，2026-11-20 合入 CBT 位图导出/导入（squash） | 产品线，`build-deb` 打这个分支。**本地同名分支（`424651e`）是旧的分叉副本，不要推它**，合并以 `kairos/stable-11.1-dev` 为准 || `memtier-stats` | 有 | 写盘统计、零页不落盘、swap-cache、分块日志打洞/压实、`resident-cold-bytes` | 已合入 |
| `memtier-heat` | `d9cff878a1` | `x-memtier-query-heat` 热度直方图、抽样扫描、先探测 page_idle 再回退 write-aging | 已合入；本地当前检出 |
| `cdp-rename` | `3ef92ae` | QMP `io-redirection-*` 改名为 `cdp-*`，并修写钩子范围、写零、fifo、阻塞模式 | **未合入、未编译** |
| `memory-tiering` | 有 | memtier 早期开发分支（10.0 血统） | 历史，内容已进 11.1 |
| `desktop` | 仅本地 | xdesk 阶段一（10.0 基线） | 已 cherry-pick 进 11.1 |
| `stable-10.0(-dev)` | 有 | 10.0 血统产品线 | 历史 |
| `cdp-upgrade`、`cdp-timestamp-ns` | 仅本地 | CDP 相关实验分支 | 未确认 |
| `qemu-10.0-cdp-port-*`、`x-query-ram-layout-*` | 有 | 临时移植分支 | 历史 |

## 3. 存储、快照、备份与数据面仓库

| 仓库 | 分支 | HEAD | 上次 fetch | 职责 / 现状 |
|---|---|---|---|---|
| petrel | main | `dff698b` | 11-09 | Rust 分布式块存储：mon/osd/vhost/ublk/deploy/manager/agent/snapdiff/bench；副本/EC、按设备类分池、COW 快照/克隆/resize；Mon Raft 化进行中 |
| fulmar | main | `f76e2fc` | — | Rust 用户态共享盘文件系统（替代 OCFS2）：本地模式 mkfs/fulmard/libfulmar/CLI/fsck 链路已写，设计见 `docs/fulmar-design.md` v1.1，**未编译未测试** |
| aegis-sdk | main | `ac0c5f0` | 11-09 | C 类 VDDK 备份 SDK：`libaegisdisk.so`、`aegis-cbt`、CBT 读盘、本地模式 `Aegis_OpenLocal` |
| cdp | main | `1dc78e1` | 11-12 | 现行 CDP（由 cdp、cdp-worker、xbackup-tools 合并）：`recorder/`（`cdp-recorder`，v2 格式）、`cdplog/`（Rust 日志引擎：ublk 守护进程 `cdplog-ublkd`、搬运服务 `cdplog-shipd`、nbdkit 插件）、VMware 恢复工具；到 `6efead6` 的代码已在 10.8.1.195 编译测试通过，`1dc78e1` 只改文档；设计见 `docs/log-engine.md` |
| cdp-agent | main | `498f026` | 11-09 | guest 内 CDP：Rust agent + Linux `dm-cdp` / Windows `cdpflt` 驱动 + cdp-sink（iolog/disk/replica） |
| snapshot-helper | main | `d3aa196` | 11-09 | QEMU 内存快照 helper：迁移流改为稀疏页覆写文件；`--memtier=keep\|merge` 内存分层快照（已实现未真机验证） |
| xvirt-snapmig | main | `ed14e8f` | 11-09 | 跨存储快照树迁移（`offline-tree`：根全量 + 逐层 reflink/merge） |
| xvirt-blockdiff | main | `eb6c78e` | 11-09 | FIEMAP 块级 diff 与 `.xvdiff` 工具（diff/merge/apply/extract/splat/show），用于 OCFS2 ↔ petrel 快照链互迁 |
| imgtools | main | `574909b` | 11-09 | C 磁盘镜像工具库和 CLI：raw/qcow2/vmdk、allocmap、convert |
| ocfs2 | main | `ae425dc` | 11-09 | OCFS2 kmod/tools 按内核版本自构（现行共享 datastore；fulmar 拟替代） |

## 4. 部署、安装与测试仓库

| 仓库 | 分支 | HEAD | 上次 fetch | 职责 / 现状 |
|---|---|---|---|---|
| build-deb | main | `9f6de00` | 11-09 | QEMU deb 打包（`qemu-kvm-xvirt 11.1.1`，epoch 2，9 个包）与 Jenkins 流水线 `ci/`（deb-qemu / deb-xvirt / iso-xvirt，Jenkins 在 192.168.60.112:8081） |
| lycan | master | `947749a` | 11-09 | Ubuntu 26.04 离线 TUI 安装器与 ISO 构建：网络（VLAN/MTU/Bond）、磁盘分区、账号与 SSH 密钥、PXE 与按角色的自动应答 |
| fledge | main | `6f82d03` | 11-09 | PXE/iPXE 自动装机服务：按 MAC 静态 IP/VLAN/磁盘/角色无人值守装机、未知 MAC 交互菜单、进度回调与防重装 |
| xvirt-autotest | main | `2d072a2` | 11-09 | Python stdlib 自动化测试：package → iso → PXE → center / storage / network / VM（含 memtier）；最近全量 2026-06-28 为 71/0/5 |
| krait | main | `6d8822c` | 11-11 | **[legacy]** 旧 Rocky/Anaconda 安装向导，已被 lycan + fledge 取代 |

## 5. Guest、桌面与参考仓库

| 仓库 | 分支 | HEAD | 上次 fetch | 职责 / 现状 |
|---|---|---|---|---|
| xvirt-tools | main | `a17b8f2` | 11-09 | Go guest tools，装在 VM 内，经 virtio-serial `xvirt-tools.0` 与 host 通信 |
| xdesk-client | main | `38572e1` | 11-09 | xdesk 远程桌面客户端（GTK4 + GStreamer，阶段一），服务端是 QEMU `-display xdesk`；控制面未接入 |
| spdk-analysis | master | `d33b1df` | 10-01 | 上游 SPDK clone，只读参考，不是自有仓 |

## 6. 推荐分组视图

### 6.1 日常开发最常改

- `ferrovirt`、`xvirt-client`、`hyper-agent`、`lictor`
- `qemu`（`stable-11.1-dev`）
- `petrel`、`cdp`
- `kestrel`、`fulmar`（新组件）
- `xvirt-autotest`

### 6.2 存储 / 备份 / CDP

- `petrel`、`fulmar`、`ocfs2`
- `aegis-sdk`、`snapshot-helper`、`imgtools`
- `xvirt-snapmig`、`xvirt-blockdiff`
- `cdp`（recorder + cdplog + tools/xbackup）、`cdp-agent`

### 6.3 内存分层与资源规划

- `qemu`（memtier）、`hyper-agent`（总控、资源规划、放置）、`lictor`（placement）、`ferrovirt`、`xvirt-client`
- 文档在 ferrostack `docs/`（memtier 系列、`resource-planning.md`）

### 6.4 监控

- `xvirt/monitoring`（现行 VictoriaMetrics 栈）
- `kestrel`（拟替代）

### 6.5 安装部署

- `build-deb`、`xvirt`（`build-deb-xvirt`）、`lycan`、`fledge`、`xvirt-autotest`

### 6.6 历史 / 参考

- `krait`：旧 Rocky/Anaconda 安装路线。
- `spdk-analysis`：上游 SPDK 只读参考。
- 已移除的 `virt-control`、`cdp-worker`、`xvirt-libs` 见 §0。

## 7. clone 示例（kairos）

```bash
mkdir -p /root/work && cd /root/work
git clone git@kairos.xve-vt.com:local/ferrostack.git
cd ferrostack
for r in aegis-sdk build-deb cdp cdp-agent concilium ferrovirt fledge fulmar \
         hyper-agent imgtools kestrel krait lictor lycan ocfs2 petrel qemu \
         snapshot-helper xdesk-client xvirt xvirt-autotest xvirt-blockdiff \
         xvirt-client xvirt-snapmig xvirt-tools
do
  git clone git@kairos.xve-vt.com:local/$r.git
done
git -C qemu checkout stable-11.1-dev
git -C qemu remote add upstream https://gitlab.com/qemu-project/qemu.git
# 可选：只读参考
git clone https://github.com/spdk/spdk.git spdk-analysis
```

子仓被 ferrostack 的 `.gitignore` 忽略，各自独立提交。`qemu` 默认分支为 `stable-11.1-dev`；`lycan` 默认分支为 `master`。

## 8. 远端切换记录（2026-11-09，已完成）

各仓已从旧内网 GitLab / GitHub 切到 kairos。kairos 上多数仓库的历史是重新导入的，与旧 origin 没有共同祖先。当时的做法是先 `git fetch kairos`，以 kairos 为准重置，再 cherry-pick 本地独有提交，没有强推。若还有旧工作区要切换，照此办理：

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

## 9. 需要后续核对或处理的点

- **qemu CBT bitmap export/import 是 11.1 重写版，待真 VM 验证**：10.0 原实现 `1f5bbc160f` 没进 kairos，开发机、195、GitHub 上都找不到；2026-11-20 按 `xvirt/CBT_BITMAP_PERSISTENCE_SPEC.md` 重写并合入 `stable-11.1-dev`（`b63be76397`），iotests 和调用方兼容测试都过了。出包前还要经 lictor 在真 VM 上验证（见 `xvirt-architecture.md` §11.1、§19）。
- **qemu `cdp-rename` 未合入**：产品线 QMP 仍是 `io-redirection-*`，cdp 仓 `tests/vm` 已改用新名，两边版本要配套。
- **kestrel 旧分支**：`wip/async-query`、`wip/class-retention`（内容已压成 `099b07a`、`912e7c6` 合进 main）已于 2026-11-20 从 kairos 和测试机 195 删除；开发机本地 kestrel 仓里的同名分支待清理（`git -C kestrel branch -D wip/async-query wip/class-retention && git -C kestrel fetch --prune kairos`）。
- **fulmar 从未 fetch**（只推过），本地即为最新提交；首次在测试机上编译前先确认 kairos 上没有别处的改动。
- kairos → GitHub 的自动同步是否启用、同步范围与方向。
- 哪些仓库私有、哪些公开，交接时要标注权限。
- `build-deb-xvirt`、`monitoring/`、`nginx/`、`lab-env/` 都是集成仓 `xvirt` 的目录，没有单独拆仓。
- `xvirt-libs` 是否确认已被 `cdp/common` 吸收。
