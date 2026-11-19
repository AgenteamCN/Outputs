# P2P Remote Controller Updates, EMQX Chat Verification, and Apple Silicon PWA Leaf Certificate Guide

## 1. Repository Updates (`Antigravi-ty/agyPreviewInstant`, branch `a2`, commit `3643277323c61735d74354a7cc77ede71d81bf91`)

All checks (`pnpm check` — `tsc --noEmit` + `eslint src`, and `pnpm test` — 8/8 tests) passed.

- **Auto-Connect EMQX & Auto-Enable LAN Sharing on Startup** (`apps/p2p-remote-controller/src/App.tsx`):
  - On initial mount, if `brokerUrl`, `username`, and `password` are configured in `emqxConfig`, `handleConnectEmqx()` is called automatically.
  - On initial mount in Tauri runtime, `updateLanConfig({ enabled: true })` is called automatically and stored in `lanStatus`.
- **LAN Sharing Button Active State** (`apps/p2p-remote-controller/src/App.tsx`, `src/components/LanSharingModal.tsx`):
  - Added `lanStatus` tracking and `onStatusChange` callback from `LanSharingModal`.
  - The `LAN Sharing` header button now uses `className={lanStatus?.running ? "primary" : ""}`, turning blue when active and gray when stopped (matching the `EMQX: Live` button).
- **Pending Reply vs. Notifications & Chat** (`apps/p2p-remote-controller/src/components/EmqxBridgePanel.tsx`, `src/components/PocketDeckView.tsx`):
  - Renamed `Pending Approvals` to **Pending Reply** (`Requests awaiting your real-time reply or approval`).
  - Renamed `Bridge Chat & Activity` to **Notifications & Chat**.
  - For `ask_user` requests, **Send Reply** (`reply_only`) is highlighted as the primary action.
- **Open Certificate Folder in Finder** (`apps/p2p-remote-controller/src-tauri/src/lan_server.rs`, `src-tauri/src/lib.rs`, `src/services/lanService.ts`, `src/components/LanSharingModal.tsx`):
  - Added Tauri command `lan_open_cert_dir` which ensures the certificate directory exists and opens it in macOS Finder (`open <cert_dir>`).
  - Replaced the Copy button next to `certDir` in `LanSharingModal` with a folder icon button (`FolderOpen`).

---

## 2. EMQX Chat Push vs. Poll Verification

- **Verified via `https://agentplayground.maboco-mark.workers.dev/mqtt/rpc`**:
  - Your message on `agy_p2p_v1/chat` (`"你可以看一下我这次的回复，你那边是直接收到的，还是说你必须要手动查询一下..."`, sent at `2026-11-19T06:36:39.847Z`) was **queued by EMQX** (because the agent uses `clientId: "agy_cloud_agent_v1"` with `cleanSession: false` and QoS 1) and **delivered immediately at `2026-11-19T06:39:41.778Z` when I polled `/mqtt/rpc`**.
  - **Conclusion**: Because this cloud sandbox does not run a persistent background daemon that wakes the LLM on incoming sockets, I **do not** receive unsolicited pushes while idle—I must call `/mqtt/rpc` (which can long-poll up to `55000ms`) to receive messages. However, EMQX queues QoS 1 messages for `cleanSession: false`, so your messages are preserved until my next poll.

---

## 3. Step-by-Step Guide: Generating a PWA-Ready Leaf Certificate on Apple Silicon Mac

Since you already have a **Root CA** (`rootCA.crt` + `rootCA.key`), here is what to run in **Apple Silicon Mac Terminal**, what parameters are required for **iOS Safari / PWA**, and what info to share if you want me to tailor the exact config for you.

### Step 1: Gather Your Mac Network & Root CA Info
Run these in your Mac Terminal (and optionally send me the output via EMQX Chat):
```bash
# 1. Your Mac's LAN IP (Wi-Fi is usually en0; Ethernet may be en1/en5)
ipconfig getifaddr en0

# 2. Your Mac's .local Bonjour hostname
echo "$(scutil --get LocalHostName).local"

# 3. Check OpenSSL
openssl version
```
*(Also note the paths/filenames of your existing Root CA certificate and private key, e.g. `~/certs/rootCA.crt` and `~/certs/rootCA.key`.)*

### Step 2: Critical Parameters for iOS Safari & PWA (`leaf.crt` + `leaf.key`)
Apple ATS / iOS Safari / Service Workers (required for PWA) enforce strict rules:
1. **`subjectAltName` (SAN) is mandatory** (Common Name alone is rejected):
   - Must include your Mac LAN IP: `IP:192.168.x.x`
   - Recommended to also include `IP:127.0.0.1`, `DNS:localhost`, and `DNS:<LocalHostName>.local`
2. **Validity <= 398 days**: Apple rejects leaf certificates valid for more than 398 days. Use **`-days 365`**.
3. **Key Usage & Extended Key Usage**:
   - `basicConstraints = critical, CA:FALSE`
   - `keyUsage = critical, digitalSignature, keyEncipherment`
   - `extendedKeyUsage = serverAuth`
4. **Unencrypted Private Key (`leaf.key`)**:
   - `lan_server.rs` loads `leaf.crt` and `leaf.key` directly without a passphrase prompt, so generate `leaf.key` without a passphrase (`-nodes` / no `-aes256`).
   - ECDSA P-256 (`prime256v1`) or RSA 2048+ are both supported by `tokio-rustls`.

### Step 3: Generate `leaf.key` and `leaf.crt` in Terminal
Replace `192.168.1.100`, `my-mac.local`, and `~/path/to/rootCA.crt` / `rootCA.key` with your actual values:

```bash
# Create a working directory
mkdir -p ~/p2p-leaf-cert && cd ~/p2p-leaf-cert

# Set your Mac LAN IP and .local hostname
MAC_IP="$(ipconfig getifaddr en0)"
MAC_LOCAL="$(scutil --get LocalHostName).local"
echo "Using IP=$MAC_IP and HOST=$MAC_LOCAL"

# 1. Generate unencrypted ECDSA P-256 private key (leaf.key)
openssl ecparam -name prime256v1 -genkey -noout -out leaf.key

# 2. Create CSR
openssl req -new -key leaf.key -out leaf.csr \
  -subj "/CN=${MAC_LOCAL}"

# 3. Create X.509 v3 extensions file for iOS Safari / PWA
cat > leaf.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
DNS.1=localhost
DNS.2=${MAC_LOCAL}
IP.1=127.0.0.1
IP.2=${MAC_IP}
EOF

# 4. Sign leaf.crt with your existing Root CA (365 days <= 398-day Apple limit)
openssl x509 -req -in leaf.csr \
  -CA /path/to/rootCA.crt \
  -CAkey /path/to/rootCA.key \
  -CAcreateserial \
  -out leaf.crt \
  -days 365 \
  -sha256 \
  -extfile leaf.ext
```

### Step 4: Verify & Install into P2P Remote Controller
```bash
# Verify SAN, EKU, and validity dates
openssl x509 -in leaf.crt -noout -text | grep -A 3 -E "Subject Alternative Name|Extended Key Usage|Validity"
```
1. In **P2P Remote Controller**, click **LAN Sharing** -> click the **Folder** icon next to the certificate directory to open it in Finder.
2. Copy `leaf.crt` and `leaf.key` into that Finder window.
3. Toggle **Disable -> Enable LAN** (or restart the app) — the badge will switch from `HTTP Fallback` to **`HTTPS / WSS Active`**.
4. On iPhone/iPad: ensure your Root CA is installed AND enabled under **Settings -> General -> About -> Certificate Trust Settings**, then open the `https://<MAC_IP>:14210/?view=pocket-deck` URL in Safari and **Add to Home Screen** for PWA mode.
