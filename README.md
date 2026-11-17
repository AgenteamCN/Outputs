# Apple Silicon Mac 无障碍自动化操作 Google AI Studio 与 P2P 远程重排可行性报告

## 一、 核心结论速览

| 评估维度 | 结论 | 关键说明 |
| :--- | :--- | :--- |
| **Apple Silicon 无障碍支持** | **完全支持** | macOS 辅助功能 API（`AXUIElement`）与芯片架构无关，M 系列 Mac 与 Intel Mac 一致；但需显式开启 Safari 的 `AXEnhancedUserInterface` 属性才会暴露完整网页 DOM 树。 |
| **“单标签页权限隔离”诉求** | **纯无障碍 API 无法做到** | macOS 的 `Accessibility`（TCC）权限是**进程/全局级**的，一旦授权即可读取整个 Safari。若要严格做到“仅限受控页面、无权访问其他标签页”，需改用 **Safari Web Extension（域名白名单）** 或 **独立 `WKWebView` 容器**。 |
| **Google AI Studio 无障碍匹配度** | **中高（良好）** | 基于 Angular + Angular Material 构建，标准 WAI-ARIA 标签较完整（已验证输入框、运行按钮等均有明确 `aria-label`），但模型下拉框、折叠面板等存在浮层/动态挂载特性，需针对性适配。 |
| **风控与 ToS 对比 Playwright** | **显著优于 Playwright** | 实测 Playwright 驱动 AI Studio 会被后端直接拦截权限；本方案运行在真实用户浏览器/系统辅助通道下，无自动化指纹，属于个人辅助控制与界面重排。 |
| **技术栈建议** | **推荐原生 Swift 后台 + PWA** | 相比 Tauri，Mac 端本质是无头/菜单栏常驻 P2P 服务，用 Swift 写 `AXUIElement` 和 `WebRTC` 桥接成本最低；手机/平板端用 Web/PWA 重排 UI 即可。 |

---

## 二、 权限隔离深度解析：能否“只管自己唤起的标签页”？

你提出了一个非常好的安全边界设想：**“由 App 自己调用 Safari 并管理特定标签页，其他个人标签页它没有权限去 access”**。在 macOS 架构下，不同技术路径的权限边界截然不同：

### 1. 纯 `AXUIElement`（系统级无障碍 API）—— 无法实现物理隔离
* **权限粒度**：macOS「系统设置 → 隐私与安全性 → 辅助功能」授予的是**系统级全局权限**。一旦你的 App 获批，它不仅能读取 Safari 的所有窗口和所有标签页，甚至能读取你电脑上其他任何应用的界面。
* **Safari 网页暴露机制**：默认情况下，Safari 出于性能考虑，不会向普通辅助功能客户端展开网页内部的 `AXWebArea` 树。你的 App 必须先获取 Safari 进程的 `AXUIElement`，并将属性 `AXEnhancedUserInterface` 设为 `true`（模拟 VoiceOver 激活状态），WebKit 才会把网页 DOM 映射为无障碍节点。
* **局限性**：只能在**代码逻辑层**检查窗口标题或 `AXDocument`（当前 URL）是否为 `aistudio.google.com`，无法在**操作系统权限层**阻止它访问其他标签页。

### 2. 推荐的两种“真正权限隔离”方案
如果你希望从机制上确保程序**绝对碰不到其他个人页面**，建议采用以下两种方案之一：

* **方案 A：Safari Web Extension（Native Messaging 通信）—— 最标准的浏览器内隔离**
  * 在扩展的 `manifest.json` 中，将 `host_permissions` 严格限制为 `["https://aistudio.google.com/*"]`。
  * **隔离效果**：浏览器内核强制限制该扩展**只能**注入并读取 Google AI Studio 页面，对你日常打开的网银、邮箱、社交媒体标签页完全无权访问。
  * **与 P2P 联动**：扩展通过 Safari 原生的 `browser.runtime.connectNative()` 与你 Mac 上的常驻原生宿主程序（P2P Server）建立本地双向管道，再通过 WebRTC 转发到手机/平板。
* **方案 B：App 内嵌独立 `WKWebView`（或独立 WebKit 容器）—— 最彻底的进程隔离**
  * 不调用外部日常使用的 Safari，而是由你的 Mac App 自己创建一个独立的 `WKWebView` 窗口来加载 `https://aistudio.google.com`。
  * **隔离效果**：使用独立的 `WKWebsiteDataStore` 存储登录态（只需在你的 App 里登录一次 Google 账号），与外部 Safari **物理隔离**。你的 App 既不需要申请全局无障碍权限，也能通过原生 `WKWebView` 接口（或仅针对自身窗口的无障碍树）精准控制页面。

---

## 三、 Google AI Studio (`aistudio.google.com`) 无障碍标签与操作评估

### 1. 客观访问边界说明
经实测请求 `https://aistudio.google.com/` 与 `https://aistudio.google.com/prompts/new_chat`，服务端在未登录状态下会返回 `HTTP 302` 重定向至 `/welcome` 及 `accounts.google.com` 登录页。因此，**完整工作区内部的实时 DOM/AXTree 需要在已登录的本地浏览器会话中探测**。

### 2. 已知技术栈与无障碍特征分析
Google AI Studio 前端采用 **Angular + Angular Material（`@angular/cdk/a11y`）** 构建。Angular Material 对 WAI-ARIA 规范遵循度较高，核心交互控件普遍具备语义化角色（`role`）和 `aria-label`。结合社区对 AI Studio 页面的实际 DOM 观测记录，已可确认部分关键控件的无障碍特征：

* **对话输入框**：`textarea[aria-label="Type something"]` → 在 macOS `AXUIElement` 中映射为 `AXTextArea`（`AXDescription` 为 `"Type something"`）。
* **运行/发送按钮**：`button[aria-label="Run"]` → 映射为 `AXButton`。*(注：在 Angular 表单下，若直接修改输入框文本而未触发原生键盘输入或 `input` 事件，Run 按钮会保持 `disabled` 状态；使用无障碍 `AXValue` 写入或系统级输入可正常激活，或需触发对应事件。)*
* **单条消息选项**：`ms-chat-turn` 容器内的 `button[aria-label="Open options"]` → 映射为 `AXButton`。
* **弹窗关闭按钮**：`button[aria-label="close"]` → 映射为 `AXButton`。

### 3. 你关心的 4 项核心操作可行性与适配难点

| 目标操作 | 无障碍/DOM 定位难度 | 预期映射与实操注意点 |
| :--- | :---: | :--- |
| **1. 新建（New Prompt / Chat）** | **低（极易）** | 位于左侧边栏，通常为标准 `AXButton` 或 `AXLink`（带有 `"Create new prompt"` / `"New chat"` 等标签），直接执行 `AXPress` 即可触发。 |
| **2. 调整 Model（模型切换）** | **中（有浮层坑）** | 位于右侧配置栏，通常为 `mat-select` / `combobox` 按钮。**关键坑点**：点击展开后，模型列表（`role="listbox"` / `mat-option`）由 Angular CDK Overlay **动态挂载到 `<body>` 末尾的独立浮层中**，而不在原按钮的子树里。遍历 `AXUIElement` 时需去 `AXWebArea` 根节点的末尾查找新出现的弹出列表。 |
| **3. 调整 System Instruction** | **中（有折叠态坑）** | 通常是一个可折叠面板（`mat-expansion-panel`，带 `aria-expanded` 属性）或展开后的 `AXTextArea`。**关键坑点**：默认可能处于收起状态，必须先读取其展开状态，若未展开则先 `AXPress` 标题栏展开，再定位内部 `AXTextArea` 写入文本。 |
| **4. 选择 / 管理 API Key** | **低~中** | 通常位于左侧边栏「Get API key」独立路由页面，按钮与列表项具备标准文本/ARIA 标签。因涉及密钥复制与项目关联，建议在移动端重排 UI 上增加二次确认保护。 |

> **快速实测验证方法**：在你的 Mac 上打开 Safari 并登录 Google AI Studio，启动 Xcode 自带的 **Accessibility Inspector**（菜单栏 `Xcode → Open Developer Tool → Accessibility Inspector`），将鼠标悬停在上述 4 个区域的按钮/输入框上，即可直接看到每个元素精确的 `Role`、`Description`、`Identifier` 和层级路径。

---

## 四、 为什么本方案比 Playwright 更不容易触发风控？

社区已有开发者实测证明：**使用 Playwright 启动 Chromium 访问 Google AI Studio 并点击 Run 时，后端会直接返回权限错误（拦截自动化浏览器请求）**。

1. **指纹与协议差异**：Playwright/Puppeteer 依赖 CDP（Chrome DevTools Protocol）注入，存在 `navigator.webdriver`、特定的 JS 运行时特征及非自然的事件时序，处于与反爬/风控系统的“攻防对抗”中。
2. **本方案的本质优势**：
   * 无论是 **macOS `AXUIElement`**、**Safari Web Extension** 还是 **`WKWebView`**，网页都运行在 Apple 原生 WebKit 引擎和你的真实登录会话中。
   * 操作频率是**低频、真人交互级**的（你在手机/平板上点击重排后的按钮 → Mac 执行一次点击或文本填充），不涉及批量并发抓取，本质上是**辅助功能远程控制与跨设备界面重排**，风控风险极低。

---

## 五、 技术栈选型：为什么不一定非要用 Tauri？

你已经拥有了一套 **WebRTC P2P ICE Signaling Server**，整个系统的核心数据流是：

> **手机/平板端（重排后的触控 UI） ⇄ WebRTC DataChannel ⇄ Mac 常驻端（P2P Server） ⇄ Google AI Studio（Safari / WKWebView）**

在这个架构下，**Mac 端其实不需要一个复杂的桌面前端窗口**，它更像一个“带菜单栏图标的常驻后台守护进程（Menu Bar Daemon）”。

### 方案对比：Tauri v2 vs 原生 Swift

| 对比维度 | Tauri v2（Rust + WebView） | 原生 Swift（Menu Bar App）—— **更推荐** |
| :--- | :--- | :--- |
| **macOS 无障碍 API 调用** | 需通过 `axuielement` / `objc2` 等 Rust crate 桥接 C/CoreFoundation 指针，`AXObserver` 内存管理与类型转换踩坑多。 | **一等公民**。直接调用 `ApplicationServices`（或成熟库 `AXSwift`），零桥接开销，文档与 AI 辅助代码最丰富。 |
| **WebRTC P2P 接入** | 依赖 `webrtc-rs`，编译时间较长，部分复杂 ICE/DTLS 边界情况调试成本高。 | 可直接集成官方 `WebRTC.framework`（SPM/CocoaPods）或 LiveKit Swift SDK，Apple Silicon 原生性能优化极佳。 |
| **Safari 扩展 / WKWebView 整合** | 若要搭配 Safari Web Extension（Native Messaging）或原生 `WKWebView` 隔离，仍需写大量 Swift/ObjC 桥接代码。 | Xcode 原生支持同一个 Project 内同时打包 **Mac 主程序 + Safari Web Extension**，直接共用 App Group 数据。 |
| **适用场景** | 你非常精通 Rust，且希望 Mac 端也拥有一套复杂的 React/Vue 本地设置面板。 | **最适合当前需求**：极简常驻服务 + 极致稳定的系统权限与 WebRTC 表现。 |

### 移动端/平板端 UI 建议
* 不需要为 iOS/Android 单独开发原生 App，直接用你熟悉的 **React / Vue / Next.js 写一个响应式 PWA 网页**即可。
* 在手机/平板上，你可以把原本电脑端拥挤的三栏布局彻底重排：顶部固定 Model 切换胶囊、折叠/展开式 System Instruction 抽屉、中间大面积对话流、底部适配单手拇指的输入框与发送大按钮，通过 WebRTC DataChannel 实时同步 Mac 端状态。

---

## 六、 建议的 MVP 落地路线（三步跑通）

1. **第一步：本地无障碍/DOM 探测验证（1 天）**
   * 若走纯无障碍路线：用 Xcode `Accessibility Inspector` 或写一段极简 Swift 脚本，设置 Safari `AXEnhancedUserInterface = true`，打印出 `aistudio.google.com` 的元素树，确认 4 个核心操作（新建、切模型、改 System Instruction、选 API Key）的稳定定位策略。
   * 若走隔离路线：建一个最简 Safari Web Extension，仅配置 `https://aistudio.google.com/*` 权限，验证 Console 脚本读写这 4 个控件的稳定性。
2. **第二步：Mac 常驻 P2P 通道打通（2~3 天）**
   * 在 Mac 端用 Swift（或 Rust/Node）连接你现有的 ICE Signaling Server，建立 WebRTC DataChannel，定义好简单的 JSON 控制协议（例如 `{"cmd": "set_model", "value": "gemini-2.5-pro"}`、`{"cmd": "run_prompt", "text": "..."}`）。
3. **第三步：移动端重排界面联调（2~3 天）**
   * 编写轻量 PWA 控制端，在手机/平板 4G/5G 外网环境下通过你的 P2P Server 连回 Mac，完成“外网发指令 → Mac 端实时操作并回传结果”的完整闭环。
