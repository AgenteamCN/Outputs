# C-Track W3 交付报告：Bug 4（BoostPad 模型闪烁防抖与重复帧跳过）

## 1. 基线与工作分支
- **基准 Commit**: `2565774` (`merge(wave0): octopus merge Arch-0A, Arch-0B, Arch-0C, Arch-0D into z/agy/rc/multiplayer-recheck`)
- **分支**: `z/agy/rc/multiplayer-recheck`
- **轨道范围**: **C-Track W3**（仅修改 `src/entities/BoostPadSystem.ts` 与 `scripts/runMultiplayerSuiteTests.ts`，与 A-Track `ReplicationStateInterpolator.ts` 及 B-Track `SimulationStage.ts` / `simulationNetworkReconcilePipeline.ts` / `simulationFrameClassifierPipeline.ts` 完全正交无冲突）

## 2. 根因分析与实现改动

### 修改文件
- `src/entities/BoostPadSystem.ts`
- `scripts/runMultiplayerSuiteTests.ts`

### 核心修复（`src/entities/BoostPadSystem.ts`）
1. **重复帧跳过（Duplicate Frame Skip）**:
   - `update()` 新增可选参数 `options?: { isDuplicateFrame?: boolean }`，同时通过 `_lastStateTick`（对比 `stateBuffer[SIM_OFFSETS.TICK]`）与 pad `(isActive, cooldown)` 快照状态一致性自动识别重复帧。
   - 当 `isDuplicateFrame === true` 时，跳过状态机边沿检测、`padCooldownTimers` 本地倒计时扣减与 `startCollectAnimation()` 触发，仅推进已存在的视觉动画，彻底消除静止/重复帧导致的 `pad.full.visible` 翻转。
2. **冷却微抖阈值保护与 `respawnedAndCollected` 严格化**:
   - 收紧 `respawnedAndCollected` 判定条件，要求 `prevBufferCd <= 1.0 && bufferCooldown >= fullTimeout - 1.0` 且 `bufferCooldown > prevBufferCd + 0.5`，防止网络抖动或回滚造成的冷却微幅上跳误触发收集动画。
   - 当 `!isActive` 时增加单调微抖保护：忽略 `<= 0.5s` 的小幅向上冷却抖动，禁止已进入 refill 区间（`<= threshold`）的计时器因微抖回弹至 `> threshold`，并将本地外推下限钳制在 `Math.max(0.05, bufferCooldown - 0.35)`（防止 `padCooldownTimers` 提前归零后又被远端 `bufferCooldown` 拉回，引发 `0 <-> >0` 锯齿闪烁）。
3. **复活清理残留动画**:
   - 当 pad 从 `!wasActive` 恢复为 `isActive` 时，主动清理残留的 `collectAnimations.delete(i)`，避免 `pad.base.visible` 与 `pad.full.visible` 短暂错态。

## 3. 验证结果
- **`pnpm run check`（`radar` + `tsc --noEmit`）**: **PASS (GREEN)**
  - `1428/1428` TS 文件，0 Engine JS，`tsc --noEmit` clean，`NameTagScatter=1/1`，`BoolHotspots=7`，`UnscopedCaches=0`
- **`pnpm run radar:audit`**: **PASS (GREEN)**
- **`pnpm run test:multiplayer-suite`**: **PASS (25/25 全部通过)**
  - 新增第 25 项测试 `25. BoostPadSystem Duplicate Frame Skip & Cooldown Micro-Jitter Debounce (Bug 4 / C-Track)`，覆盖重复帧零翻转、冷却微抖防回弹、真实满周期重收集触发与复活清理。
- **环境说明**: 当前容器无 GPU/WebGL 渲染环境，未执行 `pnpm run repro:dev-broadcast-v3`；待 A/B/C 三路合并后在具备 WebGL 渲染能力的 Runner 执行完整 V3 回归门禁。
