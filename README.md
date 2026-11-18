# P2P Remote Controller: EMQX Bridge, Log Analysis & Safari SSE Interception Design

## 1. Log Analysis (`2026-11-18T10:27:34Z` & `10:35:14Z`)
- **Window lifecycle**: `open_managed_window` opened Safari window `#22817` (`https://aistudio.google.com/`) at `10:26:25`, `AXLoadComplete` fired at `10:26:31.950` (`elements=26`).
- **Manual scan & action**: At `10:26:48`, manual `scan_ax_tree` discovered `elements=49`, including `Link API Key` (`w22817:3.0.7`). Clicking `perform_press` at `10:26:58` triggered `post_action_rescan` (`elements=4`, too early for SPA modal rendering), followed by `AXLiveRegionChanged` (`elements=13`) and `AXUIElementDestroyed` (`elements=11`).
- **Takeaway**: Compact timeline logs already capture AX tree transitions and user actions cleanly. Adding a `reload_window` command in `macos-ax-helper` + passive network capture will close the loop when AI Studio fails to resolve a project.

## 2. Implemented in `apps/p2p-remote-controller`
Committed as verified bot commit `0674d9b` on branch `a2`:
- `apps/p2p-remote-controller/src/services/emqxClient.ts`: Zero-dependency MQTT 3.1.1 over WebSocket client with random `clientId` support, QoS 1 publish/subscribe, and byte counters.
- `apps/p2p-remote-controller/src/utils/emqxBridgeUtils.ts`:
  - `localStorage` persistence (`p2p_emqx_bridge_config_v1`) so credentials and low-traffic defaults survive commits/reloads without hardcoding secrets in tracked files.
  - `maskSecretPreview`: displays first 3 + `***` + last 3 characters (e.g. `vas***xqa`) to verify credential versions safely.
  - `generateRandomClientId`: allows multiple clients to share username `CNremoteController` without session collisions.
  - `buildPiggybackTimelineDiff`: automatically attaches unseen `timelineEvents` (`recentUserActions`) to every response and chat message.
- `apps/p2p-remote-controller/src/components/EmqxBridgePanel.tsx` & `App.tsx`:
  - **EMQX Bridge** tab + topbar status button (`EMQX: Live/Off (pendingCount)`).
  - **Settings** panel (Broker URL, Username, Password with 3+3 preview, Topic Prefix `agy_p2p_v1`, Client ID Prefix, Default Level `compact`, Max Body Chars `600`, Regex Filter).
  - **Emergency Stop / Disconnect** kill switch.
  - **Approval Queue**: `Approve`, `Reply & Approve`, `Reply Only`, `Deny`, `Deny with Reason`.
  - **Bidirectional Chat** feed.

Verification:
- `pnpm p2p:check` (`tsc --noEmit && eslint src`): passed.
- `pnpm --filter p2p-remote-controller test`: 8/8 tests passed (`emqxBridgeUtils.test.ts`, `logExport.test.ts`).
- `pnpm --filter p2p-remote-controller build`: passed.

## 3. EMQX CN Quota, ACL & Workflow Answers
- **1 GB Quota**: Both subscribe and publish count toward quota, but MQTT overhead is minimal (2-byte header/keepalive). At ~2–10 KB per `compact` round-trip, 1,000 rounds consume ~10 MB (~0.01 GB); 1 GB supports ~100,000 rounds.
- **Recommended ACL for `CNremoteController`**:
  - `agy_p2p_v1/#` -> Publish & Subscribe -> **Allow**
  - `#` -> Publish & Subscribe -> **Deny**
- **5-Minute Tool Timeout & Approval Workflow**:
  - Cloud agent publishes a command to `agy_p2p_v1/cmd/<requestId>` and waits on `agy_p2p_v1/resp/<requestId>` with a bounded timeout (e.g. 240s, safely under the 300s tool limit).
  - Local user approves/replies/denies in `EMQX Bridge`, automatically piggybacking `recentUserActions` since the last sync.

## 4. Safari Network Interception Without Breaking SSE
- Safari does not expose CDP `Network.*`. Use a Safari Web Extension (`document_start`, `world: "MAIN"`) or Apple Events JS injection to wrap `window.fetch` / `XMLHttpRequest`.
- For SSE (`text/event-stream`), tee the response stream (`response.body.tee()` or `response.clone()`):
  - **Branch A** returns immediately to Google AI Studio (never blocked or terminated).
  - **Branch B** passively reads chunks in the background to extract `Interaction ID` and truncated body previews (600/1000 chars). Clicking **Stop** cancels **only Branch B** (`reader.cancel()`), leaving the active SSE stream intact.
