# Container Crash Diagnostics & Pre-Heavy-Op Breadcrumb Note (Sanitized)

## 1. Silent Container Stalls Observed in This Session
- **Partner R (`ryan_v2`)**: Last active heartbeat `2026-11-26T16:55:05Z` (`status: working`), assigned to `p2p-browser` redactor/CDP implementation. No error message or exit status emitted prior to disconnect.
- **Partner O (`orbit_5100a26f`)**: Last active heartbeat `2026-11-26T16:55:58Z` (`status: working`), assigned to `singbox-gui` Enhanced Mode cross-review (`cargo test` / `pnpm` verification). No error message or exit status emitted prior to disconnect.
- **Prior Session Reference (`scott`)**: Previously hit `error loading container: file does not exist` during heavy Rust `cargo test` / snapshot persistence.

## 2. Suggested Export / Post-Mortem Checklist (User Side)
When inspecting host/container platform logs around **`2026-11-26 16:54:00Z - 16:56:30Z`**, check:
1. Whether both containers were simultaneously running heavy Rust compilation (`cargo test` / `rustc` memory spike / OOM kill) or hitting the 100 MB OverlayFS / `/tmp` tmpfs budget.
2. Whether a specific tool invocation (`cargo test` without `--locked`, large `git` fetch, or parallel loopback test) immediately preceded the container termination event.
3. Any container runtime exit codes (`137` OOMKilled, `error loading container: file does not exist`, or snapshot-restore failure).

## 3. Mitigation: One-Liner Pre-Heavy-Op Breadcrumb (Next-Round Partner Rule)
Before executing any heavy/high-risk command (`cargo test`, `pnpm build`, `vitest`, or large `git` operations), Partners emit a single-line breadcrumb to their channel (e.g., `[breadcrumb] running <command>`) so that even if the container terminates mid-command without a final error, the exact triggering command is immediately known.
