# Track D (`feat/env-tmpfs-restore-guard`) 完成报告

## 1. 分支与文件边界

- **Base**: `origin/z/agy/rc/multiplayer-recheck` (`7252e42`)
- **Branch**: `feat/env-tmpfs-restore-guard` (`7fffabf`)
- **独占修改文件**（严格限定在 Track D 边界内）：
  - `scripts/env/tmpfsGuard.mjs`（新建）
  - `scripts/env/bootstrap.sh`（修改）
  - `scripts/env/checkEnv.mjs`（修改）
  - `scripts/env/doctor.mjs`（修改）

---

## 2. 核心实现内容

### 2.1 `pnpm` 自动安装到 tmpfs 与自愈包装器
- 容器内默认仅预装 `npm` 而无 `pnpm`。`scripts/env/tmpfsGuard.mjs --install-pnpm`（及 `bash scripts/env/bootstrap.sh --pnpm-only`）会通过 `npm` 将 `pnpm@10.28.2` 安装到 `/tmp/rc-cache/pnpm-global`（tmpfs），并清理 `/tmp/rc-cache/npm-cache`，避免占用 OverlayFS。
- 在 `/usr/local/bin/pnpm`（及 `pnpx`，不可写时回退 `~/.local/bin`）生成约 1.2KB 自愈包装脚本：当检测到 `/tmp` 被快照恢复清空时，自动重新安装 `pnpm` 到 `/tmp/rc-cache/pnpm-global` 并调用 `node scripts/env/tmpfsGuard.mjs --ensure` 修复软链接。

### 2.2 OverlayFS 基线尺寸记录与 tmpfs 全量分流
- 在 `.git/rc-overlay-baseline.json` 中记录 `git clone` 后的 OverlayFS 初始基线尺寸（`29 MB`），设定预算上限 `100 MB`（可通过 `RC_OVERLAY_BUDGET_MB` 覆盖）。
- 将所有大体积缓存、依赖、子模块与构建产物分流并软链接至 `/tmp/rc-cache/`：
  - `node_modules` -> `/tmp/rc-cache/node_modules`
  - `node_modules/.vite` & `.vite` -> `/tmp/rc-cache/vite-cache`
  - `dist` -> `/tmp/rc-cache/dist`
  - `public`（`RocketCompanionStatics` submodule，使用 `--depth 1` + pinned SHA checkout + `git update-index --assume-unchanged public`）-> `/tmp/rc-cache/public`（避免 `.git/modules/public` + `public/` 占用 ~195MB OverlayFS）
  - `src-tauri/target` -> `/tmp/rc-cache/tauri-target`（`crates/target` -> `/tmp/rc-cache/wasm-target`）
  - `CARGO_TARGET_DIR` / `CARGO_HOME` / `RUSTUP_HOME` -> `/tmp/rc-cache/cargo-target` / `cargo-home` / `rustup-home`
  - `pnpm` store/cache -> `/tmp/rc-cache/pnpm-store` / `pnpm-cache`
  - Headless Chrome `user-data-dir`（`doctor.mjs`）-> `/tmp/rc-cache/chrome-userdata/`
  - `.rcreplay` 3v3 fixture -> `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`

### 2.3 `/tmp` 重置自动检测（`/tmp/.rc-env-sentinel`）与 `--auto-restore`
- `tmpfsGuard.mjs` 在 `/tmp/.rc-env-sentinel` 写入环境指纹（含 `repoRoot`、`cacheRoot`、`viteCacheDir`、`cargoTargetDir`、`replayFixturePath`、`replayFixturePresent`、`overlayBaselineMB`、`symlinks` 等）。
- `checkEnv.mjs` 与 `doctor.mjs` 会检测以下重置信号：
  1. 存在悬空软链接（dangling symlinks，如 `public` / `node_modules` 指向已清空的 `/tmp/rc-cache/...`）；
  2. `.git/rc-overlay-baseline.json` 存在但 `/tmp/.rc-env-sentinel` 丢失；
  3. sentinel 中记录为存在的资产（`viteCacheDir`、`cargoTargetDir`、`replayFixturePresent: true` 对应的 `.rcreplay`）在 `/tmp` 中丢失。
- 检测到 `/tmp` 重置时，打印醒目诊断横幅及可复制执行的恢复命令（`node scripts/env/doctor.mjs --auto-restore`、`pnpm run env:setup`、`pnpm run env:setup:rust`、`pnpm run replay:generate-ducky`）。
- 支持 `--auto-restore` 参数（`doctor.mjs --auto-restore`、`checkEnv.mjs --auto-restore`、`tmpfsGuard.mjs --auto-restore`）：自动重建 `/tmp/rc-cache` 目录与软链接、重装 tmpfs `pnpm`、同步 `public/` 子模块、恢复 `node_modules`、通过 6-Ducky WASM 仿真重生成 `.rcreplay` fixture 并刷新 `/tmp/.rc-env-sentinel`。
- 修复了 `checkEnv.mjs`（5s -> 15s）和 `doctor.mjs`（8s -> 15s）在 SwiftShader 冷启动时的 `Runtime.evaluate` WebGL2 超时问题，并将 `doctor.mjs` 中的失效提示 `scripts/env/setup.sh` 修正为 `scripts/env/bootstrap.sh`。

---

## 3. 实测验证结果

1. **沙箱真实 `/tmp` 重置自愈实测**：
   - 运行 `pnpm run env:check` 时，`/usr/local/bin/pnpm` 自动检测到 `/tmp` 被清空，3 秒内重装 `pnpm@10.28.2` 到 `/tmp/rc-cache/pnpm-global`；随后 `checkEnv.mjs` 准确识别 `Dangling tmpfs symlinks detected: public` 与 `.rcreplay` 缺失并输出恢复命令。
   - 执行 `node scripts/env/doctor.mjs --auto-restore` 后，自动重建全部目录与 6-Ducky `.rcreplay` fixture（246560 bytes），全部检查项 `[OK]`：
     - `OverlayFS footprint: 29 MB (baseline: 29 MB, budget: <= 100 MB)`
     - `Headless WebGL (SwiftShader): WebGL 2.0 (OpenGL ES 3.0 Chromium) | WebKit WebGL`
2. **`pnpm run env:check`**：ALL `[OK]`（OverlayFS `29 MB / <= 100 MB`，WebGL2 ready）。
3. **`pnpm run check`**（Semantic Architecture Radar + `tsc --noEmit`）：`STATUS: GREEN`（`1430/1430` TS，0 循环依赖，0 hard errors）。
4. **`pnpm run test:multiplayer-suite`**：`26/26 PASS`。
