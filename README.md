# 分支 1（Bug 3 Client Lookahead 与缓冲/调度修复）完成报告

## 分支与提交信息
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base` (`e8aae572a729b535bae90e74adaba069a4b41524`)
- **新分支**：`z/agy/rc/fix-bug3-client-lookahead`
- **Verified Bot Commit**：`22d0160524d9c0609fded6c34265435d8b215762`（`verified: true`, `reason: "valid"`, `antigravity-fine-grained-token[bot]`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 修改文件与修复内容（严格锁定在分支 1 范围内）
1. `src/network/sync/ReplicationStateInterpolator.ts`
   - 将 `SpectateInterpolationBuffer` 默认 `maxFrames` 从 `64`（120Hz 下约 0.53s）提升至 `360`（3s 缓冲），防止 SwiftShader 长渲染帧期间数据包堆积淘汰旧帧，进而避免 `sample()` 中 `Math.max(oldestTick, ...)` 引发的单帧大跨度硬切。
2. `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
   - 将 `applySpectateSampleOrLookahead` 中 lookahead 恢复步长的 `dtMs` 上限从 `50ms` 放宽至 `200ms`，单步 `ticksToStep` 上限从 `4` 放宽至 `24`（同时受剩余 `SPECTATE_LOOKAHEAD_MAX_TICKS - lookaheadTicksActive` 预算约束），避免长帧恢复后外推持续滞后。
3. `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
   - 在 `isRemoteReplayWatchTogether` 模式下，新增 `!matchState.paused && !(isRemoteReplayWatchTogether && sampled?.cursorState === 'stalled')` 门控，当 Host 录像暂停或播完停止推进（`cursorState === 'stalled'`）时停止 Client 继续外推 180 ticks，避免触发 `3a-max-la-freeze` 死锁。
4. `src/game/RenderClockScheduler.ts`
   - 在 `wake()` 中同步重置 `this.nextSimTime = 0`。
   - 在 `measureDisplay`（`rAF` 回调）中，当 `this.simulate && this.pending.length >= 2` 且 `now >= this.nextSimTime - 0.1` 时推进一次 `this.simulate(now)`，消除 SwiftShader GPU fence 堆积期间 `SimulationStage` 调度饥饿。
   - 在 `schedule()` 中将 `nextTargetTime` 对 `this.nextSimTime` 的对齐条件从 `effectiveInterval > SIM_INTERVAL_MS` 放宽为 `this.simulate && this.nextSimTime > 0`，确保任意帧率限制下均不会延迟仿真步进。

## 校验结果
- `pnpm run check`：**GREEN**（`1430/1430` TS 100%，`tsc --noEmit` 0 errors，硬边界与体积门禁全部通过）。
- `RC_SUITE_WATCHDOG_MS=120000 pnpm run test:multiplayer-suite`：**28/28 PASS**（EXIT=0）。
- `pnpm run repro:dev-broadcast-v3`（仅含分支 1 改动，尚未合入分支 2 的 `runDevBroadcastReproV3.mjs` 判定调整）：
  - `bug-1`: `PASS_FIXED`（`bufFrames` 正常缓冲至 89–141，未再因 64 帧上限淘汰）
  - `bug-2`: `PASS_FIXED`
  - `bug-3` 实测变化：
    - 修复前：`distinctTicks: 8`, `frozenBallFrames: 24`, `maxBallStepUu: 3607.3`，播放途中即出现帧冻结与 +451 tick 硬切（`1545 -> 1545`）。
    - 修复后：`distinctTicks: 14`, `frozenBallFrames: 16`, `maxBallStepUu: 1127.2`，`sampleSlice` 前 12 帧（`1012 -> 1023 -> 1064 -> 1164 -> 1169 -> 1271 -> 1369 -> 1464 -> 1560 -> 1572 -> 1664 -> 1767`）**连续推进无冻结帧**，直至第 14 帧触达 1800 帧录像末尾后稳定停住。
