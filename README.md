# Incident Report: Sandbox `/workspace` Reset & OverlayFS Snapshot Upload Timeout

- **Timestamp**: 2026-11-23T13:10Z – 13:30Z (UTC)
- **Repo**: `Antigravi-ty/agyDevKit` (branch `a2` @ `4f6a0ea`), `Antigravi-ty/agyWorker` (branch `main` @ `d29118e`)
- **Severity**: Low (zero code loss; ephemeral `/workspace/.chat` local logs lost)

---

## 1. What Happened

After context compaction, the Chief Agent resumed and ran `cargo check` inside `/workspace/agyPreviewInstant` without overriding `CARGO_TARGET_DIR`.

1. **First symptom**: `execute_bash` hit an upstream RPC timeout (`CORTEX_STEP_TYPE_RUN_COMMAND: generic::deadline_exceeded: Environment request failed for /code: upstream request timeout`) while compiling all 4 Rust/Tauri crates sequentially in a single command.
2. **Root cause of snapshot failure**: Even when running `cargo check` on a single crate (`packages/runner-core`), Cargo defaulted to writing build artifacts into `<crate>/target/` inside `/workspace/agyPreviewInstant/...` (which sits on the snapshotted OverlayFS root filesystem).
   - Tauri/Rust `target/` directories quickly grow to hundreds of MB / GBs.
   - At the end of the tool call, the sandbox runtime attempted to snapshot and upload the OverlayFS diff to GCS, failing with:
     ```
     failed to persist changes in filesystem snapshot: failed to upload snapshot .../snapshots/...tar:
     snapshot upload to GCS failed: failed to upload blob to GCS: context deadline exceeded
     ```
3. **Container reset**: Following the failed snapshot upload / upstream timeout, the sandbox fell back to a fresh container at `2026-11-23T13:10Z`:
   - `/workspace/agyPreviewInstant` was empty.
   - `/tmp` (tmpfs) was reset.
   - Local non-tracked files under `/workspace/.chat/` (local chat transcripts and `/workspace/.chat/cf_env.sh`) were lost.

---

## 2. Impact Assessment

- **Source Code**: **Zero loss.** All commits had already been pushed via Verified Bot Commit to GitHub:
  - `Antigravi-ty/agyDevKit` (`a2` @ `4f6a0ea`)
  - `Antigravi-ty/agyWorker` (`main` @ `d29118e`, deployed live at `Version ID: 228743d7-6aed-44b1-a1d5-b04e80bf321d`)
- **Local Ephemeral State**: Lost `/workspace/.chat/*` local chat logs and `/tmp/agy-cache/*`.

---

## 3. Recovery & Verification

1. **Re-cloned repository**: Cloned `Antigravi-ty/agyDevKit` (`a2` @ `4f6a0ea`) into `/workspace/agyPreviewInstant`.
2. **Restored toolchain**: Ran `bash scripts/env/bootstrap.sh --with-rust` (installed `pnpm 10.28.2`, `node_modules` in tmpfs, `cargo 1.100.0` + `cc` in `/tmp/agy-cache/cargo-home`).
3. **Verified Rust builds with `CARGO_TARGET_DIR` in `/tmp`**:
   - Cleaned any workspace `target/` directories.
   - Exported `CARGO_TARGET_DIR=/tmp/agy-cache/cargo-target` (ephemeral `tmpfs`, excluded from OverlayFS snapshots).
   - All 4 crates passed `cargo check` with 0 errors / 0 warnings:
     - `packages/runner-core` (11.73s)
     - `apps/unified-tray/src-tauri` (2m04s cold build, cached in `/tmp` thereafter)
     - `apps/agy-terminal/src-tauri` (10.40s)
     - `apps/agy-ax-terminal/src-tauri` (2.20s)
4. **OverlayFS Budget Check**:
   - `pnpm run env:check` confirmed **OverlayFS size: 10 MB (baseline: 9 MB, budget: <= 100 MB)**.

---

## 4. Preventive Measures & Recommendations

1. **Mandatory `CARGO_TARGET_DIR` for all agents**:
   - Chief Agent and `coder_1` (via `ch_code_a2_worker1`) must ALWAYS set:
     ```bash
     export CARGO_HOME=/tmp/agy-cache/cargo-home
     export RUSTUP_HOME=/tmp/agy-cache/rustup-home
     export CARGO_TARGET_DIR=/tmp/agy-cache/cargo-target
     ```
   - Consider adding `.cargo/config.toml` or updating `scripts/env/bootstrap.sh` / `scripts/env/checkEnv.mjs` in `agyDevKit` to enforce/warn if `<repo>/**/target/` exists on OverlayFS, or default `CARGO_TARGET_DIR` in env scripts.
2. **External / Cloud Persistence for Chat Logs**:
   - Relying solely on `/workspace/.chat/*` is vulnerable if a sandbox snapshot fails or resets.
   - Future improvement: add an agent-accessible cloud persistence endpoint (e.g., Worker + R2/KV or DO storage) for `.chat` transcripts and session state.
