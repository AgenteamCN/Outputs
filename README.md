# RocketCompanion 多人联机 P0-P2 接力开发与真实浏览器实测检查报告

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **基准 Commit**: `dfbcf57340873c68cfe6e36f74e1efe79aacb486`
- **工作分支**: `z/agy/rc/multiplayer-p0-p1-relays`
- **已签名 Verified Commit**: [`af528a850a84466d6a263ba92f8bab974cc052a1`](https://github.com/Antigravi-ty/RocketCompanion/commit/af528a850a84466d6a263ba92f8bab974cc052a1) (`signature.isValid: true`)

---

## 一、核心结论：距离 Real Device Test 还差多少？

**结论：`af528a8` 已打通前序报告中的全部 3 个 P0 级断层及 P1/P2 项，真实多标签页 Headless Chrome 联机实测全绿通过，已具备进入 Real Device Test（真机公网/跨路由器 UPnP & NAT 打洞验证）的条件。**

- **P0-1 (`repro:multiplayer-room` Client 断连 & Fail-Fast)**：已根治。Client 不再在 `~391ms` 被公共 MQTT fallback 报错提前 `abortJoin()`，且脚本通过 CDP `PUT /json/new` 稳定拉起 Client 标签页。实测 Host 与 Client 均在 70.8s 内完成全流程并进入 `match/kickoff`，`bw=711.78kbps`、`rtt=139ms`、`detectedPoseAnomalies=0`，无 Watchdog 超时。
- **P0-2 (`tunnelEndpoint` 信令交换 & 自动预建连)**：已打通 Offer/Answer Token 编解码与 `HandshakeSession` -> `P2PWebRTCChannel` -> `MultiplayerSessionManager` 生命周期，收到对端 `tunnelEndpoint` 后立即预连 `sessionTunnel.connectPeer()` 并挂载 `attachTunnelTransport()`。
- **P0-3 (`ROOM_ACTIVITY_SWITCH` 端到端广播与消费)**：已打通 Host (`OnlineMatchControlBridge.switchRoomActivity` -> `HostSessionController.broadcastRoomActivitySwitch`) 到 Client (`DataChannelTransport.drainTransportEvents()` -> `PredictionReconciler.reconcile()` -> `OnlineSessionCoordinator` / `onlineSessionEnvironmentPipeline.applyClientRoomActivitySwitch`) 完整链路。

---

## 二、改动明细与模块化拆分（P0 ~ P2）

### 1. P0-1：修复 `repro:multiplayer-room` Client 建链与 Fail-Fast 硬断言
- **`src/network/transport/ClientJoinNegotiationSubController.ts`**:
  - 修复根因：默认无外部信令配置时 `signalingHandler.connect(timeoutMs)` 会快速 reject 并触发 `finishFailure` -> `abortJoin()`，直接销毁 Client `P2PWebRTCChannel`。现当存在 `BroadcastChannel` 且 `ch` 已初始化时，改为调度 `3500ms` 宽限定时器等待本地跨标签页握手完成。
- **`scripts/runMultiplayerRoomRepro.mjs`** & **`src/game/scenario/multiplayerRoomReproHostFlowPipeline.ts`**:
  - Chrome 启动增加 `--remote-debugging-port=0`，监听 `hostOnlineWarmup completed` 后通过 CDP `PUT http://127.0.0.1:<port>/json/new?...` 拉起 Client 标签页（并传入 `reproNoWindowOpen=1` 避免 `window.open` 在无头模式下静默失效）。
  - 在 `main()` 末尾增加 Fail-Fast 硬断言：校验 `autoTerminatedByWatchdog === false`、`hostSummary.enteredMatch === true`、`clientSummary.enteredMatch === true && finalMatchMode === 'match'`、`realizedThroughputKbps > 0`，并在 `--glitch-check` 下断言 `runtime_error` 为 0；失败时打印双向 `handshakeDiagnostics` 并以退出码 `1` 退出。

### 2. P0-2：`tunnelEndpoint` 写入信令 Token 并提前触发 Rust UDP Tunnel 建连
- **`src/network/transport/webrtc/webrtcHandshakePipeline.ts`** & **`HandshakeSession.ts`** & **`P2PWebRTCChannel.ts`**:
  - `buildHostOfferResult` 与 `buildClientAnswerResult` 支持打包 `tunnelEndpoint`；`acceptOfferAndCreateAnswer` 与 `acceptAnswerToken` 解码后若存在 `tunnelEndpoint`，立即回调 `onRemoteTunnelEndpoint`。
- **`src/network/transport/HostSessionController.ts`** & **`ClientSessionController.ts`** & **`ClientJoinNegotiationSubController.ts`** & **`MultiplayerSessionManager.ts`**:
  - Host 在 `initializeChannel()` 之前打开本地 `sessionTunnel` 以获取 `localTunnelDescriptor`；Host 与 Client 在收到对端 `tunnelEndpoint` 后立即调用 `connectRemoteSessionTunnelEndpoint()` 完成 `sessionTunnel.connectPeer()` 与 `channel.attachTunnelTransport()`。

### 3. P0-3：打通 `ROOM_ACTIVITY_SWITCH` 端到端广播与客户端状态同步
- **Host 广播**：`HostSessionController.ts` 新增 `broadcastRoomActivitySwitch()`，`MultiplayerSessionManager.ts` 对外暴露同名方法；`OnlineMatchControlBridge.ts` 与 `runtimeDomainPortTypes.ts` 新增 `switchRoomActivity()`，同时驱动本地 `authoritativeServer.switchActivity` 与远端 `broadcastRoomActivitySwitch`。
- **Client 消费**：`P2PWebRTCChannel.ts` 暴露 `drainTransportEvents`；`PredictionReconciler.ts` 在 `reconcile()` 中排空 `channel.drainTransportEvents()` 并触发 `onRoomActivitySwitch`；`OnlineSessionCoordinator.ts` 与 `onlineSessionEnvironmentPipeline.ts` 实现 `applyClientRoomActivitySwitch()`，同步切换 `replayPlaybackMode`、`modeManager.setMode(...)` 与 `useAppStore`。

### 4. P1-1 & P1-2：传输层诊断导出、断连一键导出通知与 `--chaos` 活动切换实测
- **`src/network/transport/webrtc/DataChannelTransport.ts`**: 新增 `getTransportPath()` 委托 `resolveActiveTransportPath()`。
- **`src/game/runtime/DiagnosticReplayBufferBridge.ts`** & **`src/ui-system/core/diagnosticDumpExportPipeline.ts`**:
  - `RuntimeReplayPort` 新增 `getTransportDiagnostics()`，将 `transportPath`、`iceConnectionState`、`upnpMapped`、`tunnelStats`、`framerStats` 写入 `diagnostic-manifest.json`。
  - 新增 `notifyTransportDiagnosticDumpReady()`，在 Tunnel 降级、重传尖峰或异常断连时通过 `pushQuickNotification()` 弹出带一键 `Export Dump` 按钮的快捷通知。
- **`scripts/runMultiplayerRoomRepro.mjs`** & **`multiplayerRoomReproHostFlowPipeline.ts`**: 支持 `--chaos` (`reproChaos=1`)，在实测流程中执行 `multiplayer_training -> replay -> freeplay` 活动切换遍历。

### 5. P2：大文件解耦（保持 Radar 18KB 硬限制下安全余量）
- 拆分 **`src/network/transport/webrtc/dataChannelProbePipeline.ts`**（146 行）：将 `DataChannelTransport.ts` 中的模拟发包、Opportunistic Ping/Pong、16B 稀疏帧与自适应 Pacing 逻辑抽离，使 `DataChannelTransport.ts` 降至 **16.9 KB / 500 lines**。
- 拆分 **`src/network/transport/sessionTunnelLifecyclePipeline.ts`**（129 行）：将 `MultiplayerSessionManager.ts` 中的 Rust UDP Tunnel 生命周期与端点交换逻辑抽离，使 `MultiplayerSessionManager.ts` 降至 **16.4 KB / 490 lines**。

---

## 三、完整验证结果（全部实测通过）

| 检查项 | 命令 | 结果 | 关键指标 |
|---|---|---|---|
| 架构门禁 + 类型检查 | `pnpm run check` | **PASS** | 1409/1409 TS 文件 (100%)，`tsc --noEmit` 0 错误，0 边界/循环依赖违规 |
| Node 多人矩阵单测 | `pnpm run test:multiplayer-suite` | **PASS** | 6/6 项全部通过（Tunnel/Kick/Voice/Training/Spectate/Telestrator/6-Car） |
| Node 混沌流测试 | `pnpm run repro:multiplayer-chaos` | **PASS** | 3/3 项全部通过（Cancel/Re-Host/Re-Join、6 活动遍历、18 种人机组合） |
| 生产构建 | `pnpm run build` | **PASS** | Vite 3786 modules 构建成功（28.25s） |
| 真实浏览器 E2E 联机实测 | `pnpm run repro:multiplayer-room -- --mode=2p --probes=lobby-probes,network-latency-hud --max-duration-ms=90000 --chaos --glitch-check` | **PASS** | 耗时 `70832ms`，`Watchdog=false`；Host `match/kickoff` (`bw=500.76kbps`, `rtt=135ms`, `snaps=0`, `anomalies=0`)；Client `match/kickoff` (`isCrossTab=true`, `bw=711.78kbps`, `rtt=139ms`, `pings=2/4`, `visibleCars=[0,1]`, `anomalies=0`)；`Chaos activity switch traversal completed` |

> 注：本沙盒未安装 `gcc`/GTK 系统库，Rust/Tauri 桌面端 UPnP IGD 映射与跨公网对称 NAT UDP 打洞仍需在具备完整桌面工具链的真机环境中执行最终 Real Device Test 验证。
