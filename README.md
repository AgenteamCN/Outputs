# `agyPreviewInstant` (`p2p-remote-controller`) 问题诊断、方案讨论与代码实现报告

## 一、时间轴与错误日志联合诊断

### 1. 时间轴操作还原
从导出的 232 条 Timeline 事件与 Console 日志可以完整还原你的真实操作链路：

1. `06:41:55.560Z` (`id: 58`)：打开受管窗口 `#21367`（`https://aistudio.google.com/`）。
2. `06:42:31.973Z ~ 06:42:34.522Z` (`id: 73~77`)：页面重定向至 `https://aistudio.google.com/prompts/new_chat`，`AXLoadComplete` 触发，自动扫描出 31 个元素（`AXLoaded=true`）。
3. `06:42:59.511Z` (`id: 78`)：点击 `AXButton "No thanks"`，重扫后主界面展开为 96 个元素。
4. `06:43:10.654Z` (`id: 80`)：点击 `AXButton "Link API Key"`，弹窗出现（元素收敛为 4 个）。
5. `06:43:25.235Z` (`id: 85`)：点击 `AXButton "Skip"`；`06:43:27.298Z` (`id: 93`)：点击 `AXButton "Select key"`，弹窗关闭，主界面恢复为 104 个元素。
6. **故障窗口 `06:43:33 ~ 06:44:55`**：
   - `06:43:33.633Z` 与 `06:43:36.486Z`：Safari 连续推送大量 `AXUIElementDestroyed`（`windowId: null`），说明 SPA 内部的 Toast / 遮罩层或动态容器刚被销毁，DOM 树结构发生了缩减。
   - `06:44:00.722Z ~ 06:44:55.462Z`：你在前端连续点击了 9 次元素，全部在 `ax_perform_action` 报错 `Element path out of bounds at index 0`。
7. **恢复与后续操作 `06:44:58 ~ 06:46:40`**：
   - `06:44:58.854Z` (`id: 138`)：你手动点击 `Scan AXTree`（重新扫描出 101 个元素）。
   - 随后 `06:45:31 ~ 06:46:40`：聚焦并点击 `Select Antigravity agent harness model` -> 选择 `Gemini 4 Hydrogen` -> 点击 `1 rule` -> `Add to allowlist` -> 对 `Domain` 输入框执行两次 `perform_set_value` -> 点击 `Save`，全部一次成功。

### 2. 为什么会出现 `Element path out of bounds at index 0`？
这是**自动化侧刷新/定位机制的遗漏**，而不是页面 AX 标签改名导致的问题：

1. **纯下标路径脆弱性**：`AccessibilityEngine.swift` 使用从 `AXWebArea` 出发的整数下标数组 `path: [Int]`（如 `[3, 0, 21]`）定位元素。一旦 SPA 动态插入或销毁顶层容器（如弹窗、Toast、Banner），`AXWebArea` 的第 0 层子节点数就会变化，旧路径立刻失效。
2. **`AXUIElementDestroyed` 未触发自动重扫**：此前 `App.tsx` 中 `isScanNotif` 只监听 `AXLoadComplete`、`AXTitleChanged`、`AXLiveRegionChanged`，没有包含 `AXUIElementDestroyed`（且该事件 `windowId` 为 `null`），导致 `06:43:36` 界面结构变化后 `scanResult` 未刷新。
3. **失败未入 Timeline 且无过期自愈**：此前 `ax_perform_action` 抛错后没有写入 Timeline（导致时间轴上看不到这 9 次失败点击），也没有在遇到 `out of bounds` 时自动触发重扫。
   - **安全原则**：后台检测到路径过期后，**不能直接用旧下标盲目重试**（因为重扫后同一坐标可能是别的按钮），正确做法是自动触发后台重扫刷新元素树（或后续升级为基于 `role + title/description/identifier` 的语义重定位）。

---

## 二、加载状态、`Refresh Current` 与手机端 Workflow 思路

1. **加载状态获取**：
   - 整页导航时，`AXWebArea` 的 `AXLoaded` 与 `AXLoadComplete` 可靠；但 SPA 内部路由切换或异步请求期间，`AXLoaded` 往往保持 `true`。
   - 后续可补充**双信号判断**：`AXLoaded` + **事件静默期（Debounce Stability）**，即收到 `AXLiveRegionChanged` / `AXUIElementDestroyed` 后标记为 `Settling...`，静默超过 300ms~500ms 再标记为 `Ready`。
2. **`Refresh Current` / `Retry Current`**：
   - 当前已有 `Open New` 和 `Focus Current`，建议在控制条增加 `Refresh Current`（通过 Apple Events `tell active tab to reload` 或对 Safari 发送 `Cmd+R`），用于在网络波动时直接刷新当前受管窗口，后续可联动代理/节点切换插件。
3. **手机端远程控制 Workflow 建议**：
   - PC 端（`p2p-remote-controller`）承担 **状态聚合 + 语义执行器** 的角色：维护受管窗口、AX 树、错误日志与页面网络状态。
   - 手机端不直接传递脆弱的 `path: [Int]`，而是接收 PC 端提炼后的**高阶语义卡片**（按钮名、输入框、当前状态徽标、最近报错摘要），点击后由 PC 端执行、自动重扫并回传结果。

---

## 三、不注入脚本获取 Safari HTTP 非 200 Body 的技术方案对比

### 1. 本地 HTTPS 代理 vs 传统外部 MITM，为什么不触发 Certificate Pinning？
- **本质差异**：本地 HTTPS 代理在协议层依然是 TLS 代理（浏览器 <-> 本地代理 <-> 目标服务器），但**信任机制完全不同**：
  - 外部 MITM 使用未知 CA，会被操作系统与浏览器拦截。
  - 在 macOS 上将本地代理自签 Root CA 放入 **System Keychain（系统钥匙串）并设为“始终信任”** 后，Safari/WebKit（`com.apple.WebKit.Networking`）会将其视为合法受信任根证书。
- **为什么不会触发 Certificate Pinning**：
  - 现代浏览器（包括 Safari）已废弃 HTTP Public Key Pinning (HPKP)；而浏览器内置的预置 Pinning 列表（如对部分自家域名的保护）在行业惯例上**会对用户手动安装到系统信任库的本地 Root CA 豁免**（否则企业代理和开发者调试工具全部无法工作）。这与独立原生 App 在代码里硬编码公钥的做法不同。
- **需要注意的坑**：
  - **HTTP/3 (QUIC)** 走 UDP 443，会绕过传统 TCP HTTP/HTTPS 代理，需要在 Safari 关闭 QUIC 使其回退到 HTTP/2 over TCP，或者使用 `NETransparentProxyProvider`。
  - **SSE 流式响应**：代理必须支持分块流式透传（不能等整个响应结束才转发），只在状态码 `>= 400` 时缓存并提取错误 JSON Body。

### 2. 为什么 Safari Web Inspector / `safaridriver` 拿不到 HTTP Body？
- **`safaridriver`**：严格遵循 W3C WebDriver 规范，定位是 DOM 查询与输入自动化，标准协议中**没有**像 Chrome DevTools Protocol (`Network.getResponseBody`) 那样暴露底层网络响应体读取接口。
- **`webinspectord`**：这是 Apple 为自家 Safari 开发者工具提供的私有守护进程，协议未公开稳定文档且受系统签名/沙盒边界约束，第三方应用无法可靠挂载并抓取 Response Body。

### 3. JavaScript Hook（`fetch` / `XHR`）的技术原理与监听范围
- **技术原理**：在页面主世界（Main World）中对 `window.fetch` 和 `XMLHttpRequest.prototype.open / send` 做只读包装（Monkey Patch）。当 `fetch` 返回 `!response.ok`（或 `XHR` 的 `status >= 400`）时，通过 `response.clone().text()`（注意必须 `clone()`，否则会提前消耗掉业务代码的响应流）旁路读取 Body，连同 URL、Status、Method 通过本地 WebSocket 或 `console.error` 回传给控制端。
- **是否为全局监听**：
  - 它是**当前注入页面（Tab/Frame）内的全局监听**——页面内任何 JS 模块发起的 `fetch` / `XHR` 都会经过它；
  - 但**不是跨 Tab / 跨站点的系统级全局监听**，需要每个受管页面在加载后注入（或者用轻量 Safari Web Extension 的 `content_scripts` 在 `document_start` + `world: "MAIN"` 自动注入，避免每次手动触发）。
- **局限**：无法捕获图片/字体等非 JS 请求，也无法读取 CORS `opaque` 响应；对于正常 200 的 SSE 流应直接跳过 `clone()` 以免增加内存开销。

---

## 四、本次已落地的代码修复与验证

仓库：`https://github.com/Antigravi-ty/agyPreviewInstant`（分支 `a2`，Verified Bot Commit `fa2e84c2a52e2a2df8413d37d0649cc45741bfa5`）

1. **`set_value` 支持多行换行**：
   - 将 `apps/p2p-remote-controller/src/App.tsx` 中 `AXTextField` / `AXTextArea` 的 `<input type="text">` 替换为 `<textarea rows={2}>`，并在 `apps/p2p-remote-controller/src/styles/app.css` 补充 `textarea` / `select` 样式，解决单行 input 自动吞掉 `\n` 的问题。
2. **关闭 Regex 搜索自动纠错**：
   - 为过滤输入框（以及 `set_value` 输入框）添加 `autoCorrect="off" autoCapitalize="off" autoComplete="off" spellCheck={false}`，防止 macOS WKWebView 篡改正则表达式字符。
3. **分级日志导出与复制（`Compact` / `Standard` / `Full`）**：
   - 新增 `apps/p2p-remote-controller/src/utils/logExport.ts` 与单元测试 `apps/p2p-remote-controller/src/utils/logExport.test.ts`。
   - 支持在顶部 `Quick Copy` 与 `Timeline` 导出中切换 `Compact`（仅错误 + 关键操作 + 扫描摘要，不含冗长元素列表）、`Standard`（折叠连续重复 `AXUIElementDestroyed` 噪音 + 摘要）、`Full`（全量原始数据）。
4. **防过期与错误可观测性增强**：
   - 将 `AXUIElementDestroyed` 纳入 `autoRescan` 触发条件；
   - `handlePerformAction` 失败时写入 `perform_<action>_failed` 到 Timeline；遇到 `out of bounds` 且开启 `Auto-Rescan` 时自动后台刷新 AX 树并提示重试。
5. **工具链与验证**：
   - 已在沙盒安装 `pnpm`（`12.21.0`）并完成依赖安装；
   - `pnpm p2p:check`（`tsc --noEmit` + `eslint src`）：通过；
   - `pnpm --filter p2p-remote-controller test`（`vitest run`）：4/4 测试全部通过；
   - `pnpm p2p:build`（`tsc && vite build`）：构建成功。
