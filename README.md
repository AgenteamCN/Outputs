# Track A (Bug 3) 修复报告：`fix/bug3-lookahead-wiring`

## 1. 分支与提交信息
- **仓库**: `Antigravi-ty/RocketCompanion`
- **基准分支**: `origin/z/agy/rc/multiplayer-recheck` (`7252e42`)
- **新建分支**: `fix/bug3-lookahead-wiring`
- **Verified Bot Commit**: [`ed108be49ff8e9233d61d572b98b1d846e969b89`](https://github.com/Antigravi-ty/RocketCompanion/commit/ed108be49ff8e9233d61d572b98b1d846e969b89) (`signature.isValid: true`, `state: VALID`, `author: antigravity-fine-grained-token[bot]`)

## 2. 独占文件变更范围
严格限制在 Track A 独占文件内（零跨线冲突）：
1. `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
2. `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
3. `scripts/runMultiplayerSuiteTests.ts`（末尾追加 Test 27）

## 3. 根因分析与具体修复

### 3.1 W2-A `sampled.isLookahead` 未接入 W2-B WASM 物理前瞻
- **根因**：`SpectateInterpolationBuffer.sample()` 在游标进入 `'lookahead'` 状态时（`renderTick > newestTick + 1e-4`），`isLookahead` 为 `true` 且 `isDuplicateFrame` 为 `false`。`applySpectateSampleOrLookahead` 原先仅判断 `if (sampled && !sampled.isDuplicateFrame)`，导致 `isLookahead === true` 时直接应用 W2-A 的运动学线性外推快照并清零 `lookaheadTicksActive`，从未触发 WASM `physics.step(1)` + `lastAcknowledgedControls` 预测。
- **修复**：
  - 将首分支条件改为 `if (sampled && !sampled.isDuplicateFrame && !(shouldRunLookahead && sampled.isLookahead))`，使 `shouldRunLookahead && sampled.isLookahead` 路由到 WASM `physics.step(1)` + `setCarControls` 前瞻分支。
  - 当 `lookaheadTicksActive >= SPECTATE_LOOKAHEAD_MAX_TICKS`（30 ticks）达到上限时，在回退 `if (sampled)` 分支中保持当前前瞻物理状态（`return false`），避免将 `physics.state` 回跳覆盖为陈旧的 `sampled.snapshot`。

### 3.2 Replay Watch Together 客户端绕过 `SpectateInterpolationBuffer` 与 `matchPaused` 阻断
- **根因**：
  1. Replay Watch Together 下，Client `reconciler.replayPlaybackMode === true`，导致 `simulationFrameClassifierPipeline` 将 `policy.mode` 归类为 `'local-replay-playback'`（`shouldRunLookahead: false`），且 `simulationNetworkReconcilePipeline.ts` 中 `useSpectateBuffer` 原先显式排除了 `!reconciler.replayPlaybackMode`，使客户端直接跳过 `SpectateInterpolationBuffer` 与前瞻管道。
  2. Headless Chrome E2E 实测发现，Replay Watch Together 期间 Client 侧 `rt.match.state.paused` 为 `true`（`matchPhase: "lobby"`），若用 `!matchState.paused` 门控 `shouldRunLookahead`，会再次阻断 WASM 前瞻；而 Host 暂停时 `HostBroadcastPublisher` 会停发包，`SpectateInterpolationBuffer` 与 `SPECTATE_LOOKAHEAD_MAX_TICKS`（30 ticks）本身即可在 250ms 后自动封顶并进入 `stalled`。
- **修复**：
  - 在 `simulationNetworkReconcilePipeline.ts` 中新增 `isRemoteReplayWatchTogether = Boolean(reconciler.replayPlaybackMode && isPureCsrSpectate && !flags.isLocalReplayControllerStepping)`，将其纳入 `useSpectateBuffer`。
  - 计算 `shouldRunLookahead = policy.shouldRunLookahead || (isRemoteReplayWatchTogether && !flags.isReplayPaused && !flags.isMatchEnded && !flags.isGoalReplay)` 并传入 `applySpectateSampleOrLookahead`。

## 4. 验证结果
- **`pnpm run check`**（`radar` + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 hard errors）。
- **`pnpm run test:multiplayer-suite`**：**27/27 PASS**（含新增 Test 27：`W2-A isLookahead WASM lookahead wiring & Replay Watch Together buffer routing`）。
- **Headless Chrome + WebGL2 + WASM 观察**：
  - 沙箱中双标签页（Host + Client）在 SwiftShader 软渲染下，Replay Watch Together 默认 12s `.rcreplay`（`720` 帧 / `tick 719`）在 `seekToTime(8.0) + 2500ms` 后易提前播放到末尾 `tick 719`；在加入 `matchPaused` 修复前实测 `distinctTicks` 已从 `1` 提升至 `7` 并观测到 `719 -> 724 -> 720 -> 719` 的回跳现象，随后已通过 `SPECTATE_LOOKAHEAD_MAX_TICKS` 状态保持修复消除回跳。
