# RocketCompanion 基线合并与下一阶段并行计划

## 1. 基线合并与工具链验证结果

- **分支合入与签名推送**：已将 `origin/fix/bug1-goal-replay-client-dynamics` (`4364561`, Track B) 与 `origin/fix/bug3-lookahead-wiring` (`ed108be`, Track A) 无冲突合并至 `origin/z/agy/rc/multiplayer-recheck-base`。
  - 已通过 GitHub REST API 生成 Verified Bot Merge Commit：`743573520e45ed4b1de47d300ca68fc48c89c2fb`（`verified: true`）。
- **工具链与环境**：已完成 `bootstrap.sh`、tmpfs `node_modules`、`public/` submodule、Headless Chrome WebGL2 及 3v3 `.rcreplay` fixture 生成（OverlayFS 29 MB <= 100 MB 预算）。
- **门禁实测结果（基于 `7435735`）**：
  - `pnpm run check`：**PASS**（1430/1430 TS，0 循环依赖，0 hard errors）
  - `pnpm run test:multiplayer-suite`：**PASS**（全部单测通过；注：Track A 与 Track B 在 `scripts/runMultiplayerSuiteTests.ts` 各自新增了编号为 `27` 的用例，后续可顺手重编号）
  - `pnpm run repro:dev-broadcast-v3`（Headless Chrome + WebGL2 + WASM）：**`reproduced=4/5`**
    - `bug-1-goal-replay-client-static-no-dynamics`: **REPRODUCED**（`enteredReplay: true`, `distinctBallPositions: 1`, `bufFrames: 1`, 球静止在 `0, 0, 93.2`）
    - `bug-2-replay-nametag-and-team-desync`: **REPRODUCED**（`teamMismatch: false`, `arenaCarTeams: [0,0,0,1,1,1]`, 但 `client.tags: []` — Client 未渲染出 NameTag）
    - `bug-3-replay-choppy-motion-no-physics-lookahead`: **REPRODUCED**（`distinctTicks: 9`, `frozenBallFrames: 26`, `maxBallStepUu: 1533.9`, `tick` 在 `719 -> 720 -> 719 -> 723 -> 719` 间倒退振荡）
    - `bug-4-boostpad-model-flicker`: **PASS_FIXED**（`totalPadFlips: 2`, `maxSinglePadFlips: 1`）
    - `bug-5-high-memory-and-unreleased-on-exit`: **REPRODUCED**（`retainedGeometries: 3`，退出后由 `62` 降至 `42` 但未回到基线 `39`）

---

## 2. 对 Track A / Track E 能否再细分及取舍的评估

### Track E：不是空活，不能删
- 实测中 `bug-2-replay-nametag-and-team-desync` 再次复现：`arenaCarTeams` 已同步，但 `client.tags` 为 `[]`。
- 原因指向客户端在 Replay Watch Together 下 NameTag 渲染/更新时机（例如后台标签页 rAF 节流下 `NameTagSystem` 未刷新，或 roster/teams 同步后未立即更新 `_visibleCards2D` / `_offscreenIndicators2D`）。
- **结论**：Track E 必须保留，但本身改动面很小，**无需继续向下拆分**，保持单条轻量 Track 即可。

### Track A（Bug 3）：压力确实偏大，建议拆分为 Track A-1 + Track A-2
- 从 `repro:dev-broadcast-v3` 采样切片可以看到两个互相独立的缺陷叠加：
  1. **Lookahead tick 倒退与 30-tick 上限冻结**：`spectatePhysicsLookaheadPipeline.ts` 在收到非 lookahead 快照时无条件执行 `physics.state[SIM_OFFSETS.TICK] = targetTick`，把已经由 `physics.step(1)` 推进到 `720/723` 的 tick 拉回 `719`；同时 `SPECTATE_LOOKAHEAD_MAX_TICKS = 30`（仅 250ms）触顶后直接 `return false`，导致连续 26 帧冻结。
  2. **Replay Watch Together 广播节拍与插值缓冲协同**：客户端在 Replay Watch Together 模式下仍出现 `1533.9 uu` 的大跳变，说明 `SpectateInterpolationBuffer` 与 Host Replay 广播帧率/去重逻辑之间仍有断层。
- **结论**：Track A 文件边界天然正交，**可以拆分为 Track A-1（Lookahead 状态机）和 Track A-2（缓冲与广播节拍）两条并行线**。

---

## 3. 全新并行推进计划（均从 `origin/z/agy/rc/multiplayer-recheck-base` @ `7435735` 切出）

| Track | 优先级 | 建议分支 | 独占文件边界 | 核心改造与验收目标 |
| :--- | :--- | :--- | :--- | :--- |
| **Track A-1** | P0 | `fix/bug3-lookahead-monotonic-tick` | `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts` | 禁止 stale/低 tick 快照回退已推进的 lookahead `tickCount`；平滑收敛而非直接覆写回弹；放宽或分段重置 `SPECTATE_LOOKAHEAD_MAX_TICKS` 避免 26 帧冻结。 |
| **Track A-2** | P0 | `fix/bug3-replay-buffer-cadence` | `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`<br>`src/network/sync/ReplicationStateInterpolator.ts` | 修复 Replay Watch Together 下 Host 广播与 `SpectateInterpolationBuffer` 采样窗口匹配，消除 `maxBallStepUu > 200`（当前 `1533.9`）跳变。验收：`bug-3` 转 `PASS_FIXED`。 |
| **Track B-2** | P0 | `fix/bug1-goal-replay-snapshot-stream` | `src/game/pipeline/FramePipeline.ts`<br>`src/game/pipeline/stages/SimulationStage.ts`<br>`src/game/modes/FreeplayMode.ts`（或 GoalReplay 广播相关） | 解决 Track B 合入后 Client 进入 Goal Replay 仍只有 `bufFrames: 1`、球静止在 `(0,0,93.2)` 的问题，确保 Host Goal Replay 期间持续向 Client 推流且 Client 正常消费。验收：`bug-1` 转 `PASS_FIXED`。 |
| **Track C-2** | P1 | `fix/bug5-remaining-geometry-leak` | `src/replay/` / `src/entities/` 中 Replay 退出未释放的 3 个 Geometry 来源 | 排查 `onReturnToFreeplay()` 后仍残留 `retainedGeometries: 3`（39 -> 62 -> 42）的资源来源并接入 `clearReplay` 释放。验收：`bug-5` 转 `PASS_FIXED`。 |
| **Track E** | P1 | `fix/bug2-nametag-render-stability` | `NameTagSystem` 相关文件<br>`scripts/runDevBroadcastReproV3.mjs`（仅 Bug 2 采样前窗口激活/等待逻辑） | 修复 Client `client.tags: []` 问题，并在采样前确保 Client 至少完成渲染帧更新；合入后连续执行 3 次 `repro:dev-broadcast-v3` 均稳定 `PASS_FIXED`。 |
