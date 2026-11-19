# Session Summary (agyPreviewInstant / branch `a2`)

## Completed & Committed (`135f3bb`)

Verified bot commit pushed to `Antigravi-ty/agyPreviewInstant` (`a2`):
- **Commit**: `135f3bb2a2a025e126dd3a73c47eb66c6a9d9de3` (`signature.isValid: true, state: "VALID"`)
- **Message**: `fix(p2p-remote-controller): block play.google.com trackers, extract v1_ IDs, and auto-poll Network tab`

### 1. Block `play.google.com` Tracker Requests Before Sending
- **File**: `apps/p2p-remote-controller/src/services/networkRecorderScript.ts`
- Added `isBlockedUrl(url)` matching `^https?:\/\/([^/]+\.)?play\.google\.com(\/|$|\?)`.
- In `window.fetch`: immediately rejects with `Promise.reject(new TypeError(...))` without calling `origFetch` or recording into `entries`.
- In `XMLHttpRequest.prototype.send`: immediately aborts without calling `origSend`, dispatching `error` and `loadend` events asynchronously.
- In `navigator.sendBeacon`: returns `false` without sending.

### 2. Direct `v1_` Interaction ID Extraction
- **File**: `apps/p2p-remote-controller/src/services/networkRecorderScript.ts`
- Updated `scanForInteractions` to extract `\b(v1_[a-zA-Z0-9_-]{20,})\b` directly (capturing IDs from `CreateInteraction` responses and `GetInteractionStream` requests) and normalized `CreatePrompt` prompt IDs (`prompts/...`).
- `clients6` XHR responses (such as `CreateInteraction` 429 `"Project environment storage quota exceeded."` and 200 responses) are captured via `xhr.responseText` in `syncXhrBody`.

### 3. Auto-Polling on Network Tab
- **File**: `apps/p2p-remote-controller/src/App.tsx`
- Added `silent` mode + `netPollInFlightRef` guard to `handlePollNetRecorder` so background polling does not toggle `busy` or spam timeline events.
- Added a `useEffect` that silently polls `handlePollNetRecorder(false, true)` every `2000ms` while `rightTab === "network"`, `netRecording === true`, `selectedWindowId !== null`, and Safari is running.

## Verification
- `pnpm exec tsc --noEmit` in `apps/p2p-remote-controller`: passed (exit code 0).
- `pnpm test` in `apps/p2p-remote-controller`: 2 test files, 11 tests passed.
- Verified `buildNetworkRecorderInstallScript` and `buildNetworkRecorderPollScript` output parses cleanly with `new Function(...)`.

## Pending (Waiting for Interactive Session / Fresh Page)
- **`aiStudioInit.ts` (`Link API key` & `Failed to verify Domain field value is '*'`)**:
  - `Link API key` currently only detects the button without clicking/selecting a key in the modal.
  - `Domain` field uses `AXRaise` + `set_value` without `focus` or DOM `input`/`change` event fallback.
  - Sent questions on EMQX (`agy_p2p_v1/chat`) and waited 3 consecutive ~45s rounds with no reply, so these two items are left untouched for interactive debugging on a fresh AI Studio page.
