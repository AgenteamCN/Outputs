# 小队 A 排查报告：Bug 3 —— Replay Watch Together 下 Client Lookahead 冻结与跳变定位

- **基线版本**: `origin/z/agy/rc/multiplayer-recheck-base` (`7af0003476afc0b275e34e7abebfd08df018d990`)
- **聚焦文件**:
  - `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
  - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
  - `src/network/sync/ReplicationStateInterpolator.ts`
  - （关联调度）`src/game/RenderClockScheduler.ts`、`src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`

---

## 一、核心结论摘要

在 `7af0003` 上对 Replay Watch Together 进行实机 Headless Chrome + SwiftShader 打点（直接记录 `applySpectateSampleOrLookahead` 每帧分支与 `SpectateInterpolationBuffer.getStats()`），我们确认 Bug 3 在测试窗口中出现的「连续冻结 + 大幅跳变（如 `222 -> 265 -> 375 -> 434 -> 444 -> 750` / `frozenBallFrames: 24`）」由 **三个叠加机制** 共同导致：

1. **`shouldRunLookahead` 在 Replay Watch Together 中已生效，并非标志位漏开**（`simulationNetworkReconcilePipeline.ts:293-298` 的 `isRemoteReplayWatchTogether` 覆盖了 `policy.shouldRunLookahead === false`）。正常收包期间，`applySpectateSampleOrLookahead` 持续走 `1b-snap`（平滑消费 `SpectateInterpolationBuffer` 插值帧）。
2. **前/中段连续冻结（`tick` 不变、`dist < 0.01`）与随后 +43 / +110 / +178 tick 跳变的直接原因**：
   - 在 SwiftShader 双 Tab 并发渲染下，`RenderClockScheduler` 与测试侧 `requestAnimationFrame` 解耦（`MessageChannel` + `setTimeout(..., 1)` + `pending.length < 2` WebGL fenceSync 门控）。实测在 `f:1..f:2`（983ms）和 `f:3..f:5`（280ms）期间，`requestAnimationFrame` 正常触发，但 `SimulationStage` / `applySpectateSampleOrLookahead` **一次都未被调用**（`traceLen` 停留在 `119`、`120` 不动）。
   - 与此同时，Host 发来的 WebRTC 数据包在后台持续到达。当 `SimulationStage` 恢复执行时，`SpectateInterpolationBuffer`（默认容量 `maxFrames = 64`）被突发包填满并从头部淘汰旧帧，`oldestTick` 直接跳升，导致 `sample()` 中 `this.renderTick = Math.max(oldestTick, ...)` 硬切到新 `oldestTick`（实测 `renderTick: 439.9 -> 618`，单帧跳变 **+178 ticks**）。
3. **后段 20+ 帧死锁冻结（如 `tick: 902` 连续冻结 21 帧）的直接原因**：
   - `runDevBroadcastReproV3.mjs` 在 `seekToTime(1.0)` 后需等待 `2500ms` + NameTag 等待 + 两次 `edgeSync`（各 1.2s–5s），到达 Bug 3 采样窗口时，**6 秒（720 tick）合成录像已播至终点 `tick: 719`** 并停止推进。
   - Client 侧 `flags.isReplayPaused` 仅检查本地 `standaloneReplayViewer`，无法感知 Host 录像已暂停/播完，因此 `shouldRunLookahead` 仍为 `true`。
   - `SpectateInterpolationBuffer.sample()` 在 `sinceAdvanceMs > 250ms` 后进入 `cursorState: 'stalled'` 并返回 `isDuplicateFrame: true`；`applySpectateSampleOrLookahead` 随之进入 `2-lookahead-step` 外推 `+180` ticks（`719 -> 902`），触及 `SPECTATE_LOOKAHEAD_MAX_TICKS = 180` 后落入 `3a-max-la-freeze` 永久返回 `false`，导致后续所有帧完全冻结在 `tick: 902`。

---

## 二、逐一回答小队 A 三个必查问题（附实测打点数据）

### 实测打点环境与数据
我们在 `7af0003` 上通过专用诊断脚本（复现 `runDevBroadcastReproV3.mjs` 的 Host/Client Dev Sync Room + Replay Watch Together 流程，`seekToTime(1.0)` 后等待 `800ms` 切 Client 前台并采集 `30` 帧 `rAF` + `applySpectateSampleOrLookahead` 分支追踪）获得以下真实数据：

#### 1. `applySpectateSampleOrLookahead` 到底走了哪个分支？
在 Replay Watch Together 活跃播放阶段（前 80 次调用，`t=112089..113677`，`rTick=99.5..301.4`）：
- 第 1 次调用走 `1a-converge`（`laActive: 0` -> 实际是刚切入时收敛一次，`alpha: 0.3`）。
- 随后 **79 / 80 次调用全部走 `1b-snap`**（`shouldRunLookahead: true, dup: false, isLa: false, cursor: "tracking", withinConv: true`），每 8–40ms 稳定推进 `rTick`（`100 -> 301`）。

```json
{"t":112089,"branch":"1a-converge","shouldRunLookahead":true,"dup":false,"isLa":false,"cursor":"tracking","rTick":99.5,"laActive":0,"engTick":89,"monotonicTick":100}
{"t":112105,"branch":"1b-snap","shouldRunLookahead":true,"dup":false,"isLa":false,"cursor":"tracking","rTick":102.1,"laActive":0,"engTick":100,"monotonicTick":102}
...
{"t":113677,"branch":"1b-snap","shouldRunLookahead":true,"dup":false,"isLa":false,"cursor":"tracking","rTick":301.4,"laActive":0,"engTick":299,"monotonicTick":301}
```

而在 `30` 帧 `rAF` 采样窗口中，我们观测到完整的「冻结 -> 跳变 -> 再次冻结」过程：

```json
{"f":0,"t":114953,"tick":430,"bx":1486.3,"by":-242.1,"bz":112,"cursorState":"tracking","latestHostTick":451,"renderTick":429.5,"interpSamples":208,"snapSamples":11,"dupSamples":8,"traceLen":118}
{"f":1,"t":120575,"tick":440,"bx":1548.7,"by":-260.2,"bz":105.5,"cursorState":"tracking","latestHostTick":453,"renderTick":439.9,"interpSamples":209,"snapSamples":11,"dupSamples":8,"traceLen":119}
{"f":2,"t":121558,"tick":440,"bx":1548.7,"by":-260.2,"bz":105.5,"cursorState":"tracking","latestHostTick":453,"renderTick":439.9,"interpSamples":209,"snapSamples":11,"dupSamples":8,"traceLen":119}
{"f":3,"t":125072,"tick":618,"bx":-382.7,"by":329.1,"bz":171,"cursorState":"tracking","latestHostTick":719,"renderTick":618,"interpSamples":209,"snapSamples":12,"dupSamples":8,"traceLen":120}
{"f":4,"t":125178,"tick":618,"bx":-382.7,"by":329.1,"bz":171,"cursorState":"tracking","latestHostTick":719,"renderTick":618,"interpSamples":209,"snapSamples":12,"dupSamples":8,"traceLen":120}
{"f":5,"t":125352,"tick":618,"bx":-382.7,"by":329.1,"bz":171,"cursorState":"tracking","latestHostTick":719,"renderTick":618,"interpSamples":209,"snapSamples":12,"dupSamples":8,"traceLen":120}
{"f":6,"t":127089,"tick":772,"bx":-1800.2,"by":1071.2,"bz":93.1,"cursorState":"stalled","latestHostTick":719,"renderTick":719,"interpSamples":221,"snapSamples":40,"dupSamples":35,"traceLen":160}
{"f":7,"t":127550,"tick":826,"bx":-2281.4,"by":902.6,"bz":136.3,"cursorState":"stalled","latestHostTick":719,"renderTick":719,"interpSamples":221,"snapSamples":81,"dupSamples":76,"traceLen":201}
{"f":8,"t":127659,"tick":838,"bx":-2399.8,"by":852,"bz":132.5,"cursorState":"stalled","latestHostTick":719,"renderTick":719,"interpSamples":221,"snapSamples":87,"dupSamples":82,"traceLen":207}
{"f":9,"t":129577,"tick":902,"bx":-2998.4,"by":609.8,"bz":107.6,"cursorState":"stalled","latestHostTick":719,"renderTick":719,"interpSamples":221,"snapSamples":169,"dupSamples":164,"traceLen":289}
...
{"f":29,"t":147801,"tick":902,"bx":-2998.4,"by":609.8,"bz":107.6,"cursorState":"stalled","latestHostTick":719,"renderTick":719,"interpSamples":221,"snapSamples":944,"dupSamples":939,"traceLen":400}
```

关键观察：
- **`f:1 -> f:2` 与 `f:3 -> f:5` 的冻结**：`traceLen` 完全没有增长（`119 -> 119`、`120 -> 120`），`interpSamples / snapSamples / dupSamples` 全部静止。说明在这几个 `rAF` 周期内，`SimulationStage` 根本没有执行到 `applySpectateSampleOrLookahead`（SwiftShader 下 `RenderClockScheduler` 的 `MessageChannel`/`setTimeout` 与 `rAF` 调度错位/被长渲染帧阻塞）。
- **`f:2 -> f:3` 的 +178 tick 跳变**：在 `f:2` 到 `f:3` 的 `3.5s` 间隔内，`SimulationStage` 仅执行了 1 次（`traceLen: 119 -> 120`），期间收到的大量数据包把 `SpectateInterpolationBuffer`（`maxFrames = 64`）顶满并淘汰了 `440..617` 的帧，`oldestTick` 变为 `618`，`sample()` 第 458 行 `Math.max(oldestTick, ...)` 强行将 `renderTick` 从 `439.9` 拉到 `618`。
- **`f:6 -> f:9 -> f:29` 的外推触顶冻结**：`latestHostTick` 停在 `719`（6s 录像结束），`cursorState` 转为 `'stalled'`，`dupSamples` 从 `8` 暴增到 `939`。`applySpectateSampleOrLookahead` 走 `2-lookahead-step` 推进到 `719 + 183 = 902` 后，因 `lookaheadTicksActive >= 180` 落入 `3a-max-la-freeze`，`f:9..f:29` 连续 21 帧完全冻结在 `tick: 902`。

---

#### 2. `sampled` 是否返回了 `isDuplicateFrame: false` 的旧帧？`shouldRunLookahead` 在 Replay Watch Together 某个子阶段是否为 `false`？
1. **`shouldRunLookahead` 状态**：
   - 在 Client 端 Replay Watch Together 期间，Client 的 `matchState` 为 `{ mode: "replay-viewer", phase: "lobby", paused: true }`。
   - 由于 `paused === true`，`evaluateSimulationFrameFlags` 将 `flags.shouldHoldAccumulator` 置为 `true`，进而 `classifySimulationDrivePolicy` 返回 **`policy.shouldRunLookahead = false`**。
   - 但 `simulationNetworkReconcilePipeline.ts:293-298` 通过 `isRemoteReplayWatchTogether && !flags.isReplayPaused && !flags.isMatchEnded && !flags.isGoalReplay` 将最终传入 `applySpectateSampleOrLookahead` 的 `shouldRunLookahead` 强制置为了 **`true`**（全周期 80 条 trace 均证实 `shouldRunLookahead: true`）。
   - **反向问题**：由于 Client 侧 `flags.isReplayPaused` 不感知 Host 录像是否已暂停或播完，当 Host 在 `tick: 719` 停止时，`shouldRunLookahead` 依然为 `true`，导致 Client 盲目外推 180 ticks 后卡死。
2. **`isDuplicateFrame` 行为**：
   - 正常跟踪期（`tracking`）`isDuplicateFrame` 为 `false`，返回的是平滑插值帧。
   - 卡顿超过 `250ms`（`SPECTATE_STALL_TIMEOUT_MS`）后，`renderTick` 顶到 `maxRenderTick`，`sample()` 返回 `tail` 且 `isDuplicateFrame: true`（仅首次到达尾帧时为 `false`，之后均为 `true`），从而触发 `2-lookahead-step`。
   - 注意 `ReplicationStateInterpolator.ts:384-387`：若 `wasInterpolated === false` 且恰好有新包刷新了 `frameRevision`（即便 `renderTick` 仍卡在边界），`isDuplicateFrame` 会变为 `false`，导致偶发重复应用边界快照。

---

#### 3. 当 Host 一次性跳 110 ticks 抵达时，`isBallDeltaWithinConvergenceThreshold`（阈值 1800 uu）是否触发硬切瞬移？
1. **代码逻辑分析**：
   - `spectatePhysicsLookaheadPipeline.ts:25` 定义 `SPECTATE_CONVERGENCE_DIST_SQ = 1800 * 1800`。
   - 当 Client 处于 Lookahead 状态（`lookaheadTicksActive > 0`）且 Host 补发到达并跳变 `+110 ticks`（约 `0.917s`）：
     - 若球速 ~2000 uu/s，位移约 `1833 uu > 1800 uu`，**`isBallDeltaWithinConvergenceThreshold` 直接返回 `false`**，跳过 `1a-converge` 指数平滑收敛，直接走 `1b-snap` 执行 `applyReplayFrameToEngine(physics, sampled.snapshot, interpolator)` **硬切瞬移**。
     - 即使位移在 `1739.6 uu`（略小于 `1800 uu`），若此时 Client 根本没进入 Lookahead（`lookaheadTicksActive === 0`，如上文 `f:2 -> f:3` 因 `SimulationStage` 饥饿导致 `maxFrames=64` 淘汰旧帧），`1a-converge` 同样不会触发（条件要求 `state.lookaheadTicksActive > 0`），而是直接走 `1b-snap` 硬切，单帧位移即为两帧间全部差值（`1739.6 uu`）。
2. **`isTrueTeleportJump` 边界**：
   - `ReplicationStateInterpolator.ts:188-196` 中 `maxDist = Math.max(900, tickDelta * 65)`。对于 `tickDelta = 110`，`maxDist = 7150 uu`，因此 `pushFrame` 不会误判为 `this.reset()`；硬切主要来自 **`maxFrames = 64` 淘汰旧帧** 以及 **`1800 uu` 收敛阈值被突破**。

---

## 三、修复建议（供汇总合并阶段参考）

1. **小队 A 侧（`ReplicationStateInterpolator.ts` / `spectatePhysicsLookaheadPipeline.ts`）**：
   - 将 `SpectateInterpolationBuffer` 默认 `maxFrames` 从 `64`（仅 ~0.53s@120Hz）提升至 `240`–`360`（2–3s 缓冲），避免 SwiftShader 下 `SimulationStage` 偶发卡顿 1–2s 时旧帧被淘汰引发 `renderTick` 单帧硬跳 +100~+180 ticks。
   - 在 `2-lookahead-step` 中放宽或分段处理 `dtMs` 上限（当前硬限 `50ms` / 4 ticks），避免从渲染长帧恢复时 Lookahead 进度严重滞后于真实时间。
   - 同步 Host 录像暂停/播完状态到 Client（或在 `tick === durationTicks - 1` 时停止外推），避免播完后继续外推 180 ticks 触发 `3a-max-la-freeze`。
2. **联动小队 B / 调度侧**：
   - 将合成 3v3 录像默认时长从 `6s`（720 ticks）提升至 `12s–15s`，避免 `runDevBroadcastReproV3.mjs` 在完成 NameTag 与 `edgeSync` 验证后，Bug 3 采样窗口恰好撞上 `tick: 719` 播完边界。
   - 检查 SwiftShader 下 `RenderClockScheduler` 的 `simulate` 调度：确保即使 `render()` 耗时较长或 `pending.length < 2`，在 `rAF` 之间也能稳定推进 `SimulationStage`，不出现连续 2–4 个 `rAF` 完全不调用 `applySpectateSampleOrLookahead` 的饥饿现象。
