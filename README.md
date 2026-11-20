# Track B（Bug 1）修复报告：`fix/bug1-goal-replay-client-dynamics`

## 1. 分支与提交信息
- **Base Branch**: `origin/z/agy/rc/multiplayer-recheck` (`7252e42`)
- **Feature Branch**: `fix/bug1-goal-replay-client-dynamics`
- **Verified Bot Commit**: `4364561030117f4b27981cbf44e908ad303adac8`（GitHub GraphQL `createCommitOnBranch`，`signature.isValid: true, state: "VALID"`，`Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 2. 根因定位
1. **Host 端 Goal Replay 阶段驱动策略误判导致停止发包**：
   - 在 `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts` 的 `resolveSimulationDriveMode` 中，原本先判断 `if (flags.isLocalDirectorPreview || flags.isLocalReplayControllerStepping) return 'local-replay-playback'`，再判断 `if (hasServer)`。
   - Freeplay Goal Replay 启动时 `flags.isLocalDirectorPreview` 为 `true`，导致广播中的 Host（`hasServer: true, isBroadcastingHost: true`）被错误归类为 `'local-replay-playback'`（`shouldBroadcastSnapshot: false`），而不是 `'host-goal-replay'`（`shouldBroadcastSnapshot: true`）。结果 `SimulationStage` 重置了 `broadcastPublisher` 并停止发送 Goal Replay 快照，Client 侧 `spectateInterpolationBuffer` 始终停留在 `bufFrames: 1`。
   - 同时 `src/game/pipeline/stages/SimulationStage.ts` 中 `stepNetworkReconciliationAndReplayStream` 被 `&& !flags.isLocalDirectorPreview` 拦截，导致 Host 广播期间 `server.update()` 也被跳过。
2. **Host 端 `executeSimulationOnly` 节流路径跳过 `GameplayModeStage`**：
   - 在 `src/game/pipeline/FramePipeline.ts` 中，当 `rAF` 节流或后台运行触发 `executeSimulationOnly(timestamp)` 时，仅执行 `stages[0]`（`InputStage`）与 `stages[1]`（`SimulationStage`），跳过了 `stages[2]`（`GameplayModeStage`）且未更新 `dt` / `lastTimestamp`，导致 `FreeplayMode.update()` 中的 `DirectorPreviewSubController` / `ReplayTimelineController.stepPlayback()` 无法推进回放游标。
3. **关于 `repro:dev-broadcast-v3` 用例时窗的补充核查**：
   - `scripts/runDevBroadcastReproV3.mjs` 在启动 Host + Client（约 3.5s 静止在 `(0, 0, 93.2)`）后直接调用 `setBallState({ pos: [0, 4850, 95], vel: [0, 2950, 40] })`，且未触发 `lastTouchTick` 或 `onResetKickoff`。`extractGoalReplayFrames` 按 `lookbackSeconds: 10.0` 提取了约 1351 帧（前 ~1185 帧 Host 自身球也静止在 `(0, 0, 93.2)`），而 `runDevBroadcastReproV3.mjs` 仅在进入 replay 后采样前 `6 * 200ms = 1.2s`。
   - 修复后实测：未改脚本时 Client `bufFrames` 已由 `1,1,1,1,1,1` 恢复为持续递增 `6 -> 7 -> 22 -> 32 -> 40 -> 51`（说明 Host 持续发包、Client 持续入队并插值）；当将 Host `replayController` 定位到进球动态窗口（`tot - 168`）验证时，`bug-1-goal-replay-client-static-no-dynamics` 立即转为 **`PASS_FIXED`**（`enteredReplay: true, distinctBallPositions: 3`，球位置从 `5387.9 -> 5648.1 -> 5884.3` 连续推进）。
   - 另外核查了 `simulationNetworkReconcilePipeline.ts` 中的 `hasResetSignal` 与 `drainedReplayPackets`：`reconcilerInboundPacketParser.ts` 解析包未挂载 `.signals` 字段，不会每帧误重置 buffer；`policy.mode === 'client-goal-replay'` 时 `shouldFeedReplayTimeline` 为 `true`，`drainedReplayPackets` 正常驱动 `ReplayTimelineController`。

## 3. 修改文件清单
- `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`：调整 `resolveSimulationDriveMode` 顺序，使 `hasServer` 优先于 `isLocalDirectorPreview`（`isLocalReplayControllerStepping && !isLocalDirectorPreview` 仍走 `local-replay-playback`），确保 Host Goal Replay 归类为 `host-goal-replay` 并持续广播。
- `src/game/pipeline/stages/SimulationStage.ts`：将 `stepNetworkReconciliationAndReplayStream` 入口条件改为 `if (isBroadcastingHost || (reconciler && !flags.isLocalDirectorPreview))`，保证 Host Goal Replay 期间 `server.update()` 持续执行。
- `src/game/pipeline/FramePipeline.ts`：在 `executeSimulationOnly` 中计算有界 `dt`、更新 `lastTimestamp`，并在 `isGoalReplayActive()` 或 `match.state.phase === 'replay'` 时执行 `GameplayModeStage`（`stages[2]`）。
- `scripts/runMultiplayerSuiteTests.ts`：新增第 27 项单测 `testHostGoalReplayBroadcastAndSimOnlyStepping`。

## 4. 验证结果
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 hard errors）。
- `pnpm run test:multiplayer-suite`：**27/27 PASS**。
- `pnpm run repro:dev-broadcast-v3`（Headless Chrome + WebGL2 + WASM）：
  - Client `bufFrames` 恢复为 `6 -> 7 -> 22 -> 32 -> 40 -> 51`（不再卡在 `bufFrames: 1`）。
  - 定位至进球动态窗口（`tot - 168`）验证：`bug-1-goal-replay-client-static-no-dynamics` 判定 **`PASS_FIXED`**（`enteredReplay: true, distinctBallPositions: 3`）。
