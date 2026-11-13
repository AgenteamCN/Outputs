# RocketCompanion Multiplayer Suite Implementation Report

- **Branch**: `z/agy/rc/multiplayer-suite`
- **Base Commit**: `24fc4a4e32a5546c80a99d194088c910f38e7dcc`
- **Verified Bot Commit**: [`9a5d06376e121a8296ef4b12ed5a42cdfbd1706a`](https://github.com/Antigravi-ty/RocketCompanion/commit/9a5d06376e121a8296ef4b12ed5a42cdfbd1706a) (`verified: true`)

---

## 1. Tauri P2P Tunnel / WebTransport 主链路接入

- **`src/network/transport/webrtc/DataChannelTransport.ts` & `dataChannelFramingPipeline.ts`**:
  - 新增 `attachTunnelTransport`、`detachTunnelTransport`、`hasActiveTunnel`、`transmitViaTunnelOrDataChannel` 与 UTF-8 JSON 二进制帧解码。
  - 将高频状态帧（`reliable = false`）与可靠控制帧（`reliable = true`）优先路由至 `TunnelTransport`；保留 `tunnel_endpoint_advertise` 通过 WebRTC DataChannel / Signaling 通道交换以完成端点发现与打洞 bootstrap。
  - 将入站控制帧分发逻辑抽取至 `dataChannelFramingPipeline.ts`（`dispatchInboundControlFrame`），保证 `DataChannelTransport.ts` 保持在 17.3 KB（低于 18 KB 硬红线）。
- **`src/network/transport/P2PWebRTCChannel.ts` & `MultiplayerSessionManager.ts`**:
  - 接入 `TunnelFactory.create()`、`tunnel_endpoint_advertise` 双向端点交换、`onTunnelEndpointAdvertise` 与隧道包路由。
  - 语音轨 `MediaStreamTrack` 继续通过 WebRTC `RTCPeerConnection` 混合模式承载。

## 2. 模块 A：Audio / WebRTC 语音整合

- 从 `origin/z/agy/rc/track-b-voice-fix` 按文件精准移植并重构（未执行 `git merge`，避免回退 PR #14 / #18 / #19）：
  - `src/audio/voice/IVoiceAudioBridge.ts`
  - `src/audio/voice/voiceAudioGraph.ts`（统一经 `VoiceMasterOutputBus` 播放远端流，移除静默 `.catch(() => {})` 与 `hiddenAudioEl` 兜底，补充结构化诊断状态）
  - `src/audio/voice/voiceRenegotiationPipeline.ts`（从 `WebRtcNativeVoiceBridge.ts` 拆出独立协商管线）
  - `src/audio/voice/WebRtcNativeVoiceBridge.ts`
  - `src/ui-core/config/voiceChatConfig.ts`、`src/ui-core/stores/voiceChatStore.ts`、`src/ui-system/core/runtimeVoiceBridge.ts`
  - `src/ui-core/recipes/settings/components/VoiceChatSettingsSection.tsx`、`AudioTab.tsx`
- 新增房间语音模式 `voiceChatMode: 'open' | 'team_only' | 'off'`：
  - `open`：全员互通。
  - `team_only`：对局中按 Peer `team` 动态 Gate 发送与接收（`sender.replaceTrack` + 远端增益归零）；**Lobby 阶段自动启用全员互通覆盖**。
  - `off`：禁用麦克风采集与远端播放，并向 UI 返回状态提示。

## 3. 模块 B：Multiplayer Training（`MULTIPLAYER_TRAINING` ActivityKernel）

- 新增 `src/network/server/activities/MultiplayerTrainingPolicy.ts` 与 `MultiplayerTrainingActivityKernel`（`src/network/server/activities/ActivityKernel.ts`）：
  - 注册 `ACTIVITY_IDS.MULTIPLAYER_TRAINING`（`ReplicationBusProtocol.ts`），默认 `hideScoreboard: true`，保留进球后正常进入 Cinematic Goal Replay。
  - **4 档 Boost 策略**（`limited | limit_when_aerial | unlimited | custom`，默认 `limit_when_aerial`）：
    - 地面状态（`sim.getCarStateView(carIndex).isOnGround`）不消耗 Boost 且允许拾取 Boost Pad 补满；离地后切换为真实消耗。
    - `PhysicsEngineManager.ts` 在物理步进前调用 `applyTrainingBoostPolicyToSim` 实时应用策略。
    - `custom` 模式支持 Per-Player Boost 覆盖。
  - **Ghost 模式与 Host Relay**：
    - `Layer3HostTrainingRecipe.tsx` 中 Client 端 Ghost 默认 OFF（`manual`），提供 Segmented Control 与 Custom Per-Player 列表。
    - `GhostBallRelayCoordinator` 支持 Host 接收各端球体状态并按需广播多球快照。

## 4. 模块 C：Spectating 升级与全局状态广播包

- 新增 `src/network/protocol/GlobalWorkspaceProtocol.ts` 与 `src/ui-core/stores/spectateWorkspaceStore.ts`：
  - 拆分“对局实时数据包”与“全局状态广播包”（仅包含 `menuRoute`、`openFloatingWindows` `{ id, isOpen, isMinimized }`、`telestratorPermission` 默认 `'host_only'`、`playbackPaused`、`pausedAtTick`、`pauseHeartbeatHz` 1–10Hz；**不包含**窗体屏幕坐标、Notification 与 Telemetry）。
  - 客户端二选一观战模式：
    - `spectate_state_only`：仅同步主画面状态/输入/Boost。
    - `spectate_mirror`：同步 Host 打开/最小化的浮动窗体清单与设置页路由并本地只读锁定，同时保留 Client 本地窗体坐标（Client 可自由拖拽位置）；退出观战或切回 `spectate_state_only` 后自动恢复本地状态。
  - `RoomActivityOrchestrator.ts` 新增 `buildGlobalWorkspaceBroadcast`、`hasAllPeersPausedAck`、`setTelestratorPermission`、`setPauseHeartbeatHz`。

## 5. 模块 D：Kick & Leave 双向信令修复

- `OnlineRemotePeerVehicleSynchronizer.ts`：`removeRemotePlayer(carIndex)` 在清理本地状态前发送可靠 `peer_kick` 控制帧，解除隧道绑定并调用 `ch.destroy()`。
- `MultiplayerSessionManager.ts`：`leaveSession()` 主动发送可靠 `peer_leave` 控制帧通知 Host 与房间内其余端；Host/Client 分别注册 `onPeerLeave` / `onPeerKick` 回调以立即刷新 Roster 与语音列表。

## 6. 架构清理

- 将 `src/ui-core/panels/dev/TelestratorSettingsPanel.tsx` 重命名为 `TelestratorSettingsFloatingWindow.tsx` 并同步更新 `TelestratorToolbarFloatingWindow.tsx`。

## 7. 验证结果与后续测试提示

- `pnpm check`（`radar` + `tsc --noEmit`）：**PASS**（1404/1404 TS 文件 100% 覆盖、0 循环依赖、0 值重导出、0 向上越层导入、全部文件 ≤ 18 KB / 800 行）。
- `pnpm run test:multiplayer-suite`（`scripts/runMultiplayerSuiteTests.ts`）：**6/6 PASS**：
  1. Tunnel Transport & Kick/Leave Bidirectional Signaling
  2. Voice Routing (`open` | `team_only` + Lobby all-chat | `off`)
  3. Multiplayer Training（4 档 Boost 策略 + `GhostBallRelayCoordinator`）
  4. Spectating & `GlobalWorkspaceBroadcast` + Pause ACK
  5. Telestrator 50-Stroke Stress Test & `TelestratorHashAck`
  6. `MultiClientLoopbackSimulator` Bot Takeover & 6-Car Matrix
- `pnpm build`：**PASS**。
- **提示**：本次自动化测试矩阵基于 Node 无头闭环与模拟通道运行；Tauri 真机环境下的跨网段 UDP 打洞（`TunnelManager`）以及浏览器真实麦克风硬件采集建议在桌面端双开实测时重点回归。
