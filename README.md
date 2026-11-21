# RocketCompanion MVP-1 Fixes Report

**Base Commit**: `0deab37c9b218cdf43bd127f673d22d6cd2e2a52` (`feat/mvp1-rust-core-split-v2`)  
**Pushed Commit**: `3c5c0308380b2af772fd6745a58e9df1dc5c0df8` (`verified: true`, `web-flow`)  
**Branch**: `feat/mvp1-rust-core-split-v2`

---

## 1. `localhost:3000` MVP-1 Testbed Quick Entry
- **Root Cause**: `mountMvp1ShortcutButton()` had been added to `src/main.ts` (which serves `:3001` HTTPS and `:3003` HTTP), while `src/entry/EntryHubApp.tsx` (the launchpad served on `localhost:3000`) had no quick entry to `mvp1.html`.
- **Fix**:
  - Removed `mountMvp1ShortcutButton()` from `src/main.ts` (preserving `/?mvp1=1` direct boot support).
  - Added `MVP-1 Rust Testbed (mvp1.html)` quick-launch buttons to `src/entry/EntryHubApp.tsx` on both Card 1 (`https://<host>:3001/mvp1.html`) and Card 3 (`http://<host>:3003/mvp1.html`).

## 2. `Q+D` / `A+E` Stall (无效翻滚)
- **Root Cause**: In `crates/rocketsim/src/sim/car/base.rs` (`Car::update_double_jump_or_flip`), when `deadzone_dodge_dir` evaluated to `Vec3A::ZERO` (as in a `Q+D` or `A+E` stall where `yaw + roll == 0`), `self.state.flip_rel_torque` was not cleared. If the car had performed any prior directional flip, `flip_rel_torque` retained the old flip's torque and applied it during the subsequent stall.
- **Fix**:
  - Explicitly reset `self.state.flip_rel_torque = Vec3A::ZERO` when grounded (`is_on_ground`) and in the `else` branch when `deadzone_dodge_dir` is zero.
  - Added regression test `stall_after_prior_flip_clears_flip_torque` in `crates/rc_core/src/lib.rs`.
  - Rebuilt `src/mvp1/rc_core.wasm` via `pnpm run build:rc-core`.

## 3. Boost Pads Partially Disappearing / Not Disappearing on Collect
- **Root Cause**: `crates/rocketsim/src/sim/boost_pad/boost_pad_grid.rs` (`BoostPadGrid::new`) sorts all 34 Soccar boost pads by `(pos.y, pos.x)` (RLBot/RLGym ordering). However, `src/mvp1/ArenaRenderer.ts` constructed `boostPadMeshes` in `[28 small, 6 big]` unsorted order and checked `i >= 28` for big pads. Only the first 3 pads (`y = -4240`, `-4184`) happened to align by coincidence, causing all subsequent pads to map to the wrong bitmask index.
- **Fix**:
  - Sorted `STANDARD_BOOST_PADS` in `src/mvp1/ArenaRenderer.ts` by `(loc[1], loc[0])` to match `BoostPadGrid::new`, and used `STANDARD_BOOST_PADS[i].isBig` in `render()`.
  - Added `soccar_boost_pads_are_sorted_by_y_then_x` test in `crates/rc_core/src/lib.rs`.

## Verification
- `pnpm run build:rc-core` — succeeded, updated `src/mvp1/rc_core.wasm`.
- Node WASM harness on `src/mvp1/rc_core.wasm` — verified `Q+D` stall after a prior front-flip no longer applies stale flip torque.
- `pnpm run check` (`radar` + `tsc --noEmit`) — `STATUS: GREEN`, 1397/1397 TS files, 0 errors.
