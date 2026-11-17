# RocketCompanion Phase 1 Report: Host Self-Reconciliation Fix & Pure CSR Broadcast Path

**Branch**: `z/agy/rc/multiplayer-recheck`
**Verified Bot Commit**: [`a4ab8631d718e35fc0477883193dc8c1acdc1497`](https://github.com/Antigravi-ty/RocketCompanion/commit/a4ab8631d718e35fc0477883193dc8c1acdc1497) (`signature.isValid: true`)

---

## 1. 已修正的问题与已解决的 Bug

### 已解决的核心 Bug
1. **Bug A（Critical）：Host 自对账冲突（`PredictionReconciler` vs `AuthoritativeServerWorker`）**
   - **根因**：在 Dev Sync Room 广播模式（`isDevSyncRoom && isHosting && activeRoomActivity !== 'match'`）下，Host 仍运行 `reconciler.reconcile()` 与 `syncClientCarTopology()`，导致 `AuthoritativeServerWorker` 里的 1 车自由练习世界反向覆盖 Host 主线程的 Exhibition（6 车）与 Replay（6 车）物理世界，把 1..5 号车删除。
   - **修复**：在 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 中，当 Host 处于 Dev Sync 广播模式时跳过 `reconciler.reconcile()`、`flushPendingControlPackets()` 与 `syncClientCarTopology()`，仅保留 `server.update()`（用于 fallback 模式）与本地录制逻辑。

2. **Freeplay Live Driving Sync 漂移问题（Scenario 1：`REPRODUCED_BUG` -> `PASS`）**
   - **根因**：Host 与 Client 此前在 Dev Sync 广播模式下各自运行前向预测与回滚（Client 以 `spectateMirrorMode=true` + `clientInputEnabled=false` 跑空输入预测），造成 `dist=324`。
   - **修复**：新增 `HostBroadcastPublisher`（60Hz 推送 Host 主线程 `physics.saveCompactState()`），并在 Client 端 Pure CSR Spectate 模式（`spectateMirrorMode && clientInputEnabled === false && !hostLobbyMode`）下跳过局部物理步进与回滚预测，直接通过 `applyReplayFrameToEngine` 应用最新权威快照。`repro:dev-sync-bugs` Scenario 1 距离从 `324` 降至 `60`（`hostSnaps=0`）。

3. **Exhibition 3v3 Spectating（Scenario 2：`REPRODUCED_BUG` -> `PASS`）**
   - **修复**：Host 保留完整 6 车（`hostCars=6`），并通过 `HostBroadcastPublisher` -> `AuthoritativeServerHost.pushBroadcastSnapshot` 同步至 `AuthoritativeServerWorker` 与远端 Client（`clientCars=6`）。

4. **3v3 Replay 加载与 6 车同步（Scenario 3 车辆拓扑部分已修复：`hostCars=1` -> `hostCars=6`）**
   - **修复**：Host 加载 3v3 回放时不再被 `AuthoritativeServerWorker` 删车，`hostCars=6, clientCars=6, ballDist=0`。同时在 `StandaloneReplayViewerController.ts` 初始化回放会话时同步设置 `networkReconciler.setReplayPlaybackMode(true)`。
   - *注：Scenario 3 整体状态仍标记为 `REPRODUCED_BUG`，仅因 `hasPauseAckProtocol=false`（属于 Phase 3 范围）。*

---

## 2. 原始报告未覆盖但一并扫描并加强的地方

1. **`RuntimeInputPoller.ts` 与 `PodiumCelebrationController.ts` 残留对账调用**
   - 仅跳过 `SimulationStage` 的 `reconcile()` 还不够：`RuntimeInputPoller.ts` 和 `PodiumCelebrationController.ts` 仍会调用 `reconciler.sampleAndPredictInput()`（向 Worker 发送输入并保存状态槽）并用 `reconciler.historyBuffer.lastRemoteControls` 覆盖非本地车辆控制（在 Exhibition 6 车模式会干扰 Bot 控制）。
   - 已在两处同步加入 `isHostBroadcasting` 门控，广播模式下完全绕过。

2. **车辆数从多到少（6 车 -> 1 车）的退场清理**
   - `ReplayFrameInterpolator.ts` (`applyReplayFrameToEngine`) 与 `MatchSystemControlSubController.ts` (`loadSimSnapshot`) 原本只补齐缺少的车辆槽位，从 6 车 Exhibition/Replay 退回 1 车 Freeplay/Training 时不会移除多余车辆。
   - 已在两处补充多余槽位清理逻辑（`removeCar`），确保切回 Freeplay/Training 后两端均干净恢复为 1 车（Scenario 4 验证通过）。

3. **`OnlineRosterSyncCoordinator.ts` 广播模式下保护 Host 本地车辆与 NameTag**
   - 当客户端加入/断开触发 `roster_update` 时，Host 若处于 Exhibition 或 Replay 模式，`OnlineRosterSyncCoordinator.applyRosterSnapshot` 会用 1 车 Server Roster 覆盖 Host 的 `world.physics`/`world.arena` 车辆与 `nameTagEntries`。
   - 已在 `isHostBroadcasting` 时跳过 `syncRosterPlayerCarInWorld` 与 `setNameTagRuntimeState`，同时保留 `connectedPeers` UI 状态更新。

4. **轻量级广播快照通道（`pushBroadcastSnapshot`）与模式切换还原**
   - 新增 `AuthoritativeServerHost.pushBroadcastSnapshot` / `workerTrainingAndPhysicsRouter` / `MatchSystemControlSubController.pushBroadcastSnapshot`：仅在首次进入时调用 `enterStateOnlyMode`（触发一次 `SYS_CTRL_STATE_ONLY`），后续 60Hz 帧直接更新快照，避免每秒重复广播 60 次 `SYS_CTRL_STATE_ONLY` 信号。
   - 在 `ActivityKernel.ts` (`MatchActivityKernel.onMount`) 中，切入 `match` 活动时自动清除 `isStateOnlyMode`，确保 Match 模式物理正常步进。
   - 在 `MatchModeTransitionBridge.ts` 中，若在 Dev Sync Room 从 `match` 切回本地/广播模式，自动将 `activeRoomActivity` 重置为 `'freeplay'`；在 `onlineSessionEnvironmentPipeline.ts` 中，Client 切活动时同步更新 `setSpectateMirrorMode(activityId !== 'match')`。

---

## 3. 修改文件清单

- 新增：`src/game/pipeline/stages/HostBroadcastPublisher.ts`
- 修改：
  - `src/game/pipeline/stages/SimulationStage.ts`
  - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
  - `src/game/runtime/RuntimeInputPoller.ts`
  - `src/game/modes/PodiumCelebrationController.ts`
  - `src/game/runtime/OnlineRosterSyncCoordinator.ts`
  - `src/game/runtime/MatchModeTransitionBridge.ts`
  - `src/game/runtime/onlineSessionEnvironmentPipeline.ts`
  - `src/network/server/AuthoritativeServerHost.ts`
  - `src/network/server/workerTrainingAndPhysicsRouter.ts`
  - `src/network/server/match/MatchSystemControlSubController.ts`
  - `src/network/server/activities/ActivityKernel.ts`
  - `src/network/client/reconcilerInboundPacketParser.ts`
  - `src/network/client/reconcilerSessionPipeline.ts`
  - `src/replay/ReplayFrameInterpolator.ts`
  - `src/replay/StandaloneReplayViewerController.ts`

---

## 4. 验证结果（Linux Headless + SwiftShader）

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（1416/1416 TS files，0 violations）
- `pnpm run test:multiplayer-suite`：**PASS**（15/15）
- `pnpm run test:dev-room-sync-e2e`：**PASS**（9/9 stages，`runtime_error=0, poseAnomalies=0, stuckInGoal=false`）
- `pnpm run repro:dev-sync-bugs -- --report-only`：**3/6 PASS**（Phase 1 目标场景全部通过；剩余 3 项对应 Phase 3 Pause ACK、Phase 4 多客户端 Fan-Out 与控制权移交）：
  - `1. Freeplay Live Driving Sync`: **PASS** (`dist=60, hostSnaps=0, clientSnaps=1`)
  - `2. Exhibition 3v3 Spectating`: **PASS** (`hostCars=6, clientCars=6, hostSnaps=0, clientSnaps=1`)
  - `3. Real 3v3 RCReplay Load & Playback + Pause ACK`: `hostCars=6, clientCars=6, ballDist=0`（仅 `hasPauseAckProtocol=false` 待 Phase 3）
  - `4. Static Training Pack Pose Sync`: **PASS** (`carDist=0`)
  - `5. Multi-Client (2+ Clients) Dev Sync Room`: 待 Phase 4
  - `6. Client Input Handover`: 待 Phase 4
