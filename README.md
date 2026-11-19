# RocketCompanion Multi-Issue Reproduction & Test Suite Update Report

- **Repository**: `Antigravi-ty/RocketCompanion`
- **Branch**: `origin/z/agy/rc/multiplayer-recheck` (`943b749`)
- **Replay Fixture**: `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` (6,811,237 bytes)
- **Test Suite Command**: `RC_VITE_ROLE=rc-http pnpm run repro:dev-broadcast-v3` (`scripts/runDevBroadcastReproV3.mjs`)

## Verification Summary
- `pnpm run check` (Radar + `tsc --noEmit`): **GREEN** (1425/1425 TS files)
- `pnpm run test:multiplayer-suite`: **24/24 PASS**
- `RC_VITE_ROLE=rc-http pnpm run repro:dev-broadcast-v3`: **5/5 REPRODUCED** (`reproduced=5/5` in `/tmp/dev-broadcast-repro-v3.json`)

## Reproduced Issues & Root Causes

### 1. Go Replay enters on Client with zero dynamics (static) until exit (`bug-1-goal-replay-client-static-no-dynamics`)
- **Observed**: Client enters `matchPhase === "replay"` and `hudPhase === "replay"`, but ball position remains static (`[0, 0, 93.2]`, `distinctBallPositions: 1`), `spectateInterpolationBuffer` stays at `bufferedFrames: 1`, and `camMode` remains `"pov"` with `override: true`.
- **Root Cause**:
  1. In `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`, `isGoalReplay` forces `shouldHoldAccumulator = true`.
  2. `src/game/pipeline/stages/SimulationStage.ts` passes `isPaused: flags.shouldHoldAccumulator || ...` (`true` during Goal Replay) to `HostBroadcastPublisher.publishIfReady`.
  3. `src/replay/ReplayFrameInterpolator.ts` (`applyReplayFrameToEngine`) restores historical frames with `{ preserveEngineTick: true }`, so `physics.getHeaderView().tickCount` never changes during Goal Replay.
  4. `HostBroadcastPublisher.publishIfReady` sees `isPaused && !tickChangedWhilePaused` and returns `false` every frame, sending zero snapshots until Goal Replay exits.
  5. On Client, `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` enters `if (isPureCsrSpectate && !reconciler.replayPlaybackMode && spectateInterpolationBuffer)` and skips the `else if` branch that calls `replayTimelineController.appendStreamedReplayFrame(...)`.

### 2. Replay Watch Together syncs UI roster/scoreboard but not internal 3D NameTags (`bug-2-replay-nametag-and-team-desync`)
- **Observed**: Host renders 3D NameTags (`Nexto #2`, `Nexto #5`, `Nexto #4`, `Nexto #6`), while Client has `tags: []` despite having 6 cars and synced DOM roster/scoreboard.
- **Root Cause**: `src/game/runtime/devSyncChannelHandlers.ts` and `src/game/runtime/onlineSessionEnvironmentPipeline.ts` update `replayViewerStore` and `hudStore` on the Client, but never call `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode, carEntries })`, leaving `VehicleNameTagSystem` (`getNameTagRuntimeState().carEntries`) unsynced.

### 3. Choppy/stuttery motion & state-only spectate without deterministic physics lookahead (`bug-3-replay-choppy-motion-no-physics-lookahead`)
- **Observed**: Client spectate/replay motion shows `distinctTicks: 1`, `frozenBallFrames: 20`, and `maxBallStepUu: 2973.1`.
- **Root Cause**: `SpectateInterpolationBuffer` (`src/network/sync/ReplicationStateInterpolator.ts`) only linearly/slerp-interpolates raw 60Hz snapshots, and `SimulationStage` calls `interpolator.sync(timestamp)` on `isPureCsrClient`. There is no buffer-ahead threshold (e.g. `>= 5` frames) driving deterministic WASM physics simulation across buffered future frames.

### 4. BoostPad Model state flickering violently every frame (`bug-4-boostpad-model-flicker`)
- **Observed**: Across consecutive frames during Replay Watch Together, Client recorded `totalPadFlips: 20` (`maxSinglePadFlips: 4`, e.g., pads 0..3 each flipping 4 times in a short window).

### 5. High memory footprint & unreleased resources on close/exit (`bug-5-high-memory-and-unreleased-on-exit`)
- **Observed**:
  - Post-boot: `jsUsedMB: 161`, `wasmMB: 259`, `geometries: 40`, `textures: 13`.
  - After loading 3v3 `.rcreplay` (`20107` frames): `jsUsedMB: 415`, `wasmMB: 310` (+51 MB), `geometries: 109` (+69).
  - After exiting replay: `jsUsedMB: 388` (`retainedJsMB: 227`), `wasmMB: 310` (`retainedWasmMB: 51`), `geometries: 105` (`retainedGeometries: 65`).
  - Tauri close behavior: `src-tauri/src/lib.rs` intercepts `WindowEvent::CloseRequested` with `api.prevent_close(); window.hide();`, keeping the WebView, Three.js, WASM, and Workers resident until Quit.
