# Live Chat Verification, Investigation & P2P Fixes Report

## 1. Live Chat Worker (`agentlivechatchannel`) Verification
- Inspected `worker.mjs` via Cloudflare API: `defaultWaitMs=300000`, `maxWaitMs=300000`, `pollIntervalMs=10000` with MQTT `PINGREQ` keep-alive ticks (negligible CPU usage, safe from the 10ms CPU limit).
- Root cause of the initial `WAIT_TIMEOUT`: our first test explicitly passed `waitMs: 10000` (10s), whereas the user approved at ~12s.
- Using long-poll (`waitMs: 270000`) on channel `ch_1712e873`, bidirectional Live Chat and remote command execution (`get_status`, `scan_window`, `perform_action`, `execute_javascript`) worked end-to-end.

## 2. Live Findings on Windows `47694` & `51371`
- **Carousel / Dynamic Path Stability**: On `47694`, `"Next apps"` shifted from `[3,0,15]` to `[3,0,13]` between scans, and `perform_action` with `targetRegex` hit `[3,0,13]` accurately.
- **`targetRegex` Anchored Matching**: `handlePerformActionCommand` previously tested `targetRegex` only against `${el.role} ${el.title} ...`, causing `^Next apps$` to fail. Fixed to test individual fields (`title`, `description`, `value`, `identifier`) as well as the combined string.
- **Network `"1 rule"` / `partial` State**: Live JS & click inspection on `51371` confirmed that `"1 rule"` is only `"Gemini API (default)"` and the modal contents are lazy-loaded on click. Added `hasCustomNetworkRules` (`>= 2 rules`) in `aiStudioWindowState.ts`.
- **Duplicate Window Chips / `[Recent]`**: `buildVisibleWindows` did not deduplicate by `windowId`. Added `seenIds` deduplication (preferring active windows over missing chips).
- **EMQX Channel Bar UI**: Split `EmqxBridgeChannelBar.tsx` into a fixed top row (`Channel: All | default | Add channel... Switch`) and a second horizontal-scrolling row for `#ch_xxx` channels with hidden scrollbar (`.emqx-channel-scroll::-webkit-scrollbar` + `scrollbarWidth: "none"`).

## 3. Verification & Commit
- `pnpm radar -- --audit`: PASS (`291 files`, `HardVolume=0/0`, `LayerViolations=0`, `Cycles=0`)
- `pnpm p2p:check`: PASS (`tsc --noEmit && pnpm lint`)
- Verified Bot Commit: [`a89484faecf519ae975f76b68358fe73fcb3df63`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/a89484faecf519ae975f76b68358fe73fcb3df63) (`signature.isValid: true`, `state: "VALID"`) on branch `a2`.
