# Google AI Studio 与 Antigravity 长连接机制、断线恢复与防丢进度实战指南

针对你在使用 **Google AI Studio（特别是 Antigravity Agent）** 时遇到的长连接、断线丢进度、`Precondition check failed`、Dashboard 显示 `-1` 以及如何拿回 `interaction_id` 并接力输出的问题，结合 Google 官方 **Interactions API** 文档与社区最新实测，为你拆解底层原理与可落地的解决方案（含官方方案与社区技巧）。

---

## 一、底层原理：Antigravity 到底需不需要保持长连接？

**结论：后端计算本身不需要浏览器死守长连接，但前端网页状态与流式通道（SSE）非常脆弱。**

1. **后端是异步沙盒执行（Background Execution）**
   Google AI Studio 中的 Antigravity（如 `antigravity-preview-05-2026` / `antigravity-preview-09-2026`）底层基于 **Interactions API** [1, 2]。当你下发任务时，Google 会在云端为你分配一个隔离的 Linux 沙盒环境（对应一个 `environment_id`）和一条交互记录（对应一个 `interaction_id`），任务在服务端以 `background: true` 的方式异步执行（状态为 `in_progress`）[2, 3]。
2. **你看到的“实时吐字”其实是 `background=true` + `stream=true`（SSE）**
   它并不是纯同步阻塞请求，而是后端在后台跑，同时通过 **Server-Sent Events (SSE)** 实时把 `step.start`、`step.delta` 推送到浏览器 [4, 5]。
3. **为什么有时候刷新/换标签页能接力，有时候断了就永远断了？**
   - **能接力的原因**：如果该 Task 已经**保存（Saved）**（或前端已将 `interaction_id` / 会话标识持久化到 URL 或 Google Drive），刷新或在新标签页打开时，前端能拿到任务句柄，重新向后端发起状态查询或带 `last_event_id` 的流式重连，把云端已经跑出来的步骤拉回来 [5]。
   - **永远断掉的三大原因**：
     1. **未保存导致前端句柄丢失**：如果任务还没保存，网络一抖动或页面刷新，浏览器内存里的 `interaction_id` 直接蒸发，新页面找不到对应任务。
     2. **SSE 长连接本身有已知 Bug（高发！）**：官方社区在 2026 年 11 月的实测反馈指出，当 Background Interaction 的 SSE 事件流持续打开超过约 **5 分钟** 时，有概率触发 `UNAUTHENTICATED` 错误，**直接导致整个后端 Interaction 任务失败**；而同一任务如果改用非流式轮询则能正常跑完 [6]。
     3. **连接中断后未及时落盘**：如果断网发生在首条 `interaction.created` 返回之前，或者前端还没同步状态，就彻底失联了。

---

## 二、为什么 Dashboard / Logs 显示 `-1`（negative one）？怎么拿回 `interaction_id`？

### 1. 为什么显示 `-1` 或查不到？
- 在 AI Studio 的 Logs / Dashboard 中，处于 `in_progress`（未完成）或流式异常中断的任务，其最终输出、Token 消耗和耗时统计尚未结算落库，因此在面板上会显示占位符 `-1`（或一直转圈、无法展开详情）[7, 8]。
- 此外，官方文档和社区实测均指出：**在任务处于 `in_progress` 时，非流式的 `GET` 或日志面板往往看不到中间 `steps`**，必须等到状态流转为 `completed` 才会完整写入 [6]。

### 2. 怎么拿到 `interaction_id` 并捞回结果？（实用 & 社区技巧）
如果你是在网页端用 AI Studio，一旦任务丢失就很难在 UI 上找回。可以通过以下方法提前“截胡”拿到 ID：

- **方法 A：F12 开发者工具抓包（最直接）**
  1. 在 AI Studio 点击发送任务**之前**，按 `F12` 打开浏览器开发者工具 -> 切换到 **Network（网络）** 面板。
  2. 勾选 **Preserve log（保留日志）**，并在过滤框输入 `interactions` 或 `antigravity`。
  3. 任务发出后，查看对应请求（或 EventStream）里的第一条事件 `interaction.created` / `interaction.start`，里面包含 `"id": "v1_..."`（这就是 `interaction_id`）以及 `environment_id` [4, 9]。
  4. **复制保存这个 ID**。如果你使用的是自己的 API Key / 同一个 GCP 项目，哪怕网页卡死、断网，只要任务在后端跑完了，你随时可以用 REST 或 SDK 调 `GET /v1beta/interactions/{interaction_id}` 把完整结果捞回来 [5]。

- **方法 B：油猴脚本（Tampermonkey）自动拦截备份（社区黑科技）**
  如果你经常在网页版跑长任务，可以用油猴脚本劫持页面的 `window.fetch` / `EventSource`：只要检测到响应体或 SSE 流中出现 `v1_` 开头的 `interaction_id` 和 `event_id`，就自动写入浏览器的 `localStorage` 并在屏幕角落显示一个悬浮小标签。这样哪怕页面崩溃或断网刷新，也能从 `localStorage` 里把 ID 翻出来。

- **方法 C：养成“先建后存”的习惯，慎用 Auto-Save**
  社区用户普遍反馈 AI Studio 的 Auto-Save 在长对话刷新时容易回滚或丢进度 [10]。**最佳实践是：开新任务前先写个标题手动点击 Save**，确认 URL 已经变成带持久化 ID 的链接再下发重任务；跑完一轮就手动点一次 Save。

---

## 三、为什么总报 `Precondition check failed`？为什么等一分钟就好了？

你在多轮操作或断线重连后经常遇到的 `Error 400: Precondition check failed`，**官方工程师已经在开发者论坛（`discuss.ai.google.dev`）明确解释过核心原因** [11]：

1. **状态机异步延迟（最常见原因）**：
   即使前端 SSE 流已经吐出了 `interaction.completed` 事件，Google 后端的沙盒环境清理与 Interaction 状态持久化仍有短暂延迟，此时该 Interaction 在数据库里可能还短暂处于 `in_progress` 状态 [11]。
   如果你紧接着拿着它的 ID 作为 `previous_interaction_id` 发起下一轮请求，服务端前置检查发现上一轮不是 `completed`，就会立刻抛出 `400 Precondition check failed` [5, 11]。**等个几十秒到一分钟，后端状态落定为 `completed` 后再发，就恢复正常了。**
2. **环境参数 `environment` 传错**：
   续接 Antigravity 任务时，如果传 `environment="remote"` 会新开一个空白沙盒；要复用原有文件与状态，必须传 `environment=previous_interaction.environment_id` [2, 11]。
3. **网络/地区限制**：
   如果你的代理出口 IP 频繁飘移到不支持的地区，也会触发 `FAILED_PRECONDITION` [12]。

---

## 四、如何彻底不丢进度、强行续传与接力输出？

### 方案 1：改用 API + 纯轮询（最稳妥，抗网络抖动 100%）
如果你的本地网络经常抖动，**强烈建议不要在网页端死扛 SSE 长连接**。使用官方 `google-genai`（Python）或 `@google/genai`（JS）走后台模式 [5]：

1. **提交任务（秒拿 ID，断网也不怕）**：
   设置 `background=True, store=True`。请求会在 1 秒内返回 `interaction.id`，之后即使你拔掉网线，Google 服务器也会继续执行 [5]。
2. **用“短连接轮询”替代“SSE 长连接”**：
   鉴于目前 SSE 长连接开超过 5 分钟有触发 `UNAUTHENTICATED` 的风险 [6]，网络不稳定时**直接关闭 `stream=True`**，改用每隔 5~10 秒发一次普通 `GET` 请求轮询状态，断了直接重试即可：
   ```python
   import time
   from google import genai

   client = genai.Client()

   # 1. 提交后台任务，立刻拿到 interaction.id
   interaction = client.interactions.create(
       agent="antigravity-preview-09-2026",
       input="你的复杂任务描述...",
       environment="remote",
       background=True,
       store=True,
   )
   print("请务必保存此 ID:", interaction.id)

   # 2. 抗断网轮询（每次都是独立短连接，不怕中间断网）
   while True:
       try:
           res = client.interactions.get(id=interaction.id)
           if res.status not in ("in_progress",):
               break
       except Exception as e:
           print(f"网络抖动，5秒后重试: {e}")
       time.sleep(5)

   if res.status == "completed":
       print(res.output_text)
   else:
       print("任务结束，状态为:", res.status)
   ```

### 方案 2：如果一定要实时看输出，使用 `last_event_id` 断点续传
官方 Interactions API 原生支持 SSE 断点续传 [5]：
- 消费流式输出时，把每条事件里的 `event.event_id` 持久化存下来。
- 一旦网络断开，捕获异常并重连，传入 `last_event_id` 即可从断掉的那一刻继续吐字，不会丢失中间数据 [5]：
  ```python
  stream = client.interactions.get(
      id=interaction_id,
      stream=True,
      last_event_id=last_event_id
  )
  ```
  *(注：使用流式重连时，若任务预计超过 5 分钟，建议定期主动断开并用 `last_event_id` 重连，或直接切回方案 1 的轮询，避开 5 分钟连接超时 Bug [6]。)*

### 方案 3：如何从上一次 Interaction “接力继续输出”？
拿到上一次的 `interaction_id` 后，按以下标准流程接力，**可 100% 避开 `Precondition check failed`** [11]：
1. **先做非流式状态确认**：调用 `prev = client.interactions.get(id=prev_id)`，确认 `prev.status == "completed"`（如果还是 `in_progress`，就等几秒再查）[11]。
2. **带上 `previous_interaction_id` 和 `environment_id` 发起下一轮**：
   ```python
   next_turn = client.interactions.create(
       agent="antigravity-preview-09-2026",
       input="请接着刚才的进度继续完成...",
       previous_interaction_id=prev.id,
       environment=prev.environment_id,  # 复用同一个沙盒里的所有文件和上下文！
       background=True,
       store=True,
   )
   ```
   *(如果是普通模型长文本被截断，如 Gemini 4 Hydrogen，在 `v1beta1` 下 SDK >= 2.34.0 支持通过 `continuation_token` 自动续写 [13]。)*

### 方案 4：实在不想在 API 排队、必须用 AI Studio 网页版的折中办法
1. **在云端跑浏览器 / 固定出口 IP**：把网页开在一台网络极稳定的海外 VPS 桌面、云桌面或远程服务器上，本地通过远程桌面连过去看，彻底隔离本地网络抖动。
2. **使用 Webhooks（适合开发者）**：Interactions API 支持配置静态 Webhook（监听 `interaction.completed` / `interaction.failed`）[14]，任务跑完 Google 主动回调通知你，完全不需要维持任何长连接。
3. **拆分大任务（Checkpointing）**：尽量让 Antigravity 每次只做 3~5 分钟以内的子任务，做完立刻把成果写入文件并手动点一次 Save，避免单次长连接跨越 5 分钟雷区。

---

### 参考资料
[1] [Antigravity agent - Gemini API - Google AI Studio](https://aistudio.google.com/docs/antigravity-agent)
[2] [Environments in managed agents - Google AI Studio](https://aistudio.google.com/docs/agent-environment)
[3] [Antigravity preview - Gemini API - Google AI Studio](https://aistudio.google.com/docs/models/antigravity-preview-05-2026)
[4] [Streaming interactions - Gemini API - Google AI Studio](https://aistudio.google.com/docs/streaming)
[5] [How to Run Long-Running AI Tasks With Gemini Background Execution (mirrors AI Studio Background Execution Guide)](https://blog.redlinesoft.net/posts/gemini-interactions-background-execution/)
[6] [Background Interactions fail with UNAUTHENTICATED once the event stream has been open about 5 minutes - Google AI Developers Forum](https://discuss.ai.google.dev/t/background-interactions-fail-with-unauthenticated-once-the-event-stream-has-been-open-about-5-minutes-gemini-4-argon-and-gemini-3-1-pro-preview/200578)
[7] [Logs and datasets - Gemini API - Google AI Studio](https://aistudio.google.com/docs/logs-datasets)
[8] [AI Studio Logs: all tested entries fail to open, but list rows and previews are visible - Google AI Developers Forum](https://discuss.ai.google.dev/t/title-ai-studio-logs-all-tested-entries-fail-to-open-but-list-rows-and-prompt-response-previews-are-visible/198451)
[9] [Deep Research API - Interaction stuck "in_progress" - Google AI Developers Forum](https://discuss.ai.google.dev/t/deep-research-api-interaction-stuck-in-progress-for-some-domains/129020)
[10] [AI Studio AutoSave Failure – Conversation Progress Lost After Refresh - Google AI Developers Forum](https://discuss.ai.google.dev/t/ai-studio-autosave-failure-conversation-progress-lost-after-refresh/83094)
[11] ["Error 400 : Precondition check failed" when reusing interaction+environment from a streamed background interaction with agents - Google AI Developers Forum](https://discuss.ai.google.dev/t/error-400-precondition-check-failed-when-reusing-interaction-environement-from-a-streamed-background-interaction-with-agents/177782/2)
[12] [How to Fix Gemini API FAILED_PRECONDITION Error - AI Free API](https://www.aifreeapi.com/en/posts/gemini-api-request-failed-precondition)
[13] [Developer's guide to Gemini 4 Hydrogen - Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/guides/gemini-4-hydrogen)
[14] [Webhooks - Gemini API - Google AI Studio](https://aistudio.google.com/docs/webhooks)
