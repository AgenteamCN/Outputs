# Tauri P2P Tunnel (`db751d8`) 深度审查与验证报告

> 审查目标：`origin/z/agy/rc/tauri/upnp-tunnel` @ `db751d8c379c47fe4788b80a08ad13db3958e19c`
> 审查结论：**架构方向正确、主链路已打通、理论上完全可行，但存在 2 个需修复的逻辑 Bug（1 高危 1 中危），建议修复后再提交 PR。**

---

## 一、核心问题直接回复

### 1. `db751d8` 是否使用了 Web Worker + WebTransport 或 WebSocket 直连 Rust？
**没有。** `db751d8` 采用的是 **Tauri v2 原生二进制 IPC（Binary IPC）**，而非 Web Worker + WebTransport 或本地 WebSocket：
- **下行（Rust → JS）**：`src-tauri/src/tunnel_manager.rs` 使用 `tauri::ipc::Channel<InvokeResponseBody>` 推送 `InvokeResponseBody::Raw(frame)`（命令 `tauri_tunnel_attach_rx_channel`），在二进制帧头部打包 `[pid_len(1B) | addr_len(1B) | peer_id | from_addr | payload]`，彻底消除了此前 `app.emit` 序列化 `Vec<u8>` 为 JSON `number[]` 的膨胀与 GC 开销（仅在 `Channel` 不可用时降级回 `rc-tunnel-packet` 事件）。
- **上行（JS → Rust）**：`RustTunnelTransport.ts` 通过 `invoke('tauri_tunnel_send_raw', bytes, { headers })` 发送 `Uint8Array` 原始字节流，Rust 端通过 `tauri::ipc::Request` 读取 `InvokeBody::Raw(raw)`，用 HTTP-like Headers（`x-session-id` / `x-peer-id` / `x-target-addr`）传递路由元信息（失败时自动降级为 JSON `tauri_tunnel_send`）。
- **运行线程**：JS 侧的收发与 `TunnelReliabilityLayer` 仍运行在 **WebView 主线程**，未剥离到独立 Web Worker。
- **架构评价**：对于 60–120Hz、单包通常 <1KB 的状态/输入数据包，Tauri Binary IPC 的性能已经足够且零额外端口/TLS/鉴权复杂度（WebTransport 在本地需要自签名证书 + HTTP/3，本地 WebSocket 需要额外引入 `tokio-tungstenite` 并做端口防 CSRF 鉴权）。**数据链路走 Rust UDP + 音频链路保留 WebRTC `MediaStreamTrack` 的混合模式在当前设计下完全可行。**

### 2. Tauri 交换的 Token 与 WebRTC 是同一套逻辑吗？4 服务器架构需要跟着调吗？
- **完全不需要改动 4 服务器架构（Multi-Relay E2EE 信令池）。**
- **原因**：在 `SignalTokenCodec.ts` 与 `webrtcHandshakePipeline.ts` 中，Rust Tunnel 的端点描述符 `tunnelEndpoint`（包含 `localIp`、`localPort`、`externalIp`、`externalPort`、`upnpMapped`）是**直接内嵌在现有的 `RL_OFFER_` / `RL_ANSWER_` 信令 Token 载荷中**。
- 对 4 服务器（Cloudflare WebSocket Worker / Supabase Realtime / MQTT Broker 等）而言，它们只负责透传加密后的 Offer/Answer 信令信封，完全不感知载荷里多了一个 `tunnelEndpoint` 字段。
- **兼容性兜底**：如果双方位于对称型 NAT（Symmetric NAT）且路由器关闭了 UPnP，Rust UDP 盲打洞失败时，由于 `HandshakeSession` 仍同步完成了 WebRTC SDP/ICE 协商，连接会自动走 WebRTC DataChannel（STUN/TURN），不会断连。

---

## 二、沙盒实测状态

在 `/work/RocketCompanion`（checkout 至 `db751d8`）完成了全量静态检查与编译验证：
1. **`pnpm check`（`radar.ts` + `tsc --noEmit`）**：**PASS**（1395/1395 TS 文件，0 报错，0 循环依赖，0 越层导入）。
2. **`pnpm build`（Vite 生产构建）**：**PASS**（构建无报错）。
3. **Rust 工具链补齐与编译检查**：沙盒初始缺少 `gcc` 及 GTK/WebKit2GTK 开发依赖，已通过 `apt-get` 补齐 `gcc`、`pkg-config`、`libgtk-3-dev`、`libwebkit2gtk-4.1-dev` 等工具链；随后在 `src-tauri/` 执行：
   - **`cargo check`**：**PASS**（0 error，仅 `window_manager.rs` 存在 2 处与本 PR 无关的 macOS 未使用变量 warning）。
   - **`cargo test`**：**PASS**（0 error，当前 `src-tauri` 尚无 Rust 单元测试）。

---

## 三、代码审查发现的亮点与待修复 Bug

### 做得好的部分（相较此前审查已补齐）
1. **已接入主链路**：`HostSessionController` 通过 `HostTunnelCoordinator` 共享单个 UDP Socket + UPnP 端口映射服务多 Peer；`P2PWebRTCChannel` 为 Client 自动创建 `TunnelPeerBridge`；`DataChannelTransport.sendViaTunnelOrFallback` 优先路由至 `rust-upnp-udp`，未就绪时无缝回退 WebRTC DataChannel。
2. **多 Peer 路由与打洞过滤**：`tunnel_manager.rs` 已维护 `peers: HashMap<String, SocketAddr>` 与 `addr_to_peer` 反向映射，且 `rx_thread` 正确拦截了 4 字节 `RCPH` / `RCPA` 打洞包并触发 `rc-tunnel-peer-ready` 事件，不再透传给业务层。
3. **MTU 分片与可靠层**：新增 `TunnelReliabilityLayer.ts`（1160B 分片、2B 不可靠单包头、可靠帧重传与去重）。

### 必须修复的 2 个 Bug（阻塞 PR 合入）
1. **【高危】`TunnelReliabilityLayer.ts` 多片可靠消息 ACK 过早，丢包会导致可靠帧永久丢失**
   - **位置**：`src/network/transport/tunnel/TunnelReliabilityLayer.ts` 第 202–222 行。
   - **原因**：收到 `KIND_RELIABLE_FRAG` 后，代码在 **第 207 行无条件调用 `this.sendAck(seq)`**，然后才调用 `ingestFragment(...)`。当一条可靠消息被拆为多个分片（`total > 1`）时，只要第 0 片到达，接收端就立刻回复 `ACK(seq)`；发送端收到后直接 `this.pendingReliable.delete(ackSeq)` 停止重传。若后续分片（如第 1/2 片）在 UDP 传输中丢失，发送端不会再重传，接收端等待 3 秒后 GC 丢弃，导致 `RELIABLE_CONTROL` 等关键控制帧静默丢失。
   - **修复建议**：仅当 `completed === true`（或 `total === 1` / 全部分片重组完成，以及 `recentReliableSeqs.has(seq)` 命中重复重传时）才调用 `this.sendAck(seq)`。
2. **【中危】`P2PWebRTCChannel.ts` 的 `handleWebRtcFailureDisconnect()` 未豁免已就绪的 Rust Tunnel**
   - **位置**：`src/network/transport/P2PWebRTCChannel.ts` 第 340–345 行。
   - **原因**：`markDataChannelClosed`（第 179 行）正确判断了 `if (this.tunnelBridge?.isReady) return;`，但 `handleWebRtcFailureDisconnect()`（当 ICE/PC 状态变为 `failed` 时触发）只检查了 `if (this.isCrossTab) return;`，未检查 `this.tunnelBridge?.isReady`。在“WebRTC 被防火墙阻断、但 Rust UPnP UDP 隧道已连通”的场景下，WebRTC ICE 超时失败会错误地把 `this.isOpen` 置为 `false` 并触发 `onDisconnected()`，强行中断正常的 Rust UDP 会话。
   - **修复建议**：在 `handleWebRtcFailureDisconnect()` 开头加上 `if (this.isCrossTab || this.tunnelBridge?.isReady) return;`。

---

## 四、实机验证方案（如何验证 & 如何观察走哪条 Tunnel）

> 注：当前沙盒为无头 Linux 容器且无 UPnP 网关，无法直接拉起 Tauri GUI 与真实浏览器进行端到端 UDP 联机，以下为本地真机可直接执行的验证步骤。

### 场景 A：Tauri 客户端 ↔ 浏览器（验证兼容性与自动回退）
- **理论预期**：标准浏览器沙盒**不支持原生 UDP Socket**，因此 Tauri ↔ 浏览器 **无法** 建立 `rust-upnp-udp` 连接，但必须能**自动回退到 WebRTC DataChannel 正常联机**。
- **验证步骤**：
  1. Tauri 端作为 Host 启动（`pnpm tauri dev`），浏览器端作为 Client 打开对应前端页面加入同一房间（或反向）。
  2. **Console 观察**：在两端 DevTools Console 检查连接对象：
     - `channel.transportMode` 应为 `'webrtc'`（绝不能卡在 `'none'` 或报错断线）。
     - 若 WebRTC 建立成功，双端可正常同步位置/颜色。
  3. **Rust Terminal 观察**：Tauri 终端调用或查看 `tauri_tunnel_get_status`（或 `channel.tunnelBridge` 状态），此时 `peerCount` 为 `1`（Host 尝试连接了对端），但 `bytesReceived === 0`、`tunnelBridge.isReady === false`——证明没有收到浏览器发来的 `RCPH` UDP 包，系统成功走 WebRTC 兜底。

### 场景 B：Tauri 客户端 ↔ Tauri 客户端（验证 `rust-upnp-udp` 主链路）
- **验证步骤**：
  1. 在两台支持 UPnP 的主机（或同机双开 Tauri 实例）运行 `pnpm tauri dev`。
  2. Host 建房、Client 加入，完成信令 Token 交换。
  3. **Console 观察（最快确认方式）**：
     - 在 WebView DevTools Console 查看 `channel.transportMode`，成功打洞后应变为 **`'rust-upnp-udp'`**。
     - 检查 `channel.tunnelBridge?.isReady === true`。
  4. **Rust Terminal / 原生状态观察**：
     - 在 Console 执行 `await window.__TAURI__.core.invoke('tauri_tunnel_get_status', { sessionId: channel.tunnelBridge.sessionId })`（Host 端为 `rc_host_<roomId>`）。
     - 观察返回 JSON：`open: true`、`upnpMapped: true`、`peerCount >= 1`，且随游戏进行 **`bytesSent` 与 `bytesReceived` 持续增长**，即可 100% 实锤流量正经过 Rust UDP Socket + Tauri Binary IPC 传输。
     - 若需在 Rust 终端直接打点，可在 `src-tauri/src/tunnel_manager.rs` 的 `HOLE_PUNCH_PROBE` / `HOLE_PUNCH_ACK` 分支临时加 `log::info!("[tunnel] peer ready from {}", src_addr);` 观察打洞瞬间的输出。

---

## 五、PR 结论

- **架构与 4 服务器兼容性**：通过，无需改动服务器。
- **能否现在直接合入 `alpha`**：**建议先修复上述 2 个 Bug（`TunnelReliabilityLayer` 分片 ACK 时机 + `P2PWebRTCChannel.handleWebRtcFailureDisconnect` 豁免），并在本地双 Tauri + Tauri/浏览器各跑通一次场景验证后，再合并 PR。**
