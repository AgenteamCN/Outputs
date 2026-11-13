# 防崩溃门禁、统一弹窗外壳与超大组件模块化拆分报告

## 一、方案评估与结论

完全认可原提案中的四项核心约束方向（**静态门禁 + 统一弹窗外壳 + 渐进式模块化拆分 + 弹窗开关烟雾测试**），并已在 `a2` 分支完成落地与推送。

关于 `src-tauri` 构建日志中的两处 Rust 编译警告：
- `src-tauri/src/google_api/client.rs`：`unused import: redact_sensitive_blocks`
- `src-tauri/src/google_api/output_recovery.rs`：`unused import: redact_sensitive_blocks`

这两处属于无实际副作用的未使用导入警告（不影响运行逻辑，但会持续干扰开发期构建日志），已在基准提交 `bcf3503` 中一并清理。

---

## 二、已落地改动

### 1. 静态门禁、统一弹窗外壳与烟雾测试（Commit `bcf3503`）
- **ESLint + Hook 规则门禁**：新增 `eslint.config.mjs` 与 `pnpm lint`，启用 `react-hooks/rules-of-hooks: "error"` 阻断条件分支/提前返回后的 Hook 调用，并配置 `max-lines` 告警抑制单文件继续膨胀。
- **统一弹窗外壳 `<PrimerModalShell>`**：新增 `src/components/common/PrimerModalShell.tsx`，统一接管 `isOpen` 提前返回、`useBackdropClose`、Escape 关闭与背景滚动锁定；全量 11 个弹窗组件（`AppSettingsModal`、`CredentialsSettingsModal`、`StatementSnippetModal`、`RequestQuotaModal`、`EnvironmentManagerModal`、`EnvironmentDeleteConfirmModal`、`DraftBatchDuplicateModal`、`ImportTaskModal`、`LanQrScannerModal`、`TaskDeleteConfirmModal`、`TaskAbortRefreshModal`）已迁移至该外壳。
- **弹窗开关烟雾测试**：新增 `vitest.config.ts`、`src/test/setup.ts` 与 `src/test/modals.smoke.test.tsx`（`pnpm test`），覆盖 `closed -> open -> closed` 状态切换渲染。
- **凭据设置弹窗拆分 & Rust 警告清理**：`CredentialsSettingsModal.tsx` 已拆分为 `src/components/settings/credentials/` 下的独立凭据域 Section；`src-tauri/src/google_api/client.rs` 与 `src-tauri/src/google_api/output_recovery.rs` 中未使用的 `redact_sensitive_blocks` 导入已移除。

### 2. 超大组件按功能/区域进一步解耦拆分（Verified Bot Commit `a685da5fec7c30d105f92df4279e6768d05aa458`）
在 `bcf3503` 基础上，针对高频修改且仍偏大的弹窗与侧栏文件按职责进一步拆分为低耦合子组件：

- `StatementSnippetModal.tsx`（342 行 / 10.6 KB）
  - 拆出 `src/components/composer/StatementSnippetRow.tsx`（305 行 / 8.6 KB）：负责单条代码片段的展示、编辑、复制、插入与删除交互。
- `RequestQuotaModal.tsx`（309 行 / 9.8 KB）
  - 拆出 `src/components/quota/RequestQuotaColorRulesSection.tsx`（342 行 / 10.9 KB）：负责配额阈值与颜色规则编辑区。
- `EnvironmentManagerModal.tsx`（440 行 / 14.2 KB）
  - 拆出 `src/components/environments/EnvironmentQuotaBanner.tsx`（105 行 / 2.9 KB）：负责环境配额状态横幅。
  - 拆出 `src/components/environments/EnvironmentManagerToolbar.tsx`（207 行 / 5.9 KB）：负责筛选、排序与批量操作工具栏。
- `TaskConfigurationPanel.tsx`（364 行 / 13.0 KB）
  - 拆出 `src/components/composer/TaskConfigurationRuntimeFooter.tsx`（191 行 / 4.7 KB）：负责运行时参数与底部状态/操作区。
- `TaskOverviewSidebar.tsx`（479 行 / 16.2 KB）
  - 拆出 `src/components/inspector/TaskOverviewMetadataFields.tsx`（206 行 / 6.1 KB）：负责任务元信息字段展示与编辑区。
  - 拆出 `src/components/inspector/TaskOverviewFooterActions.tsx`（235 行 / 8.2 KB）：负责侧栏底部操作按钮区。

---

## 三、验证结果

- `pnpm test`：通过（`src/test/modals.smoke.test.tsx` 13/13 用例通过）。
- `pnpm lint`：通过（0 errors）。
- `pnpm build`（`tsc && vite build`）：通过（0 errors）。
- `cargo check`（`src-tauri`）：当前沙盒容器未预装 C 链接器 `cc`/`gcc`，无法在容器内完成 Rust 原生链接检查；两处 `unused import: redact_sensitive_blocks` 清理已通过源码核验，建议在本地运行 `pnpm tauri dev` 或 `cargo check` 复核。

---

## 四、Git 提交信息

- 分支：`a2`
- Verified Bot Commit：[`a685da5fec7c30d105f92df4279e6768d05aa458`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/a685da5fec7c30d105f92df4279e6768d05aa458)
- 基准提交：`bcf3503`
