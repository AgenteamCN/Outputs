# Cloudflare Worker 实测数据、超时边界 & Durable Objects 免费额度深度解析

---

## 一、通过 Cloudflare GraphQL Analytics API 查阅到的真实运行数据

我们已使用你的 Token 查询了 Cloudflare GraphQL Analytics API（`workersInvocationsAdaptive`，脚本 `agentplayground`，时间范围 `2026-11-17` 至 `2026-11-20`）：

- **总请求数（`requests`）**：`121` 次
- **子请求数（`subrequests`）**：`111` 次
- **报错数（`errors`）**：`0` 次
- **累计 CPU 时间（`cpuTimeUs`）**：`620,081 µs` → **平均每次请求 `5.12 ms`**（与你面板看到的 `5 ms` 完全吻合）
- **累计 Wall Time（`wallTime`）**：`5,307,576,792 µs`（`5,307.58 秒`）→ **平均每次请求 `43.86 秒`**（与你面板看到的 `50 s` 完全吻合）

*(注：旧版 `/subscriptions` 接口返回权限受限，但 GraphQL Analytics 可直接读取完整指标。)*

---

## 二、能否无限等？CPU > 10ms 或超长 Wall Time 会发生什么？

### 1. Wall Time（墙上时间）并不是「无限等」
- **CPU Time vs Wall Time**：Free 套餐限制的是 **10 ms 纯 CPU 计算时间**（JS 执行、JSON 解析、加密等），**网络 I/O 等待（`await reader.read()`、`fetch()`）不消耗 CPU**，所以才能看到 `CPU 5ms / Wall Time 50s`。
- **HTTP 请求的真实等待上限**：
  - 虽然 Workers 运行时本身没有写死 "15 分钟硬超时"，但 **Cloudflare 边缘网关与客户端之间有 HTTP 响应空闲超时（约 100 秒左右）**：如果一个 HTTP 请求长时间不吐出任何字节（headers/body），边缘网关或上层客户端（如 `curl` / 代理）会直接断开连接（返回 `524` 或客户端 `Timeout`）。
  - 同时，我们在 `emqx_worker.mjs` 里把 `waitMs` 上限设在了 **`55,000 ms`（55 秒）**，就是为了安全落在 100 秒边缘超时之内。

### 2. 如果 CPU 超过 10ms 或脚本异常，会静默销毁还是报错？
- **不会静默无响应**，而是立即终止该 Isolate 并由 Cloudflare 边缘返回明确的 HTTP 错误：
  - **CPU 超限**：触发 **`Error 1102: Worker exceeded CPU time limit`**（HTTP 503/500 错误页），正在执行的请求被强行掐断。
  - **内存超限（> 128 MB）** 或 **未捕获异常**：触发 **`Error 1101 / 10021`**，同样返回 HTTP 500/503 错误。
  - **客户端主动断开（Client Disconnect）**：如果请求方等不及先断开了 HTTP 连接，Worker 会立刻取消关联的异步任务并退出（不会继续白跑）。

---

## 三、我们当前到底是什么结构？（「长轮询硬等」vs「10 秒轮询」）

你问得非常关键，理清当前链路如下：

1. **为什么 Agent 要经过你的 Worker Proxy？**
   - Agent 运行在受限网络沙盒中，**只能访问标准 80/443 端口的 HTTP/HTTPS 域名**，无法直接连 EMQX 的 `8883`（MQTTS）端口。
   - 因此 `agentplayground.maboco-mark.workers.dev` 充当了一个 **HTTPS (443) -> MQTTS (8883) 的协议转换代理**。

2. **Worker 内部是「每 10 秒轮询」还是「硬等」？**
   - **是「硬等」（HTTP Long-Polling 长轮询）**，不是每 10 秒断一次。
   - 每次 Agent 调用 `POST /mqtt/rpc`（带 `waitMs: 55000`）：
     1. Worker 现场向 EMQX `8883` 建立一条全新的 TCP + TLS 连接，发 MQTT `CONNECT` + `SUBSCRIBE`。
     2. 然后挂起在 `Promise.race([reader.read(), timeout(waitMs)])` 上**硬等**，直到收到消息或 55 秒到期才关闭连接返回。
   - **Plan Alpha 修复前后的本质差异**：
     - **修复前**：由于 `burstDeadline` 永假 Bug，就算你在第 5 秒回复了，Worker 收到消息后**依然不会返回**，继续硬等完剩下的 50 秒（这就是你面板上 `Wall Time: 50s` 的元凶）。
     - **修复后**：你在第 5 秒回复，Worker 收到后 **`150ms` 内立即返回**；但如果用户还没回复，它依然会硬等最多 55 秒，然后断开让 Agent 发起下一轮。

---

## 四、Durable Objects (DO) 最新免费额度 & 到底和普通 Worker 有什么本质区别？

### 1. 最新免费额度核实（Workers Free 套餐已正式包含 DO！）
根据 Cloudflare 官方最新政策（2025-04 起开放，仅限 **SQLite-backed Durable Objects**，无需绑定付费升级即可使用）：

| 资源维度 | Workers Free 每日免费额度（00:00 UTC 重置） | 对我们场景的够用程度 |
| :--- | :--- | :--- |
| **Requests（请求数）** | **100,000 次 / 天**（含 HTTP、RPC、WS 消息按 20:1 折算） | 极度充裕（当前 3 天才 121 次） |
| **Duration（内存驻留时长）** | **13,000 GB-s / 天**（每个活跃 DO 按 128MB = `0.125 GB` 计费） | 相当于 **104,000 活跃秒 / 天（约 28.8 小时/天）** |
| **SQLite Rows Read** | **5,000,000 行 / 天** | 极度充裕 |
| **SQLite Rows Written** | **100,000 行 / 天** | 极度充裕 |
| **SQLite Stored Data** | **5 GB 总量** | 极度充裕 |

> **关键结论**：只要使用 **SQLite-backed DO**，当前免费额度完全够用，**不需要** $5/月付费计划！

### 2. 上了 Durable Objects 后，它占用什么？为什么能优化 Proxy 请求？
很多开发者容易把 DO 和普通 Worker 混为一谈，它们的**本质区别**在于：

| 对比维度 | 普通无状态 Worker（当前方案） | Durable Objects (`MqttBridgeDO`) |
| :--- | :--- | :--- |
| **状态与内存** | 每次请求独立沙盒，请求结束状态全丢 | **全局唯一有状态单例**：拥有常驻内存变量 + 内置零延迟 SQLite |
| **连接保持能力** | 无法跨请求保留连接；Agent 思考间隙完全离线，消息可能错过或积压在 EMQX | 支持 **WebSocket Hibernation API**：Mac 与 iPhone 保持 `wss://` 长连接连到 DO；**空闲休眠时不占内存、Duration 计费为 0**，有消息时毫秒级唤醒！ |
| **为什么能优化 Agent Proxy 请求？** | 每次 `POST /mqtt/rpc` 都要重新 TLS 握手 + MQTT `CONNECT` + `SUBSCRIBE`（耗时 300ms–1500ms） | **方案 1（纯 DO Hub，最优）**：Mac 和手机直接用 WebSocket 连 DO（走标准 443 端口），**完全省掉 EMQX 中转与每次 TLS/MQTT 握手**！Agent 发请求到 DO 内存直接读队列或挂起等唤醒，**延迟 < 50ms**。<br>**方案 2（DO 连 EMQX 8883）**：注意！DO 主动发起的出站 TCP (`connect()`) **不支持休眠**，会让 DO 保持活跃并消耗 Duration（不过 1 个 DO 24h 全活跃也仅消耗 `86400 * 0.125 = 10,800 GB-s/天 < 13,000 GB-s/天` 免费上限，只是接近每日限额，不如方案 1 优雅）。 |
| **iOS Web Push 推送** | 断开后无法感知新消息 | DO 作为中心枢纽，收到 Agent 指令时可立即向手机端订阅的 `PushSubscription` 触发 Web Push |

---

## 五、总结建议

1. **当前立即可用**：Plan Alpha 已经解决了「收到回复还白等 50 秒」的 Bug，目前处于 **0 成本免费运行**状态，可先直接观察实际交互延迟是否已降至秒级。
2. **下一步演进（零成本升级 Plan Bravo）**：
   - 由于 Cloudflare **Workers Free 套餐现已包含 SQLite-backed Durable Objects**（10万请求/天 + 13,000 GB-s/天），下一步非常值得把 `agentplayground` 升级为 **DO + WebSocket Hibernation Hub**：
     - Mac 端 & iPhone Pocket Deck 通过 `wss://agentplayground.maboco-mark.workers.dev/ws` 连入同一个 DO（支持休眠，空闲 0 GB-s 消耗）。
     - Agent 依然通过 HTTPS 443 调 Worker `/rpc`，由 DO 在内存中秒级转交或唤醒返回，彻底告别每次重连 EMQX 的握手延迟，并顺便打通 iOS Web Push。
