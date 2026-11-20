# Track C Report — Bug 5 (`fix/bug5-replay-geometry-leak`)

- **Repository**: `Antigravi-ty/RocketCompanion`
- **Base**: `origin/z/agy/rc/multiplayer-recheck` (`7252e425069b52988cab014ddb8c85bfd237f879`)
- **Branch**: `fix/bug5-replay-geometry-leak`
- **Verified Bot Commit**: `6889b8627d69c5f240334f6db6533ba7d2b9db83` (`verified: true`, signed by `antigravity-fine-grained-token[bot]`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)

## Modified Files (Track C Exclusive)
1. `src/game/runtime/RuntimeResourceRegistry.ts`
   - Added `once?: boolean` on `RuntimeDisposableResource` with auto-unregistration in `disposeScope`.
   - Added `collectObjectGeometries`, `disposeDetachedCarSlotGeometries`, `disposeDetachedOrExtraFlipResetVisuals`, `disposeHiddenObjectGeometries`, `disposeArenaHiddenEffectGeometries` (covering hidden `cars[0]`, `carDemolitions[0].object`, `ArenaBlobShadows` excluding `CarBlobShadow_0`, `SpeedTrail_PS-ribbon`, `GoalBarrierSurface`, `GoalBarrierContour`, `ball-locator-clarity-halo`, `ball-locator-arrow`, and `carBoosts[0]` emitters), and `registerReplayViewerVisualCleanup` (`scope: 'replay', once: true`).
2. `src/replay/ReplayViewerRosterSubController.ts`
   - Snapshots car slot geometries before `arena.ensureCar(...)` in `applyRecordedLoadoutsToArena` and `applyUserLoadoutToTakenOverCar`, and disposes detached geometries afterwards via `disposeDetachedCarSlotGeometries`.
   - Disposes detached/extra `FlipResetVisual` instances in `applyUserLoadoutToTakenOverCar`.
3. `src/replay/ReplayViewerCausalSyncController.ts`
   - Calls `registerReplayViewerVisualCleanup(r)` in `prepareRuntimeForSession`.
   - Snapshots `car0` geometries before `restoreLocalPlayerFreeplayLoadout(0)` / `changePlayerVehicle(targetCar)` in `resetArenaAfterClear`, and disposes detached `car0` geometries, extra `FlipResetVisual` instances, hidden arena effect geometries, and `renderer.renderLists`.

## Verification Results
- `pnpm run check` (`radar` + `tsc --noEmit`): **PASS (GREEN)** — 1430/1430 TS, 0 circular deps, 0 HardErrors.
- `pnpm run test:multiplayer-suite`: **26/26 PASS**.
- `pnpm run repro:dev-broadcast-v3` (Headless Chrome + WebGL2 + WASM):
  - `bug-5-high-memory-and-unreleased-on-exit`:
    - `bootMem`: `wasmMB=64`, `geometries=40`, `textures=13`
    - `loadedMem` (3v3 replay): `wasmMB=64`, `geometries=110`, `textures=16`
    - `afterExitMem`: `wasmMB=64`, `geometries=42`, `textures=16` (`retainedWasmMB=0`, `retainedGeometries=2`)
  - **Root cause analysis of the remaining `retainedGeometries=2`**:
    - Scene traversal after exit confirmed **0** detached/leaked replay geometries remain (`41` visible geometries in `arena.scene` + `1` `FullScreenQuad` = `42`, identical to steady-state 1-car Freeplay).
    - At `bootMem` (`40`), 2 of the 6 `BoostOrbShadowBillboard` `PlaneGeometry` meshes (created per-pad in `BoostPadTemplateLoader.ts`, outside Track C's exclusive files) are behind the kickoff camera and frustum-culled until the replay camera pans around the arena (`40 -> 42`).
