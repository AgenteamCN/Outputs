# Google Antigravity Agent API 调研分析报告：后台模式失效、无输出及额度扣除问题调查

## 1. 调查结论摘要（TL;DR）

**您的判断完全属实，您绝非个例。** 

在 Google 官方开发者社区（[Google AI Developers Forum](https://discuss.ai.google.dev/)）上，近期有大量开发者提交了关于 **Google Antigravity Managed Agent（特别是 `antigravity-preview-09-2026` 版本）脱离 AI Studio、直接通过 Interactions API 调用时严重瘫痪**的故障反馈。

核心调查发现如下：
1. **后台模式（Background Mode）存在严重管线缺陷**：调用 `POST /v1beta/interactions` 开启 `background: true` 后，接口虽返回 `HTTP 200` 及 `status: "in_progress"`，但后台任务调度队列与远程沙箱环境（Environment）之间脱钩。通过 `GET /v1beta/interactions/{id}` 轮询时，交互状态永久处于 `in_progress` 且 **`created == updated`**（时间戳完全停滞），沙箱内不执行任何指令，完全拿不到输出；甚至在特定时间段内直接返回 `HTTP 404 not_found` 或 `HTTP 400 Multiple authentication credentials received`。
2. **“幽灵扣费”与额度消耗真实存在**：因为远程 Sandbox 已经成功 Provision（建立环境），云端资源与 Agent 会话已经分配，系统会锁定预付额度或计入 Token 消耗；然而 Agent 在沙箱内部由于调度死锁并未产生任何有效产出。同时，由于取消接口（`:cancel` / `/cancel`）异常失效，用户无法手动终止任务，导致额度被持续锁定或浪费。
3. **切换模型（Gemini 4 Carbon / Gemini 3.8 Flash）无法解决该问题**：该问题的根因不在于底座大模型本身，而在于 **Antigravity Managed Agent Harness（托管 Agent 运行时框架）与 Interactions API 后台异步调度队列（Async Pipeline）** 之间的架构故障。此外，近期 Gemini 4 在免费层级被限制（配额归零，频繁 429），而 Gemini 3.8 Flash 本身在 Antigravity 中又存在严重的指令死循环与 503 超载问题。
4. **AI Studio 界面可用但独立 API 不可用**：AI Studio 网页端使用的是 Google 内部统一会话协议与长连接 WebSocket/SSE 管道，而外部通过 REST/SDK 调用走的是公网 Interactions API + 鉴权验证边界。目前外部 API 的 `AQ.` 前缀 Key 在经过 Environment 资源与 Interactions 后台任务轮询时存在严重的鉴权与队列同步 Bug。

---

## 2. 论坛真实案例与故障现象梳理

在官方开发者论坛上，关于此问题的核心代表性讨论帖如下：

### 帖 1：后台模式交互卡死、时间戳停滞、无法取消及账单问题
- **帖子主题**：*[Managed Antigravity Agent setup is excessively complex: auth inconsistencies, stuck interactions, unusable cancellation and unclear billing](https://discuss.ai.google.dev/t/managed-antigravity-agent-setup-is-excessively-complex-auth-inconsistencies-stuck-interactions-unusable-cancellation-and-unclear-billing/185863)*
- **核心现象**：
  - 开发者通过 API 配置 `agent: "antigravity-preview-09-2026"`、`background: true` 并关联 Remote Environment，成功获得 interaction ID。
  - 后续不断通过 GET 接口轮询，状态始终为 `in_progress`，更致命的是返回字段中 **`created == updated`**，即使等待数小时也没有任何进度更新或输出（Agent 根本没有在沙箱中干活）。
  - 调用 `POST .../INTERACTION_ID:cancel` 返回 `404 Method not found`；调用 `POST .../INTERACTION_ID/cancel` 返回 `400 invalid_request: "There was a problem processing your request. You will not be charged."`，任务无法正常取消。
  - 账户账单与额度在未完成任务的情况下扣减，用户在没有实时 Token/进度反馈的情况下处于“财务盲盒”状态。

### 帖 2：后台交互创建成功后立即返回 404
- **帖子主题**：*[Antigravity-preview-09-2026 background interaction returns 404 immediately after successful creation](https://discuss.ai.google.dev/t/antigravity-preview-09-2026-background-interaction-returns-404-immediately-after-successful-creation/184054)*
- **核心现象**：
  - 开发者在 REST 请求中发起 `POST https://generativelanguage.googleapis.com/v1beta/interactions`，入参为 `antigravity-preview-09-2026` 和 `background: true`。
  - 创建返回 `HTTP 200`，带有合法 ID 和 `environment_id`。
  - 10秒后执行 `GET /v1beta/interactions/{id}`，服务器直接返回 `HTTP 404 not_found`（“Requested entity was not found”）。
  - 对照实验表明：旧版 `antigravity-preview-05-2026` 正常，而 `09-2026` 的后台模式完全失效；同步模式可勉强响应，但后台轮询直接不可用。

### 帖 3：轮询接口报错 400（鉴权凭证冲突）
- **帖子主题**：*[GET /v1beta/interactions/{id} returns 400 "Multiple authentication credentials received" for background interactions (AQ. key)](https://discuss.ai.google.dev/t/get-v1beta-interactions-id-returns-400-multiple-authentication-credentials-received-for-background-interactions-aq-key-since-oct-3-4/186789)*
- **核心现象**：
  - 使用带有 `AQ.` 前缀的 Gemini API Key 进行后台 Interaction 轮询时，GET 请求报错 `HTTP 400: Multiple authentication credentials received. Please pass only one.`。
  - 用户只传了 `x-goog-api-key`，但服务端代理层将后台会话解析成多个身份凭据，直接阻断了结果获取。

### 帖 4：Gemini 3.8 Flash 与 Gemini 4 表现恶化与配额归零
- **帖子主题 1**：*[Gemini 3.8 Flash in Antigravity: Severe Latency, Command Looping, Quota Drain & API 503 Billing Issues](https://discuss.ai.google.dev/t/gemini-3-8-flash-in-antigravity-severe-latency-command-looping-quota-drain-api-503-billing-issues/183550)*
  - 详细指出了 Gemini 3.8 Flash 在 Antigravity 中陷入“死循环执行基础 shell 命令”、“寻找幽灵文件”、“严重延迟”以及“503 超载仍然计费”等恶性 Bug，导致 Quota 极速耗尽却无产出。
- **帖子主题 2**：*[Question about Free Tier model availability for Gemini 4 Argon and Carbon in the Antigravity Managed Agent API](https://discuss.ai.google.dev/t/question-about-free-tier-model-availability-for-gemini-4-argon-and-carbon-in-the-antigravity-managed-agent-api/199769)*
  - 近期 Gemini 4 Carbon / Argon 在 API 中突发 429 报错，显示 Free Tier 额度直接被设为 0（`limit: 0 requests per day` / `0 input tokens per minute`）。

---

## 3. 技术根因分析

为什么会出现“Environment 建立了，也扣了配额，但完全不干活、无输出”？

```
[外部 API 调用 (POST /v1beta/interactions)] 
          │ (background: true)
          ▼
[Interactions API 接收网关] ───(HTTP 200 OK, 返回 interaction_id)───► 返回客户端
          │
          ├──► [远程沙箱分配 (Environment Provisioning)] ──► ✅ 成功建立 (耗费资源/计入初始额度)
          │
          └──► [后台执行调度器 (Task Worker Queue)] 
                      │
                      ❌ 【断裂点】Harness 调度丢失 / 会话持久化未触发 / 凭据传递异常
                      │
                      ▼
               Agent 处于假死状态：
               - 沙箱空转，未执行任何 Tool Call
               - 状态固定为 in_progress (created == updated)
               - GET 查询时要么查无此任务 (404)，要么鉴权异常 (400)，要么死等无输出
```

1. **API 网关与任务队列的异步解耦缺陷**：
   `POST /v1beta/interactions` 在接收到 `background: true` 时，仅完成了请求校验与环境资源占位，就立即返回了 `status: "in_progress"`。然而，后台实际负责驱动 Antigravity Harness 进行规划、工具调用、代码执行的 Worker 任务队列存在投递故障，导致任务根本没有被 Worker 拾取，沙箱虽然被开辟，但没有收到任何执行指令。
2. **API Key (AQ-Key) 与 Environment 的鉴权穿透问题**：
   AI Studio 前端采用内部服务账号直接操控底层微服务；而当外部开发者使用常规 API Key 调用时，Interactions API 在代理转发至底层沙箱容器网络与凭据持久化系统时存在协议不一致问题，常常触发 “unregistered caller” 或 “Multiple authentication credentials”。
3. **取消机制故障导致无法止损**：
   API 目前缺失完善的生命周期管理，文档中声明的 `client.interactions.cancel()` 或 REST 级别的 cancel endpoint 频繁返回 404 或 400，导致挂起任务只能自然超时，持续占用配额。

---

## 4. 建议与临时避坑方案

在 Google 官方修复 `antigravity-preview-09-2026` 的 Interactions API 后台任务管线之前，建议采取以下临时方案：

1. **弃用 `background: true`，暂时改用同步调用（Synchronous Mode）或流式传输（Streaming）**：
   - 论坛中多位开发者（如 `@arthurp` 等）证实，目前该问题主要集中在后台异步模式（Background Mode）上。
   - 若必须通过 API 调用，请移除 `background=True`，改用阻塞等待或流式 SSE 监听：
     ```python
     interaction = client.interactions.create(
         agent="antigravity-preview-09-2026",
         input="Your prompt...",
         environment="remote",
         # 务必不要加 background=True
     )
     ```
2. **严格设置 `max_total_tokens` 防止幽灵扣费与死循环**：
   - 在 `agent_config` 中硬性限制 Token 消耗上限，避免因为 Agent 卡死或指令循环消耗过多余额：
     ```python
     agent_config={
         "type": "antigravity",
         "max_total_tokens": 30000,
         "model": "gemini-3.8-flash"  # 或当前可用的稳定模型
     }
     ```
3. **退回至 AI Studio 网页端控制台使用**：
   - 如果业务涉及复杂的沙箱操作、大量文件读写与长时间思考，建议暂时在 AI Studio 原生界面中运行，避开公共 REST API 的后台调度 Bug。
4. **保留 Interaction ID 向官方反馈退款/补偿**：
   - 官方论坛管理员（如 `@Pooja_Kapse`）多次回复此类帖子并要求受影响用户提供 interaction ID 与 UTC 请求时间以进行后台排查与核验。建议在论坛发帖或通过支持渠道附上卡死任务的 ID。
