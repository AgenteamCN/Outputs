# Old Environment (`300cc1fb`) vs Current `origin/a2` (`e2bd2ee7`) Requirements Gap Report

Generated: 2026-11-26T07:00Z | Baseline: `origin/a2` @ `e2bd2ee7`

## 1. Already Landed / Resolved on `origin/a2` (`e2bd2ee7`)
- **EMQX Multi-Tab & Offline Session (`bugs.md #7`)**:
  - `05885d77` (`clientIdLease.ts` Web Locks + `localStorage` heartbeat lease, Primary `cleanSession=false`, Secondary `_tab_` `cleanSession=true`).
  - `524074b1` (`DEFAULT_EMQX_SESSION_EXPIRY_SEC = 86400`, 24h offline session expiry).
  - Client-side `mergeAndSort` dedup/sort by `sentAt`.
- **Live Chat UI & Animations (`bugs.md #8`)**:
  - `486db608` (`@agy/ui` `<ChatMessageBubble />` with per-message timestamps, bottom-left outside-bubble delivery status, top-right checkmark animation preview).
  - `b927719e` (4-stage countdown `R1/3..R3/3` -> `+MM:SS` -> `Expired (>1 h)` + `Pending Reply` fix).
  - `1118c9ab` & `e2bd2ee7` (3-mode Thinking Glow with independent params + `AppleCheckmarkCompletion` wired into live PC/Mobile chat).
- **Tray & Watchdog (`bugs.md #10`)**:
  - `git_watchdog.rs` now handles `ELIFECYCLE Command failed.` during `devWatch` restart grace window (`git_watchdog.rs:604`) and ignores pre-existing crashed presets (`13328ea`).
  - `apps/emqx-live-chat` standalone app created.
- **Raw SSE Export (`/cmd`)**:
  - `e2024eee` (`/cmd export_raw_sse` merged); `p2p-remote-controller` UI head/tail slicing in Task #67 (Fix).

## 2. Open Gaps & Remaining Backlog (Prioritized P0 -> P2)

### P0 (Immediate / High Impact)
1. **`agyWorker` Optimization (`DEC-02` — Locked, Pending Execution)**:
   - `1A`: DO hot cursor in memory + periodic flush.
   - `2A`: Per-`agentId` independent cursors (Chief / Secretary / Individual).
   - `3A`: `POST /agents/github-commit` (GitHub App Bot signed + `allowedBranches` check).
   - Plus: `chief_prev` isolated token, `ackedMsgIds` exact read receipts, `/help` completion, `todoSync` fix, `/chat/send` `connackCode===null` auto-retry, and unifying `getgithubapptoken`/`agentoutputposter` into `agyWorker`.
2. **macOS WKWebView `<a download>` / `blob:` TCC Prompt (`bugs.md #12`)**:
   - `URL.createObjectURL(blob)` still exists in `ThinkingGlowTunerModal.tsx:112`, `useLogExportActions.ts:100`, `CellularP2pSettingsCard.tsx:43`, and `PocketDeckOutputsTab.tsx:117`.
   - Fix: Replace with Rust Tauri command writing to App Support `exports/` + copy path / reveal in Finder.

### P1 (Core Feature & Parity Gaps)
3. **`apps/singbox-gui` 5 Missing Pieces (`#38` & `Req 17`)**:
   - `experimental.clash_api` runtime client.
   - Certificate-less TLS ClientHello SNI / domain observer.
   - Live proxy-stream header/body sanitizer.
   - Real MITM listener + dynamic leaf-cert signing + Rewrite Engine (`toggle_mitm_listener` currently stubbed/incomplete).
   - Request-stall detector + automatic Clash API node failover (`Req 17`).
4. **`AGY Preview Instant` Pre-Flight Racing (`active_spec.md`)**:
   - Safe `HEAD`/`OPTIONS` pre-flight racing + Keep-Alive/H2 connection pool reuse.
5. **`p2p-remote-controller` Dual Mode & PC/Mobile Parity**:
   - `ControllerModeToggle` / `NoControllerChatView` was removed in `403452b` — confirm with User whether to restore `With Controller` vs `No Controller` dual mode.
   - PC missing `In Progress`/`Compaction` badges & `QuickGitGraphView`; Mobile missing `Jump to latest` pill & `ActiveTodoPanel`.

### P2 (Cross-Platform & Polish)
6. **Windows Dual-Controller (`#71` & Task 25)**:
   - `Admin Controller` + Mac-to-Windows independent relay channel; cross-platform `keyring` 2560B split; `singbox-gui` Windows support.
7. **HarmonyOS NEXT (`hmos_next_design_summary.md`)**:
   - Native `ArkWeb` shell replacing PWA + `--with-hmos` / `--require-hmos` in `bootstrap.sh`/`doctor.mjs`.
8. **Remaining `bugs.md` & Tray Polish**:
   - `bugs.md #2`: PC/Mac notification chime intermittent delay/suppression.
   - `bugs.md #11`: `unified-tray` orange build icon occasionally turning blue prematurely.
   - `#44`: LAN remote Mac voice dictation hotkey (`CGEventPost`).
