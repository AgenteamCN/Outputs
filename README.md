# P2P Remote Controller 架构调研与可行性评估报告

> 基于现有仓库 `Antigravi-ty/agyPreviewInstant`（Tauri v2 + React 18 + Vite + TypeScript + pnpm）的代码结构，以及目标运行环境 **Apple Silicon Mac / macOS Sequoia 15.7.5+** 进行评估。

---

## 一、 核心结论速览（TL;DR）

1. **技术栈取舍：强烈推荐「Tauri v2 Monorepo + Swift Sidecar CLI (`ax-bridge`)」混合架构**，而不是纯 Rust 调 AXUIElement，也不建议放弃 Tauri 转纯 SwiftUI/Xcode。
   - **原因**：Rust 生态的 macOS `AXUIElement` 绑定简陋且与 `CFRunLoop` 冲突；而纯 SwiftUI 会丢失 `agyPreviewInstant` 的 React UI 复用、Web DevTools 调试体验，以及未来与 PWA/WebRTC 共享编解码逻辑的便利性。
   - **是否需要用 Xcode IDE**：**不需要**。只需安装 Xcode Command Line Tools 提供 `swift build` 编译器，即可在现有 Git + `pnpm` 工作流中自动编译 Swift Sidecar 二进制，无需切换至 Xcode 工程管理。
2. **「完全不干扰鼠标且后台无感操作任意网页」的可行性：部分可行，但有明确边界，不能 100% 保证。**
   - **读取页面内容与元素树**：`AXUIElement` 可以完全在后台读取，不抢焦点、不动鼠标。
   - **触发点击/输入**：标准 HTML 按钮/链接可通过 `AXPress` 触发且不移动鼠标；但现代 SPA 网页（自定义 `div` 点击、React 受控输入框、`hover` 菜单、Canvas）经常不响应 `AXPress` 或 `AXSetValue`，若回退到 `CGEvent` 模拟键鼠，则会抢占鼠标或窗口焦点。
3. **「独立虚拟桌面（Space）后台激活窗口保活」：不可行。**
   - macOS 只有一个全局键盘/焦点窗口，跨 Space 激活 Safari 窗口会导致桌面切换或打断你当前的工作输入。
   - **正确解法**：由应用主动创建一个**独立的 Safari 专用窗口**（而非混用你的日常标签页），结合 `window_id` 精确追踪与隔离。

---

## 二、 关键技术问题逐项解答

### 1. AXUIElement 权限与「不干扰鼠标/键盘」的真实能力边界

| 能力 | 是否可行 | 技术细节与限制 |
| :--- | :--- | :--- |
| **读取全系统/Safari 页面结构** | ✅ 可行（不抢焦点） | 申请一次 macOS「辅助功能（Accessibility）」权限后，即可通过 `AXUIElementCopyAttributeValue` 遍历 Safari 窗口内的 `AXWebArea` 树，获取角色（Role）、标题、链接、按钮、文本等。 |
| **开启 Safari 网页无障碍树** | ⚠️ 需显式激活属性 | Safari 默认会懒加载网页的无障碍树，必须在代码里对 Safari 的 `AXUIElement` 设置 `AXEnhancedUserInterface = true`，否则 `AXWebArea` 内部经常是空的。 |
| **不移动鼠标点击网页元素** | ⚠️ 部分可行 | 调用 `AXUIElementPerformAction(el, kAXPressAction)` 可以在不移动光标的情况下触发原生/语义化 `<button>`、`<a>`。但对于非语义化 `div` 绑定 `pointerdown` 或复杂 SPA 组件，经常无效。 |
| **不抢焦点向网页输入框打字** | ⚠️ 容易失效 | `AXUIElementSetAttributeValue(el, kAXValueAttribute, ...)` 可直接改值，但 React/Vue 等框架依赖 `input`/`change` 事件，直接改 AXValue 往往无法触发前端状态更新。 |
| **完全后台无感操作** | ❌ 无法 100% 保证 | 一旦遇到 `AXPress` 失效的控件，若要兜底就必须用 `CGEvent` 模拟真实点击/按键，这必然需要窗口可见/聚焦并移动光标。 |

---

### 2. Safari 页面保活机制、标签页拽出与「独立 Space 后台激活」

- **Safari 后台保活与节流机制**：
  - **后台标签页（非当前 Tab）**：WebKit 会严格限制 `setTimeout`/`setInterval` 频率（降频至 1s 甚至休眠），暂停 `requestAnimationFrame`，并触发 `document.hidden = true`。
  - **窗口遮挡 / 最小化（Window Occlusion & App Nap）**：当 Safari 窗口被最小化、或处于另一个不可见的 Space（虚拟桌面）时，macOS WindowServer 会将其标记为不可见，进一步触发渲染暂停与 App Nap。
- **把标签页拽出来（独立窗口）**：
  - **完全可行，且不需要鼠标拖拽**。通过 AppleScript（`NSAppleScript` / `osascript`）可以直接命令 Safari 新建独立窗口打开目标 URL（`make new document with properties {URL: "..."}`），每个独立窗口中只有一个当前 Tab，就能避开多 Tab 切换带来的后台标签页节流。
- **能否单独划一个 Space（桌面），让程序在后台轮流 Focus 窗口保活，同时不影响你正常干活？**
  - **结论：不行。** macOS 的窗口系统不支持“多焦点”。只要后台程序为了唤醒页面而执行 `activate` 或 `AXRaise` 抢占焦点，你的系统焦点就会被夺走（正在打字的窗口失焦，甚至触发 Space 切换动画）。
  - **推荐替代方案**：让自动化专用 Safari 窗口留在当前桌面角落或副屏（哪怕尺寸缩小，只要不最小化、不被完全遮挡），即可维持页面活跃。

---

### 3. 页面追踪与隔离机制：标题和 URL 都会变，到底靠什么追踪？

你提到的痛点非常准确：SPA 路由会让 URL 变，页面加载会让 Title 变，如果全量扫描所有 Safari 窗口会非常混乱。

1. **物理隔离优于事后过滤**：
   - 不建议扫描用户自己打开的所有 Safari 窗口。
   - **推荐流程**：点击「启动/连接目标页」时，检测 Safari 是否运行（`NSRunningApplication`）-> 弹出确认框询问是否启动 Safari 或新建一个**专用独立窗口** -> 仅对该窗口的 ID 进行追踪。
2. **Safari 窗口与标签页的稳定标识**：
   - **窗口级稳定 ID**：通过 AppleScript 可以拿到 Safari 窗口的整数 `id`（或者通过私有/底层函数 `_AXUIElementGetWindow` 将 `AXWindow` 映射到稳定的 `CGWindowID`）。只要这个专用窗口不关闭，**无论内部怎么 redirect、SPA 怎么跳转、Title 怎么改，它的 `window_id` 永远不变**。
   - **弹窗（Popup）与新标签页跟随**：在轮询或 `AXObserver` 监听中，记录该 `window_id` 下已知的 Tab 列表与全局 Safari 窗口 ID 集合；当页面触发 `target="_blank"` 或 `window.open()` 产生新 Tab 或新窗口时，能立刻捕捉到增量 ID，由程序自动跟随或弹窗询问是否纳入接管。

---

## 三、 架构对比与推荐方案：Monorepo + Tauri v2 + Swift Sidecar

### 现有代码基础评估（`agyPreviewInstant`）
目前仓库已经具备非常扎实的基础：
- 前端：React 18 + TypeScript + Vite + `lucide-react`（已内置 `monitor-cloud` 图标）。
- 后端（`src-tauri/src/`）：已实现 `lan_commands.rs`、`lan_http_io.rs`、`lan_ws_codec.rs`、`lan_store.rs` 以及 `load_synced_presets_cmd` / `save_synced_presets_cmd`，并且 `Cargo.toml` 默认开启了 `devtools`。
- 本地存储（`src-tauri/src/storage/paths.rs`）：配置持久化在系统的 `app_config_dir()` 中。

### 为什么推荐「Tauri v2 + Swift Sidecar (`ax-bridge`)」？

| 方案 | 优点 | 缺点 | 推荐度 |
| :--- | :--- | :--- | :---: |
| **纯 Rust 调 AXUIElement** | 单一二进制，无跨语言通信 | Rust 绑定极不成熟，`CFRunLoop` 与 Tokio 难兼容，踩坑极多 | ⭐ |
| **纯原生 SwiftUI / Xcode** | 调 AXUIElement 最顺手 | 无法复用 `agyPreviewInstant` 前端生态；调试网页 UI 麻烦；难与 PWA 共享编解码；需重学 Xcode 工程配置 | ⭐⭐ |
| **Tauri v2 + Swift Sidecar CLI** | **Swift 原生写 AX 逻辑（零桥接坑）+ Rust 管 WebRTC/信令/HTTP 服务 + React 做 Debug 与移动端预览 UI + Monorepo 共享 Presets** | 需多维护一个极轻量的 `Package.swift` 命令行工具（约几百行 Swift） | ⭐⭐⭐⭐⭐ |

### Monorepo 目录规划建议

利用现有的 `pnpm-workspace.yaml`，将仓库演进为多应用 Monorepo：

```text
agyPreviewInstant/
├── apps/
│   ├── preview-instant/          # 原有的 AGY Preview Instant (独立 Tauri 应用 & 图标)
│   └── p2p-remote-controller/    # 新增的 P2P Remote Controller (独立 Tauri 应用 & 图标)
│       ├── src/                  # React Debug UI：AX 树查看、Quick Copy、移动端画面模拟器
│       └── src-tauri/            # Rust 后端：Tray(monitor-cloud)、DevTools、LAN/WebRTC/Signaling
├── native/
│   └── ax-bridge-swift/          # 纯 Swift Package (无 Xcode 工程)，编译为 Tauri Sidecar
│       ├── Package.swift
│       └── Sources/              # AXUIElement 扫描、AXPress 执行、Safari 独立窗口追踪
└── packages/
    └── shared-presets/           # 共享 TypeScript 类型、System Instruction Preset 协议等
```

- **双应用独立启动与共存**：通过不同的 `pnpm` 命令分别启动（例如 `pnpm dev:instant` 与 `pnpm dev:controller`），各自拥有独立的 `tauri.conf.json`、`identifier` 和应用图标。
- **预设（Presets / Tag / Title）共享**：除了在 `packages/shared-presets` 共享 Schema，运行时的个人预设可约定存储在公共目录（如 `~/Library/Application Support/com.antigravity.shared/presets.json`），两个 Tauri 应用均可直接读写互通。
- **Debug 与控制台需求实现**：
  - **Tray 图标**：将 Lucide 的 `monitor-cloud` 导出为单色模板 PNG（macOS Menu Bar 规范），配合 Tauri `TrayIconBuilder` 实现。
  - **控制台与终端**：托盘右键菜单增加 `Open DevTools`（调用 `window.open_devtools()`）；页面内支持右键打开检查器（`devtools` feature 已开启）；在 `tauri dev` 启动时终端可直接交互或接收 Rust 标准输入指令。
  - **Quick Copy**：前端界面提供一键复制按钮，将当前抓取的 AX 树 JSON / 精简 DOM 快照写入剪贴板，方便直接粘贴给 AI 分析下一步操作。

---

## 四、 后续演进路径：WebRTC + 私有协议 + PWA + Signaling

1. **常驻 Signaling 与 WebRTC P2P**：
   - 将 Signaling Server（公网长连接）和 WebRTC DataChannel 放在 **P2P Remote Controller 的 Rust 后端**（如 `webrtc` crate 或 Rust 信令客户端）常驻运行，即使关闭调试主窗口只留 `monitor-cloud` 托盘图标，远程通道依然在线。
2. **PWA + Service Worker 移动端**：
   - 手机端通过 PWA（由 Service Worker 提前离线缓存整套 React 渲染壳子），连上 WebRTC DataChannel 后，Mac 端通过私有紧凑协议发送「精简后的页面可交互元素树」；手机端将其**重绘（Recreate）为适合触屏操作的原生感 UI**，点击后发回指令 ID 给 Mac 执行。

---

## 五、 环境准备与 Git / Xcode 工作流说明（针对 Apple Silicon + Sequoia 15.7.5+）

因为采用的是 **Swift Package Manager (`swift build`) Sidecar** 模式，**你不需要使用 Xcode IDE 来写代码或点 Run/Archive**：

1. **本地提前准备**：
   - 安装 **Xcode Command Line Tools**（`xcode-select --install`），或直接从 App Store 安装 Xcode（仅用于提供 Swift 编译器与 macOS 15 SDK）。
   - **Git 工作流完全不变**：依然是 `git clone` -> `pnpm install` -> `pnpm dev:controller`。构建脚本会自动触发 `swift build -c release` 并把生成的 `ax-bridge` 二进制放到 `src-tauri/binaries/` 下供 Tauri 打包。
2. **macOS Sequoia 15.7.5 开发期权限踩坑预警（重要）**：
   - **辅助功能权限（TCC）重置问题**：Tauri 默认使用 ad-hoc 签名（`"signingIdentity": "-"`）。每次重新编译后，macOS 可能会认为二进制签名变了，导致你必须在「系统设置 -> 隐私与安全性 -> 辅助功能」里删掉重新勾选。
   - **开发期规避技巧**：
     1. 开发调试时，直接把权限授予你的终端（如 iTerm2 / Terminal / VS Code），从终端启动 `pnpm dev` 时子进程会继承终端的辅助功能权限；
     2. 打包 `.app` 时，在本地 macOS「钥匙串访问」创建一个自签名代码签名证书（Self-Signed Certificate），并在 `tauri.conf.json` 中固定 `signingIdentity`，即可一劳永逸避免每次编译重新授权。
   - **Apple Events（控制 Safari）权限**：如果使用 AppleScript 控制 Safari 新建独立窗口，需在 `Info.plist` 中声明 `NSAppleEventsUsageDescription`，首次触发时系统会弹窗请求「允许控制 Safari」。

---

## 六、 建议的下一步行动（Phase 1）

如果认可上述架构结论，建议第一步先做最小可验证闭环（MVP）：
1. 将当前仓库改造为 `apps/preview-instant` 与 `apps/p2p-remote-controller` 的 pnpm Monorepo 结构，配置双命令启动与 `monitor-cloud` 托盘图标（含右键打开 DevTools）。
2. 编写极简的 `native/ax-bridge-swift` Sidecar，验证：**申请辅助功能权限 -> 询问并打开独立 Safari 窗口 -> 实时抓取该窗口的 `AXWebArea` 可点击元素列表**。
3. 在 P2P Remote Controller 界面中展示抓取结果并提供 **Quick Copy** 按钮，由你实际在本地测试几个真实登录后的目标网页，把复制出来的数据发给我，我们再根据实际拿到的无障碍树结构决定下一步的精细解析与 WebRTC 协议设计。
