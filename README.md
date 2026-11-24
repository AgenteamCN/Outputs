# AGY DevKit & Chatting System Session Report (2026-11-24 Final)

## 1. Repository State
- **Branch**: `Antigravi-ty/agyDevKit` -> `a2`
- **Latest Verified Bot Commit HEAD**: `e0a874a` (`merge(a2-alpha): task_alpha_072 iOS LAN 30s probe cap, tap Offline pill, cross-origin Passkey handoff`)
- **Mac Sync Note**: Mac was previously at `43aed2a`. Run `quick_git pull_ff` to `e0a874a`, then rebuild/restart `AGY Terminal` / `AGY AX Terminal` / `Unified Tray` (Rust updates in `067`, `069`, `070`, `071`); `p2p-remote-controller` frontend updates (`068`, `072`) hot-reload cleanly.

---

## 2. Newly Completed Tasks Since `c6af380`
1. **`task_alpha_070` (`b18e82e` on `a2`) — Terminal `.backup.app` Build Rotation & Caller-Aware Routing**
   - `scripts/macos/build-runners.sh`: Added `install_bundle_with_rotation` and `--rotate-backup=yes|no|ask` (default `ask`, non-TTY treated as `no`). Creates `.backup.app` if absent; if `.backup.app` exists, prompts `"Is current active version of terminal working properly? [y/N]: "` and only rotates on `y`/`Y`.
   - `apps/agy-terminal/src-tauri/src/main.rs` & `apps/unified-tray/src-tauri/src/{main.rs,ipc_client.rs}`: `AGY Terminal` passes `AGY_CALLER_APP` alongside `AGY_STATE_DIR` to `Unified Tray`; `Unified Tray` resolves `AGY AX Terminal.backup.app` (with fallback to `AGY AX Terminal.app` if missing) when launched from a `.backup.app` caller while sharing `.agy-state`.
2. **`task_alpha_071` (`3f1b4ee` on `a2`) — Runner Single-Instance Lock & `.agy-state` Mismatch Detection**
   - `packages/runner-core/src/{lock.rs,lib.rs,ipc.rs}`, `apps/agy-terminal`, `apps/agy-ax-terminal`, `apps/unified-tray`: Added atomic `SingleInstanceLock` (`create_new` + alive-PID check + stale-lock recovery) preventing duplicate instances from clobbering active sockets.
   - Added `active_state_dir.txt` breadcrumb + `.agy-state` mismatch detection (`stateDirWarning` in `TrayStatusOverview`, system logs, and `App.tsx` banner).
3. **`task_alpha_072` (`e0a874a` on `a2`) — iOS LAN 30s Probe Cap, Tap `Offline` Pill & Cross-Origin Passkey Handoff**
   - `apps/p2p-remote-controller/src/utils/lanAutoProbe.ts`: Tightened `DEFAULT_PROBE_BACKOFF_MS` to `[5_000, 10_000, 20_000, 30_000]`.
   - `apps/p2p-remote-controller/src/components/PocketDeckHeader.tsx`: Made `Offline` status pill clickable to trigger `lanProbe.probeNow(true)`.
   - `apps/p2p-remote-controller/src/utils/webauthnPasskey.ts` & `PocketDeckView.tsx`: Persisted `rpId` (`agy_pocket_passkey_rp_id_v1`), skipped incompatible prompts on `!isSecureContext` or `rpId` mismatch, and added `#passkey_trust=<expiresAtMs>` cross-origin trust handoff.

---

## 3. Verification Gate (All Passing on `a2` @ `e0a874a`)
- `bash -n scripts/macos/build-runners.sh`: **PASS**
- `pnpm --filter p2p-remote-controller exec tsc --noEmit`: **PASS**
- `pnpm --filter p2p-remote-controller test -- --run`: **52 test files, 197 passed**
- `pnpm --filter p2p-remote-controller build`: **PASS** (2169 modules)
- `pnpm --filter @antigravity/unified-tray exec tsc --noEmit`: **PASS**
- `cargo test --manifest-path packages/runner-core/Cargo.toml`: **9 passed, 0 failed**
- `cargo test --manifest-path apps/unified-tray/src-tauri/Cargo.toml`: **20 passed, 0 failed**
- `cargo check --manifest-path apps/agy-terminal/src-tauri/Cargo.toml`: **PASS**
- `cargo check --manifest-path apps/agy-ax-terminal/src-tauri/Cargo.toml`: **PASS**
- `pnpm run env:check`: **PASS** (`OverlayFS size: 73 MB <= 100 MB`, `Cargo target/: clean`)

---

## 4. Investigation `#72`: Worker `curl` vs Python `urllib` HTTP 403
Tested directly against `https://agentlivechatchannel.maboco-mark.workers.dev/help`:
- **Python `urllib.request` (default headers)**: `HTTP 403 Forbidden`, `Server: cloudflare`, `Body: b'error code: 1010\n'`
- **Python `urllib.request` with `User-Agent: curl/8.5.0`**: `HTTP 200 OK` (`2097 bytes`)
- **Python `requests` (default headers)**: `HTTP 200 OK` (`2097 bytes`)
- **`curl` CLI**: `HTTP 200 OK`
- **Root Cause**: Cloudflare Error `1010` blocks `Python-urllib/3.12`'s default `User-Agent` signature via Browser Integrity Check / WAF. Using `curl`, `requests`, or setting a non-`Python-urllib` `User-Agent` resolves it immediately.

---

## 5. Investigation `#73`: `send_and_poll` Self-Echo & Sub-Agent Routing
- **Observation**: While `/cmd` `alsoListenChat` was fixed in `agyWorker` (`ca3f467`), `POST /chat/send_and_poll` still echoes `sender: "agent"` messages on subscribe, causing `send_and_poll` to return immediately before a human replies.
- **Operational Guardrail**: Always inspect `received[]` on `send_and_poll`; if no human message is present, immediately follow up with `POST /chat/poll`.

---

## 6. Proactive Pitfalls & Guardrails Summary (Saved in `/workspace/.chat/pitfalls_and_guardrails.md`)
1. **Pre-Merge Rust Gate**: Always run `cargo test` / `cargo check` on modified Rust crates before merging to `a2`, paired with `git_watchdog.rs` (45s rollback) and `build-runners.sh` `.backup.app` rotation on Mac.
2. **macOS Main-Thread & LaunchServices Safety**: Never call AppKit UI / `requestUserAttention` off the main thread; wait for `Unified Tray` before `tauri::Builder::default().run(...)` (20s cap + `!` badge fallback).
3. **Zero `osascript System Events`**: Use `CGWindowListCopyWindowInfo`, `ps -eo pid=,ppid=`, `SIGUSR1`, and `NSRunningApplication`.
4. **Shared `.agy-state` & Single-Instance Lock**: Always pass `AGY_STATE_DIR` + `AGY_CALLER_APP`, enforce `SingleInstanceLock`, and surface `stateDirWarning` on mismatch.
5. **OverlayFS Budget**: Keep `target/` offloaded to `/tmp/agy-cache/cargo-target` and verify `pnpm run env:check` (`<= 100 MB`).
6. **Worker & Secret Hygiene**: Use `curl` (avoid `Python-urllib` 1010), use `"payload"` on `/channel/backup`, never put emojis in `agyDevKit`, and never post sensitive credentials/Initial User Input to `agentoutputposter`.

---

## 7. Remaining Backlog for Next Session
- `p2p-remote-controller` split (`EMQX Live Chat` vs remote control) & LAN sharing / Mac PWA architecture
- `agyWorker` updates (`#40` dead-connection `PINGREQ` / token revocation, `#53` `includeMySubAgents: true`, `#35` APNs & monorepo consolidation, `/help` sync for `channels[]` & `payload`)
- `#38` (`sing-box` observation), `#42` (AI Studio `Inject Live Chat` AX calibration), `#43` (`EMQX Live Chat.app`), `#44` (LAN remote Mac dictation), `#48` (`agy-preview-instant` rename/bug), `#71` (Windows dual-controller architecture)
