# Batch A Branch 4 (`refactor/agy-clients-and-utils`) 完成报告

## 1. 交付概览
- **分支**：`refactor/agy-clients-and-utils`（基于 `a2` `c3f9e15`）
- **Verified Bot Commit**：[`10167da`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/10167daa4c3c0abc217c81577ccb0a5685d4cc71)（GitHub `signature.isValid = true`, `state = VALID`，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 2. 拆分结果

### 2.1 `src/services/tauriBackendClient.ts`（1,121L）→ 共享传输层 + 4 个域客户端
按命令域与窄腰契约拆分，无跨模块共享可变状态、无 `export ... from` 门面、无循环依赖：
1. `src/services/tauriIpcTransport.ts`（256L）：`isTauriRuntime`、`invokeCompat`、`onLanUnauthorized`、`reconnectLanWebSocket`，以及通用的 `createTauriOrLanEventSubscription`（统一管理 Tauri `listen` 与 LAN WebSocket 订阅生命周期）。
2. `src/services/credentialsClient.ts`（189L）：凭据状态、GitHub Auth Prompt Block 缓存与刷新、Synced Presets 读写及 `onLanPresetsUpdated` / `onLanCredentialsUpdated`。
3. `src/services/taskIpcClient.ts`（266L）：任务列表/详情/创建/导入/刷新/取消/删除/重命名/标签/已读/置顶、本地存储统计与清理、以及 `subscribeToStreamEvents` / `subscribeToTaskSyncEvents`。
4. `src/services/lanIpcClient.ts`（109L）：`normalizeLanServerStatus`、`checkLanAuthStatus`、`fetchLanServerStatus`、LAN 配置与配对 Token 管理。
5. `src/services/telemetryIpcClient.ts`（164L）：环境详情与缓存/远端列表、遥测分片与快照读写、探针控制及 `onLanTelemetryUpdated` / `onLanProbeControlRequested` / `onLanEnvironmentsUpdated`。

同时顺手清理了原 `tauriBackendClient.ts` 中的 7 处空 `catch` 与 10 处静默 `.catch(() => {})`，改为 `console.warn` / `console.debug` 输出上下文。

### 2.2 `src/utils/draftRegexDuplicate.ts`（1,138L）→ 3 个纯计算管道
1. `src/utils/draftDuplicateRules.ts`（365L）：类型定义、`PROMPT_REGEX_PRESETS`、`splitTextByLastLines`、`generateSequenceTokens`、`inferDefaultBatchDuplicateConfig`。
2. `src/utils/draftRegexEvaluation.ts`（393L）：`normalizeUserRegexPattern`、`compactPlainSegmentsForPreview`、`buildSourceHighlightSegments`、`evaluateTextWithRegexConfig`、`applyEvaluationReplacement`（同时消除了原有的 3 个 `no-useless-assignment` lint 警告）。
3. `src/utils/draftBatchGenerator.ts`（351L）：`buildTitleSourceHighlightSegments`、`applyTitleReplacement`、`buildTruncatedSequencePreview`、`evaluateBatchDuplicatePreview`、`generateBatchDuplicatePayloads`。

### 2.3 调用方更新
- 已更新 `src/components/`、`src/stores/`、`src/utils/rawPromptCopy.ts` 中所有引用路径，并移除 `src/services/tauriBackendClient.ts` 与 `src/utils/draftRegexDuplicate.ts`。

## 3. 验证结果
- `pnpm check`（`tsc --noEmit && eslint src && vitest run`）：**通过**（0 errors，2 个测试文件共 16 个测试全部通过；ESLint warnings 从 40 降至 35，新增 8 个文件均为 0 warning）。
- `pnpm build`（`tsc && vite build`）：**通过**。
- `pnpm p2p:check`：**通过**。
- `pnpm radar --skip-tsc`：
  - `HardVolume`：从 `21/21` 降至 **`19/21`**（8 个新文件全部 `<= 393` 行，均未触发 Soft/Hard 体积告警）。
  - `SilentCatches`：从 `66/66 + 60/60` 降至 **`59/66 + 50/60`**。
  - `Cycles`：`0`；`Re-Exports`：`0`；`LayerViolations`：`5/5`（未新增）。
  - 剩余 4 项基线违规（`githubAuthBlock.ts`、`App.tsx`、`quotaAlertParser.test.ts`）属于同批并行分支 1 与分支 3 的独占范围，未在本分支越界修改。
