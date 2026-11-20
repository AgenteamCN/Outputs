# CC-Track (Bug 1 / Track B-2) 修复交付报告

- **分支**: `fix/bug1-goal-replay-snapshot-stream`（基于 `origin/z/agy/rc/multiplayer-recheck-base` @ `7435735`）
- **Verified Bot Commit**: `7147838d3259b3b9d82d76446c475dceb9e4034f`（`verified: true`, `antigravity-fine-grained-token[bot]`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 1. 根因定位

在基线 `7435735` 上通过 Headless Chrome + WebGL2 + WASM 联调诊断发现：
1. Track B (`4364561`) 合入后，Host 在 Goal Replay 期间已持续推流，Client 的 `spectateInterpolationBuffer` 也能正常接收帧（`bufFrames` 从 `8` 增长到 `62`），且当 Host 播放到球运动片段时，Client 能同步产生球体运动。
2. 但 `bug-1-goal-replay-client-static-no-dynamics` 依然复现的根因在于：**Freeplay 模式下无玩家触球（`lastTouchTick < 0`）时**，`prepareFreeplayGoalReplaySessionData()` 调用 `extractGoalReplayFrames()` 会回溯 `lookbackSeconds: 10.0`（约 1200 帧）。当开球后球在中圈 `(0, 0, 93.2)` 静止等待数秒再触发射门/进球时，Host 提取的 Goal Replay 前 ~1186 帧（约 9.88 秒）全部是静止在中圈的帧，导致 Host 与 Client 在 Goal Replay 前 1.2s 采样窗口内看到的球位置始终静止在 `(0, 0, 93.2)`。

## 2. 修复内容

1. **`src/replay/ReplayPlaybackState.ts`**
   - 新增并导出纯函数 `trimLeadingIdleGoalReplayFrames(frames, leadInFrames = 12)`：检测第一帧球体离开静止开球位置/速度（`|posX| > 2 || |posY| > 2 || |vel| > 5`）的索引，当其前导静止帧超过 `leadInFrames` 时裁剪至仅保留 `12` 帧（100ms）前导缓冲。
2. **`src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts`**
   - 在 `prepareFreeplayGoalReplaySessionData()` 中，当 `incident.replayType !== 'expire' && incident.lastTouchTick < 0` 时调用 `trimLeadingIdleGoalReplayFrames(extractedFrames, 12)`，避免将长达数秒的静止开球画面带入 Goal Replay 与广播流。
3. **`scripts/runMultiplayerSuiteTests.ts`**
   - 新增单测 `28. Freeplay Goal Replay Leading Idle Kickoff Trim (Bug 1 / CC-Track)`（`testFreeplayGoalReplayLeadingIdleTrim`），验证前导静止帧被正确裁剪且 12 帧前导后即进入球体运动状态。

## 3. 门禁与实测验证结果

- **`pnpm run check`**（`radar` + `tsc --noEmit`）：**PASS**（1430/1430 TS，0 循环依赖，0 hard errors）
- **`pnpm run test:multiplayer-suite`**：**PASS**（全部 28+ 用例通过，含新增用例 `28`）
- **`pnpm run repro:dev-broadcast-v3`**：**`bug-1-goal-replay-client-static-no-dynamics` 转 `PASS_FIXED`**
  - `enteredReplay: true`
  - `distinctBallPositions: 4`（采样序列中 `by`: `5315.9 -> 5695.4 -> 5854.9 -> 5884.3`，`bufFrames`: `11 -> 22 -> 29 -> 39 -> 49 -> 59`）
