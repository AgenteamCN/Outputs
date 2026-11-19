# A-Track (W2-A) Completion Report: `SpectateInterpolationBuffer` >= 5-Frame Lookahead Window & Cursor State Machine

- **Repository**: `https://github.com/Antigravi-ty/RocketCompanion`
- **Base Commit**: `2565774` (`origin/z/agy/rc/multiplayer-recheck`)
- **Working Branch**: `wave2-lookahead-buffer` (`2ccf6a2`)
- **Track**: `A-Track W2-A` (Bug 3 Phase 2-A)

## 1. Environment & Toolchain Setup

- Installed `pnpm@12.21.0` globally (`npm install -g pnpm@12.21.0`) and installed workspace dependencies via `pnpm install --no-frozen-lockfile --ignore-scripts`.
- Verified `node v22.23.5`, `tsc --noEmit`, `radar`, and `runMultiplayerSuiteTests.ts`.

## 2. Architecture & Implementation Summary

Strictly scoped to `src/network/sync/` (zero overlap with B-Track `SimulationStage.ts` / `simulationNetworkReconcilePipeline.ts` / `simulationFrameClassifierPipeline.ts` or C-Track `BoostPadSystem.ts`):

1. **Pure Lookahead Pipeline ([spectateLookaheadPipeline.ts](file:///workspace/RocketCompanion/src/network/sync/spectateLookaheadPipeline.ts))**:
   - Defined `SpectateCursorState`: `'empty' | 'buffering' | 'tracking' | 'lookahead' | 'stalled'`.
   - Enforced `SPECTATE_MIN_LOOKAHEAD_FRAMES = 5` (configurable up to `SPECTATE_MAX_LOOKAHEAD_FRAMES = 16`) and `SPECTATE_STALL_TIMEOUT_MS = 250`.
   - Implemented `extrapolateSpectateSnapshot()` for velocity/gravity-aware kinematic lookahead of ball and cars while preserving discrete BoostPad `isActive` states and monotonically decrementing `cooldown`.
   - Implemented `fillSpectateLookaheadWindow()`, `findSpectateBracketFrames()`, and `upsertSpectateLookaheadFrame()`.

2. **Buffer & Cursor State Machine ([ReplicationStateInterpolator.ts](file:///workspace/RocketCompanion/src/network/sync/ReplicationStateInterpolator.ts))**:
   - Upgraded `SpectateInterpolationBuffer` with a `>= 5`-frame lookahead window (`lookaheadFrames`), automatic kinematic lookahead fill when `frames.length >= 2`, and `pushLookaheadFrame()` for external WASM physics lookahead injection (automatically pruned on authoritative `pushFrame()`).
   - Added non-breaking state machine transitions (`empty` -> `buffering` -> `tracking` <-> `lookahead` -> `stalled`), avoiding backward rubberbanding when `renderTick` enters the lookahead horizon.
   - Extended `SpectateInterpolationSample` (`cursorState`, `isLookahead`, `lookaheadFramesAvailable`) and `SpectateInterpolationStats` (`cursorState`, `lookaheadWindowFrames`, `lookaheadFramesAvailable`, `lookaheadSamples`, `stallCount`) while keeping all existing methods/fields 100% backward-compatible.
   - Kept `ReplicationStateInterpolator.ts` at `17.6 KB / 554 lines` (under the 18.0 KB / 800-line hard limit).

3. **Test Coverage ([runMultiplayerSuiteTests.ts](file:///workspace/RocketCompanion/scripts/runMultiplayerSuiteTests.ts))**:
   - Extended `testSpectateInterpolationBufferReverseDelay` (Section 7) to verify `>= 5`-frame lookahead window enforcement, `empty` -> `buffering` -> `tracking` -> `lookahead` -> `stalled` transitions, non-duplicate smooth lookahead sampling, and `pushLookaheadFrame()` + authoritative overwrite behavior.

## 3. Gate Verification Results

- `pnpm run check` (`radar` + `tsc --noEmit`): **PASS (GREEN)** — 1429/1429 TS files, 0 Circular Deps, 0 Value Re-Exports, 0 Upward Imports, `NameTagScatter=1/1`, `BoolHotspots=7`, `UnscopedCaches=0`.
- `pnpm run radar:audit`: **PASS (GREEN)**.
- `pnpm run test:multiplayer-suite`: **PASS (24/24)** — all suite tests including updated Test 16 (`SpectateInterpolationBuffer`) passed.
