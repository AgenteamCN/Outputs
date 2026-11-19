# Interim Reproduction Clues (RocketCompanion `z/agy/rc/multiplayer-recheck` @ `dad6dee`)

1. **Issue 1 — Goal Replay enters on Client but has no dynamics (static), then syncs on exit**:
   - In `src/game/pipeline/stages/SimulationStage.ts`, `stepNetworkReconciliationAndReplayStream` is gated by `!flags.isLocalDirectorPreview` (`activeMode?.directorPreview?.isReplayActive`). During Freeplay Goal Replay on Host, `directorPreview.isReplayActive` becomes `true`, so `server.update()` is skipped completely.
   - Additionally, `HostBroadcastPublisher.publishIfReady` receives `isPaused: flags.shouldHoldAccumulator` (`true` during `isGoalReplay`), and `tickCount` on Host stays frozen.
   - Observed in Headless Chrome: Client enters `phase: "replay"`, `camMode: "director"`, `override: true`, while Client ball position stays frozen at `(0, 0, 93)` with `bufFrames: 1`.

2. **Issue 2 — Replay Watch Together: UI roster syncs, but 3D NameTags & `arena.carTeams` do not**:
   - `src/game/runtime/devSyncChannelHandlers.ts` updates `useReplayViewerStore.resolvedRoster` (UI), but never calls `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode: true, carEntries })` and never updates `arena.carTeams` on Client.
   - Observed in Headless Chrome with `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`: Host has `arenaCarTeams: [0,0,0,1,1,1]`, whereas Client has `arenaCarTeams: [0,0,0,1,0,1,0,1]` and missing/unsynced `nameTagSystem` entries.

3. **Issue 3 — Choppy/stuttery motion (state-only updates, no deterministic physics lookahead)**:
   - Observed in Headless Chrome: Client `physics.getHeaderView().tickCount` stays frozen (`528`), ball position repeats identically for 3 consecutive `requestAnimationFrame` ticks (`1095.6, -2944.5`) and then jumps ~1400 uu in a single frame (`189.8, -4015.6` -> `-714.2, -5000.8`).

4. **Issue 4 — BoostPad Model flickering/jittering violently**:
   - Observed in Headless Chrome: `BoostPad` `pad.full.visible` flipped 27 times across 30 frames (e.g., pads 11 and 28 flipped 4 times each).
   - Root cause observed on pad 11: `bufferCooldown` jumped `4 -> 1.91 -> 4 -> 3.12`, triggering `respawnedAndCollected` (`bufferCooldown > prevBufferCd + 0.5`) in `BoostPadSystem.update()`, which repeatedly re-triggered `startCollectAnimation(11)` and toggled `pad.full.visible`.

5. **Issue 5 — High Memory Usage & Memory Not Released on Close**:
   - `RocketSimArenaBarrierController.ts` / `onlineSessionEnvironmentPipeline.ts` allocate the 256MB C++ WASM Replay Buffer at startup (`1_BOOT`: `jsUsedMB: 161`, `wasmMB: 259`).
   - Loading `.rcreplay` increases `wasmMB` to `310` (+51 MB), `jsUsedMB` to `248` (+87 MB), `geometries` to `57` (+17). Exiting back to Freeplay leaves `wasmMB: 310`, `jsUsedMB: 212`, `geometries: 61` (not freed).
   - In `src-tauri/src/lib.rs`, `WindowEvent::CloseRequested` calls `api.prevent_close(); window.hide();`, so closing the Tauri window never destroys the WebView or releases memory until Quit.
