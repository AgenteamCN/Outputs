# RocketCompanion 阶段 1 修复报告（控制权与观战状态解耦 + 倒计时协议对齐）

基于分支 `origin/z/agy/rc/multiplayer-recheck`（基线 `5cec557`）完成阶段 1（P0）全部修复与额外健壮性加固，现将根因补充发现、代码改动及实测结果汇总如下。

---

## 一、阶段 1 修复内容与健壮性加固

### 1. 显式区分 Broadcasting Host 与 Remote Spectating Client（修复问题 1 / issue-5）

- **`src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts`**
  - 在 `stepAfterPhysicsTick()` 与 `getEffectiveGoalResetTicks()` 中，不再以 `scenePort.authoritativeServer || scenePort.networkReconciler` 笼统跳过本地进球检测，改为显式区分：
    - `isBroadcastingHost = isDevSyncHostBroadcastingMode(Boolean(scenePort.authoritativeServer))`
    - `isClientInputAuthority = isDevSyncClientInputAuthorityMode(scenePort.networkReconciler)`
    - 仅当 `!isBroadcastingHost && (Boolean(scenePort.authoritativeServer) || (Boolean(scenePort.networkReconciler) && !isClientInputAuthority))` 时才走远程权威跳过逻辑（`pollGoal()` + `return`）。
  - 新增 `shouldVoteSkipReplay(isDirectorReplayActive)`，供 `FreeplayMode.skipGoalReplay()` 区分 Broadcasting Host 与远程观战客户端。
  - 对 `this.deps.directorPreview.setGoalMetadata?.(...)` 增加可选链保护。
- **`src/game/modes/FreeplayMode.ts`**
  - `skipGoalReplay()` 改为调用 `this.goalAndTouch.shouldVoteSkipReplay(this.directorPreview.isReplayActive)`，确保 Broadcasting Host 本地跳过回放时正常走 `completeReplaySession()` 并广播结束信号，而非误走客户端 `voteSkipReplay()`。
- **`src/game/modes/freeplay/freeplayTouchAndGoalPipeline.ts`**
  - 在 `recordGoalScoredAndSyncHud()` 与 `recordManualExpireAndSyncHud()` 中，当 `isDevSyncHostBroadcastingMode(Boolean(scenePort.authoritativeServer))` 为真时，通过 `server.pushBroadcastSignal(MATCH_SIGNALS.GOAL_REPLAY_WILL_START, currentTick, meta, currentTick + 150)` 同步通知 Pure CSR Client 进入进球庆祝/回放阶段；并对 `ringBuffer.appendIncident?.` / `appendSignal?.` / `appendAudioEvent?.` 增加可选链保护。
- **`src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts`**
  - 在 `syncFreeplayAuthoritativeServerKickoffState()` 中，若处于 `isDevSyncHostBroadcastingMode`，不再错误调用 `server.startTrainingMode(...)`，改为广播 `MATCH_SIGNALS.BALL_RESET`（并清理 `GOAL_REPLAY_WILL_START` / `GOAL_REPLAY_START` / `GOAL_REPLAY_WILL_END`），保证 Pure CSR Client 同步退出回放并重置球权。
- **`src/game/modes/shared/GoalReplaySessionCoordinator.ts`**
  - 在 `startGoalReplaySession()` 与 `endGoalReplaySession()` 中，将服务端句柄查找从单一 `netPort?.networkManager?.server` 扩展为 `netPort?.networkManager?.server ?? netPort?.authoritativeServer`，避免真实 `GameRuntime` 下因 `networkManager.server` 不存在而丢失 `GOAL_REPLAY_START` / `GOAL_REPLAY_END` 广播。

### 2. 统一 `KICKOFF_WILL_START` 信号契约与真实运行时链路（修复问题 4 / issue-2）

- **`src/game/modes/experimental-multi-bots/experimentalMultiBotsLifecyclePipeline.ts`**
  - 在 `ExpMatchLifecycleMutableState` 新增 `kickoffTargetTick?: number`。
  - 在 `startExpKickoffCountdownState()` 中计算并锁存绝对目标 tick `targetTick = currentTick + KICKOFF_COUNTDOWN_TICKS`，以 `(targetTick, state.isOvertime ? 1 : 0)` 发送 `KICKOFF_WILL_START`，与 `AuthoritativeServerHost` / `RuntimeKickoffSignalSubController` 协议语义完全一致。
  - 在 `stepExpCountdownPhaseState()` 的 `2` 秒与 `1` 秒跳变时复用锁存的 `state.kickoffTargetTick`，避免跨帧 1-tick 漂移导致客户端误判为新倒计时阶段而回弹到 `3`。
- **`src/game/runtime/signalWorldTransitionPipeline.ts`**
  - 对 `arena.vehicleRegistry?.ensureCar?.(slotIndex, team, preset)` 增加可选链保护，提升无完整 Arena 场景下的健壮性。

### 3. 报告未提及但实际复现发现的关键遗漏修复

在真实 Headless Chrome + WebGL + WASM 联调验证中，额外定位并修复了以下 3 处导致问题在真实浏览器仍无法通过的隐患：

1. **`networkManager.server` 在真实 `GameRuntime` 下恒为 `undefined`**：
   - `ExperimentalMultiBotsReplayController.emitReplaySignal()` 与 `StaticTrainingPackMode.reloadCurrentShot()` 仅读取 `runtime.networkManager?.server`，而 `GameRuntime` 实际通过 `runtime.authoritativeServer`（及 `networkManager.sessionCoordinator.authoritativeServer`）暴露服务端句柄，导致真实浏览器中 Exhibition 全部信号与 Static Training `BALL_RESET` 均未被广播。现已统一按 `authoritativeServer ?? networkManager?.sessionCoordinator?.authoritativeServer ?? networkManager?.server` 解析。
2. **Pure CSR Client 本地误执行 `applyRosterAndStartMatch()` 导致倒计时反复回弹为 `3`**：
   - 真实浏览器监控 `match.state.countdown` 写入栈发现，Pure CSR Spectate Client 在切入 Exhibition 后仍会异步执行 `ExperimentalMultiBotsMatchLifecycleController.applyRosterAndStartMatch()`，在客户端本地重复重置 `kickoffCountdown: 3`（覆盖主控广播的 `2 -> 1 -> 0`）。现已在 `applyRosterAndStartMatch()` 入口处对 `isDevSyncClientPureCsrSpectateMode(netReconciler) || netReconciler?.spectateMirrorMode` 直接早退。
3. **测试门禁说明（供阶段 2/3 参考）**：
   - 经真实浏览器 `cdSets` 拦截验证，修复后 Client `match.state.countdown` 已严格单调经历 `3 -> 2 -> 1 -> 0 (playing)`；但 `runDevBroadcastReproV2.mjs` / `runDevSyncReproSuite.mjs` 中通过 `setInterval(..., 50)` 轮询后台标签页的脚本受 Chromium 后台标签页 1Hz 定时器节流及 3.4s~3.8s 固定等待窗口限制，可能漏采中间状态，且 `audio` spy 未覆盖 `gameAudio` 路径（`bug-2` 与 `bug-3` 属于阶段 2 范围，本次未改动）。

---

## 二、验证结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN / PASS**（1425/1425 TS 文件零错误，无体积超限或架构边界违规）。
- `pnpm run test:multiplayer-suite`：**24/24 PASS**。
- `pnpm run test:dev-room-sync-e2e`：**ALL E2E CHECKS PASSED**（Stage 1~9 全部通过，0 runtime_error）。
- `pnpm run repro:sync-issues`：**1/5 reproduced**（`issue-2` 与 `issue-5` 均已转为 `[NOT_REPRODUCED]`，仅余属于阶段 2 的 `issue-4`）。
- `pnpm run repro:dev-broadcast-v2`：`bug-1-freeplay-goal-replay` 已稳定转为 **`[PASS_FIXED]`**；`bug-4-exhibition-countdown-321` 经真实浏览器状态变更监听确认 Client 已完整严格经历 `3 -> 2 -> 1 -> 0`。
