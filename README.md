# RocketCompanion vNext Deep Scan, Reproduction Suite & Parallel Fix Plan

- **Baseline**: `integration/vnext-core` (`af6c9f9`) -> Repro Suite pushed at [`f945884`](https://github.com/Antigravi-ty/RocketCompanion/commit/f945884dca3c7681ef09e49574585f8679dd09bd) (Verified Bot Commit)
- **Reproduction Command**: `pnpm run repro:vnext-issues` (`scripts/runVNextIssuesReproSuite.mjs`)
- **Result**: **5/6 issues deterministically reproduced** in Headless Chrome + WebGL2 + WASM (plus background/foreground WebAudio lifecycle probe)

---

## 1. Deep Scan Root-Cause Findings

### Issue 1: Free Play Developer Broadcasting — Boosting / Car Jitter vs. Smooth Wheel Supersonic Trail
- **Repro ID**: `repro-1-broadcast-boost-jitter-vs-wheel-trail` -> **REPRODUCED**
  - Observed on Client: `bufferMode: "passthrough"`, `avgIntervalMs: 17.92`, `zeroDeltaFrames: 43`, `jumpFrames: 1`, `sampleDeltas: [0, 255.93, 0, ...]`.
- **Root Cause**:
  1. `HostBroadcastPublisher` (`src/game/pipeline/stages/HostBroadcastPublisher.ts`) publishes snapshots at ~60Hz (`publishIntervalMs = 16.67ms`).
  2. `TimelineFrameBuffer` (`src/network/sync/TimelineFrameBuffer.ts`) has `DEFAULT_PASS_THROUGH_THRESHOLD_MS = 22`. Because `avgIntervalMs <= 22ms`, `isPassThroughActive()` returns `true` and **disables all frame interpolation**, returning `latest.snapshot` verbatim (`alpha = 1`).
  3. On `simulationNetworkReconcilePipeline.ts` (`stepNetworkReconciliationAndReplayStream`), `applyReplayFrameToEngine(physics, sampled.snapshot, interpolator, null, 1.0)` overwrites both `interpolator.prevState` and `interpolator.currState` with the same snapshot, and `SimulationStage` calls `interpolator.sync(timestamp)`.
  4. Any rAF phase mismatch or 1–2ms transport jitter produces `0 packets` on frame N (car/ball/boost freeze) followed by `2 packets` on frame N+1 (car/ball jump 2–4 physics ticks), causing violent jitter on the car mesh, camera, and boost emitter (`vehicleBoostEmitter.ts` emits every rAF regardless of whether `carRoot.position` moved).
  5. **Why `SupersonicWheelTrailSystem` looks normal**: `resolveCarStepTicks` (`src/entities/vehicle/supersonic-trail/SupersonicWheelTrailReplaySync.ts`) checks `posDeltaSq < 1e-4 && quatDot > 0.999999` (`isPoseStationary`). On held zero-delta frames it returns `dtTicks: 0, isPausedFreeze: true`, freezing ribbon emission until `carRoot.position` actually advances, then bridging `lastPos -> currPos` smoothly.

---

### Issue 2: Free Play Goal Replay — Team Assignment, Goal Anthem Broadcast & Unified Cinematic Camera
- **Repro ID**: `repro-2-freeplay-goal-replay-team-anthem-camera` -> **REPRODUCED**
  - Observed on Client: `matchPhase: "replay"`, `hudScorerTeam: 0`, `hudAnthemId: null`, `hudLocalTeam: 1`, `hostTeam: 0`.
- **Root Cause**:
  1. **Opposing Team Assignment**: In `applyReplayFrameToEngine` (`src/replay/ReplayFrameInterpolator.ts`) and `freeplayGoalReplaySessionPipeline.ts` (`resolveFreeplayGoalScoringTeam`), cars and goals fall back to `carIndex % 2` or `ballPosY >= 0 ? 0 : 1`. In Dev Broadcasting spectate, Host is `team 0` while Client resolves `hudLocalTeam: 1` (or spectate lacks a unified neutral/shared-perspective team policy).
  2. **Missing Goal Anthem in `GOAL_REPLAY_WILL_START`**: `GoalReplaySessionCoordinator.beginGoalReplaySession` (`src/game/modes/shared/GoalReplaySessionCoordinator.ts`) only broadcasts `MATCH_SIGNALS.GOAL_REPLAY_START` (with `scoringTeam | (scorerCarIndex << 1)`), never `GOAL_REPLAY_WILL_START` with an `anthemId` (and `RuntimeGoalReplaySignalSubController.handleGoalReplayWillStart` only reads `anthemId` from `lastRosterSnapshot`, which isn't synced for Freeplay random anthems).
  3. **Divergent Replay Pipeline**: Freeplay Goal Replay does not share `UnifiedGoalReplayStateMachine` + `vnextGoalReplayCameraConsumer` cinematic camera pose broadcast from Match/Exhibition, so Host and Client can diverge on camera framing and HUD state.

---

### Issue 3: Exhibition (`exp-multi-bots`) Broadcasting Mode — No Pause & Goal Replay Freeze/Stall
- **Repro ID**: `repro-3-exhibition-broadcast-no-pause-and-goal-replay-stuck` -> **REPRODUCED**
  - Observed: `hasPauseMethod: false`, `matchPaused: false`; Client Goal Replay sampled `17` unique ball positions across `30` frames (stalling/dropping frames).
- **Root Cause**:
  1. **No Pause in Exhibition Broadcasting**: `ExperimentalMultiBotsMode` (`src/game/modes/ExperimentalMultiBotsMode.ts`) and `experimental-multi-bots/` have zero `paused` / `togglePause` handling, and `HostPauseAckController` is not wired into Exhibition's simulation/lifecycle loop.
  2. **Goal Replay Stall / Desync**: On the Host, `SimulationStage.execute` runs *before* `GameplayModeStage.execute` (`ExperimentalMultiBotsMode.update`), so `HostBroadcastPublisher` snapshots `physics` before `replaySession.stepUpdate` advances it; meanwhile on the Client, `RuntimeGoalReplaySignalSubController.handleGoalReplayStart` activates `ctx.replayTimelineController` while `ExperimentalMultiBotsMode` is unaware of the streamed replay state, causing dual-controller contention and stalled frames.
  3. **Unified State Broadcast Principle (Exhibition & Training Packs)**: Instead of running independent replay/training state machines on clients, the Host should broadcast the authoritative `VNextWorldFrame` (physics snapshot + `playbackState` + `cameraPose` + `hudState` + `anthemId`), and clients in `client-csr-spectate` / `client-goal-replay` should purely render the interpolated state stream.

---

### Issue 4: Replay `4/5/6` Switching — HUD Left/Right Swap Glitch & `1..6` Car Color / Model / Hitbox Mismatch
- **Repro ID**: `repro-4-replay-456-hud-swap-and-loadout-mismatch` -> **REPRODUCED**
  - Observed: Pressing `Digit1..Digit3` sets `hudLocalTeam: 0`; pressing `Digit4..Digit6` flips `hudLocalTeam: 1` (`hudSwappedOn456: true`), swapping the scoreboard left/right teams.
- **Root Cause**:
  1. **HUD Left/Right Swap**: `ReplayViewerRosterSubController.selectPovSlot` (`src/replay/ReplayViewerRosterSubController.ts`) calls `this._syncNameTagsAndHudTeam(null, this.activePovTeam)`, which mutates `hudStore.localTeam` to the focused car's team (`0` for slots 1–3, `1` for slots 4–6). `useInGameHudChoreography` (`src/ui-core/hud/useInGameHudChoreography.ts`) derives `isOrangeLeft = localTeam === 1`, flipping the entire scoreboard left/right whenever switching between Blue (`1..3`) and Orange (`4..6`).
  2. **1..6 Car Color / Model / Hitbox Mismatch**:
     - `ReplayViewerRosterResolver.ts` maps `Octane` (`bodyProductId: 23`) in `RLRCVehicleProductCatalogTable.ts` to `visualId: 'fennec'`, and falls back to `defaultColorSlot = team === 0 ? 3 : 0` when `metadata.roster` or `PLAYER_LOADOUT_UPDATED` signals are absent.
     - `slotKey` (`1..6`, sorted by team left/right) can differ from `carIndex` (`0..5`), and `ReplayFrameEventDispatcher` / `applyReplayFrameToEngine` / `RLRCPlaybackEntityLifecyclePipeline` can overwrite `carIndex` team/hitbox/color after `ReplayViewerRosterSubController.applyRecordedLoadoutsToArena` runs.

---

### Issue 5: Camera Frame-Rate Dependence (Should Run at Fixed 120Hz Step / Worker-Aligned)
- **Repro ID**: `repro-5-camera-fps-dependence` -> **REPRODUCED**
  - Observed over 1.0s identical trajectory:
    - `30fps vs 120fps`: **`6.831 uu`** position divergence (`deltaFov: 0.01`)
    - `240fps vs 120fps`: **`0.311 uu`** position divergence
- **Root Cause**:
  - `GameCameraSubsystem` (`src/game/subsystems/GameCameraSubsystem.ts`) calls `CleanPovCameraKernel.stepView` once per `requestAnimationFrame` on the main thread with variable `dt`.
  - `CleanPovCameraKernel` (`src/camera/CleanPovCameraKernel.ts`) integrates springs (`stepSpringFilter3D`), ground blend (`groundRate * dt`), and FOV (`fovRate * dt`) with variable `dt` Euler steps, so higher FPS = faster/tighter response and lower FPS = slower/laggy response.
  - **Fix Direction**: Step `CleanPovCameraKernel` inside a fixed `1/120s` accumulator (or inside the physics worker alongside 120Hz physics ticks) and interpolate `prevCameraPose -> currCameraPose` by `interpolator.alpha` at render time.

---

### Issue 6: Tauri Background -> Foreground Audio Delay
- **Repro ID**: `repro-6-audio-background-foreground-latency` -> **PROBE**
  - Note: In headless Linux Chrome (without `--enable-features` WebView2/WKWebView OS suspension), `ctx.currentTime` stayed synced (`driftMs: 6`), whereas earlier under CPU load/throttling `driftMs` spiked to **`188–266ms`**.
- **Root Cause in `src/audio/audioContext.ts` & `src/audio/AudioMixer.ts`**:
  1. On `blur` / `hidden`, `audioContext.ts` calls `master.setActive(false)` (disconnecting `master.output` from `ctx.destination`) and `slotPool.silenceAll()`, **without calling `ctx.suspend()`**. In Tauri (WebView2 / WKWebView), leaving `AudioContext` running while backgrounded/throttled causes OS audio buffer/clock drift or OS-level suspension.
  2. On foreground (`focus` / `rc-tauri-window-shown`), `audioContext.ts` triggers an `80ms` `focusRecheckTimer`, then `master.flush()`, then `ctx.resume()` (async promise), and `AudioMixer.apply()` applies a `50ms` `linearRampToValueAtTime`, while `VehicleAudioEmitter` / `AudioSlotPool` has to re-acquire and ramp up from silence.

---

## 2. Parallel Branch Execution Plan (Max Velocity, Zero Merge Conflicts)

Split into **5 independent parallel branches** off `integration/vnext-core` (each owning disjoint subsystems), plus a final integration pass:

| Branch | Owner Scope (Disjoint Files) | Issues Covered | Verification Gate |
| :--- | :--- | :--- | :--- |
| `fix/vnext-broadcast-interp-and-exhibition` | `src/network/sync/TimelineFrameBuffer.ts`, `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`, `src/game/pipeline/stages/SimulationStage.ts`, `src/game/modes/ExperimentalMultiBotsMode.ts`, `src/game/modes/experimental-multi-bots/*` | **Issue 1** (disable passthrough / enable dual-frame Hermite/linear interp at 60Hz; pause boost emitter on zero-delta held frames) + **Issue 3** (Exhibition pause + unified state broadcast during Goal Replay & Training Pack) | `repro-1` & `repro-3` pass (`zeroDeltaFrames == 0` or smooth interp, `hasPauseMethod == true`, `uniqueBallPositions >= 25`) |
| `fix/vnext-goal-replay-team-anthem-cam` | `src/game/runtime/RuntimeGoalReplaySignalSubController.ts`, `src/game/modes/shared/GoalReplaySessionCoordinator.ts`, `src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts`, `src/replay/UnifiedGoalReplayStateMachine.ts`, `src/game/subsystems/vnextGoalReplayCameraConsumer.ts` | **Issue 2** (neutral/shared spectate team in Freeplay; broadcast `anthemId` in `GOAL_REPLAY_WILL_START`; reuse Match `UnifiedGoalReplayStateMachine` + Cinematic Camera) | `repro-2` passes (`hudAnthemId` synced, unified camera override, consistent team view) |
| `fix/vnext-replay-456-hud-and-loadouts` | `src/replay/ReplayViewerRosterSubController.ts`, `src/replay/ReplayViewerRosterResolver.ts`, `src/replay/rlrc/catalog/RLRCVehicleProductCatalogTable.ts`, `src/ui-core/hud/useInGameHudChoreography.ts` | **Issue 4** (decouple `hudStore.localTeam` from `activePovTeam` in Replay Viewer so 4/5/6 never flips scoreboard; fix `Octane` visual/hitbox mapping and enforce per-car color/model/hitbox on `slotKey` -> `carIndex`) | `repro-4` passes (`hudSwappedOn456 == false`, `visualMismatches == 0`, colors/hitboxes match 1..6) |
| `fix/vnext-camera-fixed-120hz` | `src/game/subsystems/GameCameraSubsystem.ts`, `src/camera/CameraManager.ts`, `src/camera/CleanPovCameraKernel.ts` | **Issue 5** (fixed `1/120s` camera accumulator + `prevPose/currPose` render interpolation, optional move into physics worker) | `repro-5` passes (`deltaPos30vs120 < 0.05`, `deltaPos240vs120 < 0.05`) |
| `fix/tauri-audio-focus-resume` | `src/audio/audioContext.ts`, `src/audio/AudioMixer.ts`, `src/audio/VehicleAudioEmitter.ts`, `src-tauri/src/window_manager.rs` | **Issue 6** (proper `ctx.suspend()`/`resume()` or fast-path unmute without 80ms recheck + 50ms ramp stall on `rc-tauri-window-shown`; resync `VehicleAudioEmitter` on resume) | `repro-6` + manual Tauri Windows/macOS background/foreground test (`driftMs < 20ms`) |

**Merge Order**:
1. Merge `fix/vnext-camera-fixed-120hz`, `fix/tauri-audio-focus-resume`, and `fix/vnext-replay-456-hud-and-loadouts` first (zero overlap).
2. Merge `fix/vnext-broadcast-interp-and-exhibition`, then rebase & merge `fix/vnext-goal-replay-team-anthem-cam` (both touch Goal Replay coordination).
3. Run full gate: `pnpm run check && pnpm run repro:vnext-issues && pnpm run repro:dev-broadcast-v3`.

---

## 3. Architecture Recommendation: Real-Time Telemetry Export (Tauri) & Remote Live Debugging

### Why Not Stream 60Hz/120Hz Telemetry Over Supabase Realtime?
- **Bandwidth & Quota Math**: A single `GameStateBufferPod` is 448 floats = `1.79 KB` binary (or `~3–5 KB` JSON). At 60Hz, that is **60 msg/s** (~`108–300 KB/s`, **388 MB–1 GB/hour**), which immediately hits Supabase Realtime's **100 msg/s channel rate limit** and burns through free-tier **2M messages/month** and **5 GB egress**.
- **Recommended Split-Plane Architecture** (reusing existing `src-tauri/src/tunnel_bridge.rs` + `SupabaseRealtimeSignalingClient.ts`):
  1. **Control Plane (Supabase Realtime)**: Keep using `SupabaseRealtimeSignalingClient` **only** for session discovery, WebRTC SDP/ICE handshake, and lightweight scenario commands (`"start_scenario"`, `"mark_incident"`, `"stop_recording"`). Traffic: `< 20 messages/session`.
  2. **Data Plane — Live Streaming (WebRTC DataChannel or Tauri `tunnel_bridge.rs`)**:
     - Stream **delta-compressed binary `GameStateBufferPod`** frames over an unordered/unreliable **WebRTC DataChannel** (or Tauri's existing `tokio-tungstenite` / `wtransport` server in `src-tauri/src/tunnel_bridge.rs` exposed via Cloudflare Tunnel / Tailscale / UPnP).
     - Binary delta + zstd/deflate reduces 60Hz 3v3 telemetry to **~300–500 bytes/frame (~20–30 KB/s)** with **zero Supabase quota**.
  3. **Tauri Real-Time Local Disk Export**:
     - Add a lightweight Tauri Rust command (`append_telemetry_chunk` writing to a `BufWriter<File>` in `.rcreplay` / `.rlrc` or raw binary ring-buffer format).
     - On scenario end (or ring-buffer trigger), compress the slice (`.rcgr.gz` / `.rcreplay`) so you can record individual scenarios in seconds and share/replay them deterministically without continuous cloud streaming.
