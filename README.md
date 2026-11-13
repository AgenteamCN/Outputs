# 架构防崩溃改造与超大文件拆分报告

完全认可该方案（**硬性静态门禁 + 架构防呆 + 渐进拆小文件 + 轻量烟雾测试**），并已完成全部落地与推送（Verified Bot Commits on `a2`：`bcf3503`、`a685da5`、`a5b1291`）。

## 一、已落地改动

1. **ESLint + `eslint-plugin-react-hooks` 硬性静态门禁**
   - 配置 `eslint.config.mjs`，开启 `react-hooks/rules-of-hooks: "error"`、`react-hooks/exhaustive-deps: "warn"` 与 `max-lines` 告警门禁。
   - 在 `package.json` 中提供 `pnpm lint` 与 `pnpm check`。

2. **统一弹窗外壳 `<PrimerModalShell>`（架构防呆）**
   - 新增 `PrimerModalShell.tsx`，统一处理 `if (!isOpen) return null;`、`useBackdropClose`、Escape 关闭、滚动锁定与安全条件挂载，消除各 Modal 手动维护 Hook 顺序的心智负担。
   - 全部 11 个弹窗组件已迁移至 `<PrimerModalShell>`。

3. **Vitest + React Testing Library 弹窗开关烟雾测试**
   - 新增 `vitest.config.ts`、`setup.ts` 与 `modals.smoke.test.tsx`，覆盖全部 13 个弹窗的 `closed -> open -> closed` 状态切换渲染测试，第一时间拦截 `Rendered more hooks` 类崩溃。

4. **超大文件按域拆小（低耦合、指哪儿打哪儿）**
   - **`taskStore.ts`（1625 行 -> 77 行）**：拆分为 `types.ts`、`taskDispatchHelpers.ts`、`taskHydrationSlice.ts`、`taskDraftSlice.ts`、`taskExecutionSlice.ts`、`taskMutationSlice.ts`、`taskSyncSlice.ts` 7 个职责独立的 Slice/Helper，对外保持 `useTaskStore` API 完全兼容。
   - **`CredentialsSettingsModal.tsx`（1074 行）**：按凭据域拆分为 `ApiKeyAndGithubWorkerSection.tsx`、`EmqxAndTelemetrySection.tsx`、`LegacyGithubPemSection.tsx`、`credentialFieldStyles.ts`。
   - **`TaskOverviewSidebar.tsx`（865 行）**：拆出 `TaskOverviewMetadataFields.tsx` 与 `TaskOverviewFooterActions.tsx`。
   - **`TaskConfigurationPanel.tsx`（521 行）**：拆出 `TaskConfigurationRuntimeFooter.tsx`。
   - **`EnvironmentManagerModal.tsx`（704 行）**：拆出 `EnvironmentQuotaBanner.tsx` 与 `EnvironmentManagerToolbar.tsx`。
   - **`RequestQuotaModal.tsx`（621 行）**：拆出 `RequestQuotaColorRulesSection.tsx`。
   - **`StatementSnippetModal.tsx`（573 行）**：拆出 `StatementSnippetRow.tsx`。

5. **清理 `src-tauri` 编译器 `unused_imports` 警告**
   - 移除 `client.rs` 与 `output_recovery.rs` 中未使用的 `redact_sensitive_blocks` 导入，消除 `pnpm tauri dev` 时的 2 条 `unused import` warning。

## 二、验证结果

- `pnpm test`：1 个测试文件，13 个弹窗烟雾测试全部通过（2.54s）。
- `pnpm lint`：0 errors（`react-hooks/rules-of-hooks` 0 违规）。
- `pnpm build`（`tsc && vite build`）：0 错误，构建成功。

## 三、Git 提交记录（Branch `a2`）

- `bcf3503`: `feat(modals/lint/test): add PrimerModalShell, ESLint hooks gate, Vitest modal smoke tests, and split CredentialsSettingsModal`
- `a685da5`: `refactor(ui): split oversized modals and sidebars into focused subcomponents`
- `a5b1291`: `refactor(stores): split taskStore.ts (1625 lines) into modular domain slices`
