# agyPreviewInstant & p2p-remote-controller 优化与问题修复报告

- **Commit**: `3781b45dbd795111af8ae263ca2193312cb471ef` (`origin/a2`, Verified Bot Commit, `signature.isValid: true`)
- **Commit URL**: https://github.com/Antigravi-ty/agyPreviewInstant/commit/3781b45dbd795111af8ae263ca2193312cb471ef

---

## 一、已完成的功能与修复

### 1. Google API Token / Key 脱敏（源头 + EMQX 传输双层防护）
- **源头脱敏**（`apps/p2p-remote-controller/src/services/networkRecorderScript.ts`）：在注入 Safari 页面的 `redactSecrets` 中，对 `fetch` / `XMLHttpRequest` 的 `url`、`requestPreview`、`responsePreview` 以及提取的 `input` 在写入缓冲前即完成脱敏。
- **传输与展示脱敏**（`apps/p2p-remote-controller/src/utils/networkSanitizer.ts`）：新增 `redactSensitiveTokens` 与 `sanitizeNetworkSnapshot`，在 `processPolledNetworkSnapshot`（本地轮询与 EMQX `net_record_poll` 响应）及 `Copy Net JSON` 时再次兜底脱敏，确保明文 EMQX 通道不泄露密钥。
- **覆盖的凭证模式**：
  - Google AI Studio / Cloud API Key：`AQ.[A-Za-z0-9_-]{16,}`、`AIza[0-9A-Za-z_-]{20,}` -> `[REDACTED_GOOGLE_API_KEY]`
  - Google OAuth Token：`ya29.[0-9A-Za-z_-]{16,}` -> `[REDACTED_OAUTH_TOKEN]`
  - GitHub Token：`ghp_/ghs_/gho_/ghu_/ghr_`、`github_pat_` -> `[REDACTED_GITHUB_TOKEN]`
  - Bearer Token：`Bearer ...` -> `Bearer [REDACTED_TOKEN]`
  - Cloud API Key 资源 UUID：`projects/.../locations/.../keys/<uuid>` -> `keys/[REDACTED_KEY_ID]`

### 2. Interaction ID 列表排版与换行修复
- **修复位置**：`apps/p2p-remote-controller/src/App.tsx`（Network Tab 中的 `Captured Interaction IDs` 列表）。
- 将外层行改为 `flexDirection: "row"`, `alignItems: "flex-start"`，左侧信息容器设置 `flex: 1, minWidth: 0, flexDirection: "column"`，并对 `<code>{item.id}</code>` 与时间戳/URL 元数据应用 `wordBreak: "break-all"`, `overflowWrap: "anywhere"`；右侧操作按钮组设置 `flexShrink: 0`，彻底解决长 ID 溢出或挤压按钮的问题。

### 3. 过滤 `prompt:` ID，仅保留真实 Interaction ID
- **修复位置**：`apps/p2p-remote-controller/src/services/networkRecorderScript.ts`、`apps/p2p-remote-controller/src/utils/networkSanitizer.ts`。
- 移除 `CreatePrompt` 响应中提取 `prompt:<id>` 的逻辑，并在 `addInteraction` 与 `sanitizeNetworkSnapshot` 中显式过滤 `prompt:` / `prompts/` 前缀，避免误将 Prompt 文档 ID 当作 Task Interaction ID。

### 4. Remote Controller <-> AGY Preview 联动（自动/手动同步 ID + Input）
- **Remote Controller 端**：
  - 在 `networkRecorderScript.ts` 中新增 `extractCreateInteractionInput`，当拦截到 `CreateInteraction` 请求时，从请求体 `parsed[3]` 的嵌套消息数组中提取用户输入的 `prompt` 文本（经脱敏后附着在 `NetworkRecorderInteraction.input`）。
  - 在 `apps/p2p-remote-controller/src/App.tsx` 中新增 `publishInteractionToAgy`、**Auto-Sync to AGY** 开关（默认开启）及每条记录旁的 **Send to AGY** 按钮，通过 EMQX Topic `${prefix}/agy/import_task` 发送 `remote_interaction_detected` 消息（含 `interactionId` 与 `input`）。
- **AGY Preview 端**：
  - `src/stores/telemetryStore.ts`：订阅 `+/agy/import_task`，新增 `registerRemoteImportTaskHandler` 与 `tryHandleRemoteImportMessage`（带去重集合 `handledRemoteImportIds`）。
  - `src/App.tsx`：注册处理器自动调用 `useTaskStore.getState().importTaskById(interactionId, input)`。
  - `src/stores/task/taskExecutionSlice.ts` & `src/stores/task/types.ts`：`importTaskById` 支持接收 `initialInput`，当任务的 `fullPrompt` 为空或处于占位状态时自动填入 `fullPrompt` 与 `promptPreview`。

### 5. HTTP 429 / Quota Exceeded 响应检测、全局通知栏与已读机制
- **解析器**：`apps/p2p-remote-controller/src/utils/quotaAlertParser.ts`（`formatQuotaAlertSummary`、`parseQuotaAlertFromEntry`）。
  - 在 `networkRecorderScript.ts` 中对 `status >= 400` 的响应放宽截取上限至至少 `2000` 字符，防止 `Please retry in ...` 被 600 字符上限截断。
  - 针对 `Quota exceeded for metric: <metric>` + `Please retry in <duration>` 精确输出：`Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_input_token_count, retry in 4.304696932s.`
  - 针对 `Project environment storage quota exceeded.` 与 `Resource has been exhausted (e.g. check quota).` 直接输出清晰摘要。
  - 未知 429 错误完整输出提取后的错误信息，便于后续扩充规则库。
- **全局通知栏与已读机制**：`apps/p2p-remote-controller/src/App.tsx` 在顶部栏下方渲染全局 `warn-banner`，展示未读 429 告警数、接口名、精简摘要、时间与 `windowId`，支持展开 `Raw` 原文、单条 **Mark Read（已读）** 以及 **Mark All Read（全部已读）**，并基于 `entry.id` 去重，避免轮询重复弹窗。

### 6. AGY 导入 Task 标题取消 `Task v1_` 前缀
- **Rust 端**：`src-tauri/src/commands/task_commands.rs` 中 `build_imported_task_placeholder` 剥离 `v1_` 前缀并直接以短 ID（前 18 字符）作为 `title`，不再拼接 `"Task "`。
- **前端端**：`src/stores/task/taskExecutionSlice.ts`（`importTaskById`）与 `src/utils/taskGrouping.ts`（`cleanTaskTitle`）同步剥离 `^Task\s+v1_` 前缀，使新导入及历史已存储的导入任务在紧凑的 Task History 中直接显示核心编号，方便定位并清理对应 environment。

---

## 二、测试与校验结果

- `pnpm check`（主应用 `tsc --noEmit` + `eslint src` + `vitest run`）：**通过**（0 errors，2 个测试文件共 16 个测试全部通过）。
- `pnpm p2p:check`（`p2p-remote-controller` `tsc --noEmit` + `eslint src`）：**通过**（0 errors）。
- `pnpm --filter p2p-remote-controller test`（`vitest run`）：**通过**（5 个测试文件共 19 个测试全部通过，包含新增的 `networkSanitizer.test.ts`、`quotaAlertParser.test.ts` 与 `networkRecorderScript.test.ts` 用例）。
- 注：沙箱环境未安装 Rust 工具链，Rust 部分（`src-tauri/src/commands/task_commands.rs`）已做语法检查，但未执行 `cargo check` / `tauri build`。

---

## 三、从抓包 Request / Response 中分析出的关键信息

1. **内部模型代号映射**：在 `CreateInteraction` 请求载荷中可以看到 `"antigravity-preview-09-2026"` 与 `["gemini-4-hydrogen"]` 成对出现，说明当前前端选择的 `models/antigravity-preview-09-2026` 在后端路由到了 `gemini-4-hydrogen`。
2. **两种截然不同的 `429` / gRPC Code `8` 配额耗尽原因**：
   - **环境存储配额满**：`[8,"Project environment storage quota exceeded."]` —— 并非 Token 限流，而是关联项目（`projects/186980883783`）的远端沙箱/环境存储空间已满，必须清理旧 environment 才能继续创建新任务（这也是取消 `Task v1_` 前缀方便定位清理的核心原因）。
   - **Free Tier TPM 限流**：`generativelanguage.googleapis.com/generate_content_free_tier_input_token_count, limit: 100000, model: antigravity` —— 当前绑定的 API Key（`projects/186980883783/locations/global/keys/d9eab9c5-...`）处于 Free Tier，每分钟输入 Token 上限为 `100,000`，超过后会在错误文本末尾给出精确退避时间（如 `Please retry in 4.304696932s.`）。
   - **流式接口连带报错**：`GetInteractionStream` 返回 `[,[8,"Resource has been exhausted (e.g. check quota)."]]`，通常紧随 `CreateInteraction` 配额受限之后出现，且不包含具体 metric，需要结合前序 `CreateInteraction` 的 429 详情一起判断。
3. **`CreatePrompt` 与 `CreateInteraction` 的职责边界**：`CreatePrompt` / `UpdatePrompt` 仅用于在 AI Studio 中创建或保存 Prompt 文档（返回 `prompts/<id>` 并记录 `api_key_reference`），真正触发 Agent 运行并返回 `v1_...` Task ID 的是 `CreateInteraction`。
4. **明文凭证暴露点**：`ListCloudApiKeys` 响应和 `GetTracesLoggingStatus` 请求中直接以 JSON 数组明文传输 `AQ.Ab8RN6...` 格式的 Google API Key 以及完整的 `projects/.../keys/<uuid>` 路径，因此必须在网络录制器与 EMQX 转发层同时做脱敏拦截。
