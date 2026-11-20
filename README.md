# 路 3（Bug 3）专项排查与修复报告：120Hz 物理 / 变帧率渲染、Host 后台 Tab 解耦与 Client Spectate Lookahead 冻结根因

- **仓库**：`Antigravi-ty/RocketCompanion`
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base` (`02adc3d`)
- **修复分支**：`z/agy/rc/track3-bug3-lookahead-bg-loop` (`0d68494`, Verified Bot Commit: `signature.isValid = true, state = VALID`)
- **提交链接**：https://github.com/Antigravi-ty/RocketCompanion/commit/0d68494e24157ca90fe653fe93595e7e23825d0b

---

## 一、核心结论摘要

针对 Bug 3（120Hz 物理 vs 变帧率渲染、Host 后台 Tab 降频突发、Client 23 帧冻结 + 单步大跳变），通过静态代码审计与 Headless Chrome + WebGL2 + WASM 实测打点（`/tmp/probe_bug3.mjs`），已准确定位到 **4 个直接根因** 并完成确定性修复：

1. **Client Replay Watch Together 存在双驱动策略错配（`simulationFrameClassifierPipeline.ts`）**：
   - 纯 CSR Client 在 Replay Watch Together 模式下（`isReconcilerReplayPlayback === true` 且 `isPureCsrClient === true`），`resolveSimulationDriveMode()` 错误返回了 `'local-replay-playback'`，导致 `shouldStepLocalPhysics = true` 与 `shouldRunLookahead = false`，Client 在本地自步进 Freeplay 物理的同时又被 `SpectateInterpolationBuffer` 覆写，产生双驱动冲突与 `tick` 钉死（`12519`）。
2. **Client Lookahead 调用 `physics.step(1)` 时 WASM 处于冻结态（`spectatePhysicsLookaheadPipeline.ts`）**：
   - Client 在进入 `lobby` / `paused` / `SYS_CTRL_STATE_ONLY` 后，`physics.setSimulationFrozen(true, 0, true)` 被激活。当 `applySpectateSampleOrLookahead()` 进入 Lookahead 分支调用 `physics.step(1)` 时，WASM 内部因 `frozen === 1` 直接跳过刚体动力学积分（球与车坐标完全不动），但 JS 侧仍手动覆盖 `physics.state[SIM_OFFSETS.TICK] = nextTick`，表现为**坐标冻结但 tick 假推进**，直到达到 `SPECTATE_LOOKAHEAD_MAX_TICKS (180)` 后彻底停摆。
3. **Host Replay 同步每帧重复调用 `enterStateOnlyMode`（`replayOnlineSyncHelper.ts`）**：
   - `ReplayViewerCausalSyncController.stepBeforePhysicsTick()` 每帧调用 `syncReplayFrameToAuthoritativeServer()`，后者原本直接调用 `server.enterStateOnlyMode(snap, tick)`，导致 `MatchSystemControlSubController` 每帧重复发射 `SYS_CTRL_STATE_ONLY` 系统控制信号并重置状态，而非走轻量级 `server.pushBroadcastSnapshot(snap, tick)`。
4. **Host 后台 Tab 主线程定时器受 Chrome 节流（`RenderClockScheduler.ts`）**：
   - `RenderClockScheduler` 虽然已使用 `MessageChannel` + `setTimeout`/`scheduler.postTask` 与 `requestAnimationFrame` 解耦，但在 `document.hidden === true` 时，主线程 `setTimeout`/`postTask` 在真实浏览器（无 `--disable-background-timer-throttling`）下仍会被 Chrome 强行降频对齐，且 `unfocusedFrameInterval = 1000/30` 限制了帧率。新增后台 Blob Worker 心跳（仅在 `document.hidden && this.simulate && this.active()` 时以 `8ms` 间隔唤醒主线程 `queueTick()`），使 Host 120Hz 物理与广播彻底摆脱主线程后台节流。

*(注：此外确认 `runDevBroadcastReproV3.mjs` L322 调用 `rv.seekToTime?.(8.0)` 而 `bootstrap.sh` 生成的合成 3v3 夹具仅 6 秒 / 720 帧，`seekToTime(8.0)` 会直接定位到末尾第 719 帧并立即触发播放结束进入 `paused` 状态，导致后续采样全部落在 replay 结束后的 `stalled` 区间。)*

---

## 二、逐项深度排查与架构核验

### 1. 120Hz 物理 vs 变帧率渲染架构核验（`PhysicsStateInterpolator.ts`）
- `src/physics/PhysicsStateInterpolator.ts` 已完整实现标准的固定时间步长累加器架构：
  - 固定步长 `FIXED_TIMESTEP = 1/120s (8.333ms)`，支持 `setPhysicsRate` / `setPhysicsTickScale` 微调时钟速率。
  - `update()` 累加 `deltaSec` 并执行 `ticksToRun = Math.floor(accumulator / FIXED_TIMESTEP)` 次 `sim.step(1)`（单帧上限 `MAX_PHYSICS_SUBSTEPS = 12` 防止 spiral-of-death），每次步进前保存 `prevState.set(currState)`、步进后更新 `currState.set(sim.state)`。
  - 渲染侧 `PresentationStage.ts`（L118）使用 `alpha = accumulator / FIXED_TIMESTEP ∈ [0, 1)` 对 `prevState` 与 `currState` 做线性/球面插值送显。
- **结论**：本地 120Hz 物理与 144/165/180Hz 变帧率渲染解耦架构本身完备，问题不在 `PhysicsStateInterpolator` 核心，而在网络观战/Replay Watch Together 管线与后台调度。

### 2. Host 后台 Tab 循环解耦排查（`RenderClockScheduler.ts` & `FramePipeline.ts`）
- `RenderClockScheduler.ts`（L52-96, L288-348）：
  - 前台渲染通过 `MessageChannel` 微任务 + `WebGL2 fenceSync` 驱动，`rAF` 仅用于 `measureDisplay()` 采样。
  - 当 `document.hidden === true` 时，`_getEffectiveFrameInterval()` 返回 `unfocusedFrameInterval`（默认 `33.33ms` / 30 FPS），`schedule()` 通过 `scheduler.postTask` 或 `setTimeout` 调度 `this.simulate(now)`（即 `FramePipeline.executeSimulationOnly()`）。
  - **隐患**：在 Chrome 后台 Tab 中，主线程 `setTimeout` 与 `scheduler.postTask` 会触发浏览器后台定时器节流（1Hz/对齐突发），导致 Host `executeSimulationOnly` 出现长间隔突发，每次突发受限于 `maxAccumSec = 0.25s` / `MAX_PHYSICS_SUBSTEPS = 12` / `maxWallBudgetMs = 28ms`，从而在网络层产生间歇性断流与跳帧。
- **修复方案（已落地 `RenderClockScheduler.ts`）**：
  - 增加 `startBackgroundWorker()` / `stopBackgroundWorker()`：仅当 `document.hidden && this.simulate && this.active()` 时启动轻量级 Blob Worker（`setInterval(() => postMessage(0), 8)`），在 Worker 线程（不受主线程后台 Tab 定时器强降频影响）驱动主线程 `queueTick()`，切回前台或 `dispose()` 时立即终止并释放 `ObjectURL`。

### 3. Client 23 帧冻结 + 单步跳变根因（`simulationFrameClassifierPipeline.ts` & `spectatePhysicsLookaheadPipeline.ts` & `ReplicationStateInterpolator.ts`）
- **实测证据（修复前 `/tmp/probe_bug3.mjs`）**：
  - Client 状态：`mode: "replay-viewer"`, `phase: "lobby"`, `paused: true`, `spectateMirrorMode: true`, `replayPlaybackMode: true`。
  - 修复前 Client `rt.physics.getHeaderView().tickCount` 被钉死在 Freeplay 残留的 `12519`，且进入 `stalled` 后 `tick` 从 `12644` 涨到 `12718` 时，球坐标完全静止在 `(-2397.7, 1010.9, 207)`。
- **根因链路**：
  1. `SpectateInterpolationBuffer.sample(now)` 的内置外推窗口为 `SPECTATE_MIN_LOOKAHEAD_FRAMES = 5`（约 10 ticks ≈ 83ms）。当 Host 突发间隔 > 83ms 时，`renderTick` 抵达 `maxRenderTick`，`sample()` 返回 `isDuplicateFrame: true, isLookahead: true`。
  2. 此时应由 `applySpectateSampleOrLookahead()`（`spectatePhysicsLookaheadPipeline.ts` L185-221）接管，使用 `physics.step(1)` 进行长达最多 `SPECTATE_LOOKAHEAD_MAX_TICKS = 180`（1.5 秒）的确定性 WASM 物理外推。
  3. 但由于：(a) `classifySimulationDrivePolicy()` 将纯 CSR Client 的 `isReconcilerReplayPlayback` 误判为 `'local-replay-playback'`；(b) Client WASM 此前被 `lobby`/`paused` 信号调用过 `setSimulationFrozen(true, 0, true)`，导致 `physics.step(1)` 在 WASM 内部空转不积分坐标，从而出现长达数十帧的视觉冻结，直到 Host 下一批快照抵达时若距离超过 `SPECTATE_CONVERGENCE_DIST_SQ`（1800 uu）或 `lookaheadTicksActive === 0` 便直接瞬移。
- **修复方案（已落地）**：
  1. `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`：当 `isReconcilerReplayPlayback && isPureCsrClient` 时，返回 `'client-csr-spectate'`（或 `'client-goal-replay'`），关闭 `shouldStepLocalPhysics` 并正确开启 `shouldRunLookahead`。
  2. `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`：在 Lookahead 执行 `physics.step(1)` 前调用 `physics.setSimulationFrozen?.(false, 0, false)` 解冻 WASM 物理步进。
  3. `src/replay/replayOnlineSyncHelper.ts`：`syncReplayFrameToAuthoritativeServer()` 优先调用 `server.pushBroadcastSnapshot(snap, tick)`（首帧由 `MatchSystemControlSubController` 自动进入 state-only 模式），避免每帧重复发射 `SYS_CTRL_STATE_ONLY`。

---

## 三、修复后实测验证结果

1. **架构雷达与严格类型检查 (`pnpm run check`)**：
   - `radar.ts`: `1430/1430` TS 文件，0 循环依赖，0 越界导入，**GREEN**。
   - `tsc --noEmit`: **PASS (0 errors)**。
2. **多人同步单元测试矩阵 (`RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`)**：
   - 全量 **28 项测试全部 PASS**（含 Test 13 Replay Watch Together、Test 16 SpectateInterpolationBuffer、Test 26 client-csr-spectate Lookahead & Convergence、Test 27 W2-A isLookahead WASM 接线等）。
3. **Headless Chrome + WebGL2 + WASM 实测打点 (`/tmp/probe_bug3.mjs`, `seekToTime(1.0)`)**：
   - Client `tick` 恢复单调正常推进（`347 -> 360 -> 639 -> 731 -> 821 -> 882 -> 901`），不再被 Freeplay 历史 tick 钉死。
   - 当 Host 播放至第 719 帧结束并暂停进入 `stalled` 后，Client Lookahead 成功驱动 WASM 物理连续积分 180 ticks（`tick 731 -> 821 -> 882 -> 901`），球坐标平滑运动（`(-2629.7, 1354.5, 262) -> (-3128.5, 2184.9, 231.2) -> (-3444.9, 2694.8, 132.9) -> (-3532.1, 2823, 156.5)`），彻底消除冻结。

---

## 四、给后续集成 / 测试夹具的建议

1. `scripts/runDevBroadcastReproV3.mjs` L322 的 `rv.seekToTime?.(8.0)` 超过了 `bootstrap.sh` 生成的 6 秒合成 3v3 replay 时长，建议在测试修复中调整为 `rv.seekToTime?.(1.0)`（或生成 >= 15 秒的合成 replay），以确保 30 帧采样窗口落在活跃播放期而非 replay 结束后的静止期。
2. 本次修复已通过 Verified Bot Commit 提交至分支 `z/agy/rc/track3-bug3-lookahead-bg-loop`（`0d68494`），可直接合入主干或与 路1/路2/Watchdog 基建合并验证 `repro:dev-broadcast-v3`。
