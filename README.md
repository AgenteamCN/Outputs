# 工程防崩溃约束、统一弹窗外壳与模块拆分实施报告

已完全采纳并落地该防崩溃与工程约束方案，并同步清理了 Rust 后端的两个 `unused import` 警告，所有改动已通过 Verified Bot Commit 推送至 `origin/a2`（Commit: `bcf350371c7091294f701c9a9d3a0d185c4338df`，`signature.isValid: true`）。

---

## 一、已完成的核心改动

### 1. 硬性静态门禁：ESLint + `eslint-plugin-react-hooks`
- 新增 `eslint.config.mjs`：
  - 启用 `react-hooks/rules-of-hooks: "error"`，在 lint / check 阶段直接拦截条件分支或提前 `return` 之后的 Hook 调用。
  - 启用 `react-hooks/exhaustive-deps: "warn"` 与 `max-lines: ["warn", { max: 450, skipBlankLines: true, skipComments: true }]`，持续暴露超大文件。
- 在 `package.json` 中新增脚本：
  - `pnpm lint` (`eslint src`)
  - `pnpm test` (`vitest run`)
  - `pnpm check` (`tsc --noEmit && pnpm lint && pnpm test`)

### 2. 架构防呆：统一弹窗外壳 `<PrimerModalShell>`
- 新增 `src/components/common/PrimerModalShell.tsx`：
  - 统一托管 `if (!isOpen) return null;` 条件挂载、`useBackdropClose` 点击防误触、`Escape` 快捷键关闭、`document.body` 滚动锁定以及 `role="dialog"` / `aria-modal="true"` 无障碍属性。
  - 内部 `PrimerModalShellContent` 仅在 `isOpen === true` 时挂载，彻底消除各业务弹窗手动维护 `useBackdropClose` 位置的心智负担。
- 全量迁移全部 11 个弹窗外壳（13 个弹窗组件）至 `<PrimerModalShell>`：
  - `ImportTaskModal.tsx`
  - `TaskAbortRefreshModal.tsx`
  - `TaskDeleteConfirmModal.tsx`
  - `EnvironmentDeleteConfirmModal.tsx`
  - `DraftBatchDuplicateModal.tsx`
  - `LanQrScannerModal.tsx`
  - `AppSettingsModal.tsx`
  - `StatementSnippetModal.tsx`（含 `InsertRepoModal.tsx`、`InsertScanModal.tsx`）
  - `EnvironmentManagerModal.tsx`
  - `RequestQuotaModal.tsx`
  - `CredentialsSettingsModal.tsx`

### 3. 超大文件拆解：`CredentialsSettingsModal.tsx`（1074 行 → ~330 行）
按凭据域拆分为独立低耦合子模块（位于 `src/components/settings/credentials/`）：
- `credentialFieldStyles.ts`：共享缺失告警输入/标签样式
- `ApiKeyAndGithubWorkerSection.tsx`：Gemini API Key、GitHub Worker Request ID、Co-Author 与 `required_instruction` 覆盖区
- `EmqxAndTelemetrySection.tsx`：SZ (Bemfa)、EMQX CN、EMQX GB 与 Webhook 配置及连接控制区
- `LegacyGithubPemSection.tsx`：Legacy GitHub App PEM 手动模式配置区

### 4. 轻量烟雾测试：Vitest + React Testing Library
- 新增 `vitest.config.ts`、`src/test/setup.ts` 与 `src/test/modals.smoke.test.tsx`。
- 覆盖全部 13 个弹窗的 `closed -> open -> closed` 状态切换渲染与 `Escape` 关闭测试，实时捕获任何 `Rendered more hooks` 类运行时崩溃。

### 5. Rust 后端编译警告清理
- 移除 `src-tauri/src/google_api/client.rs` 与 `src-tauri/src/google_api/output_recovery.rs` 中未使用的 `redact_sensitive_blocks` 导入，消除 `cargo check` / `pnpm tauri dev` 中的 2 个 `unused_imports` 警告。

---

## 二、验证结果

- `pnpm check`（`tsc --noEmit && pnpm lint && pnpm test`）：
  - `tsc --noEmit`：**通过（0 错误）**
  - `pnpm lint`：**通过（0 错误**，`react-hooks/rules-of-hooks` 0 违规）
  - `pnpm test`：**13/13 弹窗烟雾测试全部通过**
- `pnpm build`（`tsc && vite build`）：**通过**
- GitHub Verified Commit：[`bcf350371c7091294f701c9a9d3a0d185c4338df`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/bcf350371c7091294f701c9a9d3a0d185c4338df)（`signature.isValid: true`, `state: VALID`）
