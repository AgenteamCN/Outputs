# Track C 完成报告：Bug 5 — Replay 退出后残留 Geometry 与资源作用域清理

## 1. 分支与提交信息
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck` (`7252e42`)
- **工作分支**：`fix/bug5-replay-geometry-leak`
- **Verified Bot Commit**：[`559fbafc705534bdc7c43dd55753867f3498444f`](https://github.com/Antigravi-ty/RocketCompanion/commit/559fbafc705534bdc7c43dd55753867f3498444f) (`signature.isValid: true`, `state: "VALID"`, 含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)

## 2. 根因定位
通过 Headless Chrome + WebGL2 + CDP 实测追踪 `StandaloneReplayViewerController` 加载 3v3 `.rcreplay` 到退出回 Freeplay 期间 `renderer.info.memory.geometries` 与场景对象的变化，定位到以下残留来源与生命周期缺口：
1. **多车 `FlipResetVisual` 残留（3~4 个 Geometry/车）**：
   - 3v3 回放期间会通过 `runtime.getFlipResetVisual(carIndex)` 在 `runtime.flipResetVisuals` (`Map<number, FlipResetVisual>`) 中为多辆车按需创建 `FlipResetVisual`（包含 `RingGeometry`、描边 `RingGeometry`、`ShapeGeometry`、`BufferGeometry` 粒子）。
   - 此前 `resetArenaAfterClear` 仅尝试清理单个 `r.flipResetVisual`（且未从 `flipResetVisuals` Map 中移除），导致回放中触发过 Flip Reset 的其余车辆（`carIndex >= 1`）或已触发车辆的 `FlipResetVisual` Geometry 未被 `dispose()`，造成退出后 `retainedGeometries` 偶发残留 `1~3+`。
2. **`clearReplay()` 直接退出路径未触发 `RuntimeResourceRegistry` `'replay'` 作用域清理**：
   - `StandaloneReplayViewerController.clearReplay()` 调用 `ReplayViewerCausalSyncController.resetArenaAfterClear(runtime)`，但此前仅在 `MatchModeTransitionBridge.returnToFreeplay` / `applyCleanFreeplaySubsystemsReset` 中调用了 `runtimeResourceRegistry.disposeScope('replay')`。若通过 UI 直接清除 Replay，`'replay'` 作用域下的 WASM replay buffer、renderLists、`THREE.Cache` 等不会释放，且 `networkReconciler.setReplayPlaybackMode(false)` 也未重置。
3. **`RuntimeResourceRegistry.disposeScope` 缺少重入保护与快照遍历**：
   - 当 `'replay'` 作用域内的资源 `dispose()` 间接触发 `disposeScope('replay')` 时需要重入保护（`activeDisposingScopes`）与集合快照遍历。

## 3. 代码改动（严格控制在 Track C 独占文件边界内）
1. `src/game/runtime/RuntimeResourceRegistry.ts`
   - 在 `disposeScope(scope)` 增加 `activeDisposingScopes` 重入保护、`Array.from(this.resources.values())` 快照遍历及 `try/finally` 清理。
2. `src/replay/ReplayViewerRosterSubController.ts`
   - 在 `ReplayViewerRosterRuntimePort` 补充 `flipResetVisuals` Map 类型声明。
   - 新增并导出 `disposeRuntimeFlipResetVisuals(runtime)`，遍历 `runtime.flipResetVisuals`，对每个条目执行 `stopVisual?.()`、`dispose?.()` 并从 Map 中 `delete(carIdx)`（无 Map 时回退 `flipResetVisual?.dispose?.()`）。
3. `src/replay/ReplayViewerCausalSyncController.ts`
   - 在 `resetArenaAfterClear(runtime)` 中：
     - 调用 `disposeRuntimeFlipResetVisuals(r)` 清理全部车辆的 `FlipResetVisual` 及其 Geometry/Material；
     - 补充清理 `r.arena?.supersonicWheelTrailSystem?.clear?.()` 与 `r.arena?.resetBallTrail?.()`；
     - 重置 `networkReconciler?.setReplayPlaybackMode?.(false)`；
     - 调用 `void runtimeResourceRegistry.disposeScope('replay')`，确保直接 `clearReplay()` 路径同样触发 `'replay'` 作用域资源释放。

## 4. 验证结果
- `pnpm exec prettier --write`（已格式化全部 3 个修改文件）
- `pnpm run check`（`radar.ts` + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 hard errors，全部文件满足 <= 18.0 KB / 800L 体积上限）
- `pnpm run test:multiplayer-suite`：**26/26 PASS**
- `pnpm run repro:dev-broadcast-v3`（Headless Chrome + WebGL2 + WASM E2E）：
  - `bug-5-high-memory-and-unreleased-on-exit`: **PASS_FIXED**
    - `bootMem`: `{"jsUsedMB":162,"wasmMB":64,"geometries":39,"textures":13}`
    - `loadedMem`: `{"jsUsedMB":234,"wasmMB":64,"rvFrames":720,"geometries":63,"textures":13}`
    - `afterExitMem`: `{"jsUsedMB":229,"wasmMB":64,"rvFrames":0,"geometries":32,"textures":13}`
    - `retainedWasmMB: 0`, `retainedGeometries: -7` (`<= 0`), `tauriCloseHidesWindow: false`
