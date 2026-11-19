# RocketCompanion Multiplayer & Replay Reproduction Report (`z/agy/rc/multiplayer-recheck`)

- **Base Commit**: `971ef3d` (`fix(multiplayer): preserve spectate camera focus across kickoff and mode leave in Pure CSR`)
- **Test Suite Commit**: `943b749cf1c1abbd651c5862d91edecceac360bd` (`verified: true`)
- **Replay Asset**: `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` (downloaded from wenshushu)
- **New E2E Suite**: `scripts/runDevBroadcastReproV3.mjs` (`pnpm run repro:dev-broadcast-v3`)
- **Result**: **5 / 5 issues reproduced** in Headless Chrome + WebGL + WASM (`reproduced=5/5`).

---

## 1. Goal Replay Enters on Client Without Dynamics (`bug-1-goal-replay-client-static-no-dynamics`)
- **Symptom**: Client enters Goal Replay (`phase: "replay"`, `hudPhase: "replay"`, `camMode: "director"`, `override: true`), but the ball and cars are completely static (`bx: 0, by: 5884.3, bz: 171.6` -> `bx: 0, by: 0, bz: 93.2`, `bufFrames: 1`), and only resync after Goal Replay exits.
- **Root Cause**:
  1. In `src/game/pipeline/stages/SimulationStage.ts`, `stepNetworkReconciliationAndReplayStream` is gated by `!flags.isLocalDirectorPreview`. On the Host during Freeplay Goal Replay, `activeMode.directorPreview.isReplayActive` is `true`, so `server.update()` is skipped completely.
  2. `HostBroadcastPublisher.publishIfReady` receives `isPaused: flags.shouldHoldAccumulator` (`true` during `isGoalReplay`), while Host `tickCount` remains frozen.

## 2. Replay Watch Together: UI Roster Syncs, 3D NameTags Desynced (`bug-2-replay-nametag-and-team-desync`)
- **Symptom**: Client syncs HUD scoreboard and 1-6 roster strips, but 3D in-world NameTags do not match Host:
  - Host tags: `[{0: "Nexto #4"}, {1: "Nexto #5"}, {2: "Nexto #6"}, {4: "Nexto #2"}]`
  - Client tags: `[{0: "Player2511"}, {1: "Captain Ping"}, {2: "Necto"}]` (fallback preview names).
- **Root Cause**:
  - `src/game/runtime/devSyncChannelHandlers.ts` updates `useReplayViewerStore.resolvedRoster` (UI), but never calls `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode: true, carEntries })` on the Client, so `VehicleNameTagSystem` (`vehicleNameTagFramePipeline.ts`) falls back to `config.previewNames`.

## 3. Choppy/Stuttery Motion Without Deterministic Physics Lookahead (`bug-3-replay-choppy-motion-no-physics-lookahead`)
- **Symptom**: Client `physics.getHeaderView().tickCount` stays frozen (`distinctTicks: 1`, `tick: 1102`), ball position repeats identically across consecutive `requestAnimationFrame` callbacks (`frozenBallFrames: 16` out of 30), and then teleports (`maxBallStepUu: 2959.6`).
- **Root Cause**:
  - `applyReplayFrameToEngine` restores raw snapshots with `{ preserveEngineTick: true }` without stepping deterministic 120Hz physics over a buffered lookahead window (e.g., buffering `>= 5` frames and simulating continuous 120Hz physics between broadcast states).

## 4. BoostPad Models Flicker Violently Every Frame (`bug-4-boostpad-model-flicker`)
- **Symptom**: Client `pad.full.visible` toggled `18` times across 30 frames (`maxSinglePadFlips: 2` across 9 pads).
- **Root Cause**:
  - Observed on pad 11 during replay: `bufferCooldown` jumped `4 -> 1.91 -> 4 -> 3.12`. In `src/entities/BoostPadSystem.ts`, `respawnedAndCollected` (`!wasActive && !isActive && prevBufferCd > 0 && bufferCooldown > prevBufferCd + 0.5`) repeatedly re-triggers `startCollectAnimation(i)` and toggles `pad.full.visible`.

## 5. High Baseline Memory & Unreleased Resources on Exit/Close (`bug-5-high-memory-and-unreleased-on-exit`)
- **Measurements** (Headless Chrome with GC forced via `HeapProfiler.collectGarbage`):
  - **Boot**: `jsUsedMB: 161`, `wasmMB: 259` (256 MB C++ WASM Replay Buffer allocated at startup in `RocketSimArenaBarrierController.ts` / `onlineSessionEnvironmentPipeline.ts`), `geometries: 39`, `textures: 13`.
  - **After loading `.rcreplay`**: `jsUsedMB: 414` (+253 MB), `wasmMB: 310` (+51 MB), `geometries: 108` (+69).
  - **After `onReturnToFreeplay()`**: `jsUsedMB: 389` (**+228 MB retained**), `wasmMB: 310` (**+51 MB WASM retained**), `geometries: 104` (**+65 geometries not disposed**).
- **Tauri Close Behavior**:
  - In `src-tauri/src/lib.rs`, `WindowEvent::CloseRequested` calls `api.prevent_close(); let _ = window.hide();`, so closing the window hides it without destroying the WebView or releasing memory until Quit.

---

## Verification
- `pnpm run typecheck`: PASS
- `pnpm run repro:dev-broadcast-v3`: PASS (`reproduced=5/5`, report written to `/tmp/dev-broadcast-repro-v3.json`)
