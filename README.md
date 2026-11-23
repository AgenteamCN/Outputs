# Rust/Tauri Cloud Sandbox Toolchain Completion & Verification Report

- **Repository**: `Antigravi-ty/agyDevKit` (`a2` @ `60fa9a8`)
- **Implemented by**: `atlas` (`a2-env-rust-ergonomics` @ `cce10b4`)
- **Reviewed & Verified by**: Chief Agent (`60fa9a8`, Verified Bot Commit)

---

## 1. Problems Resolved

1. **Cross-Shell `PATH` & `CARGO_TARGET_DIR` Loss**:
   - Previously, `scripts/env/bootstrap.sh --with-rust` installed Rust into `/tmp/agy-cache/cargo-home` with `--no-modify-path` and exported `CARGO_TARGET_DIR` only inside its own subshell.
   - Because each sandbox tool call starts a fresh shell process, subsequent `cargo check` calls either failed with `cargo: command not found` or wrote hundreds of MB of `target/` onto `/workspace` OverlayFS (causing GCS snapshot upload timeouts).
2. **False-Negative `Rust/Tauri` Status in `checkEnv.mjs` & `doctor.mjs`**:
   - `envWithCargo` only set `PATH`, omitting `CARGO_HOME` and `RUSTUP_HOME`, causing `rustup` shims to fail with `[WARN] Rust/Tauri (optional) not installed` even right after `bootstrap.sh --with-rust`.
3. **No Early Guard for Stray `target/` Directories**:
   - Nothing warned when a `target/` directory was accidentally created under `packages/` or `apps/` on Linux OverlayFS.

---

## 2. Changes Implemented (`60fa9a8`)

1. **`scripts/env/bootstrap.sh`**:
   - On Linux with `WITH_RUST=1` and writable `/usr/local/bin`, writes idempotent `# agy-env-wrapper` executable wrappers at `/usr/local/bin/{cargo,rustc,rustup}` that export:
     - `CARGO_HOME="${CARGO_HOME:-/tmp/agy-cache/cargo-home}"`
     - `RUSTUP_HOME="${RUSTUP_HOME:-/tmp/agy-cache/rustup-home}"`
     - `CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-/tmp/agy-cache/cargo-target}"`
     and `exec "${CARGO_HOME}/bin/<tool>" "$@"`.
   - Also detects stale wrappers after `/tmp` resets (`! -x "${CARGO_HOME}/bin/cargo"`) and triggers reinstall automatically.
2. **`scripts/env/tmpfsGuard.mjs`**:
   - Added `ensureRustWrappers(cacheRoot)` and invoked it inside `autoRestoreTmpfs` when `{ withRust: true }`.
3. **`scripts/env/tmpfsLayout.mjs`, `checkEnv.mjs`, `doctor.mjs`**:
   - Fixed `envWithCargo` to pass `CARGO_HOME`, `RUSTUP_HOME`, and `CARGO_TARGET_DIR`.
   - Added `findStrayCargoTargetDirs()` check on Linux to warn/fail if any `target/` directory exists inside `packages/` or `apps/` on OverlayFS.

---

## 3. Independent Verification Results (Clean Environment)

- `bash -n scripts/env/bootstrap.sh`: **PASS**
- `bash scripts/env/bootstrap.sh --with-rust`: **PASS**
- `pnpm run env:check`: **PASS**
  - `OverlayFS size: 10 MB (baseline: 9 MB, budget: <= 100 MB)`
  - `Rust/Tauri (optional): cargo 1.100.0 (3acc79c0c 2026-11-05) + cc`
  - `Cargo target/ (Linux): clean (offloaded to tmpfs)`
- `node scripts/env/doctor.mjs --require-rust`: **PASS** (all required checks passed)
- `env -u CARGO_HOME -u RUSTUP_HOME -u CARGO_TARGET_DIR cargo check --manifest-path packages/runner-core/Cargo.toml`: **PASS** (finished in `12.12s` into `/tmp/agy-cache/cargo-target`, zero OverlayFS growth)
- TS / Vitest suites (`root`, `p2p-remote-controller`, `control-protocol`, `unified-tray`): **PASS**
