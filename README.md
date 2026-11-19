# Wave 2 B-Track（Bug 3 / 2-B）完成报告

- **分支**：`z/agy/rc/multiplayer-recheck`
- **基线 Commit**：`1114df1`
- **已合入 Verified Bot Commit**：[`4ec70dc6efce64790c2f560b20580ffd1c2c695f`](https://github.com/Antigravi-ty/RocketCompanion/commit/4ec70dc6efce64790c2f560b20580ffd1c2c695f)（`signature.isValid = true, state = VALID`）

## 1. 实现内容

1. **策略表开启前瞻（`src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`）**
   - 在 `classifySimulationDrivePolicy` 的 `client-csr-spectate` 分支中，仅当未处于暂停/结束/Goal Replay/冻结状态时开启 `shouldRunLookahead: true`。
2. **本地 WASM 物理前瞻与平滑收敛（`src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts` & `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`）**
   - 仅依赖 `SpectateInterpolationBuffer` 现有契约（`sample()`、`isDuplicateFrame`、`renderTick`），无跨轨道冲突。
   - **正常插值帧（`!sampled.isDuplicateFrame`）**：同步更新 `physics` 与 `interpolator` 状态，并保证 `SIM_OFFSETS.TICK` 单调跟随 `sampled.renderTick` 推进（避免 `preserveEngineTick: true` 导致客户端 `tickCount` 冻结）。
   - **重复帧 / 缓冲预热（`sampled === null || sampled.isDuplicateFrame`）**：在 `policy.shouldRunLookahead === true` 且已收到首帧后，注入最近 `acknowledgedControls` 并按 120Hz 步进本地 WASM 物理（`physics.step(1)`，上限 30 ticks），同步更新 `interpolator.prevState` / `currState` 与单调 `tickCount`。
   - **前瞻恢复收敛**：当非重复帧恢复到达时，若球体位移差在阈值内（`<= 900uu`），通过 `interpolateReplaySnapshots(..., 0.45)` 平滑收敛回权威插值快照；超出阈值（如重置/传球瞬移）则立即对齐。
   - 为严格满足 Radar 单文件体量（`<= 18KB`）与布尔复杂度门禁（`<= 15`），将纯前瞻状态机抽离至 `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`。
3. **单元测试（`scripts/runMultiplayerSuiteTests.ts`）**
   - 新增第 26 项测试 `client-csr-spectate Physics Lookahead & Convergence (Bug 3 / B-Track)`，覆盖策略开关、重复帧触发 `physics.step` 与 `setCarControls`、以及恢复后的平滑收敛。

## 2. 验证结果

- **工具链**：全局安装 `pnpm@12.21.0`，完成 `pnpm install --no-frozen-lockfile --ignore-scripts`。
- **`pnpm run check`**（Radar + `tsc --noEmit`）：**PASS (GREEN)**（1429/1429 TS 文件，0 循环依赖，0 未托管缓存，`BoolHotspots=7`）。
- **`pnpm run radar:audit`**：**PASS (GREEN)**。
- **`pnpm run test:multiplayer-suite`**：**PASS (26/26)**。
- **`npx prettier --check`**：修改文件全部格式合规。
- **环境限制**：`pnpm run repro:dev-broadcast-v3` 需具备 GPU/WebGL 渲染能力的 Runner 在与 A-Track 合入后统一执行完整端到端验收。
