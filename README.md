# Cellular 环境下 iOS 远程访问本地 Agent Workflow 方案深度调研报告

**调研对象仓库**：
- `Antigravi-ty/RocketCompanion`（P2P / 多路信令 / WebRTC 架构参考）
- `Antigravi-ty/agyPreviewInstant`（本地 Agent Workflow 桌面端与 LAN Companion 前端）

---

## 一、 核心结论与决策摘要

| 评估维度 | 路线 A：iOS PWA + 自签 Root CA + RocketCompanion P2P 架构 | 路线 B1：SideStore + LocalDevVPN + LiveContainer（免费远程续签） | 路线 B2：付费 Apple Developer ($99/yr) + TestFlight / Ad-Hoc OTA |
| :--- | :--- | :--- | :--- |
| **理论与工程可行性** | **高（强烈推荐作为主力方案）** | 中高（可行，但多设备运维成本高） | 极高（最省心，适合多设备） |
| **Cellular 蜂窝网络可用性** | **可用**（EMQX WSS 信令 + WebRTC，建议保留 EMQX 加密消息作为对称型 NAT 兜底） | **可用**（纯手机端完成重签与回环 VPN 安装，无需电脑在线） | **可用**（原生支持全球 OTA / TestFlight） |
| **无需物理连接持续更新** | **天然支持**（内置按钮触发 Service Worker 更新即可） | **支持**（首次 USB 生成 Pairing File 后，后续 7 天内可无线/蜂窝自动或手动续签） | **支持**（90 天 TestFlight / 1 年 Ad-Hoc，全程无线 OTA） |
| **多设备支持 (5~20 台)** | **无限制**（安装一次 Root CA + 添加到主屏幕即可） | **受限严重**：单个免费 Apple ID 每 7 天最多绑定 **3 台设备**、最多 **3 个 App**（可用 LiveContainer 绕过 3 App 限制，但 10~20 台设备需拆分 4~7 个免费 Apple ID） | **完全满足**（单账号支持 100 台 iPhone + 100 台 iPad） |
| **后续 MITM 操作契合度** | **极佳**（手机与本地共享同一套自签 Root CA，PWA 走系统 WebKit TLS 栈，无证书固定阻碍） | 一般（原生 App 若开启 ATS 或自建网络栈需额外信任配置） | 良好 |
| **综合推荐度** | **★★★★★（当前无 VPS 条件下最优解）** | **★★★☆☆（仅适合 1~3 台主力机的免费原生替代）** | **★★★★☆（若后续决定走原生且设备 > 5 台时的最终归宿）** |

**一句话建议**：
在**没有 always-accessible VPS**、且 **ZeroTier/NetBird/Tailscale 均不稳定**的前提下，**路线 A（iOS PWA + 自签 Root CA + 复用 RocketCompanion 的 EMQX/WebRTC 架构）不仅理论完全可行，而且与 `agyPreviewInstant` 的现有代码底座高度契合**。至于原生 App 方案，若坚持免费且要覆盖 5~20 台设备，Apple 免费账号的「3 台设备 UDID 限制」会让你陷入多账号轮换的泥潭，**仅建议在 ≤3 台设备上使用 SideStore 方案**。

---

## 二、 路线 A 深度技术评估：iOS PWA + 自签 Root CA + RocketCompanion P2P

### 2.1 两个仓库的现状盘点（基于源码实测）

1. **`agyPreviewInstant` 现状**：
   - **PWA 外壳已就绪一半**：`public/manifest.json` 已配置 `"display": "standalone"`、`"start_url": "/"`、`"scope": "/"` 及完整图标（`pwa-192x192.png`、`pwa-512x512.png`、`apple-touch-icon.png`）；`index.html` 已包含 `apple-mobile-web-app-capable`、`viewport-fit=cover` 等 iOS 独立应用 Meta 标签。
   - **缺失离线缓存核心**：仓库中**尚未注册任何 Service Worker**（`src/` 中无 `serviceWorker` 注册逻辑，也未引入 `vite-plugin-pwa`）。目前每次在手机打开 PWA 仍依赖重新拉取静态资源。
   - **远程传输层尚未打通**：`src-tauri/src/commands/lan_commands.rs` 与 `lan_http_io.rs` 启动的是绑定在局域网 IP 的明文 HTTP/1.1 + WebSocket 服务（`http://<lanIp>:<port>/`）；前端 `src/services/tauriBackendClient.ts` 的 `dispatch_rpc_command` 仅支持 Tauri IPC 或同源 LAN HTTP/WS。虽然项目已集成 `src/services/mqttWebSocketSubscriber.ts`（EMQX WSS）和 `src/services/bemfaHeartbeatClient.ts`，但它们目前仅用于遥测与心跳，**尚未承载任务下发与结果同步的 RPC 通道**。

2. **`RocketCompanion` 现状**：
   - **高可用多路信令池**：`SignalingMultiRelayTransportPool.ts`、`MqttWebSocketClient.ts` 与 `SignalingRelayLifecyclePolicy.ts` 实现了多路 Relay（EMQX WSS / Supabase Realtime / WS）错峰并发探测、自动重连与健康度监督。
   - **端到端加密信令**：`SignalingCryptoSession.ts` 基于 Web Crypto API 实现了 **ECDH (P-256) 临时密钥交换 + HKDF-SHA256 密钥派生 + AES-256-GCM 信令信封加密**，即便经过公网公共/免费 EMQX Broker，传输内容也是端到端密文。
   - **WebRTC 与 Bridge 兜底机制**：`P2PWebRTCChannel.ts` 与 `WebRTCConnection.ts` 封装了完整的 SDP/ICE 协商与 `RTCDataChannel` 传输；同时 `SignalingMultiRelayTransportPool.ts` 中的 `sendBridgeOnRelay` 已具备通过 MQTT/Supabase 中继帧的能力。

---

### 2.2 关键可行性验证与 iOS WebKit 隐形坑点

#### 1. 自签 Root CA 与 iOS PWA Service Worker 的兼容性
- **硬性规则**：iOS WebKit 强制要求 **HTTPS 安全上下文**才能注册 Service Worker 和启用 WebRTC/WebCrypto。
- **可行性结论**：**完全可行，但必须遵循正确的证书链层级**。
  - 若你在本地使用「裸自签名叶证书（Self-signed Leaf Cert）」，iOS Safari/PWA 往往会静默拒绝注册 Service Worker。
  - 若你使用的是**已安装到 iOS 并开启「设置 -> 通用 -> 关于本机 -> 证书信任设置 -> 针对根证书启用完全信任」的自签名 Root CA**，再用该 Root CA 签发本地 Server 证书（满足 Apple 规范：含 `SAN` 扩展匹配实际域名或私有 IP、EKU 包含 `serverAuth`、有效期 ≤ 825 天、RSA ≥ 2048 或 ECDSA P-256），则 WebKit 会将其视为**受系统完全信任的 HTTPS 站点**，Service Worker 注册、CacheStorage 读写及 `wss://` 连接均可正常工作。

#### 2. 本地缓存“不主动拉新 + 手动按钮更新”的实现机制
这正是 Service Worker **App Shell 缓存模式**最擅长的场景：
- **策略选择**：
  - 对 Vite 构建产物（带内容 Hash 的 JS/CSS/WASM）与 `index.html` 均采用 **`CacheFirst`** 策略。
  - 每次从桌面打开 Standalone PWA 时，Service Worker 直接从本地 `CacheStorage` 秒级返回上一版本的 HTML 与 JS Bundle，**零网络请求即可渲染出完整 UI**。
- **手动触发更新**：
  - 在 PWA 设置页或顶部栏提供一个「Check for Updates / Force Refresh」按钮。
  - 点击时调用 `navigator.serviceWorker.getRegistration().then(reg => reg?.update())`；检测到 `waiting` 状态的新 Worker 后，发送 `SKIP_WAITING` 消息并 `window.location.reload()`，即可拉取本地 Server（或 P2P 推送）的最新前端包。
- **iOS 缓存驱逐（ITP）注意事项**：
  - iOS 17+ 对添加到主屏幕的 Standalone PWA 分配独立存储容器，通常不会像普通 Safari 标签页那样在 7 天未访问后被 ITP 强行清空。
  - 但 WebKit 仍存在隐式容量上限（通常建议单个 PWA 缓存控制在 **50 MB 以内**以防系统空间紧张时被静默驱逐）。`agyPreviewInstant` 为轻量级 React + Primer UI 应用，构建后总体积仅数 MB，远低于安全阈值。

#### 3. Cellular 蜂窝网络下的 WebRTC 穿透率与无 VPS 兜底方案
- **现实瓶颈**：`WebRTCConnection.ts` 目前默认仅配置了 Google/Cloudflare/Mozilla 的公共 STUN 服务器（`DEFAULT_STUN_SERVERS`），`turn` 服务器需通过 `credentialBundleManager.getTurnIceServers()` 外部注入。在 **4G/5G 蜂窝网络（普遍存在 CGNAT / 对称型 Symmetric NAT 及运营商 UDP QoS 限速）**下，纯 STUN 的 P2P 直连成功率通常在 **60%~75%** 左右，无法做到 100% 稳定。
- **破局方案（零 VPS 成本，完美契合 Agent Workflow）**：
  - `RocketCompanion` 是 60fps 实时物理同步游戏，必须依赖低延迟 UDP WebRTC；但 **`agyPreviewInstant` 是文本/JSON 驱动的 Agent Workflow**（发指令、收流式输出、审批选项），对毫秒级抖动不敏感！
  - 因此在 `agyPreviewInstant` 中可以采用 **「EMQX E2EE 信令通道 + WebRTC 优先 + EMQX E2EE 加密数据通道兜底」的双模架构**：
    1. 打开 PWA 后，立即通过 `MqttWebSocketClient` 连接 EMQX（`wss://` 走 443 端口，在 Cellular 下穿透率接近 100%）。
    2. 双方通过 `SignalingCryptoSession` 完成 ECDH 密钥协商（获得共享 `AES-256-GCM` 会话密钥）。
    3. 并行尝试建立 WebRTC `RTCDataChannel`：若当前蜂窝 NAT 允许直连，则升级为点对点直连（传输大体积日志/Diff 更快、不耗 Broker 流量）；**若 ICE 失败或超时（如 3 秒），无缝降级为直接通过 EMQX WSS 发送 AES-256-GCM 加密的 RPC 帧**（类比 `SignalingMultiRelayTransportPool` 中的 `sendBridgeOnRelay`）。
  - 也可补充免费额度的公共 TURN（如 Cloudflare Calls / Metered Open Relay 免费层）作为 WebRTC 中继备选，进一步提高直连率。

#### 4. 本地电脑端（Host）如何接入 P2P 架构？
目前 `agyPreviewInstant` 的本地端是 Tauri (Rust + WebView)：
- **快速落地路径（Zero-Rust-WebRTC）**：直接在 Tauri 桌面端的前端窗口中运行 `P2PWebRTCChannel` + `MqttWebSocketClient` 作为 `role: 'host'`。手机 PWA 发来的加密 RPC 指令，由桌面端前端解密后直接调用 `invoke(cmd, args)` 转给 Rust 后端执行，再把结果加密回传。优点是 **100% 复用 TypeScript 现成代码**；缺点是桌面端 App 需保持开启（可通过最小化到系统托盘运行）。
- **iOS 前后台切换**：iOS PWA 切到后台 5~30 秒后 JS 会被挂起，WebRTC 与 WSS 必然断开。前端需监听 `visibilitychange` 事件，回到前台时自动触发 EMQX 重连并拉取本地桌面端最新的任务状态快照（`list_tasks` / `get_task_detail`），实现体感上的“无感恢复”。

---

## 三、 路线 B 深度调研：Xcode 自签与“首次插线、后续免费远程续签”方案

用户希望评估：**在首次物理连接（USB）安装 App 后，后续能否在不需要每次物理插线、且不花钱的情况下，远程给 5~20 台设备持续续签/更新 App？**

### 3.1 Apple 免费开发者账号（Free Apple ID）的四大硬性限制
无论采用任何开源自签工具，只要使用免费 Apple ID，都受制于 Apple 服务端的硬性配额：
1. **7 天有效期**：签名证书与 Provisioning Profile 固定 7 天失效，过期后 App 闪退（数据保留，重签后恢复）。
2. **每台设备最多 3 个自签 App**：含签名工具自身在内，最多同时激活 3 个（可通过 **LiveContainer** 绕过）。
3. **每 7 天最多 10 个 App ID**：每个 Apple ID 每周最多注册 10 个不同的 Bundle Identifier。
4. **【致命限制】单账号设备数上限（通常为 3 台 UDID / 7 天）**：
   - 免费 Apple ID 每 7 天周期内最多只能往 **3 台不同 UDID** 的 iOS 设备签发部署。
   - **对你的直接影响**：如果你有 **5~20 台设备**，单个免费 Apple ID **完全不够用**！你必须注册并维护 **2~7 个不同的免费 Apple ID**（按每 3 台设备绑定 1 个账号分组管理）。

### 3.2 主流免费远程续签方案横向对比

#### 方案 B1：SideStore + LocalDevVPN（或 StosVPN）+ LiveContainer
- **工作原理**：
  1. **首次物理连接**：用电脑（macOS/Windows/Linux 均可）通过 `iloader` 将 SideStore 刷入 iPhone，并在同一时间生成该设备的 **Pairing File（配对文件）** 写入 SideStore。
  2. **后续脱离电脑**：手机上安装 App Store 免费的 **LocalDevVPN / StosVPN**（本地回环虚拟 VPN，**不经过任何外部服务器、不占 3-App 名额**）。需要续签或更新版本时，开启 LocalDevVPN，SideStore 直接在手机端利用配对文件模拟 Xcode 无线调试协议完成重签与本地安装。
- **是否支持 Cellular 蜂窝网络远程续签**：**支持！** 只要有蜂窝网络即可随时随地刷新，**完全不需要电脑开机，不需要在同一局域网，不需要 VPS**。
- **如何解决 App 更新分发**：你只需把新编译的 `.ipa` 托管到任意静态地址（如 GitHub Releases 私有/公开库或自建静态 JSON Source），手机端 SideStore 即可一键拉取新 `.ipa` 并重签更新。
- **3-App 限制绕过**：使用 `LiveContainer + SideStore` 合体版，只占用 1 个名额即可在容器内运行多个 App（注意：涉及复杂后台权限或特殊 Entitlements 的原生 App 需实测兼容性，但 WebView/Tauri Mobile 壳子通常可正常挂载）。
- **核心痛点**：
  1. **iOS 单 VPN 互斥**：iOS 系统同一时刻只能启用一个 VPN。如果手机平时挂着其他 VPN/代理工具，每次续签时必须手动切到 LocalDevVPN，签完再切回。
  2. **Anisette 依赖**：Apple ID 登录与重签依赖 Anisette 服务（SideStore 内置公共节点偶有波动，如 `-80045` / `-80046` 报错，需切换节点或自建 Docker）。
  3. **多设备规模化噩梦**：5~20 台设备意味着你要在每台设备上单独做 USB 配对、配置 2~7 个不同的 Apple ID，且每 7 天要在每台设备上点一次续签（或依赖并不算 100% 可靠的 iOS 后台定时任务）。

#### 方案 B2：AltStore Classic（含 2.3+ Remote AltServer）
- **工作原理**：首次 USB 安装 AltStore，后续通过电脑端运行的 AltServer 无线续签。
- **局限性**：传统 AltStore 要求手机与电脑处于同一局域网（依赖 mDNS/Bonjour）；虽然新版 AltStore 2.3rc 支持 Remote AltServer，但跨 Cellular 仍需要手机能稳定直连你电脑上的 AltServer（又回到了 ZeroTier/Tailscale 不稳定或无公网 IP 的老问题）。因此**在无 VPS 的 Cellular 场景下不如 SideStore**。

#### 方案 B3：Xcode 无线调试（`xcodebuild` / `xcrun devicectl`）
- 首次 USB 在 Xcode 中勾选 *Connect via network* 后，同局域网可无线部署；但底层依赖 `usbmuxd` 与 mDNS 发现，**无法直接穿透 Cellular 蜂窝网络**，除非建立稳定的三层 VPN 隧道。

#### 方案 B4：付费妥协参考（Apple Developer Program $99/年）
- 若未来确实需要为 **10~20 台设备**长期维护原生 App，建议重新评估 $99/年账号的 ROI：
  - 支持 **100 台 iPhone + 100 台 iPad**，彻底告别多 Apple ID 分组。
  - **TestFlight**：90 天有效期，纯远程 OTA 推送更新，**零配置、零配对文件、零 VPN 冲突**。
  - **Ad-Hoc + OTA**：1 年有效期，把签名后的 `.ipa` 与 `manifest.plist` 放在任意 HTTPS 下，用 `itms-services://` 链接即可在 Cellular 下随时点击安装更新。

---

## 四、 路线落地建议：如何低成本改造 `agyPreviewInstant` 实现路线 A

如果决定采纳 **路线 A（PWA + 自签 Root CA + RocketCompanion P2P）**，改造路径非常清晰，无需推翻现有架构：

1. **HTTPS 与证书准备（本地 Server 侧）**：
   - 使用你已信任的 Root CA，为本地开发/运行环境签发一张带 SAN（如 `localhost`, `127.0.0.1`, 局域网 IP 及自定义内网域名）的 TLS 证书。
   - 若直接用局域网 Server 提供 PWA 初次安装与后续手动更新，可在 `src-tauri/src/commands/lan_commands.rs` 中接入 `tokio-rustls`，或前端用 Caddy/Nginx/Vite HTTPS 代理一层 `14200` 端口。
2. **补齐 PWA Service Worker 与手动更新 UI（`agyPreviewInstant` 前端）**：
   - 引入 `vite-plugin-pwa`（配置 `registerType: 'prompt'`，禁用自动静默刷新）或编写极简 `public/sw.js`：对 App Shell 采用 `CacheFirst`，确保在 Cellular 下哪怕本地服务未启动也能秒开界面。
   - 在界面顶部/设置抽屉增加 **「Update App Shell」** 按钮，手动触发 `reg.update()` + `skipWaiting()`。
3. **移植 `RocketCompanion` 传输层并增加 EMQX 兜底**：
   - 将 `RocketCompanion/src/network/transport/` 中的核心无 UI 模块（`MqttWebSocketClient.ts`、`SignalingCryptoSession.ts`、`P2PWebRTCChannel.ts`、`SignalingMultiRelayTransportPool.ts`）抽离或同步到 `agyPreviewInstant/src/services/p2p/`。
   - 封装一层 **Unified Remote RPC Client**：在 `src/services/tauriBackendClient.ts` 中新增 `remote-p2p` 运行模式（当既不是 Tauri 桌面端、又无法直连 LAN HTTP 时自动激活）。
   - 消息路由策略：**WebRTC DataChannel 已连接时优先走 P2P；未连接或 Cellular NAT 打洞失败时，自动通过 EMQX WSS 发送 AES-256-GCM 加密的 RPC 请求/响应帧**。
4. **桌面端 Host Bridge**：
   - 在 `agyPreviewInstant` 桌面端运行一个轻量级 Host 监听实例，收到手机端发来的加密指令后，桥接调用本地 Tauri IPC（`create_task`、`refresh_task`、`get_task_detail` 等）并将流式输出回推给手机。

---

## 五、 附带问题排查：AntiGravity API 排队与卡顿现状

在 `agyPreviewInstant/src-tauri/src/google_api/client.rs` 中，任务底层调用的是 Google Gemini Interactions API（`https://generativelanguage.googleapis.com/v1beta/interactions`，配合如 `antigravity-preview-05-2026` 等托管 Agent）。

根据 **Google AI Developers Forum (`discuss.ai.google.dev`)** 近期公开讨论汇总 [1][2]：
1. **现状确认**：社区已有大量开发者反馈 Antigravity 服务出现严重排队、单步挂起 5~10 分钟、高负载下连接被远端中断（`due to high demand`），以及特定后端集群（如 Gemini 3.8 serving cluster）出现高 TTFT（首字延迟）与流式输出卡顿的回归问题。
2. **是否有预计修复时间（ETA）？**
   - **截至目前，Google 官方尚未公布统一的全局修复 ETA**。
3. **为什么网页端相对较好？**
   - 网页端（Web UI）与 API / CLI / Antigravity 后台长任务接口通常路由至不同的前端网关调度池、并发配额队列与默认模型集群，且网页端交互式对话更容易命中高优先级推理资源池，而长链路 Agent 后台任务更容易在算力紧张时触发限流排队或降级。
4. **官方建议的临时缓解措施**：
   - **切换模型/变体**：在可用时临时切换测试其他后端模型（例如 `Gemini 4 Argon`、`Gemini 4 Hydrogen`、`Gemini 3.8 Flash` 或 Claude 变体）以避开拥堵集群 [2]。
   - **重置会话路由**：清除缓存的 Session Token（在 IDE/CLI 中执行 `Antigravity: Reset Onboarding` 或重新建立认证会话），强制重新分配到健康的后端生产实例 [2]。
   - **检查推理层级**：若通过 Google AI Studio / Gemini API 调用，留意是否处于 Standard/Flex 排队层，有条件时可评估 Priority Inference 通道 [3]。

---

## 参考资料
- [1] [Antigravity Slow - Google AI Developers Forum](https://discuss.ai.google.dev/t/antigravity-slow/172219)
- [2] [Severe latency regression on Gemini 3.8 in Antigravity - Google AI Developers Forum](https://discuss.ai.google.dev/t/severe-latency-regression-on-gemini-3-8-in-antigravity-unusually-high-ttft-and-slow-streaming/191002)
- [3] [Priority inference - Gemini API - Google AI Studio](https://aistudio.google.com/docs/priority-inference)
- [4] [Frequently Asked Questions - SideStore Docs](https://docs.sidestore.io/docs/faq)
- [5] [Pairing File - SideStore Docs](https://docs.sidestore.io/docs/advanced/pairing-file)
- [6] [LiveContainer + SideStore Documentation](https://livecontainer.github.io/docs/installation/lc_sidestore)
- [7] [Install Apps on Non-Jailbroken Devices with Sideloadly (Device UDID & App ID Limits)](https://support.corellium.com/features/apps/sideloadly)
