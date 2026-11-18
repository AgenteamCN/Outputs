# RocketCompanion Multiplayer Sync (Step 1~3) 完成与验证报告

## 一、架构与实现摘要

基于分支 `origin/z/agy/rc/multiplayer-recheck`（已包含 Step 1 提交 `4907669`），已完成 **Step 2**（Pure CSR 服务端信号转发通道、Exhibition 倒计时/事件、Training 球重置、Flip Reset 视觉/音频、Goal Replay & Cinematic Director 镜头复用）与 **Step 3**（Replay Watch Together 元数据/Roster/队伍颜色同步、边沿触发相机同步非覆盖），并补全回归测试套件。最新已签名 Bot 提交：`aba3e1f`（[Commit](https://github.com/Antigravi-ty/RocketCompanion/commit/aba3e1fc0c004d8770648ba2169c7ebd28d3456a)）。

### 1. Step 1 复核：Pure CSR `acknowledgedControls` 与 Coach 接管多端输入广播
- `4907669` 已打通 `ClientAuthorityPublisher` -> `HostInputAuthorityTracker` -> `server.pushBroadcastSnapshot(cf.snapshot, cf.tick, cf.acknowledgedControls)` -> `MatchSystemControlSubController.stepStateOnlyFrameIfActive()` -> 全量 Client 广播链路，以及 `GameCameraSubsystem` 在 Client Spectate 与 Host 观战远端控制者（`isHostSpectatingRemoteAuthority`）时对 `CarNoseOrientationArrows` / `keyboardInputOverlayTelemetry` 的远端按键映射。
- `scripts/runMultiplayerSuiteTests.ts` Section 22 已覆盖 Client Alpha 接管控制期间 Coach（Host）与 Client Bravo 同时实时看到 Alpha 按键输入的全链路回归验证。

### 2. Step 2：AuthoritativeServerHost 主线程信号转发与 Goal Replay 复用（Issues 2, 3, 5）
- **Worker / Server 信号通道**：
  - `src/network/server/match/MatchSystemControlSubController.ts`：新增 `pushBroadcastSignal(signalType, frameId?, param1?, param2?, replaceTypes?)`，在 `isStateOnlyMode` 下通过 `commandQueue.retireSignalTypes` 与 `commandQueue.dispatchSignal` 写入 `SignalEventBuffer` 并随 `stepStateOnlyFrameIfActive` 下发。
  - `src/network/server/AuthoritativeServerHost.ts` & `src/network/server/workerTrainingAndPhysicsRouter.ts`：暴露并路由 `pushBroadcastSignal`。
- **Flip Reset 同步（Issue 3）**：
  - `src/game/pipeline/stages/HostBroadcastPublisher.ts` 新增 `broadcastHostFlipResetSignalsIfNeeded`，`SimulationStage.ts` 在 `isBroadcastingHost && server` 时转发 `MATCH_SIGNALS.FLIP_RESET_GAINED`（同时将主机输入采集逻辑抽离为 `collectHostBroadcastControls`，保持 `SimulationStage.ts` 低于 Radar 18KB 硬上限）。
- **Exhibition 倒计时/事件 & Training 重置（Issue 2）**：
  - `src/game/modes/experimental-multi-bots/ExperimentalMultiBotsReplayController.ts`：`emitReplaySignal` 在 `isDevSyncHostBroadcastingMode` 下同步转发至 `server.pushBroadcastSignal`（覆盖 `KICKOFF_WILL_START`、`KICKOFF_START`、`SCORE_UPDATED`、`OVERTIME_START`、`MATCH_ENDED`、`PODIUM_START`、`GOAL_REPLAY_WILL_START`）。
  - `src/game/modes/static-training/StaticTrainingPackMode.ts`：`reloadCurrentShot()` 在广播模式下推送 `MATCH_SIGNALS.BALL_RESET`；`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` 在收到 `BALL_RESET` / `ROUND_STARTED` 时重置 `SpectateInterpolationBuffer`，消除瞬移插值拖影。
- **Goal Replay & Cinematic Director 镜头复用（Issue 5）**：
  - `src/game/modes/shared/GoalReplaySessionCoordinator.ts`：直接复用既有 Match 信号管线，在 `beginGoalReplaySession` / `endGoalReplaySession`（非 `network-streamed` 且处于广播模式时）推送 `GOAL_REPLAY_START` 与 `GOAL_REPLAY_END`（并 retire `GOAL_REPLAY_WILL_START` / `GOAL_REPLAY_START` / `GOAL_REPLAY_WILL_END`），由 Client 端 `RuntimeGoalReplaySignalSubController` 自动接管 `startGoalReplayDirectorSession`。
  - `src/game/pipeline/stages/SimulationStage.ts`：在 `isPureCsrClient` 模式下允许 `interpolator.sync(timestamp)` 在 `flags.isGoalReplay` 期间继续推进。

### 3. Step 3：Replay Watch Together 元数据/Roster/队伍颜色与边沿触发相机同步（Issue 4）
- **协议扩展**：
  - `src/network/protocol/GlobalWorkspaceProtocol.ts`：新增 `WorkspaceReplayRosterEntry`、`WorkspaceReplaySyncPayload` 与 `GlobalWorkspaceBroadcastPayload.replaySync`，并在 `sanitizeGlobalWorkspaceBroadcast` 中完成校验与清洗。
- **Host 端发送与边沿切换（Non-Override）**：
  - `src/game/pipeline/stages/AuthorityCameraSyncTracker.ts`：新增 `bumpLocalSeq()`。
  - `src/replay/replayOnlineSyncHelper.ts`：`syncReplayWorkspaceToOnlineRoom` 打包 `replaySync`（`fileName`、`totalFrames`、`currentFrame`、`playbackState`、`flowPct`、`focusedCarIndex`、`roster`）至 `broadcastRoomActivitySwitch` 与 `broadcastGlobalWorkspace`；`ReplayOnlineHeartbeatTracker.maybeSync` 仅在切活动或相机真实发生边沿变化（且非首次基线初始化）时递增 `cameraSyncSeq`，周期性 1-2Hz 心跳保持原 `seq` 不覆盖 Client 本地相机。
- **Client 端 Store 与 HUD Roster 联动**：
  - `src/ui-core/stores/replayViewerStore.ts`：新增 `resolvedRoster` 字段。
  - `src/game/runtime/onlineSessionEnvironmentPipeline.ts`（`applyClientRoomActivitySwitch`）与 `src/game/runtime/devSyncChannelHandlers.ts`（`onGlobalWorkspaceBroadcast`）：同步写入 `useReplayViewerStore` 的 `viewerState`、`loadedFileName`、`totalFrames`、`cursorFrame`、`effectiveFlowPct`、`activePovCarIndex`、`resolvedRoster`（退出 replay 时重置为 `'idle'`）。
  - `src/game/pipeline/stages/hudRosterResolverPipeline.ts` & `src/game/pipeline/stages/TelemetryStage.ts`：当 `replayViewerState !== 'idle'` 且存在同步的 `replayViewerResolvedRoster` 时，优先使用同步 Roster 解析蓝/橙两队名单与颜色。

---

## 二、验证结果

1. **`pnpm run check`（Radar + `tsc --noEmit`）**：**PASS / GREEN**
   - `1425/1425` files 100% TS，`tsc --noEmit` 0 错误，0 架构边界/体积违规。
2. **`pnpm run repro:sync-issues`**：**0/5 issues reproduced**
   - `[NOT_REPRODUCED] issue-1: User Input Broadcast & Keyboard Input Overlay`
   - `[NOT_REPRODUCED] issue-2: Exhibition Countdown, Incidents & Training Sync`
   - `[NOT_REPRODUCED] issue-3: Flip Reset Visual & Audio Sync`
   - `[NOT_REPRODUCED] issue-4: Replay Watch Together UI, Target Player, Names & Colors`
   - `[NOT_REPRODUCED] issue-5: Goal Replay & Cinematic Camera Trigger`
3. **`pnpm run test:multiplayer-suite`**：**24/24 PASS**
   - 新增 Section 23（Pure CSR `pushBroadcastSignal` 与 Goal Replay 信号退场）与 Section 24（Replay Watch Together 元数据/Roster 解析与边沿触发 `cameraSyncSeq` 非覆盖）。
