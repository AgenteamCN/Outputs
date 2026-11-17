# Phase 4 Multiplayer Sync, Authority Camera, & Coach Handover Report

**Commit**: `8f459e4bf4256b4a3bd60fb16e352beba14400a6` (`origin/z/agy/rc/multiplayer-recheck`, `verified: true` as `antigravity-fine-grained-token[bot]`)

---

## 1. Summary of Implemented Changes

### 1.1 Edge-Triggered Authority Camera Sync & Non-Override (`camera_state_sync`)
- **`src/game/pipeline/stages/AuthorityCameraSyncTracker.ts`**:
  - Tracks local camera state (`isFreeCam`, `ballCam`, `focusedCarIndex`, `freeCamPosition`, `freeCamTarget`) and emits a `camera_state_sync` control frame **only on edge transitions** while the local peer holds control authority.
  - Rejects stale or out-of-order incoming camera syncs via `(authorityEpoch, cameraSyncSeq)` fencing, and never applies incoming camera syncs when the local peer is the active authority.
  - While the authority holder does not switch cameras, other peers are free to switch their own cameras locally without periodic workspace/replay heartbeat overrides.
- **Wire & Transport Plumbing**:
  - `src/network/protocol/GlobalWorkspaceProtocol.ts`: Added `cameraSyncSeq`, `authorityEpoch`, `authorityPeerId` to `WorkspaceCameraSyncPayload` and `AuthorityCameraSyncPayload`.
  - `src/network/transport/webrtc/dataChannelControlFraming.ts`, `webrtcHandshakePipeline.ts`, `DataChannelTransport.ts`, `P2PWebRTCChannel.ts`, `HostSessionController.ts`, `MultiplayerSessionManager.ts`: Wired `camera_state_sync` control frames with Host fan-out to all other connected peers.
  - `src/game/runtime/devSyncChannelHandlers.ts`: Extracted Dev Sync client channel bindings (keeping `RuntimePostprocessingAndSessionBootstrap.ts` under Radar's 18 KB cap) and gated `applyCameraSyncToPort` via `sharedAuthorityCameraSyncTracker.shouldApplyIncomingCameraSync`.
  - `src/game/runtime/OnlineMatchControlBridge.ts` & `src/replay/replayOnlineSyncHelper.ts`: Attached the current `cameraSyncSeq` to workspace/replay heartbeats so periodic broadcasts do not override client-local camera choices.

### 1.2 Coach Authority Handover & Host Spectating Student
- **`src/game/runtime/CoachAuthorityHandoverController.ts`**:
  - Supports `'host_only' | 'client_only' | 'fpv_master_slave'` (normalizing `'explicit'`).
  - `hasActiveCoachControlInput` checks `throttle`, `steer`, `jump`, `boost`, and `handbrake`.
  - Provides immediate Host preemption on active Coach input, **300ms** (`DEFAULT_IDLE_HANDOVER_MS`) idle auto-handover back to the student peer when `clientInputEnabled === true`, and monotonic `authorityEpoch` fencing.
- **`src/network/transport/hostCrossTabAndAuthority.ts` & `src/game/pipeline/stages/SimulationStage.ts`**:
  - `HostInputAuthorityTracker` preserves `latestFrame` on reclaim, queues up to 32 frames in `pendingFrames` (`consumePendingFrames()`), and syncs handover state.
  - On `reclaim_to_host`, Host applies `lastClientFrame.snapshot` before resuming local physics stepping so there is no position snap-back.
  - While Student holds authority, Host runs `hostAuthoritySpectateBuffer` (`SpectateInterpolationBuffer`) to smoothly spectate the Student.
- **Bug Fixes Discovered During E2E Verification**:
  - **`src/network/sync/ReplicationStateInterpolator.ts` & `src/replay/ReplayFrameInterpolator.ts`**: Replaced hardcoded `< 410` float length checks in `SpectateInterpolationBuffer.pushFrame` and `applyReplayFrameToEngine` with dynamic `SIM_OFFSETS.CARS + numCars * 40` validation so 1-car compact snapshots (`62` floats) from `ClientAuthorityPublisher` are accepted and interpolated cleanly.
  - **`src/game/runtime/OnlineMatchControlBridge.ts`**: When `setClientInputEnabled(true)` is called in a multi-client room, Host targets `getConnectedPeerIds()[0]` instead of `'any'` so multiple clients do not send competing `ClientAuthorityFrame`s.
  - **`src/game/pipeline/stages/HostBroadcastPublisher.ts` & `simulationNetworkReconcilePipeline.ts`**: Ensured Client falls back to Pure CSR Spectate whenever `activeInputPeerId === 'host'`, even when `clientInputEnabled === true` in `fpv_master_slave` mode.

### 1.3 Expanded Unit & E2E Test Coverage
- **`scripts/runMultiplayerSuiteTests.ts`**: Added Tests 20 (`CoachAuthorityHandoverController`) and 21 (`AuthorityCameraSyncTracker`).
- **`scripts/runDevSyncReproSuite.mjs`**: Upgraded Scenario 3 (Replay cursor advance + Pause ACK + Seek + Resume) and added Scenarios 7 (Dynamic Training Pack Shot & Multi-Client Sync), 8 (1 Host + 2 Client Telestrator `0x05` Sync & Gating), and 9 (Edge-Triggered Authority Camera Sync & FPV 300ms Idle Handover).

---

## 2. Verification Results

| Check / Suite | Command | Result |
| :--- | :--- | :--- |
| Radar + TypeScript | `pnpm run check` | **PASS** (1424/1424 TS files, `tsc --noEmit` clean, all volume/boundary rules green) |
| Multiplayer Unit/Loopback Suite | `pnpm run test:multiplayer-suite` | **PASS** (21/21) |
| Dev Room Sync E2E | `pnpm run test:dev-room-sync-e2e` | **PASS** (9/9 stages, `runtime_error=0`, `poseAnomalies=0`, `stuckInGoal=false`) |
| Dev Sync Repro Suite | `pnpm run repro:dev-sync-bugs` | **PASS** (9/9 scenarios, `reproducedBugs=0`) |
