# RocketCompanion Checkout 行为与 Tauri 版本/Commit ID 同步机制分析报告

## 1. 从 `ec5d23d` 切换到 `68c3ae3` 的日志行为是否符合预期？

**结论：在两个分支尚未合并到一起的当前状态下，该日志行为完全符合预期（属于正常的跨架构分支切换现象，而非 Bug）。**

原因拆解如下：

1. **两个分支的结构差异**：
   - `ec5d23d`（`z/agy/rc/git-hook-auto-reload`）将 `scripts/devLauncher.ts` 重构为**外层 Supervisor**，把 4 端口 Vite 启动逻辑拆到了新文件 `scripts/devWorker.ts`，并在 `scripts/tauriDevWatch.ts` 中加入了「仅前端变更时不重启 Tauri 壳」的逻辑。
   - `68c3ae3`（`z/agy/ui/tuner-startup-optimization`）是直接从 `alpha`（`8b6b0a2`）切出来的平行分支，**工作区中还没有 `scripts/devWorker.ts`**。
2. **切换时发生了什么**：
   - 你在 `ec5d23d` 下启动 `pnpm tauri:dev:tray`，此时内存里常驻的是 `ec5d23d` 版本的 `tauriDevWatch.ts` 和 `devLauncher.ts`（Supervisor）。
   - 当你 `git checkout` 切到 `68c3ae3` 时：
     - `tauriDevWatch.ts` 对比 `src-tauri/` 目录无变化，因此打印 `Frontend-only HEAD change detected; keeping Tauri shell alive while Vite servers restart.`，保持 Tauri 进程存活。
     - `devLauncher.ts`（Supervisor）停止旧的 `devWorker.ts` 进程，尝试重新拉起磁盘上的 `scripts/devWorker.ts`。由于 `68c3ae3` 没有这个文件，Node 抛出 `MODULE_NOT_FOUND`；Supervisor 捕获了退出码并进入等待态：`[dev-supervisor] Worker exited unexpectedly (code 1). Waiting for next Git checkout to retry...`，**没有崩溃退出**。
   - 当你切回 `ec5d23d` 时，`scripts/devWorker.ts` 重新出现在磁盘上，Supervisor 立即自动恢复并拉起了 4 端口 Vite Worker。
   - 至于切换瞬间 Vite 打印的一连串 `page reload ...`，是因为 `git checkout` 改写工作区文件时，Vite 的文件监听在 Supervisor 的 1000ms 防抖杀掉 Worker 前短暂触发了一次事件，随后 Worker 即被重启，不影响最终状态。
3. 一旦两个分支都合并进 `alpha`，后续所有分支都会包含 `scripts/devWorker.ts`，就不会再出现 `MODULE_NOT_FOUND`。

---

## 2. Tauri 在不重启的情况下，能保证使用的是当前 Checkout 的版本吗？标题和托盘的 Commit ID 会自动更新吗？

**结论：前端运行代码可以保证是最新 Checkout 的版本，但原生窗口标题与托盘菜单/Tooltip 中显示的 Commit ID「不会」自动更新！**

### 2.1 为什么前端与原生功能本身不会错位？
- **WebView 前端代码**：Tauri 开发模式下加载的是本地 Vite Dev Server（`http://localhost:3003` / `https://localhost:3001`）。当发生仅前端的 `git checkout` 时，`devLauncher.ts` 会重启 4 端口 Vite Worker，配合 `gitHeadAutoRestartPlugin.ts` 的 `bootId` Ready Gate 机制自动刷新 WebView，因此页面 UI 和前端逻辑**确实是当前 Checkout 的最新版本**。
- **Tauri 原生壳代码**：`tauriDevWatch.ts` 通过 `git diff --quiet <prevSha> <nextSha> -- src-tauri/` 确认 `src-tauri/` 目录没有任何改动，因此 Rust 端的功能逻辑与当前 Checkout 完全一致，无需重新编译重启。

### 2.2 为什么标题和托盘图标上的 Commit ID 不会自动更新？（关键隐患）
检查 `src-tauri/build.rs` 与 `src-tauri/src/build_info.rs` 可以看到：
- `src-tauri/build.rs` 在 **Rust 编译期**执行 `git rev-parse --short=7 HEAD`，并通过 `println!("cargo:rustc-env=RC_GIT_COMMIT_SHORT={}", short_sha)` 写入编译期环境变量。
- `src-tauri/src/build_info.rs` 中的 `short_commit_id()` 使用的是编译期宏 `option_env!("RC_GIT_COMMIT_SHORT")`，并直接用于：
  - `format_window_title()`（主窗口 / Configurator 窗口标题：`Rocket Companion (<sha>)`）
  - `format_tray_launch_label()`（托盘菜单：`Launch (<sha>)`）
  - `format_tray_tooltip()`（托盘悬停提示：`Rocket Companion (<sha>)`）
- 并且 `src-tauri/src/lib.rs` 仅在应用启动时调用一次 `apply_window_titles(handle)` 和 `setup_system_tray(handle)`。

**因此**：只要 `tauriDevWatch.ts` 跳过了重启 `rc_tauri_shell`，Rust 二进制就不会重新编译或重启，**窗口标题、托盘 Tooltip 和托盘 `Launch (...)` 菜单项上显示的 Commit ID 会一直停留在最初启动 Tauri 时的那个旧 Commit（例如 `ec5d23d`），不会随前端热切换而更新**。

> **改进方向（如需标题/托盘 Commit ID 同步更新）**：在 dev 模式下不能只依赖编译期 `option_env!`，需要让 Rust 端在运行时（例如窗口焦点/显示时或通过本地信号/定时检查 `.git/HEAD`）动态读取当前 `HEAD` 短 SHA 并调用 `set_title` / `set_tooltip` / `launch_item.set_text` 刷新。

---

## 3. 关于 PR 提交与 Merge 到 `alpha` 的状态说明

由于你设定了前提条件：*「如果上述行为完全符合预期，而且 Tauri 能够保证及时的更新，完全没有问题……然后这两次 commit 都提交 PR，然后直接 merge 到 alpha」*，本次**暂未直接执行 PR 创建与 Merge**，原因有二：

1. **上述 Commit ID 静态编译的问题**：Tauri 原生标题与托盘上的 Commit ID 在仅前端切换时**不会**自动更新。
2. **两个分支之间存在合并冲突（已通过本地试合并验证）**：
   - `z/agy/rc/git-hook-auto-reload`（`ec5d23d`）可以无冲突合入 `alpha`。
   - 但在其合入后，再合并 `z/agy/ui/tuner-startup-optimization`（`68c3ae3`）会在 `scripts/devLauncher.ts` 产生 **Content Conflict**：
     - `ec5d23d` 把 `startAllServers()` 从 `scripts/devLauncher.ts` 移到了 `scripts/devWorker.ts`；
     - `68c3ae3` 修改了原 `scripts/devLauncher.ts` 中的 `startAllServers()`（增加了 `ep.role === 'tuner'` 时对 `/src/ui-tuner/main.tsx` 的 `warmupRequest`）。
   - 解决方式很明确：将 `68c3ae3` 中对 `tuner` 的 warmup 逻辑迁移合并到 `scripts/devWorker.ts` 中即可。

如果你确认「dev 模式下原生标题/托盘保留构建时的 Commit ID 可以接受（或后续另开 PR 修复动态刷新）」，我可以立即按以下顺序处理：
1. 先将 `z/agy/rc/git-hook-auto-reload` 提交 PR 并 merge into `alpha`；
2. 基于最新 `alpha` 解决 `z/agy/ui/tuner-startup-optimization` 的 `devLauncher.ts` -> `devWorker.ts` 冲突（通过 Verified Bot Commit 更新分支），提交 PR 并 merge into `alpha`。

---

## 4. 术语澄清：`merge to alpha` vs `alpha merge 两个 branch`

**建议统一使用：`merge <分支> into alpha`（中文：「把这两个分支合并到 `alpha`」或「merge to `alpha`」）。**

- **为什么不要说 `"alpha merge 两个 branch"`**：
  在中文口语中，`A merge B` 极易产生主宾/方向歧义——可能被理解成「在 `alpha` 上 merge 两个分支（正确方向）」，也可能被理解成「把 `alpha` 合并进两个分支（反向同步：`git checkout branch && git merge alpha`）」。
- **最无歧义的表述**：
  - 英文：`Create PRs for both branches and merge them into alpha`（或 `merge branchA and branchB into alpha`）。
  - Git/PR 语义：`base: alpha <- head: <feature-branch>`。
