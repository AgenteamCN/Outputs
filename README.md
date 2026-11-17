# RocketCompanion Developer Sync Room (Pure CSR) — Headless Bug Reproduction Report & Detailed Architecture Plan

**Repository:** `Antigravi-ty/RocketCompanion`  
**Branch:** `z/agy/rc/multiplayer-recheck`  
**Base HEAD:** `3918fae` -> **Updated HEAD:** `98efe40` (`test(dev-sync): add Linux headless repro suite for Dev Sync Room bugs & fix E2E HTTP default`)  
**Replay Fixture Tested:** `rc-replay-3v3-0-1-2026-11-17T05-00-24.RCReplay` (12,653 frames, 3v3)

---

## 1. Executive Summary & Linux Headless Test Results

We set up the full Linux headless toolchain (`public` submodule initialized, `pnpm install`, `google-chrome --headless=new` with SwiftShader WebGL + CDP) and scanned both the existing test suites and real user flows.

### 1.1 Existing Suites vs. Real-World Reproduction
1. `pnpm run check` (`radar` + `tsc --noEmit`): **PASS** (1415/1415 TS files).
2. `pnpm run test:multiplayer-suite`: **PASS** (15/15 unit/loopback tests).
3. `pnpm run test:dev-room-sync-e2e`: **Previously FAILED** on Linux headless due to `net::ERR_CERT_VERIFIER_CHANGED` on the second Chrome tab under Vite's self-signed `rc-https` mode. **Fixed** in `scripts/runDevRoomSyncE2E.mjs` by defaulting `RC_VITE_ROLE` to `rc-http` when unset (`runtime_error=0`, all 9 stages PASS).
4. **Why `test:dev-room-sync-e2e` missed the bugs you hit:**
   - Stage 6 (`Replay Watch Together`) never loaded a real `.RCReplay` file via `StandaloneReplayViewerController.handleUploadFile`; it manually called `authoritativeServer.enterStateOnlyMode(snap6, 500)`.
   - Exhibition (`onStartMatch` -> `exp-multi-bots`) was never tested in Dev Sync Room.
   - Multi-client (`2+` clients) and live Client driving handover (`clientInputEnabled=true`) were never tested end-to-end.

### 1.2 New Linux Headless Repro Suite (`pnpm run repro:dev-sync-bugs`)
Added `scripts/runDevSyncReproSuite.mjs` (report written to `/tmp/dev-sync-repro-report.json`). Running `pnpm run repro:dev-sync-bugs -- --report-only` against `z/agy/rc/multiplayer-recheck` produces:

| # | Scenario | Status | Observed Output / Root Cause |
|---|----------|--------|------------------------------|
| 1 | Freeplay Live Driving Sync | **PASS** | Host `car0=[1435,1947,17]`, Client `car0=[1420,1932,17]`, `dist=21` |
| 2 | Exhibition 3v3 Spectating (`onStartMatch`) | **REPRODUCED_BUG** | `hostCars=1`, `clientCars=1` (expected 6 on both) |
| 3 | Real 3v3 `.RCReplay` Load & Playback + Pause ACK | **REPRODUCED_BUG** | `hostCars=1`, `clientCars=6`, `hasPauseAckProtocol=false` |
| 4 | Static Training Pack Pose Sync | **PASS** (initial pose) | `hostCar0=[0,4608,17]`, `clientCar0=[0,4608,17]`, `carDist=0` |
| 5 | Multi-Client (2+ Clients) Dev Sync Room | **REPRODUCED_BUG** | `hostConnectedPeers=1`, `client2Mode="freeplay"` while Host is in `"static-training-pack"` |
| 6 | Client Input Handover (`clientInputEnabled=true`) | **REPRODUCED_BUG** | `clientInputEnabledOnClient=true`, but Client driving leaves Host `car0` unchanged at `[-256,3840,17]` |

---

## 2. Root-Cause Scan of Every Reproduced Bug

### Bug A (Critical): Host Runs `PredictionReconciler` Against a Competing `AuthoritativeServerWorker`
**Smoking-gun stack trace captured in headless Chrome on Host during BOTH Exhibition and Replay:**
```
syncClientCarTopology (src/network/client/NetworkTelemetryCollector.ts)
  <- ReconcilerRollbackStepper.stepReconcilePacket (src/network/client/ReconcilerRollbackStepper.ts)
  <- ReconcilerRollbackStepper.reconcileInboundBatch (src/network/client/ReconcilerRollbackStepper.ts)
  <- OnlineMatchReconcilerBinder (src/game/runtime/OnlineMatchReconcilerBinder.ts)
```
- **What happens in Exhibition (`ExperimentalMultiBotsMode`):**
  1. `ExperimentalMultiBotsRosterController.spawnRosterFromStore()` calls `physics.addCar(...)` 6 times (`spawnedCars=6`).
  2. `AuthoritativeServerWorker` is still running its own separate 1-car Freeplay simulation (`activeCarMask=0x01`).
  3. Host's own `PredictionReconciler` receives the 1-car `serverTick` packet from `AuthoritativeServerWorker`, calls `syncClientCarTopology`, and **deletes cars 1..5 from Host's `physics`**, dropping Host back to `numCars=1`!
  4. `AuthoritativeServerWorker` never receives Exhibition's 6-car state, so Client also stays at `numCars=1` while `GlobalWorkspaceBroadcast` mirrors `matchMode: 'exp-multi-bots'` to Client's UI.
- **What happens in Real `.RCReplay` (`StandaloneReplayViewerController.handleUploadFile`):**
  1. Host loads `rc-replay-3v3-0-1-2026-11-17T05-00-24.RCReplay` -> `rosterSubController.resolveAndApplyToArena()` spawns 6 cars (`numCarsImmediate=6`).
  2. `_emitStateTransition` sends `enterStateOnlyMode(frame0)` asynchronously to `AuthoritativeServerWorker`, so `AuthoritativeServerWorker` loads 6 cars and streams them to Client (`clientCars=6`).
  3. **Before** `reconciler.replayPlaybackMode = true` takes effect on Host, an in-flight 1-car `serverTick` packet from `AuthoritativeServerWorker` hits Host's `PredictionReconciler` -> `syncClientCarTopology` **deletes cars 1..5 on Host** (`numCarsAfter500=1`, `activeMask=1`)!
  4. This directly explains: *"replay 这边的一个问题是，两端还是会被设置为对方的一个状态"* — Host and Client are fighting over two independent physics worlds.

### Bug B: No Pause ACK / Retry Loop (`pauseAck` missing)
- `GlobalWorkspaceProtocol.ts` has `playbackPaused`, `pausedAtTick`, and `pauseHeartbeatHz`, and `ReplayOnlineHeartbeatTracker` periodically broadcasts workspace state.
- However, there is **no `pause_ack` message** from Client to Host, no per-client ACK tracking, and no reliable retry loop until ACK.

### Bug C: Multi-Client (`2+` Clients) Fails in Dev Sync Room
- When `Client2` joins the same `roomId` after `Client1`, Host `connectedPeers` remains `1` and `Client2` never receives `GlobalWorkspaceBroadcast` or state snapshots (`client2Mode="freeplay"` while Host is `"static-training-pack"`).
- Root cause: Cross-tab/signaling channel multiplexing in `P2PWebRTCChannel` / `HostSessionController` / `HostClientChannelRegistry` needs per-peer `peerId` isolation rather than single-room-broadcast assumptions.

### Bug D: Client Input Handover (`clientInputEnabled=true`) Does Not Move the Car
- In Dev Sync Room, Client joins as `role='spectator'` (`playerCarIndex=0`, `spectateMirrorMode=true`, `replayPlaybackMode` or `spectateMirrorMode` bypassing input send).
- Even when Host sets `clientInputEnabled=true`, Client's inputs are not forwarded to Host's active car `0` (nor does Client take authority over car `0`), so Client driving has zero effect.

---

## 3. Architectural Answers & Design Decisions

### 3.1 Spectating / Broadcasting vs. Match Gameplay (Do We Need Reverse Delay?)
**Yes — your intuition is spot-on.** Gameplay and Spectating have opposite requirements:
1. **Match Mode (Forward-Predicted Gameplay):**
   - Needs forward prediction (`PredictionReconciler` + input buffer lead ticks) so local inputs feel instant and arrive at the server before the server steps that tick.
2. **Developer Broadcasting Host / Spectating Mode (Pure CSR):**
   - **Zero forward prediction, zero rollback, zero local physics stepping** on the Client.
   - **Should Client render immediately on packet arrival or use a reverse-delayed interpolation buffer?**
     - **Use a small reverse-delayed Jitter/Interpolation Buffer**, NOT raw immediate-apply.
     - Why: Over real P2P/WebRTC, packets arrive with jitter (e.g. RTT=60ms, one-way=30ms ± 15ms jitter). If you apply every packet immediately on arrival, car/ball motion will micro-stutter whenever 2 packets arrive in one render frame and 0 in the next.
     - **Formula:** Target `renderTick = latestReceivedHostTick - interpDelayTicks`, where:
       - `interpDelayTicks = clamp(ceil((jitterMs * 2 + tickIntervalMs) / 8.333), 2, 12)` (at 60ms RTT with ~15ms jitter, `3..5` frames = `25..42ms` behind Host).
       - Client keeps a ring buffer of received `HostBroadcastFrame` snapshots sorted by `hostTick` and **interpolates** (`lerp` pos/vel, `slerp` quat) between `snap[floor(renderTick)]` and `snap[ceil(renderTick)]`!
       - **Important:** Do NOT use `RTT / 2` as extra delay on top of network transit — the packet already took `RTT / 2` to arrive; the buffer only needs to absorb **packet-interval + jitter** (`~2–5 frames` at 120Hz, i.e., `16–42ms`).

### 3.2 Should We Rewrite From Scratch? -> Clean Two-Archetype Split
You do **not** need to rewrite RocketSim, Three.js rendering, or WebRTC/Tunnel transport from scratch. You **do** need a clean architectural separation between the two core archetypes so every mode (Match, Spectate, Replay, Exhibition, Training) is a clean subset:

1. **Archetype 1: `MatchModePipeline` (Forward-Predicted Gameplay)**
   - Used for: Online 1v1/2v2/3v3 Match, Room Match Auto-Pull.
   - `AuthoritativeServerWorker` owns physics; all players (including Host) run `PredictionReconciler`.
2. **Archetype 2: `BroadcastSpectatePipeline` (Developer Broadcasting Host / Pure CSR)**
   - Used for: Freeplay Watch, Exhibition Spectating, Replay Watch Together, Match Spectate Mode (carved out by disabling dev-only windows/telestrator/input handover).
   - **Host side:**
     - Host's main-thread `physics` (whether Freeplay, `ExperimentalMultiBotsMode`, `StandaloneReplayViewerController`, or `StaticTrainingPackMode`) is the **sole source of truth**.
     - Host **MUST NOT** run `PredictionReconciler` or `syncClientCarTopology` against `AuthoritativeServerWorker`.
     - Host runs a lightweight `HostBroadcastPublisher` at the end of `SimulationStage` (only when `isDevSyncRoom` and not in `match` mode):
       - Captures `physics.saveCompactState()` + `cameraSync` + `modeId` + `tick` + `pauseState` (at 60Hz or 120Hz configurable) and fans it out to **all N connected peer channels** (`HostClientChannelRegistry`).
   - **Client side:**
     - Client runs `PureCsrInterpolationReceiver` (no `AuthoritativeServerWorker` rollback, no `syncClientCarTopology` fighting local modes):
       - Ensures `physics` car count matches `frame[SIM_OFFSETS.NUM_CARS]`.
       - Interpolates between buffered snapshots at `latestTick - interpDelayTicks` and calls `applyReplayFrameToEngine(physics, interpSnap, interpolator)`.
       - Mirrors `cameraSync` (FreeCam eye/target or BallCam + `focusedCarIndex`) and `GlobalWorkspaceBroadcast` (optionally gated by `followHostWorkspace`).

### 3.3 Replay Pause, Retry-Until-ACK, and Low-Frequency Heartbeat
- **Protocol addition (`0x06` or `GlobalWorkspaceProtocol` control frame):**
  - `PAUSE_STATE_SYNC`: `{ pauseSeq: number, paused: boolean, pausedAtTick: number, snapshotHash?: number }`
  - `PAUSE_STATE_ACK`: `{ pauseSeq: number, paused: boolean, pausedAtTick: number, peerId: string }`
- **Host state machine:**
  1. Host clicks Pause -> increments `pauseSeq`, stops streaming physics snapshots, enters `awaitingPauseAck` for all connected `peerIds`.
  2. While `unackedPeers.size > 0`, Host resends `PAUSE_STATE_SYNC` every `100ms` (with exponential cap or max 5s warning toast if a peer is dead).
  3. Once all peers ACK (`PAUSE_STATE_ACK` with matching `pauseSeq`), Host transitions to `pausedIdle`:
     - `0 Hz` physics state traffic.
     - `1–2 Hz` lightweight keepalive heartbeat (`{ type: 'presence_ping', pauseSeq, paused: true, pausedAtTick }`).
  4. On Resume -> increments `pauseSeq`, sets `paused: false`, resumes physics snapshot stream.

### 3.4 Telestrator (Drawing)
- **Already implemented and passing** (`Telestrator (0x05)` bidirectional stroke sync, 50-stroke stress test, `host_only` / `everyone` permission gating). **Keep as-is.**

### 3.5 Training / Coaching Session (Static & Dynamic Packs) + Control Handover
You compared two approaches for handing control from Coach (Host) to Student (Client):
- **Option 1 (Input Forwarding):** Client sends `CarControls` to Host -> Host simulates -> Host broadcasts state back.
  - *Pros:* Easy authority, Host never loses state.
  - *Cons:* Student feels `RTT` (60–120ms) input lag on every turn/flip, ruining muscle-memory training.
- **Option 2 (Authority Handover / FPV Master-Slave — Recommended):**
  - **Normal (Coach Demo):** Host is `inputAuthority = 'host'`. Host simulates locally (0ms lag) and broadcasts state to Client(s).
  - **Student Turn (`clientInputEnabled = true` / `inputAuthority = 'client'`):**
    1. Host sends `CONTROL_AUTHORITY_GRANT` (`targetPeerId`, `shotStateSnapshot`, `tick`).
    2. Student simulates **locally** at 120Hz (0ms input lag!) and streams `saveCompactState()` back to Host via the exact same `BroadcastFrame` format (`authorityEpoch`).
    3. Host switches to `PureCsrInterpolationReceiver` for the duration of the attempt and watches the Student in real time.
  - **FPV Master/Slave Override:**
    - When `authorityMode === 'fpv_master_slave'`:
      - If Host has **any non-neutral input** (`|throttle| > 0.05 || |steer| > 0.05 || jump || boost`), Host immediately increments `authorityEpoch`, sends `CONTROL_AUTHORITY_RECLAIM` (ignoring further Student frames with older `authorityEpoch`), and resumes local simulation from the latest received Student snapshot (or resets shot).
      - When Host is idle for `> 300ms` (or explicitly hands over), Student can drive.

### 3.6 "Looks Like You Are Playing" (100ms Idle Necto Takeover)
- **Recommendation: Defer to Phase 5 (after Core Broadcast/Exhibition/Replay/Training Sync are solid).**
- Keep it completely decoupled inside `RuntimeInputPoller.ts` / `TouchInputAgentController.ts` as an optional local input source (`idleBotTakeoverMs = 100`, default off) so it never entangles with network authority or `PredictionReconciler`.

---

## 4. Detailed Step-by-Step Implementation Plan (Executable by Any Engineer)

### Phase 1: Fix Host Self-Reconciliation & Pure CSR Broadcast Path (Fixes Bugs #2 & #3)
1. **Disable Host `PredictionReconciler` in Broadcasting Modes:**
   - Target files: `src/game/pipeline/stages/SimulationStage.ts`, `src/game/runtime/OnlineMatchReconcilerBinder.ts`, `src/game/runtime/MatchModeTransitionBridge.ts`.
   - When `onlineSession.isDevSyncRoom && activeRoomActivity !== 'match'`:
     - Skip `reconciler.reconcile()` / `syncClientCarTopology()` on Host so `AuthoritativeServerWorker` can **never** overwrite Host's Exhibition (`6` cars), Replay (`6` cars), or Training (`1` car) physics.
2. **Add `HostBroadcastPublisher` at the end of `SimulationStage`:**
   - Target files: `src/game/pipeline/stages/SimulationStage.ts`, `src/network/server/AuthoritativeServerHost.ts`, `src/network/transport/HostSessionController.ts`.
   - For Freeplay, `exp-multi-bots` (Exhibition), `static-training-pack`, `dynamic-training-pack`, and `replay-viewer`: Host pushes `physics.saveCompactState()` directly to `enterStateOnlyMode` / broadcast fan-out (throttled to 60Hz or 120Hz) so `AuthoritativeServerWorker` / Client always mirrors whatever Host's `physics` has, including all 6 cars in Exhibition and `.RCReplay`.
3. **Synchronous `replayPlaybackMode` & Car-Count Sync:**
   - Target files: `src/replay/StandaloneReplayViewerController.ts`, `src/replay/ReplayFrameInterpolator.ts`.
   - Ensure `applyReplayFrameToEngine` / `StandaloneReplayViewerController` syncs `addCarAtSlot` / `removeCar` cleanly when loading a 3v3 `.RCReplay`, and Host does not get reset by `OnlineRosterSyncCoordinator`.

### Phase 2: Client Pure CSR Interpolation Buffer (Reverse Delay)
1. **Implement `SpectateInterpolationBuffer`:**
   - Target files: `src/network/sync/ReplicationStateInterpolator.ts`, `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`.
   - Buffer incoming `BroadcastFrame`s by `hostTick`.
   - Dynamically compute `interpDelayTicks = clamp(ceil((jitterMs * 2 + 8.33) / 8.33), 2, 10)` (~25–50ms at typical RTT).
   - Interpolate car transforms (`pos`, `vel`, `quat`) and ball transform between bracketing frames instead of immediate-snapping.

### Phase 3: Reliable Pause/Resume ACK + Low-Frequency Heartbeat
1. **Extend `GlobalWorkspaceProtocol.ts` & `HostSessionController.ts`:**
   - Add `pauseSeq`, `PAUSE_STATE_SYNC`, `PAUSE_STATE_ACK`.
   - Host retries `PAUSE_STATE_SYNC` every `100ms` per un-ACKed `peerId` until all peers ACK; then drops physics stream to `0Hz` and keeps `1–2Hz` presence heartbeat.

### Phase 4: Multi-Client Fan-Out & Control Handover (Fixes Bugs #5 & #6)
1. **Fix Multi-Client Room Fan-Out (`HostClientChannelRegistry` / `P2PWebRTCChannel`):**
   - Ensure `HostSessionController` maintains independent `PeerSession` / `DataChannel` / `BroadcastChannel` routes per `peerId` so 2+ clients all receive `GlobalWorkspaceBroadcast` and `BroadcastFrame` simultaneously.
2. **Implement Authority Handover + FPV Master/Slave:**
   - Add `authorityEpoch` and `activeInputPeerId` to room state.
   - When `clientInputEnabled` is granted to a peer, Client runs local physics and streams `BroadcastFrame` back to Host; if Host moves (FPV override), Host increments `authorityEpoch` and immediately reclaims simulation authority.

### Phase 5: Carve Out Production "Match Spectating Mode" & Optional Necto Idle Takeover
1. **Match Spectating Mode:**
   - Reuse `BroadcastSpectatePipeline` with `devFloatingWindows = disabled`, `clientInputEnabled = false`, `followHostWorkspace = false` (Spectator keeps local camera/scoreboard control).
2. **Optional "Looks Like You Are Playing" (100ms Idle Necto):**
   - Implement in `RuntimeInputPoller.ts` behind an explicit toggle (`autoTakeoverIdleMs = 100`), completely isolated from network state.

---

## 5. Verification Gate
Once Phases 1–4 are implemented, the acceptance gate on Linux headless is:
```bash
pnpm run check
pnpm run test:multiplayer-suite
pnpm run test:dev-room-sync-e2e
pnpm run repro:dev-sync-bugs
```
All 6 scenarios in `repro:dev-sync-bugs` must transition from `REPRODUCED_BUG` to `PASS` (`reproducedBugs === 0`).
