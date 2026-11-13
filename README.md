# Telestrator CMF v4 Upgrade Report

- **Branch**: `z/agy/rc/telestrator-cmf-v4` (based on `origin/alpha` `ae6c593`, absorbing `z/agy/rc/track-a-telestrator-v3` `541087f` and `z/agy/rc/telestrator-overlay-v3` `3d28980` improvements)
- **Verified Commit**: `5b1ee5c3b78bb16bcfc35eb1688b6d930904b97a` (`signature.isValid: true`, `state: VALID`)
- **Verification**:
  - `pnpm run check` (`radar.ts` + `tsc --noEmit`): **PASS** (1367/1367 TS files, 0 errors, 0 circular deps, 0 upward imports, 0 value re-exports)
  - `pnpm run build`: **PASS**

## Summary of Implemented Improvements

1. **CMF Collision Mesh Raycasting**
   - Added [`cmfCollisionMeshBuilder.ts`](file:///work/RocketCompanion/src/entities/telestrator/cmfCollisionMeshBuilder.ts) to parse `/arenamesh/*.cmf` and `/arenamesh/throwback_stadium/*.cmf` chunks into pure-Three raycast meshes with `(i0, i2, i1)` index winding so normals face inward (`THREE.FrontSide` for standard stadium, `THREE.DoubleSide` for Throwback).
   - Updated [`arenaSurfaceRaycaster.ts`](file:///work/RocketCompanion/src/entities/telestrator/arenaSurfaceRaycaster.ts) to prioritize `cmfTargets` over visual GLB meshes and 12 analytical planes.
   - Removed `ball` and `cars` from Telestrator raycast targets in [`TelestratorOverlaySubsystem.ts`](file:///work/RocketCompanion/src/game/subsystems/TelestratorOverlaySubsystem.ts), eliminating accidental ball/car surface jumps.

2. **Round Caps / Joins, View-Ray Arrowhead, & Live Preset Rebuild**
   - Split math/simplification/hash into [`telestratorStrokeMath.ts`](file:///work/RocketCompanion/src/entities/telestrator/telestratorStrokeMath.ts) and updated [`telestratorStrokeGeometry.ts`](file:///work/RocketCompanion/src/entities/telestrator/telestratorStrokeGeometry.ts):
     - Added semicircular **Round Caps** on continuous & dashed strokes and **Round Joins** at corners (> 15°).
     - Adopted `3d28980`'s `(accumPhase + localOffset) % dashPeriod` dashed phase and `1050 / 1055` `renderOrder`.
     - Switched arrowhead billboard normal to `normalize(camPos - tipPt)` and accumulated view-plane projected arc length (~28px equivalent at `tipDist`) to eliminate wide-FOV edge perspective skew and corner jitter.
   - Added `presetIndex` metadata to strokes and `rebuildStrokesForPresets(...)` in [`TelestratorManager.ts`](file:///work/RocketCompanion/src/entities/telestrator/TelestratorManager.ts), subscribed from [`TelestratorOverlaySubsystem.ts`](file:///work/RocketCompanion/src/game/subsystems/TelestratorOverlaySubsystem.ts) so preset width/opacity and arrowhead sliders update all matching historical strokes in real time.

3. **UI, Eye Visibility, Initial State, & Dump Last Stroke**
   - Updated [`telestratorStore.ts`](file:///work/RocketCompanion/src/ui-core/stores/telestratorStore.ts):
     - `drawingEnabled` initializes as `false` (remains off when opening the overlay until explicitly activated).
     - Added `strokesVisible` (Eye toggle): hiding strokes automatically sets `drawingEnabled = false`.
     - Added Host/Client permission state (`isHost`, `localAuthorId`, `isCameraSynced`, `allowClientDrawing`, `allowedClientIds`) and `canLocalDrawInTelestrator` guard.
   - Updated [`TelestratorToolbarFloatingWindow.tsx`](file:///work/RocketCompanion/src/ui-core/panels/dev/TelestratorToolbarFloatingWindow.tsx) and extracted [`TelestratorSettingsPanel.tsx`](file:///work/RocketCompanion/src/ui-core/panels/dev/TelestratorSettingsPanel.tsx):
     - Absorbed `3d28980`'s `createPortal` adaptive left/right flipping bubble tooltips and `useEffect` window size sync.
     - Added Host-only Eye/EyeOff visibility toggle, scoped Undo/Clear labels (1st click clears own strokes; 2nd click clears all strokes for Host only), and **Dump Last Stroke** button (copies `{ rawPoints, simplifiedPoints, renderedPoints, wireBytes, hash, ... }` JSON to clipboard and displays point counts).
   - Cleaned up [`TelestratorDeveloperSection.tsx`](file:///work/RocketCompanion/src/ui-core/recipes/settings/components/TelestratorDeveloperSection.tsx) by removing duplicate brush sliders and adding Host global/per-Player-ID client drawing permission controls.

4. **Network Wire Codec (`TelestratorWireCodec.ts`)**
   - Upgraded [`TelestratorWireCodec.ts`](file:///work/RocketCompanion/src/network/protocol/TelestratorWireCodec.ts) to 8B/point (`Int16` quantized positions + 2B Octahedral normals), added `STROKE_DRAFT` (for 10Hz piggyback), `HOST_PERMISSION_SYNC`, `presetIndex`, `strokeSeq`, and fixed `computeStrokeHash` to include `presetIndex`, `arrowLengthRefPx`, `arrowWidthRefPx`, and `Math.round(opacity * 255)`.

## Remaining Integration Notes (For Subsequent Network / Track D-F Merge)
- `TelestratorOverlaySubsystem.handlePointerUp()` still performs local encode/decode/ack round-trip verification; actual DataChannel dispatch (`STROKE_DRAFT` at 10Hz, `STROKE_COMMIT` / `HOST_PERMISSION_SYNC` on Reliable channel) will plug directly into `TelestratorWireCodec` once Track D's `RoomReplicationBus` dual-channel mux lands.
- `useTelestratorStore.getState().setHostRole(...)` and `setIsCameraSynced(...)` should be driven by the multiplayer room / camera sync state when online rooms are active.
