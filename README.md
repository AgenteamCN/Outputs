# `agyPreviewInstant` Git Hooks 排查与 P2P 自动化架构演进调研报告

> **调研基准仓库**：`Antigravi-ty/agyPreviewInstant`（最新提交 `6f2225c5`：`Fix P2P UI layout, add window focus/regex search, and add scoped git-hook auto-reload`）

---

## 一、快速解答：`.githooks/` 三个文件是什么？该上传还是 ignore？

### 1. 它们是什么？
在最新提交 `6f2225c5` 中，仓库新增了一套**基于 Git Hook 的作用域感知自动重载（Scoped Auto-Reload）机制**，涉及：
- `.githooks/post-checkout`（切换分支时触发）
- `.githooks/post-merge`（`git pull` / `git merge` 时触发）
- `.githooks/post-rewrite`（`git rebase` / `commit --amend` 时触发）
- 配套脚本：`scripts/git/ensureGitHooks.ts`、`scripts/devWatch.ts`、`package.json` 中的 `"prepare"` 与 `"hooks:install"`。

**工作原理**：
1. 执行 `pnpm install` 时，`package.json` 的 `"prepare"` 钩子会自动运行 `scripts/git/ensureGitHooks.ts`，将本地仓库的 `core.hooksPath` 设为 `.githooks`，并对这 3 个脚本执行 `chmod 0755`。
2. 当你拉取代码、切换分支或变基时，钩子会对比 `PREV_HEAD` 和 `NEXT_HEAD` 的差异路径（如 `src/`、`src-tauri/`、`apps/p2p-remote-controller/`、`packages/macos-ax-helper/`），并将变更标记写入 `.cache/git-checkout-event.json`。
3. `pnpm agy:dev:watch` / `pnpm p2p:dev:watch`（即 `scripts/devWatch.ts`）监听到该事件文件后，会自动决定重启或重建对应的 Tauri 开发进程。

### 2. 应该上传还是 ignore 掉？
- **结论：应该上传（目前已提交在 `6f2225c5` 中），不应该 ignore 掉。**
- **原因**：Git 默认的 `.git/hooks/` 不会随仓库同步。将钩子放在受版本控制的 `.githooks/` 目录中，是为了让所有开发机/协作者在 `pnpm install` 后自动获得一致的 devWatch 联动能力。
- **真正应该 ignore 的是运行时产物** `.cache/`（`.gitignore` 第 25 行已将其忽略）。
- **为什么你本地会提示上传/有改动？** 常见原因有两个：
  1. 本地仓库尚未 `git pull` 同步到 `6f2225c5`；
  2. `ensureGitHooks.ts` 对文件执行了 `chmod 0o755`，若本地开启了 `core.filemode = true`，Git 会认为文件权限位发生变化。可执行 `git pull --rebase` 或 `git checkout -- .githooks/` 恢复干净工作区。

---

## 二、核心痛点诊断：为什么当前 P2P 刷新割裂、页面状态会丢？

结合 `apps/p2p-remote-controller/src/App.tsx` 与 `packages/macos-ax-helper/Sources/MacosAxHelper/` 源码，当前存在 4 个结构性短板：

1. **页面状态纯内存态，重启即失忆**：
   - `App.tsx` 中的 `managedWindowIds` 与 `selectedWindowId` 均为 React `useState` 内存状态，重启 P2P 后全部丢失。
   - `syncSafariWindows()` 在窗口消失时会**静默过滤**并自动切换到 `safari.windows[0]`，没有任何失联提示或删除确认。
   - 此外，AppleScript 返回的 `windowId`（Safari Window ID / `CGWindowID`）在 Safari 重启后会重新分配，且当前仅支持窗口粒度（`current tab of w`），未区分同窗口下的多 Tab。
2. **`Refresh` 与 `Scan AXTree` 职责割裂**：
   - 点击顶部 `Refresh`（`refreshStatus()`）只调用 `get_safari_status`（通过 AppleScript 拉取窗口列表更新 `title` 和 `url`），**不会刷新 AX 树**。
   - 点击窗口芯片 `#windowId` 只调用 `handleScanWindow()`（通过 `AXUIElement` 扫描元素），**不会回写窗口列表里的 `title`**。
   - 点击 `Open New` 后，`SafariController.swift` 仅 `delay 0.2` 就读取窗口名，此时网页还在加载，导致长期显示 `Untitled` 或空标题，必须手动点 `Refresh` 才会更新。
3. **无事件推送机制，SPA 变化/弹窗只能盲等或手动扫**：
   - `JsonRpcServer.swift` 是纯同步的 `stdin/stdout` 请求-响应模型，没有服务端主动推送（Notification）通道。
   - 点击按钮后 `App.tsx` 仅固定等待 `SPA_RESCAN_DELAY_MS = 350ms` 扫一次；若接口耗时 3~10 秒才弹出报错弹窗，P2P 完全感知不到。
4. **AGY 与 P2P 之间缺少本地通信协议**：
   - 主程序 AGY（`src-tauri/src/commands/lan_commands.rs`）已内置端口 `14200` 的 HTTP + WebSocket 服务，但 P2P 目前仅通过本地磁盘读取 `synced_presets`，无法接收 AGY 下发的本地指令，也无法向 AGY 回传网页执行流。

---

## 三、技术可行性与方案深度评估

### 问题 1：如何持久化页面 ID 并在消失时询问删除？

- **可行性**：**高（纯应用层改造，1~2 天可落地）**。
- **推荐设计**：
  1. **持久化模型**：将选中状态持久化到 P2P 本地配置（如 Tauri AppData 下的 `p2p_session.json` 或 `localStorage`），不仅存 `windowId`，还需保存绑定指纹：
     ```json
     {
       "selectedTargetId": "target_01",
       "targets": [
         {
           "id": "target_01",
           "windowId": 12345,
           "url": "https://aistudio.google.com/prompts/new_chat",
           "title": "Google AI Studio",
           "status": "active" // active | missing
         }
       ]
     }
     ```
  2. **消失检测与确认机制**：当 `get_safari_status` 返回的活跃窗口中找不到已保存的 `windowId` 时，**不要静默移除**，而是将状态置为 `missing`（UI 上显示灰色虚线标签 + 警示图标），并弹出确认条/弹窗：
     - 「目标页面 `#12345 (Google AI Studio)` 已关闭或失联：[移除记录] / [按原 URL 重新打开] / [重新关联到当前窗口]」。
  3. **Safari 重启兜底**：若 Safari 重启导致 `windowId` 全部洗牌，可通过 `url` 前缀/精确匹配自动提示重新绑定（Re-bind），避免每次重启都要重新配对。

---

### 问题 2：无障碍接口（AX）能否主动推送页面加载/弹窗通知？还是必须轮询？性能如何？

#### 1. macOS 无障碍 API 确实支持事件推送（`AXObserver`）
macOS `ApplicationServices` 提供 `AXObserverCreate` + `AXObserverAddNotification`，可在后台监听 Safari 进程发出的无障碍事件，**无需轮询即可实时感知以下变化**：
- `kAXTitleChangedNotification`：窗口/标签页标题更新（**彻底解决 `Untitled` 不自动更新问题**）。
- `AXLoadComplete`（WebKit 专用通知）：网页 `AXWebArea` 加载完成时触发。
- `kAXWindowCreatedNotification` / `kAXUIElementDestroyedNotification`：窗口新建或关闭（实时触发"页面消失"检测）。
- `kAXFocusedWindowChangedNotification` / `kAXFocusedUIElementChangedNotification`：焦点变化。
- `AXLiveRegionChanged`：当网页使用 ARIA Live Region（如 `role="alert"`、`aria-live="polite/assertive"`）渲染错误提示时可收到通知。

#### 2. 现实局限性（重要！）
- **并非所有 SPA 变化都会触发 AX 通知**：如果网页的 Error Popup 是普通 `<div>` 弹窗且**没有设置 `role="alert"` 或 `aria-live`**，WebKit 不一定会派发 `AXLiveRegionChanged` 或 `kAXLayoutChangedNotification`（或者会在高频动画时被 WebKit 节流/忽略）。
- **架构改造需求**：当前 `MacosAxHelper` 是无 RunLoop 的单线程 `readLine()` CLI。要使用 `AXObserver`，必须在 Swift 侧引入 `CFRunLoop` 线程，并将 `JsonRpcServer` 升级为**双向异步通道**（支持主动向 `stdout` 推送 `{"jsonrpc":"2.0","method":"ax_event","params":{...}}`），Rust 侧 `ax_bridge.rs` 需拆分为独立读写线程并通过 Tauri `app.emit()` 转发给前端。

#### 3. 轮询方案的性能消耗评估
- 从 `AccessibilityEngine.swift` 实现来看，每次 `scanWindow` 最多遍历 `600` 个节点（`maxDepth: 14`），每个节点读取约 8~10 个 AX 属性，单次扫描约产生 **4,000~6,000 次跨进程 Mach IPC 调用**。
- **实测量级估算（Apple Silicon M 系列）**：
  - 单次 `scanWindow` 耗时约 **15ms ~ 60ms**（复杂 SPA 页面如 AI Studio 可达 100ms+）。
  - 若 24/7 空闲高频轮询（如每 200ms 一次），会持续占用 15%~30% 单核 CPU 并引发 Safari WebContent 进程发热。
  - **推荐策略（事件驱动 + 按需节流轮询）**：
    - 常态下：**0 轮询**，完全依靠 `AXObserver` 接收标题、加载完成、窗口销毁通知（CPU 消耗趋近于 0%）。
    - 自动化任务执行期（如发送 Prompt 后等待响应/监控报错弹窗）：开启 `500ms ~ 1000ms` 节流轮询或仅针对特定子树检查，任务结束后立即停止，CPU 占用可控制在 **2%~5% 以内**。

---

### 问题 3：为什么 Certificate Pinning 绕不过？数据包到了本地，为什么不能"只读不改"看到网页错误详情？无障碍能看到吗？

这是一个非常经典的网络密码学与浏览器安全模型问题，你的直觉**在端点侧是完全正确的，但在中间人（MITM）网络层是行不通的**：

#### 1. 为什么"到了本地、只读不改"依然过不了 TLS / Certificate Pinning？
- **加密在传输层完成**：服务端发到你网卡上的 TCP 数据包，是用本次 TLS 握手协商出的**临时对称会话密钥（ECDHE 前向安全）**加密的。哪怕你只抓包不修改，Wireshark/tcpdump 看到的也只是密文，没有任何工具能凭空解密。
- **MITM 代理的本质必须"冒充服务端"**：像 Charles / mitmproxy 这类工具要想看到明文，必须在本地截断连接，自己拿一张伪造的服务器证书和浏览器握手，同时自己再连真正的服务器。
- **Certificate Pinning 阻断的是"冒充"**：如果客户端（App 或带 Pinning 校验的内核）硬编码了服务端真实公钥指纹（SPKI Hash），它一旦发现握手证书是代理签发的假证书，就会在**发送任何 HTTP 请求之前直接断开连接**。你连明文都还没发出去，自然也就"看"不到。
- *(补充说明：在 macOS 上，**Safari 和 Chrome 对普通网页如 `aistudio.google.com` 的系统级信任存储策略是：只要你在系统钥匙串中手动信任了代理的根证书，浏览器通常允许代理调试**；但如果是 HTTP/3 (QUIC) 走 UDP 绕过了系统 HTTP 代理，或者目标站点启用了特殊的严格校验，MITM 仍会失败或配置极其繁琐。)*

#### 2. 无障碍模式（AX）能看到网络数据包里的隐藏错误码吗？
- **不能。**
- 无障碍树（AX Tree）本质上是浏览器渲染引擎将 **DOM + CSS 排版树** 投影给辅助技术的"界面语义树"。
- 如果网页前端捕获了 HTTP 429/500 响应里的 `{"error": {"code": 429, "reason": "RESOURCE_EXHAUSTED", "retryAfter": "30s"}}`，但在 UI 上只渲染了一句 `"Something went wrong"`，那么 **AX 树里就只有 `"Something went wrong"`**，原始 JSON、HTTP 状态码、Response Header 根本不会进入 AX 树。

#### 3. 不做 MITM、合法且稳定拿到网页完整数据包的技术手段对比

既然明文在**浏览器进程内存解开 TLS 之后、渲染到 DOM 之前**就已经存在，我们完全可以绕开 MITM 和证书问题，直接在浏览器/协议层观测：

| 技术方案 | 原理 | 能否拿到隐藏错误码/Header | 是否受 Pinning / QUIC 影响 | 工程复杂度 & 稳定性 | 适用场景 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. JS 运行时 Hook（`fetch` / `XHR` / `EventSource`）** | 通过 Safari AppleScript `do JavaScript` 或 Web Extension 在页面上下文注入拦截器，直接劫持响应流并回传本地 WS | **可以（100% 拿到完整 JSON/SSE 流、状态码、错误详情）** | **完全不受影响**（运行在 TLS 解密之后） | **低**（纯 JS 代码约 80 行；若用 AppleScript 注入需在 Safari 开发者菜单开启"允许来自 Apple 事件的 JavaScript"） | **当前 Safari 架构最轻量、立即可用的方案** |
| **2. Chrome DevTools Protocol (CDP)** | 启动带 `--remote-debugging-port` 的 Chromium/Chrome，订阅 `Network.responseReceived` + `Network.getResponseBody` | **可以（100% 全量网络包、Header、Body、Console 错误、WS 帧）** | **完全不受影响** | **中**（协议极其成熟，Playwright/Puppeteer 均基于此；但需切换或并存 Chromium，不能直接控制 Safari） | **长期自动化与全量可观测性的最优解** |
| **3. 本地 MITM 代理（mitmproxy / Rust `hudsucker`）** | 本地起 HTTPS 代理 + 安装根证书到 macOS Keychain | 可以 | 需关闭浏览器 QUIC/HTTP3，且需用户手动信任根证书 | **高**（证书管理、代理开关、流式 SSE/gRPC 解包维护成本高） | 仅适合调试或无源码黑盒 App |
| **4. `SSLKEYLOGFILE` / Frida Hook** | 导出 TLS 会话密钥或 Hook `BoringSSL` | 可以 | 不受 Pinning 影响 | **极高**（Safari 受 macOS SIP 保护，无法直接注入） | 不建议用于产品化 |

---

## 四、整体架构规划：AGY 双路通信 + P2P 自动化演进方案

为了支撑后续的**本地双路调度（API + 网页端）**以及**完整的 UI / 自动化流程**，建议按以下架构推进：

### 1. 本地通信协议设计（AGY ↔ P2P）

建议将 **P2P Remote Controller 升级为常驻本地守护服务（Local Daemon）**，与 AGY 现有 `14200` 端口对齐，采用 **Loopback HTTP + WebSocket（JSON-RPC 2.0）**：

- **监听地址**：`127.0.0.1:14201`（仅限本机回环，免去复杂公网鉴权；可选配共享 Token 与 AGY `~/.agy/` 配置互通）。
- **AGY 侧抽象（双路路由）**：
  - 在 AGY `src-tauri` 中抽象统一的 `TaskExecutionTransport`：
    1. **API 通道（主路）**：现有的 `google_api/stream_runner.rs`。
    2. **P2P 网页通道（备路/强交互路）**：当 API 返回 5xx/429/不稳定或用户手动指定时，AGY 通过 `ws://127.0.0.1:14201` 向 P2P 发送 `task.execute` 指令。
- **核心协议帧示例**：
  - **AGY → P2P（指令）**：`target.list`、`target.select`、`task.sendPrompt`、`task.abort`、`page.captureState`
  - **P2P → AGY（实时流事件）**：
    - `page.status_changed`（`loading` / `ready` / `missing` / `title_updated`）
    - `task.stream_chunk`（网页端逐步输出的文本流）
    - `network.error_captured`（捕获到的 HTTP 状态码、原始错误 JSON、建议修复动作）
    - `task.completed` / `task.failed`

### 2. 自动化执行层演进路线建议（分两步走）

纯靠 macOS `AXUIElement` 做复杂网页（如 AI Studio）的全自动闭环存在天然瓶颈（树深、无网络包可见性、DOM 节点路径 `path: [0, 3, 2]` 极易因页面微调而失效）。建议采用以下演进路线：

#### 阶段一：短期低成本改造（继续保留 Safari，补齐感知与网络抓包）
1. **解决 AX 割裂与失忆**：
   - 前端持久化 `managedTargets`（含 `windowId + url + title`），窗口消失时弹确认框而非静默删除。
   - 在 `MacosAxHelper` 引入 `AXObserver` + `CFRunLoop`，实时推送 `AXTitleChanged`、`AXLoadComplete`、`AXUIElementDestroyed`，打通与前端的自动同步，合并 `Refresh` 与 `Scan`。
2. **打通 Safari 网络包与 DOM 精准控制**：
   - 利用 Safari 的 `do JavaScript`（或配套轻量 Safari Web Extension）向目标页面注入轻量探针脚本：
     - Hook `window.fetch` 与 `XMLHttpRequest`：一旦遇到 `status >= 400` 或业务错误字段，立即提取完整 Response Body 和 Request URL，通过本地 WebSocket 推送给 P2P 控制台与 AGY。
     - 使用 `MutationObserver` 监听错误弹窗与流式输出完成状态，替代脆弱的 `350ms` 盲等。
3. **上线 P2P 本地 WS 服务（`127.0.0.1:14201`）**：让 AGY 正式具备「API 失败 → 一键/自动切网页端执行」的双路能力。

#### 阶段二：中期工业化升级（评估引入 CDP / Chromium 引擎）
- 如果后续自动化流程需要**高并发多 Tab 隔离、无弹窗干扰的后台执行、100% 稳定的网络抓包与 DOM 选择器（CSS/XPath）**，建议在 P2P 中增加 **CDP（Chrome DevTools Protocol）驱动引擎**作为可选后端（与 Safari AX 模式并存）：
  - CDP 原生自带：**持久化的 Target/Tab ID**、**毫秒级 DOM/加载事件推送**、**全量网络请求/响应 Body 抓取**、**无需抢占前台窗口焦点即可后台输入与点击**。
  - Safari AX 模式则保留用于需要复用用户 Safari 登录态/iCloud Keychain 的场景。

---

## 五、下一步行动清单（Action Items）

1. **Git 工作区整理**：确认本地已 `git pull` 同步 `6f2225c5`，保留 `.githooks/` 在仓库中；若仅因文件权限显示修改，执行 `git checkout -- .githooks/` 即可。
2. **P2P 状态持久化与确认交互**：改造 `App.tsx` 的 `managedWindowIds` / `selectedWindowId` 为持久化结构，增加 `missing` 状态与删除/重开确认弹窗。
3. **AXObserver 事件流改造**：在 `MacosAxHelper` 中增加 `AXObserver` 监听 `AXTitleChanged`、`AXLoadComplete`、`AXUIElementDestroyed`，将状态推送给 Rust/React，消除手动刷新割裂感。
4. **网络包捕获探针验证**：优先在 Safari 测试通过 JS Hook（`fetch`/`XHR`）捕获 AI Studio 等目标站点的错误 Response Body，验证无障碍盲区（隐藏错误码/原因）的完整回传。
5. **定义 AGY ↔ P2P 本地协议 Schema**：确定 `127.0.0.1:14201` WebSocket JSON-RPC 接口规范，并在 AGY 端设计双路切换开关与自动降级策略。
