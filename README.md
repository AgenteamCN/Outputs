# Fix Tauri `keep_awake` Compilation & Green Unread Channel Indicator

## 1. Rust / Tauri `keep_awake` Compile Fix
- **Root cause**: `pub mod keep_awake;` was missing in `lib.rs` and `#[tauri::command]` macros were not placed in `commands.rs`.
- **Fix**: Kept `caffeinate` process management in `apps/p2p-remote-controller/src-tauri/src/keep_awake.rs`, added `#[tauri::command]` wrappers (`get_keep_awake_status`, `set_keep_awake`) in `commands.rs`, and declared `pub mod keep_awake;` + imported commands from `commands` in `lib.rs`.

## 2. Green Highlight for Inactive Unread / Unhandled EMQX Channels
- Updated `apps/p2p-remote-controller/src/components/EmqxBridgePanel.tsx` and `EmqxBridgeChannelBar.tsx` to track `lastViewedCountByChannel` and `pendingApprovals` purely via `useMemo`.
- Inactive channels (`default` or `#ch_*`) with unread inbound chat messages or pending approvals now render with a green highlight (`#1a7f37`) until selected.

## 3. Verification
- `pnpm radar -- --audit`: PASS
- `pnpm p2p:check`: PASS
- `pnpm --filter p2p-remote-controller test`: 7 files, 27 tests PASS
