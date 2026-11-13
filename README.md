# 调查报告：agyPreviewInstant Latest 版本无法调用 API 问题排查（2026-11-13）

## 结论摘要

1. **我们仓库最近 24–48 小时的代码改动没有破坏 API 调用逻辑。**
   - 对 `Antigravi-ty/agyPreviewInstant`（`HEAD: a5b1291`）过去 48 小时的全部提交（`35f402e` ~ `a5b1291`）及前后端调用链进行了排查，并运行了 `node scripts/syncPrivateProfile.mjs && pnpm check`（`tsc --noEmit`、`eslint`、13 项 Vitest 冒烟测试全部通过，0 错误）。
   - Stream 模式、Background 模式、Tool Hooks 开关、Model/Agent 预设以及 `taskStore` 切片拆分均未破坏 `create_task` -> `generativelanguage.googleapis.com/v1beta/interactions` 的请求构建与下发。
2. **Gemini Interactions API 接口本身并未改版或下线，但 Google 在 2026-11-12 至 2026-11-13 对 Antigravity Managed Agent 实施了配额与权限收紧。**
   - 2026-11-12（JST）起，Google AI Developers Forum 已有明确报告：Antigravity Managed Agent（`antigravity-preview-09-2026`）在 Free Tier 项目上的配额被降为 **0**，直接返回 `HTTP 429`（`limit: 0 requests per day` / `limit: 0 input tokens per minute`）；Google 官方也已明确 Managed Agent API 需要**开启付费结算（Paid API Key）**，不能仅靠 Free Tier 或 Google AI Pro/Ultra 订阅使用。
   - 同时，2026-11-12 随 **Gemini 4 Hydrogen** 发布，Google 同步收紧了风控，社区出现大量 `HTTP 403 PERMISSION_DENIED`（Terms of Service / 第三方或自动化调用拦截）报告，而官方网页端（AI Studio）、Antigravity App 与官方 CLI（`agy`）走的是第一方 OAuth/订阅配额通道，因此会出现“网页端 / App / CLI 可用，但 API Key 通道不可用”的现象。

---

## 一、仓库最近 24–48 小时改动排查

### 1. 验证环境与静态检查
- 在 `/work/agyPreviewInstant`（`a5b1291`）执行：
  - `node scripts/syncPrivateProfile.mjs && pnpm check` -> **通过**（0 TS errors，0 ESLint errors，Vitest `1 passed (13 tests)`）。
  - 注：当前沙盒未安装宿主 C 链接器 `cc` 与 WebKitGTK 系统库，未运行 `cargo test`，但已人工逐行核对 `src-tauri/src/google_api/`、`src-tauri/src/commands/task_commands.rs` 与 `src-tauri/src/models/` 的全部 Rust 源码及 diff。

### 2. 过去 48 小时提交逐项审计
| Commit | 时间 (UTC) | 改动内容 | 对 API 调用的影响评估 |
| --- | --- | --- | --- |
| `dd05502` | 11-12 08:22 | 新增 `HookSettingsModel` 与 `.agents/hooks.json` 自定义配置 | `enableToolHooks` 为 `false` 时 `build_new_sandbox_env` 完全跳过 `sources` 注入；开启时仅修改 hooks JSON 内容，不影响请求顶层结构。 |
| `fefe1de` / `3340391` / `1ad46ab` / `6753e64` | 11-12 09:23–15:12 | 剪贴板 Copy Raw 与后端 `REDACTED_GITHUB_AUTH_HEADER_MARKER` 还原 | 仅当 `prompt` 以 `[REDACTED_GITHUB_AUTH_AND_COMMIT_PREFERENCE]` 开头时触发后端还原，不影响普通 prompt。 |
| `4b248dd` / `fab1672` | 11-13 00:22–01:22 | 新增 `modelPresets` / `agentNamePresets` 与模态框 Hooks 修复 | `buildDraftCreateTaskRequest` 仍按 `draft.agentName.trim() || DEFAULT_AGENT_NAME`、`draft.modelName.trim() || DEFAULT_MODEL_NAME` 传参，未改变请求字段。 |
| `bcf3503` / `a685da5` / `a5b1291` | 11-13 01:46–01:55 | `PrimerModalShell`、UI 组件拆分、`taskStore.ts` 领域切片拆分 | `submitDraftTask` -> `runAsyncDispatch` -> `submitNewTask` -> `invokeCompat("create_task", { req })` 调用链完整无误。 |

### 3. 请求构建核对
`src-tauri/src/google_api/client.rs`、`stream_runner.rs`、`payload_builder.rs` 当前发送的请求为：
- URL: `POST https://generativelanguage.googleapis.com/v1beta/interactions`
- Header: `x-goog-api-key: <api_key>`, `Content-Type: application/json`, `Accept: text/event-stream` (Stream) 或 `application/json` (Background)
- Body:
  - `agent`: `agent_name`
  - `agent_config`: `{ "type": "antigravity", "model": model_name }`
  - `input`: `req.prompt`
  - `stream: true`（Stream 模式）或 `background: true`（Background 模式）
  - `environment`: `{ "type": "remote", "network": { "allowlist": [{ "domain": "*" }] } }`

上述格式与 Google AI Studio 2026-11-12 / 2026-11-13 最新官方文档完全一致，**无需为 API 改版而修改客户端协议代码**。

---

## 二、Google API 侧变更与社区已知报告调查

### 1. API 版本与接口格式是否改版？
- **未改版。** Google AI Studio 于 **2026-11-12** 发布的 *What's new in Gemini 4 Argon* 与 **2026-11-13** 发布的 *What's new in Gemini 4 Carbon* 文档中，示例仍使用 `https://generativelanguage.googleapis.com/v1beta/interactions`，且 Managed Agent 参数仍为 `agent: "antigravity-preview-09-2026"` 与 `agent_config: { "type": "antigravity", "model": "..." }` [1][2][3]。
- 注：`antigravity-preview-05-2026` 已于 **2026-10-05** 下线，若本地持久化配置（OS 应用配置目录或 `Agent Name` 手动输入）仍残留旧值 `antigravity-preview-05-2026` 会导致失败，应确保使用 `antigravity-preview-09-2026` [4]。

### 2. 过去 24–48 小时（2026-11-12 ~ 11-13）的已知外部事件与报告
1. **Free Tier 配额归零（2026-11-12 JST 起，HTTP 429）**
   - Google AI Developers Forum 已有开发者报告：此前在 Free Tier 项目上可正常调用的 Antigravity Managed Agent（`antigravity-preview-09-2026`），自 **2026-11-12 (JST)** 起开始全部返回 `HTTP 429`（`limit: 0 requests per day on Free Tier` / `limit: 0 input tokens per minute on Free Tier`）[5]。
   - Google AI 团队此前（2026-11-05）已在论坛明确回复：Google AI Studio 中的 Agents（包含 Antigravity Preview）**需要绑定付费结算（Paid API Key）的项目**，仅凭 Free Tier 或 Google AI 个人订阅计划无法使用 [6]。
2. **Gemini 4 Hydrogen 发布与 403 风控拦截浪潮（2026-11-12 ~ 11-13）**
   - **2026-11-12** Google 正式发布 **Gemini 4 Hydrogen**（`gemini-4-hydrogen`，取代原有 Flash 命名体系）[7]。
   - 同日（2026-11-12 至 11-13），论坛集中出现大量 `HTTP 403 PERMISSION_DENIED`（`This service has been disabled in this account for violation of Terms of Service`）反馈，包括部分用户反馈仅官方 VS Code 扩展 / 官方 App / 官方交互式终端可用，而脚本化、第三方包装工具或部分客户端请求被风控拦截 [8][9][10]。
3. **为何网页端 / App / CLI 可用而本仓库 API 不可用？**
   - 网页端（AI Studio / Gemini Web）、Antigravity App 与官方 `agy` CLI 使用的是第一方 OAuth 登录与 Google AI Pro / Ultra 订阅配额（或官方客户端白名单通道）。
   - `agyPreviewInstant` 使用的是 **Gemini Developer API Key（`x-goog-api-key`）** 直接请求 `/v1beta/interactions`。当 API Key 所属 Google Cloud 项目未开启付费结算（Free Tier 配额被归零为 0）、或该账号/项目触发了 11-12 后的风控/权限限制时，**无论切换 Stream 模式、Background 模式、开关 Hook 还是更换模型（`gemini-3.8-flash` / `gemini-4-hydrogen` 等），都会在创建交互阶段直接失败**。
   - 补充：`gemini-4-hydrogen` 于 2026-11-12 刚刚发布，目前 Antigravity Agent（`antigravity-preview-09-2026`）文档明确列出的支持模型为 `gemini-3.8-flash`（默认）、`gemini-4-argon`、`gemini-4-carbon` [3]，`gemini-4-hydrogen` 尚不一定已接入 `agent_config.model`；但 `gemini-3.8-flash` 同样全线失败，进一步印证了问题根因在 **API Key 配额/计费策略或账号权限拦截**，而非单一模型或客户端代码。

---

## 三、排查建议（如何定位具体报错）

由于仓库代码本身无异常，建议在本地运行时直接查看任务卡片错误详情（`errors[0].message`）中的原始 HTTP 状态码：
1. **若返回 `HTTP 429`（`limit: 0` / `RESOURCE_EXHAUSTED`）**：确认当前 API Key 所在的 Google Cloud Project 是否已启用 **Paid Billing（付费结算）**；Free Tier 自 2026-11-12 起已无法调用 Antigravity Managed Agent。
2. **若返回 `HTTP 403 PERMISSION_DENIED`**：检查账号是否被 2026-11-12 后的自动化风控误封，或 API Key 权限受限；可尝试在 Google AI Studio 使用同一项目的付费 API Key 发起一次最小 `curl` 测试验证。
3. **若本地配置残留旧 Agent 名称**：在设置或 Composer 的 `Agent Name` 中确认值为 `antigravity-preview-09-2026`（而非已于 10-05 下线的 `antigravity-preview-05-2026`）。

---

## 参考来源
- [1] [What's new in Gemini 4 Argon - Google AI Studio](https://aistudio.google.com/docs/whats-new-gemini-4-argon)
- [2] [What's new in Gemini 4 Carbon - Google AI Studio](https://aistudio.google.com/docs/whats-new-gemini-4-carbon)
- [3] [Antigravity agent - Gemini API - Google AI Studio](https://aistudio.google.com/docs/antigravity-agent)
- [4] [Release notes - Gemini API - Google AI Studio](https://aistudio.google.com/docs/changelog)
- [5] [Question about Free Tier model availability for Gemini 4 Argon and Carbon in the Antigravity Managed Agent API - Google AI Developers Forum](https://discuss.ai.google.dev/t/question-about-free-tier-model-availability-for-gemini-4-argon-and-carbon-in-the-antigravity-managed-agent-api/199769)
- [6] [Problem with Antigravity Agent Preview for prompts - Google AI Developers Forum](https://discuss.ai.google.dev/t/problem-with-antigravity-agent-preview-for-prompts/196928)
- [7] [Introducing Gemini 4 Hydrogen for Agentic Workflows - Google Blog](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-hydrogen/)
- [8] [Appeal / Bug: False Positive 403 TOS_VIOLATION after Gemini 4 Hydrogen - Google AI Developers Forum](https://discuss.ai.google.dev/t/appeal-bug-false-positive-403-tos-violation-after-gemini-4-hydrogen-official-vs-code-extension-only/200029)
- [9] [Antigravity CLI 403 after headless reviews run by my AI assistant - Google AI Developers Forum](https://discuss.ai.google.dev/t/appeal-submitted-antigravity-cli-403-after-headless-reviews-run-by-my-ai-assistant-all-automated-agy-use-removed/199919)
- [10] [Appeal: Antigravity CLI (agy) 403 "disabled for violation of Terms of Service" - Google AI Developers Forum](https://discuss.ai.google.dev/t/appeal-antigravity-cli-agy-403-disabled-for-violation-of-terms-of-service-ai-pro-subscriber-official-cli-only/199882)
