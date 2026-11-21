# Live Chat 联调踩坑总结 & 最终 System Instruction

> 分支：`Antigravi-ty/agyPreviewInstant` (`a2`) · 最新 Verified Bot Commit：`6f85635`

---

## 一、Live Chat 联调过程中遇到的主要问题与坑

1. **Worker 字段名版本变更导致“静默发空消息 / 退回全局频道”**
   - **现象**：`/chat/send` 与 `/chat/poll` 误传旧字段 `channelId` / `message` 时，`/chat/poll` 会退回全局主题 `agy_p2p_v1/chat`，而 `/chat/send` 因缺少 `text` 字段会发出空消息。
   - **解决**：必须先调用 `GET /help` 读取协议文档，严格使用 `{"channel": "<ch>", "text": "..."}` 与 `{"channel": "<ch>", "stream": true}`。

2. **Cloudflare Worker 未带 `User-Agent` 或偶发 MQTT `pubackedCount: 0`**
   - **现象**：Python `urllib` 默认无 `User-Agent` 会触发 Cloudflare 403；偶发 `/chat/send` 返回 `timedOut: true, pubackedCount: 0`；个别时候 Worker 连接 EMQX 握手耗时较长（曾出现 ~81s）。
   - **解决**：使用 `curl`（或显式设置 `User-Agent`），每次 `/chat/send` 检查 `pubackedCount === 1`（为 `0` 时重发一次）。

3. **上下文压缩（Compaction）对频道与协议的冲击**
   - **现象**：长会话发生上下文压缩后，容易遗忘接口细节或误改尚未对齐的架构。
   - **解决**：把“Live Chat 通信协议 + 动工前先对齐”放进 **System Instruction**（抗压缩），把具体任务的 Token、仓库与输出备份要求放在 **User Input**。

4. **`/cmd` 参数透传范围限制（`targetId` vs `targetRegex`）**
   - **现象**：Cloudflare Worker `/cmd` 仅白名单透传 `targetRegex`、`action`、`value`、`nodePath` 等字段，直接传 `targetId` 会被忽略为空。
   - **解决**：远程调用 `guide_self_ui` / `click_self_ui` 时统一使用 `targetRegex`（例如 `^Self UI$`、`Live Chat System Instruction`）。

5. **Tauri 自身 UI 与跨进程权限边界**
   - **现象**：开发期 `pnpm p2p:dev` 的 macOS AX 权限绑定在 Terminal / Responsible Process 上，无法靠 `AppHandle` 跨 OS 进程越权遍历父/兄弟进程；且靠窗口标题或 PID 做白名单会因重启/空格连字符变化而失效。
   - **解决**：
     - 本进程内窗口管理直接通过 Rust `app.webview_windows()` + `monitor.work_area()`（自动避让 macOS Dock 与菜单栏）；
     - 自身 UI 控件采用前端自动扫描（`data-ui-action` → `aria-label` → 文本/`title`/`placeholder`），默认 `guide_only` + **Spotlight 高亮引导**，仅对 `Safe UI Button` 白名单（预置无害导航 Tab）放行自动点击。

---

## 二、最终沉淀的 Live Chat System Instruction

```text
Live Chat Protocol (Always Active):
1. On start, call GET "https://agentlivechatchannel.maboco-mark.workers.dev/help" with headers "Content-Type: application/json" and "x-agy-request-id: <x-agy-request-id from user prompt>" to get your isolated `channel` and protocol.
2. Send chat via POST /chat/send with {"channel":"<ch>","text":"..."} and verify pubackedCount === 1.
3. After every reply or step, immediately wait for the user via POST /chat/poll with {"channel":"<ch>","stream":true} until the user explicitly ends the session.
4. Always align in chat and announce briefly to the user before starting implementation; push code only via Verified Bot Commit (never local git push).
```

---

## 三、待后续解决与演进事项

1. **AI Studio `Inject Live Chat` 自动注入路径 Bug**：
   - 目前桌面端 `SafariControlStrip` 中的 `Inject Live Chat` 按钮在真实 AI Studio 窗口上执行 AX 注入时失败（路径未对齐且缺少完整 `Automation` 轨迹），待后续在真实 AI Studio 窗口上执行 `scan_window` 核对精确 AX role/title 后修复。当前可先在 `Self UI` Tab 底部直接复制粘贴。
2. **统一主托盘 + 专属 `AGY Terminal`（Multi-Tab PTY）**：
   - 后续计划单独构建一个轻量的统一托盘 + 多标签伪终端应用（Rust `portable-pty` + `xterm.js`），打包为固定 `.app` 并一次性授予 macOS AX 权限；由其拉起的各仓库 `dev` 子进程自动继承 AX 权限，并通过 `flock`（按 `identifier` 防重）与 Rust IPC 向 Remote Controller 同步受管进程白名单。
