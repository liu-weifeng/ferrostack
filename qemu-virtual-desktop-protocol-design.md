# QEMU 虚拟桌面协议设计方案（v2，贴合 xvirt 栈）

> v2 修订要点：删除 libvirt / SPICE guest agent 假设，组件映射到 xvirt 现有栈
> （ferrovirt / hyper-agent / lictor / xvirt-tools）；采集机制钉死在
> `-display dbus` + DMA-BUF；Windows 第一版显示设备改走 QXL/SPICE；客户端阶段
> 反转为浏览器优先；补齐延迟预算、编码容量、文本清晰度、输入传输层、TURN 成本
> 等 v1 悬空决策。

## 1. 设计目标

构建一种面向 QEMU/KVM 虚拟机的远程桌面协议，满足：

- 使用服务器 GPU 进行桌面视频编码加速；
- 支持 Windows、Linux、BSD 等多种客户机系统；
- 支持键盘、鼠标、音频、多显示器和分辨率热切换；
- 支持客户端 USB 重定向；
- 支持内网、公网和 NAT 环境；
- 支持安全认证、权限控制、审计和断线重连；
- 为后续 GPU passthrough、vGPU、AV1 和高性能 3D 留出扩展空间。

**非目标**（第一版明确不做）：macOS 客户机、浏览器内 USB 重定向（WebUSB 权限
模型不足）、跨主机会话迁移。

## 2. 总体架构

```text
客户端（第一版=浏览器；原生客户端=第二版，定位“USB 增强版”）
 ├─ 视频解码：H.264 / H.265 / AV1（浏览器原生 / 原生客户端硬解）
 ├─ 输入采集：键盘、鼠标、触摸 → WebRTC DataChannel
 ├─ 音频播放与采集（Opus track）
 └─ WebRTC 传输层（SRTP/DTLS + ICE/STUN/TURN）
              │
服务器（每台宿主机）
 ├─ ferrovirt        ← 鉴权(JWT/PAM)、会话票据、WebRTC signaling（复用既有 WS）
 ├─ desktop-worker   ← 新组件：每 VM 一个，hyper-agent 托管（同 petrel 守护进程模式）
 │    ├─ 画面获取：QEMU -display dbus（org.qemu.Display1，per-scanout DMA-BUF）
 │    ├─ GPU 编码：VAAPI / NVENC / AMF（零拷贝 dmabuf 入编码器）
 │    ├─ 音频：QEMU dbus 音频导出 → Opus
 │    └─ WebRTC endpoint（GStreamer webrtcbin 或 webrtc-rs）
 ├─ lictor           ← 已有：QMP 持有者；输入注入走 QMP input-send-event
 ├─ hyper-agent      ← 已有：desktop-worker 生命周期 reconcile；USB 代理托管
 ├─ usb-redirector   ← 新组件：usbredir 代理 + 授权 + 审计（仅对原生客户端）
 └─ QEMU/KVM（自维护 fork，已启用 spice/virgl/dbus 模块）
       ├─ virtio-gpu（Linux guest）/ QXL（Windows guest v1）
       ├─ virtio-input / virtio-sound / virtio-net / virtio-blk
       └─ QMP（lictor）/ xvirt-tools（guest 内 agent，virtio-serial NDJSON）
```

核心原则：

> QEMU/KVM 负责虚拟机，QMP/virtio/xvirt-tools 提供底层设备能力；宿主机新增
> desktop-worker 做“采集 + GPU 编码 + WebRTC”，控制面全部复用 ferrovirt。
> **不引入 libvirt，不引入 SPICE guest agent**——本项目栈已有等价物。

## 3. 显示和 GPU 方案

### 3.1 按客户机 OS 的显示设备矩阵（v1 明确承诺）

| 客户机 | 显示设备 | 采集路径 | 分辨率热切换 | 备注 |
|---|---|---|---|---|
| Linux | virtio-gpu | `-display dbus` DMA-BUF | virtio-gpu EDID（QMP） | 首选路径，零拷贝 |
| Windows | **QXL**（v1）| `-display dbus`（QXL scanout） | QXL 驱动 + xvirt-tools RPC | virtio-gpu Windows 驱动（viogpudo）多显/热切不成熟，**不作为 v1 依赖**；成熟后切换 |
| BSD | std VGA | dbus scanout（共享内存帧） | 固定模式表 | 兼容模式，非零拷贝可接受 |

### 3.2 基础方案：服务端捕获 + GPU 编码（v1）

```text
Guest GUI → virtio-gpu / QXL scanout
  → QEMU -display dbus（DMA-BUF / 共享内存帧 + damage 区域 + 独立光标事件）
  → desktop-worker（dmabuf 直入 VAAPI/NVENC，不落 CPU 内存）
  → H.264 → WebRTC → 客户端
```

多台 VM 共享宿主机 GPU 编码资源，不要求每 VM 独占物理 GPU。damage 信息用于
帧调度（静态桌面不重编码），光标作为独立层（DataChannel 传位图+坐标，客户端
本地合成），避免鼠标移动触发整帧重编。

### 3.3 中等性能方案：VirGL/Venus（阶段三）

Linux 桌面轻量 3D。我们的 QEMU deb 已编入 virglrenderer；Windows 不承诺。

### 3.4 高性能方案：GPU passthrough / vGPU（阶段三）

**架构上必须明确**：直通后宿主机看不到 framebuffer，服务端采集路径失效，
只能走**客户机内采集 + 客户机内编码**（Parsec/Sunshine 模式，编码器用直通
GPU 自己的），desktop-worker 退化为纯转发。这是与 §3.2 并列的第二条数据
路径，协议层（信令/传输/输入）两者共用，采集/编码层完全不同——实施时按
两个 Capture/Encode Provider 对待，不要试图统一。

## 4. 协议分层

### 4.1 控制通道（信令）

复用 ferrovirt：HTTPS + WebSocket（既有 STOMP 基础设施），消息 Protobuf/JSON。
会话建立流程对齐 aegis access-ticket 模式：

1. 客户端持 JWT 调 `POST /api/vcenter/vm/{vm}/desktop-session` → ferrovirt 校验
   权限，向该 VM 宿主机的 desktop-worker 申请 offer，返回 `{session_ticket, sdp}`；
2. ICE/SDP 经 ferrovirt WS 转发（ferrovirt 即 signaling server）；
3. 会话票据短期有效、绑定 vm uuid + 用户 + 通道类型（显示/USB/剪贴板独立授权）。

```protobuf
message SessionHello {
  string ticket = 1;
  repeated string codecs = 2;      // ["h264", "h265", "av1"]
  repeated string features = 3;    // ["cursor-layer", "multi-display", "clipboard"]
  uint32 max_width = 4;
  uint32 max_height = 5;
}
message SetDisplayMode {
  uint32 display_id = 1;
  uint32 width = 2;
  uint32 height = 3;
  uint32 refresh_rate = 4;
}
```

### 4.2 视频通道

WebRTC MediaStream，**每个显示器一条 video track**（多显 = 多 track，客户端
每 track 一个窗口/画布；不做服务端拼接大画布——坐标映射简单且带宽按需）。

| 场景 | 编码 |
|---|---|
| 第一版 | H.264（浏览器兼容性、硬编普及度最高） |
| 高质量低带宽 | H.265（原生客户端优先启用） |
| 新硬件 | AV1（阶段三；注意仅新代 NVENC/VCN 支持硬编） |

**文本清晰度**（v1 必须正视的产品差距）：4:2:0 视频编码对文字有色度模糊，
与 RDP 混合编码有可感差距。对策分层：
- v1：静态 damage 区域降帧率 + 提高 QP 质量（"静态桌面优化模式"具体化）；
- v2：文本密集区域检测 → 无损/近无损补丁（DataChannel 传 PNG/WebP 区域，
  客户端叠加），形成"视频层 + 无损补丁层"混合渲染；
- 4:4:4 硬编（HEVC RExt）仅在原生客户端 + 新硬件上作为可选项，不作依赖。

### 4.3 输入通道

**传输：WebRTC DataChannel**（与音视频同一条 ICE 路径，NAT 行为一致；
有序可靠模式）。服务端注入：**lictor QMP `input-send-event`**（发 scancode，
键盘布局问题天然消解；lictor 已持有 QMP，零新增客户机依赖）。

必须处理：修饰键状态机、焦点丢失/断线时的全键释放（worker 维护按下集合，
连接断开即注入全部 release）、多显示器绝对坐标按 track 映射。

### 4.4 剪贴板与文件通道（v1 缺失，v2 补定义）

独立 DataChannel + 独立授权位（默认关）。文本剪贴板经 xvirt-tools RPC 出入
guest；文件传输 v2 起步，走分块 + 审计日志。

## 5. USB 重定向（仅原生客户端，阶段二）

复用 usbredir 协议（我们的 QEMU 已编入 usbredir 支持），不自研 USB 协议。
浏览器客户端**不承诺** USB——这是原生客户端存在的核心理由。

```text
原生客户端 USB Capture → usbredir over DataChannel/QUIC
  → 宿主机 usb-redirector（hyper-agent 托管，沙箱隔离）
  → QEMU usbredir chardev → Guest USB Controller
```

授权策略：按用户/VM/VID:PID/序列号白名单、设备黑名单、存储只读、速率限制、
全程审计。**延迟预期要写进产品文档**：HID 类正常；大容量存储可用但慢
（协议往返放大）；等时类设备（摄像头/声卡）不走 USB 重定向——摄像头在
阶段三作为独立视频 track 反向注入（虚拟摄像头设备），体验远好于 usbredir。

## 6. 桌面采集管线

Provider 收敛为三个（v1 砍掉清单式罗列）：

```text
CaptureProvider
 ├─ DbusDisplayCapture   ← v1 唯一路径：QEMU -display dbus，DMA-BUF per scanout
 ├─ SharedMemCapture     ← 兼容回退（BSD/std VGA，dbus 共享内存帧）
 └─ InGuestCapture       ← 阶段三 GPU 直通路径（guest 内采集+编码，worker 纯转发）
```

零拷贝目标路径：`GPU scanout → DMA-BUF → VAAPI/NVENC → encoded packet`，
全程不经 CPU 内存拷贝。damage/光标由 dbus 接口原生提供。

## 7. 音频

QEMU dbus 音频导出（与 `-display dbus` 同总线）→ desktop-worker → Opus →
WebRTC audio track。麦克风反向：客户端 Opus track → worker 解码 → QEMU
audiodev 注入 virtio-sound。回声消除用 WebRTC 内建 AEC（客户端侧）。

## 8. 客户机支持策略

| 客户机 | v1 | 后续 |
|---|---|---|
| Linux | virtio-gpu + virtio-input + virtio-sound + xvirt-tools（已有 Go agent，加 display RPC） | VirGL/Venus |
| Windows | QXL 显示 + virtio 磁盘/网卡 + xvirt-tools Windows 版（已有）| virtio-gpu 驱动成熟后切换 |
| BSD | std VGA 兼容模式 | virtio-gpu |
| macOS | 仅作为**客户端**支持（浏览器/原生），不承诺作为客户机 | — |

分辨率热切换统一走 xvirt-tools RPC（guest 内改模式）+ QMP EDID 更新，
不依赖 SPICE vdagent。

## 9. 服务端组件与现有栈映射

| 方案组件 | 落地 | 状态 |
|---|---|---|
| vm-control-service | **ferrovirt**（生命周期/调度/REST/WS 已有） | 已存在 |
| session-gateway 鉴权+signaling | **ferrovirt 扩展**（desktop-session API + WS 信令转发） | 扩展 |
| session-gateway NAT 穿透 | 自建 STUN/TURN（coturn），独立部署 | 新增 |
| desktop-worker | 新组件（Rust/C+GStreamer），**hyper-agent 托管**（reconcile 模式同 petrel 守护进程），每 VM 一个 | 新增 |
| usb-redirector | 新组件，hyper-agent 托管，沙箱隔离 | 新增（阶段二） |
| guest-agent-service | **xvirt-tools**（已有，加 display/clipboard RPC） | 扩展 |

管理面与数据面分离不变：客户端只见 WebRTC endpoint 与 ferrovirt API，
永不直连 QMP / lictor / dbus 总线。

## 10. 网络和安全

- ICE/STUN/TURN；**TURN 容量要进成本模型**：严格 NAT 下中继流量双倍，
  按“并发会话 × 平均码率 × TURN 命中率”规划带宽与部署位置（内网部署
  TURN 可覆盖绝大多数企业场景，公网 TURN 按需扩）；
- TLS + DTLS-SRTP；短期会话票据（绑定 vm/用户/通道）；设备证书 mTLS 可选；
- USB、剪贴板、文件传输三者独立授权位，默认关闭；
- 会话超时、断线重连（票据宽限期内直接 ICE restart，不重走登录）；
- 水印：客户端叠加为主（不破坏零拷贝编码路径）；高安全场景服务端烧录
  水印作为可选模式（接受该路径退化为经 CPU/GPU 合成）；
- QEMU seccomp + AppArmor；desktop-worker 独立 uid + 最小权限。
- UDP 不可用时 TURN/TCP 回退，接受延迟退化。

## 11. 延迟预算（v1 目标：局域网端到端 < 60ms，广域 < 120ms）

| 环节 | 预算 | 手段 |
|---|---|---|
| 采集（scanout→dmabuf） | ~0（事件驱动） | dbus damage 通知 |
| 编码 | ≤ 8ms | 硬编低延迟档（zerolatency，无 B 帧） |
| 网络 | RTT/2 + 抖动缓冲 | WebRTC 自适应，目标缓冲 ≤ 2 帧 |
| 解码+呈现 | ≤ 16ms（一帧） | 浏览器硬解 / 原生硬解 |
| 输入上行 | RTT/2 + QMP ≤ 2ms | DataChannel + input-send-event |

## 12. 编码容量规划（商业密度指标，v1 必做实测）

- NVENC：消费卡有并发会话数限制，数据中心卡（L4/A 系列）无限制——选型时
  按“每 GPU 并发 1080p30 会话数”实测建表（经验量级：L4 ≈ 20-30 路 H.264
  1080p30 低延迟档，需实测修正）；
- VAAPI（Intel iGPU/Arc）：办公桌面密度方案，同样实测建表；
- desktop-worker 内建编码会话调度：静态桌面自动降帧释放编码器时间片，
  超容量时新会话降级软编或拒绝并提示；
- 容量数据进 ferrovirt 调度：VM 开桌面会话时按宿主机编码余量选择降级策略。

## 13. 分阶段实施路线（v2 调整）

### 阶段一：可用原型（最小闭环）

- QEMU `-display dbus` 采集（Linux guest：virtio-gpu；Windows guest：QXL）；
- desktop-worker：dmabuf → VAAPI/NVENC H.264 → WebRTC（GStreamer webrtcbin）；
- **浏览器客户端**（xvirt-client 控制台窗口内嵌 WebRTC view，替换 noVNC）；
- 输入：DataChannel → lictor QMP input-send-event（含断线全键释放）；
- 音频下行（dbus 音频 → Opus）；
- ferrovirt desktop-session API + WS 信令；内网直连（无 TURN）。

### 阶段二：产品化

- 多显示器（多 track）与分辨率热切换（xvirt-tools RPC）；
- 自动码率 + 静态桌面优化 + 光标独立层；
- 文本无损补丁层（混合渲染）；
- **原生客户端**（Rust + 硬解）：USB 重定向（usbredir + 授权策略 + 审计）；
- 剪贴板（xvirt-tools）/ 文件传输（独立授权）；
- 麦克风上行、会话恢复（ICE restart）、TURN 部署；
- 监控指标（帧率/码率/RTT/丢包/编码会话占用）进 ferrovirt 告警体系。

### 阶段三：高性能场景

- GPU passthrough / vGPU：InGuestCapture 第二数据路径（guest 内编码）；
- VirGL/Venus（Linux 3D）；
- H.265 普及 + AV1（新硬件）；4:4:4 可选档；
- 摄像头反向注入（虚拟摄像头设备）、手柄、音视频外设；
- 高帧率模式（120Hz）与游戏向帧调度。

## 14. 关键风险（v2 排序）

1. **QEMU dbus display 的 QXL/多显/damage 行为**——v1 采集路径的地基，先做
   PoC 验证（现有集群一台 VM：dbus 采集 → VAAPI → 浏览器显示端到端）；
2. Windows QXL 驱动在新 Windows 版本上的维护状态（SPICE 社区活跃度下降），
   以及后续切换 virtio-gpu 的时间点；
3. 文本清晰度与 RDP 的产品差距（混合渲染层的工程量）；
4. 每 GPU 编码密度实测低于预期 → 商业密度模型不成立的风险；
5. USB 授权/异常断开/恢复的健壮性；
6. 弱网（TURN/TCP 回退）下的交互体验下限；
7. guest agent 不可用时的降级路径（纯 QMP：能看能操作，无热切换/剪贴板）。

## 15. 结论

第一版最小闭环调整为：

> Linux（virtio-gpu）/ Windows（QXL）客户机 + `-display dbus` DMA-BUF 采集
> + VAAPI/NVENC H.264 + WebRTC + **浏览器客户端**（键鼠 + 音频下行）
> + ferrovirt 信令与票据。USB 重定向随原生客户端在阶段二交付。

自研工作集中在：desktop-worker（采集/编码/WebRTC）、ferrovirt 会话层、
xvirt-tools display RPC、编码容量调度、混合渲染。QEMU fork 的自维护能力
（已验证于 memtier/CBT 等）是把 dbus 采集路径按需补洞的底气；libvirt 与
SPICE agent 不进入依赖面。
