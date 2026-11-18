# P2P Remote Controller 自动化初始化升级、Network Bug 修复与 Cloudflare 机制说明

## 1. 本次修复与新增能力（Commit `6a5813c` @ `a2`）

1. **Network Injection 状态同步与流式 XHR 捕获修复**：
   - **`apps/p2p-remote-controller/src/App.tsx`**：
     - 监听选中窗口 URL 变化（`syncSafariWindows`）与 `AXLoadComplete` 事件：页面发生跳转或重新加载时，自动将 `netVerifyStatus` 从 `verified` 重置为 `unchecked`，避免出现“UI 显示 Injection Verified，实际脚本已随页面跳转丢失”的假阳性。
     - 远程 `net_record_start` / `net_record_poll` 执行后同步更新本地 UI 的 `netSnapshot`、`netRecording`、`netVerifyStatus`、`netVerifyDetail`。
   - **`apps/p2p-remote-controller/src/services/networkRecorderScript.ts`**：
     - 针对 Google AI Studio 使用流式 XHR（如 `GenerateContent`、`CreatePrompt`）导致原先只监听 `loadend` 时在流式传输期间显示 `POST 0` 且无响应预览的问题，新增 `readystatechange`（`readyState >= 3`）和 `progress` 事件监听，实时同步 `xhr.status`、`content-type` 与 `responseText`。
     - 在 `scanForInteractions` 中增加对 `CreatePrompt` 响应中 prompt ID 的提取（记为 `prompt:<id>`）。
     - （注：日志中 `pagead`/`ccm/collect` 的 `fetch POST 0` 属于浏览器 `keepalive` / `no-cors` 上报请求，浏览器安全策略本身不暴露其 status，属于正常现象。）

2. **轮播图 / 动态 DOM 路径偏移与 `focus + set_value` 一步到位**：
   - **`apps/p2p-remote-controller/src/types/emqxBridge.ts` & `App.tsx`**：
     - `perform_action` 新增可选参数：`targetRegex`、`targetIndex`、`focusFirst`、`doubleCheck`。
     - 传入 `targetRegex` 时，会在执行动作前**实时重扫一次 AX Tree**，直接定位当前匹配元素的真实 `path`，彻底免疫 6 秒轮播图或输入换行导致的 `nodePath` 偏移；若未匹配到任何元素则立即报错。
     - 传入 `focusFirst: true` 时自动先执行 `focus` 再等 80ms 执行 `set_value` 等动作，减少一次 EMQX 往返。
     - 传入 `doubleCheck: true` 时在动作执行后自动再扫一次窗口并返回 `postCheck`。

3. **新增 `wait_for_element` 远程等待命令（Waiting Until Loaded）**：
   - 轮询 `scanSafariWindow`，直到 `webAreaLoaded === true` 且（若提供 `regexFilter`）至少匹配到一个元素；超时（默认 15s，上限 30s）未出现则直接报错返回。

4. **`get_status` 窗口过滤**：
   - 默认按 `managedTargets`（即受控 Safari 窗口，如 `#24640`）过滤返回，不再把系统里无关的 6 个 Safari 窗口全部返回；如需查看全部窗口可显式传 `includeAllWindows: true` 或在 UI 勾选 `Include All Safari Windows`。

---

## 2. 推荐的 AI Studio 自动初始化标准流程

1. **打开直达链接**：`https://aistudio.google.com/prompts/new_chat?model=antigravity-preview-09-2026`（跳过 Models/Agents 切换）。
2. **等待加载**：`wait_for_element` 等待 `Link API Key|Enter a prompt`（确认 `webAreaLoaded=true`）。
3. **绑定 API Key（如出现 `Link API Key`）**：
   - `perform_action(targetRegex="Link API Key", action="press", doubleCheck=true)`
   - `wait_for_element(regexFilter="Skip|Link API key")`
   - `perform_action(targetRegex="^AXButton.*Skip$", action="press", doubleCheck=true)`
   - `wait_for_element(regexFilter="Select key")`
   - `perform_action(targetRegex="Select key", action="press", doubleCheck=true)`
4. **确认模型与 Network Allowlist**：
   - `wait_for_element(regexFilter="Enter a prompt")`
   - 点击 `Select Antigravity agent harness model` -> `wait_for_element(regexFilter="Gemini 4 Hydrogen")` -> `press`。
   - 点击 Network 规则按钮（`1 rule`）-> 展开旧规则 -> `Delete rule` -> `Add to allowlist` -> `perform_action(targetRegex="AXTextField.*Domain", action="set_value", value="*", focusFirst=true)` -> `press("Save")` -> `wait_for_element(regexFilter="Enter a prompt")`。
5. **注入网络录制器 & 提交任务**：
   - `net_record_start`
   - `perform_action(targetRegex="AXTextArea.*Enter a prompt", action="set_value", value=prompt, focusFirst=true, doubleCheck=true)`
   - `perform_action(targetRegex="AXButton.*Run$", action="press", doubleCheck=true)`

---

## 3. Cloudflare Worker 机制与 API Key 解答

1. **请求方式**：我们是直接向已部署的 `https://agentplayground.maboco-mark.workers.dev/mqtt/rpc` **POST JSON 请求**，由 Worker 通过 `cloudflare:sockets` 转发到 EMQX `8883` 端口，**不需要**每次重新部署 Worker。
2. **Cloudflare API Key 的作用与权限**：
   - 只要不需要修改/重部署 Worker 代码，云端 Agent 只需要访问 `agentplayground.maboco-mark.workers.dev/mqtt/rpc` 即可，**不需要** Cloudflare API Key。
   - 使用你提供的 API Token 调用 Cloudflare REST API（`GET .../workers/scripts/agentplayground`）**可以读取**当前已部署的 Worker 源码（我刚才就是这样确认它已经实现了 `/mqtt/rpc` 的），也可以通过 `PUT` 覆盖更新。
   - **安全建议**：目前这个 Worker 本身**没有写死任何 MQTT 密码或密钥**（host/username/password 都是在每次 `/mqtt/rpc` POST 请求体里传入的），所以 Worker 代码本身不含 secret。但既然你在聊天里给过 Cloudflare API Key，等本次联调结束后随时可以在 Cloudflare 控制台 **Roll / Revoke** 这个 API Token，不影响已部署的 `agentplayground` Worker 继续运行。
   - **仓库存放**：只要不把 API Token、EMQX 密码或 Cloudflare Account ID 放进仓库，纯粹的通用代理脚本可以按需放在独立目录或保留在 Cloudflare 上不入主仓。
