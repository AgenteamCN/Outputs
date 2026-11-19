# RocketCompanion Multi-Issue Reproduction Clues (Preliminary)

Base branch: `origin/z/agy/rc/multiplayer-recheck` (`dad6dee`)

## Issue 1: Go Replay enters normally on Client, but has no dynamics (static) until exit
- **Reproduced live**: Scored goal in Freeplay Dev Sync Room; Client entered `matchPhase === "replay"` (`tlActive: true`), but across 8 samples (1.2s), Client `ballPos` stayed frozen at `[0, 0, 93]`, `spectateInterpolationBuffer` stayed at `bufferedFrames: 1, latestHostTick: 8907`, `snappedSamples` climbed 21->108, and `replayTimelineController` had `tlFrames: 0`.
- **Root Cause**:
  1. In `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`, `isGoalReplay` sets `shouldHoldAccumulator = true`.
  2. In `src/game/pipeline/stages/SimulationStage.ts`, `this.broadcastPublisher.publishIfReady` receives `isPaused: flags.shouldHoldAccumulator || flags.isReplayPaused || isSpectatePlaybackPaused()` (`true` during Goal Replay).
  3. In `src/replay/ReplayFrameInterpolator.ts`, `applyReplayFrameToEngine` calls `targetPort.restoreState(frameBuffer, { preserveEngineTick: true })`, so `physics.getHeaderView().tickCount` never advances during Goal Replay.
  4. In `src/game/pipeline/stages/HostBroadcastPublisher.ts`, `isPaused && !tickChangedWhilePaused` returns `false`, so Host sends **zero** state snapshots during Goal Replay.
  5. Additionally, on Client, `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` enters `if (isPureCsrSpectate && !reconciler.replayPlaybackMode && spectateInterpolationBuffer)` and skips the `else if` branch where `replayTimelineController.appendStreamedReplayFrame(...)` is called.

## Issue 2: Replay Watch Together syncs UI (scoreboard/roster) but not internal 3D NameTags
- **Root Cause**: `src/game/runtime/devSyncChannelHandlers.ts` and `src/game/runtime/onlineSessionEnvironmentPipeline.ts` update `replayViewerStore` and `hudStore`, but never call `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode, carEntries })` on the Client. Thus `VehicleNameTagSystem` (`shouldShowVehicleNameTags`) keeps `root.visible = false` or stale entries.

## Issue 3: Choppy/stuttery motion & state-only broadcast without deterministic buffer-ahead physics
- **Observation**: Client `SpectateInterpolationBuffer` only interpolates raw 60Hz snapshots (`snappedSamples` spikes whenever jitter exceeds `interpDelayTicks`, and `SimulationStage` calls `interpolator.sync(timestamp)` on `isPureCsrClient`), with no deterministic physics simulation over a lookahead buffer (e.g. >=5 frames).

## Issue 4: BoostPad Model state flickering/jittering every frame
- **Reproduced live**: Over 20 `requestAnimationFrame` frames during Replay Watch Together, `Host` recorded 17 `pad.full.visible` flips and `Client` recorded 22 `pad.full.visible` flips (e.g. `pad1` flipped `full1: false -> true` while `act1` remained `0`).

## Issue 5: High memory usage (~1.6GB immediate allocation, ~4.38GB Web Content, not released on close)
- **Root Cause (Tauri close-not-releasing)**: `src-tauri/src/lib.rs` intercepts `WindowEvent::CloseRequested` with `api.prevent_close(); window.hide();` (close-to-tray), keeping the entire WebView + Three.js + WASM + Workers resident until Quit.
- **Boot memory breakdown**: Measured immediately after boot in headless Chrome: JS heap `161 MB / 175 MB`, WASM `HEAPU8` `259 MB` (plus `AuthoritativeServerWorker` / `BotInferenceWorker` / WebGL render targets & textures).
