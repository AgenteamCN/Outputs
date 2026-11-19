# Cloudflare Worker 指标审计、Wall Time / 10ms CPU 机制 & Durable Objects 免费额度深度解析

本文基于通过 Cloudflare GraphQL Analytics API 实测拉取的 `agentplayground` 线上调用数据，以及 Cloudflare 官方最新文档（2025–2026）整理。

---

## 一、通过 Cloudflare API 实测查到的 `agentplayground` 真实数据

使用当前 Token 查询 `workersInvocationsAdaptive`（GraphQL Analytics API），查到 `2026-11-18 14:00 UTC`（即你昨天测试时段）的真实指标如下：

- **请求数 (`requests`)**：`56` 次（`errors: 0`）
- **CPU Time**：P50 = **`5.10 ms`**（5,102 µs），P99 = **`14.05 ms`**（14,049 µs）
- **Wall Time**：P50 = **`50.001 s`**（50,001,070 µs），P99 = **`50.004 s`**（50,004,480 µs）

**数据印证**：昨天 56 次请求的 P50/P99 Wall Time 全部精准卡在 **50.00 秒**，正是因为修复前 `emqx_worker.mjs` 的 `burstDeadline` Bug 导致哪怕用户提前回复，Worker 也硬生生等满了 `waitMs = 50000` 才返回。

---

## 二、核心问题逐一解答

### 1. 「CPU 才 5ms、Wall Time 50s，岂不是可以无上限地等？如果超过 10ms CPU 会怎样？」

#### (1) Wall Time 可以无上限地等吗？
- **Worker 引擎层**：Cloudflare 官方文档明确规定，**Incoming HTTP request 的 Wall Time 没有硬性上限（"No hard limit while the client remains connected"）**——只要客户端不断开且不超 CPU，Worker 理论上可以一直挂着。
- **但实际工程中不能无限等，原因有三点**：
  1. **Cloudflare 边缘代理空闲超时**：如果超过约 **100 秒**没有任何响应体字节流出，Cloudflare CDN 边缘层可能触发 `524 A Timeout Occurred` 或断连。
  2. **调用端（沙盒 / `curl` / HTTP Client）超时**：大多数 HTTP 客户端和代理网关默认读超时在 **60s–120s** 之间。
  3. **Worker 部署/运行时更新 Grace Period**：如果长期挂起几分钟，遇到边缘节点滚动更新会被强制终止。
- **结论**：目前 `emqx_worker.mjs` 将单次 `waitMs` 上限设为 **55 秒** 是最稳妥的工程甜点；现在修复了提前退出 Bug 后，**用户一回复即在 ~150ms 内返回**，平时没有消息时最多等 55 秒优雅返回再由 Agent 发起下一轮，不会撞超时。

#### (2) 如果 CPU 超过 10ms 会怎样？（是静默销毁还是报错？）
- **不会静默返回空结果**。当持续超出 Free 计划的 **10ms CPU 上限**时，Cloudflare Runtime 会**立即中止该次执行**，向客户端返回 **`Error 1102`（`Worker exceeded resource limits`，通常表现为 HTTP 500/502/503）**，并在后台 Analytics 中记录为 `exceededCpu`。
- 之所以昨天 `cpuTimeP99` 达到 `14.05 ms` 却依然 `errors: 0`，是因为 Cloudflare Free 计划对**偶发、轻微超限**（如冷启动或单次 TLS 握手波动）有一定弹性容忍，但不可依赖此弹性做重计算。

---

### 2. 「我们目前到底是什么结构？Worker 是每 10 秒等一次，还是硬等？」

- **为什么必须经过 Worker Proxy？**
  因为当前 Agent 运行在受限沙盒/云端网络环境中，**仅允许 80/443 端口的 HTTP/HTTPS 出站**，无法直接连接 EMQX 的 `8883`（MQTTS）等非标准端口。
- **目前真实结构（单次长轮询 / Long-Polling）**：
  1. Agent 调 `POST https://agentplayground.maboco-mark.workers.dev/mqtt/rpc`（443 端口）。
  2. Worker 内部通过 `cloudflare:sockets` 建连 EMQX `8883`（TLS），发 `CONNECT` + `SUBSCRIBE`，然后在一个 `while` 循环里用 `Promise.race([reader.read(), timeout])` **单次硬等最多 `waitMs`（当前上限 55s）**——**不是**在 Worker 里每 10 秒重连一次。
  3. **Plan Alpha 修复前**：收到消息后因 `burstDeadline` Bug 未 `break`，导致每次都等满 50s–55s。
  4. **Plan Alpha 修复后**：一旦收到用户回复（`PUBLISH` 帧），回发 `PUBACK` 后在 `burstMs`（默认 150ms，若设 `returnOnFirstMessage: true` 则 **0ms**）内**立即返回并关闭连接**。

---

### 3. 「Durable Objects (DO) 的最新免费额度、限制是什么？按时间还是按次数？」

Cloudflare 已于 **2025 年 4 月 7 日**起将 **Durable Objects 正式纳入 Workers Free Plan**（但必须使用 **SQLite Storage Backend**，旧版 KV 后端仍仅限 Paid 计划）。最新官方免费额度如下（每日 `00:00 UTC` 重置，超限后报错但不扣费）：

| 计费维度 | Free Plan 每日免费额度 | 折算与说明 |
| :--- | :--- | :--- |
| **Requests（请求/消息数）** | **100,000 次 / 天** | 包含 HTTP 请求、RPC、Alarm；WebSocket 消息按 **20:1** 折算（即 200 万条 WS 消息 = 10 万次请求） |
| **Duration（实例活跃时长）** | **13,000 GB-s / 天** | 按每个 DO 128MB (0.125 GB) 内存算：`13,000 / 0.125 = 104,000 秒/天`（约 **28.8 小时/天**，即**单实例 24 小时全天在线也够用**！） |
| **WebSocket Hibernation** | **休眠期不计 Duration** | 若使用 Hibernation API，客户端连着 WebSocket 但不发消息时，DO 休眠，**Duration = 0** |
| **SQLite 存储** | **5M 行读/天 · 100K 行写/天 · 5 GB** | 足以存储历史消息队列、Pending Approvals 与 iOS PushSubscriptions |

---

### 4. 「上了 Durable Objects 后，在你的 Proxy 请求场景下到底有什么本质区别？」

这是一个非常关键的架构对比——**取决于连接由谁发起**：

1. **如果 DO 只是像现在一样，由 Agent 发 HTTP 请求时才去连 EMQX 8883**：
   - **几乎没有本质区别**！因为 Agent 仍然只能通过 443 发 HTTP 长轮询，而 DO 内部如果靠出站 `cloudflare:sockets` 连 EMQX，在没有入站请求时 DO 仍可能被空闲回收，无法真正永久保活。
2. **如果把结构反过来（DO 真正发挥威力的用法）**：
   - **Mac 端（`p2p-remote-controller`）和手机端（`Pocket Deck`）主动通过 `wss://`（443 端口）连入 DO**（**入站 WebSocket，完美享受 Hibernation 零 Duration 消耗！**）。
   - **本质优化 1（彻底消除空闲丢消息与重复握手）**：Mac 和手机一直连在 DO 上；Agent 思考/推理的空档期，用户发的回复**直接毫秒级写入 DO 内存/SQLite**。Agent 下次发 HTTP `/rpc` 时 **0ms 拿走结果，不需要再连 EMQX，也不受 55 秒建连折腾**。
   - **本质优化 2（真正实现 iOS Web Push）**：Mac 收到需要审批的指令或 Agent 发来 `ask_user` 时，DO 立刻感知并直接调用 `web.push.apple.com` 给 iOS PWA 弹窗，而无状态 Worker 在断开后根本无法感知。
   - **本质优化 3（可彻底省掉 EMQX 中转或仅作备份）**：DO 本身就是全球边缘状态 Hub，Mac <-> DO <-> 手机/Agent 少了一跳公网 Broker 延迟。

**总结建议**：
- **当前阶段（零改动成本）**：刚上线的 **Plan Alpha** 已经把「收到回复还卡 50 秒」的问题彻底解决，且免费额度完全够用，可先实测几天体验延迟改善。
- **下一阶段（需要 iOS Web Push + 0ms 握手）**：由于 Cloudflare **Free 计划现在已经支持 SQLite-backed Durable Objects**（且每天 13,000 GB-s 足够单实例全天跑 + 支持 WS Hibernation），**无需升级 $5/月付费版也能直接上 Plan Bravo**！只需把 Mac/手机的 WS 连到 DO，Agent 继续走 443 HTTPS 调 DO `/rpc` 即可。
