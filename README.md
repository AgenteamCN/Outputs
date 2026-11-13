# Antigravity / Gemini Interactions API 近期改版、排队与性能差异深度审查报告

**调查日期**：2026 年 11 月 13 日  
**关联仓库分析**：`Antigravi-ty/agyPreviewInstant`（经核查 `src-tauri/src/google_api/client.rs` 与 `payload_builder.rs`，该客户端通过 `POST https://generativelanguage.googleapis.com/v1beta/interactions` 调用 Google Managed Antigravity Agent，默认采用 `background: true`、`agent_config.type = "antigravity"` 及 `environment.type = "remote"`）

---

## 一、 核心结论摘要（TL;DR）

1. **API 确实经历了密集改版与默认模型切换**：
   - **Agent 版本迭代**：旧版 `antigravity-preview-05-2026` 已于 **2026 年 10 月 5 日** 正式停服，全面切换为 `antigravity-preview-09-2026`（9 月 17 日发布）。
   - **底层模型强制路由与换代**：`gemini-3.7-flash` 于 **10 月 8 日** 废弃并自动路由至 `gemini-3.8-flash`；随后 `gemini-4-argon`（10 月 13 日）、`gemini-4-carbon`（11 月 10 日）、`gemini-4-hydrogen`（**11 月 12 日**）陆续上线，且 **`gemini-4-hydrogen` 已成为 Antigravity Managed Agent 的新默认模型**。
2. **从 API 调用 Antigravity 的情况尚未完全恢复，存在严重的“模型池分化”**：
   - **Gemini 3.8 Flash 仍处于间歇性严重拥堵状态**：自 10 月中旬至 **11 月 12–13 日**，官方开发者论坛持续存在大量关于 `gemini-3.8-flash`（尤其是 `high` 档位）的 `503 No capacity available` 报错与单轮 1–3 分钟以上的高延迟投诉。Google 官方工程师在 11 月上旬确认了服务器端容量耗尽及路由队列延迟问题。
   - **Gemini 4 系列响应明显更快**：同一时间、同一网络下，切换至 `gemini-4-argon` 或新发布的 `gemini-4-hydrogen` 通常可立即恢复正常响应速度。
3. **API 调用确实存在“排队（Queue）/ 退避重试”现象，这是导致 API 耗时 1 小时+、而 AI Studio 网页端仅需 10 分钟的核心原因**：
   - **多步 Agent 放大效应**：官方论坛实测表明，当单轮请求遭遇服务器端路由排队或隐式 `503` 退避重试（`4s → 6s → 10s → 24s → 40s → 120s`）时，一次仅 0.01 秒的工具调用会耗费 2–4 分钟等待。30–50 轮的复杂 Agent 任务总耗时会被线性放大至 **1–3 小时**。
   - **`background: true` 异步路径与算力优先级差异**：`v1beta/interactions` 的 `background: true`（强制伴随 `store: true`）走的是服务器端异步多跳调度与远程 Linux Sandbox 分配路径，不仅存在 `status: "in_progress"` 长时间无更新（`created == updated`）的已知卡顿问题，且标准 API / 异步队列在算力紧张时优先级低于 Google AI Studio 网页端交互流式通道及 `Priority inference` 付费高优通道。

---

## 二、 证据一：API 改版、模型弃用与默认路由变化

根据 Google AI Studio 官方更新日志（Changelog）及弃用公告（Deprecations），2026 年 9 月至 11 月间与 Antigravity 及 Interactions API 相关的重大变更如下：

| 时间 | 变更内容 | 对调用方的直接影响 |
| :--- | :--- | :--- |
| **2026-09-17** | 发布 `antigravity-preview-09-2026`，废弃 `05-2026` | 引入新版工具链与 Sandbox 交互契约；部分用户反馈 `09-2026` 在 `background: true` 下出现轮询状态异常或卡死。 |
| **2026-10-05** | **`antigravity-preview-05-2026` 正式停服** | 未迁移的旧版 Agent 请求全部失效，必须使用 `antigravity-preview-09-2026`。 |
| **2026-10-08** | **`gemini-3.7-flash` 废弃** | 所有发往 `gemini-3.7-flash` 的 API 请求被**自动透明路由至 `gemini-3.8-flash`**，大量流量集中涌入 `3.8-flash` 算力池。 |
| **2026-10-13** | 发布 `gemini-4-argon` | 接入 Antigravity，但社区实测显示其上下文缓存易失效、配额消耗速度约为 3.8 Flash 的 3 倍以上。 |
| **2026-11-10** | 发布 `gemini-4-carbon` | 支持 Antigravity Managed Agent，定位为重度推理模型，每轮思考 Token 更多、延迟更高。 |
| **2026-11-12** | **发布 `gemini-4-hydrogen`（GA）** | 官方宣布其速度为 3.8 Flash 的 2.5 倍，**成为 Antigravity Agent 在 Gemini Managed Agents 中的新默认模型**；同日，`gemini-4-argon` 和 `gemini-4-carbon` 在 Antigravity API 的 Free Tier 配额被清零（返回 429）。 |

**关键洞察**：如果客户端（如 `agyPreviewInstant` 的运行时配置）仍显式指定了 `gemini-3.8-flash`（或因习惯沿用旧模型字符串被自动路由到 `3.8-flash`），则无法享受到 11 月 12 日切换默认模型 `gemini-4-hydrogen` 带来的算力分流，反而会持续撞在全网最拥堵的 `3.8-flash` 集群上。

---

## 三、 证据二：Antigravity 调用状态与官方论坛实测（未完全恢复）

在 Google AI Developers Forum（`discuss.ai.google.dev`）上，过去一个月直至 **2026 年 11 月 13 日**，关于 Antigravity 与 Gemini 3.8 Flash 极度缓慢的集中反馈并未平息：

1. **官方承认“服务器端路由队列延迟”与“容量耗尽”**：
   - 在帖子 *Antigravity IDE 2.6.0: agent latency dominates everything*（11 月 6 日发布，用户 `guessingr` 实测 **0.01 秒的本地工具调用，每轮往返耗时达 2–4 分钟**，95% 以上时间耗费在等待模型响应）中，Google 官方人员 `Rebbanpalli_Naveen` 于 **11 月 11 日** 回复确认：
     > *"Between November 4 and November 6, a server-side routing issue caused requests on Gemini 3.8 Flash to experience multi-minute queue delays per turn."*（11 月 4 日至 6 日期间，一个服务器端路由问题导致 Gemini 3.8 Flash 的请求在**每一轮都经历数分钟的队列延迟**。）
   - 在 *Gemini 3.8 Flash stuck on "Working..." indefinitely*（11 月 5 日）与 *Gemini 3.8 Flash slow again on Nov 3*（11 月 3–7 日，45 条回复）中，CLI 日志直接暴露了底层报错：
     `UNAVAILABLE (code 503): No capacity available for model gemini-3.8-flash-high on the server`
     客户端随后进入静默指数退避重试（`4s → 6s → 10s → 24s → 40s → 120s`），而 GUI/异步状态只显示“Working...”或 `in_progress`，不向用户透出 503。
2. **截至 11 月 12–13 日的最新状态**：
   - 虽然官方称 11 月 4–6 日的路由问题已修复，但 **11 月 12–13 日** 的最新帖子 *Critical Performance Issue – Gemini 3.8 Flash Extremely Slow in Antigravity IDE 2.31.2*（199986）与 *Persistent Gemini Flash 503 Errors*（200314）中，仍有大量 Paid Pro 用户报告：**简单提问 `"2+2"` 耗时 2–3 分钟，简单编码/CSS 修改耗时 1–3 小时**。
   - **对照实验证明问题在特定模型池**：同一台机器、同一网络下，当 `gemini-3.8-flash` 卡死数小时时，用户在同一会话或 CLI 中切换为 `Gemini 4 Argon (Low)` 或第三方模型（如 Claude Sonnet 5.5）仅需 **8.9 秒** 即可返回。

---

## 四、 证据三：为什么 AI Studio 网页端 10 分钟完成，API 调用却要 1 小时以上？

结合官方架构文档、GitHub SDK Issue 及论坛技术拆解，网页端与 API 端出现 **6 倍以上耗时差距** 主要由以下 5 个技术与调度因素叠加导致：

### 1. 算力池与调度优先级隔离（Priority vs. Standard/Background）
- Google 在官方文档中明确推出了 **Priority inference（优先推理）**、**Standard（标准）** 与 **Flex** 三个推理层级，明确指出标准 API 及异步队列流量在算力紧张时优先级低于高优流量。
- 2026 年 Q3 财报及 11 月 13 日行业分析均指出 Google Cloud 当前正面临严峻的算力瓶颈（Google 甚至因此宣布 2027 年 1 月起取消 3.8 Flash 折扣、API 价格翻倍）。
- **AI Studio 网页端（Build Mode）** 运行在面向交互式用户的实时流式通道中，且更容易动态路由至新上线的低负载模型池（如 11 月 12 日的 `gemini-4-hydrogen`）；而 **API 端**（尤其是 `v1beta/interactions` 的 `background: true` 异步任务）在服务器高负载时更容易进入后端吞吐队列等待调度。

### 2. `background: true` 的异步调度与“静默卡死（created == updated）”缺陷
- 在 `agyPreviewInstant` 等客户端中，调用 `POST /v1beta/interactions` 时传入 `background: true` 与 `environment: { type: "remote" }`。
- 论坛深度反馈贴 *Managed Antigravity Agent setup is excessively complex: auth inconsistencies, stuck interactions...*（185863）及 Deep Research / Antigravity 相关 Issue（168729）明确记录：`background: true` 与 `stream: true` 走的是**不同的后端执行路径/队列**。在 `antigravity-preview-09-2026` 下，`background: true` 频繁出现任务已受理（返回 `status: "in_progress"`）但后端长期未推进（轮询显示 `created == updated`），甚至需要依赖远程 Linux Sandbox 初始化与多跳（long decode）接力，一旦某次内部接力遇到 503 或容器调度排队，就会拉长整体等待时间。
- 此外，GitHub `googleapis/python-genai#3059`（10 月 5 日）已证实：**开启 `store=true`（`background=true` 强制启用 `store=true`）会显著增加请求等待时间**（仅存储持久化开销就使首字延迟中位数增加了近 3 秒以上）。

### 3. 每轮 503 隐式退避在 Agent 循环中的“乘数效应”
- Antigravity 是多步自主 Agent，一个中等复杂度的编码任务通常需要 **20–60 轮** 模型推理与工具调用交互。
- 网页端若遇瞬时抖动或可快速切换/流式重连；而 API 异步后台执行在每遇到一次 `503 No capacity` 时，服务器端/客户端会执行指数退避（每次几十秒至 2 分钟）。只要有 20–30 轮触发了排队或 503 重试，总耗时就会从 **10 分钟直接膨胀到 60–90 分钟**。

### 4. `antigravity` Managed Agent 无法通过 API 下调 Thinking Level
- 论坛帖子 *Bug on antigravity-preview-latest API, hardcoded unchangable effort level*（197099，10 月 31 日）经官方人员确认：`antigravity` Managed Agent 的 `agent_config` **目前仅支持 `model` 和 `max_total_tokens` 参数，不支持直接设置 `thinking_level`（Low/Medium/High）**。
- 当底层默认/硬编码的推理预算较高（如 `high`），遇到 `gemini-3.8-flash-high`（全网 503 最严重的档位）时，API 调用方无法像网页端或 IDE 那样手动在下拉框中将思考等级降为 `Low` 或 `Medium` 来绕过拥堵池。

### 5. `09-2026` 版本的上下文膨胀与工具搜索循环
- 论坛（183550、198760）多人报告 `gemini-3.8-flash` + 新版 Antigravity Harness 在处理代码库时更容易陷入“重复读取文件 / 全局搜索死循环（searching loop）”。如果远程 Sandbox 环境中未配置完善的 `.gitignore` / `.agentignore` 排除 `node_modules`、`target`、`dist` 等目录，Agent 在 API 后台模式下无人干预地持续循环搜索，也会将原本 10 分钟的任务拖成 1 小时以上。

---

## 五、 针对性排查与优化建议（面向 `agyPreviewInstant` 与 API 调用）

1. **检查并升级 Agent 与模型标识（最优先）**：
   - 确认配置中的 `Agent Name` 为 `antigravity-preview-09-2026`（确保未残留已停服的 `05-2026`）。
   - **立即避免使用 `gemini-3.8-flash`（及会自动路由过去的 `gemini-3.7-flash`）**。建议在配置中将 `Model Name` 切换为 **11 月 12 日刚发布的 `gemini-4-hydrogen`**（新默认模型，官方标称速度提升 2.5 倍且当前算力池相对充裕），或测试 `gemini-4-argon`。
2. **对比测试 `stream: true` 与 `background: true`**：
   - 在 `agyPreviewInstant` 中尝试对比流式模式（`stream: true`）与后台模式（`background: true`）完成同一小型任务的耗时，验证是否因 `background: true` 的异步队列与 `store: true` 路径导致了严重排队。
3. **检查远程 Sandbox 忽略规则**：
   - 确保目标仓库根目录存在规范的 `.gitignore` 或 `.agentignore`，屏蔽依赖目录与构建产物，防止 Antigravity Agent 在无人值守的后台模式中陷入搜索循环。
4. **关注配额与计费等级**：
   - 若使用 Free Tier Key，注意 11 月 12 日起 `gemini-4-argon` / `carbon` 在 Antigravity API 上的免费配额已归零；Paid Tier 如对延迟极其敏感，可评估 Google 新推出的 `Priority inference` 通道（支持 Interactions API 端点）。

---

## 六、 主要参考资料与来源

- Google AI Studio — Release Notes (Changelog, 更新至 2026-11-12): https://aistudio.google.com/docs/changelog
- Google AI Studio — Antigravity Agent 文档: https://aistudio.google.com/docs/antigravity-agent
- Google AI Studio — Interactions API 概览与 Background Execution: https://aistudio.google.com/docs/interactions-overview
- Google AI Studio — Priority Inference 文档: https://aistudio.google.com/docs/priority-inference
- Google AI Studio — Gemini Deprecations: https://aistudio.google.com/docs/deprecations
- Google AI Developers Forum — *Antigravity IDE 2.6.0: agent latency dominates everything* (官方确认 11 月 4–6 日服务器端路由队列延迟): https://discuss.ai.google.dev/t/antigravity-ide-2-6-0-agent-latency-dominates-everything-a-0-01s-tool-call-takes-minutes-measured-reproducible/198760
- Google AI Developers Forum — *Gemini 3.8 Flash slow again on Nov 3: repeated 503 capacity errors* (官方确认 503 容量耗尽与退避): https://discuss.ai.google.dev/t/gemini-3-8-flash-slow-again-on-nov-3-repeated-503-capacity-errors-and-3-minute-waits/197975
- Google AI Developers Forum — *Critical Performance Issue – Gemini 3.8 Flash Extremely Slow (Nov 12–13)*: https://discuss.ai.google.dev/t/critical-performance-issue-gemini-3-8-flash-extremely-slow-in-antigravity-ide-2-31-2-despite-google-ai-pro-subscription/199986
- Google AI Developers Forum — *Gemini 3.8 Flash stuck on "Working..." indefinitely (while Gemini 4 Argon Low works fast)*: https://discuss.ai.google.dev/t/gemini-3-8-flash-stuck-on-working-indefinitely-for-2-days-while-gemini-4-argon-low-works-fast/198327
- Google AI Developers Forum — *Managed Antigravity Agent setup: stuck interactions (created == updated)*: https://discuss.ai.google.dev/t/managed-antigravity-agent-setup-is-excessively-complex-auth-inconsistencies-stuck-interactions-unusable-cancellation-and-unclear-billing/185863
- Google AI Developers Forum — *Bug on antigravity-preview-latest API, hardcoded unchangable effort level*: https://discuss.ai.google.dev/t/bug-on-antigravity-preview-latest-api-hardcoded-unchangable-effort-level/197099
- Google AI Developers Forum — *Question about Free Tier model availability for Gemini 4 Argon and Carbon (Nov 12)*: https://discuss.ai.google.dev/t/question-about-free-tier-model-availability-for-gemini-4-argon-and-carbon-in-the-antigravity-managed-agent-api/199769
- GitHub `googleapis/python-genai` Issue #3059 — *Gemini API latency increased significantly when store=true*: https://github.com/googleapis/python-genai/issues/3059
