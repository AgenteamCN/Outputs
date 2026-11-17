# P2P Remote Controller 架构设计与技术可行性评估报告

> **基准环境**：Apple Silicon Mac（`arm64` only），最低系统要求 `macOS Sequoia 15.7.5`  
> **基准仓库**：`Antigravi-ty/agyPreviewInstant`（当前为 Tauri v2 + React 18 + TypeScript + Vite + Rust `hyper` LAN Server）

---

## 一、核心结论速览（TL;DR）

| 问题 | 结论 | 说明 |
| :--- | :--- | :--- |
| **是否需要放弃 Tauri 全面转向原生 Swift / Xcode？** | **不需要，强烈推荐「Tauri 主体 + SwiftPM Sidecar」复合架构** | 既能保留当前仓库的 React/Vite/DevTools、LAN Server、WebRTC/PWA 调试能力与 Preset 共享，又能用原生 Swift 优雅调用 `AXUIElement` 与 Safari AppleScript。 |
| **是否需要学习复杂的 Xcode GUI / `.xcodeproj` 工作流？** | **不需要** | 使用 **Swift Package Manager (`Package.swift`)** 编译为独立 `arm64` CLI/Daemon 作为 Tauri `externalBin` Sidecar，直接通过 `pnpm` 脚本与 Git 管理，无需维护 `.xcodeproj`。 |
| **`AXUIElement` 能否在完全不干扰鼠标/焦点的情况下操作任意网页？** | **不完全可行（需明确边界）** | **读取**页面树完全无干扰；**点击**标准控件可通过 `kAXPressAction` 不动鼠标，但复杂 SPA 的键盘输入、Hover、部分自定义组件仍可能依赖窗口焦点或 `CGEvent`，无法保证 100% 后台无感操作任意网站。 |
| **能否把 Safari 放到独立桌面（Space）并在后台自动激活/保活，同时电脑前台正常干别的？** | **不行** | macOS 全局同一时刻只有一个活跃 Space 与焦点窗口。后台强制激活另一个 Space 的 Safari 窗口会导致**屏幕自动切过去**，直接打断你当前的工作。 |
| **如何精准追踪我们自己打开的页面（应对标题变化、Redirect、SPA 路由变化、Popup 弹窗）？** | **可行，通过「独立 Safari 窗口 + `window id` (`CGWindowID`)」追踪** | 不要在用户现有窗口开标签页，而是由程序新建一个**专属 Safari 窗口**。窗口 ID 在 Redirect、标题变化、SPA 跳转时均保持不变；Popup 可通过监听新增 Tab 或新增 `window id` 自动接管。 |

---

## 二、仓库现状与 Monorepo 演进方案

通过检查 `Antigravi-ty/agyPreviewInstant` 现有结构：
- 根目录已有 `pnpm-workspace.yaml`，技术栈为 **Tauri v2 (`2.2.0`) + React 18 + Zustand + `lucide-react` (`1.64.0`) + `hyper` HTTP/WS LAN Server**。
- 配置与预设（如 `agy_private_profile.json`、`agy_lan_config.json`、`agy_credentials.json`）通过 `src-tauri/src/storage/paths.rs` 持久化在系统 `app_config_dir` 中，`src-tauri/Cargo.toml` 已默认开启 `devtools` feature。

### 推荐 Monorepo 目录结构

```text
agyPreviewInstant/
├── pnpm-workspace.yaml
├── Cargo.toml                        # Rust Cargo Workspace（共享核心 crate）
├── apps/
│   ├── agy-preview-instant/          # 原应用（独立启动命令，独立图标）
│   └── p2p-remote-controller/        # 新 Tauri 应用（独立启动命令，monitor-cloud 图标）
│       ├── src/                      # React Debug UI + 移动端画面模拟器 + Quick Copy
│       └── src-tauri/                # Tauri v2 主进程 + WebRTC/Signaling + Sidecar 调度
├── packages/
│   ├── shared-config/                # 共享 Preset、Tag、Title、类型定义（TS）
│   ├── shared-rust/                  # 共享存储路径解析、Preset 读写（Rust）
│   └── macos-ax-helper/              # 纯 SwiftPM 原生辅助进程（AXUIElement + Safari 控制）
```

### 双应用共存与独立启动
- 两个 Tauri 应用拥有各自独立的 `productName`、`identifier`（如 `com.antigravity.p2premotecontroller`）和打包图标，可以单独启动，也可以同时运行。
- **托盘图标**：使用 `lucide-react` 的 `monitor-cloud`（`MonitorCloud`）SVG 导出为 macOS 菜单栏专用的黑白 Template PNG（启用 `icon_as_template(true)`，自动适配深色/浅色菜单栏）。
- **控制台与 Debug 能力**：
  1. **托盘右键菜单**：提供 `Show/Hide Window`、`Open DevTools (Console)`、`Restart AX Helper`、`Quit`。
  2. **页面内控制**：由于 `Cargo.toml` 已开启 `tauri/devtools`，页面可直接通过快捷键或按钮调用 `openDevTools()`；提供 **Quick Copy** 按钮，一键将抓取到的 `AXUIElement` 树结构、Safari 窗口/Tab 元数据、错误日志格式化为 JSON/Markdown 复制到剪贴板。
  3. **Rust 后端终端选择**：保留 CLI 参数或交互式启动菜单，支持指定监听端口、Signaling Server 地址或目标浏览器模式。

---

## 三、macOS `AXUIElement` 机制与真实边界评估

### 1. 权限申请与开发者签名坑
- **申请方式**：调用 `AXIsProcessTrustedWithOptions([kAXTrustedCheckOptionPrompt: true])`，系统会弹窗引导在 **系统设置 -> 隐私与安全性 -> 辅助功能** 中授权。
- **开发期注意事项**：`cargo tauri dev` 或频繁重编译时，如果二进制仅做 ad-hoc 签名（`-`），部分 macOS 版本可能出现 TCC 权限失效需要重新勾选。建议在本地钥匙串创建一个自签名代码签名证书（或使用 Apple Development 证书）固定签名身份。

### 2. 「不干扰鼠标/工作」的可行性拆解（实话实说）
1. **读取页面内容（完全无干扰）**：
   - 通过 `AXUIElementCreateApplication(safariPid)` 读取 Safari 的窗口列表、`AXWebArea`、按钮、链接、输入框、文本内容与坐标，**100% 不会移动鼠标、不会抢占焦点、不影响你干别的活**。
2. **模拟操作（存在明显限制）**：
   - **能后台无感做的**：对标准原生控件、常规 `<button>` / `<a>` 调用 `AXUIElementPerformAction(element, kAXPressAction)`，不会移动物理鼠标。
   - **做不到稳定后台无感的**：
     - 现代 React/Vue SPA 中，大量自定义 `<div>` 点击事件、Hover 菜单、Canvas 交互对 `kAXPressAction` 响应不稳定。
     - **文本输入**：通过 `kAXValueAttribute` 直接改值经常**无法触发网页 JS 的 `onInput` / `onChange` / `keydown` 事件**；若改用 `CGEventPost` 模拟键盘敲击，则通常要求目标窗口处于激活（Focused）状态。
     - **总结**：简单表单和链接点击可以尝试后台 `AXPress`，但**不要对「全自动无感后台操作任意复杂网页」抱 100% 幻想**。当遇到不响应 `kAXPressAction` 的页面时，需要有降级策略（例如短暂激活窗口执行点击后切回，或提示用户）。

---

## 四、Safari 保活机制、多标签页与独立桌面（Space）问题

### 1. Safari 页面保活机制（为什么后台会失活？）
在 Apple Silicon macOS 上，Safari / WebKit 为了省电有严格的后台节流策略：
- **后台 Tab（同一窗口未选中的标签页）**：JS `setTimeout`/`setInterval` 被大幅降频或冻结，`requestAnimationFrame` 彻底暂停；一段时间后对应的 `com.apple.WebKit.WebContent` 进程会被挂起，导致 **`AXWebArea` 无障碍树变空或数据陈旧**。
- **最小化窗口（Minimized）**：窗口最小化后渲染管线暂停，同样会触发休眠。
- **App Nap**：当 Safari 没有任何可见窗口时，macOS 会触发 App Nap。

### 2. 独立桌面（Space）后台轮流激活可行吗？
- **结论：不行。**
- 原因一：macOS 同一时刻全局只能有一个活跃 Space 和一个焦点窗口。如果你把 Safari 放在桌面 2，后台脚本一旦调用 `activate` 或 `AXRaise` 聚焦它，**系统会强制把你的屏幕从桌面 1 切到桌面 2**，严重打断你的正常工作。
- 原因二：macOS 没有提供跨 Space 创建或静默移动窗口的公开 API（只有易失效的私有 CGS API）。

### 3. 推荐的务实替代方案
- **独立窗口 + 同桌面不最小化（遮挡可接受）**：
  - 让程序通过 AppleScript/JXA 为自己新建一个**独立的 Safari 窗口**（而不是塞进你正在用的窗口里做标签页）。
  - 只要这个窗口**不最小化**（可以放在屏幕边缘，或者直接被你当前的工作窗口盖在下面），WebKit 通常会认为该窗口的当前活跃 Tab 仍处于「可见窗口容器」中，配合必要时的 `ProcessInfo` 防休眠或轻量心跳，可用性远高于后台 Tab。
- **把「把某个标签页拽出来」作为可选动作**：通过 Safari 脚本接口或读取当前 URL 后在新窗口打开，即可将目标页面隔离到独立窗口。

---

## 五、页面追踪、过滤隔离与 Redirect / Popup 处理机制

### 1. 页面到底靠什么追踪？（不是靠 Title，也不是靠 URL）
- 网页 Title 会变，SPA 的 URL（History API `pushState`）也会变，因此**不能用 Title 或 URL 作为唯一标识**。
- 正确的追踪锚点是 **Safari 窗口 ID（`window id` / `CGWindowID`）**：
  - 在 Safari AppleScript/ScriptingBridge 中，每个窗口都有一个唯一的整数 `id`（直到窗口关闭前恒定不变）。
  - 在 `AXUIElement` 中，可通过桥接获取对应的 `CGWindowID`。
  - **特点**：无论页面发生 302 Redirect、前端路由跳转、还是动态修改 `<title>`，**这个 `window id` 始终不变**。

### 2. 隔离流程与 Popup / Redirect 跟随设计
1. **启动检查**：检测 Safari 是否在运行。若未运行，弹窗询问用户是否启动 Safari；若已运行，询问是否允许新建一个**受控专属 Safari 窗口**。
2. **白名单绑定**：记录该新建窗口的 `window_id` 为 `managedWindowIds` 集合。扫描时**只读取 `managedWindowIds` 内的窗口**，彻底过滤掉用户自己打开的所有其他 Safari 窗口与标签页（避免误读隐私数据，也避免扫描大量无关节点导致卡顿）。
3. **Redirect 跟随**：同一窗口内重定向，无需额外处理，`window_id` 不变，重新读取 `AXWebArea` 即可。
4. **Popup 弹窗跟随**：
   - 情况 A（弹窗变成了该窗口内的新 Tab）：监测到受控窗口的 `tabCount` 增加，自动将焦点指针切换到最新 Tab。
   - 情况 B（弹窗打开了新的 Safari 窗口）：在执行可能触发弹窗的操作前后，对比 Safari 的 `window_id` 列表；若紧随操作后出现新 `window_id`，自动将其纳入 `managedWindowIds` 进行接管。

---

## 六、技术栈选型结论：为什么是「Tauri (Rust) + SwiftPM Sidecar」？

### 1. 为什么不建议纯 Rust 调用 `AXUIElement`？
- `AXUIElement` 属于 C/CoreFoundation 层 API，充满 `CFTypeRef`、`AXValueRef` 手动内存管理与回调桥接。在 Rust 中通过 FFI 编写不仅代码冗长、容易内存泄漏，而且调试极其痛苦。
- 在 **Swift** 中，`ApplicationServices` + `AppKit` + `NSAppleScript` 是原生一等公民，代码量仅为 Rust FFI 的 1/5，且可直接配合 Xcode 自带的 **Accessibility Inspector** 进行可视化调试。

### 2. 为什么不建议完全放弃 Tauri 改用纯 Swift App？
- **调试与预览需求**：你的核心需求之一是在桌面端预览/模拟移动端 PWA 拿到的画面与数据结构，使用 **Tauri + React + DevTools** 可以直接复用浏览器控制台、DOM 调试和现有仓库组件。
- **WebRTC + 私有协议 + PWA + Signaling Server**：
  - 现有仓库已经有一套稳定的 Rust `hyper` LAN Server；
  - 后续你的手机端打算采用 **PWA + Service Worker 离线缓存 UI 重建**，桌面端作为 WebRTC Peer 常驻连接 Signaling Server。这些网络与前端编解码逻辑放在 **Tauri (Rust + TypeScript)** 中开发效率最高，且能直接与 `AGY Preview Instant` 共享 Preset、Tag、Title 等配置。

### 3. 复合架构通信方式
- 在 `packages/macos-ax-helper/` 下建立一个纯 Swift 命令行工具（`SwiftPM`）：
  - 负责：`AXUIElement` 权限检查、Safari 窗口创建与生命周期管理、`AXWebArea` 树抓取与精简序列化、执行 `AXPress` 等动作。
  - 通信：由 Tauri Rust 后端以 **Sidecar (`externalBin`)** 形式拉起，通过 **标准输入输出（JSON-RPC / NDJSON）** 或本地 Unix Domain Socket 与 Rust 高速通信。
  - **优点**：完全解耦。Swift 只管「系统无障碍层抓数据与执行动作」，Rust/TS 只管「WebRTC 信令、私有协议编解码、PWA 数据推送、Debug UI 展示」。

---

## 七、Xcode 与开发环境准备清单（面向初次使用者）

既然你之前没有使用过 Xcode 的 Git 编译运行工作流，采用上述 **Tauri + SwiftPM Sidecar** 方案可以让你**几乎不需要接触复杂的 Xcode 工程配置**：

1. **安装工具**：
   - 从 Mac App Store 安装 **Xcode**（或先运行 `xcode-select --install`，但推荐安装完整 Xcode，因为内置了调试无障碍树必备的 **Accessibility Inspector**）。
   - 安装后运行一次 `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`。
2. **日常开发与 Git 工作流（完全命令行化）**：
   - 你依然只需要 `git clone` -> `pnpm install` -> `pnpm tauri dev`。
   - `packages/macos-ax-helper` 通过 `Package.swift` 管理，构建脚本只需执行：
     ```bash
     swift build -c release --arch arm64
     ```
     即可产出 Apple Silicon 原生二进制文件，不需要打开 Xcode GUI 点 Run，也不会产生难解的 `.xcodeproj` Git 冲突。
3. **何时才需要打开 Xcode？**
   - **仅用于调试工具**：打开 `Xcode -> Open Developer Tool -> Accessibility Inspector`，用鼠标指着 Safari 页面，即可实时看到每个元素的 `AXRole`、`AXTitle`、`AXIdentifier` 与支持的操作，极大加速开发。
   - **如果真想在 Xcode 里单步调试 Swift 代码**：直接在终端执行 `open packages/macos-ax-helper/Package.swift`，Xcode 会自动识别并打开它。
4. **最低系统要求利用**：
   - 锁定 `macOS 15.0+` (`Sequoia 15.7.5`) 与 `arm64` 后，无需兼容 Intel (x86_64) 和老旧 macOS API，Swift 并发（`async/await`）与最新系统框架均可直接使用。

---

## 八、建议的下一步行动路线（Phase 1）

1. **Monorepo 脚手架搭建**：将现有应用迁移/组织为 `pnpm + Cargo` workspace，新增 `P2P Remote Controller` Tauri 应用，配置 `monitor-cloud` 托盘图标与右键 Console 菜单。
2. **Swift AX Helper 原型**：创建 `macos-ax-helper`（`arm64`, macOS 15+），实现：
   - 申请 `AXUIElement` 权限；
   - 检测/启动 Safari 并新建受控专属窗口，返回 `window_id`；
   - 抓取受控窗口内 `AXWebArea` 的精简 JSON 树（过滤无关节点）。
3. **Debug 界面与 Quick Copy**：在 `P2P Remote Controller` 窗口中展示抓到的 Safari 页面结构树，提供 **Quick Copy** 按钮，你实际拿几个常用网站（含登录态、SPA、弹窗）抓一次数据发给我，我们再根据真实节点数据设计私有协议与 WebRTC/PWA 端 UI 重建方案。
