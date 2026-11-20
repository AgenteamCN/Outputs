# EMQX Live Chat Channel 迁移与 Worker 升级报告

## 1. 问题排查（上一次 `TimeoutError` 原因分析）
1. **硬编码 55 秒上限**：原 `agentplayground` Worker 中 `/mqtt/rpc`、`/chat/poll`、`/cmd` 将 `waitMs` 硬编码限制为 `Math.min(55000, ...)`，无法满足人工听写/思考（约 1~3 分钟）的等待需求。
2. **Dangling `reader.read()` 与缺少心跳**：原 `runMqttSession` 每次循环在 `Promise.race([reader.read(), timeoutPromise])` 超时后直接 `break`，且未在长等待期间发送 MQTT `PINGREQ` 保活，容易在连接空闲或接近客户端 `timeout=65` 时触发 `TimeoutError: The read operation timed out`。
3. **`/cmd` 的 `message` 字段透传缺失**：原 `agentplayground` 的 `/cmd` 仅将 `body.message` 额外发一条到 `chat` topic，若未放在 `body.payload.message` 中，Mac 端 `EmqxPendingApprovalsSection.tsx` 审批卡片内的 `req.message` 不会显示。

## 2. `agentlivechatchannel.maboco-mark.workers.dev` 迁移与改造
已完成从 `agentplayground` 到 `agentlivechatchannel` 的完整迁移与升级：

### 2.1 明文 Runtime Variables（`plain_text`，非 Secret）
已在 Cloudflare Workers Settings -> Variables 中以 `plain_text` 类型部署以下环境变量，可在控制台直接查看与修改：
- `MQTT_HOST`: `e9c769c2.ala.cn-hangzhou.emqxsl.cn`
- `MQTT_PORT`: `8883`
- `MQTT_USERNAME`: `CNremoteController`
- `MQTT_PASSWORD`: `weqgum-2qijmY-didnem`
- `MQTT_CLIENT_ID`: `agy_cloud_agent_v1`
- `MQTT_TOPIC_PREFIX`: `agy_p2p_v1`
- `AUTH_UUID`: `94da204c-3104-4de8-986f-5f15cbb3c545`
- `DEFAULT_WAIT_MS`: `300000`（5 分钟）
- `MAX_WAIT_MS`: `300000`（5 分钟）
- `POLL_INTERVAL_MS`: `10000`（10 秒）

### 2.2 UUID 鉴权
除 `GET /` 健康检查外，`/version`、`/mqtt/rpc`、`/chat/send`、`/chat/poll`、`/cmd` 均校验 `AUTH_UUID`（支持 `x-agy-request-id`、`x-auth-uuid` 或 `Authorization: Bearer <uuid>` 请求头），缺失或不匹配返回 `401` 与 `errorCode: "UNAUTHORIZED"`。

### 2.3 5 分钟等待 + 10 秒轮询心跳 + 结构化错误码
- **单 `pendingRead` 轮询循环**：复用同一个 `reader.read()`，每 `10s`（`POLL_INTERVAL_MS`）触发一次 tick 并向 EMQX 发送 `PINGREQ` (`0xc0 0x00`) 保活，不产生悬空读请求与多余 CPU 消耗。
- **可选流式心跳 (`stream: true`)**：立即返回 `200 application/x-ndjson`，每 `10s` 输出一行 `{"type":"heartbeat","pingCount":...,"elapsedMs":...}`，彻底规避中间代理空闲断连，最终输出 `{"type":"result",...}`。
- **结构化错误码**：
  - `UNAUTHORIZED` (401)
  - `BAD_REQUEST` (400)
  - `WAIT_TIMEOUT` (408，`/cmd` 等待超时未收到响应)
  - `MQTT_CONNECT_ERROR` / `MQTT_CONNACK_REJECTED` / `MQTT_SOCKET_CLOSED` / `MQTT_SOCKET_WRITE_ERROR` / `INTERNAL_ERROR` (500)
- **`/cmd` 内联 `message`**：`body.message` 直接写入 `cmdPayload.message`，Mac 端审批卡片可直接展示说明文本（无需再额外配对一条 chat 消息；如需同步发 chat 可传 `mirrorToChat: true`）。

## 3. 实测验证结果
1. **鉴权测试**：
   - 无 `AUTH_UUID` 访问 `/version` -> `HTTP 401` (`errorCode: "UNAUTHORIZED"`)。
   - 携带 `x-agy-request-id: 94da204c-3104-4de8-986f-5f15cbb3c545` 访问 `/version` -> `HTTP 200`。
2. **Mac 端双向联调（`ping`）**：
   - 调用 `POST /cmd` (`command: "ping"`, `stream: true`, `waitMs: 240000`)，Mac 端 `p2p_mac_4jwa_rbi65qxy` 成功收到审批卡片并点击 `reply_and_approve` 回复 `"hi"`，耗时 `6764ms`，返回 `pong: true`。
3. **长等待与 10 秒心跳超时测试（120 秒 > 100 秒代理阈值）**：
   - `stream: true` (`waitMs: 120000`, `pollIntervalMs: 10000`)：共发送 `11` 次 `PINGREQ` 心跳，在 `120000ms` 正常结束并返回 `timedOut: true`，无 524 或 CPU 超时。
   - `stream: false` (`waitMs: 120000`, `pollIntervalMs: 10000`)：同样稳定等待 `120000ms` 后正常返回 `HTTP 200` 与 `timedOut: true`（`pingCount: 11`），未触发 524 或 CPU 超时。
4. **旧 Worker 清理**：
   - `agentplayground.maboco-mark.workers.dev` 已清空所有 `bindings` 并替换为 `410 Gone` (`errorCode: "MIGRATED"`) 占位脚本。

## 4. 后续调用示例
```bash
curl -sN -X POST "https://agentlivechatchannel.maboco-mark.workers.dev/cmd" \
  -H "Content-Type: application/json" \
  -H "x-agy-request-id: 94da204c-3104-4de8-986f-5f15cbb3c545" \
  -d '{
    "command": "get_status",
    "message": "请批准获取 Safari 窗口列表",
    "waitMs": 300000,
    "pollIntervalMs": 10000,
    "stream": true
  }'
```
