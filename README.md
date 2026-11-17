# Apple Silicon Mac 基于无障碍/浏览器扩展远控 Google AI Studio 技术可行性与架构报告

## 一、 核心结论速览

1. **Safari 标签页“最小权限隔离”可行性**：
   - **纯 macOS 无障碍模式（`AXUIElement`）无法做到标签页级权限隔离**。macOS 的辅助功能权限（`kTCCServiceAccessibility`）是**进程级/系统级**的：一旦授权，你的应用就能读取并操作 Safari 的所有窗口和标签页（乃至系统其他应用），OS 层面不支持“只管自己打开的标签页”。
   - **最优替代方案**：采用 **Safari Web Extension（Manifest V3）+ Native Messaging** 作为主力控制层。Safari 原生支持**按域名精细授权**（用户仅需勾选允许扩展访问 `aistudio.google.com`），从权限模型上彻底隔离其他个人标签页，同时完全复用真实 Safari 会话，无需开启开发者菜单、无 Playwright/CDP 指纹。
2. **Google AI Studio 无障碍匹配度**：
   - 静态/基础控件（新建对话按钮、System Instructions 文本框、Get API Key 入口）通常具备较规范的 ARIA/按钮语义，匹配度**高**；
   - 动态弹出层（如 **Model 选择器**下拉列表、项目选择弹窗）多挂载在页面末尾的 Overlay 容器中，若用纯 `AXUIElement` 跨进程遍历会比较脆弱且延迟高；若用 Safari 扩展结合语义选择器（`aria-label` / `role`）操作，稳定性显著提升。
3. **技术栈选型建议（是否用 Tauri？）**：
   - **不建议以 Tauri 作为核心宿主**。你的 Mac 端本质是一个**常驻后台/菜单栏控制服务 + WebRTC P2P Server**，重心在 macOS 原生 API（`AXUIElement` / `SafariServices` / 防休眠）而非桌面端复杂 UI。
   - **推荐架构**：**原生 Swift（Menu Bar App）+ Safari Web Extension（按域隔离控制）+ WebRTC DataChannel（对接现有 ICE Signaling Server）+ 移动端 Web/PWA（重排版 UI）**。

---

## 二、 关键技术点深度评估

### 1. macOS 无障碍模式（`AXUIElement`）与 Safari 隔离能力

在 Apple Silicon Mac 上，macOS 原生提供 `ApplicationServices` 框架下的 Accessibility API（`AXUIElement`）[1.2.2, 1.2.3]。但在控制 Safari 网页时，存在以下硬性约束：

| 评估维度 | 方案 A：纯 `AXUIElement` 驱动 Safari | 方案 B：内嵌 `WKWebView`（自管浏览器容器） | 方案 C：Safari Web Extension + 原生宿主（推荐） |
| :--- | :--- | :--- | :--- |
| **标签页隔离能力** | **无隔离**。TCC 辅助功能权限一经授予，可访问 Safari 全部窗口与标签页。 | **完全隔离**。应用仅能访问自身进程内的 WebView，不碰用户 Safari。 | **原生按域名隔离**。用户在 Safari 中可仅授权 `aistudio.google.com` [1.1.3, 1.1.6]。 |
| **Google 登录与风控** | 复用用户真实 Safari 登录态，无自动化指纹。 | **严重踩坑**：Google OAuth 明确拦截内嵌 WebView（报 `403: disallowed_useragent`）[3.3.11, 3.3.14]，且不与 Safari 共享 Cookie。 | 复用真实 Safari 会话与 Passkey/Cookie，零风控指纹。 |
| **后台标签页可读性** | **仅限当前激活标签页**。后台标签页未渲染 `AXWebArea`，无法实时读取。 | 可后台保活多个 WebView 实例。 | 可通过 Extension Background/Content Script 精准绑定特定 Tab ID。 |
| **DOM/元素操作精度** | 跨进程树遍历较慢，复杂 SPA 易超时，地址栏 URL 读取脆弱 [1.1.1, 1.2.11]。 | 进程内直接执行，精度 100%。 | 直接读取标准 DOM 与 ARIA 属性，精度 100%，响应毫秒级。 |
| **系统权限门槛** | 需引导用户在“系统设置 -> 隐私与安全 -> 辅助功能”授权，且重签名易导致权限失效 [1.1.7, 1.4.8]。 | 无需辅助功能权限。 | 仅需在 Safari 扩展设置中勾选启用并授权目标域名。 |

> **小结**：如果你希望“由应用自己打开或接管特定页面，且绝不触碰用户其他隐私标签页”，**方案 C（Safari Web Extension + 原生 Swift 宿主）**是目前 macOS 生态下唯一同时满足**最小权限隔离**与**顺利通过 Google 登录校验**的标准方案。

---

### 2. Google AI Studio (`aistudio.google.com`) 无障碍与可操作性评估

#### 观察边界说明
我们在当前沙盒环境中实测访问 `https://aistudio.google.com/`，服务端返回 `302` 重定向至 `https://aistudio.google.com/welcome`，完整的 Studio 工作台界面需要 Google 账号登录后才会加载前端应用主体。因此，以下评估基于可验证的公开结构文档与 Google 现代 Web 应用（Angular / Material 体系）的通用无障碍特征给出：

#### 核心操作点匹配度拆解

1. **新建对话 / Prompt（New Chat / + 按钮）—— 匹配度：高**
   - **页面位置**：左侧边栏顶部或主工作区入口 [3.2.6, 3.2.7]。
   - **无障碍特征**：标准 `<button>` 元素，通常带有明确的 `aria-label` 或文本标签（映射为 AX 树的 `AXButton`）。
   - **操作方式**：无论是 `AXPress` 还是扩展内调用 `.click()`，均可稳定触发。

2. **System Instructions（系统指令）—— 匹配度：高**
   - **页面位置**：工作区顶部/右侧 Run settings 面板入口（点击展开指令区域）[1.3.1, 3.2.5]。
   - **无障碍特征**：展开后包含标准多行文本输入区（映射为 `AXTextArea` / `<textarea>`）。
   - **操作注意**：若在单页应用（SPA）中通过扩展或无障碍写入文本，不能只改 `value` 属性，需确保触发标准的 `input` / `change` 事件（或通过原生输入法/焦点写入），否则 Angular 响应式表单状态不会同步更新。

3. **Model（模型）选择器 —— 匹配度：中（纯 AX 较脆弱 / 扩展较稳）**
   - **页面位置**：右侧控制面板顶部 [3.2.5, 3.2.11]。
   - **无障碍特征**：通常由触发按钮（`role="combobox"` 或 `button`）与动态弹出的选项列表（Overlay 浮层）组成。
   - **难点**：弹出层在 DOM 中往往挂载于根节点末尾的浮层容器，**不在触发按钮的子树下**。若用 `AXUIElement` 遍历，需要在点击后重新扫描整个 `AXWebArea` 末尾新出现的列表项；而用 Safari 扩展直接监听 DOM 变化或查询 `[role="option"]` 则非常可靠。

4. **API Key 选择与管理（Get API Key / Create API Key）—— 匹配度：中**
   - **页面位置**：左侧边栏底部 `"Get API key"` 入口 [3.2.7]，进入后涉及 Cloud Project 关联与密钥列表 [3.1.1]。
   - **难点**：涉及弹窗确认与动态表格，部分敏感操作（如复制 Key、创建新项目）建议保留“二次确认”或仅在扩展中读取当前关联状态，复杂项目创建建议留给桌面端人工处理。

---

### 3. 技术栈选型：为什么不建议继续用 Tauri？

| 对比维度 | Tauri (Rust + WebView) | 原生 Swift + Safari Web Extension（推荐） |
| :--- | :--- | :--- |
| **macOS API 亲和度** | 弱。Rust 调用 `AXUIElement`、`NSWorkspace`、`SafariServices` 需大量 Objective-C/Swift FFI 桥接，维护成本高 [1.4.6]。 | **原生一等公民**。直接调用所有 macOS 系统框架与 Safari 扩展通信协议（`NSExtensionRequestHandling`）。 |
| **TCC 签名与权限稳定性** | 开发调试期二进制签名变化易导致辅助功能权限频繁失效 [1.1.7, 1.4.8]。 | Xcode 标准签名与 Bundle 管理，权限状态稳定；且若用 Safari 扩展为主，甚至可免去全局辅助功能权限。 |
| **常驻服务资源占用** | 自带一套桌面 WebView 窗口进程（尽管你真正要用的是手机端 UI，Mac 端并不需要重型界面）。 | 极轻量级菜单栏常驻应用（`MenuBarExtra` / `LSUIElement`），Apple Silicon 上内存占用极低。 |
| **WebRTC 集成** | Rust 侧（`webrtc-rs`）或前端 WebView 内跑 WebRTC（需保活隐藏窗口）。 | 原生集成 `WebRTC`（SwiftPM `stasel/WebRTC`）直接连你的 ICE Signaling Server，后台断线重连更可控。 |

> **补充折中方案**：如果你的团队非常熟悉 TypeScript/Node.js，也可以采用 **“极简 Swift 壳（仅负责 Safari 扩展宿主 / 菜单栏保活） + 本地 WebSocket 桥接 Node/Rust 服务跑 WebRTC”**。但无论如何，不建议为了一个无桌面 UI 需求的常驻服务去引入完整 Tauri 框架。

---

## 三、 结合 WebRTC P2P 的推荐系统架构

为了实现“手机/平板在外部网络远程连接家里 Mac，并对 AI Studio 页面进行移动端重新布局”，建议采用**“语义状态同步 + 指令下发”**模式（走 `RTCDataChannel`），而**不是**传输桌面视频流（视频流在手机端依然是桌面排版，体验差且费流量）：

```text
[手机/平板 移动端 Web/PWA]  <=== WebRTC RTCDataChannel (双向 JSON) ===>  [Mac 常驻 Swift Server]
  (移动端原生重排版 UI)          (经由你现有的 ICE Signaling Server 建连)     ├── WebRTC Engine (P2P 常驻)
                                                                             ├── 防休眠控制 (IOPMAssertion)
                                                                             └── Native Messaging / 本地 IPC
                                                                                       │
                                                                             [Safari Web Extension]
                                                                               (仅授权 aistudio.google.com)
                                                                                       │
                                                                             [真实 Safari 标签页]
```

### 两阶段落地路线建议

1. **Phase 1：本机验证期（跑通页面读写与隔离）**
   - 用 Xcode 创建一个最小 macOS App + Safari Web Extension。
   - 在 Safari 中仅对该扩展授予 `aistudio.google.com` 权限。
   - 验证扩展脚本能否准确提取页面核心状态（当前模型名、System Instruction 文本、对话输出流、输入框状态）并执行动作（新建对话、切换模型、填入并发送 Prompt）。
   - *（可选兜底）*：测试当 Safari 未打开或目标标签页不在时，由 Swift 宿主调用 `NSWorkspace` 唤起特定 URL 的 Safari 标签页。

2. **Phase 2：常驻 P2P Server 与移动端重排版**
   - 在 Mac 端接入你现有的 **WebRTC P2P ICE Signaling Server**，通过 `RTCDataChannel` 暴露一组语义化 RPC 接口（例如 `getState`、`newChat`、`setSystemInstruction`、`selectModel`、`sendPrompt`）。
   - **关键工程细节——防休眠与锁屏处理**：
     - Mac 合盖或锁屏后，网络与 Safari 标签页渲染可能被系统挂起。需在常驻服务中按需申请 `IOPMAssertionCreateWithName`（阻止系统空闲休眠，或配合插电桌面模式运行）。
     - 若依赖 `MutationObserver` 监听 AI Studio 的流式回复，扩展可实时将增量文本通过 Native Messaging -> Swift -> `RTCDataChannel` 推给手机端，实现近似原生 App 的打字机流式体验。

---

## 四、 合规与使用边界提示

1. **自动化与风控边界**：相比 Playwright/Puppeteer 会注入 `navigator.webdriver` 与 CDP 特征，基于真实 Safari + 个人扩展 + 辅助语义交互的方案在客户端指纹上与正常人工操作一致，不会触发常规反爬特征拦截。但仍需注意控制操作频率，将其定位为**个人辅助交互工具（Human-in-the-loop）**，避免高频并发刷请求，以符合平台服务条款（ToS）的合理使用预期。
2. **关于 API 与网页端的取舍**：如果你的核心诉求只是在手机上使用特定 System Instruction 和模型进行对话，Google 官方提供的 **Gemini API** 本身即可直接接入自建移动端 UI；而本方案的最大价值在于**保留网页端 AI Studio 的特定工作流能力**（如网页端免费额度体验、Prompt 历史记录同步、Build 模式及网页端 API Key 管理等）。

---

## 参考资料
- [1.1.3] [cupertino/docs/safari.md - GitHub](https://github.com/mgcrea/cupertino/blob/main/docs/safari.md)
- [1.1.6] [Safari Extension Dispatch Message & Apple Events](https://github.com/mgcrea/cupertino/blob/main/docs/safari.md)
- [1.1.7] [snappy-ax - macOS Accessibility TCC & Signing](https://skills.snappy.ai/skills/snappy-ax)
- [1.2.2] [AXUIElement.h - Apple Developer Documentation](https://developer.apple.com/documentation/applicationservices/axuielement_h)
- [1.2.11] [Reading Safari/Chrome URL via AXUIElement - Stack Overflow](https://stackoverflow.com/questions/79987651/reliably-reading-the-current-tabs-url-from-chrome-and-safari-via-the-macos-acce)
- [1.3.1] [Google AI Studio quickstart - Gemini API](https://aistudio.google.com/docs/ai-studio-quickstart)
- [1.4.6] [Tauri Issue #12137 - Allow macOS plugins to be written in Swift](https://github.com/tauri-apps/tauri/issues/12137)
- [3.2.5] [Google AI Studio Quick Tutorial](http://larryschrenk.com/MIS112/MIS112-Files/Google%20AI%20Studio%20Tutorial.pdf)
- [3.2.7] [Google AI Studio Tutorial - DataCamp](https://www.datacamp.com/tutorial/google-ai-studio-tutorial)
- [3.3.11] [How to Fix 403 disallowed_useragent in Embedded WebView](https://www.testmuai.com/latest-version/macos-user-agents/)
- [3.3.14] [Using OAuth 2.0 for client-side web applications - Google Developers](https://developers.google.com/youtube/data/live/guides/auth/client-side-web-apps)
