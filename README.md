# Git Checkout 自动重启与刷新机制重构报告

- **分支**: `z/agy/rc/git-hook-auto-reload`（基于 `alpha` 最新提交 `8b6b0a2`）
- **Verified Bot Commit**: `ec5d23d3861fca169cc7ad52df6421a9e693b5df`（`verified: true`，`antigravity-fine-grained-token[bot]`，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 一、核心架构升级（Hook 确定性通知 + 外层 Supervisor + Ready Gate）

放弃原先纯轮询打补丁的模式，重构为以下四层协同机制：

1. **版本化 Git Hooks (`.githooks/`) 与自动安装 (`scripts/git/ensureGitHooks.ts`)**
   - 新增 `.githooks/post-checkout`、`.githooks/post-merge`、`.githooks/post-rewrite`（`100755`）：Git 保证仅在工作区文件全部落盘且 `index.lock` 释放后触发。
   - Hook 原子写入 `.cache/git-checkout-event.json`（含 `eventId`、`tauriChanged`、`depsChanged`），并异步非阻塞 POST `/__rc_git_checkout_done` 实现 0ms 瞬时唤醒。
   - 在 `package.json` 增加 `prepare` / `hooks:install` 脚本，并在 `devLauncher.ts` 与 `tauriDevWatch.ts` 启动时自动执行 `ensureGitHooks()`，将 `core.hooksPath` 设为 `.githooks`。

2. **外层 Supervisor 统一编排 (`scripts/devLauncher.ts` + `scripts/devWorker.ts`)**
   - 将原 4 端口 Vite 启动逻辑拆分至 `scripts/devWorker.ts`，`scripts/devLauncher.ts` 升级为纯外层守护进程。
   - 检测到 checkout（Hook HTTP / `.cache/git-checkout-event.json` 监听 / submodule fallback）后，立即通过 IPC (`rc:checkout-settling`) 通知 4 个 Vite 实例进入静默态，等待 **1000ms 尾随防抖**（上限 3000ms）后统一重启 `devWorker.ts` 子进程。
   - 彻底解决 4 端口并发 `server.restart()` 争抢以及 `vite.config.ts` / `scripts/vite/*.ts` ESM 模块缓存不更新的问题。

3. **Ready Gate 与 HMR 抑制 (`scripts/vite/gitHeadAutoRestartPlugin.ts`)**
   - 每次 `configureServer` 生成唯一 `bootId`，并通过 `transformIndexHtml` 注入客户端。
   - 在锁文件存在、Hook 静默期或重启期间，`/__rc_git_head` 严格返回 `{ restarting: true, signature: null, bootId }`，同时拦截 `server.ws.send` 的 `update` / `full-reload` 消息，杜绝半成品状态抢跑刷新。
   - 客户端在检测到 `restarting: true` 或 WS 断开后显示顶部非侵入提示条，以 400ms 间隔探测直到 `restarting === false && data.bootId !== INITIAL_BOOT_ID` 才执行 `window.location.reload()`。

4. **Tauri 开发守护优化 (`scripts/tauriDevWatch.ts`) 与 Watcher 降噪 (`vite.config.ts`)**
   - `tauriDevWatch.ts` 不再在每次切换分支时无脑杀掉 Rust/Tray 进程：仅当 `src-tauri/` 目录发生变更（`tauriChanged: true`）时才等待 1000ms 并重启 `tauri dev`；纯前端/脚本变更时保持 Tauri 托盘常驻，由 `devLauncher.ts` 重启 4 端口 Vite 后 WebView 自动无感刷新。
   - `vite.config.ts` 的 `server.watch.ignored` 补充忽略 `**/.git/**`、`**/.cache/**`、`**/dist/**`、`**/rc-vite-cache/**`、`**/.DS_Store`、`**/._*`、`**/._AppleDouble`。

## 二、关于后台常驻命令 `pnpm tauri:dev:tray` 的建议

- **日常开发继续使用 `pnpm tauri:dev:tray` 即可**：本次优化后，纯前端分支切换不再重启 Tauri Rust/Tray 外壳，只会平滑重启 4 端口 Vite Worker，体验会比之前快且稳得多。
- **高频对比特定 Commit 推荐使用 `git worktree`**：如果需要在 2–3 个固定 commit 之间频繁对比效果，建议用 `git worktree add ../RocketCompanion-compare <commit>` 开出独立工作目录，避免反复触发磁盘 IO 与 Vite 缓存失效。

## 三、验证结果

- 工具链：已全局安装 `pnpm` (`12.21.0`) 并执行 `pnpm install --frozen-lockfile`。
- 静态检查：`pnpm run check`（`radar` 1389/1389 TS + `tsc --noEmit`）通过（GREEN）。
- 端到端闭环实测（本地启动 `devLauncher.ts` 监听 `:3003`）：
  - 初始 `/__rc_git_head` 返回 `bootId: e8148cc2-...`，`restarting: false`，HTML 中嵌入的 `INITIAL_BOOT_ID` 与之一致。
  - 触发 `POST /__rc_git_checkout_done` 后，立即进入 `settling: true, restarting: true, signature: null`。
  - 等待 1000ms settle + Worker 重启后，返回新 `bootId: e8d4db1d-...`，`restarting: false`。
