# RocketCompanion 多人联机测试审计、真实化改造与分步修复计划

- **目标仓库与分支**：`Antigravi-ty/RocketCompanion` (`z/agy/rc/multiplayer-recheck`)
- **本次推送 Verified Commit**：[`5cec557dfc96e19eec083b308c6dd9ff47516a45`](https://github.com/Antigravi-ty/RocketCompanion/commit/5cec557dfc96e19eec083b308c6dd9ff47516a45) (`signature.isValid: true`, `state: VALID`)
- **回放基准文件**：`rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`

---

## 一、Commit 历史审计：假测试是如何产生的？

通过检查 `scripts/` 目录下所有联机测试脚本的 Git 提交历史，结论是**“两者兼有”**：一部分测试**从最初创建就是绕过真实管线的 Mock 测试**，而更严重的是 **`runMultiplayerSyncIssuesRepro.ts` 在实现（implement）阶段被直接修改，手动注入了预期信号和载荷，使其从“未覆盖真实管线”进一步退化为“必定通过的假测试”**。

### 1. `scripts/runMultiplayerSyncIssuesRepro.ts`（`repro:sync-issues`）—— 先天 Mock + 实现期人为改测试
- **创建阶段（`836fe89`）**：从第一天起就只实例化了底层 `HostBroadcastPublisher`、`MatchSystemControlSubController`、`PredictionReconciler` 和纯函数 `resolveModeRosterEntries`，**完全没有运行** `FreeplayGoalAndTouchSubController`、`ExperimentalMultiBotsLifecyclePipeline`、`RuntimeKickoffSignalSubController`、`TelemetryStage` 或 React HUD。
- **实现阶段被篡改（`4907669` & `aba3e1f`）**：
  - 在 `4907669` 中，开发者修改了 `reproIssue1`，**手动把构造好的 `testControls` 直接传给 `publisher.publishIfReady({ acknowledgedControls: testControls })`**，绕过了真实控制采集链路。
  - 在 `aba3e1f` 中，开发者进一步修改了 `reproIssue2`、`reproIssue3`、`reproIssue4`、`reproIssue5`：
    - `reproIssue2` 直接手动调用 `systemControl.pushBroadcastSignal(MATCH_SIGNALS.KICKOFF_WILL_START, 200, 180, 3)` 和 `SCORE_UPDATED`，**自己发信号给自己收**，完全没经过 `ExperimentalMultiBotsLifecyclePipeline` 和 `RuntimeKickoffSignalSubController`。
    - `reproIssue3` 直接手动调用 `systemControl.pushBroadcastSignal(MATCH_SIGNALS.FLIP_RESET_GAINED, ...)`。
    - `reproIssue4` 直接手动把伪造的 `replaySync: { fileName, totalFrames, roster: [...] }` 塞进 `syncReplayWorkspaceToOnlineRoom`，并把 `clientUiNotEntered` 的判定条件放宽（去掉了 `clientMode !== 'replay-viewer'` 的检查）。
    - `reproIssue5` 直接手动调用 `systemControl.pushBroadcastSignal(MATCH_SIGNALS.GOAL_REPLAY_WILL_START / GOAL_REPLAY_START)`，绕过了 `FreeplayGoalAndTouchSubController`。
  - 结果：该脚本在 `9c7d27e` 基线上全部报 `0/5 issues reproduced (NOT_REPRODUCED)`，制造了“问题已修完”的假象。

### 2. `scripts/runDevSyncReproSuite.mjs`（`repro:dev-sync-bugs`）—— 真浏览器但浅层 Happy-Path 断言
- 创建于 `98efe40`，在 `c50af67` 和 `8f459e4` 中扩展为 9 个场景（`c50af67` 的修改属于修复 `kb.keys.add` Set API 与增加 `waitFor`，并非刻意造假）。
- **核心问题在于断言过浅**：
  - **Scenario 1 (Freeplay)**：仅检查直线行驶距离 `< 250`（且 `kb.keys['KeyW'] = true` 对 `Set` 无效），未测 Freeplay 射门进球触发 Goal Replay。
  - **Scenario 2 (Exhibition 3v3)**：仅检查 `numCars === 6`，未检查 Client 是否完整经历 `3 -> 2 -> 1` 倒计时。
  - **Scenario 3 (3v3 Replay)**：仅检查帧游标与 Pause ACK，未检查 Client DOM 上方的积分牌（`StepperScoreboardHUD`）与左右 1-6 名单条（`roster-card-*`）。
  - **Scenario 9 (Camera Sync)**：仅检查 `ballCam` 布尔切换，未检查 `focusedCarIndex`（1-6 号位切人）在 Client 端是否同步并持久保持。

### 3. `scripts/runDevRoomSyncE2E.mjs`（`test:dev-room-sync-e2e`）—— Stage 6 仍为手动状态注入
- Stage 6（Replay Watch Together）中直接通过 `authoritativeServer.enterStateOnlyMode(snap, 420)` 与 `onBroadcastGlobalWorkspace({ playbackPaused: true, ... })` 手动注入状态帧，未加载真实 `.rcreplay` 文件，也未检查 Client DOM 积分牌与 1-6 名单条，导致在功能实际残缺的情况下依然全绿通过。

---

## 二、本次测试机制改造与真实运行结果

已在提交 [`5cec557`](https://github.com/Antigravi-ty/RocketCompanion/commit/5cec557dfc96e19eec083b308c6dd9ff47516a45) 中完成以下改造并验证：

1. **重写 `scripts/runMultiplayerSyncIssuesRepro.ts`（并新增 `scripts/runTsWithVite.mjs` 以支持 Vite SSR 模块解析）**：
   - **移除所有手动 `pushBroadcastSignal` / `replaySync` / `acknowledgedControls` 假注入**，改为直接驱动真实业务管线：
     - `issue-1`：通过 `collectHostBroadcastControls` -> `HostBroadcastPublisher` -> `PredictionReconciler` -> `mapRemoteControlsToOverlayKeys`。
     - `issue-2`：在 `tick >= 400` 房间运行态下，直接运行 `startExpKickoffCountdownState` + `stepExpCountdownPhaseState` -> `MatchSystemControlSubController` -> `PredictionReconciler` -> `RuntimeKickoffSignalSubController.handleSignal`。
     - `issue-3`：通过 `broadcastHostFlipResetSignalsIfNeeded` 验证信号链。
     - `issue-4`：直接实例化 `TelemetryStage` 并调用 `_syncEdgeRosterHud()`（模拟 Client `matchMode='freeplay'` 且 `activeRoomActivity='replay'`）。
     - `issue-5`：直接实例化 `FreeplayGoalAndTouchSubController` 并调用 `stepAfterPhysicsTick()`（模拟 Dev Sync Broadcasting Host 存在 `authoritativeServer` 且球过球门线 `posY=5600`）。
   - **实测结果（`pnpm run repro:sync-issues`）**：从 `0/5` 变为 **`3/5 issues reproduced`**，精准命中：
     - `[REPRODUCED] issue-2`：Client 倒计时观测到 `[]`，缺失 `[3, 2, 1]`。
     - `[REPRODUCED] issue-4`：`TelemetryStage._syncEdgeRosterHud()` 提前 return，输出 `roster entries=0`。
     - `[REPRODUCED] issue-5`：`FreeplayGoalAndTouchSubController` 提前 return 并调用 `pollGoal()` 吞掉进球，`goalResetTicks=0`。

2. **强化 `scripts/runDevSyncReproSuite.mjs`（真实 Headless Chrome + WebGL + WASM）**：
   - 修复 Scenario 1 中 `kb.keys.add/delete`；
   - Scenario 2 增加 Client `match.state` 轮询，强制要求观测到 `[3, 2, 1]`；
   - Scenario 3 增加 Client DOM 检查，强制要求 `domScoreboard === true` 且 `domRosterCards >= 6`；
   - Scenario 9 增加 Host 切换 `focusedCarIndex = 2` 后等待 600ms 校验 Client `camera.focusedCarIndex === 2`。
   - **实测结果（`RC_REPLAY_PATH=/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-sync-bugs`）**：从 `8/9` 变为 **`passed=6/9, reproducedBugs=3`**，直接暴露 Scenario 2（`clientCountdownNums: [3]`）、Scenario 3（`domScoreboard: false, domRosterCards: 0`）、Scenario 9（`c2FocusedAfterSwitch: 0`）。

3. **额外暴露的新问题（审计期间观察到）**：
   - **Host 端 3v3 回放 Roster DOM 卡片数量异常**：在 `repro:dev-broadcast-v2` 中，3v3（6 车）回放时 Host 侧 `domRosterCards` 达到 **10**（`cardNames` 中混杂了 `1PlayerYOU33x`、`2Nexto #2...`、`4Player 4...` 与 `4Nexto #4...` 等重复/残留卡片），需要在修复 `TelemetryStage` 与 `EdgeRoster` 渲染时一并排查去重/清理逻辑。
   - **Countdown 音效 Hook 盲区**：`repro:dev-broadcast-v2` 中 `hostCountdownAudioCount` 和 `clientCountdownAudioCount` 均为 `0`，因 Headless 环境下挂载在 `rt.audioSubsystem` 上的音频触发路径与测试 spy 挂载点不一致，后续修复 Issue 4 时应同步核对 `GameAudioSubsystem` / `SignalAudioEngine` 的真实触发。

---

## 三、分步修复计划与团队分工安排

建议按 **“架构解耦 -> 核心 Bug 修复 -> 门禁全绿回归”** 分 3 个阶段推进，预计可由 **2 名工程师（或按 Runtime/网络 与 UI/HUD 双线并行）** 完成：

### 阶段 1（P0）：控制权与观战状态解耦 + 倒计时协议对齐（Owner A：Runtime / Network）
1. **显式区分 `Broadcasting Host` 与 `Remote Spectating Client`（修复问题 1 / issue-5）**
   - **文件**：`src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts`、`src/game/modes/freeplay/FreeplayMode.ts`、`src/game/pipeline/stages/SimulationStage.ts`
   - **动作**：
     - 将提前 return 守卫从 `scenePort.authoritativeServer || scenePort.networkReconciler` 改为 `Boolean(scenePort.networkReconciler) && !isDevSyncHostBroadcastingMode(Boolean(scenePort.authoritativeServer))`。
     - 确认 `SimulationStage.ts` 中 `physics.pollGoal()` 不会在 `FreeplayMode.stepAfterPhysicsTick()` 之前消费掉 goal latch。
     - 在 `beginGoalReplay` / `completeReplaySession` 时，若处于 `isDevSyncHostBroadcastingMode`，通过 `authoritativeServer.pushBroadcastSignal` 广播 `GOAL_REPLAY_WILL_START`、`GOAL_REPLAY_START`、`GOAL_REPLAY_END` 给 Pure CSR Client；同步修复 `FreeplayMode.skipGoalReplay()`。
2. **统一 `KICKOFF_WILL_START` 信号契约（修复问题 4 / issue-2）**
   - **文件**：`src/game/modes/experimental-multi-bots/experimentalMultiBotsLifecyclePipeline.ts`、`src/game/runtime/RuntimeKickoffSignalSubController.ts`
   - **动作**：
     - 发送端在进入 kickoff 时计算绝对目标 tick `targetTick = currentTick + KICKOFF_COUNTDOWN_TICKS`，以 `(targetTick, isOvertime ? 1 : 0)` 发送 `KICKOFF_WILL_START`（或明确协议区分绝对 `targetFrame` 与显式秒数），消除 `effectiveTick >= 240/360` 时 2 和 1 被吞掉的问题。
     - 确保 Client 端 `latchedKickoffSeconds` 稳定触发 `3 -> 2 -> 1` 与 `audio.play('match_countdown_321', 1.0)`。

### 阶段 2（P0）：Replay Watch Together UI/积分牌同步 & 摄像机切人同步（Owner B：HUD / Camera / Workspace）
3. **打通 Client Replay 积分牌与 1-6 名单条（修复问题 2 / issue-4）**
   - **文件**：`src/game/pipeline/stages/TelemetryStage.ts`、`src/ui-core/hud/InGameHUD.tsx`、`src/ui-core/hud/GoalReplayHudOverlay.tsx`、`src/network/protocol/GlobalWorkspaceProtocol.ts`、`src/replay/replayOnlineSyncHelper.ts`、`src/game/runtime/devSyncChannelHandlers.ts`
   - **动作**：
     - 将 `isClientSyncedReplay`（`onlineSession?.activeRoomActivity === 'replay' && rvStoreState?.viewerState !== 'idle'`）提前到 `TelemetryStage._syncEdgeRosterHud()` 第 169 行的入口守卫条件中，避免死代码。
     - 在 `InGameHUD.tsx` / `GoalReplayHudOverlay.tsx` 中，当处于 `isDevSyncRoom && activeRoomActivity === 'replay'` 时允许显示 `showTopScoreboard` 与 `showEdgeRosterStrips`，不再强依赖本地 `totalFrames > 0`。
     - 扩展 `WorkspaceReplaySyncPayload` 增加 `blueScore`、`orangeScore`、`remainingSeconds`、`isOvertime`，并在 `devSyncChannelHandlers.ts` 中同步写入 `hudStore`。
     - **顺带排查**：检查 3v3 回放时 Host 出现 `domRosterCards: 10` 的重复卡片问题，确保双端严格只渲染当前模式的 6 张卡片。
4. **修复 1-6 号位观战目标同步与初始对齐（修复问题 3）**
   - **文件**：`src/game/runtime/devSyncChannelHandlers.ts`、`src/game/subsystems/GameCameraSubsystem.ts`、`src/game/modes/freeplay/StandaloneReplayViewer.ts`
   - **动作**：
     - 在 `channel.onCameraStateSync` 中，当 `shouldApplyIncomingCameraSync` 为真时，除调用 `applyCameraSyncToPort(camera, camSync)` 外，必须同步更新 `useSpectateWorkspaceStore` 的 `hostCameraSync`（以及 `useReplayViewerStore.activePovCarIndex`），防止下一帧 `GameCameraSubsystem.update()` 被旧 store 状态覆盖回 `0`。
     - 严格遵守边沿触发语义（`cameraSyncSeq` 递增时才覆写 Client），保证 Host 切人时 Client 立即跟随，Host 未切换时 Client 本地自由视角不被周期拉回。
     - 初始进入房间或切换至 Replay 时，Host 主动广播一次含当前 `focusedCarIndex` 的 `cameraSync`。

### 阶段 3（P1）：最终验收门禁（Owner A + Owner B 联调）
所有修复完成后，必须依次运行以下全部命令并达到指定通过标准，方可合入：
1. `pnpm run check`（`radar` + `tsc --noEmit` 零错误）
2. `pnpm run test:multiplayer-suite`（24/24 PASS）
3. `pnpm run repro:sync-issues`（从 `3/5 reproduced` 降为 **`0/5 reproduced`**）
4. `RC_REPLAY_PATH=/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-sync-bugs`（从 `6/9` 变为 **`9/9 PASS, reproducedBugs=0`**）
5. `RC_REPLAY_PATH=/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-broadcast-v2`（从 `4/4 reproduced` 降为 **`reproduced=0/4`**）
6. `pnpm run test:dev-room-sync-e2e`（`ALL E2E CHECKS PASSED`，且后续建议将 Stage 6 也升级为真实 `.rcreplay` 加载 + DOM 校验）
