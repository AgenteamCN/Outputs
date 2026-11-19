# Track C Deliverable — Bug 5 (`bug-5-high-memory-and-unreleased-on-exit`)

- **Repository**: `https://github.com/Antigravi-ty/RocketCompanion`
- **Base Branch**: `origin/z/agy/rc/multiplayer-recheck` (`943b749cf1c1abbd651c5862d91edecceac360bd`)
- **Branch**: `fix/bug-5-memory-release-and-tauri-close`
- **Verified Bot Commit**: [`0bc5c69034f4df560636be9b2edccb9be97f99e4`](https://github.com/Antigravi-ty/RocketCompanion/commit/0bc5c69034f4df560636be9b2edccb9be97f99e4) (`signature.isValid: true`, `state: VALID`)

## Root Causes & Fixes

1. **Eager 192 MB WASM Boot Allocation & Unreclaimed WASM Linear Memory**:
   - `RocketSimArenaBarrierController.initEngine()` previously called `_physics_initReplayBuffer()` unconditionally at boot, growing `core.wasm` linear memory from **64 MB** to **256.25 MB** (+192 MB) before any replay was used.
   - Removed the eager `_physics_initReplayBuffer()` call at boot, tracked `initialWasmHeapBytes`, and added `reclaimWasmMemoryIfGrown()` (with `clearCachedRocketSimWasmModule()` in `RocketSimWasm.ts` and `RocketSimPhysicsEngine.reclaimWasmMemoryIfGrown()`) to atomically swap in a fresh 64 MB WASM instance when returning to Freeplay if the heap had grown.
   - Guarded `RocketSimWasmStateAndReplayBase.recordReplayFrame()` when `_physics_getReplayBufferSize() === 0` so Freeplay/diagnostic ticks do not re-allocate the 192 MB replay buffer after reclaim.

2. **Unreleased Three.js Geometries & JS Replay State on Exit**:
   - `ReplayViewerCausalSyncController.resetArenaAfterClear()` and `freeplayKickoffAndArenaResetPipeline.resetFreeplayExtraCarsAndArenaToSingleCar()` now remove extra vehicle slots (`vehicleRegistry.removeCar(i)`), reset Car 0 boost/supersonic trails and `ballSpeedTrail`, and dispose `renderer.renderLists`.
   - `ArenaVehicleLoadoutManager.disposeObjectGeometryAndMaterials()` + `arenaVehicleSlotPipeline.ts` now dispose `-shell` GLB geometries unless still referenced by another active car slot (`keepGeometryUuids`).
   - Added WebGL `geometry.dispose()` on reset/stop in `SupersonicWheelRibbon.reset()`, `SpeedTrail.reset()`, `PerformanceBoostTrailVisual.reset()`, `VehicleBoostEmitter.resetVisual()`, and `FlipResetVisual.stopVisual()`.
   - Cleared `FreeplayMode.replayRingBuffer` in `FreeplayGoalAndTouchSubController.applyCleanFreeplayReset()` and reset `resolvedRoster: []` in `StandaloneReplayViewerController.clearReplay()`.

3. **Tauri Close Behavior (`src-tauri/src/lib.rs`)**:
   - Removed unconditional `WindowEvent::CloseRequested` `api.prevent_close(); window.hide();` so closing the window destroys the WebView/Three.js/WASM/Worker resources instead of keeping them resident in memory.
   - Switched macOS activation policy sync to `WindowEvent::Destroyed`, and moved `tray_only_mode` retention to `tauri::RunEvent::ExitRequested` (`code.is_none()`).

## Verification Results

- **`pnpm run check`**: PASS (`tsc --noEmit` + `radar.ts` 1425/1425 TS, 0 circular deps, 0 volume violations).
- **`cargo check` (`src-tauri`)**: PASS (`rustc 1.100.0`, 0 errors).
- **`pnpm run repro:dev-broadcast-v3`**:
  - `bug-5-high-memory-and-unreleased-on-exit`: **`PASS_FIXED` (`reproduced: false`)**
  - Baseline -> Current:
    - `bootMem.wasmMB`: `259 MB` -> **`64 MB`**
    - `afterExitMem.wasmMB`: `310 MB` -> **`64 MB`** (`retainedWasmMB: 0`)
    - `afterExitMem.geometries`: `105` -> **`33`** (`retainedGeometries: -7 <= 0`, down from `65`)
    - `tauriCloseHidesWindow`: `true` -> **`false`**
    - `loadedMem.jsUsedMB -> afterExitMem.jsUsedMB`: `410 MB -> 380 MB` (3v3 replay frames released; remaining JS heap above boot is the active Dev Sync Room `AuthoritativeServerHost` session kept alive by `onReturnToFreeplay()`).
