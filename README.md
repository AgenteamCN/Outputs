# AGY DevKit & Chatting System Session Report (2026-11-24)

- **Main Repo (`Antigravi-ty/agyDevKit`)**: branch `a2` HEAD `c6af380` (verified via git log & repo scan)
- **Worker Repo (`Antigravi-ty/agyWorker`)**: branch `main` HEAD `3939f3a` (deployed live: Version `84902513-b1d4-4748-ac22-81421ca89d3f`)
- **Active Channel**: `ch_b308b620` | **Sub-Agent**: Alpha (`ch_sub_d3b650df5b` / `a2-alpha`)

---

## Part 1: Completed Items (Verified Against Repo & Git History)

### A. `Antigravi-ty/agyWorker` (`main` @ `3939f3a`, Live Deployed)
1. **DO `/channel/backup` -> MQTT `backup_verified` Push (`3939f3a`)**: `POST /channel/backup` now publishes `{ type: "backup_verified", backupEventId, sha256, verifiedAt, channel }` on `${defaults.topicPrefix}/status/${doData.channel}` (`retain: false`), resetting client `unbackedCount`.
2. **Worker Core Hardening (`ca3f467`, deployed `a1760ab9`)**:
   - `/cmd` `alsoListenChat` defaults to `false` + `cmdExcludeSender` includes `defaults.clientId` so self-echo/chat does not prematurely abort `/cmd`.
   - `/chat/poll` per-agent single-channel `clientId` (`agy_poll_${agentId}_${channel}`) + multi-channel `channels: string[]` (`#30`).
   - `POST /chat/send` `400 BAD_REQUEST` validation on missing `text` + tiered `GET /help?topic=cmd|chat|agents|backup|all`.
   - Non-admin `excludeSender` fix (`#33`), `wrangler.toml` `keep_vars = true` (`#34`), and `ChannelCoordinatorDO` SQLite `channel_checkpoints` (`GET/POST /channel/backup`).

### B. `Antigravi-ty/agyDevKit` (`a2` @ `c6af380`)
1. **Tray, Dock & Process Lifecycle (`Unified Tray`, `AGY Terminal`, `AGY Preview Instant`, `P2P Remote Controller`, `P2P Browser`, `Singbox GUI`)**:
   - `task_alpha_061` (`9986f3d`): Upgraded `Unified Tray` native menu to `IconMenuItem` with 5 anti-aliased RGBA status icons + `CGWindowListCopyWindowInfo` visible-window detection.
   - `task_alpha_062` (`accc36e`): Injected `AGY_TRAY_MANAGED=1` for preset child processes in `runner-core` & `unified-tray`.
   - `task_alpha_063` (`950a1fe`): Added `ActivationPolicy::Accessory` on `CloseRequested` hide, `Regular` on reopen, `#[cfg(unix)] SIGUSR1` unhide/focus handlers (`p2p-remote-controller`, `p2p-browser`, `singbox-gui`), and removed all `osascript System Events` calls from `Unified Tray`.
   - `task_bravo_021` (`d89e03e`): Added `git_watchdog.rs` in `Unified Tray` for `last_good_commit.txt`, 45s health-check window, and automatic `git reset --hard` rollback on crash/build failure.
   - `7b27ae6` + `9dc603d` + `41085e4`:
     - Added `is_tray_managed()` close-to-tray (`Accessory`/`Regular` + `SIGUSR1`) to root `AGY Preview Instant` (`src-tauri/src/lib.rs`).
     - Added `ps -eo pid=,ppid=` descendant-PID foreground matching (`pnpm` -> Tauri child) & `TrayRunningSinceState` `(started_at, has_been_foreground)` so closing a window goes straight to `RunningBackground` (blue ring) without relapsing into `Starting` (amber dot).
     - Added `TrayMenuCache` with stable `preset:<id>` / `stop:<id>` / `force_stop:<id>` IDs to update icons & enabled states **in-place** without calling `tray.set_menu` (menu stays open during live updates).
     - Added `set_macos_dock_icon()` via `run_on_main_thread` when transitioning `Accessory` -> `Regular` (`AGY Preview Instant`, `p2p-remote-controller`, `p2p-browser`).
     - Passed `AGY_STATE_DIR` from `AGY Terminal` to `Unified Tray` so `.app` sibling `.agy-state` and `tauri dev` share the exact same socket/log directory.
   - `task_alpha_064` (`117010d` + `21a1bbe`) -> `task_alpha_067` (`3298fe3`) -> `task_alpha_069` (`c6af380`):
     - `AGY Terminal` waits up to 20s for `Unified Tray` (`read_alive_tray_pid`) before entering `tauri::Builder::default().run(...)` so macOS LaunchServices provides the authentic native Dock bounce (zero `requestUserAttention`/`objc2` bounce loop); on <=20s success switches to `Accessory`; on 20s timeout keeps `Regular` + sets `dockTile().setBadgeLabel("!")` + logs error to stderr.
2. **EMQX, Network Recorder, Pocket Deck & API Caller (`AGY Preview Instant`)**:
   - `task_alpha_058` + `bee6d9d` (`d4f20db` + `cb3cbee`): Fixed `ActiveTodoPanel` / `TodoSyncPanel` narrow-rail clipping, full-height flex scroll, `100dvh` mobile cap, and centered TODO modal via `createPortal`.
   - `task_alpha_059` (`cb3cbee`) + `21a1bbe`: Restricted `networkRecorderScript.ts` `isAgentInteractionRpcUrl` to `CreateInteraction`, `GetInteractionStream`, `/interactions/` (explicitly blocking `GenerateContent`, `ListInteractions`, `GetInteraction`) + `scriptVersion: 3` hot-patching & pruning for already-installed Safari tabs.
   - `779ad11` + `13d228a`: Fixed `PocketDeckView.tsx` `↑ Back to Quote` disappearance & smooth-scroll `followBottom` jitter (verified working on Mac).
   - `task_alpha_065` (`4df1be1`): Fixed `Try Only Output` (`refresh_output_only`) HTTP 400 `INVALID_ARGUMENT` by removing unsupported `?fields=id,...` query in `src-tauri/src/google_api/output_recovery.rs`.
   - `task_alpha_066` (`43aed2a`): Fixed Mac EMQX WebSocket `Closed (code 1005)` silent drop & added fast keepalive / reconnect watchdog.
   - `task_alpha_068` (`0327d64`): Fixed duplicate EMQX inbound chat messages on reconnect (QoS 1 redelivery dedup & skip redundant `SUBSCRIBE`).
3. **Environment, Singbox & Monorepo (`a2` verified in earlier commits)**:
   - `task_alpha_060` (`bdef0d8`): Dynamically symlinked all `apps/*/src-tauri/target` & `packages/*/src-tauri/target` to `/tmp/agy-cache/cargo-target` (OverlayFS ~31–72 MB).
   - `#24` (`bafe826`): `Singbox GUI` 500k-line / 64MB `LogRingBuffer` + virtualized console.
   - `#39` (`415ccec` & prior): Absorbed `P2PBrowser` (`apps/p2p-browser`), `SingboxGUIV0` (`apps/singbox-gui`), `QuickGit` (`apps/quick-git`), `EMQX Live Chat` (`apps/emqx-live-chat`), and shared packages (`control-protocol`, `emqx-chat-core`, `macos-ax-helper`, `runner-core`, `shared-config`).

---

## Part 2: Remaining / Uncompleted Backlog

### A. Ready to Finalize & Execute (Discussed This Session)
1. **Terminal `.app` Dual-Version Backup Build Script + `Unified Tray` `.backup.app` Routing**:
   - Interactive prompt before overwriting existing `.app`: `"Is current version of terminal working properly? [y/N]:"` (strict `y`/`Y` + Enter rotates current -> `*.backup.app`; blank/`N` preserves existing `*.backup.app`).
   - Caller-aware variant routing: `AGY Terminal.backup.app` passes caller identity (`AGY_CALLER_APP_NAME`) to `Unified Tray`, which routes `AX` startup to `AGY AX Terminal.backup.app` (mutually exclusive two-choose-one sharing `.agy-state`).
2. **Runner Single-Instance Lock & `.agy-state` Mismatch Guard**:
   - Add `flock` / lockfile guard in `agy_terminal` & `agy_ax_terminal` to prevent duplicate runner processes.
   - Add startup diagnostic warning in `Unified Tray` if `Runner Apps Dir` is unset while runners show `OfflineLaunch`.
3. **Mobile LAN Auto-Probe & Cross-Origin Passkey UX**:
   - Add tap-to-probe LAN on the `Offline` badge in mobile Pocket Deck (`probeNow(true)`).
   - Shorten `useLanAutoProbe` max backoff from `120s` to `30s`.
   - Avoid redundant Passkey prompt when switching back to LAN from an already-unlocked session (cross-origin trust handoff).
4. **`AGY Preview Instant` Rename & Long-Term Roadmap**:
   - Rename `AGY Preview Instant` to an accurate name reflecting its true role as an **API Caller / Invoker** (name TBD after 8h).
   - Long-term goal: support **web-side task dispatch** directly from the API Caller (currently manual flow suffices under lower concurrency).
5. **`p2p-remote-controller` Structural Split**:
   - Decouple chat UI from remote control / LAN sharing (noting Mac PWA cannot host LAN sharing); finalize split boundary with Mark after 8h.

### B. Long-Term / Independent Backlog Items
- **`#21`** `Singbox GUI` dual-core graceful handover (热切换).
- **`#35`** iOS APNs Push Notification Worker + unify `getgithubapptoken` / `agentoutputposter` / `push` into `agyWorker` monorepo.
- **`#38`** `sing-box` certificate-less traffic observation + credential sanitizer + auto-generated routing/anti-tracking rules.
- **`#42`** AI Studio `Inject Live Chat` AX path calibration (requires Mac `scan_window`).
- **`#43`** Standalone `EMQX Live Chat.app` desktop shell (`apps/emqx-live-chat` PWA exists; Tray slot reserved).
- **`#44`** LAN remote Mac voice dictation hotkey (`CGEventPost` + live input sync).
- **`#71`** Windows dual-controller architecture (`Admin Controller` + `Backup Controller`).

---

## Part 3: Chatting System, `agyWorker` & Instruction Optimization Report

### 1. `agyWorker` (`agentlivechatchannel`) Findings & Recommendations
1. **`POST /chat/send` missing/empty `channel` warning**:
   - Currently omitting `channel` silently routes to the global default channel without telling the caller. Keep HTTP 200 fallback so messages aren't lost, **but return `"warning": "No valid 'channel' parameter was provided; sent to global default..."`**.
2. **`/chat/send_and_poll` & `/chat/poll` leaking `resp/` topic echoes**:
   - Observed live right after `POST /cmd` `quick_git pull_ff`: the immediate next `/chat/send_and_poll` returned the `agy_p2p_v1/resp/...` payload instead of waiting for user chat. Filter `/chat/poll` & `/chat/send_and_poll` strictly to `chat/` topics (or exclude `resp/`).
3. **`cleanSession: false` QoS 1 redelivery & ordering**:
   - Both `/chat/poll` (DO side) and frontend clients experienced duplicate/out-of-order redeliveries after WebSocket `1005` drops. DO should track per-channel `(sender, sentAt)` dedup cursor and flush `PUBACK` before disconnect.
4. **`POST /channel/backup` schema clarity**:
   - `POST /channel/backup` requires `"payload"` (not `"data"`). Document `"payload"` clearly in `GET /help?topic=backup` and return a helpful hint if `"data"` is passed instead.
5. **`POST /cmd` heartbeat & argument ergonomics**:
   - Refresh `working`/`thinking` presence heartbeat during long `/cmd` waits, and document both top-level fields and `args` object in `GET /help?topic=cmd`.

### 2. User / System / Human Instruction & Workflow Optimizations
1. **Compaction Recovery Protocol (System/User Instruction)**:
   - On compaction resume, enforce a strict 2-step pattern: (1) immediately send a non-blocking `POST /chat/send` `[still working]` ACK informing the user of compaction, (2) restore `/workspace/.chat/*` state (`backlog.md`, `completed.md`, `arch_notes.md`) and verify `git log`, then send the substantive reply.
   - Maintain `/workspace/.chat/backlog.md` actively: whenever a task merges to `a2`, move it out of `backlog.md` into `completed.md` immediately so post-compaction status lists never include stale completed items.
2. **Sub-Agent Offloading & Chief Responsiveness**:
   - Keep Chief focused on live conversation, architecture discussion, and code review/merges; offload all implementation/testing to Alpha (`a2-alpha`).
   - Add a **sub-agent liveness check** (`git ls-remote` + `POST /chat/send` ping) if `/chat/poll` on the sub-agent channel times out — in this session Alpha pushed `4e5738da` silently before the poll returned.
3. **Cross-OS Rust `#[cfg]` & `state_dir` Discipline (DevKit `AGENTS.md` / Sub-Agent Prompt)**:
   - Linux sandbox `cargo check` does NOT type-check `#[cfg(target_os = "macos")]` blocks: require sub-agents to explicitly audit every `#[cfg(target_os = "macos")]` dependency (`tokio`, `objc2`, `objc2-app-kit`, `objc2-foundation`) against `Cargo.toml` before committing.
   - Enforce Single Source of Truth for `state_dir` (`AGY_STATE_DIR` propagation + lockfiles) to prevent `.app` bundle vs `tauri dev` split-brain failures.
4. **Spelling-Engine Tolerance & Non-Blocking Progress Updates**:
   - User uses a voice/spelling engine (`"back to code"` -> `↑ Back to Quote`, `"Dark tile"` -> `dockTile`, `"poor request"` -> `pull request`, `"H Y"` -> `AGY`, `"upload"` -> `offload`). Confirm intent naturally without pedantic correction.
   - Use `[still working]` non-blocking `POST /chat/send` before multi-step tool runs so the user never waits >15s without feedback.
5. **Client-Side Backup Verification Sync**:
   - Observed `unbackedCount` remaining >300 on the client even after `POST /channel/backup` succeeded with `sha256` (`49e04e...`). Verify client-side `backup_verified` MQTT listener matches `backupEventId` across reconnects.
