# Tauri UPnP/UDP Tunnel 接入完成报告 & 后续分支架构评估

## 一、已交付分支与 Verified Commit

- **分支名**：`z/agy/rc/tauri/upnp-tunnel`（基于 `origin/z/agy/rc/tauri-tunnel-settings`）
- **Commit**：[`db751d8c379c47fe4788b80a08ad13db3958e19c`](https://github.com/Antigravi-ty/RocketCompanion/commit/db751d8c379c47fe4788b80a08ad13db3958e19c)（`verified: true`, Bot GPG Signed）

### 1.1 核心改动清单（低耦合、按职责拆分）

1. **Rust 后端多 Peer 路由与零 JSON 二进制 IPC（`src-tauri/src/tunnel_manager.rs`, `src-tauri/src/lib.rs`）**
   - **单 Socket 多 Peer 路由**：`ActiveTunnelSession` 新增 `peers: HashMap<String, SocketAddr>` 与 `addr_to_peer: HashMap<SocketAddr, String>`，Host 端全房间仅绑定 **1 个 UDP Socket + 1 次 UPnP IGD 映射**，即可同时接收多 Player 输入并向无上限数量的 Spectator/Broadcast 端发送数据。
   - **打洞探针过滤**：`rx_thread` 自动拦截 4 字节 `"RCPH"` 探针并回复 `"RCPA"`，同时触发 `rc-tunnel-peer-ready` 事件，不再将打洞包透传给 JS 游戏层。
   - **消除 JSON `number[]` 序列化开销**：新增 `tauri_tunnel_attach_rx_channel`（使用 Tauri v2 `Channel<InvokeResponseBody::Raw>` 直接推送紧凑二进制帧）与 `tauri_tunnel_send_raw`（通过 `tauri::ipc::Request` 直接接收原始 `Uint8Array`），同时保留旧事件/命令作为回退。
   - **单 Peer 清理**：新增 `tauri_tunnel_disconnect_peer`，支持 Kick/Leave 时精准移除单个 Peer 而不关闭 Host 的 UDP Socket。

2. **TS 隧道可靠性与协调层（`src/network/transport/tunnel/`）**
   - `TunnelReliabilityLayer.ts`：纯协议层，为裸 UDP 提供 **MTU 分片重组（1160B 安全分片）** 与 **选择性重传/ACK/去重**。
     - 不可靠路径（`PHYSICS_SNAPSHOT`、`USER_INPUT`、`PING_PONG_16B`）：<= MTU 仅 2 字节头部，零 ACK、零队头阻塞。
     - 可靠路径（`RELIABLE_CONTROL`、`TELESTRATOR_SYNC`、`ROOM_ACTIVITY_SWITCH`、`CSR_STATE_BROADCAST`、`STATE_PAUSED_ACK`、`color_change`）：序列号 + 65ms RTO 自动重传 + 接收端去重。
   - `HostTunnelCoordinator.ts`：Host 端共享协调器，确保 `HostSessionController` 下所有 Peer 共享同一个 `RustTunnelTransport`，按 `peerId` / `fromAddr` 路由。
   - `TunnelPeerBridge.ts`：单 Peer 桥接器，将 `TunnelReliabilityLayer` 连接到 `HostTunnelCoordinator`（Host 侧）或独立 `RustTunnelTransport`（Client 侧）。
   - `RustTunnelTransport.ts` & `TunnelTransport.ts`：支持二进制 `Channel` 接收、`send_raw` 发送、`connectPeer`（同时尝试 `externalIp:externalPort` 与 `localIp:localPort` 双路打洞）。

3. **主链路无缝接入（`P2PWebRTCChannel.ts`, `DataChannelTransport.ts`, `HandshakeSession.ts`, `SignalTokenCodec.ts`, `webrtcHandshakePipeline.ts`, `HostSessionController.ts`）**
   - **Token 自动交换 UPnP Endpoint**：`createOfferToken` / `acceptOfferAndCreateAnswer` / `acceptAnswerToken` 在 Tauri 环境下自动打开本地 UDP/UPnP 端点并将 `tunnelEndpoint` 嵌入加密 Offer/Answer Token（与 WebRTC SDP/ICE 并行交换）。
   - **透明切换与自动回退**：顶层业务（`EstablishedSessionManager`、`RoomReplicationBus`、`PredictionReconciler` 等）继续面向 `P2PWebRTCChannel` 接口；当 Tauri `TunnelPeerBridge` 打洞成功（`onReady`）时自动切换至 `rust-upnp-udp` 快路径；在 Web 端或 UDP 受限时透明保持 `webrtc` DataChannel 或 `crosstab`。

### 1.2 验证结果

- `pnpm check`（Radar 架构检查 + `tsc --noEmit`）：**PASS**（1395/1395 TS，0 循环依赖，0 越层导入）
- `cargo check`（`src-tauri`）：**PASS**（0 errors）
- `TunnelReliabilityLayer` 单元验证（小包直发、2500B 超 MTU 分片重组、可靠帧丢包重传 + ACK）：**PASS**

---

## 二、架构问题专项评估

### 2.1 为什么建议“混合传输模式”而不是全部走 Rust Tunnel？
- 即将合入的语音模块（`z/agy/rc/track-b-voice-fix`）基于 WebRTC `RTCPeerConnection` 的 `MediaStreamTrack`，直接复用浏览器原生 Opus 编解码、回声消除（AEC）、降噪（NS）与抖动缓冲。
- 如果将音频也改走 Rust UDP，需要在 Rust/TS 自行实现音频采集、Opus 编码、Jitter Buffer 与 AEC，工作量大且收益低；同时保留 `RTCPeerConnection` 还能在极少数对称 NAT + 无 UPnP 环境下作为 DataChannel 的自动兜底。
- 因此，**“游戏状态/控制走 Rust UPnP/UDP 快路径 + 语音/兜底走 WebRTC”** 是当前阶段耦合最低、稳定性最高的方案。

### 2.2 关于“绕过 IPC / 虚拟网卡 / Socket 直连”的跨平台限制与性能最优解
- **虚拟网卡（TUN/TAP，如 Windows `wintun`、macOS `utun`）**：需要管理员权限（UAC）或 macOS System Extension 公证，对桌面伴侣应用而言安装与权限门槛过高，且 JS 沙盒本身仍无法直接操作原始网卡。
- **WebView 沙盒限制**：Windows WebView2 与 Apple Silicon macOS WKWebView 均禁止 JS 直接打开原生 UDP Socket。
- **已落地的优化**：本次已将旧版 `Vec<u8> -> JSON number[]` 替换为 **Tauri v2 二进制 `Channel` + `tauri_tunnel_send_raw`（原始 `Uint8Array`）**，消除了 JSON 序列化开销。
- **进一步零 IPC 演进方案（如 120Hz 多端压测需要）**：可在 Rust 侧启动 `ws://127.0.0.1:<port>` 本地回环二进制 WebSocket 服务，由 WebView 原生 C++ 网络栈直连（无需任何系统权限，Windows 与 Apple Silicon macOS 通用），将吞吐完全卸载出 Tauri IPC。

### 2.3 Host 多路拓扑评估（5 接收 + 无上限发送）
- 改造后的 `HostTunnelCoordinator` + `tunnel_manager.rs` 采用 **单 UDP Socket + `HashMap<peerId, SocketAddr>` 路由**，不再受限于单 Peer 绑定：
  - 5 个 Player 的高频输入与 N 个 Spectator 的只读广播共用同一端口与同一条 UPnP 映射；
  - 发送链路在 Rust 侧无硬上限，仅受宿主机上行带宽约束（建议后续在 `HostTunnelCoordinator` 对 Spectator 广播按需降频或复用同一份序列化快照缓冲以减少内存拷贝）。

---

## 三、后续两个分支拆分规划

### 3.1 分支 2：`z/agy/rc/tauri/fs-direct`（Tauri 文件系统直连与敏感数据隔离）
- **存储抽象**：在 `src/utils/` 或 `src/loader/` 定义统一的 `LocalStoragePort`（`readFile` / `writeFile` / `listFiles` / `deleteFile`），Web 端保持 IndexedDB 实现，Tauri 端通过 Rust `fs_vault` 命令直连本地目录。
- **目录与敏感数据物理隔离**：
  - 用户可分享目录（例如 `<AppData>/RocketCompanion/workspace/`）：存放 `replays/`（`.RCReplay`、`.RLRCReplay`）、`training_packs/`（`.RCDPack`、`.RCSPack`）、`configs/`（镜头、快捷键、画质与常规设置）。用户打包该目录分享给他人时**不会泄露任何敏感凭据**。
  - 敏感凭据隔离目录（例如 `<AppData>/RocketCompanion/vault/` 或系统 Keychain）：单独存放登录 Token、私有 Signaling Relay URL、房间密码缓存等，与 `workspace/` 处于同级隔离目录，永不随配置包导出。

### 3.2 分支 3：`z/agy/rc/tauri/ws-stats-overlay`（WebSocket Stats-API 广播/接收 + 无边框 Overlay）
- **WebSocket 广播与接收**：
  - 接收端（Client Listener）：可直接在 TS 层通过 `WebSocket` 连接本地/局域网 Rocket League 数据源，或由 Rust 侧提供本地 WS 桥接（避免 HTTPS/Mixed-Content 限制），将标准化后的球/车/Boost 状态注入引擎只读同步状态。
  - 广播端（Simulated RL Client）：将当前仿真状态按约定格式通过本地 WebSocket Server（Rust `tokio-tungstenite` 暴露 `ws://127.0.0.1:<port>`）广播出去，配合 **HUD-Only 模式**（关闭 3D 场地渲染，仅保留 HUD/Telestrator 叠层）。
- **全屏无边框透明 Overlay**：在 `src-tauri/src/window_manager.rs` 新增 Overlay 窗口模式（`transparent: true`, `decorations: false`, `always_on_top: true`, `ignore_cursor_events` 穿透切换，主要面向 Windows 桌面端；macOS 保持常规窗口与冻结/监听模拟模式）。
