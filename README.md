# AA-Track A-1 完成报告：Lookahead 单调 Tick 与平滑收敛修复

## 1. 分支与提交信息
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base` (`7435735`)
- **目标分支**：`fix/bug3-lookahead-monotonic-tick`
- **Verified Bot Commit**：`a657c4920f82bde403ee08fe837b3d7660ddb5af`（`verified: true`，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）
- **独占修改文件**：`src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`

## 2. 核心改造内容
1. **禁止 stale / 低 tick 快照回退已推进的 lookahead `tickCount`**：
   - 在 `SpectateLookaheadState` 中新增 `lastAppliedTick` 与 `lastSampledTick`（在 `resetSpectateLookaheadState` 时同步清零）。
   - 对 `sampled.renderTick + 0.5 < state.lastSampledTick` 的乱序过期样本直接拒绝，不覆盖本地状态。
   - 新增 `enforceMonotonicTick` 与 `getCurrentEngineTick`，在非 lookahead 快照分支及回退分支中均以 `Math.max(16, currentEngineTick, state.lastAppliedTick, rawSampledTick)` 作为单调下界，防止 `applyReplayFrameToEngine` 或 `sampled.renderTick` 将已推进的 `SIM_OFFSETS.TICK` 拉回低值。
   - 在 lookahead 步进分支中，基于 `Math.max(16, getCurrentEngineTick(physics), state.lastAppliedTick)` 推进 `nextTick` 并同步更新 `lastAppliedTick`。
2. **平滑收敛而非直接覆写回弹**：
   - 当 `rawSampledTick < monotonicFloor`（网络快照滞后于本地 lookahead 进度）时，采用更温和的收敛系数 `SPECTATE_CONVERGENCE_BEHIND_ALPHA = 0.2`（正常追上时为 `SPECTATE_CONVERGENCE_ALPHA = 0.3`），避免向旧 tick 位置剧烈回拉。
   - 将 `SPECTATE_CONVERGENCE_DIST_SQ` 从 `900^2` 放宽至 `1800^2`，减少高速球运动下的硬切回弹。
   - 收敛帧的 `converged[SIM_OFFSETS.TICK]` 同步锁定为 `monotonicTick`。
3. **放宽 `SPECTATE_LOOKAHEAD_MAX_TICKS` 与分段衰减，避免 26 帧冻结**：
   - 将 `SPECTATE_LOOKAHEAD_MAX_TICKS` 从 `30`（250ms）放宽至 `180`（1.5s @ 120Hz），避免短时断流/抖动触顶后 `return false` 导致连续 26 帧冻结。
   - 新鲜帧到达时对 `lookaheadTicksActive` 进行分段快速衰减（追上时直接减半/归零，滞后时 `- 4` 替代 `- 2`），防止长期贴近上限。

## 3. 门禁与验证结果
- `pnpm run check`：**PASS**（TS 类型检查与循环依赖检查通过）
- `pnpm run test:multiplayer-suite`：**PASS**（含 Test 26 `client-csr-spectate Physics Lookahead & Convergence` 与 Test 27 `W2-A isLookahead WASM lookahead wiring` 全部通过）
- AA-Track A-1 专项不变量验证：
  - 连续 40 帧 lookahead 步进（> 30 ticks）无冻结，持续推进；
  - 推进至高位 tick 后收到低 `renderTick`（719）快照不发生 `tickCount` 回退；
  - 乱序退步 `renderTick`（718 < 719）被拒绝；
  - `resetSpectateLookaheadState` 后可正常从初始 tick（16）重新开始。
