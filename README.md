# Safari Automation TCC (`-1743` / 空列表) 根因分析与修复报告

## 1. 现象与根因分析

从你提供的日志和诊断 JSON 可以 100% 定位到问题所在：

1. **为什么点击 `Grant Safari Automation` 后瞬间返回 `-1743 (denied)`（`durationMs=0`），且 `Automation` 设置里是空的（0 条）？**
   - 关键日志：
     - `prompt=false` → `status=-1744 (prompt_required) bundleId=nil`（耗时 ~15ms，走了 TCC 查询）
     - `prompt=true` → `status=-1743 (denied) bundleId=nil`（耗时 **0ms**）
     - 之后再次 `prompt=false` → 依然是 `-1744 (prompt_required)`！
   - 这说明 **macOS 根本没有弹窗，也没有把拒绝记录写入 TCC 数据库**，而是在 `AEDeterminePermissionToAutomateTarget(..., prompt=true)` 入口处**直接预检失败并拦截了**。
   - 根本原因：真正调用 `AEDeterminePermissionToAutomateTarget` 的子进程是 Swift CLI `macos-ax-helper`（`apps/p2p-remote-controller/src-tauri/target/debug/macos-ax-helper`）。此前只给 Rust 主程序 `p2p-remote-controller` 嵌入了 `Info.plist`，而 **`macos-ax-helper` 自身没有任何 `Info.plist`（`bundleId=nil`，也缺少 `NSAppleEventsUsageDescription`）**。
   - 在 macOS（包括 Sequoia 15+）中，如果调用 `AEDeterminePermissionToAutomateTarget(..., true)` 的进程自身 `Info.plist` 里没有 `NSAppleEventsUsageDescription`，系统会**直接返回 `-1743` 而不弹窗**；因为从未真正向 `tccd` 发起授权弹窗，所以 **系统设置 -> 隐私与安全性 -> 自动化 (Automation)** 里不会出现任何条目（不像 Accessibility 可以手动点 `+` 添加，Automation 只有成功触发过 TCC 请求才会出现条目）。

2. **为什么 UI 上也没有看到具体的拒绝提示？**
   - 在 `apps/p2p-remote-controller/src/App.tsx` 的 `handleRequestAutomation` 中：先 `setErrorMessage(auto.message)`，紧接着调用 `await refreshStatus()`，而 `refreshStatus()` 开头会执行 `setErrorMessage(null)`，把刚刚设置的错误提示瞬间清掉了。

3. **开发模式 (`pnpm dev:all`) 下的 Responsible Process 注意事项（macOS Sequoia）：**
   - 即使 `macos-ax-helper` 嵌入了 `Info.plist`，在 `tauri dev`（`cargo run` 未打包成 `.app`）下，macOS TCC 的 Responsible Process 会向上追溯到**启动 `pnpm dev:all` 的终端/IDE**（例如 `Terminal.app`、`iTerm2`、或者 `Cursor`/`VS Code`/其他终端）。
   - 如果你的终端或 IDE 本身带有 `NSAppleEventsUsageDescription`（如 `Terminal.app`、`iTerm2`），弹窗会以该终端名义弹出并出现在 Automation 列表中；如果某些第三方 IDE/终端开启了 Hardened Runtime 且没有 `com.apple.security.automation.apple-events` 权限或缺少 `NSAppleEventsUsageDescription`，`tccd` 仍可能拒绝弹窗——此时用 `Terminal.app` 运行或构建 `.app` 包（`pnpm p2p:build:app`）即可让 `P2P Remote Controller.app` 自身作为 Responsible Process 正常弹窗。

---

## 2. 已完成的修复

已提交 Verified Bot Commit 至分支 `z/p2p-remote-controller-monorepo`：
- **Commit**: `6a79c1a4a988a6090419b75648dadba06980a0da` (`verified: true`)
- **链接**: https://github.com/Antigravi-ty/agyPreviewInstant/commit/6a79c1a4a988a6090419b75648dadba06980a0da

具体改动：
1. **`packages/macos-ax-helper/Info.plist`**（新增）
   - 添加 `CFBundleIdentifier` (`com.antigravity.p2premotecontroller.ax-helper`)、`CFBundleName`、`CFBundleExecutable`、`CFBundleShortVersionString`、`CFBundleVersion` 以及 `NSAppleEventsUsageDescription`。
2. **`packages/macos-ax-helper/Package.swift`**
   - 通过 `linkerSettings` 的 `-Xlinker -sectcreate -Xlinker __TEXT -Xlinker __info_plist` 将 `Info.plist` 直接内嵌进 `macos-ax-helper` Mach-O 二进制。
3. **`scripts/buildAxHelper.sh`**
   - 复制构建产物后默认执行 `codesign --force --sign "${AGY_CODESIGN_IDENTITY:--}"`（未配置签名证书时自动使用 ad-hoc `-` 签名），确保内嵌 `Info.plist` 后具备有效签名与 `cdhash`。
4. **`apps/p2p-remote-controller/src-tauri/Info.plist`**
   - 补全 `CFBundleIdentifier` (`com.antigravity.p2premotecontroller`)、`CFBundleName`、`CFBundleDisplayName` 等基础 Bundle 元数据，避免 `tauri dev` 裸二进制运行时 `Bundle.main.bundleIdentifier` 为 `nil`。
5. **`packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift`**
   - 在 `[safari:automation]` 日志中增加 `hasUsageDesc=<true|false>` 诊断输出；当 `prompt=true` 仍返回 `-1743` 时给出明确的 dev 模式父进程/打包 `.app` 排查指引。
6. **`apps/p2p-remote-controller/src/App.tsx`**
   - 调整 `handleRequestAutomation` 执行顺序，在 `await refreshStatus()` 之后再设置 `errorMessage`，避免错误提示被 `refreshStatus()` 清空。

---

## 3. 本地验证步骤

请在 macOS 本地拉取最新提交后重新启动：
```bash
git pull origin z/p2p-remote-controller-monorepo
pnpm dev:all
```
然后点击 **Grant Safari Automation**：
- 在诊断 JSON / 控制台中应能看到 `bundleIdentifier: "com.antigravity.p2premotecontroller.ax-helper"` 及 `hasUsageDesc=true`。
- 若你是从 `Terminal.app` / `iTerm2` 启动 `pnpm dev:all`，macOS 将弹出 Automation 授权确认框，授权后 **System Settings -> Privacy & Security -> Automation** 中会出现对应条目。
- 若当前 IDE 终端自身限制了 Apple Events，可改用 `Terminal.app` 启动或运行 `pnpm p2p:build:app` 打开打包后的 `P2P Remote Controller.app` 进行授权。
