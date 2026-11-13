# Track D Deliverable Report: Unified RoomReplicationBus & Hot-Swappable Activity Orchestrator

- **Branch**: `z/agy/rc/track-d-replication-bus`
- **Verified Bot Commit**: `cf5cfeee60fc0cd54eb2dcdc13dbc2aa845d80a1` (`isValid: true`, `state: VALID`)
- **Verification**:
  - `npx prettier --check` on all 14 Track D files: **PASS**
  - `pnpm run check` (Radar 2.0 + `tsc --noEmit`): **PASS (GREEN)** — `1359/1359` TS files, `0` circular deps, `0` upward imports, `0` orphan modules, `0` TypeScript errors.
  - Node runtime assertions (`ReplicationBusProtocol`, `AdaptiveReplicationPacer`, `SyncAuthorityRouter`, `ReplicationStateInterpolator`, `RoomActivityOrchestrator`, `HostWorkerMessageRouter`): **ALL PASSED**.

---

## 1. Implemented Modules & Architecture

### A. Dual-Mode Replication Protocol (`src/network/protocol/`)
- **`ReplicationBusProtocol.ts`**:
  - Dual replication modes: `REPLICATION_MODES.CONTINUOUS` (`'continuous'`, 60/120Hz) and `REPLICATION_MODES.SPARSE` (`'sparse'`, frozen frame + 2Hz 16-byte Ping-Pong).
  - Dynamic packet section bitmask (`REPLICATION_PACKET_FLAGS`: `STATE_SNAPSHOT`, `USER_INPUT`, `CLOCK_AND_SCORE`, `TELESTRATOR_SYNC`, `CSR_SUMMARY`, `SPARSE_FROZEN`, `FOLLOW_HOST_CAMERA`, `EDGE_LOCK_ACTION`) and `resolvePacketFlagsForActivity(...)` so Match, Freeplay, Replay, and Sandbox only carry the sections they need.
  - Hot-switch & CSR contracts: `RoomActivitySwitchPayload`, `StatePausedAckPayload`, `CsrStateBroadcastPayload`.
  - Exact 16-byte binary Ping-Pong codec: `encodeSparsePingPong16B`, `isSparsePingPong16B`, `decodeSparsePingPong16B` (`PING_PONG_16B_MAGIC = 0x52`, `PING_PONG_16B_SIZE = 16`).
- **`HostWorkerProtocol.ts`**:
  - Added `ROOM_ACTIVITY_SWITCH`, `REPLICATION_MODE_SET`, `STATE_PAUSED_ACK`, and `ROOM_CSR_STATE_BROADCAST` to `HOST_WORKER_MSG_TYPES`.

### B. Dual WebRTC DataChannel Multiplexing & Adaptive Pacing (`src/network/transport/webrtc/` + `src/network/transport/`)
- **`HandshakeSession.ts`**:
  - Host now creates both `'game_net'` (`ordered: false, maxRetransmits: 0`, unreliable snapshot/input channel) and `'game_ctrl'` (`ordered: true`, reliable control/signaling channel) during `createOfferToken`.
- **`DataChannelTransport.ts` & `dataChannelFramingPipeline.ts`**:
  - `bindDataChannel` routes `'game_ctrl'` to `bindControlDataChannel` while maintaining `'game_net'` for high-frequency snapshots/inputs.
  - `transmitReliableControlFrame` prefers `'game_ctrl'`, falling back to `'game_net'` or `BroadcastChannel` if `'game_ctrl'` is not open.
  - `handleIncomingMessage` handles 16-byte binary sparse Ping-Pong frames (`isSparsePingPong16B`) and `room_activity_switch` / `replication_mode_set` / `state_paused_ack` / `room_csr_state_broadcast` control frames.
- **`AdaptiveReplicationPacer.ts`**:
  - Congestion-aware downscaling (120Hz -> 60Hz -> 30Hz) based on `rttMs`, `packetLossRate`, and `bufferedAmountBytes`, plus 2Hz (`500ms`) sparse Ping-Pong cadence.
- **`P2PWebRTCChannel.ts`**:
  - Exposed `sendReliableControl`, `sendRoomActivitySwitch`, `sendStatePausedAck`, `sendCsrStateBroadcast`, and `setReplicationMode`.

### C. Entity Authority Routing & Steady-State Interpolation (`src/network/sync/`)
- **`SyncAuthorityRouter.ts`**:
  - Entity-level physical authority routing (`'host' | 'peer' | 'replay_readonly' | 'takeover_handoff'`) with `applyActivitySwitch`, `beginTakeoverHandoff`, `commitTakeoverHandoff`, `revertToReplayReadonly`, `canAcceptClientInput`, and `followHostEnabled` / `latestCsrSummary` state for Client-Side Rendering (CSR) / Follow-Host mode.
- **`ReplicationStateInterpolator.ts`**:
  - Reuses `interpolateRigidBodyHermiteSlerp` and `vDist` from `src/replay/rlrc/synthesis/RLRCHermiteSlerpMathPipeline.ts` (cycle-free) for smooth Hermite + SLERP rigid-body resampling, teleport snapping (`maxTeleportDistanceUu = 450`), and frozen-frame hold in `'sparse'` mode.

### D. Hot-Swappable Activity Orchestrator (`src/network/server/`)
- **`activities/ActivityKernel.ts`**:
  - Defines `RoomActivityKernel` and `ActivityKernelContext`, plus lightweight default kernels (`MatchActivityKernel`, `FreeplayActivityKernel`, `ReplayActivityKernel`, `SandboxActivityKernel`) ready for Track E to expand.
- **`RoomActivityOrchestrator.ts`**:
  - Manages active kernel hot-swapping (`switchActivity` with epoch increment and `STATE_PAUSED_ACK` barrier clearing), `setReplicationMode`, `recordStatePausedAck`, and `buildCsrBroadcast`.
- **`HostWorkerMessageRouter.ts`, `workerReplayAndDiagnosticsRouter.ts`, `AuthoritativeServerHost.ts`**:
  - Wired `RoomActivityOrchestrator` into `HostWorkerMessageRouter` (`MSG.ROOM_ACTIVITY_SWITCH`, `MSG.REPLICATION_MODE_SET`, `MSG.STATE_PAUSED_ACK`, `MSG.ROOM_CSR_STATE_BROADCAST`) and exposed `switchActivity`, `setReplicationMode`, `sendStatePausedAck`, and `broadcastCsrSummary` on `AuthoritativeServerHost`.

---

## 2. Open Design Items for Cross-Team Alignment (Track E / F / UI)
1. **Follow-Host CSR Notification & Opt-In UX**: The protocol and `SyncAuthorityRouter` now carry `followHostEnabled` and `CsrStateBroadcastPayload`, while the low-priority persistent notification ("Completely Follow Host" toggle) should be wired in `ui-core` / `ui-system` once the overlay team finalizes the notification slot.
2. **Training/Freeplay Packet Compression**: `ACTIVITY_DEFAULT_PACKET_FLAGS` omits `CLOCK_AND_SCORE` for `freeplay`; Track E can further customize `FreeplayActivityKernel` and `DevSandboxKernel` inside `src/network/server/activities/` without touching the bus layer.
3. **Headless Multi-Client Repro**: `scripts/runMultiplayerRoomRepro.mjs` can be extended in Track E (`branch18`) to exercise `ROOM_ACTIVITY_SWITCH` (`freeplay -> match -> replay -> freeplay`) across headless Chrome peers.
