# 虚拟机模板方案（1.0）

> 状态：方案已拍板（2026-11-24），按第一期实施。
> 对应需求：REQ-VM-060 转换为模板、REQ-VM-061 从模板部署、REQ-VM-064 批量部署、REQ-VM-065 Linux 系统定制（部署时设置 IP 等），并与 REQ-VM-062 完整克隆共用复制链路。
> 现状见 xvirt-architecture.md §10.5：库里没有模板概念；管理中心“克隆为模板”目前如实报“暂未实现”；单机向导有“从模板部署”入口但没有流程。

## 1. 已拍板

1. **模板就是虚拟机，只多一个标记**。不另建表，模板仍是 `vm` 表里的一行，盘、网卡、配置都沿用。
2. **模板不能开机，但可以编辑**（改 CPU / 内存 / 网卡 / 加减盘等）。
3. **从模板部署有“快速部署”选项**：勾选时磁盘用 reflink 复制（共享数据块、写时复制，秒级）；不勾选则普通复制（整盘拷贝）。
4. **部署时可以设置虚拟机 IP 等配置**（客户机定制）。
5. **可以批量部署**。
6. **快速部署按盘生效**：VM 的盘可能分布在不同存储上，有的支持 reflink、有的不支持。勾选快速部署时，**只对支持的盘做 reflink 复制，其余盘普通复制**，不因个别盘不支持而拒绝整个部署。
7. 其余按建议（原 §9 待拍板项的结论，见 §9）：
   - 客户机定制 1.0 只做 xvirt-tools 通道，cloud-init 放第二期；
   - Petrel 盘 1.0 不做瘦克隆，按第 6 条视为“不支持快速”的盘，完整拷贝；瘦克隆与依赖管理放 1.1；
   - 带 SR-IOV 直通网卡的 VM 转模板时释放 VF（网卡保留对 SR-IOV 交换机的引用），部署时按交换机重新分配；
   - 批量部署单次最多 100 台，默认并发 4 台；
   - 模板允许带快照，部署只用当前盘状态；
   - REQ-VM-064 批量部署调入 1.0；REQ-VM-063 链接克隆放 1.1。

## 2. 数据模型

| 变更 | 说明 |
|---|---|
| `vm.is_template TINYINT NOT NULL DEFAULT 0` | 模板标记。启动时 `ALTER TABLE … ADD COLUMN IF NOT EXISTS` |
| `vm.template_uuid VARCHAR(64) NULL` | 部署来源模板（只用于展示 / 追溯，模板删了也不影响虚拟机） |
| `vm_template_deploy`（新表） | 一次批量部署的记录：`id, template_uuid, request_json, created_by, created_at, total, succeeded, failed`。用于界面查看批量结果、重试失败项 |

模板沿用虚拟机的全部字段（所属存储、盘、网卡、NVRAM、TPM 开关……）。列表接口 `GET /api/vcenter/vm` 增加 `is_template`，默认**不返回模板**，带 `?templates=only|include` 才返回。

## 3. 转换

### 3.1 虚拟机 → 模板

`POST /api/vcenter/vm/{vm}/template`（同步，或轻量任务 `convertToTemplate`）

前置条件：

- VM 已关机（`shutoff`）；
- 带 SR-IOV 直通网卡时：转换时释放 VF（`sriov_vf` 回到空闲），网卡改记为“待分配”（`type=sriov`，保留 SR-IOV 交换机名），部署时按交换机重新分配；转回虚拟机时同样在下次开机前重新分配；
- 快照：**允许带快照转换**，部署时只用当前盘状态，不带快照；

动作：置 `is_template = 1`；清空安装介质（`installation_iso_path`）；HA 纳管的 VM 从 concilium 移除资源（模板不参与 HA）。

### 3.2 模板 → 虚拟机

`DELETE /api/vcenter/vm/{vm}/template`：置 `is_template = 0`，之后可正常开机。若存在依赖它的 Petrel 瘦克隆（§4.3），照样允许（依赖的是快照，不是“模板”身份）。

### 3.3 模板上的限制（控制面统一拦）

| 操作 | 模板上 |
|---|---|
| 开机 / 开机（恢复）/ 重启 / 强杀重启 / 控制台 | **拒绝**：“模板不能开机，先转换为虚拟机” |
| 编辑设置（CPU / 内存 / 盘 / 网卡 / 选项） | 允许（关机态编辑，走现有 update_vm 离线路径） |
| 在线迁移 | 不适用（模板不运行）；存储迁移、冷迁移允许 |
| 创建快照 | 允许（便于模板自身的版本回退），但部署不使用快照 |
| 克隆 | 允许，结果仍是模板（“复制模板”），或在部署向导里选“部署为虚拟机” |
| 删除 | 允许；若有 Petrel 瘦克隆依赖它的基础快照则拒绝（§4.3） |

实现位置：`write::start_vm_task` / `power_cycle_task` / `console_ticket_create` 入口统一查 `is_template`；前端模板行隐藏这些按钮。

并发：部署时对模板取**共享锁**（`TaskObj::shared("vm", tpl)`），编辑 / 转换 / 删除取**独占锁**，避免部署中途模板盘被改。

## 4. 部署（单台与批量共用）

### 4.1 流程

`POST /api/vcenter/template/{tpl}/deploy`，生成父任务 `deployFromTemplate`，每台虚拟机一个子任务。单台部署就是 `count = 1`。

每台虚拟机的子任务复用现有克隆链路（`vm_create::clone_vm_task` → `create_vm_task_labeled`），差别：

1. 源是模板（关机），**不需要**运行中克隆那套“暂停取时间点副本”；
2. 磁盘复制方式由 `fast` 决定（§4.2）；
3. 新 VM `is_template = 0`、`template_uuid = <tpl>`；MAC 重新生成（已有）、NVRAM 拷贝（已有）、SR-IOV 网卡重新分 VF（已有）、vTPM 状态不继承（新 VM 全新 TPM）；
4. 建好后若带定制（§5）：开机 → 等 xvirt-tools 上线 → 下发定制 → 按需重启；
5. `power_on = true` 且无定制时，建好直接开机。

### 4.2 快速部署 vs 普通复制

| 盘所在存储 | 快速部署（`fast=true`） | 普通复制（`fast=false`） |
|---|---|---|
| fulmar（本地 / 集群） | fulmar 控制 API `reflink`，秒级；新盘独立（写时复制），模板可随后删除 | `qemu-img convert` 经 FUSE 整盘拷贝 |
| xfs 本地（`reflink=1`） | `cp --reflink=always`，秒级，新盘独立 | `qemu-img convert` 整盘拷贝 |
| NFS | **不支持**（NFS 没有 reflink） | `qemu-img convert` |
| Petrel | 瘦克隆 `petrel-ublk clone --mode thin`（基于模板基础快照，写时复制），见 §4.3 | `petrel-ublk clone --mode copy` 完整拷贝（沿用克隆 Petrel 盘的实现） |

快速部署**按盘**生效（已拍板）：勾选后，每块盘单独判断能否 reflink，能的 reflink，不能的普通复制，整个部署不会因个别盘不支持而失败。一块盘能 reflink 的条件：

- 盘所在存储类型支持 reflink：fulmar，或 `reflink=1` 的 xfs；NFS、Petrel（1.0）不支持；
- 新盘落在**与模板盘同一个数据存储**上（reflink 只能在同一文件系统 / 同一 fulmar 数据存储内做）。新 VM 的所属存储或该盘的目标存储换到别处时，这块盘只能普通复制；
- xfs 本地存储只在模板所在主机上，目标主机不是模板所在主机时这些盘只能普通复制；fulmar 集群存储集群内任一挂载了它的主机都行；
- xfs 没用 `reflink=1` 格式化时，agent 的 `cp --reflink=always` 会失败，此时这块盘**退回普通复制**并在任务消息里注明（不让整台失败）。

前端在选择目标主机 / 所属存储后，调 `GET /api/vcenter/template/{tpl}/deploy-options?host=&datastore=` 拿到**逐盘**的复制方式预估（`fast` / `full` 及原因），在确认页列出“哪些盘快速复制、哪些整盘拷贝”，让用户对耗时有预期。

当前 agent 的 `image.clone_img` 在 fulmar 同数据存储时**总是** reflink，需改为由参数 `copy_mode: reflink|full` 决定，保证“不勾快速部署 = 真复制”。

### 4.3 Petrel 盘的快速部署（瘦克隆，1.1）

> 已拍板：1.0 不做，Petrel 盘在快速部署时按“不支持”处理、完整拷贝。以下为 1.1 设计备忘。


reflink 副本互不依赖，Petrel 的瘦克隆则依赖父卷的快照：

- 模板首次被快速部署时，给模板的每块 Petrel 盘打基础快照 `tpl-base-<n>`（模板关机，离线打）；模板盘之后被编辑写入，下次快速部署前重新打一个新的 `tpl-base-<n+1>`（旧的仍被已部署的瘦克隆引用，不能删）；
- 记录依赖：`vm_disk` 增加 `petrel_parent`（`<vol>@<snap>`），基础快照的引用计数由此可查；
- 删除模板 / 删除基础快照前检查是否还有瘦克隆依赖；提供“解除依赖”（flatten，把瘦克隆补成完整卷）操作；
- 这是 REQ-VM-063 链接克隆在 Petrel 上的实现。


### 4.4 批量部署

请求里 `count = N`，名称、主机、IP 按规则展开：

- **命名**：`name_pattern`，支持 `{n}`（序号，可写 `{n:03}` 补零）和起始序号 `start_index`，如 `web-{n:02}` → `web-01 … web-10`；名称冲突在提交前整体检查，有冲突整批拒绝；
- **主机分布**：指定一台主机，或给一组主机按“轮询 / 按剩余内存”分布（复用资源规划的放置评分）；快速部署时只能落在满足 §4.2 约束的主机上；
- **所属存储**：统一一个，或跟随模板；
- **IP**：见 §5.3，按起始 IP 递增分配；
- **上限与并发**：单次最多 100 台；默认同时进行 4 台（可配置），普通复制大盘时避免把存储打满；
- **部分失败**：每台独立成败；父任务汇总“成功 n / 失败 m”，失败项可在批量记录里一键重试；已建好但定制失败的 VM 保留并标注，不自动删除。

## 5. 客户机定制（部署时设置 IP 等）

### 5.1 可设置项

| 项 | 说明 |
|---|---|
| 主机名 | 支持 `{n}` 模板，如 `web-{n:02}` |
| 网卡网络配置 | 每块网卡：DHCP 或静态（IP / 前缀 / 网关 / DNS）。网卡按 MAC 对应 |
| 管理员密码 | Linux root / 指定用户；Windows Administrator |
| SSH 公钥 | 追加到指定用户 |
| 首次启动脚本 | 可选，部署后在客户机内执行一次 |
| 时区 | 可选 |

此外，Linux 克隆**必须**做的“去重”动作（不让用户选）：重新生成 `/etc/machine-id`（否则多台克隆机 DHCP 的 DUID 相同、拿到同一个 IP）、重新生成 SSH 主机密钥。

### 5.2 实现通道

**推荐：经 xvirt-tools（自研客户机工具）。**

xvirt-tools 已有 `net.apply`（按网卡设置 DHCP / 静态 IP，Linux 与 Windows 都有实现）、`os.set_password`、`ssh.add_authorized_key`、`os.reboot`，以及 `ip.list`（可按 MAC 找到客户机内网卡名）。补一个组合 RPC：

```
os.customize {
  hostname, timezone?,
  regenerate_machine_id: true, regenerate_ssh_host_keys: true,   // Linux
  networks: [{mac, method: dhcp|static, address, prefix, gateway, dns:[]}],
  users: [{name, password?, ssh_keys:[]}],
  script?: {interpreter, content_b64}
}
→ {applied:[...], reboot_required: bool}
```

流程：部署完成 → 开机 → 等 lictor 报告 xvirt-tools 上线（`vm.guest.state`，带超时，默认 5 分钟）→ 控制面下发 `os.customize` → 需要时 `os.reboot` → 再等 tools 上线并回读 `ip.list` 核对 IP → 子任务成功。

优点：自研、Linux / Windows 一套、可回读确认；缺点：**模板里必须装好 xvirt-tools**，且首次开机时客户机先用模板里的网络配置跑一小段时间。**模板应配置为 DHCP**（或无网络），避免多台同时带模板的静态 IP 上线冲突——转换为模板时在界面提示。

**备选：cloud-init NoCloud。** 为每台 VM 生成一张卷标 `cidata` 的小 ISO（`meta-data` / `user-data` / `network-config`），作为一次性安装介质挂上（复用 `installation_iso_path` 与开机临时光驱链路，关机自动摘除）。优点是网络配置在网络起来之前生效、行业通行；缺点是只适用于装了 cloud-init 的 Linux 模板，Windows 需另走 sysprep / unattend（REQ-VM-066，P2）。

已拍板：1.0 只做 xvirt-tools 通道；cloud-init 放第二期（届时模板上加属性 `customization: xvirt-tools | cloud-init | none`）。

### 5.3 IP 分配与冲突

- 批量部署填“起始 IP + 前缀 + 网关 + DNS”，按台递增；也可上传 / 粘贴一份逐台清单（名称、主机名、IP）；
- 提交前检查：IP 都在同一网段内、不与网关重合、不超出网段；与库里已知 VM 的 IP（`vm.ip`，来自 xvirt-tools 上报）冲突时警告；
- 1.0 不做 IP 池（REQ-NET 的 IP 地址池可后续接入）。

### 5.4 失败处理

- 等不到 xvirt-tools（模板未装 / 客户机没起来）：子任务失败，VM 保留开机状态，消息写明“未完成定制”；
- `os.customize` 部分项失败：子任务失败并列出失败项；
- 定制成功后回读 IP 不符：子任务告警（不失败），给出实际 IP。

## 6. 接口

| 方法与路径 | 说明 |
|---|---|
| `POST /api/vcenter/vm/{vm}/template` | 转换为模板 |
| `DELETE /api/vcenter/vm/{vm}/template` | 转换回虚拟机 |
| `GET /api/vcenter/template` | 模板列表（= `vm?templates=only`，附模板专属字段：customization、基础快照、已部署数） |
| `GET /api/vcenter/template/{tpl}/deploy-options?host=&datastore=` | 部署选项校验：可选主机 / 存储、`fast_supported` 及原因 |
| `POST /api/vcenter/template/{tpl}/deploy` | 部署，返回父任务 |
| `GET /api/vcenter/template/deploys/{id}` | 批量部署记录与逐台结果；`POST …/{id}/retry` 重试失败项 |

部署请求体：

```json
{
  "count": 10,
  "name_pattern": "web-{n:02}", "start_index": 1,
  "hosts": ["<host uuid>"], "placement": "round_robin",
  "datastore": "<所属存储 uuid，缺省跟随模板>",
  "fast": true,
  "power_on": true,
  "customize": {
    "hostname_pattern": "web-{n:02}",
    "networks": [
      {"nic": "net0", "method": "static", "ip_start": "10.10.0.21", "prefix": 24,
       "gateway": "10.10.0.1", "dns": ["10.10.0.2"]}
    ],
    "users": [{"name": "root", "password": "***", "ssh_keys": ["ssh-ed25519 …"]}],
    "script": null
  },
  "overrides": {"vcpu": 4, "memory": 8192}
}
```

`overrides` 可选：部署时覆盖 CPU / 内存（模板本身不变）。

## 7. 界面

单机界面：

- VM 列表右键 / 详情页：“转换为模板”（仅关机时可用）、模板上“转换为虚拟机”“从此模板部署”；
- VM 列表加“模板”筛选（或单独“模板”页签），模板行带“模板”标记，隐藏开机 / 控制台；
- 新建虚拟机向导里已有的“从模板部署”入口接通，步骤：选择模板 → 名称与数量 → 位置（主机、所属存储、快速部署开关及不可用原因）→ 硬件覆盖（可选）→ 客户机定制（主机名、每块网卡 DHCP / 静态与起始 IP、密码、SSH 公钥、脚本）→ 确认（展开后的逐台清单：名称、主机、IP）；
- 批量部署进度：父任务展开显示逐台状态，失败项可重试。

管理中心：资源树里模板用不同图标；右键“克隆为模板”改为“转换为模板 / 克隆为模板”；“从模板部署”复用同一部署接口（目前管理中心的 OVF 部署向导是写死的假数据，替换掉）。

## 8. 各仓改动与估算

| 仓 | 改动 | 估算 |
|---|---|---|
| ferrovirt | `is_template` / `template_uuid` 列与部署记录表；转换接口；开机 / 控制台入口拦截；部署编排（父子任务、命名 / 主机 / IP 展开、并发、重试）；`deploy-options` 校验；客户机定制编排（等 tools、下发、回读）；Petrel 基础快照与依赖（若 1.0 做） | 大（主要工作量） |
| hyper-agent | `clone_img` 增加 `copy_mode: reflink|full`（xfs `cp --reflink=always`，fulmar reflink，普通 `qemu-img convert`）；cloud-init seed ISO 生成（若做） | 中 |
| lictor | Petrel 瘦克隆 / flatten 动作（若 1.0 做）；`vm.guest.state` 上线事件已有，等待逻辑在控制面 | 小–中 |
| xvirt-tools | 新 RPC `os.customize`（主机名、machine-id / SSH 主机密钥重新生成、多网卡网络、用户密码与公钥、脚本），Linux 与 Windows | 中 |
| xvirt-client | 模板筛选 / 标记、转换入口、部署向导（含批量与定制）、批量进度 | 中–大 |

建议分两期：

- **第一期**：数据模型、转换与限制、单台 / 批量部署（快速 / 普通复制，文件盘与 fulmar）、xvirt-tools 定制（主机名、网络、密码、SSH 公钥、machine-id）、前端部署向导；
- **第二期**：Petrel 瘦克隆与依赖管理（若不进第一期）、cloud-init 通道、首次启动脚本、按清单批量导入、IP 池对接。

## 9. 拍板结论（2026-11-24）

| # | 问题 | 结论 |
|---|---|---|
| 1 | 客户机定制通道 | 1.0 只做 xvirt-tools；cloud-init 第二期 |
| 2 | Petrel 盘快速部署 | 1.0 不做瘦克隆，Petrel 盘按“不支持快速”的盘完整拷贝；瘦克隆放 1.1 |
| 3 | 快速部署条件不满足 | **按盘处理**：支持 reflink 的盘快速复制，其余普通复制，不拒绝整个部署 |
| 4 | 带 SR-IOV 网卡转模板 | 转换时释放 VF，部署时按交换机重新分配 |
| 5 | 批量上限与并发 | 单次最多 100 台，默认并发 4 台 |
| 6 | 模板带快照 | 允许，部署只用当前盘状态 |
| 7 | 需求范围 | REQ-VM-064 批量部署调入 1.0；REQ-VM-063 链接克隆 1.1 |
