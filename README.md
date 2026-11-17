### 问题原因分析

这两个现象里，**第一个是仓库配置本身的 Bug，第二个是正常现象加上缺少多应用并发启动脚本**：

1. **`pnpm p2p:dev` 报错 `Command "ax-helper:build" not found`（代码配置问题）**
   - 执行 `pnpm p2p:dev` 时，`pnpm --filter p2p-remote-controller tauri dev` 会把工作目录切换到 `apps/p2p-remote-controller`。
   - `apps/p2p-remote-controller/src-tauri/tauri.conf.json` 原先配置的 `beforeDevCommand` 为 `pnpm ax-helper:build && pnpm --filter p2p-remote-controller dev`，而 `ax-helper:build` 只定义在根目录 `package.json` 中，子包里找不到该脚本，导致直接退出。

2. **`pnpm tauri dev` 看起来"卡在最后一行"且只启动了一个应用（非报错）**
   - 日志里已经出现 `Finished ... in 0.45s` 和 `Running target/debug/agy-preview-instant`，说明主应用 `agy-preview-instant` **已经编译并正常启动**。`tauri dev` 在桌面窗口运行期间会持续驻留前台监听文件变更与输出日志，直到你关闭应用或按 `Ctrl+C`，并非卡死。
   - `pnpm tauri dev` 只会启动根目录 `src-tauri/` 对应的 `agy-preview-instant`（端口 `1420`），原本并没有配置同时拉起 `apps/p2p-remote-controller`（端口 `1421`）的脚本。

---

### 已完成的修复与新增命令

已在分支 `z/p2p-remote-controller-monorepo` 提交修复（Verified Bot Commit [`d19e543`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/d19e543244602652824a3958c801382c246fd3f1)）：

1. **修复 `p2p-remote-controller` 构建前命令**
   - 更新 `apps/p2p-remote-controller/src-tauri/tauri.conf.json` 中的 `beforeDevCommand` 与 `beforeBuildCommand`，改用 `pnpm --workspace-root ax-helper:build` 从工作区根目录调用 Swift helper 构建脚本。
   - 统一 `apps/p2p-remote-controller/vite.config.ts`、`apps/p2p-remote-controller/src-tauri/tauri.conf.json` 与根目录 `src-tauri/tauri.conf.json` 的开发地址为 `127.0.0.1`（分别为 `http://127.0.0.1:1420` 与 `http://127.0.0.1:1421`），并忽略 `src-tauri` 目录监听，避免 macOS 下 `localhost` IPv6 回环解析延迟和 Rust 编译产物触发 Vite 重载。

2. **整理后的启动命令（在仓库根目录执行）**
   - **一个命令同时启动两个应用**：
     ```bash
     pnpm dev:all
     ```
     通过新增的 `scripts/devAll.mjs` 并行启动两个 Tauri 应用，终端日志会带有 `[agy]` 和 `[p2p]` 前缀，按 `Ctrl+C` 会一并清理两个进程树。
   - **单独启动 `agy-preview-instant`**：
     ```bash
     pnpm agy:dev
     ```
     （原有的 `pnpm tauri dev` 依然可用，等价于 `pnpm agy:dev`。）
   - **单独启动 `p2p-remote-controller`**：
     ```bash
     pnpm p2p:dev
     ```

---

### 工具链与验证结果

- **工具链**：沙箱环境中已安装 `pnpm`（`12.21.0`）并执行 `pnpm install` 与 `node scripts/syncPrivateProfile.mjs`。你在本地 macOS 运行时，请先执行 `git pull` 拉取最新分支，并确保本地已安装 Node/pnpm、Rust (`cargo`) 以及 Xcode Swift 工具链（用于编译 `packages/macos-ax-helper`）。
- **检查结果**：
  - `node --check scripts/devAll.mjs` 通过
  - `pnpm check`（`tsc --noEmit`、`eslint src` 0 errors、`vitest run` 13/13 tests passed）通过
  - `pnpm p2p:check`（`tsc --noEmit`、`eslint src`）通过
