# Wave 0 Arch-0C 完成报告：资源生命周期总线（RuntimeResourceRegistry）

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **基线分支**: `origin/z/agy/rc/multiplayer-recheck` (`d69e01c`)
- **新建分支**: `z/agy/rc/wave0-arch-0c`
- **已验证签名提交**: [`fa35923`](https://github.com/Antigravi-ty/RocketCompanion/commit/fa35923049c8506b921898236977b924629d58a8) (`signature.isValid: true`, `state: VALID`)

---

## 一、核心改动

1. **新建统一资源生命周期总线**
   - `src/game/runtime/RuntimeResourceRegistry.ts`
   - 定义 `RuntimeResourceScope = 'session' | 'replay' | 'ai'` 与 `RuntimeDisposableResource` 接口（`id`, `scope`, `dispose()`）。
   - 提供 `RuntimeResourceRegistry` 类与全局单例 `runtimeResourceRegistry`，支持 `register`、`unregister`、`has`、`getRegisteredIds`、`disposeScope`、`disposeScopes`、`disposeAll`（同步资源立即释放，异步资源如 WASM 内存回收自动 `Promise.all` 等待）。

2. **清理全库 UnscopedCaches（`radar:audit` 指标 `3 -> 0`）并接入总线**
   - `src/ai/experimental/SafeTensorsLoader.ts`：新增 `clearSafeTensorsModelCache()`，注册为 `ai:safetensors-model-cache`（`scope: 'ai'`）。
   - `src/ai/experimental/wasmSimdMatVec.ts`：新增 `resetWasmSimdMatVec()`，重置 `sWasmMem` / `sWasmMatVec` / `sF32View` / `sWeightPtrMap` / `sPendingTensors` / `sInitPromise`，注册为 `ai:wasm-simd-matvec`（`scope: 'ai'`）。
   - `src/entities/vehicle/vehiclePaintMaterial.ts`：新增 `clearVehiclePaintMaterialCaches()`，清空 `customToonRampCache` / `carPaintCache` / `activeAnodizedMaterials` 并注册为 `entities:vehicle-paint-material-cache`（`scope: 'session'`）；同时注册 `three:global-cache`（`THREE.Cache.clear()`，`scope: 'replay'`）。
   - `src/replay/ReplayDirectorCsvParser.ts`：将静态表头集合 `STRING_CSV_HEADERS` 显式标注为 `ReadonlySet<string>`。

3. **统一退出回收入口（接入 `0bc5c69` 已有释放能力）**
   - `src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts`：
     - 新增 `registerFreeplayRuntimeResources(...)`，集中注册 `freeplay:replay-ring-buffer`、`physics:wasm-replay-buffer`（`clearReplayBuffer` + `reclaimWasmMemoryIfGrown`）、`renderer:render-lists`（`renderLists.dispose()`）至 `scope: 'replay'`。
     - 在 `applyCleanFreeplaySubsystemsReset(...)` 末尾统一调用 `runtimeResourceRegistry.disposeScope('replay')`。
   - `src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts`：在 `attach()` 时调用 `registerFreeplayRuntimeResources(...)`。
   - `src/game/modes/FreeplayMode.ts`：移除 `resetToCleanFreeplay()` 中重复分散的内联清理调用，并保持文件体积控制在 `17.9 KB`（低于 `18.0 KB` 硬上限）。
   - `src/game/runtime/MatchModeTransitionBridge.ts`：在 `returnToFreeplay()` 中统一触发 `runtimeResourceRegistry.disposeScopes(['replay', 'ai', 'session'])`。

---

## 二、验证结果

| 检查项 | 结果 | 关键输出 |
| --- | --- | --- |
| `pnpm run check` (`radar` + `tsc --noEmit`) | **PASS** | `1426/1426` TS，`0 Circular Deps`，`0 Upward Imports`，`UnscopedCaches=0` |
| `pnpm run radar:audit` | **PASS** | `[C]` 未托管缓存降为 `0`（原 `3`） |
| `pnpm run test:multiplayer-suite` | **PASS** | `24/24` 全部通过 |
| `pnpm run build` (`vite build`) | **PASS** | 打包退出码 `0` |
| `RuntimeResourceRegistry` + `SafeTensorsLoader` Node 实测 | **PASS** | `disposeScope('ai')` 后 `getCachedSafeTensors('test-model') === null` |
