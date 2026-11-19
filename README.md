# Wave 1 Track C — Bug 5 (`bug-5-high-memory-and-unreleased-on-exit`) Fix Report

- **Branch**: `fix/bug-5-memory-release-and-unreleased-on-exit` (based on `origin/z/agy/rc/multiplayer-recheck`)
- **Commit**: `8c1edd9` (`fix(bug-5): reclaim WASM/JS/Three.js memory on replay exit and close Tauri window instead of hiding`)
- **Push Status**: Local commit created; `git push` could not complete because no GitHub credentials/SSH keys are configured in the sandbox (`terminal prompts disabled`).

---

## Root Causes & Implemented Fixes

### 1. Boot & Runtime WASM Memory (`259 MB -> 64 MB`, `retainedWasmMB: 51 -> 0`)
- **`src/physics/RocketSimArenaBarrierController.ts`**: Removed eager `mod._physics_initReplayBuffer?.()` at boot in `initEngine()`, which previously pre-allocated ~194.5 MB immediately (`core.wasm` 64 MB -> 259 MB).
- **`src/network/server/PhysicsEngineManager.ts`**: Guarded `this.sim.initReplayBuffer?.()` in constructor and `init()` behind `if (this.isRecordingFullReplay)`.
- **`src/physics/RocketSimWasm.ts` & `src/physics/RocketSimPhysicsEngine.ts`**: Exported `clearCachedRocketSimWasmModule()` and added `reclaimWasmMemoryIfGrown(baselineBytes = 64 * 1024 * 1024)` to re-instantiate `core.wasm` with cached `collisionData` if `HEAPU8.byteLength` grew beyond baseline.
- **`src/game/runtime/MatchModeTransitionBridge.ts`**: Invoked `domain.physics?.reclaimWasmMemoryIfGrown?.()` in `returnToFreeplay()`.

### 2. Three.js Geometry & Asset Disposal (`retainedGeometries: +65 -> -5`)
- **`src/entities/ArenaWorld.ts` & `src/entities/arena/arenaVehicleSlotPipeline.ts`**: Exposed `removeCar(carIndex)` and updated `disposeAndRemoveRegistrySlot` to dispose unshared geometries/materials and trim trailing `null` car slots.
- **`src/entities/arena/ArenaVehicleLoadoutManager.ts` & `src/entities/arena/VehicleFactory.ts`**: Added `collectReferencedGeometries(cars, excludeIndex)` and updated `disposeObjectGeometryAndMaterials` / `removeCarMeshes` so unshared `*-shell` and body/wheel geometries are disposed when no active car references them.
- **`src/entities/arena/RespawnGhostPreview.ts`**: Updated `_disposeRoot()` to traverse and dispose unshared geometries in addition to materials.
- **`src/entities/vehicle/customGlbCarModel.ts`**: Added `pruneCachedCustomVehicleAssets(keepIds, sharedGeometries)` to dispose and evict unused custom GLB vehicle assets.
- **`src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts` & `src/replay/ReplayViewerCausalSyncController.ts`**: Updated `resetFreeplayExtraCarsAndArenaToSingleCar` / `resetArenaAfterClear` to call `removeCar(i)` (instead of only `visible = false`) and trim `carTeams`.

### 3. JS Heap & AI Model / Buffer Cleanup (`retainedJsMB: 227 -> 91`)
- **`src/ai/experimental/wasmSimdMatVec.ts` & `src/ai/experimental/SafeTensorsLoader.ts`**: Added `resetWasmSimdMatVec()` and `clearSafeTensorsModelCache()` to release cached SafeTensors weights (~114 MB) and `simdMatVec.wasm` linear memory (~120 MB) on clean freeplay return.
- **`src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts`**: Cleared `replayTimelineController?.ringBuffer`, `authoritativeServer?.hostReplayRingBuffer`, `ballTrajectoryPredictor`, `renderer?.renderLists`, `clearSafeTensorsModelCache()`, unused `customVehicleAssets`, and `THREE.Cache.clear()` in `applyCleanFreeplaySubsystemsReset`.

### 4. Tauri Window Close Semantics
- **`src-tauri/src/lib.rs`**: Updated `WindowEvent::CloseRequested` so it no longer unconditionally calls `api.prevent_close(); window.hide();`; instead closes/destroys the window (cleaning up tunnels and hidden `configurator` when `main` closes) and updates macOS activation policy on `WindowEvent::Destroyed`.

---

## Verification Results

1. **`pnpm run check` (`radar` + `tsc --noEmit`)**: **PASS**
   - `1425/1425` TS files (`100.0% TS`), `tsc --noEmit` clean, 0 circular deps, all volume caps green.
2. **`cargo check` in `src-tauri/`**: **PASS** (0 errors; 2 pre-existing unused-variable warnings in `src/window_manager.rs`).
3. **`RC_REPLAY_FILE=/workspace/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-broadcast-v3`**:
   - **`bug-5-high-memory-and-unreleased-on-exit`**: **`PASS_FIXED` (`reproduced: false`)**
     - `bootMem`: `jsUsedMB: 161, wasmMB: 64, geometries: 40, textures: 13`
     - `loadedMem`: `jsUsedMB: 295, wasmMB: 64, rvFrames: 20107, geometries: 105, textures: 13`
     - `afterExitMem`: `jsUsedMB: 252, wasmMB: 64, rvFrames: 0, geometries: 35, textures: 13`
     - `retainedWasmMB: 0`
     - `retainedJsMB: 91`
     - `retainedGeometries: -5`
     - `tauriCloseHidesWindow: false`
   - *(Bugs 1–4 remain reproduced as expected, as they are owned by Wave 1 Track A / Track B and Wave 2/3.)*
