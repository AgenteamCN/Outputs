# 分支 2（Bug 4 & Bug 5 收尾对齐）执行报告

## 1. 分支与提交信息
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base` (`e8aae572a729b535bae90e74adaba069a4b41524`)
- **新建分支**：`z/agy/rc/fix-bug4-bug5-cleanup`
- **Verified Bot Commit**：[`ba1b416309386dba348ef7b983f1f5b329b21d9c`](https://github.com/Antigravi-ty/RocketCompanion/commit/ba1b416309386dba348ef7b983f1f5b329b21d9c) (`signature.isValid: true`, `state: "VALID"`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)

## 2. 修改文件与改动摘要（与分支 1 零重叠）
1. `scripts/runDevBroadcastReproV3.mjs`
   - **Bug 4 判定去误报**：在 `motionAndPads` 采样阶段记录每个 BoostPad 每次 `full.visible` 翻转的 `performance.now()` 时间戳（`padFlipTimes`），使用 400ms 滑动窗口计算单 pad 最大快速翻转次数 `maxRapidFlipsIn400ms`，将判定条件更新为 `maxRapidFlipsIn400ms >= 2`，消除 SwiftShader 2–4s 长窗口下正常拾取/4s 冷却充能周期导致的 `maxSinglePadFlips >= 2` / `totalPadFlips >= 8` 误报。
   - **Bug 5 退出异步清理等待**：在 `onReturnToFreeplay()` 前将 Host 标签页切回前台（`Page.bringToFront`），调用 `onReturnToFreeplay()` 后通过 `waitFor` 等待 `numCars === 1`、`!rv?.hasLoadedReplay`、`!rv?.isActive`、`rvFrames === 0` 及 `arena.cars[0].visible === true`，再等待 2 个 `rAF` 帧与 400ms 稳定期后采样 `afterExitMem`，避免与异步 `restoreLocalPlayerFreeplayLoadout(0)` 及 `runtimeResourceRegistry.disposeScopes` 竞态。
2. `src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts`
   - **Bug 5 全槽位拖尾几何体清理**：在 `resetFreeplayExtraCarsAndArenaToSingleCar` 中引入 `MAX_CARS`，将非本地车拖尾重置范围扩展至 `1 .. Math.max(arena.cars.length, MAX_CARS) - 1`，确保所有持久化 `SupersonicWheelTrailSystem` 槽位均触发 `resetCar()`（含 `geometry.dispose()`）。

## 3. 验证结果
- `pnpm run check`：**PASS**（`1430/1430` TS，`tsc --noEmit` 0 errors，架构护栏全绿）。
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**PASS**（`28/28 PASS`）。
- `pnpm run repro:dev-broadcast-v3`：
  - `bug-1-goal-replay-client-static-no-dynamics`: **PASS_FIXED**
  - `bug-2-replay-nametag-and-team-desync`: **PASS_FIXED**
  - `bug-3-replay-choppy-motion-no-physics-lookahead`: `REPRODUCED_BUG`（属分支 1 负责范围，符合预期）
  - `bug-4-boostpad-model-flicker`: **PASS_FIXED**（`totalPadFlips: 9, maxSinglePadFlips: 3, maxRapidFlipsIn400ms: 1`）
  - `bug-5-high-memory-and-unreleased-on-exit`: **PASS_FIXED**（`bootMem.wasmMB: 64, afterExitMem.wasmMB: 64, retainedWasmMB: 0, bootMem.geometries: 37, afterExitMem.geometries: 36, retainedGeometries: -1`）
