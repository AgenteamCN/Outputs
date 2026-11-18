# 联调进展与修复报告

## 1. `res.result.length` 与 JS 注入报错根因及修复

### 根因
在 `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift` 中，`executeJavaScript`、`reloadWindow`、`closeWindow` 返回的是 `[String: Any]` 字典，而 `Models.swift` 中的 `AnyCodable.encode(to:)` 未处理 `[String: Any]` 与 `[Any]`，直接落入 `default: try container.encode(String(describing: value))`。这导致 `ax_execute_javascript` 返回了一整段 Swift 字典 debug 字符串，而不是 `{ ok, windowId, result }` JSON 对象，前端 `axService.ts` 读取 `res.result.length` 时即触发 `undefined is not an object (evaluating 'res.result.length')`。

### 已完成修复（已推送至 `a2` 分支）
- **Commit**: `42de8b6069fc1052d2ee5f452dae648321960a3b` (`verified: true`, `Co-authored-by: Mark`)
- 修改文件：
  - `packages/macos-ax-helper/Sources/MacosAxHelper/Models.swift`：新增 `ReloadWindowResult`、`CloseWindowResult`、`ExecuteJavaScriptResult` 强类型 `Encodable` 结构体；在 `AnyCodable.encode(to:)` 中补充 `[String: Any]` 和 `[Any]` 的递归编码。
  - `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift`：改用上述强类型结构体作为返回值，并在 `executeJavaScript` 中增加 `try/catch` 捕获页面 JS 运行时错误（返回 `{ "__jsError": ... }`）。
  - `apps/p2p-remote-controller/src/services/axService.ts`：在 `executeSafariJavaScript` 日志摘要中增加 `typeof res?.result === "string"` 防御检查。
- 验证：`pnpm p2p:check`（`tsc --noEmit` + `eslint src`）通过。
- **本地更新方式**：拉取 `a2` 分支后重新编译 Swift helper：
  ```bash
  git pull origin a2
  pnpm ax-helper:build
  ```

---

## 2. EMQX + Cloudflare Worker 双向通信打通实测

### 双方配置说明
- **Mac 端（`EmqxBridgePanel`）**：服务器地址继续填 `wss://e9c769c2.ala.cn-hangzhou.emqxsl.cn:8084/mqtt` 即可（你当前已正常连上：`p2p_mac_d6u2_dtg6dea8`）。
- **云端 Agent 端（`agentplayground.maboco-mark.workers.dev`）**：已部署 HTTPS (443) -> EMQX TLS (8883) 的 MQTT RPC 网关（`POST /mqtt/rpc`），绕开沙盒非标准端口限制。

### 实测结果
- 已通过 Worker 向 `agy_p2p_v1/cmd/ping` 发送指令 `req_ping_live_1`。
- 已成功收到你在 Mac 端点击 `Reply Only` 发回的响应：
  - `reply`: `"I see it!"`
  - `recentUserActions`: 成功收到 piggybacked 的 15 条本地 AX/刷新事件（包含 `windows=5`、`Google AI Studio` 等窗口状态）。
- 修复推送后也已通过 `agy_p2p_v1/chat` 发送了通知消息。

---

## 3. 通信架构评估（Retain / 乱序 / 长连接额度 / 并发）

1. **是否会收到过期的 `retain` 消息？**
   - 不会。`emqxClient.ts` 与 `App.tsx` 中发布 `resp` 和 `chat` 均显式设置了 `retain = false`。
2. **Worker 无法保持长连接怎么解决不丢消息、又不耗尽额度？**
   - 当前方案采用 **MQTT 3.1.1 持久会话（`cleanSession: false` + 固定 `clientId: "agy_cloud_agent_v1"` + `QoS 1`）**。
   - 当云端 Agent 未轮询时，EMQX Broker 会自动为 `agy_cloud_agent_v1` 队列缓存所有 `QoS 1` 消息；Worker 每次仅在 `POST /mqtt/rpc` 时短连几秒取回队列消息并立即断开，**零空闲 CPU 占用，不需要持久长连接**。
3. **防消息打架 / 时钟不同步**：
   - 指令响应天然通过 `requestId` 一一绑定（`agy_p2p_v1/resp/<requestId>`）。
   - 对于自由聊天（`agy_p2p_v1/chat`），后续可在 payload 中补充单调递增序号 `seq` 与会话 `sessionId`，每次建连重置计数即可彻底避免双端时钟差异或超时重试导致的顺序错乱。
4. **并发与边干活边通信**：
   - 支持并发工具调用，可以在等待/轮询消息的同时并行执行代码检查与修改。

---

## 4. 注入验证、Response 查看、内存防溢出与窗口管理

- **Response 在哪看 & 如何验证注入**：重编 `macos-ax-helper` 后，使用 **Quick Init + Record** 或 **Poll / Verify Net**，在右侧 **Network** 标签页即可查看注入状态（`verified` / `failed`）、`Captured Interaction IDs` 及 `Captured Fetch / SSE / XHR Responses`。
- **内存防溢出**：前端与注入脚本均已采用固定上限的环形缓冲区（`MAX_CONSOLE_LOGS = 200`、`MAX_TIMELINE_EVENTS = 1000`、页面内 `entries` 上限 200、`interactions` 上限 100），并已提供 **Clear Timeline**、**Clear UI Logs**、**Clear Net Buffer** 手动清空按钮及 **Close Window** 关闭指定 Safari 窗口功能。
