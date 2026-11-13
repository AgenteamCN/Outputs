# Wave 4 Implementation Report (`z/agy/rc/rust/wave4-room-kernels-csr-soak`)

- **Repository**: `Antigravi-ty/RocketCompanion`
- **Base Commit**: `b88102a`
- **Branch**: `z/agy/rc/rust/wave4-room-kernels-csr-soak`
- **Verified Bot Commit**: `50b9b01d92d2a1c62aa1f7c5129bb16e5aab4157` (`https://github.com/Antigravi-ty/RocketCompanion/commit/50b9b01d92d2a1c62aa1f7c5129bb16e5aab4157`)

## 1. Summary of Changes

### 1.1 Room Activity Protocol & Hot-Swappable Kernels (`net-04-room-orchestrator-and-kernels`)
- **`src/network/protocol/RoomActivityProtocol.ts`**: Defined `RoomActivityId` (`freeplay | match | replay_intervene | sandbox`), `ACTIVITY_MODE_FLAGS` bitmask (`SKIP_SCORE_CLOCK`, `SKIP_TELESTRATOR`, `CSR_FOLLOW_HOST`, `SPARSE_FREEZE_MODE`, `TRAINING_RULESET`), `RoomActivitySwitchPayload`, `RoomActivityStateSummary`, `SpectatorTimelineCommand`, `InterveneAuthorizationRequest`, `InterveneHandoffState`, `CsrFollowHostPayload`, and 8-byte binary codec (`encodeCsrFollowHostPacket` / `decodeCsrFollowHostPacket`).
- **`src/network/protocol/HostWorkerProtocol.ts`**: Added `ROOM_ACTIVITY_SWITCH`, `SPECTATOR_TIMELINE_CMD`, `INTERVENE_HANDOFF_CMD`, and `CSR_FOLLOW_HOST_SYNC` to `HOST_WORKER_MSG_TYPES`.
- **`src/network/server/activities/`**:
  - `ActivityKernelPort.ts`: Narrow interface & context port for hot-swappable room kernels.
  - `FreeplayActivityKernel.ts`: Decoupled multiplayer Freeplay kernel (no scoring/clock freeze, unlimited boost, `disableGoalReset`, ball reset, ghost/solid collision hot-toggle).
  - `MatchActivityKernel.ts`: Pure competitive Match kernel delegating to `MatchSessionAdmin` without freeplay/sandbox flags.
  - `SandboxActivityKernel.ts`: Developer Sandbox kernel supporting custom state broadcast and experimental mode switching.
  - `ReplayInterveneActivityKernel.ts`: Coordinates `SpectatorRoomTimelineController` and `InterveneHandoffCoordinator`.
  - `RoomActivityOrchestrator.ts`: Hot-swaps active kernels without disconnecting peer sessions and tracks `CsrFollowHostPayload` broadcast state.
- **Router & Host Wiring**:
  - Updated `src/network/server/workerReplayAndDiagnosticsRouter.ts` (`WorkerRouterContext.activityOrchestrator`), `src/network/server/HostWorkerMessageRouter.ts` (`buildWorkerRoomStateSummary`, `routeProtocolMessageInWorker`), and `src/network/server/AuthoritativeServerHost.ts` (`switchRoomActivity`, `sendSpectatorTimelineCommand`, `requestInterveneHandoff`, `broadcastCsrFollowHostState`).

### 1.2 Spectator Room, 4-3-2-1-0 Intervene Handoff & CSR Follow-Host (`net-05-spectator-csr-and-intervene-handoff`)
- **`src/network/server/spectator/SpectatorRoomTimelineController.ts`**: Authoritative host-controlled replay timeline (`play`, `pause`, `seek`, `set_rate`, `step_frame`) with sparse frozen-frame ping gating (`isSparseFrozenMode`, `shouldEmitSparsePing`) and `MatchSystemControlSubController.enterStateOnlyMode` integration.
- **`src/network/server/spectator/InterveneHandoffCoordinator.ts`**: Role-authorized takeover (`host_self` or trainee `sessionId`), 4-3-2-1-0 countdown progression, and non-POV policy enforcement (`disappear | no-input | follow-state | bot-takeover`) at takeover completion.
- **`src/ui-core/stores/csrFollowHostStore.ts`**: Pure Zustand store in `src/ui-core/stores/` (zero engine/network imports, compliant with `src/ui-core/AGENTS.md`) holding client CSR follow-host state.
- **`src/ui-system/core/csrFollowHostBridge.ts`** & **`src/ui-system/GameUIRoot.tsx`**: Bridges `useCsrFollowHostStore` with `useQuickNotificationStore` to display a persistent low-priority `background` notification with one-click `Follow Host (CSR)` toggling, and provides `buildTakeoverHudPayloadFromHandoff` for `TakeoverCountdownHUD`.

### 1.3 Verification & Architecture Audit (`net-06-multiplayer-soak-and-final-radar`)
- **`scripts/runMultiplayerRoomRepro.mjs`**: Added `--mode=freeplay` and `--mode=room-switch` CLI mode support.
- **`scripts/verifyWave4Multiplayer.ts`**: End-to-end deterministic verification suite covering CSR binary codec roundtrip, `RoomActivityOrchestrator` hot-switching (`freeplay -> match -> replay_intervene -> sandbox`), `SpectatorRoomTimelineController` sparse ping gating, `InterveneHandoffCoordinator` 4-3-2-1-0 countdown & `disappear` non-POV policy, and `csrFollowHostBridge` notification lifecycle.

## 2. Verification Results
- `pnpm run check` (`radar` + `tsc --noEmit`): **PASS** (`STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified`, 0 Circular Deps, 0 Value Re-Exports, 0 Upward Imports).
- `pnpm run radar:audit`: **PASS** (0 blocking issues).
- `pnpm run build:wasm:check`: **PASS** (`cargo check --target wasm32-unknown-unknown --workspace` finished cleanly in 8.92s).
- `scripts/verifyWave4Multiplayer.ts` (bundled via `esbuild` + `node`): **PASS** (`[PASS] Wave 4 Multiplayer Verification: 11 worker envelopes verified.`).
- Note on `pnpm run test:baseline`: Pre-existing `b88102a` baseline had 2 unrelated failures in `multiplayer_6car_loadout_stress` (`multiplayer_no_phantom_double_jump_on_flip_e`) and `training_and_kickoff` (`kickoff_ppo_ghost_trajectory_and_episode_ste`); all other checks and matrix assertions passed.

## 3. Open Architectural Notes for Downstream Merge
- When Waves 1–3 merge with Wave 4, `RoomActivityOrchestrator` can feed `ACTIVITY_MODE_FLAGS` directly into the Wave 3 `RoomReplicationBus` frame header to strip unused payload sections (score/clock in Freeplay, telestrator in Match, sparse ping in frozen Replay).
- Full live WebRTC multi-tab CSR follow-host camera lock can consume `useCsrFollowHostStore` in `CameraManager` / `FreeCameraController` once Wave 1 (`sandbox-01`) and Wave 3 (`net-03`) are merged.
