# Google Antigravity Agent API 独立调用失效与 Background 模式无输出问题调查报告

## 1. 调查概述

近期大量开发者在 Google AI 开发者论坛（Google AI Developers Forum）反映：**脱离 Google AI Studio 前端界面、直接通过 Gemini Interactions API 调用 Antigravity Managed Agent（托管智能体）时出现严重不可用状态**。具体表现为环境（Environment/Sandbox）虽然成功分配并扣除额度/计费，但智能体在后台完全卡死、无任何输出返回，且无论是切换至最新发布的 **Gemini 4 Carbon / Argon**，还是回退至默认的 **Gemini 3.8 Flash**，均无法解决。

经过对官方论坛及开发者社区的深度检索，该问题确实存在，并且是一个已被多方证实且持续发酵的**服务端架构与异步调度管道（Harness / Background Supervisor）缺陷**。

---

## 2. 论坛核心反馈与重现案例

在 Google AI Developers Forum（`discuss.ai.google.dev`）上，有多篇针对此问题的详细技术排查与复现报告：

### 案例 1：Topic 185863 —— 托管 Antigravity Agent 后台调用完全卡死与计费黑洞
* **论坛帖子**：[*Managed Antigravity Agent setup is excessively complex: auth inconsistencies, stuck interactions, unusable cancellation and unclear billing*](https://discuss.ai.google.dev/t/managed-antigravity-agent-setup-is-excessively-complex-auth-inconsistencies-stuck-interactions-unusable-cancellation-and-unclear-billing/185863)
* **现象描述**：
  * 开发者通过 API 提交请求：
    ```python
    interaction = client.interactions.create(
        agent="antigravity-preview-09-2026",
        environment=env_id, # 或 "remote"
        background=True,
        agent_config={"type": "antigravity", "max_total_tokens": 100000}
    )
    ```
  * API 成功返回 HTTP 200，并返回了有效的 `interaction_id` 和 `environment_id`，初始状态为 `"status": "in_progress"`。
  * **致命问题**：此后无论轮询（Polling）多久，查询结果始终保持 `status: in_progress`，且元数据中 **`created == updated`**（创建时间等于更新时间，表明服务器在分配完容器后根本没有驱动智能体执行任何后续 Step）。
  * **额度消耗与取消失效**：即使完全没有输出，后台仍产生资源占用或计费波动；尝试调用 `:cancel` 或 `/cancel` 取消任务时，API 分别返回 404 或 `invalid_request`，导致任务无法正常终止。
  * **同步调用同样挂起**：在不开启 `background=True` 时，HTTP 连接长时间处于挂起等待状态，既无进度回显也无日志，最终超时断开。

### 案例 2：Topic 184054 —— 09-2026 版本 Background 交互无法检索（404 错误）
* **论坛帖子**：[*Antigravity-preview-09-2026 background interaction returns 404 immediately after successful creation*](https://discuss.ai.google.dev/t/antigravity-preview-09-2026-background-interaction-returns-404-immediately-after-successful-creation/184054)
* **现象描述**：
  * 针对 `antigravity-preview-09-2026`，初始创建交互返回 200，但在数秒后调用 `GET /v1beta/interactions/{id}` 轮询状态时直接返回 `404 Not Found (Requested entity was not found)`。
  * 该问题仅出现在后台执行（`background: true`）模式下，导致异步获取结果的链路彻底中断。

### 案例 3：Topic 186789 —— AQ API Key 凭证认证冲突（400 错误）
* **论坛帖子**：[*GET /v1beta/interactions/{id} returns 400 "Multiple authentication credentials received" for background interactions (AQ. key)*](https://discuss.ai.google.dev/t/get-v1beta-interactions-id-returns-400-multiple-authentication-credentials-received-for-background-interactions-aq-key-since-oct-3-4/186789)
* **现象描述**：
  * 使用 Google AI Studio 生成的 API Key（前缀为 `AQ.`）直接调用 Interactions API 轮询后台结果时，网关反复抛出 `Multiple authentication credentials received. Please pass only one`，导致客户端拿不到数据。

### 案例 4：Topic 183550 & 199769 —— Gemini 3.8 / 4 切换无济于事，命令循环与 503 计费
* **论坛帖子**：[*Gemini 3.8 Flash in Antigravity: Severe Latency, Command Looping, Quota Drain & API 503 Billing Issues*](https://discuss.ai.google.dev/t/gemini-3-8-flash-in-antigravity-severe-latency-command-looping-quota-drain-api-503-billing-issues/183550) 与 [*Question about Free Tier model availability for Gemini 4 Argon and Carbon*](https://discuss.ai.google.dev/t/question-about-free-tier-model-availability-for-gemini-4-argon-and-carbon-in-the-antigravity-managed-agent-api/199769)
* **现象描述**：
  * 开发者尝试切换模型：Gemini 4 Carbon/Argon 刚发布即遭遇严重限流（429 免费额度为 0）以及极端容量不足；
  * 退回 `gemini-3.8-flash` 后，由于 Antigravity Harness 存在管道阻塞和指令循环缺陷，模型极易陷入死循环（如不断重复基础探测命令、反复搜索不存在的文件），空耗 Token 额度且不产生业务输出。

---

## 3. 问题根因分析

为什么 **AI Studio 网页端可以正常使用，而独立 API 调用（特别是 Background 模式）完全瘫痪且扣额度**？

1. **服务端异步调度管道（Background Task Supervisor）挂起**
   * Antigravity Agent 是一个包含沙箱环境（Remote Linux Sandbox）与多轮工具调用循环（Tool-use Loop）的复杂架构。
   * 在 AI Studio 网页端，系统通过 WebSocket / SSE 保持全双工长连接，带有心跳保活与交互式管道驱动。
   * 当通过外部 API 指定 `background: true` 时，任务移交给了 Google 后端无头任务池（Headless Task Pool）。但由于当前版本的 Task Supervisor 存在**管道阻塞（stdin/stdout 描述符未正确解耦重定向，进程等待终端交互）**以及任务状态机同步缺陷，导致沙箱初始化成功后，智能体直接在首个 Hook/Step 阻塞，交互状态永远冻结在 `in_progress`（表现为 `created == updated`）。

2. **计费与沙箱生命周期脱节（Ghost Consumption）**
   * Interactions API 的计费/配额预占机制在接收到创建请求并成功申请 `environment`（沙箱实例）与初始 Agent Loop 时即开始生效。
   * 即使智能体因服务端缺陷未执行用户任务或发生 Deadlock，沙箱的运行时间与基础上下文分配仍被计入配额，造成“**未拿到任何输出，却白白扣除了配额或费用**”的现象。

3. **模型层与 Agent Harness 层的错位**
   * 该问题的症结**不是模型推理本身的算力压力**，而是 **Google Antigravity Agent Harness 与 Interactions API 的调度网关缺陷**。
   * 因此，无论用户在 `agent_config.model` 中配置 `gemini-4-carbon`、`gemini-4-argon`，还是 `gemini-3.8-flash`，都无法绕过底层的 Harness 缺陷。

---

## 4. 应对与临时解决方案（Workarounds）

在 Google 官方完全修复 Interactions API 后台运行池前，社区推荐以下替代与规避方案：

1. **禁用 `background=True`，强制改用流式响应（Streaming Mode）**
   * 官方 Codelab 及社区实践（如 [*Build a Daily Tech Digest Agent with Managed Agents*](https://codelabs.developers.google.com/tech-digest-managed-agent)）指出：当前不要使用 Background 模式，必须显式设置 `stream=True`：
     ```python
     response_stream = client.interactions.create(
         agent="antigravity-preview-09-2026",
         input="Your prompt...",
         environment="remote",
         stream=True,  # 强制实时流式输出，绕过无头轮询队列
     )
     for event in response_stream:
         # 实时捕获输出并重置客户端超时
         print(event)
     ```
   * 流式连接能迫使服务端维持活跃的数据传输通道，极大降低任务在容器启动后挂起的几率。

2. **采用自托管环境（Self-hosted Worker Runtime / Daytona）**
   * 根据 Daytona 官方文档（[*Run Gemini Managed Agents on Daytona*](https://www.daytona.io/docs/en/guides/gemini/gemini-managed-agents/)），可以通过 `gemini-api CLI` 搭建自托管 Worker 执行工具操作，将工具执行和文件状态拉回本地/自有容器，避免 Google 托管 Remote Sandbox 引起的调度阻塞。

3. **设置严格的预算截断控制（Budget Controls）**
   * 为防止死循环空耗额度，每次调用务必在 `agent_config` 中指定 `max_total_tokens`：
     ```python
     agent_config={
         "type": "antigravity",
         "model": "gemini-3.8-flash",
         "max_total_tokens": 30000  # 设定制动上限，避免死循环消耗
     }
     ```

4. **暂时通过 AI Studio 界面或官方 IDE / agy CLI 运行关键任务**
   * 当前 AI Studio 界面内嵌的调度通道优先级最高且经过专门的连接保持优化，在官方修复 API 异步管道前，核心任务建议优先在 AI Studio 或官方 IDE 内部触发。

---

## 5. 参考来源与论坛讨论

* [[Discuss.ai.google.dev] Topic 185863: Managed Antigravity Agent setup is excessively complex: auth inconsistencies, stuck interactions, unusable cancellation and unclear billing](https://discuss.ai.google.dev/t/managed-antigravity-agent-setup-is-excessively-complex-auth-inconsistencies-stuck-interactions-unusable-cancellation-and-unclear-billing/185863)
* [[Discuss.ai.google.dev] Topic 184054: Antigravity-preview-09-2026 background interaction returns 404 immediately after successful creation](https://discuss.ai.google.dev/t/antigravity-preview-09-2026-background-interaction-returns-404-immediately-after-successful-creation/184054)
* [[Discuss.ai.google.dev] Topic 186789: GET /v1beta/interactions/{id} returns 400 "Multiple authentication credentials received" for background interactions](https://discuss.ai.google.dev/t/get-v1beta-interactions-id-returns-400-multiple-authentication-credentials-received-for-background-interactions-aq-key-since-oct-3-4/186789)
* [[Discuss.ai.google.dev] Topic 183550: Gemini 3.8 Flash in Antigravity: Severe Latency, Command Looping, Quota Drain & API 503 Billing Issues](https://discuss.ai.google.dev/t/gemini-3-8-flash-in-antigravity-severe-latency-command-looping-quota-drain-api-503-billing-issues/183550)
* [[Discuss.ai.google.dev] Topic 199769: Question about Free Tier model availability for Gemini 4 Argon and Carbon in the Antigravity Managed Agent API](https://discuss.ai.google.dev/t/question-about-free-tier-model-availability-for-gemini-4-argon-and-carbon-in-the-antigravity-managed-agent-api/199769)
* [[Google Codelabs] Build a Daily Tech Digest Agent with Managed Agents](https://codelabs.developers.google.com/tech-digest-managed-agent)
* [[Daytona Docs] Run Gemini Managed Agents on Daytona](https://www.daytona.io/docs/en/guides/gemini/gemini-managed-agents/)
