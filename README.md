# agyPreviewInstant <= 100 MB OverlayFS 存储优化与自愈环境工具链实现报告

已参考 `RocketCompanion`（commit `743573520e45ed4b1de47d300ca68fc48c89c2fb` 及其 `scripts/env/` 体系）完成可行性评估并直接落地实现。`agyPreviewInstant` 完全支持该 <= 100 MB OverlayFS 预算方案，并已通过 Verified Bot Commit 合入主分支 `a2`。

- **Commit SHA**: `4689b20eac4a685c388f9efa9d48c757e833a936` (`verified: true`, `antigravity-fine-grained-token[bot]`)
- **OverlayFS 实测占用**: 安装全部 workspace 依赖、运行 `radar` / `check` / `build` 后仍保持 **8 MB**（baseline: 8 MB，budget: <= 100 MB）

## 1. 差异适配与架构设计

结合 `agyPreviewInstant` 的 pnpm workspace 单仓结构与架构雷达（`scripts/radar.ts`）约束，做了针对性适配：

1. **Monorepo 多目录 tmpfs 软链卸载**（`scripts/env/tmpfsLayout.mjs`）
   - 将所有大体积产物目录通过软链卸载到 `/tmp/agy-cache`（不占用 `/` OverlayFS），并自动写入 `.git/info/exclude`：
     - `node_modules`、`node_modules/.vite`、`.vite`、`dist`、`.cache`、`src-tauri/target`
     - `apps/p2p-remote-controller/node_modules`、`apps/p2p-remote-controller/node_modules/.vite`、`apps/p2p-remote-controller/.vite`、`apps/p2p-remote-controller/dist`、`apps/p2p-remote-controller/src-tauri/target`
     - `packages/shared-config/node_modules`、`packages/macos-ax-helper/node_modules`
   - `public/` 在本仓库中是 100 KB 的普通 tracked 静态资源（非 git submodule），无需 submodule 同步逻辑。

2. **模块拆分满足 Radar 体积护栏**
   - `agyPreviewInstant` 的 `scripts/radar.ts` 对 `scripts/*.mjs` 强制执行 `18 KB` 硬上限与 `12 KB` 软上限。因此将 `tmpfsGuard` 拆分为单职责小模块：`tmpfsLayout.mjs`、`tmpfsPnpm.mjs`、`tmpfsGuard.mjs`。

3. **自愈 `pnpm` 包装器与 `/tmp` 快照重置检测**（`scripts/env/tmpfsPnpm.mjs`、`scripts/env/tmpfsGuard.mjs`）
   - 将 `pnpm@10.28.2` 安装到 `/tmp/agy-cache/pnpm-global`，并在 `/usr/local/bin/pnpm` 写入自愈 wrapper：
     - `/tmp` 被容器快照重置后，运行任意 `pnpm ...` 会自动重装 `pnpm` 到 tmpfs 并重建基础软链；若 `node_modules/.modules.yaml` 缺失，则提示一键修复命令。
     - 通过 `/tmp/.agy-env-sentinel` 与 `.git/agy-overlay-baseline.json` 自动检测悬空软链与 `/tmp` 重置。
   - `bootstrap.sh` 与 `doctor.mjs --auto-restore` 会自动补齐 `src/config/resolvedProfile.json`（调用 `scripts/syncPrivateProfile.mjs`），确保全新环境可直接通过 `tsc --noEmit` 与 `pnpm run check`。

## 2. 新增文件与命令

- 新增文件：
  - `scripts/env/tmpfsLayout.mjs`
  - `scripts/env/tmpfsPnpm.mjs`
  - `scripts/env/tmpfsGuard.mjs`
  - `scripts/env/bootstrap.sh`
  - `scripts/env/checkEnv.mjs`
  - `scripts/env/doctor.mjs`
  - `scripts/env/help.mjs`
  - `scripts/env/README.md`
- `package.json` 新增脚本：
  - `pnpm run env:help` — 查看环境与自愈命令速查表
  - `pnpm run env:check` — 快速状态检查（OverlayFS 体积、tmpfs sentinel、workspace 依赖）
  - `pnpm run env:doctor` — 完整诊断（支持 `--auto-restore`、`--require-rust`）
  - `pnpm run env:setup` — 一键初始化/修复（pnpm + workspace `node_modules` 卸载至 tmpfs）
  - `pnpm run env:setup:rust` — 同上 + 安装 Rust/Cargo 及 Tauri Linux C 构建依赖至 tmpfs

## 3. 验证结果

- `pnpm run env:help`：正常输出命令速查表。
- `pnpm run env:check` / `pnpm run env:doctor`：Node.js v22.23.5、pnpm 10.28.2、root & p2p `node_modules` (tmpfs)、`/tmp/.agy-env-sentinel` 全部 `[OK]`；OverlayFS footprint **8 MB / <= 100 MB**。
- `pnpm run radar`：297 个文件全部通过（`Hard=800L/18KB (0/0 ratchet)`，`Cycles=0`，`P2P Type/Lint Check : PASS`）。
- `pnpm run check`：`tsc --noEmit`、`eslint src`、`vitest run`（2 test files, 16 passed）全部通过。
- `pnpm run build`：Vite 构建成功，产物写入 `/tmp/agy-cache/dist`，OverlayFS 占用仍为 **8 MB**。

## 4. 配套提示词（可直接复用）

```text
若环境未初始化或 /tmp 快照重置：无 pnpm 时跑 bash scripts/env/bootstrap.sh，有异常时跑 node scripts/env/doctor.mjs --auto-restore（含 Rust 需求加 --require-rust）。

随时可运行 pnpm run env:help 查看环境命令，或 pnpm run env:check 检查状态与 OverlayFS 预算（<= 100 MB）。
```
