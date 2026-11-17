# RocketCompanion Multiplayer Recheck — Phases 1–3 & Phase 4A Report

**Branch:** `z/agy/rc/multiplayer-recheck`  
**HEAD Commit:** `c50af67d01e3fc995630367bbffc7eb332d89f43` (`origin/z/agy/rc/multiplayer-recheck`, Verified Bot Commit)

---

## 1. Issues Fixed & Bugs Resolved

### Phases 1–3 (Confirmed Complete & Verified)
1. **Bug A — Host Running `PredictionReconciler` Against Competing `AuthoritativeServerWorker` (Exhibition 3v3 & Real `.RCReplay`)**:
   - Separated **BroadcastSpectatePipeline** (Dev Sync Room / Pure CSR) from **MatchModePipeline** (forward-predicted online match).
   - Host's main-thread physics (`Freeplay`, `ExperimentalMultiBotsMode`, `StandaloneReplayViewerController`, `StaticTrainingPackMode`) is the sole source of truth and no longer runs `PredictionReconciler` / `syncClientCarTopology` against `AuthoritativeServerWorker`.
   - `HostBroadcastPublisher` captures compact physics snapshots (`saveCompactState()`) and streams them via `AuthoritativeServerHost.enterStateOnlyMode` / `pushBroadcastSnapshot` to all connected clients.
   - Client runs `PureCsrInterpolationReceiver` backed by `SpectateInterpolationBuffer` (reverse-delayed jitter/interpolation buffer) and `applyReplayFrameToEngine`.
2. **Bug B — Reliable Pause/Resume ACK & Low-Frequency Presence Heartbeat**:
   - Implemented `HostPauseAckController` + `pause_state_sync` / `pause_state_ack` / `presence_ping` protocol with `pauseSeq` monotonicity, 100ms retry loop until all connected peers ACK, dead-peer timeout (`5000ms`), and 1–2Hz `presence_ping` heartbeat while paused (`pausedIdle`).

### Phase 4A (Multi-Client Room Fanout, Cross-Tab Isolation & Late-Joiner Sync)
1. **Bug C — Multi-Client (2+ Clients) Dev Sync Room Failure**:
   - **Per-Peer Cross-Tab Signaling Isolation (`CrossTabSignaling`, `webrtcHandshakePipeline`, `HandshakeSession`, `DataChannelTransport`, `P2PWebRTCChannel`)**:
     - Attached `senderId` (`localPeerId`) and `targetPeerId` to room `BroadcastChannel` control/handshake/packet messages (`p2p_packet` envelope for binary/string payloads).
     - Filtered out self-echo (`senderId === localPeerId`), messages addressed to other peers (`targetPeerId !== localPeerId`), and on Host isolated per-peer channel traffic (`state.role === 'host' && state.peerId && msg.senderId !== state.peerId`).
   - **Dynamic Multi-Peer Spawning on Host (`HostSessionController`, `hostCrossTabAndAuthority`, `hostPauseAckBridge`)**:
     - Added room-level `cross_tab_join` listener on Host that renames unused `peer_1` for Client 1 and dynamically spawns new `P2PWebRTCChannel` instances for Client 2..N up to `maxBroadcastChannels`.
2. **Late-Joiner Sync (`HostSessionController`, `HostBroadcastPublisher`, `OnlineMatchReconcilerBinder`)**:
   - On `onConnected`, Host replays `activeGlobalWorkspace` and `pauseAckController.onPeerConnected(peerId)` (`pause_state_sync`) to late-joining clients.
   - `HostBroadcastPublisher` detects `connectedPeersCount` increases and forces a snapshot publish even while paused or static.
   - `OnlineMatchReconcilerBinder.handleHandshake` preserves `matchMode` already synced by `global_workspace_broadcast` instead of overwriting it with `'freeplay'`.

---

## 2. Verification Results

| Suite | Command | Result | Notes |
| --- | --- | --- | --- |
| Radar + TypeScript Strict | `pnpm run check` | **PASS** | `1421/1421` TS files, `tsc --noEmit` clean, 0 circular deps, all volume caps green |
| Multiplayer Unit/Loopback Suite | `pnpm run test:multiplayer-suite` | **PASS (19/19)** | Includes Tests 16 (InterpolationBuffer), 17 (Pause ACK), 18 (Multi-Client CrossTab Isolation & Late-Join), 19 (Authority Handover & FPV Epoch Fencing) |
| Dev Room Sync E2E | `pnpm run test:dev-room-sync-e2e` | **PASS (9/9 stages)** | `runtime_error=0`, `poseAnomalies=0`, `stuckInGoal=false` |
| Dev Sync Bug Repro Suite | `pnpm run repro:dev-sync-bugs -- --report-only` | **5/6 PASS** | Scenarios 1–5 PASS (Freeplay, Exhibition 3v3, 3v3 RCReplay + Pause ACK, Static Training Pack, Multi-Client 2+ Peers); Scenario 6 (`Client Input Handover`) remains `REPRODUCED_BUG` (client moved locally to `[1778, 2290, 17]`, host stayed at `[2048, 2560, 17]`) for Phase 4B end-to-end E2E wiring |

### Repro Suite Scenario Breakdown (`/tmp/dev-sync-repro-report.json`)
1. `1. Freeplay Live Driving Sync` — **PASS** (`hostCar0: [-1302, 1814, 17]`, `clientCar0: [-1356, 1868, 17]`, `dist: 76`)
2. `2. Exhibition 3v3 Spectating` — **PASS** (`hostMode: "exp-multi-bots"`, `hostCars: 6`, `clientMode: "exp-multi-bots"`, `clientCars: 6`)
3. `3. Real 3v3 RCReplay Load & Playback + Pause ACK` — **PASS** (`hostCars: 6`, `clientCars: 6`, `ballDist: 0`, `hasPauseAckProtocol: true`)
4. `4. Static Training Pack Pose Sync` — **PASS** (`hostMode: "static-training-pack"`, `clientMode: "static-training-pack"`, `carDist: 0`)
5. `5. Multi-Client (2+ Clients) Dev Sync Room` — **PASS** (`hostConnectedPeers: 2`, `hostMode: "static-training-pack"`, `client2Mode: "static-training-pack"`, `client2Car0: [0, 2400, 17]`)
6. `6. Client Input Handover (clientInputEnabled=true)` — **REPRODUCED_BUG** (`clientInputEnabledOnClient: true`, `hostCar0Before: [2048, 2560, 17]`, `hostCar0After: [2048, 2560, 17]`, `clientCar0After: [1778, 2290, 17]`)
