# 调查报告：agyPreviewInstant 最新版本无法调用 API 原因排查

## 结论摘要

1. **不是仓库代码改坏**：过去 24–48 小时（2026-11-11 至 2026-11-13）仓库内的改动（`taskStore` 切片拆分、Modal 拆分、`PrimerModalShell`、Hook 设置与剪贴板修复等）**未破坏**请求构造或调度链路。`pnpm check`（`tsc --noEmit`、`eslint`、`vitest` 13/13）与 `pnpm build` 均通过。
2. **不是 API 协议大改版导致请求格式失效**：`agyPreviewInstant` 当前使用的 `POST https://generativelanguage.googleapis.com/v1beta/interactions`、`agent: "antigravity-preview-09-2026"`、`agent_config: { type: "antigravity", model: ... }` 及 `background: true` / `stream: true` 仍完全符合 Google AI Studio 2026-11-12/13 最新文档规范（虽然 2026 年 11 月起官方 SDK 默认切到 `v1`，但 `antigravity-preview-09-2026` 预览版托管 Agent 文档仍明确使用 `v1beta`）。
3. **真正原因在 Google 侧（2026-11-12 起生效的配额与风控策略收紧）**：
   - **Free Tier 配额被清零（HTTP 429 `limit: 0`）**：仓库默认模型配置为 `gemini-4-argon`（搭配 `antigravity-preview-09-2026`）。自 **2026-11-12** 起，Google 将 `gemini-4-argon` 和 `gemini-4-carbon` 在 Gemini Interactions API **Free Tier（免费层 API Key）上的配额直接设为 `0`**（`limit: 0 requests per day` / `limit: 0 input tokens per minute`），导致无论使用 `background` 还是 `stream` 模式、无论是否关闭 Hook，都会在 API 侧直接失败。
   - **为什么感觉“只能从网页端 / App / CLI 调用”**：Antigravity App、Antigravity CLI 与 Gemini 网页端走的是**个人订阅配额（Free / Google AI Pro / Google AI Ultra）**，目前仍可调用 `gemini-4-argon`、`gemini-4-carbon` 以及 11 月 12 日新发布的 `gemini-4-hydrogen` / `Orchestrator`；而 `agyPreviewInstant` 走的是 **Google AI Studio API Key（Interactions API）** 通道，受 API 计费层级限制。
   - **同期伴随 403 风控封禁潮**：2026-11-06 及 **2026-11-12 ~ 11-13**（随 Gemini 4 Hydrogen 发布与 Jules 11 月 12 日进入只读状态），Google 对 Antigravity 触发了大规模 `403 PERMISSION_DENIED (TOS_VIOLATION)` 误报/封禁潮，部分账号的 CLI/API 访问也被直接禁用。

---

## 一、仓库本地排查（最近 48 小时改动检查）

### 1. 请求构造与 API 端点检查
在 `src-tauri/src/google_api/` 中检查了核心请求代码：
- `client.rs`：`GOOGLE_API_BASE_URL` 为 `https://generativelanguage.googleapis.com/v1beta`，调用 `POST /interactions`、`GET /interactions/{id}`、`POST /interactions/{id}/cancel`。
- `payload_builder.rs`：`build_create_interaction_body` 构造：
  - `agent`: `"antigravity-preview-09-2026"`（来自 `resolvedProfile.json`）
  - `agent_config`: `{ "type": "antigravity", "model": "gemini-4-argon" }`
  - `background: true`（或 `stream: true`）
  - `environment`: `{ "type": "remote", "network": { "allowlist": [{ "domain": "*" }] } }`；当关闭 Hook（`enable_tool_hooks = false`）时不会注入 `.agents/hooks.json`。
- `stream_runner.rs`：使用 `Accept: text/event-stream` 与 `"stream": true` 请求同一端点 `/v1beta/interactions`。这解释了为何切换 Stream Mode 或关闭 Hook 均无区别——两者撞上的是同一个后端准入/配额限制。

### 2. 最近 48 小时 Commit 审计（2026-11-11 ~ 2026-11-13）
- `a5b1291`（11-13）：将 `taskStore.ts` 拆分为 `taskHydrationSlice`、`taskDraftSlice`、`taskExecutionSlice`、`taskMutationSlice`、`taskSyncSlice` 及 `taskDispatchHelpers`，对外状态与调度签名保持兼容。
- `a685da5` / `bcf3503` / `fab1672` / `4b248dd`（11-13）：UI Modal 拆分、`PrimerModalShell`、模型/Agent 预设字段与 Rules of Hooks 修复。
- `dd05502` / `6753e64`（11-12）：新增 Hook 自定义配置与后端 `collapse_github_auth_header_block` 保护；未改动 Interactions API 基础请求格式。

### 3. 本地验证结果
- 执行 `node scripts/syncPrivateProfile.mjs && pnpm check`（含 `tsc --noEmit`、`eslint src`、`vitest run` 13 个测试）：**全部通过（0 errors）**。
- 执行 `pnpm build`：**构建成功**。

---

## 二、外部 API 变化与社区已知报告

1. **2026-11-12 突发报告：Free Tier 调用 `antigravity-preview-09-2026` + `gemini-4-argon` / `gemini-4-carbon` 返回 429 (`limit: 0`)**
   - Google AI Developers Forum 报告（2026-11-12）：[Question about Free Tier model availability for Gemini 4 Argon and Carbon in the Antigravity Managed Agent API](https://discuss.ai.google.dev/t/question-about-free-tier-model-availability-for-gemini-4-argon-and-carbon-in-the-antigravity-managed-agent-api/199769)
   - 核心现象：此前 Free Tier 项目一直可以通过 `antigravity-preview-09-2026` 正常调用 `gemini-4-argon` 和 `gemini-4-carbon`；但从 **2026-11-12** 起全部开始返回 `HTTP 429`：
     - `Rate limit exceeded for model gemini-4-argon (limit: 0 requests per day on Free Tier).`
     - `Rate limit exceeded for model gemini-4-carbon (limit: 0 input tokens per minute on Free Tier).`
   - 对应政策：Google 定价表本身标注 Gemini 4 Argon / Carbon "Not available on the Free tier"，11 月 12 日起后端正式将免费层配额严格执行为 `0`。

2. **API 版本（`v1` vs `v1beta`）说明**
   - Google 文档 [API versions explained](https://aistudio.google.com/api-reference/api-versions) 指出：2026 年 11 月主版本起，官方 Gen AI SDK 默认改为调用 `v1`（此前默认 `v1beta`）。
   - 但截至 **2026-11-12 / 11-13** 的最新官方指南（[What's new in Gemini 4 Argon](https://aistudio.google.com/docs/whats-new-gemini-4-argon)、[What's new in Gemini 4 Carbon](https://aistudio.google.com/docs/whats-new-gemini-4-carbon)），调用 `antigravity-preview-09-2026` 托管 Agent **仍然明确要求使用 `v1beta`**，因此仓库继续使用 `/v1beta/interactions` 是正确的，无需改切 `v1`。

3. **2026-11-12 ~ 11-13 同期 403 封禁与产品调整**
   - 2026-11-12 Google 发布了新模型 `gemini-4-hydrogen`（搭配 Antigravity Orchestrator），同时 Jules 于 11 月 12 日进入只读状态并合并至 Antigravity。
   - 同期论坛出现大量 `403 PERMISSION_DENIED (TOS_VIOLATION)` 报告（[Nov 12 CLI 403](https://discuss.ai.google.dev/t/appeal-submitted-antigravity-cli-403-after-headless-reviews-run-by-my-ai-assistant-all-automated-agy-use-removed/199919)、[Nov 13 False Positive 403](https://discuss.ai.google.dev/t/appeal-bug-false-positive-403-tos-violation-after-gemini-4-hydrogen-official-vs-code-extension-only/200029)）。如果任务报错为 `403` 而非 `429`，则属于账号/API Key 触发了风控限制，需通过官方申诉表单解封或更换项目 Key。

---

## 三、排查与解决建议

1. **先确认当前失败的具体 HTTP 状态码**：在 `agyPreviewInstant` 任务详情的 `Errors` 面板中查看后端返回的是 `HTTP 429` 还是 `HTTP 403` / `HTTP 400`。
2. **若是 `HTTP 429 (limit: 0)`（最常见）**：
   - **方案 A（继续使用 `gemini-4-argon`）**：在 Google AI Studio 为当前 API Key 所属的 Cloud 项目绑定结算账号（升级为 Paid Tier / Tier 1+）。
   - **方案 B（不升级付费层，切换模型）**：在 `agyPreviewInstant` 的 Composer 模型选择框或 `Settings -> Agent Models` 中，将 `gemini-4-argon` 改为 `gemini-3.8-flash`（`antigravity-preview-09-2026` 的默认底层模型）或测试 `gemini-4-hydrogen`。
3. **若是 `HTTP 403 (PERMISSION_DENIED / TOS_VIOLATION)`**：
   - 检查 Google AI Studio / Cloud Console 项目状态是否被风控停用，或通过官方申诉通道提交恢复请求。
