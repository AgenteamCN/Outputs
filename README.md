# Incident & Hotfix Report: `agentlivechatchannel` DO Free Tier Quota Exhaustion (2026-11-27)

## 1. Executive Summary
- **Incident**: Between `18:23Z` and `18:58Z` on `2026-11-27`, Cloudflare Worker `agentlivechatchannel` (`live-chat-v2.2-do`) exceeded the **Cloudflare Durable Objects SQLite Free Tier** daily write limit (**100,000 rows written / day**), reaching **`101.34k` SQL rows written** across `60.12k` requests (`596.91k` SQL rows read).
- **Critical Platform Behavior**: Once an account's daily DO SQLite `rows_written` quota is exceeded, Cloudflare rejects **not only `INSERT`/`UPDATE`/`DELETE`**, but **even `SELECT` queries** on SQLite-backed DOs (`Error: Exceeded allowed rows written in Durable Objects free tier.`), surfacing externally as HTTP 500 `error code: 1101`.
- **Impact**:
  - Newly dispatched Partners (`lucas`, `mason`) hit `1101` on `/chat/poll`, `/version`, `/help` and exited.
  - Existing Partners (`atlas_956d5fe0`, `simon_b67a5399`, `oscar2_36de5bd1`, `victor_98c27bac`, `secretary_fea5ee28`) lost DO SQLite `do/auth` and `do/status` access.
  - Chief <-> Human communication remained online because Chief authenticates via static `AUTH_UUID` env var and `/chat/send` + `/chat/poll` talk directly to EMQX (`do/status` errors are `.catch(() => null)`).

---

## 2. Root Cause Analysis (From Live Deployed `ChannelCoordinatorDO` Source)
Inspection of the live deployed `agentlivechatchannel` bundle (`src/worker.mjs`, `src/coordinatorDo.mjs`) identified the exact high-frequency SQLite write paths (~1.7 rows written per request):
1. **`POST /status` (`updateChannelStatus`)**: Executed `INSERT OR REPLACE INTO channel_status` on almost every `/chat/send`, `/chat/poll`, `/chat/send_and_poll`, and `GET /help` (`prewarmed`).
2. **`POST /presence/heartbeat`**: Executed `INSERT`/`UPDATE agent_presence` (`last_seen_at`) on heartbeats.
3. **`POST /buffer/append`**: Executed `INSERT INTO channel_message_buffer` + `DELETE FROM channel_message_buffer` on chat messages.
4. **`flushHotCursors()` / `alarm()`**: Flushed `poll_client_cursors` to SQLite every 10s.
5. **Account-wide Lockout**: Once `101.34k > 100k` was reached, even pure `SELECT`s (`POST /auth`, `GET /agents`, `POST /agents/revoke`) threw `Exceeded allowed rows written in Durable Objects free tier.`.

---

## 3. Emergency Hotfix Deployed (Zero-SQLite-Write Hot Path)
- **Rollback Anchor (Previous Healthy Deployment)**:
  - Deployment ID: `fa0f9b70-236d-4a16-adb2-936a625c3c43`
  - Version ID: `02c7b105-6dff-4276-8276-13fcb4c9c255` (`2026-11-27T08:04:33Z`)
- **Changes Applied in `ChannelCoordinatorDO` (`worker.patched.js`)**:
  1. **Status & Active Channels (`POST /status`, `GET /status`, `GET /channels/active`)**: Migrated from SQLite `channel_status` table to DO in-memory `this.memChannelStatus` (`Map`). **0 SQLite writes.**
  2. **Message Buffer & Poll Cursors (`/buffer/append`, `/buffer/poll-start`, `/buffer/poll-end`, `flushHotCursors`, `getHotCursor`)**: Migrated from SQLite `channel_message_buffer` & `poll_client_cursors` to DO in-memory `this.memMessageBuffer` (`Map`) and `this.hotCursors` (`Map`). **0 SQLite writes.**
  3. **Schema & Error Visibility**: Wrapped `ensureSchema()` DDL in `try/catch` and `ChannelCoordinatorDO.fetch()` in `try/catch` (returning structured `DO_CAUGHT_ERR` JSON instead of opaque `1101`).
  4. **In-Memory Agent Issue/Auth/Revoke Fallback**: Added `this.memAgents`, `this.memRevokedTokens`, `this.memRevokedAgentIds` so newly issued Partners via `POST /agents/issue` can authenticate via `POST /auth` without touching SQLite.
- **Verification Results Post-Deploy**:
  - **Working (`200 OK`)**: `GET /channels/active`, `GET /channel/status`, `GET /help`, `POST /chat/send`, `POST /chat/poll`, `POST /chat/send_and_poll`, `POST /cmd`, `POST /agents/issue` (tested `issueSeq: 101`, `ch_code_101_a0930f`).
  - **Still Blocked by Platform SQLite Lockout (until UTC 00:00 reset or Paid plan)**: `GET /agents` and `POST /agents/revoke` when querying pre-existing rows in SQLite `agents` table, as well as `POST /auth` for pre-existing Partners whose tokens live only in SQLite.

---

## 4. Additional Findings & Project State Snapshot
### 4.1 Why Chief Missed `lucas` & `mason` `[standby]` at `18:16`–`18:17Z`
- Chief had not added `ch_lucas_24634` / `ch_mason_24637` to `/chat/poll` `channelIds` prior to `batch_dispatch_tasks`, and while Chief was blocked waiting on `/cmd`, `/chat/poll` was not active. Without DO/EMQX gap replay on first subscribe, those `[standby]` messages were missed.
- **Action**: Chief must register/subscribe new `channelIds` *before* `/cmd dispatch_agent_task`, AND Worker should return a subscribe reminder + support gap replay without SQLite writes.

### 4.2 `scan_all_windows` Filtering (`apps/p2p-remote-controller`)
- Verified `emqxCommandRouter.ts:498-503`: `scan_all_windows` filters by `deps.managedTargets` (windows checked/managed in `p2p-remote-controller` UI), not hardcoded URLs. Window `#24832` was a Cloudflare Dashboard Safari tab, not in `managedTargets`, while `#24811` and `#24813` are `[ready]` AI Studio windows.

### 4.3 `agyDevKit` Repository & In-Flight Tasks Snapshot
- **`origin/a2` `HEAD`**: `a96587ce5bdc756205b5360b0b385e4b66911873` (`verified: true`, clean).
- **Already Merged to `origin/a2`**:
  - T-244 (`PocketDeckView` status bubble unification)
  - T-243 (`emqx-live-chat` `127.0.0.1:14210/ws` auto-probe & `lan_delta`)
  - T-248 (`hygemi-setup` Phase 4 WP-A)
  - T-249 (`hygemi-setup` WSS hang/cancel fix + Phase 4 WP-C)
- **In-Flight / Queued Tasks**:
  - **T-251-FIX** (`a2-feat-hygemi-setup-wp-b` @ `984d9c3f`): Remove 6 `let _ = fs::remove_file` in `toolchain_downloader.rs`, rotate cross-review (`simon`/`oscar2`, NOT `victor`), merge -> `origin/a2`.
  - **Critical `rustls` Fix** (`apps/hygemi-setup`): Add `rustls = { version = "0.23", default-features = false, features = ["ring"] }` and call `rustls::crypto::ring::default_provider().install_default()` at startup (prevents `str0m`/`tokio-tungstenite` dual-provider silent panic), then trigger `hygemi-setup-windows.yml`.
  - **T-239** (`a2-feat-lan-share-public-private-upload` @ `2674aec7`): Awaiting cross-review pass (`T-256`) -> merge.
  - **T-226** (`unified-tray` Private Window IPC): Re-assigned to `oscar2`.
  - **T-252**: `emqx-live-chat` `SIGUSR1` unhide listener + `unified-tray` GUI filter (queued after T-226).
  - **T-253 & Task Manager SSE Issues**: (a) Steps SSE regression; (b) SSE pause/resume broken; (c) 100% CPU + UI freeze + continuous network traffic when subscribing all SSE streams.
  - **T-258**: `batch_dispatch_tasks` approval UI truncation in `p2p-remote-controller`.

---

## 5. Recommended Long-Term Worker Architecture (Zero DO SQLite Dependency)
1. **Auth & Multi-Team Agent Registry -> Cloudflare KV**:
   - Free tier: `100,000 reads/day`, `1,000 writes/day` (never locks `SELECT`s).
   - Store `chief:<tokenHash>` and `agent:<tokenHash>` with `{ ownerChiefChannelId, seq, role, allowedChannels }` to isolate multiple `Human + Chief + Partner` teams.
   - Derive `agentId = <name>_<sha256(pepper:chiefChannelId:name:seq)[:8]>` so crashed/revoked Partners can be re-issued with the same clean name (incrementing `seq`) without `name2` suffixes.
2. **Presence & Status -> EMQX `retain: true` + Worker Memory**:
   - Publish status updates as `retain: true` to `agy_p2p_v1/status/<channel>`, eliminating database writes entirely.
3. **Gap-Free Messaging -> EMQX Persistent Session (`cleanStart: false`) + Memory Ring Buffer**:
   - Use stable `clientId` per Chief/Partner so QoS 1 messages sent during `/cmd` execution are queued by EMQX and delivered on next `/chat/poll`.
4. **Cross-Container Backups (`/channel/backup`) -> Cloudflare R2 (or KV)**:
   - R2 free tier (`10 GB`, `1M Class A ops/month`) for `/workspace/.chat/` archive snapshots.
