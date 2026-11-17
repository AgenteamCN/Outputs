# P2P Remote Controller 排查与修复报告

已提交 Verified Bot Commit 至分支 `z/p2p-remote-controller-monorepo`：
- Commit: `d9172f1100c2b8ee874e5b4e022c80f1f3a2eb82` (`https://github.com/Antigravi-ty/agyPreviewInstant/commit/d9172f1100c2b8ee874e5b4e022c80f1f3a2eb82`)

---

## 1. 为什么已经 `AX Trusted` 却还是提示 `Not authorized to send Apple events to Safari.`？

这是 **macOS TCC 权限机制** 与 **代码缺失配置/容错** 共同导致的：

1. **Accessibility（辅助功能）与 Automation（自动化 / Apple Events）是两套完全独立的 TCC 权限**：
   - `AX Trusted: true` 仅代表 **系统设置 → 隐私与安全性 → 辅助功能（Accessibility / `kTCCServiceAccessibility`）** 已授权，允许通过 `AXUIElement` 读取/操作窗口控件。
   - `Open Managed Safari Window` 与原来的 `listWindows()` 使用的是 `NSAppleScript`（`tell application "Safari" ...`），这需要 **系统设置 → 隐私与安全性 → 自动化（Automation / `kTCCServiceAppleEvents`）→ `<宿主应用>` → `Safari`** 的授权。
   - 仅开启辅助功能**不会**自动授予对 Safari 的 Apple Events 权限，因此 macOS 会返回 `OSStatus -1743`（`errAEEventNotPermitted`）。

2. **代码层面的三个原因**：
   - **缺少 `Info.plist` 中的 `NSAppleEventsUsageDescription`**：`apps/p2p-remote-controller/src-tauri/` 下原本没有 `Info.plist`。在 macOS 10.14+（尤其是 Sequoia）上，如果未声明 `NSAppleEventsUsageDescription`，TCC 会直接以 `-1743` 拒绝 Apple Events 请求，甚至可能不弹出授权弹窗。
   - **未显式检测/触发 Automation 权限**：`macos-ax-helper` 此前只检查了 `AXIsProcessTrustedWithOptions`，没有调用 `AEDeterminePermissionToAutomateTarget` 去检查或触发对 `com.apple.Safari` 的自动化授权。
   - **`listWindows()` 静默吞掉了 AppleScript 错误**：`SafariController.listWindows()` 此前用 `try? runAppleScript(...)` 吞掉了 `-1743` 错误并直接返回 `[]`，导致即使勾选了 `Include All Safari Windows`，界面上也显示 `Safari Windows (0)`，让人误以为完全无法读取 Safari 窗口。

---

## 2. 本次代码改动详情

### (1) Safari Automation TCC 检测 + `Info.plist` + AX 窗口枚举兜底
- **新增 `apps/p2p-remote-controller/src-tauri/Info.plist`**：声明 `NSAppleEventsUsageDescription`。
- **更新 `packages/macos-ax-helper/Sources/MacosAxHelper/` (`Models.swift`, `SafariController.swift`, `AccessibilityEngine.swift`, `JsonRpcServer.swift`)**：
  - 新增 `check_safari_automation`（基于 `AEDeterminePermissionToAutomateTarget`，区分 `authorized` / `prompt_required (-1744)` / `denied (-1743)` / `safari_not_running (-600)`）和 `open_automation_settings`（一键打开系统设置 → 隐私与安全性 → 自动化）。
  - `SafariController.listWindowsWithSource()` 在 AppleScript 失败时不再静默返回空数组，而是记录 `appleEventsError` 并自动回退到 `AccessibilityEngine.listSafariWindowsViaAX()`（纯 `AXUIElement` 枚举 Safari 窗口，无需 Apple Events），同时在状态中标明 `windowListSource: "applescript" | "ax_fallback"`。
  - `runAppleScript` 错误信息现在会带上具体的 `OSStatus` 错误码与排查提示。

### (2) 默认切换为亮色模式（Light Mode）
- **更新 `apps/p2p-remote-controller/src/styles/app.css`**：将 `:root` 默认主题改为 GitHub Primer 风格的亮色配色（`color-scheme: light`、`#f6f8fa` 背景、`#ffffff` 面板、`#1f2328` 文字），并同步调整了徽章、错误横幅、警告横幅、角色标签和代码块/控制台日志的亮色对比度。

### (3) 增强全链路控制台与调试诊断输出
为了让后续排障一眼定位“卡在哪一步、耗时多少、底层返回什么错误”：
- **Swift (`macos-ax-helper`)**：在 `JsonRpcServer`、`SafariController`、`AccessibilityEngine` 中向 `stderr` 输出带阶段标签的结构化日志（如 `[ax-helper] [rpc] id=... method=... ok/err durationMs=...`、`[ax-helper] [safari:automation]`、`[ax-helper] [ax:windows]`）。
- **Rust (`apps/p2p-remote-controller/src-tauri/src/ax_bridge.rs`, `commands.rs`, `lib.rs`)**：
  - 捕获 `macos-ax-helper` 的 `stderr` 流并写入内存环形缓冲区（最多 300 条）同时转发到终端 `eprintln!`。
  - 记录解析到的 helper 二进制路径、PID、每次 RPC 调用耗时（`durationMs`）与错误详情，新增 `ax_get_diagnostics` 命令供前端拉取。
- **前端 (`src/App.tsx`, `src/services/axService.ts`, `src/types/ax.ts`)**：
  - 顶部状态栏新增 **Safari Automation** 状态徽章及 `Grant Safari Automation` / `Open Automation Settings` 快捷按钮；若 Apple Events 被拦截且已回退到 AX 枚举，会显示黄色提示横幅。
  - 右侧面板新增 **Console** 标签页，集中展示：Helper 运行状态/PID/二进制路径、**Frontend IPC Trace**（每条命令耗时与摘要）、以及 **Rust Bridge & Swift Helper Logs**。
  - **Quick Copy** 导出的 JSON 现在会一并携带 `automation`、`windowListSource`、`appleEventsError`、`errorMessage`、`diagnostics` 和最近 30 条 `recentConsoleLogs`，方便一键复制完整现场。

---

## 3. 关于 `cargo` 是否应当上传或被 ignore

- **`Cargo.toml`**：**必须上传（track）**，它是 Rust 包与依赖清单。
- **`Cargo.lock`**：**应当上传（track）**（仓库当前已追踪 `apps/p2p-remote-controller/src-tauri/Cargo.lock` 和 `src-tauri/Cargo.lock`）。对于 Tauri 这类**可执行应用项目**，官方建议提交 `Cargo.lock` 以保证不同机器和 CI 的依赖版本一致、构建可复现。
- **`target/`**：**必须 ignore**（仓库 `.gitignore` 第 5 行和第 21 行已忽略 `src-tauri/target/` 与 `apps/*/src-tauri/target/`），里面是编译产物，体积很大不应入库。

---

## 4. 本地验证建议（macOS）

1. 拉取最新代码并重新编译 Swift helper（注意先重建 helper 再启动 dev）：
   ```bash
   git pull
   pnpm ax-helper:build
   pnpm p2p:dev
   ```
2. 如果之前已经在当前终端/IDE/App 上拒绝过一次 Safari Apple Events（`-1743`），可先在 macOS 终端重置一次 TCC 缓存再触发授权：
   ```bash
   tccutil reset AppleEvents
   ```
   然后点击界面右上角的 `Grant Safari Automation`（或去 **系统设置 → 隐私与安全性 → 自动化** 勾选对应进程下的 `Safari`）。
3. 本次在 Linux 沙盒环境中已安装 `pnpm` 与 `rustup`，并通过 `pnpm p2p:check`（`tsc --noEmit && eslint src`）与 `pnpm p2p:build`（`tsc && vite build`）；Rust 与 Swift 改动已通过源码审查，需在 macOS 本机运行 `pnpm p2p:dev` 完成原生编译与 TCC 实测。
