# Diagnostic Report: High Resource & Energy Consumption in AGY Preview Instant

**Repository:** `https://github.com/Antigravi-ty/agyPreviewInstant` (branch `a2`, commit `67abc87`)  
**Observed Symptoms (macOS Activity Monitor on Apple Silicon):**
- `AGY Preview Instant Graphics and Media`: ~20% CPU, ~10% GPU, Energy Impact ~220–240
- `127.0.0.1:1420`: Energy Impact ~100

---

## Executive Summary & Process Mapping

On macOS, Tauri renders the frontend inside `WKWebView`. In dev mode (`src-tauri/tauri.conf.json:8`, `"devUrl": "http://127.0.0.1:1420"`), macOS splits `WKWebView` into two helper processes that appear directly in Activity Monitor:
1. **`AGY Preview Instant Graphics and Media`** (`com.apple.WebKit.GPU`): WebKit's GPU, CoreAnimation, and compositing process. Its ~20% CPU / ~10% GPU / 220–240 Energy Impact is caused by **continuous, non-composited CSS animations (`box-shadow` keyframes) and infinite SVG rotation loops** forcing 60/120 Hz ProMotion repaints and layer commits even when the app is idle.
2. **`127.0.0.1:1420`** (`com.apple.WebKit.WebContent`): WebKit's web content process for origin `http://127.0.0.1:1420`. Its ~100 Energy Impact is caused by **multiple unaligned 1-second `setInterval` timers, 3.5s/5s polling loops, and bare (unselected) Zustand store subscriptions** in `App.tsx` and child components that trigger full-tree React re-renders and prevent the WebContent thread from entering idle sleep.

Backend Rust code (`src-tauri/src/`) and `macos-ax-helper` (`packages/macos-ax-helper/`) are predominantly event-driven (`tokio::select!`, `CFRunLoop` + `AXObserver`) and are **not** the primary source of idle CPU/GPU drain.

---

## Prioritized Root Causes & Concrete Fixes

### P0 — Critical (Drives `AGY Preview Instant Graphics and Media`: ~20% CPU, ~10% GPU, Energy Impact ~220–240)

#### 1. Infinite Non-Composited `box-shadow` Keyframe Animation (`primer-refresh-reminder-pulse`)
- **Files & Lines:**
  - `src/styles/controls.css:112-127`
  - `src/components/layout/TaskGroupCard.tsx:455-457`
  - `src/utils/taskGrouping.ts:383-400` (`taskNeedsFinalOutputRefresh`)
- **Root Cause:**
  ```css
  @keyframes primer-refresh-reminder-pulse {
    0%, 100% { box-shadow: 0 0 0 0 rgba(26, 127, 55, 0.45); }
    50%      { box-shadow: 0 0 0 3px rgba(26, 127, 55, 0.22); }
  }
  .primer-btn-refresh-reminder {
    ...
    animation: primer-refresh-reminder-pulse 1.8s ease-in-out infinite;
  }
  ```
  `TaskGroupCard` applies `.primer-btn-refresh-reminder` whenever `needsFinalOutputRefresh` is true (which stays true indefinitely on any unread task card without final output). Because `box-shadow` is a **non-composited, paint-triggering property**, WebKit cannot offload it to the GPU compositor. Instead, it must re-rasterize the layer on the CPU, upload new textures, and composite on **every display refresh frame (60 Hz or 120 Hz ProMotion on Apple Silicon)**.
- **Fix:**
  1. Replace the `box-shadow` keyframe with a GPU-composited `opacity` (and/or `transform: scale(...)`) animation on a `::after` pseudo-element with a static border/ring, **or** use a static highlight badge without an infinite animation.
  2. Cap `animation-iteration-count` (e.g. `3`) or pause animations when `document.hidden` / `@media (prefers-reduced-motion: reduce)`.

#### 2. Infinite `spin-icon` (`transform: rotate(360deg)`) on Persistent `waiting` / `pending_` States
- **Files & Lines:**
  - `src/styles/layout.css:402-413` (`@keyframes primerSpin`, `.spin-icon`)
  - `src/components/layout/TaskGroupCard.tsx:474-483`
  - `src/components/inspector/TaskOverviewSidebar.tsx:296-299`
  - `src/components/inspector/TaskOutputViewer.tsx:183, 238`
  - `src/components/inspector/DiagnosticErrorPanel.tsx:59, 129`
  - `src/components/environments/EnvironmentManagerModal.tsx:313`
  - `src/components/settings/sections/StorageManagementSection.tsx:274`
  - `src/components/inspector/TaskInspectorTabBar.tsx:189-193` (references undefined `"spin 1s linear infinite"` instead of `primerSpin`)
- **Root Cause:**
  `TaskGroupCard.tsx:477-480` applies `.spin-icon` (`animation: primerSpin 1s linear infinite`) not only during active `isSyncing`, but also whenever `isPendingDispatch || task.status === "waiting"`. Similarly, `TaskOverviewSidebar.tsx:298` spins on `isSyncing || isWaiting`. Any task sitting in `waiting` or `pending_` keeps a continuous 60/120 fps CoreAnimation commit loop active in `AGY Preview Instant Graphics and Media`, preventing WebKit from reaching zero-frame-rate idle.
- **Fix:**
  1. Only apply `.spin-icon` during active, short-lived network requests (`isSyncing`), and render a static icon (e.g., `Clock` or `Hourglass`) for `waiting` / `isPendingDispatch` states.
  2. Add a global visibility rule (e.g., `body.app-hidden .spin-icon { animation-play-state: paused; }`) toggled via `visibilitychange` / window blur.
  3. Fix `TaskInspectorTabBar.tsx:190` to use `className={isRefreshingCurrent ? "spin-icon" : ""}` instead of inline `animation: "spin 1s linear infinite"`.

---

### P1 — High (Drives `127.0.0.1:1420` WebContent Energy Impact ~100 & Cascading CPU Wakeups)

#### 3. Bare (Unselected) Zustand Store Subscriptions Causing Full-Tree Re-Renders
- **Files & Lines:**
  - `src/App.tsx:21-26` (subscribes to `useCredentialsStore()`, `useEnvironmentStore()`, `useLanStore()`, `useTaskStore()`, `useTelemetryStore()`, `useThemeStore()` with **no selectors**)
  - `src/components/layout/TaskHistorySidebar.tsx:44-83` (bare `useTaskStore()`, `useTelemetryStore()`, `useLanStore()`, `usePresetStore()`, `useThemeStore()`)
  - `src/components/layout/AppTopHeader.tsx:51-101`
  - `src/components/inspector/TaskInspectorView.tsx:37-45`
  - `src/components/inspector/TaskOverviewSidebar.tsx:41-58`
  - `src/components/inspector/TaskTelemetryPanel.tsx:40-44`
  - `src/stores/task/taskHydrationSlice.ts:64-65` (`setSidebarScrollTop`)
- **Root Cause:**
  Calling `useXxxStore()` without a selector subscribes the component to **every field** of that store. Whenever any telemetry heartbeat updates `activeEnvironments`, `heartbeatExpirationTimer` mutates state, or `TaskHistorySidebar` scrolls (`onScroll -> setSidebarScrollTop` at `TaskHistorySidebar.tsx:537-539`), `App.tsx` and all major layout/inspector components re-render, re-running `groupTasksByEnvironment(tasks, ...)` (`TaskHistorySidebar.tsx:99`) and diffing the entire task list.
- **Fix:**
  1. Use granular Zustand selectors (or `useShallow` from `zustand/react/shallow`) in `App.tsx` and all layout/inspector components so components only subscribe to the exact slices they read.
  2. Move `sidebarScrollTop` out of `useTaskStore` into a module-level variable or `useRef` (matching `inspectorScrollMemory` in `TaskInspectorView.tsx`).

#### 4. Proliferation of Unaligned 1-Second `setInterval` Timers (Per-Card `RelativeTime` + Inspector/Header Timers)
- **Files & Lines:**
  - `src/components/common/RelativeTime.tsx:11-44` (`setInterval(..., 1_000)` per mounted component when `ageMs < 60_000`, `10_000` when `< 1h`)
    - Mounted twice per `TaskGroupCard` (`src/components/layout/TaskGroupCard.tsx:371, 377`), plus `TaskTelemetryPanel.tsx:314`, `TaskOverviewMetadataFields.tsx:84, 120`, `TaskSearchResultTable.tsx:180`, `DraftDataTableRow.tsx:405`
  - `src/components/inspector/TaskInspectorView.tsx:84-86` (`setLiveNowMs` every `1000ms` when `in_progress`/`running`)
  - `src/components/inspector/TaskOverviewSidebar.tsx:80-82` (`setLiveNowMs` every `1000ms` when `in_progress`/`running`)
  - `src/components/inspector/TaskTelemetryPanel.tsx:78-80` (`setTickNowMs` every `1000ms` when `activeState.status === "running"`, triggering `resolveRoundTelemetrySnapshot` every second)
  - `src/components/layout/AppTopHeader.tsx:114-116` (`setNowMs` every `5_000ms` unconditionally, recomputing `calculateRequestQuotaSnapshot` and re-rendering the top header)
- **Root Cause:**
  Dozens of independent `setInterval` timers fire at unaligned intervals, causing continuous main-thread wakeups/sec in `127.0.0.1:1420` (WKWebView WebContent), preventing macOS timer coalescing and App Nap.
- **Fix:**
  1. Replace per-component `setInterval` in `RelativeTime.tsx` with a **single shared clock store/observable** (e.g., 1s tick only if any subscriber is `< 60s` old and `document.visibilityState === "visible"`, otherwise 15s–60s).
  2. Consolidate `TaskInspectorView`, `TaskOverviewSidebar`, and `TaskTelemetryPanel` live duration tickers into one shared `useLiveClock(enabled)` hook that pauses when `document.hidden`.
  3. In `AppTopHeader.tsx`, update `nowMs` on a 60s interval (or only on task/telemetry events) instead of every 5s.

#### 5. Continuous Background Polling & Uncleaned `heartbeatExpirationTimer`
- **Files & Lines:**
  - `src/services/bemfaHeartbeatClient.ts:78-83` (HTTP `GET https://apis.bemfa.com/...` every `3_500ms` while connected, invoking `onPollTick` -> `expireTimedOutEnvironments` on every tick)
  - `src/stores/telemetryStore.ts:159-176` (`heartbeatExpirationTimer` `setInterval(..., 5000)` — started on connect/load and never cleared)
  - `src/components/inspector/TaskInspectorView.tsx:127-129` (`refreshTask` every `5_000ms` when `autoPollEnabled` and task is `in_progress`/`running`)
- **Root Cause:**
  Even when no tasks are running, `BemfaHeartbeatClient` performs an uncached (`cache: "no-store"`) HTTPS fetch every 3.5s, and `heartbeatExpirationTimer` wakes up every 5s indefinitely.
- **Fix:**
  1. Exponentially back off `BemfaHeartbeatClient` polling (e.g., 3.5s when an environment is actively `running`, 15s–30s when all environments are idle, and pause/slow when `document.hidden`).
  2. Only run `heartbeatExpirationTimer` when `Object.values(activeEnvironments).some(e => e.status === "running")`, and clear it when none are running.

---

### P2 — Medium (Amplifies Layout/Paint & Dev-Mode Overhead)

#### 6. Non-Virtualized Task List & Broad CSS `transition: all`
- **Files & Lines:**
  - `src/components/layout/TaskHistorySidebar.tsx:562` (renders all `unreadGroups` and `readGroups` directly without list virtualization)
  - `src/styles/controls.css:346, 544, 584` (`transition: all 0.12s ease`)
- **Fix:**
  - Virtualize `TaskHistorySidebar` (e.g., `@tanstack/react-virtual`) or paginate/window older groups when task count grows.
  - Replace `transition: all` with explicit properties (`transition: background-color 0.12s ease, border-color 0.12s ease`).

#### 7. Vite Dev Server Watch Scope & `devWatch.ts` 500ms Fallback Poll (Dev Mode Only)
- **Files & Lines:**
  - `vite.config.ts:11-13`: `watch.ignored` only ignores `["**/src-tauri/**"]`, leaving `apps/p2p-remote-controller/src-tauri/target/**`, `packages/**`, and `docs/**` watched by Vite's chokidar instance on `127.0.0.1:1420`.
  - `scripts/devWatch.ts:14, 266`: `FALLBACK_POLL_MS = 500` runs `checkCheckoutSignals` via `setInterval` every 500ms when `agy:dev:watch` is used.
- **Fix:**
  - Expand `vite.config.ts` `server.watch.ignored` to `["**/src-tauri/**", "**/target/**", "**/apps/**", "**/docs/**"]`.
  - Increase `FALLBACK_POLL_MS` in `scripts/devWatch.ts` from `500` to `3000–5000` (since `fs.watch` already handles immediate checkout events).

#### 8. Uncapped `requestAnimationFrame` in QR Scanner Modal (Conditional)
- **Files & Lines:**
  - `src/components/layout/LanQrScannerModal.tsx:123-147`
- **Root Cause:**
  When the camera QR scanner is active, `scanFrame` runs `ctx.drawImage` + `ctx.getImageData` + `decodeQrFromImageData` on every `requestAnimationFrame` (60–120 fps).
- **Fix:**
  Throttle QR frame decoding inside `scanFrame` to ~10 fps (`now - lastScanAt >= 100`).

---

## Summary Checklist (Expected Impact)
| Priority | Target | Fix | Expected Impact |
|---|---|---|---|
| **P0** | `AGY Preview Instant Graphics and Media` | Remove `box-shadow` animation in `controls.css:112` & stop spinning `waiting` icons in `TaskGroupCard.tsx:477` | Eliminates continuous 60/120 Hz CPU repaints & GPU compositing; drops GPU from ~10% to ~0% and Energy Impact from ~230 to <10 at idle |
| **P1** | `127.0.0.1:1420` | Add Zustand selectors (`App.tsx`, sidebars), consolidate `RelativeTime` / live clocks, pause timers on `document.hidden`, gate `heartbeatExpirationTimer` | Eliminates per-second wakeups & full-tree re-renders; drops WebContent Energy Impact from ~100 to <10 at idle |
| **P2** | Both | Virtualize `TaskHistorySidebar`, replace `transition: all`, narrow `vite.config.ts` watch ignores | Reduces memory/layout overhead on large histories and dev-server FSEvents churn |
