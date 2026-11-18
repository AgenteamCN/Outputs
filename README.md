# RocketCompanion 阶段 1 修复报告（控制权/观战解耦与倒计时协议对齐）

## 一、核心修复内容

### 1. 显式区分 Broadcasting Host 与 Remote Spectating Client（问题 1 / issue-5）

- **`src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts`**：
  - 在 `stepAfterPhysicsTick()` 中用 `isDevSyncHostBroadcastingMode(Boolean(scenePort.authoritativeServer))` 显式区分 **Broadcasting Host** 与 **远程客户端 / 非广播服务端**，不再在 Dev Sync Broadcasting Host 模式下提前 `pollGoal()` 吞掉进球。
  - 在 Host Freeplay 进球触发 `POST_GOAL_CELEBRATION_TICKS` 时，通过 `authoritativeServer.pushBroadcastSignal(MATCH_SIGNALS.GOAL_REPLAY_WILL_START, ...)` 同步广播进球回放即将开始的元数据。
  - 新增 `resetGoalAndTouchForKickoff()` 与 `shouldVoteSkipOnlineReplay()`，支持在庆祝期间手动重置开球时广播 `GOAL_REPLAY_END` 清理客户端状态，并在 `skipGoalReplay()` 中区分 Host 与远程客户端。
- **`src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts` & `freeplayTouchAndGoalPipeline.ts` & `src/game/modes/FreeplayMode.ts`**：
  - 补充 `broadcastFreeplayGoalReplayEndIfPending()` 及 `authoritativeServer.pushBroadcastSignal` 类型定义，`FreeplayMode.skipGoalReplay()` 与 `onResetKickoff()` 保持在 Radar 18KB 体积上限内。
- **额外发现并修复的遗漏点 — `src/game/modes/shared/GoalReplaySessionCoordinator.ts`**：
  - 原先 `beginGoalReplaySession()` 与 `endGoalReplaySession()` 中读取 `netPort?.networkManager?.server`（实际运行时为 `undefined`），导致 `GOAL_REPLAY_START` 与 `GOAL_REPLAY_END` 无法在真实浏览器中广播出去。已修正为优先解析 `netPort?.authoritativeServer ?? netPort?.networkManager?.sessionCoordinator?.authoritativeServer ?? netPort?.networkManager?.server`。

### 2. 统一 Exhibition `KICKOFF_WILL_START` 信号契约与活动状态同步（问题 4 / issue-2）

- **`src/game/modes/experimental-multi-bots/experimentalMultiBotsLifecyclePipeline.ts`**：
  - 在 `startExpKickoffCountdownState()` 中计算并持久化 `state.kickoffTargetTick = currentTick + KICKOFF_COUNTDOWN_TICKS`，`KICKOFF_WILL_START` 信号统一以绝对 `targetTick` 作为 `param1`、`state.isOvertime ? 1 : 0` 作为 `param2`。
  - 在 `stepExpCountdownPhaseState()` 中于 `2` / `1` 秒切换时复用同一个 `state.kickoffTargetTick`（避免 `onBeforePhysicsTick` 与 `onEnter` 之间的 1 tick 偏差导致客户端误判为新开球阶段并反复清空 `latchedKickoffSeconds`）。
- **`src/game/runtime/RuntimeKickoffSignalSubController.ts`**：
  - `isNewKickoffPhase` 对 `targetFrame` 允许 `±2` tick 容差，防止微小 tick 抖动导致倒计时音效重复触发或数字回弹。
  - 放行 `appState.matchMode === 'exp-multi-bots' || appState.matchMode === 'match'` 场景下的 `KICKOFF_WILL_START` 信号处理，避免因房间 `activeRoomActivity` 仍停留在 `'freeplay'` 而导致倒计时信号被全部拦截。
- **额外发现并修复的遗漏点**：
  - **`src/game/modes/experimental-multi-bots/ExperimentalMultiBotsReplayController.ts`**：原 `emitReplaySignal()` 读取 `runtime?.networkManager?.server`（实际为 `undefined`），已修正为 `authoritativeServer ?? sessionCoordinator?.authoritativeServer ?? server`。
  - **`src/game/runtime/MatchModeTransitionBridge.ts`**：进入 `exp-multi-bots` 时在 Dev Sync Room 下同步切换 `activeRoomActivity` 为 `'exp-multi-bots'`，切回训练/Freeplay 时恢复为 `'freeplay'`。
  - **`src/game/subsystems/GameAudioSubsystem.ts`**：暴露 `get audio()` 返回 `this.runtime?.options?.gameAudio || gameAudio`，打通 `rt.audioSubsystem?.audio` 访问链路与倒计时音效观测。
  - **`src/game/runtime/signalWorldTransitionPipeline.ts`**：对 `arena.vehicleRegistry?.ensureCar?.(...)` 增加可选链保护。
  - **`scripts/runMultiplayerSyncIssuesRepro.ts`**：修正 `issue-5` mock 中 `setGoalMetadata` / `appendSignal` / `appendAudioEvent` 缺失导致的运行时错误（此前因提前 return 未暴露）。

## 二、验证结果

- `pnpm run check`（Radar + `tsc --noEmit`）：**GREEN**（1425/1425 TS 文件通过，0 错误）。
- `pnpm run test:multiplayer-suite`：**24/24 PASS**。
- `pnpm run repro:sync-issues`：从 `3/5 reproduced` 降为 **`1/5 reproduced`**（`issue-2` 与 `issue-5` 均已变为 `NOT_REPRODUCED`，仅剩属于阶段 2 的 `issue-4`）。
- `pnpm run test:dev-room-sync-e2e`：**ALL E2E CHECKS PASSED**（`runtime_error=0`）。
- `pnpm run repro:dev-broadcast-v2`（真实 Headless Chrome + WebGL + WASM）：`bug-1-freeplay-goal-replay` 已稳定输出 **`[PASS_FIXED]`**（注：后续因 pixeldrain 公开链接触发频控 403，无法重复下载 `.rcreplay` 复测完整 `v2` 套件，其余无回放依赖的检查均已完整验证）。
