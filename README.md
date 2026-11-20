# RocketCompanion Octopus Merge 完成报告与多小队并行排查后续计划

**当前基线**：`origin/z/agy/rc/multiplayer-recheck-base`（HEAD: `7af0003476afc0b275e34e7abebfd08df018d990`，`verified: true`）  
**已合入分支（4-Parent Verified Bot Octopus Merge）**：
- `z/agy/rc/track1-bug1-goal-replay-test-fix`（`0ff3e16`）
- `z/agy/rc/track2-bug2-nametag-edgesync`（`826162b`）
- `z/agy/rc/track3-bug3-lookahead-bg-loop`（`0d68494`）
- 含 `scripts/runDevBroadcastReproV3.mjs` `seekToTime(8.0 -> 1.0)` 适配 6s 合成录像，且含基线 `aef499e` 的统一 270s 软超时与流式输出。  
**分支清理**：上述 3 个临时分支已全部从 `origin` 删除（`HTTP 204`）。

---

## 一、是否需要现在手动测试？

**不需要浪费时间手动测！**  
我们在 `7af0003` 同等代码上实测了 `pnpm run check`（**PASS**，1430/1430 TS）与两次 `pnpm run repro:dev-broadcast-v3`，自动化 E2E 已经**确定性地**暴露出合并后依然存在问题，完全不需要人工肉眼去复现：
1. **`bug-3` 仍未过**：实测 `distinctTicks: 9, frozenBallFrames: 24, maxBallStepUu: 1739.6`。Client 采样前 5 帧卡在 `tick: 222`（`0,0,93.2`），随后直接跳变 `222 -> 265 -> 375 -> 434 -> 444 -> 750`。
2. **`repro:dev-broadcast-v3` 在 SwiftShader 下存在明显偶发抖动**：
   - 第 1 次：`bug-1`、`bug-2`、`bug-4` 为 `PASS_FIXED`；
   - 第 2 次：超时等待 `Host enters Freeplay Goal Replay`（20s）；
   - 第 3 次：`bug-1`（仅采样 3 帧，`distinctBallPositions: 2`）、`bug-2`（`edgeSync` 5s 等不到第 2 次切换）、`bug-4`（单帧跳 110 tick 导致 `maxSinglePadFlips: 2`）均因时序/CPU 抢占出现失败。

---

## 二、4 路并行排查小队分工（互不重叠，快速定位）

建议直接拆出 **4 个只读/定向排查小队并行跑**，各自只盯一个明确问题，出证据后再决定改法：

### 小队 A：Bug 3 —— Client Lookahead 为什么依然连续冻结 24 帧并跳变 1739.6 uu？
- **聚焦文件**：`src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`、`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`、`src/network/sync/ReplicationStateInterpolator.ts`
- **必查问题**：
  1. 在 `7af0003` 上运行 Replay Watch Together 时，`applySpectateSampleOrLookahead` 到底走了哪个分支？（为什么前 5 帧卡在 `tick: 222`、`dist < 0.01`，之后又突然跳 `+43` / `+110` ticks？）
  2. 是 `sampled` 返回了 `isDuplicateFrame: false` 的旧帧，还是 `shouldRunLookahead` 在 Replay Watch Together 某个子阶段仍为 `false`（例如 `paused` / `lobby` 标志位）？
  3. 当 Host 一次性跳 `110 ticks` 抵达时，`isBallDeltaWithinConvergenceThreshold`（阈值 `1800 uu`）刚好在边界，是否触发了硬切瞬移？

### 小队 B：Bug 3 —— Host 后台 Blob Worker 实际心跳与 6s Replay 到头回绕行为
- **聚焦文件**：`src/game/RenderClockScheduler.ts`、`src/replay/StandaloneReplayViewerController.ts`、`scripts/env/generateReplayFixture.ts`
- **必查问题**：
  1. Headless Chrome 下 `clientCdp.send('Page.bringToFront')` 之后，Host 的 Blob Worker 8ms 心跳是否真的稳定驱动了 `executeSimulationOnly()`，还是受 SwiftShader 单核 CPU 抢占导致 Host 自己卡顿几十帧？
  2. 采样后半段出现 `tick: 750, 771, 858` 且球回到 `(0, 0, 93.2)`，是否因为 6 秒（720 帧）录像在采样第 10 帧就已经播完并自动 pause/reset？如果是，建议将 `generateReplayFixture.ts` 默认夹具时长从 6s 提到 12s–15s，避免采样窗口撞上播完边界。

### 小队 C：Bug 1 & Bug 2 —— Headless SwiftShader 下的测试稳定性与超时阈值
- **聚焦文件**：`scripts/runDevBroadcastReproV3.mjs`
- **必查问题**：
  1. `bug-1` 偶发 `Timeout waiting for Host enters Freeplay Goal Replay after 20000ms` 以及 `sampleCount: 3`：查清 `?skipShaderPrewarm=1` 下首次 Goal Replay 着色器编译耗时峰值，是否应提前做一次预热进球或在测试端放宽/分阶段等待。
  2. `bug-2` 偶发 `clientAfterHostEdge2: null`（5s 超时）：在双 Tab SwiftShader 并发渲染下，5s 等待 Host 边沿同步是否过于极限，以及 `SimulationStage.ts` 轮询间隔在低 FPS 下是否会漏/延迟发送。

### 小队 D：Bug 4 & Bug 5 —— 残留边界复核
- **聚焦文件**：`src/entities/arena/BoostPadShadersAndMeshes.ts`、`src/game/runtime/RuntimeResourceRegistry.ts`、`scripts/runDevBroadcastReproV3.mjs`
- **必查问题**：
  1. `bug-4` 在一次运行中出现 `maxSinglePadFlips: 2`（同一帧跳 110 ticks 时 pad 吃掉又重生属正常物理状态变化还是闪烁？）。
  2. `bug-5` 两次运行一次 `retainedGeometries: 0`、一次 `retainedGeometries: 10`：确认这 10 个几何体是否源于 Track 2 新增的 `focusedCarIndex` 切换（切车后产生了延迟清理的临时视觉对象），还是 GC/异步清理时序差异。

---

## 三、汇合与验收标准

4 个小队各自输出**带真实打点数据的定位结论**后，统一汇总再一次性合并修复，最终验收标准：
1. `pnpm run check`：GREEN（1430/1430 TS，0 errors）
2. `pnpm run test:multiplayer-suite`：28/28 PASS
3. `pnpm run repro:dev-broadcast-v3`：**连续运行 3 次均达到 `reproduced=0/5`**（消除 Bug 3 与 SwiftShader 偶发抖动）
