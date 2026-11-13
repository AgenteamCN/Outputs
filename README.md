# RocketCompanion 技术深潜检查报告与下一阶段开发 Instruction

> **基准检查状态**：基于 `alpha` (`8d19408`)、Commit `db751d8` (`origin/z/agy/rc/tauri/upnp-tunnel`)、Commit `24fc4a4` (`origin/z/agy/rc/tauri/webtransport-tunnel`) 以及实验分支 `z/agy/rc/multiplayer-suite` (`9a5d063`)、`z/agy/rc/phase2-implementation` (`49ecac7`) 的实码扫描与 `pnpm run radar:audit` 验证结果。

---

## 一、核心技术问题逐项解答

### 1. Rust ↔ TS 本地通信：WebTransport / WebSocket (`24fc4a4`) vs Tauri Binary IPC (`db751d8`)

**结论先行**：你的直觉基本方向是对的，但真正决定选型的不只是“数据量大小”，而是 **方向性（JS→Rust vs Rust→JS）、调用模型（RPC vs 持久流）以及 Web Worker 线程隔离**。

#### 实码对比两套实现
1. **Binary IPC (`db751d8`)**：
   - **JS → Rust (`RustTunnelTransport.send`)**：调用 `invoke('tauri_tunnel_send_raw', Uint8Array, { headers: { 'x-rc-session': ..., 'x-rc-peer': ... } })`。虽然 Tauri v2 的 `InvokeBody::Raw(Vec<u8>)` 避免了旧版 `JSON number[]` 序列化开销，但每一次发包仍然会触发一次完整的 Tauri `invoke` 生命周期：**创建 Promise → 走 WebView 自定义协议（`ipc://localhost/...` 伪 HTTP fetch）→ 跨线程路由到 Rust Command → 返回 Promise**。
   - **Rust → JS (`tunnel_manager.rs`)**：使用 `tauri::ipc::Channel<InvokeResponseBody>` 发送 `InvokeResponseBody::Raw(frame)`（仅在未挂载 Channel 时回退 `app.emit`）。下行方向是流式的，开销其实比上行 `invoke` 小得多。
   - **致命限制**：`window.__TAURI__` / `__TAURI_INTERNALS__` **仅存在于主线程（Window）**，**Web Worker 内完全无法调用**。如果后续网络/物理/状态同步继续向 Worker 迁移，Binary IPC 必须经过主线程 `postMessage` 中转一次。

2. **WebTransport + WebSocket Localhost Bridge (`24fc4a4`)**：
   - **控制面仍走 Tauri `invoke`**：`tauri_upnp_probe`、`tauri_tunnel_open`、`connect`、`disconnect`、`close`、`status` 全部走标准 `tauri::command`（低频 RPC，无需自己维护连接与协议）。
   - **数据面走持久本地 Socket (`tunnel_bridge.rs` + `LocalBridgeClient.ts`)**：
     - 主通道：`wtransport`（本地 QUIC/WebTransport，`127.0.0.1:0` + 自签名证书 Hash 校验）。60–120Hz 不可靠状态包走 `wt.datagrams`（无队头阻塞），可靠控制包（`0x5243` flag `1<<1`）走 `wt.createUnidirectionalStream()`。
     - 显式降级：`tokio-tungstenite` 本地 WebSocket（`TCP_NODELAY`），用于 WebView 不支持 WebTransport 的环境（如 Linux WebKitGTK 或部分旧版 WKWebView）。
   - **核心优势**：
     - **Worker-Safe**：`LocalBridgeClient` 不依赖 `window.__TAURI__`，可在 Web Worker 中直连。
     - **消除上行逐包 IPC/Promise 开销**：高频小包（60–120Hz 状态/输入）与高频大包（全量快照/回放切片）均走持久二进制连接。

#### 为什么不是所有情况都走 WebTransport / WebSocket？
| 场景 | 推荐通道 | 原因 |
| :--- | :--- | :--- |
| **低频控制/查询 RPC**（开关 Tunnel、UPnP 探测、Add/Remove Peer、查询状态） | **Tauri `invoke` IPC** | 原生 Request/Response 类型安全、受 Tauri ACL/Capability 保护，无需等待本地 TLS/QUIC 握手与连接状态机。 |
| **高频小数据 (60–120Hz 状态/输入)** | **WebTransport Datagram / WS**（或下行 `Channel::Raw`） | 持久连接零 Promise/HTTP 协议头开销；支持 Web Worker 直连；QUIC Datagram 无 TCP 队头阻塞。 |
| **高频/突发大数据 (快照/Telestrator/切片)** | **WebTransport UniStream / WS** | 原生流式背压控制，不阻塞主线程事件循环与 UI 渲染。 |

---

### 2. 旧版检查报告是否已过时？`24fc4a4` 与 `db751d8` 各自解决了什么、还缺什么？

**结论**：旧报告中关于“Tauri P2P Tunnel 尚未做多 Peer 路由、未过滤 `RCPH`、还在用 `app.emit` JSON `number[]`”的部分**已经过时**；但旧报告中**“尚未真正接入主链路、缺少端到端可靠传输语义”**的问题，在两份 commit 中呈现**“各解决一半”**的互补状态：

| 能力项 | 旧报告状态 | `db751d8` (`upnp-tunnel`) | `24fc4a4` (`webtransport-tunnel`) | `9a5d063` (`multiplayer-suite`) |
| :--- | :--- | :--- | :--- | :--- |
| 单 Socket 多 Peer 路由 + 过滤 `RCPH`/`RCPA` | ❌ 未做 | ✅ 已完成 | ✅ 已完成（且拆分 `tunnel_upnp.rs`） | ✅ 继承 `24fc4a4` |
| 消除 JSON `number[]` IPC 瓶颈 | ❌ `app.emit` | ✅ Binary IPC (`InvokeBody::Raw` + `Channel::Raw`) | ✅ 本地 WebTransport + WS Bridge（Worker-safe） | ✅ 继承 `24fc4a4` |
| **公网 UDP 可靠/有序/分片层** | ❌ 缺失 | ✅ 已实现 `TunnelReliabilityLayer.ts` (315L) | ❌ **缺失**（WT uni-stream 只保了 `127.0.0.1` 本地段，公网 UDP 段仍会丢包/乱序） | ❌ **缺失** |
| **四服务器信令打包 (`SignalTokenCodec`)** | ❌ 未接入 | ✅ 已将 `tunnelEndpoint` 打包进 `SignalTokenCodec` / `HandshakeSession` | ❌ 未接入 | ⚠️ 改走 DataChannel 内 `tunnel_endpoint_advertise`（若 WebRTC 连不上则无法交换！） |
| **主链路 (`DataChannelTransport`) 接入** | ❌ 未接入 | ✅ 已通过 `TunnelPeerBridge` + `HostTunnelCoordinator` 接入 | ❌ 未接入 | ✅ 已接入 |

> **关键发现**：`24fc4a4` 拥有更优的**本地桥接层**（`tunnel_bridge.rs` + `LocalBridgeClient.ts`），但缺少 `db751d8` 的**公网 UDP 可靠传输层 (`TunnelReliabilityLayer.ts`)** 和 **信令打包 (`SignalTokenCodec` 携带 `tunnelEndpoint`)**。合并时必须**取 `24fc4a4` 的本地桥接 + `db751d8` 的可靠性层与信令打包**！

---

### 3. 四服务器信令架构与 Tunnel 优先级：你的理解对吗？

1. **“四服务器架构不用动，Tauri 会把 Tunnel 端点信息一并打包过去”**
   - **完全正确（`db751d8` 已验证）**：`SignalTokenCodec.ts` 的 `DecodedSignalTokenPayload` 直接增加了可选字段 `tunnelEndpoint?: TunnelEndpointDescriptor`，随现有 Offer/Answer Token 一起走原有四服务器信令通道（MQTT / Supabase / WS 等），**四服务器架构零改动**。
   - **注意避坑**：`9a5d063` 改为在 WebRTC DataChannel 连上之后再发 `tunnel_endpoint_advertise`——这意味着如果 WebRTC 本身打洞失败，Tunnel 端点根本交换不到！因此**必须保留 `db751d8` 的“信令阶段打包 `tunnelEndpoint`”方案**。

2. **“Web 端能直接打通就走 WebRTC；Web 端打不通再走 Rust UPnP；都打不通就自求多福”**
   - **思路非常合理，但请注意：当前 `db751d8` 和 `9a5d063` 的代码实现与你的预期正好相反！**
   - 实码检查 `DataChannelTransport.ts`（`db751d8` 的 `sendViaTunnelOrFallback` 与 `9a5d063` 的 `transmitViaTunnelOrDataChannel`）：它们都是 **只要 Tunnel `isOpen` 就无条件优先走 Native Tunnel**，只有 Tunnel 没开才走 WebRTC DataChannel。
   - **建议调整为可配置策略（默认 `webrtc-first` 或 `auto`）**：
     - **WebRTC 优先（默认）**：WebRTC P2P 直连成功时，自带 DTLS 加密、SCTP 可靠/不可靠双通道与拥塞控制，且与 WebRTC Audio Track 同属一条 `RTCPeerConnection`；仅当 WebRTC ICE 失败/超时或仅剩 TURN 中继时，自动切换至 Rust UPnP UDP Tunnel。
     - **Tunnel 优先（可选开关）**：供 Tauri 桌面端用户或低延迟测试显式开启，绕过 WebRTC SCTP 开销。

---

### 4. 现在的 Multiplayer 测试是怎么做的？需要单独构建 `headless=1` 版本吗？

**结论**：**不需要**单独构建一个发行版或专门的 `headless=1` 跳过版本！仓库里**已经有一套完整的双端自动化测试与本地环回基础设施**：

1. **现有自动化测试链路（`scripts/runMultiplayerRoomRepro.mjs`）**：
   - 运行 `pnpm run repro:multiplayer-room`，脚本会内嵌启动 Vite Dev Server（挂载 `/__rc_repro_stream` 实时遥测插件），并拉起多个 **Headless Chrome (`--headless=new`)** 实例分别扮演 Host 与 Client。
   - 通过 URL 参数控制角色、人数矩阵（`2p`、`2p2ai`、`3p3ai`）、仿真倍速（`--speed=1|2|4`）、模拟延迟/带宽（`--latency-ms`、`--bandwidth-kbps`）以及物理帧率（`60,90,100,120,144`）。
   - 通过 `reproBridgeSignaling=1`（对应 `MultiplayerRoomReproAgent.ts`），**在测试时已经支持本地桥接信令，无需真实连接外部云端四服务器**。
   - 内部还已有 `HostLoopbackChannel.ts`、`NetworkSimulator.ts`（可注入丢包/抖动/延迟）以及 `SignalingFourServerLoopbackHarness.ts`。
2. **接下来自动化测试矩阵怎么扩展**：
   - **不需要**另起炉灶搞 `headless=1` 独立构建；只需在现有 `runMultiplayerRoomRepro.mjs` + `MultiClientLoopbackSimulator.ts` 上扩展：
     - 用 Ducky AI 接管 User 输入（模拟真实玩家）+ 系统内 Bot Auto-fill，覆盖 1v1 ~ 3v3 全组合；
     - 用协议层本地 Loopback（或 Headless Chrome Tab）直接跑通 **Training 4 档 Boost / Ghost Relay、Voice `open/team_only/off` 路由、Spectate State-Only vs All、Kick/Leave 双向信令、Telestrator Hash 一致性校验**。

---

### 5. 为什么现在的 Agent “写多少弄多少，不多新增”？是 `AGENTS.md` 还是 System Instruction 导致的？

**答案是：两者叠加，且主要源于 `AGENTS.md` + `radar.ts` 门禁的强约束，加上 System Instruction 的精确执行倾向。**

1. **仓库 `AGENTS.md` 与 `src/ui-core/AGENTS.md` 的硬约束**：
   - 明确禁止“向后兼容 shim / 死代码 / 冗余门面”、“严禁未要求的多余注释”、“每个文件硬上限 < 600–800 行”、“禁止循环依赖、向上导入、Pass-Through 代理”。
   - 每次提交前必须过 `pnpm run radar`（`scripts/radar.ts`），一旦 Agent 擅自提前写了一堆“未来可能用到的接口或占位模块”，`radar` 可能会直接报 `Unused / PassThru / Volume` 警告甚至卡住门禁。
2. **System Instruction 的影响**：
   - 强调 `Prefer edit file using replace_file_content`、最小化改动、严格按用户请求执行。
3. **如何改善**：
   - 这种风格其实是**好事**（避免像早期 Agent 那样写出大量脱离主干、难以合并的“过度设计孤岛代码”）。
   - 你只需要在给 Agent 的 **Instruction 中明确写出验收清单（Checklist）与边界条件**（例如明确要求“同时实现 `TunnelReliabilityLayer` 端到端重传与 `SignalTokenCodec` 打包，并补充对应的 Loopback 单元测试”），Agent 就会严格且高质量地全部补齐。

---

### 6. UI 架构确认与是否需要“全仓大扫除”？

1. **UI 架构确认 — 你的理解 100% 准确**：
   - `src/ui-core/primitives/`（`SliderControl`、`SegmentedSwitch`、`ToggleSwitch`、`Button` 等）与 `compounds/`（`SettingSliderRow`、`SettingSegmentedRow` 等）均为纯展示型封装组件。
   - `recipes/` 与 `panels/` 直接通过 Zustand `stores/` 和 `config/` 绑定业务变量（并支持 `stateOverride` 供 `ui-tuner` 矩阵快照测试），通过 `src/ui-system/` 的 `syncBridge` 与引擎通信，**不直接导入引擎单例**。
   - 因此，**逻辑/网络开发只需在 `recipes/` 或 `panels/` 中用现成 `Setting*Row` 关联好 Store 字段即可**；后续 UI 团队美化样式时只需修改 `primitives/`、`compounds/` 或布局，**完全不会破坏数据绑定**。
2. **是否需要全仓大扫除？— 不需要！**
   - 刚刚在 `alpha` (`8d19408`) 实测 `pnpm run radar:audit` 结果：
     - **1387/1387 文件 100% TypeScript，`tsc --noEmit` 零报错**；
     - **0 循环依赖、0 值重导出、0 向上导入、0 多语句行**；
     - 全仓所有引擎子系统仅标记 **5 个轻微耦合目标**（如 `TelestratorOverlaySubsystem.ts` 481L / 6 个回调），全部在可控绿灯范围内。
   - **结论**：当前代码库架构健康度极高，**严禁做破坏性的全仓大扫除**。仅做顺手的小清理即可（如将 `TelestratorSettingsPanel.tsx` 重命名为 `TelestratorSettingsFloatingWindow.tsx`，以及清理 `1051b10` 等废弃远端实验分支）。

---

## 二、下一阶段开发总控 Instruction（可直接下发给执行 Agent）

> 以下 Instruction 已针对当前 `alpha` (`8d19408`) 与 `24fc4a4` / `db751d8` / `9a5d063` 的真实代码差异完成校准，可直接用于指导单体集成或分阶段 PR 落地。

```markdown
# RocketCompanion 网络基础设施与联机套件集成 Instruction

## 0. 基准分支与工作流纪律
- **基准分支**：`alpha` (`8d19408`)，新建工作分支 `z/agy/rc/net-infra-and-multiplayer-suite`。
- **严禁直接 `git merge`**：禁止直接 merge `z/agy/rc/track-b-voice-fix`（会回退 PR #14/#18/#19）或直接覆盖合并 `multiplayer-suite`。必须基于 `alpha` 按模块精准移植与集成。
- **门禁要求**：每次阶段性改动后必须通过 `pnpm run format:check`、`pnpm run check`（`radar` + `tsc --noEmit`）以及 `cargo check`（在 `src-tauri/`）。

## 1. 底层网络基础设施统一（Phase 1 — 必须最先锁定）
1. **整合 `24fc4a4` 与 `db751d8` 的各自优势**：
   - **本地桥接层采用 `24fc4a4`**：保留 `src-tauri/src/tunnel_upnp.rs`、`tunnel_bridge.rs`（`wtransport` QUIC + `tokio-tungstenite` WS 显式降级）与 `src/network/transport/tunnel/LocalBridgeClient.ts`（确保 Web Worker 兼容），保留 `tunnel_manager.rs` 的单 Socket 多 Peer 路由与 `RCPH`/`RCPA` 过滤。
   - **公网 UDP 可靠性层采用 `db751d8`**：移植 `src/network/transport/tunnel/TunnelReliabilityLayer.ts`（序列号、ACK、重传、MTU 分片重组），确保 `RELIABLE_CONTROL`、大图同步（`TELESTRATOR_SYNC`）与 `STATE_PAUSED_ACK` 在公网 UDP 下不丢包不乱序。
   - **四服务器信令零改动打包采用 `db751d8`**：将 `tunnelEndpoint?: TunnelEndpointDescriptor` 写入 `SignalTokenCodec.ts` 与 `HandshakeSession.ts`，确保即使 WebRTC ICE 失败也能通过信令完成 UDP Tunnel 地址交换（不要仅依赖 DataChannel 内发 `tunnel_endpoint_advertise`）。
2. **传输优先级策略（可配置，默认 `webrtc-first`）**：
   - 在 `DataChannelTransport.ts` / `MultiplayerSessionManager.ts` 中实现传输策略：`webrtc-first`（默认：WebRTC P2P 直连可用时走 DataChannel，失败/超时/断线时自动切换 Rust UPnP UDP Tunnel）与 `tunnel-first`（可选诊断/低延迟模式）。

## 2. 四大业务模块按序集成（Phase 2 — 依赖 Phase 1 接口）
- **模块 A（Voice Audio 整合）**：从 `z/agy/rc/track-b-voice-fix` 精准移植 `src/audio/voice/*`、`voiceChatConfig.ts`、`voiceChatStore.ts`、`runtimeVoiceBridge.ts`、`VoiceChatSettingsSection.tsx`。修复 `voiceAudioGraph.ts` 统一走 `VoiceMasterOutputBus` 播放远端流、移除静默 `.catch(() => {})`。绑定生命周期至 `EstablishedSessionManager` / `RoomReplicationBus`，支持 `voiceChatMode: 'open' | 'team_only' | 'off'`（暂不引入 RNNoise WASM）。
- **模块 B（Multiplayer Training Kernel）**：基于 `RoomActivityOrchestrator` + `ActivityKernel` 新增 `MULTIPLAYER_TRAINING`（隐藏计分板、保留 Goal Replay）。实现 4 档 Boost 策略（`limited` | `limit_when_aerial` [默认] | `unlimited` | `custom`，Advanced 页仅实现 Per-Player Boost 覆盖）。实现 Ghost Mode（Client 默认 OFF，开启时走 Host Relay 多球分发）。
- **模块 C（Spectating 升级与全局状态广播）**：拆分“对局实时数据包”与“全局状态广播包”（含设置页路由、打开/最小化的浮动窗体清单，不含屏幕坐标、Notification 与 Telemetry）。支持 `spectate_state_only` vs `spectate_all`（只读锁定本地菜单），基于 Replay 模型支持 Play/Pause ACK、暂停期 1–10Hz 降频心跳与 Telestrator 双向绘图及权限管理。
- **模块 D（Kick & Leave 双向信令）**：修复 `OnlineRemotePeerVehicleSynchronizer.removeRemotePlayer(carIndex)` 先发送可靠 `kick` 控制帧再清理本地状态并关闭对应通道；Client 主动退出先发可靠 `leave` 帧再销毁连接。

## 3. UI 规范与测试矩阵（Phase 3 — 可与 Phase 2 并行推进测试框架）
- **UI 规范**：严格遵守 `src/ui-core/AGENTS.md`，使用 `SettingSegmentedRow` / `SettingSliderRow` / `SettingToggleRow` 等现成原语绑定 Store，并为所有新增面板/Section 提供 `stateOverride` 以兼容 `ui-tuner`。顺手将 `TelestratorSettingsPanel.tsx` 重命名为 `TelestratorSettingsFloatingWindow.tsx`。
- **自动化测试矩阵（无需独立 `headless=1` 构建）**：基于 `scripts/runMultiplayerRoomRepro.mjs`（`reproBridgeSignaling=1`）与 `MultiClientLoopbackSimulator.ts`，覆盖：
  1. 1v1 ~ 3v3（真人 Ducky AI 接管 + 系统 Bot Auto-fill 全组合）；
  2. `webrtc-first` 正常直连与 WebRTC 阻断后自动降级 Rust UPnP Tunnel；
  3. Training 4 档 Boost 与 Ghost Relay；
  4. Voice `open` / `team_only` / `off` 路由与跨 Activity 热切换不掉线；
  5. Spectate `state_only` vs `all`、Play/Pause ACK 与 Telestrator `stroke.hash` 多端一致性校验；
  6. Kick / Leave 双向信令清理。
```
