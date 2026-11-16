# RocketCompanion Multiplayer & UI PR Report

## 1. UI Automated Scan PR & Merge (`z/agy/ui/phase1-tier-orthogonal-variants` -> `alpha`)
- Created PR #25 (`https://github.com/Antigravi-ty/RocketCompanion/pull/25`) from `z/agy/ui/phase1-tier-orthogonal-variants` into `alpha`.
- Merged cleanly into `alpha` (`merged: true`).

## 2. Root Cause Analysis: Dev Sync Room Host Dropping to `NOT HOSTING` & Client `RTCErrorEvent`

### What happened in the logs
1. **Client log**: `game_net` and `game_ctrl` opened, `color_change` was received, and `PredictionReconciler` snapped its timeline to server tick `2731`. Immediately afterward, both DataChannels received `OperationError: User-Initiated Abort, reason=Close called`, which happens when the remote peer (Host) tears down the `RTCPeerConnection` / `RTCDataChannel`.
2. **Host log (Tauri)**: After `DataChannel "game_net" OPEN!` and `Remote answer SDP applied`, the Host UI switched to `NOT HOSTING` and WKWebView logged `WebSocket connection to 'ws://127.0.0.1:53286/tunnel?token=...' failed: The operation couldn't be completed. Socket is not connected` (without `[LocalBridgeClient]` error logs, proving `LocalBridgeClient.close()` was invoked intentionally via `stopHosting()` -> `closeSessionTunnel()`).

### Why it happened (Universal Lifecycle Bug, Not a Tauri-only Edge Case)
- In `src/game/runtime/MatchModeTransitionBridge.ts` and `src/ui-system/core/runtimeModeBridgeHandlers.ts`:
  - `stopActiveOnlineOrLoopbackSessionIfNeeded` (called by `transitionToTrainingMode` and `transitionToExpMultiBotsMode`) and `returnToFreeplay` unconditionally called `teardownMultiplayerSignalingSession` (`session.stopHosting()`) and `teardownNetworkCoordinatorSession` (`stopOnlineServer()`).
  - `onReturnToFreeplay()` unconditionally reset `onlineSession` with `isHosting: false, roomId: ''`.
  - Worse, commit `df4a3c8` routed "Leave Room" in `MenuContainer.tsx` and `menuRouteNavigationPipeline.ts` to `onReturnToFreeplay()` when `isDevSyncRoom` was true, assuming `onReturnToFreeplay()` would keep the Dev Sync Room alive — when in fact `onReturnToFreeplay()` tore down the entire Host session!
  - Whenever the Host switched modes (Freeplay, Training, Replay Viewer, Dev Test Lab, Exp Multi Bots) or returned to Freeplay, `stopHosting()` was triggered, closing the Tauri Rust tunnel WS bridge and all WebRTC DataChannels.

### Why `test:multiplayer-suite` did not catch it
- `scripts/runMultiplayerSuiteTests.ts` previously only tested `P2PWebRTCChannel`, `HostSessionController`, `MultiplayerSessionManager`, and `PredictionReconciler` in isolation, and never exercised `MatchModeTransitionBridge` (`stopActiveOnlineOrLoopbackSessionIfNeeded` / `returnToFreeplay`) with `onlineSession.isDevSyncRoom === true`.
- `scripts/runDevRoomSyncE2E.mjs` requires full browser bootstrap with local private microkernel/assets (`ERR_POV_MICROKERNEL_TAMPERED` when those untracked assets are absent).

### Secondary Issue: Client Default Spectate State on Join
- In `src/game/runtime/OnlineMatchReconcilerBinder.ts`, when a Client received `handshake_ack` with `isDevSyncRoom: true`, `clientInputEnabled` was not gated to `false` until a subsequent `GlobalWorkspaceBroadcast` arrived, and `lobbyDualWorld.focusCameraOnCar(activePlayerCarIndex)` focused on `entityId` (car `1`, which does not exist in Host Freeplay) instead of Host car `0`.

## 3. Fixes Applied (`z/agy/rc/multiplayer-recheck`, commit `74fe936`)
- `src/game/runtime/MatchModeTransitionBridge.ts`: Skip `teardownMultiplayerSignalingSession` and `teardownNetworkCoordinatorSession` in `stopActiveOnlineOrLoopbackSessionIfNeeded` and `returnToFreeplay` when `useAppStore.getState().onlineSession?.isDevSyncRoom` is `true`.
- `src/ui-system/core/runtimeModeBridgeHandlers.ts`: Preserve `onlineSession` in `onReturnToFreeplay` when `onlineSession?.isDevSyncRoom` is `true`.
- `src/game/runtime/OnlineMatchReconcilerBinder.ts`: On `msg?.isDevSyncRoom` handshake ack, default `reconciler?.setClientInputEnabled?.(false)` and focus camera on Host car `0`.
- `scripts/runMultiplayerSuiteTests.ts`: Added Test 15 (`Dev Sync Room Lifecycle Preservation on Mode Switch & Client Spectate Default`).

## 4. Verification
- `pnpm run check` (`radar` + `tsc --noEmit`): PASS (1415/1415 TS files, 0 errors).
- `pnpm run test:multiplayer-suite`: PASS (15/15 tests passed).
- Verified bot commit `74fe936e5eb11b5c3ffa2382b18460f6a8df029c` pushed to `z/agy/rc/multiplayer-recheck` (`signature.isValid: true, state: "VALID"`).
