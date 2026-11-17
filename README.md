# Phase 3 完成报告：可靠 Pause/Resume ACK 与低频 Presence Heartbeat

- **仓库与分支**: `Antigravi-ty/RocketCompanion` (`z/agy/rc/multiplayer-recheck`)
- **Verified Bot Commit**: [`0caec73084bd6c9a394a1976327cdf7e5ac40254`](https://github.com/Antigravi-ty/RocketCompanion/commit/0caec73084bd6c9a394a1976327cdf7e5ac40254) (`signature.isValid = true`, `state = "VALID"`)

---

## 1. Phase 1–2 完成情况核查与额外加强

在开始 Phase 3 前，我们对 `origin/z/agy/rc/multiplayer-recheck`（`a4ab863` Phase 1、`24c34f6` Phase 2）进行了实测与代码扫描：
- **Phase 1（禁用 Host 自回滚 + Pure CSR 广播路径）**：已基本完成。`Freeplay`、`Exhibition 3v3`、`Static Training Pack` 与 `3v3 .RCReplay` 均可同步 6 车拓扑与状态。
- **Phase 2（`SpectateInterpolationBuffer` 逆向延迟抖动缓冲）**：已完成并集成在 `SimulationStage` / `simulationNetworkReconcilePipeline` 中，支持动态 `interpDelayTicks`（2–10 ticks）与跨拓扑/瞬移自动重置。
- **额外发现并补齐的缺口**：
  1. **`SimulationStage` / `HostBroadcastPublisher` 在暂停时仍以 60Hz 发送物理快照**：原 `HostBroadcastPublisher.publishIfReady` 仅检查了 `flags.shouldHoldAccumulator`，而 `StandaloneReplayViewerController` 暂停时 `shouldHoldAccumulator` 为 `false`。我们暴露了 `flags.isReplayPaused` 与 `isSpectatePlaybackPaused()`，并在 `HostBroadcastPublisher` 中实现了**暂停期间 0Hz 物理快照静默**（仅在暂停状态下切帧/Seek 导致 tick 变化时发送单帧，恢复播放时立即推流）。
  2. **后加入 Peer 的暂停状态同步**：当 Client 在 Host 已处于暂停状态后加入房间，`HostSessionController` 在 `onConnected` 时会立即下发当前 `PAUSE_STATE_SYNC`，确保新接入端同样进入暂停同步状态。

---

## 2. Phase 3 核心实现

1. **协议扩展 (`src/network/protocol/GlobalWorkspaceProtocol.ts`)**
   - 在 `GlobalWorkspaceBroadcastPayload` 中增加 `pauseSeq` 字段。
   - 新增 `PauseStateSyncPayload`（`pause_state_sync`）、`PauseStateAckPayload`（`pause_state_ack`）与 `PresencePingPayload`（`presence_ping`）及对应的 `sanitizePauseStateSync`、`sanitizePauseStateAck`、`computeCompactSnapshotHash` 校验工具。

2. **传输层与 BroadcastChannel 多标签页回环 (`dataChannelControlFraming.ts`, `DataChannelTransport.ts`, `P2PWebRTCChannel.ts`, `webrtcHandshakePipeline.ts`)**
   - 支持 `pause_state_sync`、`pause_state_ack`、`presence_ping` 控制帧的解析、分发与发送，并在 `P2PWebRTCChannel` 增加 `pendingPauseStateSync` 缓冲队列以防早到丢包。

3. **Host 端状态机 (`HostPauseAckController.ts`, `hostPauseAckBridge.ts`, `HostSessionController.ts`)**
   - 实现三态状态机：`playing` -> `awaitingPauseAck` -> `pausedIdle`。
   - **Pause**：`pauseSeq` 自增，停止物理快照流（`shouldStreamPhysicsSnapshots() === false`），向所有已连接 peer 下发 `PAUSE_STATE_SYNC`。
   - **100ms 重试与 5s 超时保护**：`awaitingPauseAck` 期间每 `100ms` 向未 ACK 的 `peerId` 重发 `PAUSE_STATE_SYNC`；若超过 `5000ms` 仍有死连接未响应，触发 `onPauseAckTimeout` 告警并转入 `pausedIdle`。
   - **0Hz 物理流 + 1–2Hz 保活心跳**：所有 peer ACK 后进入 `pausedIdle`，保持 0Hz 物理数据流，按 `pauseHeartbeatHz`（默认 2Hz / 500ms）发送轻量 `presence_ping`。
   - **Resume**：`pauseSeq` 自增，恢复 `playing` 状态并立即恢复物理流。

4. **Client 端 ACK 闭环 (`PredictionReconciler.ts`, `RuntimePostprocessingAndSessionBootstrap.ts`, `spectateWorkspaceStore.ts`, `runtimeSpectateWorkspaceBridge.ts`)**
   - 在 `PredictionReconciler` 新增 `sendPauseAck(...)` 方法，并在 `reconcile()` 与 `ch.onPauseStateSync` 中自动更新 `spectateWorkspaceStore`（`playbackPaused`、`pausedAtTick`、`pauseSeq`）并回传 `PAUSE_STATE_ACK`。

5. **架构门禁合规拆分**
   - 为满足 Radar 18KB/800 行硬性体积上限，将控制帧解析/分发拆至 `dataChannelControlFraming.ts`，传输监听类型拆至 `dataChannelTransportTypes.ts`，Host 暂停 ACK 桥接逻辑拆至 `hostPauseAckBridge.ts`。

---

## 3. 已修正问题与已解决 Bug

- **已解决 Bug B（`hasPauseAckProtocol=false` / 缺少 Pause ACK 与重试机制）**：
  - `repro:dev-sync-bugs` Scenario 3（`Real 3v3 RCReplay Load & Playback + Pause ACK`）已由 `REPRODUCED_BUG` 转为 **`PASS`**（`hostCars=6, clientCars=6, ballDist=0, hasPauseAckProtocol=true`）。
  - 暂停后物理快照流降为 **0Hz**，并由 **1–2Hz `presence_ping`** 维持在线保活。
- **已解决关联隐藏缺口**：
  - 修复了 Replay Viewer 暂停时 `HostBroadcastPublisher` 仍以 60Hz 广播的漏洞。
  - 补齐了暂停期间新加入 Peer 的状态同步。
- **仍属 Phase 4 范围的待办项**（未在本阶段改动）：
  - `repro:dev-sync-bugs` Scenario 5（Multi-Client 2+ Clients）与 Scenario 6（Client Input Handover `clientInputEnabled=true`）按原计划归属 Phase 4。

---

## 4. 验证门禁结果

| 检查项 | 命令 | 结果 |
| --- | --- | --- |
| 架构 Radar + TS 严格检查 | `pnpm run check` | **PASS**（1420/1420 TS 文件，0 循环依赖，0 越界导入，0 体积超限） |
| 联机单元测试矩阵 | `pnpm run test:multiplayer-suite` | **PASS**（17/17 全部通过，含新增 Test 17：Pause/Resume ACK、100ms 重试、5s 超时与 2Hz 心跳） |
| Dev Sync Room 全链路 E2E | `pnpm run test:dev-room-sync-e2e` | **PASS**（9/9 阶段全部通过，`runtime_error=0`） |
| Headless Bug 复现套件 | `pnpm run repro:dev-sync-bugs -- --report-only` | **4/6 PASS**（Scenarios 1, 2, 3, 4 全部 PASS；仅 Phase 4 的 Scenarios 5, 6 待后续实现） |
