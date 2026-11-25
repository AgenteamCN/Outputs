# Session Checkpoint (2026-11-25)

## Repository State
- `Antigravi-ty/agyDevKit` (`a2` HEAD): `d85aa288540b1b28a1079991c5a7c8a90a599610`

## Completed This Segment
1. **AI Studio #42 AX Calibration (`windowId: 14576`)**:
   - Narrow screen: `Toggle run settings panel` (`[1,0,4]`) -> `System instructions` (`[2,0,21]`) -> `AXTextArea "System instructions"` (`[1,9]`) `set_value` (`Updated AXValue (verified)`) -> `Close panel` (`[1,2]`).
   - Wide screen: Left sidebar expanded, `Run settings` already open (`[4,0,19] AXButton "System instructions"`), same 13-element modal (`[1,9] AXTextArea`, `[1,2] AXButton "Close panel"`).
   - Conclusion: Existing `aiStudioDispatchSteps.ts` role+title matching is compatible with both narrow and wide layouts.
2. **Alex — `Working (Not listening)` State-Machine Audit (`[done]`)**:
   - Confirmed inverted icon (`EmqxChatSection.tsx`, `PocketDeckBridgeTab.tsx` render `<ThinkingCheckmark />` during `state === "working"`), no sticky post-working checkmark before next real text, `kind` filtering mismatch (`in_progress` vs `command`), and `ChatStreamPanel.tsx` UI parity gap.
3. **Alex — `singbox-gui` / `p2p-browser` Dock Icon & `quick-git-v0` Crash Investigation (`[done]`)**:
   - Root cause #1 (`packages/runner-core/src/tauri_lifecycle.rs:77-96`): `hide_window_to_accessory` calls `set_activation_policy(Accessory)` synchronously inside `CloseRequested` (`windowShouldClose:`) before `orderOut:` unwinds. Fix: defer via `app_handle.run_on_main_thread(...)`.
   - Root cause #2 (`apps/singbox-gui/src-tauri/src/lib.rs`): `visible: false` startup lacks `ActivationPolicy::Accessory` in `setup()`, and tray `show_main` uses raw `win.show()` instead of `show_and_focus_main_window`.
   - Root cause #3 (`quick-git-v0` macOS 15.7.5 crash log): `spawn_sigusr1_unhide_listener` calls `show_and_focus_main_window` -> `move_window_to_active_space` -> `ns_win.setCollectionBehavior(...)` from `tokio-rt-worker` off the main thread (`EXC_BREAKPOINT / Must only be used from the main thread`). Fix: dispatch via `app_handle.run_on_main_thread(...)`.
4. **Scott — `sing-box` HarmonyOS NEXT Research (`[done]`)**:
   - Bundled research (`hmos_singbox_research.md`) and externalized core research (`hmos_singbox_external_core.md`): `filesDir` `dlopen`/`NativeChildProcess` is blocked by mandatory code signing (`hapsigner` + `fs-verity` + `ownerId`). Cleanest GPLv3 boundary for closed-source host app is a separate GPLv3 companion `.hap` using `VpnExtensionAbility`; for single-`.hap` UX without GPLv3 risk, use MPL-2.0 (`Xray-core`) or MIT (`shadowsocks-rust` / `v2ray-core`).

## Pending / In Progress
- **Ryan (`a2-fix-working-checkmark-lifecycle`)**: Implementing `Working` state-machine & UI parity fix (not yet pushed).
- **Alex (`tauri_lifecycle.rs` + `singbox-gui`)**: Ready to implement once Human confirms go-ahead (with Scott Cross-Review).
- **Durable Objects (`agyWorker`)**: Ready to discuss `channel_message_buffer` SQLite -> in-memory `Map` with Human.
