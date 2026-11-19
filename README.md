# P2P Remote Controller & EMQX Chat Diagnosis + Fixes

## 1. Root Cause Analysis

### (1) Why `chat` does not automatically listen for replies (and multi-agent collision)
- **`agentplayground.maboco-mark.workers.dev` (`emqx_worker.mjs`)** is a **stateless, short-lived HTTP-to-MQTT RPC bridge** (`waitMs` capped at `55000ms`). Each `/mqtt/rpc` call opens a TLS socket to EMQX (`8883`), optionally subscribes/publishes, waits up to `waitMs`, and then sends `DISCONNECT` (`0xe0 0x00`) and closes the socket.
- Because AI agents are turn-based, an agent **only receives messages while actively executing an `/mqtt/rpc` poll** (such as when it sends an `ask_user` command to `agy_p2p_v1/cmd/<target>` and blocks on `agy_p2p_v1/resp/<requestId>` + `agy_p2p_v1/chat`). Pure `chat` messages sent when no agent is polling are only preserved if the agent previously subscribed with the **same `clientId`** and `cleanSession: false` (persistent MQTT session).
- **Multi-agent visibility & conflict**: Both agents and the desktop client share `agy_p2p_v1/chat` and `agy_p2p_v1/cmd/#`. If two agents use the same `clientId` (e.g., the default `agy_cloud_agent_v1`), EMQX performs a session takeover and whichever agent polls next consumes the queued messages. Using distinct `clientId`s (e.g. `agy_agent_b_session`) and targeted `cmd/<agent>` + `resp/<requestId>` topics isolates approval responses cleanly.

### (2) Why mobile messages were sent twice (`~1ms` apart, e.g. `3.3 KB` then `147 B`)
- In `apps/p2p-remote-controller/src/App.tsx`, the `useEffect` registering `listen<PocketDeckRemoteAction>("lan:remote-action", ...)` stored `unlisten` asynchronously inside `.then((fn) => { unlisten = fn; })`.
- Under React `<StrictMode>` in development, `useEffect` mounts, unmounts before `listen()` resolves (while `unlisten` is still `null`), and remounts—leaving **two active `lan:remote-action` listeners**. Every mobile `POST /api/action` triggered `handleSendEmqxChat` twice: the first call consumed `recentUserActions` (`3.3 KB`) and updated `lastSyncedTimelineIdRef`, and the second call 1ms later sent an empty diff (`147 B`).

### (3) Why `Shift+Return` sent instead of inserting a newline
- Both `PocketDeckView.tsx` and `EmqxBridgePanel.tsx` used a single-line `<input type="text">` and checked `if (e.key === "Enter")` without `!e.shiftKey` or `!e.nativeEvent.isComposing`.

## 2. Implemented Fixes (Verified Bot Commits on `a2`)

1. **`9020931`** — `fix(p2p-remote-controller): prevent mobile double-send and support Shift+Enter newlines`
   - `apps/p2p-remote-controller/src/App.tsx`: Added a `disposed` guard to the `lan:remote-action` `useEffect` so any `listen()` resolving after unmount immediately calls `fn()`.
   - `apps/p2p-remote-controller/src/components/PocketDeckView.tsx` & `EmqxBridgePanel.tsx`: Replaced `<input type="text">` with `<textarea rows={1}>` and gated Enter-to-send on `e.key === "Enter" && !e.shiftKey && !e.nativeEvent.isComposing` with `e.preventDefault()`.
2. **`32cadcc`** — `fix(p2p-remote-controller): move Bridge Chat input above newest-first activity list`
   - `apps/p2p-remote-controller/src/components/EmqxBridgePanel.tsx`: Moved the `Send message to cloud agent...` textarea + Send button directly beneath the `Bridge Chat / Activity` header, right above the newest-first message list.

## 3. Toolchain & Verification
- Installed `pnpm` (`12.21.0`) and workspace dependencies (`pnpm install`).
- Ran `pnpm --filter p2p-remote-controller test` (`2 test files, 8 tests passed`), `pnpm --filter p2p-remote-controller build` (`tsc && vite build` succeeded), and `eslint` on modified files (`PocketDeckView.tsx`, `EmqxBridgePanel.tsx` — 0 errors).
- Verified live bidirectional communication with the user over `agentplayground.maboco-mark.workers.dev/mqtt/rpc` (`ask_user` -> `reply_only` received on `agy_p2p_v1/resp/req_agent_b_hello_1`).
