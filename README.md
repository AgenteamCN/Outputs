# Wave 1 `chore/strip-legacy-spectate` 执行报告

- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base` (`2fc5fc0`)
- **目标分支**：`chore/strip-legacy-spectate`
- **Verified Bot Commit**：[`7c80b445f2953076e5356320c6e91bc04da4bba8`](https://github.com/Antigravi-ty/RocketCompanion/commit/7c80b445f2953076e5356320c6e91bc04da4bba8)（`signature.isValid: true`, `state: "VALID"`）

## 变更摘要

1. **删除旧观战 Lookahead 管线**：
   - 删除 `src/network/sync/spectateLookaheadPipeline.ts`
   - 删除 `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
2. **剥离 `src/network/sync/ReplicationStateInterpolator.ts` 单调 tick / 外推 / Lookahead 耦合**：
   - 移除 `ReplicationStateInterpolator.pushSample` 中的 `snapshot.tick < last.tick` 清空缓冲逻辑。
   - 将 `SpectateInterpolationBuffer` 精简为无 Lookahead、无外推、无单调 tick 假设的干净空壳透传缓冲（`pushFrame` / `sample` / `reset` / `getStats`），天然兼容正放、倒放（Rewind）、暂停与跳帧。
3. **清理仿真驱动分类与网络同步管线旧补丁**：
   - `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`：移除 `SimulationDrivePolicy.shouldRunLookahead` 及相关分支判断。
   - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`：移除 `spectatePhysicsLookaheadPipeline` 引用、`lastAcknowledgedControls` lookahead 状态以及 `applySpectateSampleOrLookahead` 调用；Pure CSR 观战流直接通过 `spectateInterpolationBuffer.sample(now)` + `applyReplayFrameToEngine` 应用帧。
   - 核查 `src/game/pipeline/stages/SimulationStage.ts` 与 `src/ui-core/stores/hudStore.ts`，确认无残留 Lookahead 观战临时补丁。
4. **同步更新测试套件**：
   - `scripts/runMultiplayerSuiteTests.ts`：更新 Section 16（验证 `SpectateInterpolationBuffer` 空壳透传与倒放非单调 tick 兼容性）与 Section 26（验证 Pure CSR 观战不触发本地 WASM 物理 Lookahead 步进）。

## 门禁与测试验证

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（`1428/1428` TS 文件，0 循环依赖，0 类型错误）
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**PASS**（28/28 项全部通过）
