# Live Chat Worker Assessment & Codebase Readiness Report

## 1. Live Chat Worker (`/help`) Assessment
- **Status**: Ready to use (`agentlivechatchannel`, `plan: live-chat-v1`, `authEnabled: true`, `builtInConfig: true`).
- **Assigned Channel**: `ch_1712e873`
  - `cmd`: `agy_p2p_v1/cmd/ch_1712e873/<command>`
  - `resp`: `agy_p2p_v1/resp/ch_1712e873/<requestId>`
  - `chat`: `agy_p2p_v1/chat/ch_1712e873`
- **Live Verification**:
  - `POST /chat/send` succeeded (`connackCode: 0`, `pubackedCount: 1`).
  - `POST /cmd` (`get_status`, `stream: true`) connected, subscribed (`subacked: true`), published, and streamed NDJSON heartbeats cleanly before timing out at 10s (waiting for remote approval).
  - `p2p-remote-controller` (`useEmqxConnection.ts`) already subscribes to `${prefix}/cmd/#` and `${prefix}/chat/#`, so it automatically receives messages on `ch_1712e873`.
- **Minor `/help` Documentation Notes (No Worker Change Required)**:
  - `/chat/poll` does not explicitly document its max `waitMs` (unlike `/cmd`, which specifies `default 300000, max 300000`).
  - `/cmd` does not list command-specific optional fields (`nodePath`, `targetRegex`, `targetIndex`, `action`, `value`, `prompt`, `script`, etc.), though pass-through JSON forwarding works.
  - `GET /help` generates a fresh random `channel` on every call; the agent should call it once and reuse the assigned channel ID.

## 2. Repository & Architecture Baseline (`agyPreviewInstant`)
- `pnpm radar -- --audit` passed (`291 files`, `HardVolume=0/0`, `LayerViolations=0`, `Cycles=0`, `P2P Type/Lint Check: PASS`).
- **Already Present**:
  - `apps/p2p-remote-controller/src/utils/p2pSecurity.ts`: AES-GCM E2EE envelope (`E2eeEnvelope`), UUID Access Token, Base32 TOTP secret + `otpauth://` URI + 6-digit TOTP generation/verification.
  - `apps/p2p-remote-controller/src/components/PocketDeckView.tsx` & `usePocketDeckRemoteSync.ts`: Pocket Deck UI with `bridge` and `direct` tabs, plus recent SW caching commit (`ac11e4f`).
  - `apps/p2p-remote-controller/src/services/aiStudioWindowState.ts`: `classifyAiStudioWindowState` marks windows `"partial"` when `apiKeyLinked` is true but `modelSelected` or `networkConfigured` (`networkVerified || previous?.networkConfigured`) is false.
- **Next Steps for Live Chat Collaboration**:
  1. Connect via Live Chat on channel `ch_1712e873` to inspect live Safari windows, test dynamic carousel/ID-shifting pages with regex/name-based targeting (`targetRegex` / `targetIndex`), and debug the `initialize_aistudio` `partial` network flag.
  2. Implement mobile-marked windows (`isMobile`), dual-mode LAN -> P2P/E2EE-EMQX fallback, TOTP gating on connection & sensitive actions, and reload/refresh mode enhancements while keeping `pnpm radar -- --audit` green.
