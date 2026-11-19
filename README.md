# P2P Remote Controller Fixes & Leaf Certificate Guide

## 1. Verified Commit Pushed to `a2`
- **Commit**: `be2a1d4dc672d77a8246c1fb643ffaa82ad924b3` (`signature.isValid: true`, `state: VALID`)
- **Checks**: `pnpm --filter p2p-remote-controller exec tsc --noEmit` and `pnpm --filter p2p-remote-controller test` (2 test files, 8 tests passed).

### What Was Fixed
1. **LAN Server Dev Mode Proxy (No `dist/` Build Required)** (`apps/p2p-remote-controller/src-tauri/src/lan_server.rs`)
   - Added `try_proxy_vite_dev_server` (`127.0.0.1:1421` / `[::1]:1421`) and `proxy_vite_hmr_websocket` so `pnpm p2p:dev:watch` serves the live Vite dev server directly on LAN HTTP/HTTPS without running `pnpm build`, with `app.asset_resolver()` and `../dist` fallbacks.
2. **Direct Domain/IP Access Without URL Query** (`apps/p2p-remote-controller/src/main.tsx`)
   - Non-Tauri browsers now default to `PocketDeckView` automatically when visiting `http(s)://<host>:<port>/` without needing `?view=pocket-deck` (unless `?view=desktop` is explicitly passed).
3. **`mobile-viewport` `event.emit` Permission & `EMQX: disconnected` Fix**
   - `apps/p2p-remote-controller/src-tauri/capabilities/default.json`: Added `"mobile-viewport"` to `"windows": ["main", "mobile-viewport"]`, resolving `event.emit not allowed on window "mobile-viewport"`.
   - `lan_server.rs`, `lib.rs`, `lanService.ts`, `PocketDeckView.tsx`: Added `lan_get_snapshot` Tauri command and invoked it on mount in `PocketDeckView` so `mobile-viewport` immediately receives the current `emqxState`, `pendingApprovals`, `chatMessages`, and windows instead of staying stuck on `EMQX: disconnected` until the next state change.
4. **Initialize & Record + Target URL + Clear Net Buffer** (`apps/p2p-remote-controller/src/App.tsx`, `src/services/networkRecorderScript.ts`)
   - **Code-driven Target URL**: Updated `DEFAULT_TARGET_URL` to `https://aistudio.google.com/prompts/new_chat?model=antigravity-preview-09-2026` and removed stale `localStorage` override on startup.
   - **Step-by-step Page Readiness**: `handleQuickInitAndRecord` now reports `Step 1/5` through `Step 5/5` in `netVerifyDetail`, probes `document.readyState` + `location.href` + `scanSafariWindow`, redirects if stuck on `about:blank`, verifies recorder injection, and polls initial network state.
   - **Clear Net Buffer & `fetch` `tee()` Guard**: `handleClearNetRecorder` now updates `netRecording` / `netVerifyStatus` / `netVerifyDetail` accurately, and `networkRecorderScript.ts` skips `response.body.tee()` for null-body statuses (`0`, `204`, `205`, `304`) with a `try/catch` fallback.

---

## 2. Generating `leaf.crt` & `leaf.key` for HTTPS/WSS on Mobile

`lan_server.rs` looks for `leaf.crt` and `leaf.key` in `apps/p2p-remote-controller/src-tauri/certs/` (in dev mode). Without them, the server runs plain HTTP, which causes `Could not establish a secure connection to the server` when accessed over `https://`.

### Option A: Using `mkcert` (Recommended for iOS / macOS / LAN)
```bash
mkdir -p apps/p2p-remote-controller/src-tauri/certs
mkcert -install
mkcert \
  -cert-file apps/p2p-remote-controller/src-tauri/certs/leaf.crt \
  -key-file apps/p2p-remote-controller/src-tauri/certs/leaf.key \
  localhost 127.0.0.1 <YOUR_LAN_IP> <YOUR_CUSTOM_DOMAIN>
```
*(Install the `mkcert` root CA on your phone via AirDrop/Settings -> Profile Downloaded -> Certificate Trust Settings so iOS trusts the leaf cert.)*

### Option B: Using `openssl` (Self-Signed SAN Leaf Certificate)
```bash
mkdir -p apps/p2p-remote-controller/src-tauri/certs
openssl req -x509 -newkey rsa:2048 -nodes -sha256 -days 365 \
  -keyout apps/p2p-remote-controller/src-tauri/certs/leaf.key \
  -out apps/p2p-remote-controller/src-tauri/certs/leaf.crt \
  -subj "/CN=p2p-remote-controller" \
  -addext "subjectAltName=DNS:localhost,DNS:<YOUR_CUSTOM_DOMAIN>,IP:127.0.0.1,IP:<YOUR_LAN_IP>"
```
Restart or toggle the LAN server in the app after placing `leaf.crt` and `leaf.key`.

---

## 3. Note on EMQX Chat Channel
`agentplayground.maboco-mark.workers.dev/mqtt/rpc` was verified reachable, but your private EMQX Cloud broker hostname was not included in the prompt (only username `CNremoteController` and password). Please pull `origin/a2` (`git fetch && git reset --hard origin/a2`) to get commit `be2a1d4`.
