# Partner Alex (`alex_3d589d0a`) — Final Mission Output

## Mission: `packages/runner-core/src/tauri_lifecycle.rs` & macOS Dock / `SIGUSR1` Root-Cause Investigation (Read-Only)

### 1. `SIGUSR1` Main-Thread Crash (`quick-git-v0` on macOS 15.7.5)
- **Root Cause**:
  In `packages/runner-core/src/tauri_lifecycle.rs` (`spawn_sigusr1_unhide_listener`, lines 51-72), the `SIGUSR1` signal listener runs inside `tauri::async_runtime::spawn(async move { ... })` on a `tokio-rt-worker` background thread and calls `show_and_focus_main_window(&app_handle, on_macos_show)` directly on that worker thread.
  Inside `show_and_focus_main_window` (lines 28-49), only `on_macos_show` is dispatched via `app_handle.run_on_main_thread(cb)`, while `move_window_to_active_space(&win)` (`ns_win.collectionBehavior()` / `ns_win.setCollectionBehavior(...)` via `objc2_app_kit::NSWindow`) and `win.show()` / `win.unminimize()` / `win.set_focus()` execute directly on the `tokio-rt-worker` thread.
  On macOS 15.7.5 (`-[NSWMWindowCoordinator performTransactionUsingBlock:]`), calling `NSWindow.setCollectionBehavior:` off the main thread triggers `EXC_BREAKPOINT / Must only be used from the main thread`.
- **Proposed Fix**:
  Dispatch the entire `show_and_focus_main_window` execution onto the macOS main thread via `app_handle.run_on_main_thread(move || { ... })` (either inside `spawn_sigusr1_unhide_listener` or inside `show_and_focus_main_window` itself), so `set_activation_policy`, `on_macos_show`, `move_window_to_active_space`, and `win.show()`/`unminimize()`/`set_focus()` all run strictly on the main thread.

### 2. macOS Dock Icon Persistence on Window Close (`apps/singbox-gui` & `apps/p2p-browser`)
- **Code Inspected (`origin/a2` at `4f2eaa359208283f01dff62d7bddbde119d913bf`)**:
  - `apps/singbox-gui/src-tauri/src/lib.rs:162`: unconditionally calls `agy_runner_core::tauri_lifecycle::hide_window_to_accessory(window, api)` on `WindowEvent::CloseRequested`.
  - `apps/p2p-browser/src-tauri/src/lib.rs:81`: calls `agy_runner_core::tauri_lifecycle::handle_tray_managed_window_event(window, event)`, which calls `hide_window_to_accessory(window, api)` when `AGY_TRAY_MANAGED=1` and `window.label() == "main"`.
  - `apps/unified-tray/src-tauri/src/main.rs:126` (`with_tray_managed_env`) and `packages/runner-core/src/process_manager.rs:200` both inject `AGY_TRAY_MANAGED=1` when launching `singbox-gui` and `p2p-browser`.
- **Root Causes**:
  1. **Primary (`packages/runner-core/src/tauri_lifecycle.rs:77-96`, `hide_window_to_accessory`)**:
     `hide_window_to_accessory` calls `api.prevent_close()`, `window.hide()`, and `window.app_handle().set_activation_policy(tauri::ActivationPolicy::Accessory)` **synchronously inside `WindowEvent::CloseRequested` (`windowShouldClose:`)** while the app is still the active frontmost process and before AppKit finishes `orderOut:` and unwinds `windowShouldClose:`. On macOS, calling `setActivationPolicy(.accessory)` synchronously during `windowShouldClose:` frequently leaves the Dock icon visible. (By contrast, `unified-tray` has `LSUIElement=true` in `Info.plist` and a 3s background loop re-applying `apply_macos_dock_visibility`.)
  2. **Secondary (`apps/singbox-gui/src-tauri/src/lib.rs` & `tauri.conf.json`)**:
     - `singbox-gui/src-tauri/tauri.conf.json` sets `"visible": false`, but `setup()` never calls `set_activation_policy(Accessory)` on startup, so `singbox-gui` starts with a Dock icon even when its window is hidden.
     - `singbox-gui` tray menu `"show_main"` (`lib.rs:113`) calls raw `win.show()` instead of `show_and_focus_main_window(app_handle, None)`, so it would not restore `ActivationPolicy::Regular` when reopened from its own tray menu.
  3. **Note on `apps/p2p-browser`**:
     When launched standalone (outside `unified-tray`, without `AGY_TRAY_MANAGED=1`), `is_tray_managed()` is `false`, so `handle_tray_managed_window_event` does not run.
- **Proposed Fix**:
  1. In `packages/runner-core/src/tauri_lifecycle.rs` (`hide_window_to_accessory`), on `#[cfg(target_os = "macos")]`, defer `set_activation_policy(tauri::ActivationPolicy::Accessory)` to the next main-thread runloop turn via `app_handle.run_on_main_thread(move || { ... })` after `api.prevent_close()` and `window.hide()`.
  2. In `apps/singbox-gui/src-tauri/src/lib.rs`, set `ActivationPolicy::Accessory` in `setup()` on macOS when `visible: false`, and use `show_and_focus_main_window(app_handle, None)` in tray menu `"show_main"`.
