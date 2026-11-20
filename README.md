# Delivery Report: Post-Init `partial` Fix, Mobile/iPad `LLM Control` Tab & Live Chat Worker Diagnosis

- **Repository**: `Antigravi-ty/agyPreviewInstant` (branch `a2`)
- **Verified Bot Commit**: [`31592cd9104813209123a3ea6ddc35bc81934ab2`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/31592cd9104813209123a3ea6ddc35bc81934ab2) (`signature.isValid = true`, `state = VALID`)
- **Live Channel**: `ch_4a336536` (inspected Safari Window `47694` — `Google AI Studio`, `https://aistudio.google.com/prompts/new_chat?model=gemini-4-hydrogen`)

---

## 1. Fix: Post-Initialization `partial` Readiness Bug

### Root Cause
1. In `apps/p2p-remote-controller/src/services/aiStudioWindowState.ts`, `classifyAiStudioWindowState` computed `hasInteractionId` by inheriting `previous?.hasInteractionId` whenever `!urlChanged`, even when `networkVerified: true` was passed after initialization. On the immediate follow-up scan (`handleScanWindow`), `invalidatedByTaskOrNavigation` saw `hasInteractionId === true` and immediately reset `networkConditionAdjusted` back to `false` (`partial`).
2. In `apps/p2p-remote-controller/src/hooks/useSafariBatchOperations.ts`, `updateWindowReadinessFromScan` did not update `readinessRef.current` synchronously inside the state updater.
3. In `apps/p2p-remote-controller/src/services/aiStudioBatchRoundRobin.ts`, if the post-init `freshScan` failed/returned `null`, `onWindowScanned` was skipped, leaving the window in `partial`.

### Changes & State Machine Verification
- `aiStudioWindowState.ts`: When `networkVerified` is `true`, do not carry over `previous?.hasInteractionId`. Subsequent scans stay `ready` (`networkConditionAdjusted: true`) until a task actually starts running (`busy` / `hasActiveTask`), an interaction URL (`/prompts/<id>`) appears, URL changes, or the page refreshes (`!webAreaLoaded`).
- `useSafariBatchOperations.ts`: Update `readinessRef.current` synchronously inside `setWindowReadiness`.
- `aiStudioBatchRoundRobin.ts`: Fall back to `scan` when `freshScan` is `null` so `networkVerified: true` is always emitted.
- `aiStudioWindowState.test.ts`: Added regression test verifying re-initialization clears stale `hasInteractionId` and preserves `ready` on follow-up scans.

---

## 2. Mobile & iPad Viewport: 3rd Tab `LLM Control` (Default)

### Changes
- **`apps/p2p-remote-controller/src/components/PocketDeckView.tsx`**:
  - Added 3rd tab `"llm"` (**LLM Control**) and set it as the default `DeckMode`.
  - Updated container width (`width: "100%"`, `maxWidth: 900`) so it adapts cleanly to both mobile phones and iPad/tablet viewports.
- **`apps/p2p-remote-controller/src/components/PocketDeckLlmControlTab.tsx`**:
  - **Responsive Grid**: `repeat(auto-fit, minmax(260px, 1fr))` stacks on mobile and displays 2-column cards on iPad.
  - **No-API-Key Guardrail**: Inspects AX elements (`Link API Key`, `No API key`, `API key selected`) and displays `No API Key (LLM Safe)` vs `API Key Linked` so LLM mode can be verified before running.
  - **Model Selection (Click with Regex)**: 4 quick presets (`Gemini 2.5 Pro`, `Gemini 2.5 Flash`, `Gemma 3 27B`, `Gemini 4 Hydrogen`), `Open Picker`, and custom AX regex click input—resilient to carousel/DOM path changes.
  - **Output Mode Toggle**: `Stream Output (P2P)` (cellular direct live updates) vs `Final Output (EMQX)` (lightweight 1Hz thinking/byte-count relay).
  - **Focus-Safe Prompt Input & Quick Controls**: `Run`, `Stop`, `System` regex triggers and prompt injection (`AXSetValue` + `AXPress`) with `restoreFocus` toggle.
- **`apps/p2p-remote-controller/src/types/lanSharing.ts` & `apps/p2p-remote-controller/src/hooks/usePocketDeckBridge.ts`**:
  - Extended `PocketDeckRemoteAction` (`perform_action`) with `value?: string` and `restoreFocus?: boolean`, wrapping focus + AX input/click in `withCallerFocusRestored` so the previously focused macOS window is automatically restored.

### Framework Recommendation
- **Keep `apps/p2p-remote-controller` on its current React + TypeScript + Vite + CSS-variable stack** (which already shares the same React/TS/Lucide/Tauri v2 architecture as the main AGY app) rather than introducing a separate Primary React Latest runtime inside the companion controller. This avoids bundle bloat and keeps `PocketDeckView` lightweight for mobile/iPad LAN & cellular loading.

---

## 3. Verification Results
- `pnpm --filter p2p-remote-controller check` (`tsc --noEmit && pnpm lint`): **Passed**
- `pnpm --filter p2p-remote-controller exec vitest run`: **34/34 tests passed** (9 test files)
- `pnpm --filter p2p-remote-controller build`: **Passed** (`dist/assets/index-B4FmGteH.js` 395.51 kB)

---

## 4. `agentlivechatchannel` Worker & `/help` Improvement Recommendations

Observed live during this session on `https://agentlivechatchannel.maboco-mark.workers.dev/`:

1. **`POST /cmd` strips command-specific parameters**:
   - Sending `regexFilter`, `level`, `maxChars`, `script`, `action`, `targetRegex`, or `nodePath` in `/cmd` dropped those fields before publishing to `agy_p2p_v1/cmd/<channel>/<command>` (e.g., `scan_window` always fell back to `compact` 25 elements; `execute_javascript` failed with `script is required`).
   - **Fix**: Forward all whitelisted command parameters (`level`, `regexFilter`, `maxChars`, `script`, `action`, `targetRegex`, `targetIndex`, `nodePath`, `value`, `focusFirst`, `doubleCheck`, `timeoutMs`, etc.) or pass through an `args` object into the MQTT command payload.
2. **`POST /chat/poll` returns the agent's own sent messages**:
   - `chat/poll` subscribed to `agy_p2p_v1/chat/<channel>` and immediately returned messages with `sender: "agent"` that were still queued/retained, causing the agent to miss or misread user replies and re-poll prematurely.
   - **Fix**: Support `excludeSender` (default `"agent"` when polling as agent) and `since` / message ID deduplication on `/chat/poll`, and avoid returning already-acked messages on subsequent polls.
3. **`POST /mqtt/rpc` schema mismatch in `/help`**:
   - `/help` mentions `POST /mqtt/rpc` as "Low-level MQTT publish/subscribe RPC" without documenting its required JSON body fields (`pubTopic`/`subTopic` did not match, causing it to fall back to `chat` subscription).
   - **Fix**: Document the exact request/response schema for `/mqtt/rpc` in `/help` (or deprecate it once `/cmd` forwards all parameters).
4. **Fixed `waitMs: 270000` exceeds agent sandbox HTTP timeouts (~60s)**:
   - Long-polling for 4m30s causes client/sandbox tool invocations to hit `deadline_exceeded` unless the caller manually passes `curl --max-time`.
   - **Fix**: Allow callers to pass `waitMs` within a safe range (e.g. `5000..60000`, default `30000`) on `/chat/poll` and `/cmd`, and document `--max-time` in `/help`.
