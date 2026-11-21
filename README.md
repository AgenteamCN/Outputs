# Task Completion Report — `Antigravi-ty/agyPreviewInstant` (`a2`)

## Summary of Commits on `origin/a2`
1. **`97f4eed`** — `fix(p2p): network allowlist rule detection, readiness check-valve, retry state, and PocketDeck input UI`
2. **`23eab9c`** — `feat(p2p): rename dispatch_aistudio_task to dispatch_agent_task and add dispatch_llm_task`
3. **`cf668e3`** — `feat(p2p,tray,safari): passive P2P sync, 2Hz/1Hz stream push, tray shortcuts, and Safari modes`

## Changes in `cf668e3`
### 1. Passive P2P Mode & On-Demand Fetching
- Removed the 4000ms `setInterval` polling loop in `apps/p2p-remote-controller/src/components/PocketDeckView.tsx`.
- Host strips heavy `consoleLogs`, `timeline`, and `elements` from `pocket_deck_get_snapshot` and `pocket_deck_action` responses in `apps/p2p-remote-controller/src/hooks/useCellularP2pHost.ts`, and serves them on demand via `pocket_deck_get_console`, `pocket_deck_get_timeline`, `pocket_deck_get_elements`, and `pocket_deck_get_llm_output`.
- Added manual `Fetch` button, `P2P`/`LAN`/`Cached` connection badge, `SF Mono` commit/layout typography, and `☰` menu dropdown (`Layout Mode` + `Update`) in `apps/p2p-remote-controller/src/components/PocketDeckHeader.tsx`.
- Added `Mobile` / `All` window filter toggle (defaulting to mobile-opened windows) and `SF Mono` styling for model names, settings, and URLs in `apps/p2p-remote-controller/src/components/PocketDeckDirectTab.tsx` and `PocketDeckLlmControlTab.tsx`.

### 2. PC-to-Phone Push & Transport-Aware 2Hz/1Hz Streaming
- Added `CellularPushPayload` (`bridge_update`, `llm_stream_progress`) and `e2ee_push` signaling in `apps/p2p-remote-controller/src/types/cellularP2p.ts`, `useCellularP2pHost.ts`, and `useCellularP2pClient.ts`.
- Added `extractGenerateContentStreamProgress()` in `apps/p2p-remote-controller/src/utils/generateContentStream.ts` (with unit tests in `generateContentStream.test.ts`) to separate `GenerateContent` thinking chunks (trailing `, 1]`) from answer chunks.
- Added `useCellularHostPushSync` (`apps/p2p-remote-controller/src/hooks/useCellularHostPushSync.ts`) wired in `App.tsx`:
  - Pushes Bridge approval/chat updates to connected cellular clients.
  - Streams `GenerateContent` progress at **2 Hz** with `thinkingDelta` / `outputDelta` over `p2p_direct` (auto-collapsing thinking once output starts in `PocketDeckLlmControlTab.tsx`), and at **1 Hz** with `{ thinkingBytes, outputBytes, done }` only over `emqx_cn_relay` (on-demand `Fetch Output` button).

### 3. Tray Menu Shortcuts & Safari Window Modes
- Updated `apps/p2p-remote-controller/src-tauri/src/lib.rs`: enabled `.show_menu_on_left_click(true)` and added `Shortcuts` submenu with `EMQX CN (Private)` and `EMQX Global (Private)`.
- Updated `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift`, `JsonRpcServer.swift`, `apps/p2p-remote-controller/src-tauri/src/commands.rs`, and `apps/p2p-remote-controller/src/services/axService.ts` to support `mode` (`"normal"` | `"popup"` | `"private"`) and optional `width` / `height` bounds.

## Verification
- `pnpm p2p:check` (`tsc --noEmit && eslint src`): **Passed** (0 errors, 0 warnings).
- `pnpm --filter p2p-remote-controller test` (`vitest run`): **14 test files, 51 tests passed**.
- Note: `src-tauri` (Rust) and `MacosAxHelper` (Swift) are not compiled in the Linux sandbox and should be built on macOS with `pnpm p2p:build:app`.
