# Batch A Branch 3 (`fix/agy-layers-and-baseline`) 完成报告

## 1. 分支与提交信息
- **仓库**: `Antigravi-ty/agyPreviewInstant`
- **基线分支**: `origin/a2` (`c3f9e15`)
- **目标分支**: `fix/agy-layers-and-baseline`
- **Verified Bot Commit**: `c5637a2d397d9a17e7064ceb5164763d123ed7d5` (`signature.isValid: true`, `state: "VALID"`, 含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)
- **Commit URL**: https://github.com/Antigravi-ty/agyPreviewInstant/commit/c5637a2d397d9a17e7064ceb5164763d123ed7d5

## 2. 完成内容

### (1) 解除 AGY 主端 4 处 `utils -> services/stores` 单向依赖违规
- **`src/utils/rawPromptCopy.ts`** (消除 L1→L2 `tauriBackendClient.ts` 依赖):
  - 新增 `GithubAuthBlockProvider` 接口，将 `getCachedAuthBlock` 与 `fetchAuthBlock` 改为通过 `copyRawTaskPrompt(..., authProvider?)` 参数注入。
  - 更新调用方 `src/components/composer/ComposerActionBar.tsx` 传入 `tauriBackendClient` 的读写函数。
- **`src/utils/taskExportHydrator.ts`** (消除 L1→L3 `environmentStore.ts`、`taskStore.ts`、`telemetryStore.ts` 依赖):
  - 新增 `TaskExportEnvironmentSnapshot` 与 `TaskExportHydrationDeps` 接口，将 `hydrateTasksAndEnvironmentsForExport(targetTasks, deps)` 改造为纯依赖注入管道，同时将探针失败的空 `catch` 改为 `console.warn` 日志输出。
  - 更新 5 处组件调用方注入 store 快照与动作：
    - `src/components/layout/TaskHistorySidebar.tsx`
    - `src/components/inspector/StepExecutionTimeline.tsx`
    - `src/components/settings/sections/advanced/SimplifiedWorkflowExportCard.tsx`
    - `src/components/inspector/TaskOverviewFooterActions.tsx`
    - `src/components/settings/sections/advanced/EmqxDiagnosticsExportCard.tsx`

### (2) 修复雷达基线长行违规
- **`src/utils/githubAuthBlock.ts:9`**:
  - 将 423 字符的 `DEFAULT_GITHUB_COMMIT_REQUIRED_INSTRUCTION_BODY` 第 2 条指令拆分为多行字符串拼接，运行时内容保持完全一致，消除 `[LINE LENGTH OVERFLOW]` 告警。

### (3) 核查并清理孤立模块
- **`src/utils/draftRegexEngine.ts`**:
  - 全仓检索确认 0 处引用，且其全部导出函数均已存在于 `src/utils/draftRegexDuplicate.ts` 中，属于重复遗留死代码，已删除。

## 3. 验证结果
- **工具链**: 已安装 `pnpm@12.21.0` 并完成 `pnpm install --frozen-lockfile` 与 `node scripts/syncPrivateProfile.mjs`。
- **`pnpm check`** (`tsc --noEmit && pnpm lint && pnpm test`): **通过 (exit 0)** — TypeScript 编译、ESLint 检查与 2 个测试文件（16 tests）全部通过。
- **`pnpm radar --skip-tsc --audit`**:
  - `LayerViolations`: `5` → **`1`**（AGY 侧清零，仅剩 P2P 侧 `apps/p2p-remote-controller/src/types/emqxBridge.ts`，属 Branch 1 范围）。
  - `Orphans`: `1` → **`0`**。
  - `SilentCatches`: `66+60` → **`65+60`**。
  - `githubAuthBlock.ts:9` 长行告警已消除（仅剩 P2P 侧 3 项基线违规，属 Branch 1 范围）。
