# Developer Sync Room (Pure CSR) & Tauri Console Error Fix Report

**Repository:** `Antigravi-ty/RocketCompanion`  
**Branch:** `z/agy/rc/multiplayer-recheck`  
**Commit:** [`df4a3c80c569c21c1670d6027fed2fe3b2a3f570`](https://github.com/Antigravi-ty/RocketCompanion/commit/df4a3c80c569c21c1670d6027fed2fe3b2a3f570) (`verified: true`, signed bot commit)

---

## 1. Root Causes Identified

1. **Developer Sync Room was coupled to `online-warmup` / `lobby` UI & `leave-room` teardown:**
   - `Layer3DevUnderDevelopmentRecipe` launched the room via `onHostOnlineWarmup({ roomType: 'warmup', ... })` without an independent `isDevSyncRoom` flag.
   - `OnlineSessionCoordinator.hostOnlineServer` / `onlineSessionEnvironmentPipeline.finalizeHostLaunchState` forced `matchMode` to `'online-warmup'` and `OnlineMatchControlBridge` switched activity to `'multiplayer_training'`.
   - `Layer1MainMenuRecipe` / `deriveLayer1ContextVariation` saw `hasActiveOnlineRoom = true`, hijacking the main menu into `'online-warmup-host'` / `'online-warmup-client'` (replacing `Play` with `Manage Room` and putting `Leave Room` at the bottom).
   - Clicking `Leave Room` in `MenuContainer` or `menuRouteNavigationPipeline` called `bridge.onLeaveOnlineServer()`, which tore down the entire P2P WebRTC/Tunnel session instead of just returning to Free Play and syncing state.

2. **Host Workspace was not auto-synced (`Broadcast Sync Now` was manual-only):**
   - `GlobalWorkspaceBroadcast` was only sent when manually clicking `Broadcast Sync Now` or toggling switches in `DevRoomSyncFloatingWindow`.
   - Navigating menus (`activeRoute`, `isOpen`), changing `matchMode`, or opening/closing/minimizing floating windows on Host did not automatically broadcast to Clients.

3. **Host Tauri (`LocalBridgeClient`) WebSocket Console Error:**
   - The `UPNP_MAPPING_FAILED` and `WEBTRANSPORT_FALLBACK_TO_WEBSOCKET` warnings are expected in macOS Tauri WKWebView (no WebTransport API, UPnP timeout).
   - However, when `LocalBridgeClient` tore down or the local loopback WebSocket closed after WebRTC `DataChannel` opened, `ws.onerror` fired (`Socket is not connected`) and logged `console.error` even during intentional `close()` or post-connect teardown.

4. **Headless Linux E2E (`runDevRoomSyncE2E.mjs`) Stage 5 Occlusion:**
   - Stage 5 drew a Telestrator stroke at `(640, 440)`, which was directly occluded by the open `DevRoomSyncFloatingWindow` (`blocked-button-or-ui`).

---

## 2. Changes Implemented

- **Independent `isDevSyncRoom` Lifecycle:**
  - Added `isDevSyncRoom` across `appStoreTypes`, `navigationAndCarSlice`, `runtimeOnlineBridgeHandlers`, `runtimeDomainPortTypes`, `onlineLobbyTransitionPipeline`, `OnlineMatchControlBridge`, `MatchRosterPolicyState`, `AuthoritativeServerWorker`, `HostServerFallbackSubController`, and `OnlineMatchReconcilerBinder`.
  - Host creating a Developer Sync Room (`Layer3DevUnderDevelopmentRecipe`) stays in `'freeplay'` (no forced `'online-warmup'` or `'multiplayer_training'`), and `MatchRosterPolicyState.buildHandshakeJson` propagates `isDevSyncRoom` to joining Clients so Clients enter Pure CSR mirror mode without `lobbyDualWorldMode` or diagnostic probes.
  - `Layer1MainMenuRecipe` excludes `isDevSyncRoom` from `hasActiveOnlineRoom` / `isOnlineMatchRoom`, preserving normal `Play` / `Freeplay` / mode menus.
  - `MenuContainer` and `menuRouteNavigationPipeline` (`leaveOnlineRoomAndCloseMenu`) guard against `isDevSyncRoom`: normal `Leave Room` / `Return to Freeplay` actions only return to Free Play and switch room activity to `'freeplay'` without closing the P2P room.
  - Closing/leaving a Developer Sync Room is strictly done via `Stop Room` (Host) or `Leave Sync Room` (Client) inside `DevRoomSyncFloatingWindow`.

- **Continuous Host Workspace Auto-Sync:**
  - Added `matchMode` to `GlobalWorkspaceBroadcastPayload` (`GlobalWorkspaceProtocol.ts`) and synced it in `runtimeSpectateWorkspaceBridge.ts`.
  - Added `startDevSyncWorkspaceAutoBroadcast` / `stopDevSyncWorkspaceAutoBroadcast` in `runtimeSpectateWorkspaceBridge.ts` (started on Dev Sync Room host launch, stopped on `onLeaveOnlineServer`), watching `useAppStore` (`isOpen`, `activeRoute`, `matchMode`, `stadiumPreset`) and `floatingStore` (`windows` open/minimized state while ignoring window `(x, y)` coordinates) to auto-broadcast `GlobalWorkspaceBroadcast` within 16ms.

- **Quiet `LocalBridgeClient` WebSocket Teardown:**
  - Detached `ws` event handlers in `LocalBridgeClient.close()` before calling `ws.close()`, suppressed `ws.onerror` when `this.closed` is true, and downgraded post-connect `ws.onerror` to `console.warn`.

- **Headless E2E Fix:**
  - Updated Stage 5 in `scripts/runDevRoomSyncE2E.mjs` to draw at unobstructed canvas coordinates (`x: 960..1080, y: 440..480`).

---

## 3. Verification

- `pnpm run check` (`radar` + `tsc --noEmit`): **PASS** (1415/1415 TS files, 0 errors, 0 boundary violations).
- `pnpm run test:multiplayer-suite`: **PASS** (14/14 test suites passed).
- `pnpm run test:dev-room-sync-e2e`: **PASS** (all 9 E2E stages passed in headless Chrome, `runtime_error=0`, `poseAnomalies=0`, `stuckInGoal=false`).
