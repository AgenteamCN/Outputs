# Deferred / Future Backlog Report (后续待办清单)

> Generated at end of Live Chat session (2026-11-23). Items listed here were explicitly deferred and **not** implemented in this session.

## 1. Remaining Monorepo Absorptions (逐个吸收 + 逐个真机验证)
- **P2PBrowser**: Absorb into `apps/p2p-browser` with dedicated Vite port `1424`, verify `cargo check` + local SOCKS5 listen in sandbox, then Mac real-device WebView/proxy verification.
- **SingboxGUIV0**: Absorb into `apps/singbox-gui` with dedicated Vite port `1425`; requires `sing-box` binary path resolution and root/TUN/Network Extension permission handling on Mac.
- **Mac Real-Device Verification**: User to verify newly absorbed `apps/quick-git` (`1423`) and updated `Unified Tray` (`Runner Apps Dir` config, native `●/○` submenu, `ax-terminal` auto-launch) on Mac.

## 2. `Unified Tray` & `p2p-remote-controller` Follow-ups
- Migrate `Shortcuts` (`EMQX CN (Private)` / `EMQX Global (Private)`) from legacy `p2p-remote-controller` tray into `Unified Tray` or main UI if still needed.
- Finalize any UX polish for `Runner Apps Dir` first-run onboarding on Mac.

## 3. `agyWorker` & Live Chat Infrastructure
- **GitHub Token Proxy & Cascading Revocation**: Implement `POST /github/token` proxy with DO tracking, cascading `DELETE /installation/token` on sub-agent revocation, and irreversible salted-hash sub-channels (resolving the 6 open `[?]` design questions from earlier report `2dc9d7f`).
- **MQTT `/chat/send` Reliability**: Fix 5s timeout false-positive `ok: true` when `pubackedCount === 0` (report `820b2e1`).
- **EMQX Active Dead-Socket Probe**: Add active `PINGREQ -> PINGRESP` deadline probe (e.g. 5s timeout + immediate reconnect) or Dual EMQX failover to eliminate silent `Close code 1005` drops.
- **External Cloud `.chat` Persistence**: Move `.chat/*` session memory to an agent-accessible external cloud store so sandbox `/workspace` resets never wipe conversation history.

## 4. Multi-Agent Coordination & Governance
- Finalize multi-agent naming conventions (`atlas`, `orion`, `vega`, `nova` vs `atlas-1`/`atlas-2`).
- Implement branch-locking / file-ownership rules for concurrent sub-agents.
- Define cross-sub-agent communication policy (direct vs hub-and-spoke through Chief, channel ACL boundaries).

## 5. Product & Feature Backlog
- **PC Wide-Screen (16:9 / 3:2) + Chief-Pushed Todo Board**: Responsive wide-screen layout paired with a Chief-pushed Todo/Backlog pane (completed items use `strikethrough` first and are only deleted after explicit user confirmation).
- **AI Studio `Inject Live Chat` AX Path Calibration**: Live Safari `scan_window` calibration.
- **Standalone EMQX Live Chat `.app`**: Fault-isolated control plane (reserved `emqx-live-chat` slot already added in `Unified Tray`).
- **LAN-Only Remote Mac Dictation Hotkey**: `CGEventPost` + real-time synced input box.
- **Cross-Platform Expansion**: 5-in-1 monorepo + Windows CI + macOS->Windows H.265 ROI + custom MITM.

## 6. Operational Guardrail
- **Repo Redirect Check**: If a GitHub repository redirect is detected during `git clone`, the agent should immediately question/remind the user to verify that their `user initial input` points to the updated repository URL.
