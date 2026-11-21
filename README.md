# RocketCompanion MVP-1 Fixes Summary

Branch: `feat/mvp1-rust-core-split-v2`
Base commit: `0deab37c9b218cdf43bd127f673d22d6cd2e2a52`
Latest verified commits:
- `3c5c0308380b2af772fd6745a58e9df1dc5c0df8` — `fix(mvp1): move testbed entry to :3000 hub, fix stall torque & boost pad order`
- `bfc8a8825076506e79cc9f98e1cf024ff5ac1ac4` — `feat(mvp1): add KBM mouse boost/jump bindings and JUMP/FLIP HUD flags`

## 1. `localhost:3000` MVP-1 Testbed Quick Entry
- Removed the temporary floating `MVP-1 Rust Testbed →` button from `src/main.ts` (ports `3001` / `3003`).
- Added the MVP-1 Testbed (`mvp1.html`) quick-entry UI on `localhost:3000` (`src/entry/EntryHubApp.tsx`).

## 2. `Q+D` / `A+E` Stall (无效翻滚)
- Side-by-side comparison of `src/physics/core.wasm` (Alpha C++ RocketSim) vs `src/mvp1/rc_core.wasm` (Rust RocketSim):
  - `crates/rocketsim/src/sim/car/base.rs` previously did not reset `self.state.flip_rel_torque` on ground or on a zero-dodge-dir stall (`Q+D` / `A+E`), causing stale flip torque from an earlier directional flip to persist into a subsequent stall. Resetting `flip_rel_torque = Vec3A::ZERO` on ground and on zero-dodge-dir stall (`3c5c030`) fixes this.
  - Added Mouse Left (`boost`) and Mouse Right (`jump`, with canvas `contextmenu` prevented) in `src/mvp1/InputController.ts` to match standard RocketCompanion KBM bindings (`inputBindingValidators.ts`), and added `JUMP` / `FLIP` state flags to the MVP-1 HUD in `src/mvp1/Mvp1App.ts` (`bfc8a88`).

## 3. Boost Pad Collection Visual Desync
- `crates/rocketsim/src/sim/boost_pad/boost_pad_grid.rs` (`BoostPadGrid::new`) sorts all 34 pads ascending by `(pos.y, pos.x)`.
- `src/mvp1/ArenaRenderer.ts` previously appended 28 small pads then 6 big pads unsorted, shifting indices for almost every pad after `y = -4096` so collecting one pad dimmed a different pad elsewhere on the field.
- Sorted `STANDARD_BOOST_PADS` by `(y, x)` in `src/mvp1/ArenaRenderer.ts` to match `BoostPadGrid::new`.

## Verification
- `pnpm run check` (`radar` + `tsc --noEmit`): **PASS** (1397/1397 TS files, 0 errors).
- `pnpm run env:check`: **PASS** (OverlayFS size: 30 MB / 100 MB budget).
