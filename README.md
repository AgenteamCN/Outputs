# RocketCompanion Tauri 架构与 P2P Tunnel / IPC 深度可行性评估报告

> **报告定位**：针对 `z/agy/rc/tauri-tunnel-settings`（UPnP / UDP Tunnel）、未来 Tauri 文件系统直连分支、以及 Rocket League Stats API WebSocket / Overlay 分支的全局技术预研与架构决策参考。  
> **核心结论先行**：
> 1. **能否完全绕过“进程间数据拷贝”让 WebView 前端直接调用 Rust 裸 UDP Socket？不能。** 浏览器/WebView 安全沙箱在底层禁止 JS/WASM 直接操作原生 UDP Socket，也无法直接读写虚拟网卡（TUN/TAP）的裸 IP/UDP 包。
> 2. **但能否绕过 Tauri 的 JSON IPC？完全可以，且有两条更优路径**：
>    - **首选推荐（工程与性能平衡最优）**：**Tauri v2 原生二进制 IPC** —— 接收端使用 `tauri::ipc::Channel<tauri::ipc::Response>`（直接传输 `ArrayBuffer`，零 JSON/Base64 序列化），发送端使用 `tauri::ipc::Request<'_>`（Raw `Uint8Array` Body）。实测 50 MiB 吞吐仅 ~115ms，无本地端口、无防火墙弹窗、无 Mixed Content 限制、全平台（含 Win8.1/Win10/Win11/Apple Silicon）100% 稳定。
>    - **备选方案（彻底绕过 Tauri IPC 层）**：**Rust 本地 Loopback WebSocket (`ws://127.0.0.1:<port>`)** 或 **WebTransport (`https://127.0.0.1:<port>` over QUIC)**。其中 WebTransport 可提供不可靠数据报（无 TCP 队头阻塞），但 **Win8.1（WebView2 止步 v109）与旧版 macOS WKWebView（早于 Safari 26.4）不支持**；而 `ws://127.0.0.1` 通用性极好，但走 TCP 协议栈。
> 3. **“混合传输模式”（Data 走 Rust UDP Tunnel，Voice 保留 WebRTC `MediaStreamTrack`）是现阶段最合理的 Trade-off**，原因不是 Rust 不能传音频，而是浏览器 `RTCPeerConnection` 白嫖了 C++ 级的硬件 AEC（回声消除）、NS、AGC、Opus 编码与 NetEQ 抖动缓冲。

---

## 一、现状代码审计：`z/agy/rc/tauri-tunnel-settings` 的瓶颈定位

在当前 `origin/z/agy/rc/tauri-tunnel-settings` 分支中，抽象层设计（`TunnelTransport.ts`、`TunnelFactory.ts`、`TunnelProtocol.ts`）解耦非常清晰，但底层实现存在 **4 个必须解决的阻塞点**：

1. **严重的 IPC 序列化瓶颈（JSON `number[]`）**  
   在 `src-tauri/src/tunnel_manager.rs` 中：
   ```rust
   pub struct TunnelPacketEventPayload {
       pub session_id: String,
       pub from_addr: String,
       pub data: Vec<u8>,
   }
   // rx_app.emit("rc-tunnel-packet", payload)
   ```
   在 Tauri v2 中，`app.emit` 和 `Channel<&[u8]>` 都会对 `Vec<u8>` 走 `serde::Serialize`，**将其序列化为 JSON 数字数组 `[82, 67, ...]`**（体积膨胀 3–4 倍，且前端 `new Uint8Array(ev.data)` 触发高频 JSON.parse 与 GC）。在 60–120Hz 多端同步场景下，这会成为主要 CPU 瓶颈。
2. **单 Peer 绑定限制，无法支撑 Host 1 对 N 拓扑**  
   `ActiveTunnelSession` 中仅保存 `peer_addr: Arc<Mutex<Option<SocketAddr>>>`。Host 端需要同时连接 **5 个 Player + 1 个 Broadcast（及更多观战者）**，如果每个 Peer 开一个 `session_id`，就会绑定多个本地 UDP 端口并申请多次 UPnP 映射（极易被家用路由器 UPnP 表项上限拒绝）。正确做法是 **单 UDP Socket（单 UPnP 端口映射）按 `SocketAddr` + 协议层 PeerToken 路由多 Peer**。
3. **裸 UDP 缺少可靠传输语义与 MTU 分片**  
   `TunnelProtocol.ts` 定义了 `RELIABLE_CONTROL (0x03)`、`TELESTRATOR_SYNC (0x05)`、`ROOM_ACTIVITY_SWITCH (0x21)` 等控制帧，但 `RustTunnelTransport` 直接走裸 UDP，既没有重传/有序确认机制，也未处理大于路径 MTU（~1200B）的全量快照分片，公网丢包时会直接破坏房间状态机。
4. **打洞包未过滤**：`tauri_tunnel_connect` 发送的 4 字节 `"RCPH"` 打洞探测包在 `rx_thread` 未被拦截，会直接透传给 JS 层。

---

## 二、核心命题深度剖析：能否规避 IPC？“透明 Rust Socket / 虚拟网卡”可行吗？

### 2.1 为什么前端 JS/WASM 无法“直接调用 Rust 暴露的 UDP Socket”？
无论是 macOS 的 `WKWebView` 还是 Windows 的 `WebView2`，本质上都是**多进程浏览器沙箱**（UI 渲染进程与网络/宿主进程隔离）。
- 浏览器沙箱**没有**原生 UDP Socket API（WASM 跑在 JS 沙箱内存里，同样受限，除非由宿主通过 JS 胶水导入能力）。
- 浏览器唯一能发 UDP 的原生接口是 `RTCPeerConnection`（WebRTC），但它强制绑定了 **ICE + DTLS（强制加密）+ SCTP（DataChannel）/ SRTP（Media）**，无法发送裸自定义 UDP 封包。
- 因此，**数据从 WebView 内存（JS/WASM）进入操作系统网络协议栈，必然要跨越一次“WebView 进程 → Rust 宿主进程”的边界**（要么通过 Tauri IPC，要么通过 `127.0.0.1` Loopback Socket）。

### 2.2 虚拟网卡（TUN/TAP，如 `wintun` / `utun`）能否实现“前端无感知走 Rust 网络”？
**结论：不能解决 IPC 问题，且权限与维护成本极高，强烈不建议。**
- **原理误区**：即使 Rust 在操作系统内核创建了一个虚拟网卡（如 `10.66.0.1`），WebView 里的 JS **依然无法直接对该网卡发裸 UDP**——它还是只能通过 `RTCPeerConnection` 或 `WebSocket` 去连那个虚拟 IP。
- **双重开销**：如果走 `RTCPeerConnection` 连虚拟网卡，数据包会先经过浏览器的 DTLS + SCTP 加密封装，进内核虚拟网卡，再被 Rust 读取解包/转发，比直接在应用层传二进制更慢。
- **极高的系统权限与签名门槛**：
  - **Windows 10/11**：使用 `wintun.dll` 创建虚拟网卡需要 **管理员权限（UAC 提权）** 且驱动需微软签名；普通玩家每次启动弹 UAC 会严重破坏体验。
  - **macOS (Apple Silicon)**：普通用户态进程无权创建 `utun` 或修改路由表，必须 `root` 提权或申请 Apple 付费开发者账号的 **`NetworkExtension`（System Extension）特殊权限审批**。

### 2.3 五种传输桥接方案的横向对比（Trade-off 决策矩阵）

为了把“IPC 开销”降到最低，同时评估“绕过 Tauri IPC”的真实收益，我们对比以下 5 种可行架构：

| 方案 | 原理与数据流 | 延迟与 CPU 开销 | 协议特性 | 跨平台兼容性 (Win8.1 / Win10-11 / Apple Silicon) | 工程复杂度 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. 现状：`app.emit` + JSON** | Rust `Vec<u8>` → JSON `number[]` → IPC → `JSON.parse` → `Uint8Array` | **极差**（3~4x 体积膨胀，高频 GC） | 不可靠 | 全平台 | 低（不可用） |
| **2. Tauri v2 二进制 IPC（推荐）** | RX: `Channel<tauri::ipc::Response>` → 直接传 `ArrayBuffer`<br>TX: `tauri::ipc::Request` Raw Body | **极低**（实测 50MiB 仅 ~115ms，零 JSON/Base64，仅一次进程内存拷贝） | 由 Rust 侧控制（可结合 KCP/ARQ 实现可靠+不可靠双通道） | **100% 全平台兼容**（无端口冲突、无 Mixed Content、无防火墙弹窗） | **低**（直接改造现有 `tunnel_manager.rs`） |
| **3. 本地 Loopback WebSocket** | Rust 启动 `tokio-tungstenite` 监听 `ws://127.0.0.1:<随机端口>`；JS 直接 `new WebSocket` (`arraybuffer`) | **极低**（完全绕过 Tauri IPC 调度，走 OS 内核 Loopback TCP） | **有 TCP 队头阻塞**（但本地 `127.0.0.1` 几乎 0 丢包，影响微弱） | 极好（`ws://127.0.0.1` 在默认配置下均可用；注意 macOS 若开启 `useHttpsScheme` 可能触发 Mixed Content） | 中（需管理本地 WS 端口、鉴权 Token 防本地其他进程连接） |
| **4. 本地 WebTransport (QUIC)** | Rust 启动 `wtransport`/`quinn` 监听 `https://127.0.0.1:<port>`；JS 用 `WebTransport` 走 **Datagrams + Streams** | **理论最优**（绕过 Tauri IPC + 原生不可靠数据报 + 无 HoL 阻塞） | **原生双通道**（Datagram + Stream） | **有硬伤**：Safari 26.4 (2026.03) 才支持，**macOS 旧版 WKWebView 不支持；Windows 8.1 (WebView2 v109) 完全不支持** | 高（需生成 <=14 天自签名证书 + `serverCertificateHashes`） |
| **5. 本地伪装 WebRTC Peer (`str0m`)** | JS 正常创建 `RTCPeerConnection`，ICE 只连 Rust 在 `127.0.0.1` 跑的 `str0m` Sans-IO WebRTC 栈，Rust 解出数据再发公网 UDP | **较差**（JS 侧与 Rust 侧各跑一遍 DTLS 加解密 + SCTP 拥塞控制） | 完整 WebRTC 语义 | 极好（上层业务零感知） | **极高**（不值得用本地加解密换取“上层无感知”） |

#### 架构师建议：为什么“方案 2（Tauri v2 二进制 IPC）”是当前最优解？
1. **性能已足够过剩**：RocketCompanion 即使在 Host 端 60–120Hz 同步 5 个 Player + 广播，总数据吞吐量也仅在 **100 KB/s – 800 KB/s** 级别（远低于 50 MiB/115ms ≈ 430 MB/s 的二进制 IPC 极限）。真正卡顿的元凶是当前代码的 **JSON 序列化**，而不是 IPC 通道本身。
2. **如果未来 WebGL 重构后出现极端高频场景（如 240Hz+ 全场多端状态流）**：可将传输层微调为 **方案 3（`ws://127.0.0.1` + 二进制帧）** 或在支持的环境自动升级为 **方案 4（WebTransport）**，由于 `TunnelTransport.ts` 接口已经完全隔离，替换底层桥接方式对上层游戏逻辑 **零侵入**。
3. **至于方案 5（`str0m` 本地伪装 WebRTC）**：完全没有必要。上层应该面向 `TunnelTransport`（或统一的 `SessionTransport` 接口）编程，而不是为了迁就 `P2PWebRTCChannel` 而在本地做无意义的 DTLS 加解密。

---

## 三、为什么建议“混合传输模式”？语音不能全走 Rust Tunnel 吗？

用户疑问：*“他说要走混合传输模式，我没理解为什么不全都走 Rust tunnel？”*

核心原因在于：**WebRTC 传输语音的价值，70% 在于浏览器内置的 C++ 实时音频管线，而不仅仅是 UDP 传输。**
如果将语音也迁入 Rust UDP Tunnel，会面临两难选择：
1. **若由 JS 采集音频再发给 Rust**：
   - JS 必须通过 `AudioWorklet` 读取 PCM 帧，再自行用 WASM 编译 Opus 编码器压缩，通过 IPC/WS 发给 Rust；远端收到后再解包送入 JS 自研的 **Jitter Buffer（自适应抖动缓冲）** 播放。
   - 这不仅增加了额外的 IPC 频率（每 10ms–20ms 一帧），而且一旦脱离 `RTCPeerConnection` 的音频输出管线，浏览器内置的 **硬件级回声消除（AEC）参考信号同步** 极易失效，玩家开外放时队友会听到严重回声。
2. **若由 Rust 直接接管麦克风采集与播放（`cpal` + `audiopus` + `webrtc-audio-processing`）**：
   - 虽然能做到语音“零 IPC”，但需要重新开发一套跨平台的音频设备热插拔枚举、macOS/Windows 麦克风权限请求、以及与当前 `src/audio/voice/`（Zustand `voiceChatStore`、`VoiceAudioGraph`、`team_only` 路由）完全平行的原生音频引擎，研发周期会拉长数倍。

**结论**：**混合传输模式（Hybrid Mode）** 是现阶段性价比最高的工程决策：
- **游戏高频数据 + 可靠控制帧**：走 **Rust UDP Tunnel**（利用 UPnP / UDP 打洞解决对称 NAT/丢包/P2P 直连稳定性问题，且带宽大、延迟低）。
- **语音通信**：保留轻量级 `RTCPeerConnection`（**仅挂载 Audio `MediaStreamTrack`，不创建 DataChannel**）。音轨码率极低（~24–48kbps），且能直接复用浏览器原生 AEC/NS/AGC 与现有的 `voiceChatStore` 体系。

---

## 四、多平台底层细节与兼容性红线（Apple Silicon macOS / Win 10 & 11 / Win 8.1）

### 4.1 Apple Silicon macOS (M1/M2/M3/M4, WKWebView)
1. **macOS Sequoia (15+) / Sonoma 本地网络隐私权限（关键坑点）**：
   - `igd-next` 进行 UPnP 探测时，会向局域网发送 SSDP 组播包（`239.255.255.250:1900`）。
   - macOS 15+ 强制要求本地网络访问授权：系统会弹出 *“RocketCompanion 想要查找并连接到本地网络上的设备”*。若用户误点“不允许”，或遇到 macOS 重启后权限缓存失效的已知系统 Bug，**所有 SSDP 组播与局域网 UDP 将被内核静默拦截**。
   - **应对**：必须在 `src-tauri/Info.plist` 中声明 `NSLocalNetworkUsageDescription`（说明用于 P2P 局域网与 UPnP 端口映射），并在探测失败时在 UI 引导用户检查 *系统设置 → 隐私与安全性 → 本地网络*。
2. **签名与沙箱权限 (Entitlements)**：
   - 若开启 Hardened Runtime 或 App Sandbox 进行公证分发，必须配置 `com.apple.security.network.client` 与 `com.apple.security.network.server`，否则 `UdpSocket::bind` 会直接返回 `EPERM`。
3. **WKWebView Mixed Content 细节**：
   - 默认 `tauri://localhost` 下连接 `ws://127.0.0.1` 正常；但若未来启用 `useHttpsScheme: true`，部分版本的 WebKit 会将 `ws://127.0.0.1` 视为不安全混合内容拦截（推荐使用方案 2 二进制 IPC 可彻底规避此问题）。
4. **Apple Silicon 性能表现**：UMA 统一内存架构下，`Channel<Response>` 内存拷贝开销极低，120Hz 多端同步 CPU 占用通常 < 1%。

### 4.2 Windows 10 & Windows 11 (x86_64, WebView2)
1. **Windows 11 默认“公用网络（Public）”防火墙拦截（关键坑点）**：
   - Windows 11 新接入 Wi-Fi/以太网时默认设为 **公用网络**，Windows Defender 防火墙会默认拦截未授权应用的入站 UDP 及 SSDP 组播响应。
   - Rust 首次调用 `UdpSocket::bind("0.0.0.0:0")` 并收发数据时，会触发系统的防火墙弹窗。如果此时用户正处于游戏全屏/无边框窗口中，弹窗可能导致窗口失焦或被遮挡。
   - **应对**：在 Configurator 启动阶段或点击“开启联机/UPnP 诊断”时提前触发一次端口绑定/探测，确保防火墙放行提示在游戏对局前完成。
2. **WebView2 表现**：基于 Edge Chromium，支持现代 Web API、`ArrayBuffer` 零序列化传输、以及 `WebTransport`（若未来需要）。

### 4.3 Windows 8.1 (x86_64) —— 重大兼容性红线预警！
- **官方支持已终止**：Microsoft Edge 与 **WebView2 Runtime 已于 2023 年 1 月在 v109 版本正式停止对 Windows 7 / Windows 8.1 的支持**。
- **对本项目的直接影响**：
  1. **WebTransport 彻底不可用**：WebView2 v109 无法使用现代 WebTransport 及后续 WebAudio/WebGPU 修复。
  2. **Tauri v2 与 Win8.1 运行时风险**：Tauri v2 依赖的部分现代 Win32 / DWM API（例如后续做全屏透明 Overlay、某些窗口圆角/置顶/穿透属性）在 Win 8.1 上可能不存在或行为不一致。
  3. **分发策略**：Win 8.1 机器上无法自动安装 Evergreen 最新版 WebView2，若必须支持 Win 8.1，只能打包 **Fixed Version v109**（包体积增大 ~150MB+），且 Rust 侧调用 Win10+ API 必须通过 `LoadLibrary`/`GetProcAddress` 动态探测降级。
  4. **建议**：将 **Windows 10 (1809+) / Windows 11** 作为正式支持与测试基线，Windows 8.1 仅作“尽力兼容（Best-effort）”且**禁止在底座依赖 WebTransport**。

---

## 五、立即落地的工程方案：UPnP + 多 Peer Rust Tunnel 接入与分叉协作计划

为了不阻塞引擎/UI 侧同事的下一步开发（Modules A–E），建议按以下清晰边界立即改造并提交评审：

### 5.1 Rust 侧改造 (`src-tauri/src/tunnel_manager.rs`)
1. **单 Socket 多 Peer 路由（支撑 Host 5 Player + 1 Broadcast + N 观战）**：
   - 移除单 `peer_addr: Option<SocketAddr>`，改为一个 `UdpSocket` 维护 `peers: HashMap<String, PeerEntry>`（`peer_id -> SocketAddr`）与反向索引 `addr_to_peer: HashMap<SocketAddr, String>`。
   - **UPnP 只需映射 1 个外网 UDP 端口**，极大提升家用路由器成功率。
   - 支持 **“5 路接收 + 无上限发送”**：Host 端 `rx_thread` 按 `src_addr` 识别合法 Peer（过滤未知来源与 `"RCPH"` 4 字节打洞包）；发送端支持按 `peer_id` 定向发送或向所有已连接 Peer/Spectator 扇出（Fan-out），并与现有的 `SpectatorDaisyChainCoordinator.ts` 配合，在宿主上行带宽不足时自动将多余观战者分流到菊花链。
2. **二通道传输语义与分片（解决丢包与 MTU 限制）**：
   - 在 `TunnelProtocol.ts` 现有的 8-byte `0x5243` 信封基础上，利用 `flags` 位区分：
     - `UNRELIABLE`（物理快照 `0x01`、输入 `0x02`）：直接裸 UDP 发送（若超过 1200B 自动分片或压缩）。
     - `RELIABLE`（`RELIABLE_CONTROL 0x03`、`TELESTRATOR_SYNC 0x05`、`ROOM_ACTIVITY_SWITCH 0x21`）：在 Rust 层或 TS 传输适配器层引入轻量级 **序号 + ACK 滑窗重传（ARQ）** 与 **>1200B 分片重组**，确保控制信令与绘图同步 100% 必达。
3. **二进制 IPC 替换 JSON**：
   - `tauri_tunnel_open` 传入 `on_packet: Channel<tauri::ipc::Response>`，Rust `rx_thread` 构造 `[peer_id_len(1B) | peer_id_bytes | raw_packet_bytes]` 包装为 `tauri::ipc::Response::new(buf)` 推送给前端。
   - `tauri_tunnel_send` 使用 `request: tauri::ipc::Request<'_>` 读取 `InvokeBody::Raw(bytes)`，彻底消灭 `Vec<u8>` 的 JSON 序列化。

### 5.2 TS 顶层抽象对接（“有 Tauri 走 Rust，无 Tauri 走 Web”）
1. **升级 `TunnelTransport.ts`**：
   - 支持多 Peer 接口：`connectPeer(peerId, remoteEndpoint)`、`disconnectPeer(peerId)`、`sendToPeer(peerId, packetType, payload, reliable)`、`broadcast(packetType, payload, reliable)`。
   - `onPacket` 回调直接吐出 `(peerId: string, bytes: Uint8Array)`。
2. **信令阶段交换凭据**：
   - 在现有的 MQTT / WebSocket / Supabase 信令握手中，交换 `TunnelEndpointDescriptor`（包含 `externalIp:externalPort`、`localIp:localPort`、`upnpMapped`、`peerToken`）以及用于语音的轻量级 WebRTC Audio SDP Offer/Answer。
   - 双方拿到 Endpoint 后，先发 `"RCPH"`（包含 `peerToken`）进行双向 UDP 打洞（内网优先直连 `localIp`，公网走 UPnP `externalIp`）；若对称 NAT 打洞超时（如 1.5s 未通），自动无缝回退到 WebRTC DataChannel（`WebTunnelTransport`）。

---

## 六、后续两大独立分支的架构衔接预留

1. **分支二：文件系统直连与敏感数据绝对隔离**
   - **现状基础**：仓库 `src-tauri/src/config_store.rs` 已经实现了 `workspace/`（`<app_data_dir>/workspace`，存放 `launch-config.json`、`replays/`、`training_packs/`）与 `.vault/`（`<app_local_data_dir>/.vault`，隔离 `.rctbnspib` 凭据）的物理路径分离。
   - **实施建议**：在前端抽象 `PersistencePort`（Web 端映射 IndexedDB，Tauri 端通过二进制/文本 FS 命令直接读写 `workspace/`）。确保用户未来导出/打包分享整个 `workspace/` 目录（含配置、镜头、Replay、Training Pack）时，**物理上绝对不会包含 `.vault/` 内的服务器地址与 Token 凭据**。

2. **分支三：Rocket League Stats API WebSocket 接收/广播 & 无边框 Overlay**
   - **官方协议对接**：Rocket League 官方 `Stats API`（通过 `TAStatsAPI.ini` 开启）会在本地开启 TCP/WebSocket 端口，以 1–120Hz 推送 JSON 赛况与事件数据。
   - **接收模式**：若游戏端配置为 WebSocket，TS 前端可直接 `new WebSocket('ws://127.0.0.1:<port>')` 消费；若为纯 TCP Socket，由 Rust 宿主代理转发至前端。
   - **广播模式（模拟 Rocket League Client）**：由于浏览器/WASM **无法作为 Server 监听网络端口**，必须由 Rust（`tokio-tungstenite`）在本地开启 WebSocket Server，接收 WASM 物理引擎产出的快照并按官方 Stats API JSON 格式向外广播，供外部 HUD/OBS 读取。
   - **HUD-only 与全屏无边框 Overlay**：macOS 端保持常规窗口与冻结场地模拟即可（无需全透明）；Windows 10/11 端待配置文件就绪后，另起分支基于 WebView2 透明背景 + Win32 `WS_EX_TRANSPARENT` 穿透控制实现（注意做好 Win 8.1 降级或明确 Win10+ 要求）。
