# RocketCompanion PR #9 合并与 Tauri Dev 自动重启 / `.DS_Store` 修复报告

## 1. UI 字体修复 PR 合并 (`2dfd5c9` -> `alpha`)
- **PR**：[#9 fix(ui): prioritize ui-monospace and SFMono-Regular for SF Mono on Apple devices](https://github.com/Antigravi-ty/RocketCompanion/pull/9)
- **来源分支**：`z/agy/ui/fix-sf-mono-font-stack` (`2dfd5c9`) -> 目标分支：`alpha`
- **验证**：合并前运行 `pnpm check`（`radar` + `tsc --noEmit`）全部通过（1354/1354 TS，0 错误）。
- **合并结果**：已合并进 `alpha`（Merge Commit：`aa90f2f85fcaf5d31987e51b266e8e42b082a6f0`）。
  - *说明*：GitHub API 不允许 PR 创建者审批自己的 PR（返回 `422 Review Can not approve your own pull request`），但该 PR 已直接通过 API 合并进 `alpha`。

---

## 2. 问题排查结论

### 2.1 为什么 Tauri 之前不能随 `git checkout` 自动重启？
1. **`tauri dev` 的监听范围限制**：`tauri dev` 内置的文件监听器（`notify` + `ignore`）**只监听 `src-tauri/` 目录**，且默认忽略 `.git/`。
2. **`build.rs` 不会主动触发重启**：`src-tauri/build.rs` 中的 `cargo:rerun-if-changed=.git/HEAD` 仅在 Cargo 被调用编译时告诉 Cargo 是否重跑 `build.rs`，它**不能**通知 `tauri dev` 去重启进程。
3. **结果**：当你执行 `git checkout` 切换到只改动了前端 `src/` 的 commit 时，Vite 侧的 `gitHeadAutoRestartPlugin.ts` 会自动重启并刷新 WebView，但 `tauri dev` 看不到 `src-tauri/` 有改动，因此 **Tauri 进程完全不会重启**，托盘不会掉线重连，托盘/窗口标题/About 中的 Commit ID 也会停留在旧版本。

### 2.2 为什么 `.DS_Store` 会把 Tauri 进程杀掉并重启？如何避免？
- **原因**：在 macOS 上使用 Finder 打开 `src-tauri/` 或其子目录时，系统会写入 `.DS_Store`。由于仓库原先没有 `src-tauri/.taurignore`，且 `src-tauri/.gitignore` 中也未包含 `.DS_Store`，`tauri dev` 的文件监听器会将其识别为 `src-tauri/` 内的文件变更，直接杀掉正在运行的 `RocketCompanion` 并重新编译启动。
- **根治方案（已实现）**：新增 `src-tauri/.taurignore`（忽略 `.DS_Store`、`**/.DS_Store`、`._*`、`**/._*`、`target/`、`gen/`），并同步添加到 `src-tauri/.gitignore`。`tauri dev` 会原生读取 `.taurignore`，彻底无视 Finder 生成的元数据文件。

### 2.3 关于“Vite 自动重启的 MJS 能不能/有没有必要写成 TypeScript”
- 目前 Vite 的 Git HEAD 自动重启插件（`scripts/vite/gitHeadAutoRestartPlugin.ts`）和 4 端口启动器（`scripts/devLauncher.ts`）**已经是 TypeScript**，通过 Node 22+ 原生 `node --experimental-strip-types` 直接执行，无需额外编译。
- `scripts/` 中其余 `.mjs`（如 `syncPrivateProfile.mjs`、`generateTauriCrossbarIcons.mjs`）属于独立构建/工具脚本，没有必要强行全部重写。

---

## 3. 新分支实现方案 (`z/agy/rc/tauri-dev-watch`)

基于最新 `alpha`（`aa90f2f`）创建工作分支 `z/agy/rc/tauri-dev-watch`，已提交 Verified Bot Commit `c3fb43c4bbc276a2c86e06c4cf18c4848a191156`。

### 改动清单（按模块解耦）
1. `src-tauri/.taurignore` & `src-tauri/.gitignore`
   - 屏蔽 macOS `.DS_Store` 与 `._*` AppleDouble 文件，杜绝误触发 `tauri dev` 重启。
2. `scripts/git/gitHeadState.ts`
   - 从 `gitHeadAutoRestartPlugin.ts` 抽离公共的 Git HEAD 与 `public` 子模块状态快照读取逻辑（含 `index.lock` / `HEAD.lock` 锁检测），供 Vite 插件与 Tauri 监督器复用。
3. `scripts/vite/gitHeadAutoRestartPlugin.ts`
   - 复用 `readCompositeGitHeadState`，并支持 `RC_DISABLE_VITE_GIT_RESTART=1`（当由 `tauriDevWatch.ts` 统一托管重启时避免重复重启）。
4. `scripts/tauriDevWatch.ts`
   - 纯 TypeScript 的 `tauri dev` 进程组监督器：
     - 正常编辑 `src/`：走 Vite HMR，不重启 Tauri。
     - 正常编辑 `src-tauri/`：走 `tauri dev` 原生增量重编重启。
     - 执行 `git checkout` / `git switch` / `git reset`：检测到 `.git/HEAD`（或 `public` 子模块）变化且锁释放 + 150ms 防抖后，立即终止整个 `tauri dev` 进程树（**托盘图标立刻消失**）并重新拉起（**新托盘图标出现即可测试，Commit ID 自动对齐**）。
     - 若某个中间 commit 导致 Rust 编译失败退出，监督器不会退出，会继续等待你 `checkout` 到下一个正常 commit 时自动恢复启动。
5. `package.json`
   - `pnpm tauri:dev` 和 `pnpm tauri:dev:tray` 默认走 `scripts/tauriDevWatch.ts`；保留 `pnpm tauri:dev:raw` 用于直接运行裸 `tauri dev`。

### 验证结果
- `pnpm check`（`radar` + `tsc --noEmit`）：通过（1354/1354 TS，0 错误）。
- `npx prettier --check`：通过。
- `readCompositeGitHeadState()` 实测输出正常。
