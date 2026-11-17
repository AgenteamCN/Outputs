# Safari Automation TCC (`-1743`) 瞬时拒绝与 Automation 设置为空的原因及修复

## 问题原因（为什么点击 Grant Safari Automation 瞬间返回 `-1743`，且 System Settings → Automation 为空）

并不是 macOS Sequoia 取消了 Automation 弹窗，而是 **macOS TCC (`tccd`) 在弹窗前直接拦截了请求**：

1. **开发模式 (`pnpm dev:all`) 下的 Responsible Process 归因机制**  
   在 `tauri dev` 模式下，`p2p-remote-controller` 和子进程 `macos-ax-helper` 都是由终端 / IDE（如 RustRover / Cursor / 第三方终端）拉起的子进程，而不是通过 LaunchServices 启动的独立 `.app`。  
   macOS TCC 会沿着进程树向上查找 **Responsible Process（宿主终端/IDE）** 来评估 `kTCCServiceAppleEvents`（Automation）权限：
   - 如果宿主终端/IDE 的 `Info.plist` 缺少 `NSAppleEventsUsageDescription`，或者开启了 Hardened Runtime 但缺少 `com.apple.security.automation.apple-events` entitlement，**`tccd` 会直接拒绝弹窗**。
   - 表现就是：`prompt=false` 返回 `-1744 (prompt_required)`，但点击申请 (`prompt=true`) 会在 `0~2ms` 内立刻返回 `-1743 (errAEEventNotPermitted)`，**不会弹出系统授权框，也不会在 `System Settings -> Privacy & Security -> Automation` 中添加任何记录**（与 Accessibility 不同，Accessibility 不要求 `NSAppleEventsUsageDescription`）。

2. **`macos-ax-helper` 与 `p2p-remote-controller` 缺少 Apple Events Entitlement**  
   - 此前 `macos-ax-helper` 没有嵌入 `Info.plist`（日志中 `bundleId=nil`），也没有签署 `com.apple.security.automation.apple-events` entitlement。
   - `apps/p2p-remote-controller/src-tauri/tauri.conf.json` 未配置 `bundle.macOS.entitlements`。

3. **前端状态被 `refreshStatus()` 立即覆盖**  
   在 `apps/p2p-remote-controller/src/App.tsx` 的 `handleRequestAutomation` 中，当 `checkAutomationPermission(true)` 返回 `denied (-1743)` 后，紧接着调用了 `refreshStatus()`，后者又调用 `checkAutomationPermission(false)` 返回 `-1744 (prompt_required)`，把界面上的 `denied` 状态瞬间覆盖回 `Automation Prompt Needed`。

---

## 已提交的修复

已向分支 `z/p2p-remote-controller-monorepo` 提交 Verified Bot Commit [`3fe0901`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/3fe0901385abe1c06ffced88342b54234035e0d7)：

1. **添加 Apple Events Entitlements**：
   - 新增 `packages/macos-ax-helper/Entitlements.plist` 与 `apps/p2p-remote-controller/src-tauri/Entitlements.plist`，声明 `com.apple.security.automation.apple-events = true`。
   - 在 `apps/p2p-remote-controller/src-tauri/tauri.conf.json` 的 `bundle.macOS` 下配置 `"entitlements": "Entitlements.plist"`。
   - 更新 `scripts/buildAxHelper.sh`，使用 `--entitlements` 对 `macos-ax-helper` 进行签名。
   - 更新 `packages/macos-ax-helper/Package.swift`，将 `-sectcreate __TEXT __info_plist` 链接参数限定在 `.when(platforms: [.macOS])`。

2. **精准识别 TCC 静默拦截 (`-1743` vs `-1744`)**：
   - 更新 `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift` 的 `checkAutomationPermission(prompt:)`：当 `prompt=true` 返回 `-1743` 时，立即用 `prompt=false` 探测；若仍返回 `-1744`，说明是 macOS TCC 因宿主终端/IDE 缺少 `NSAppleEventsUsageDescription` 或 `com.apple.security.automation.apple-events` 而静默拦截了弹窗，并在日志中记录 `elapsedMs` 与明确提示。

3. **保留 UI 中的 `denied` 状态**：
   - 更新 `apps/p2p-remote-controller/src/App.tsx` 的 `handleRequestAutomation`：当 `checkAutomationPermission(true)` 返回 `denied` 时不再调用 `refreshStatus()` 覆盖状态，而是保留 `Automation Denied (-1743)`、显示具体原因并提供 `Open Automation Settings` 按钮。

---

## 本机验证建议（macOS）

1. **在 macOS 自带 `Terminal.app` 中运行**（`Terminal.app` 自带 `NSAppleEventsUsageDescription`），或直接打包运行独立的 `.app`（LaunchServices 会将 `P2P Remote Controller.app` 本身作为 Responsible Process）：
   ```bash
   pnpm p2p:build:app
   open apps/p2p-remote-controller/src-tauri/target/release/bundle/macos/P2P\ Remote\ Controller.app
   ```
2. 若需确认是宿主终端/IDE 被 `tccd` 拦截，可在触发授权后运行：
   ```bash
   log show --last 2m --predicate 'subsystem == "com.apple.TCC" AND eventMessage CONTAINS "AppleEvents"' --style compact
   ```
