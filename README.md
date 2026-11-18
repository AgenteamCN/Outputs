# Safari P2P 改造报告：AXObserver 主动推送、页面持久化、时间轴导出与本地抓包路径分析

> **代码提交**：`0a61e5dd1cb3e30621925a0d6bc8af79341df8b1`（`verified: true`，分支 `a2`）
> **构建说明**：本沙盒为 Linux x86_64 环境，已完成前端 TypeScript / ESLint / Vite 生产构建验证（`pnpm --filter p2p-remote-controller check && pnpm --filter p2p-remote-controller build` 全量通过）；Swift (`MacosAxHelper`) 与 Rust (`src-tauri`) 需在你的 macOS 上执行 `pnpm ax-helper:build && pnpm p2p:dev` 完成原生编译与实机测试。

---

## 一、本次已完成的功能改造（按你的需求落地）

### 1. `AXObserver` 主动推送 + 消除 `Refresh` / `Scan` 割裂
- **Swift 辅助进程（`packages/macos-ax-helper`）**：
  - 新增 `AxObserverManager.swift`：在独立 `CFRunLoop` 后台线程上挂载 macOS `AXObserverCreate`，自动监听 Safari 进程的：
    - `kAXTitleChangedNotification`（标题变化，解决 `Open New` 后长期显示 `Untitled`）
    - `AXLoadComplete`（WebKit `AXWebArea` 加载完成）
    - `kAXWindowCreatedNotification` / `kAXUIElementDestroyedNotification`（窗口新建/关闭）
    - `kAXFocusedWindowChangedNotification`（窗口焦点切换）
    - `AXLiveRegionChanged`（带 ARIA Live Region / `role="alert"` 的弹窗或提示）
  - 改造 `JsonRpcServer.swift`：增加线程安全的 `stdoutLock`，支持在处理 RPC 请求的同时，向 `stdout` 主动推送 `{"method": "ax_event", "params": {...}}` 通知，并新增 `start_observing` / `stop_observing` RPC（在 `get_safari_status`、`open_managed_window`、`scan_window_elements` 时自动挂载）。
- **Rust 桥接层（`apps/p2p-remote-controller/src-tauri`）**：
  - 改造 `ax_bridge.rs`：将原本同步阻塞的 `stdout.read_line()` 拆分为独立 `stdout` 异步读取线程 + `pending` RPC 请求映射表（15 秒超时），收到 `ax_event` 时通过 Tauri `app.emit("ax-event", params)` 实时推给前端。
- **前端统一同步（`apps/p2p-remote-controller/src/App.tsx`）**：
  - 顶部新增 **`AXObserver Active / Off`** 状态徽章。
  - 收到 `ax-event` 后自动防抖（220ms）触发窗口列表更新与（当 `Auto-Rescan` 开启且命中当前选中窗口时）AX 树重扫，并将 `windowTitle` / `url` 实时回写到窗口标签上。
  - 点击顶部 `Refresh` 时，若当前有选中的活跃窗口，会同步重扫 AX 树；点击窗口标签扫描 AX 树时，也会把最新 `title` 和 `url` 同步到标签上，彻底消除两者割裂。

### 2. 页面 ID 持久化 + 消失询问删除
- 将 `managedTargets`（含 `windowId`、`title`、`url`、`tabCount`、`status`、`lastSeenAt`）、`selectedWindowId` 与 `targetUrl` 持久化到 `localStorage`。
- 当已记录的 Safari 窗口被关闭或消失时，**不再静默移除**，而是将其标记为 `missing`（标签变为黄色虚线 `[Missing]`），并在顶部弹出确认横幅，提供：
  - **`Reopen URL`**：按原 URL 重新打开并替换旧记录
  - **`Remove Record`**：用户确认后再移除记录

### 3. 全链路时间轴记录与导出（`Timeline` 标签页）
- 右侧面板新增 **`Timeline (N)`** 标签页，按毫秒时间戳记录每一条事件，并明确标注：
  - `source`：`ax_observer`（系统无障碍推送）/ `user_action`（用户点击操作）/ `auto_refresh`（由推送触发的窗口刷新）/ `auto_rescan`（由推送或动作触发的 AX 重扫）/ `system`
  - `triggeredRefresh`：`true` / `false`（该事件是否触发了自动刷新）
  - `windowId`、`durationMs`、`detail`
- 提供 **`Export Timeline JSON`**（直接下载 `.json` 文件）和 **`Copy Timeline`**（一键复制到剪贴板），方便你实机测试后直接导出发给我分析。

---

## 二、macOS 实机操作指南（如何测试并导出时间轴）

1. 在 macOS 本地拉取最新 `a2` 分支代码并重新编译 Swift Helper 与启动 P2P：
   ```bash
   git pull origin a2
   pnpm ax-helper:build
   pnpm p2p:dev
   ```
2. 确认顶部三个状态徽章均正常：`AX Trusted`、`Safari Automation OK`、`AXObserver Active`。
3. 点击 **`Open New`** 打开 `https://aistudio.google.com`（此时不要点 Refresh，先观察 3~5 秒，看 `AXObserver` 是否自动把标题从 `Untitled` 更新并自动扫描出元素）。
4. 在 AI Studio 页面或 P2P 面板上执行几次点击/输入操作，然后再手动关闭一个窗口测试 `[Missing]` 询问条。
5. 切换到右侧 **`Timeline`** 标签页，点击 **`Export Timeline JSON`** 导出 JSON 文件发给我即可。

---

## 三、深度解答：Safari 不注入脚本能否拿到数据包错误详情？会违反 ToS 吗？

### 1. 关于 Safari 注入脚本与 ToS 的边界
- **Apple / Safari 侧**：Safari 官方在“开发”菜单中原生提供了 `允许来自 Apple 事件的 JavaScript`（`do JavaScript`），并有官方的 `Safari Web Extensions` 体系，**在自己本机浏览器里使用这些接口本身完全合规**。
- **目标网站（如 Google AI Studio）ToS 侧**：大多数在线服务条款限制的是“未授权的自动化批量抓取、绕过速率限制或逆向工程”。如果是个人辅助操作/状态观测，只要不绕过鉴权或高频刷接口，通常属于灰区辅助工具；但如果你希望**完全不向网页 DOM/JS 注入任何脚本**，下面分析纯本地系统的其他通道。

### 2. 不注入脚本、不用 Chrome：macOS Safari 还有哪些本地通道能拿到 HTTP 非 200 错误详情？

你提到一个非常关键的问题：*“数据都已经进到我电脑里了，Back End 已经把 `code` / `reason` 发过来了，为什么一定要 MITM？”*

**macOS 底层隔离真相**：
Safari 的网络请求与 TLS 加解密发生在独立且受沙盒/SIP 保护的系统进程 **`com.apple.WebKit.Networking`** 内存空间中。外部进程（包括我们的 `P2P Remote Controller`）**无法直接跨进程读取它的内存明文**。因此，在不注入页面 JS 的前提下，macOS 本地只剩两条路径：

| 通道方案 | 实现原理 | 能否拿到非 200 响应的 `code` / `reason` | 需要什么权限 / 配置 | 优缺点评估 |
| :--- | :--- | :--- | :--- | :--- |
| **路径 1：本地 HTTPS 代理 / macOS `NETransparentProxyProvider`**（推荐的零注入方案） | 在本机启动一个轻量 Rust 代理（如基于 `hudsucker`），或在系统设置中配置本地代理，将自签根证书加入 macOS **系统钥匙串（System Keychain）并设为“始终信任”**。Safari **默认信任用户手动信任的系统根证书**（不会像独立 App 那样因 Pinning 报错）。 | **可以（100% 拿到完整 HTTP 状态码、Header、JSON Error Body、SSE 流）** | 1. 引导用户安装并信任一次本地根证书（或使用 `NETransparentProxyProvider` 需 Apple Developer 签名）；<br>2. 若目标站走 HTTP/3 (QUIC)，需在 Safari 关闭 QUIC 使其回退到 HTTPS/TCP。 | **完全零页面脚本注入**；但需要处理证书信任与代理开关。 |
| **路径 2：Safari Web Inspector / `safaridriver`** | 通过 Apple 原生 `webinspectord` / `safaridriver`（WebDriver）连接 Safari。 | **极受限**（传统 `safaridriver` 主要用于 DOM 操控，对网络响应 Body 拦截支持远不如 Chrome CDP）。 | 需开启 Safari「允许远程自动化」 | 不适合用来稳定抓包解析错误 JSON。 |
| **对比：JS Hook（`do JavaScript` / Safari 扩展）** | 仅在页面内 Hook `fetch` / `XHR` 只读监听返回体并回传本地 WS。 | **可以（100% 拿到）** | Safari 开启「允许来自 Apple 事件的 JavaScript」或安装轻量 Safari 扩展。 | 工程最简单、无证书问题、无 QUIC 问题；唯一前提是允许注入只读监听脚本。 |

---

## 四、待你确认的问题（决定下一步自动化与手机端联调方向）

请你在实机跑完本次更新并导出 Timeline 后，确认以下 5 个问题，以便我们定死下一步方案：

1. **网络错误包捕获路线二选一**：
   - 你更希望走 **「本地 HTTPS 代理 + 钥匙串信任一次根证书」（零页面脚本注入）**，还是 **「Safari 开启允许 Apple 事件 JS / 扩展做只读 Hook」（零证书、无 QUIC 干扰）**？
2. **`AXObserver` 在 AI Studio 的实测覆盖度**：
   - 导出 Timeline 后，我们就能直接看到 AI Studio 页面加载、按钮变化、报错弹窗出现时，Safari 到底发了哪些 `AXObserver` 通知、哪些没发（决定是否需要在任务执行期开启 500~1000ms 节流辅助扫描）。
3. **加载等待完成（Wait-for-Idle）的判定信号**：
   - 后续自动化发送 Prompt 后，AI Studio 会有加载动画。你希望优先依据什么判断“回答已生成完毕”？（例如：`Stop` 按钮消失变回 `Run/Send`、还是结合网络 SSE 流 `[DONE]` 结束信号？）
4. **手机端优先控制路径**：
   - 手机端操作时，是让手机继续通过 **AGY 主程序的 LAN 服务（`14200` 端口）** 转发指令给 P2P，还是让 P2P 自己直接开一个局域网/本地 HTTP+WS 控制端口（如 `14201`）？
5. **Safari 窗口 vs 多标签页（Tab）**：
   - 当前设计是“一个自动化任务对应一个独立 Safari 窗口（`windowId`）”，这能避免多标签页互相抢焦点。后续是否需要支持**同一个 Safari 窗口内的多个 Tab 切换**？
