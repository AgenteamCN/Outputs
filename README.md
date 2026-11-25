# OverlayFS Storage & `wt-working-checkmark` Investigation Report

- **Agent**: Ryan (`ryan_7f3a1c20`, channel `ch_ryan_7f3a1c20`)
- **Date**: 2026-11-25T14:54Z
- **Observed OverlayFS Limit**: 100 MB
- **Measured OverlayFS Usage**: 756 MB used on `/` (`df -h /`), ~669 MB in `/workspace` + `/root`
- **Action Taken**: Read-only investigation only (0 files deleted or modified)

---

## 1. Executive Summary

Over **90% (603 MB)** of the persistent OverlayFS usage comes from **two duplicate `pnpm` `node_modules` trees** under `/workspace`:
1. `/workspace/agyDevKit/node_modules` (**306 MB**) — stale/broken `pnpm` installation from earlier sessions (with broken `../../wt-quote/node_modules/.pnpm/...` symlinks).
2. `/workspace/wt-working-checkmark/node_modules` (**297 MB**) — created during **Task 39** after `/tmp` was wiped mid-task on container snapshot restore, prompting a fallback `git worktree` + `pnpm install` directly under `/workspace`.

Additionally, the container prompt / compaction summary did **not** include the repository's environment-restore instructions (`scripts/env/bootstrap.sh`, `node scripts/env/doctor.mjs --auto-restore`, `pnpm run env:check`), which are designed to keep `node_modules` on ephemeral `tmpfs` (`/tmp`, 7.9 GB) rather than the persistent OverlayFS.

---

## 2. Exact Storage Breakdown (`df -h` / `du -sh`)

| Path | Size | Notes |
|---|---:|---|
| `df -h /` (total overlay used) | **756M** | `/tmp` is separate `tmpfs` (2.6M used / 7.9G total) |
| `/workspace/agyDevKit` | **329M** | Main repo checkout |
| - `/workspace/agyDevKit/node_modules` | 306M | Stale `pnpm` tree (contains broken `../../wt-quote/node_modules/.pnpm` symlinks) |
| - `/workspace/agyDevKit/apps` | 11M | App source files |
| - `/workspace/agyDevKit/.git` | 9.2M | Git objects & refs |
| - `/workspace/agyDevKit/packages` | 297K | Shared packages source |
| `/workspace/wt-working-checkmark` | **310M** | Task 39 `git worktree` (branch `a2-fix-working-checkmark-lifecycle`) |
| - `/workspace/wt-working-checkmark/node_modules` | 297M | Fresh `pnpm@9.15.9` install run during Task 39 |
| - `/workspace/wt-working-checkmark/apps` | 9.4M | App source files |
| - `/workspace/wt-working-checkmark/packages` | 353K | Shared packages source |
| `/root/.npm` | **29M** | Cache created by `npx -y pnpm@9` |
| `/root/.cache` | **1.9M** | CLI/tool caches |
| `/workspace/.chat` | **209K** | Persistent agent task notes & design docs |
| System (`/usr`, `/var`, `/etc`) | **8.6M** | Base OS / container delta |

---

## 3. What `wt-working-checkmark` Is & Original Task 39 Timeline

### 3.1 Original Task 39
- **Task**: Chief assigned Task 39 on `ch_ryan_7f3a1c20` at `2026-11-25T13:48:20Z` to implement the `Working (Not listening)` spinner & sticky `Working (Done)` checkmark lifecycle fix on a new branch `a2-fix-working-checkmark-lifecycle` from `origin/a2` (`d85aa288540b1b28a1079991c5a7c8a90a599610`).
- **Scope**:
  - `packages/emqx-chat-core/src/agentStatus.ts` & `src/index.test.ts`
  - `apps/p2p-remote-controller/src/components/EmqxChatSection.tsx`, `EmqxChatSection.test.tsx`, `PocketDeckBridgeTab.tsx`
  - `apps/emqx-live-chat/src/components/ChatStreamPanel.tsx`, `ChatStreamPanel.test.tsx`
- **Outcome**: Implemented, verified (`emqx-chat-core` 31/31, `emqx-live-chat` 14/14, `p2p-remote-controller` 12/12, `check:gates` 883 files / 0 violations), and pushed via Verified Bot Commit `c201b86228d5153a8b3d58a648e28466d6294340` to `origin/a2-fix-working-checkmark-lifecycle`.

### 3.2 Why `/workspace/wt-working-checkmark` Ended Up on OverlayFS
1. **Initial worktree in `/tmp`**: To keep `/workspace/agyDevKit` untouched (which had 4 leftover modified files from an earlier branch), Task 39 initially created a `git worktree` at `/tmp/wt-working-checkmark`.
2. **Broken `pnpm` / `node_modules` in `/workspace/agyDevKit`**: `pnpm` was not in `PATH`, and `/workspace/agyDevKit/node_modules/typescript` pointed to `../../wt-quote/node_modules/.pnpm/...` (a previous `/tmp/wt-quote` worktree that no longer existed after `/tmp` reset).
3. **`/tmp` reset mid-task**: During Task 39, the container's ephemeral `/tmp` (`tmpfs`) reset, wiping `/tmp/wt-working-checkmark`.
4. **Move to `/workspace/wt-working-checkmark`**: To avoid losing in-progress edits on another `/tmp` reset, the worktree was recreated at `/workspace/wt-working-checkmark` (persistent `/workspace`), and dependencies were installed via `npx -y pnpm@9 install --frozen-lockfile`. This added **310 MB** (`297 MB` `node_modules`) to `/workspace` and **29 MB** to `/root/.npm`.

---

## 4. Missing Environment Restore Instructions (Root Cause of Process Gap)

The agent's system prompt and post-compaction summary contained:
- Generic cloud sandbox instructions (`/workspace` persists, `/tmp` is ephemeral `tmpfs`).
- Live Chat Worker endpoints, GitHub token endpoint, Verified Bot Commit rules, and 10+3 Wait Policy.

They did **not** contain the repository's environment-restore / OverlayFS-budget instructions:
- `bash scripts/env/bootstrap.sh`
- `node scripts/env/doctor.mjs --auto-restore`
- `pnpm run env:check` (OverlayFS <= 100 MB check)

Without those instructions, the agent had no awareness of the 100 MB OverlayFS budget or the `doctor.mjs --auto-restore` `tmpfs` mount mechanism.

---

## 5. Recommended Remediation (Not Yet Executed)

Once approved by Chief Agent / Mark:
1. Remove the completed Task 39 worktree:
   - `git -C /workspace/agyDevKit worktree remove --force /workspace/wt-working-checkmark` (frees **310 MB**; commit `c201b862` is already pushed on `origin/a2-fix-working-checkmark-lifecycle`).
2. Remove stale OverlayFS `node_modules` and `npx` cache:
   - `rm -rf /workspace/agyDevKit/node_modules /workspace/agyDevKit/apps/*/node_modules /workspace/agyDevKit/packages/*/node_modules /root/.npm` (frees **~335 MB**).
   - Remaining persistent OverlayFS footprint will be **~25–30 MB** (`.git` 9.2M + `apps/` 11M + `packages/` 297K + `.chat` 209K + system 8.6M), well under the 100 MB limit.
3. Restore tooling via the official repo scripts (`tmpfs`-backed):
   - `bash scripts/env/bootstrap.sh` / `node scripts/env/doctor.mjs --auto-restore`
   - Verify with `pnpm run env:check`.
4. Include the `scripts/env/bootstrap.sh`, `scripts/env/doctor.mjs --auto-restore`, and `pnpm run env:check` instructions in the agent system prompt / `AGENTS.md` / compaction template so future sessions never install `node_modules` onto OverlayFS.
