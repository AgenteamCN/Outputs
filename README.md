# Timeline 诊断分析、自动化修复与 Automation Tracker 实现报告

## 1. Timeline Tracking 诊断结论

从你导出的 `compact` Timeline JSON 可以看出外层窗口切换与耗时异常，但**无法直接看到每一步自动化操作与失败原因**：

1. **可观测到的现象**：
   - 5 个新窗口（`52905`, `52907`, `52909`, `52912`, `52915`）加上已有 4 个窗口共 9 个。
   - `07:29:21.895Z`：`link_all_api_keys` 汇总 `total=9, triggered=5, alreadyLinked=0, skipped=3, failed=1`。
   - `initialize_all_windows` 期间，第 1 个新窗口 `52905`（`07:29:33.596Z`）到第 2 个窗口 `52907`（`07:29:49.055Z`）耗时 **15.46 秒**，而后续 `52907 -> 52909 -> 52912 -> 52915` 每个仅耗时 **2.7s ~ 2.9s**；且 `07:30:01.227Z` 又切回 `52905`，最终 `initialize_all_windows` 汇总 `initialized=6, failed=3`。
2. **原有 Timeline 的盲区**：
   - `auto_rescan` 仅针对当前选中的 `selectedWindowId=52915`。
   - `aiStudioBatchRoundRobin`、`aiStudioInitSteps`、`aiStudioInit`、`aiStudioAxHelpers` 内部没有任何 step 级事件上报，只记录了最后一条总结日志，无法得知具体哪 3 个窗口失败、卡在哪个步骤、以及 AX 树拿到了什么状态。

---

## 2. 根因定位与自动化逻辑修复

排查 `aiStudioInitSteps.ts`、`aiStudioBatchRoundRobin.ts`、`aiStudioAxHelpers.ts` 后，定位到以下导致首个窗口异常/重复添加网络规则并失败的问题：

1. **首个窗口激活 Safari 的 Focus Settle 竞态**：
   - `runRoundRobinInitializeAll` 与 `linkAllSafariApiKeys` 仅在 `focusWindowBestEffort` 后等待 `120ms`。当 Safari 从后台被激活时，首个窗口（`52905`）的窗口提升与 WebKit 响应链尚未稳定就触发 `AXPress` / `set_value`，导致后续步骤失败，而 Safari 已在前台后的第 2~5 个窗口全部正常。
   - **修复**：新增 `focusWindowAndSettle`（`apps/p2p-remote-controller/src/services/aiStudioAxHelpers.ts`），首个窗口等待 `280ms`（batch init）/ `240ms`（link keys / single init），后续窗口切换等待 `160ms` / `140ms`。
2. **`configureNetworkAllowlist` 重复添加规则与条件误判**（`apps/p2p-remote-controller/src/services/aiStudioInitSteps.ts`）：
   - 原 `waitForScanCondition` 使用 `fields.length > initialDomainCount || fields.length > 0`，当弹窗内已存在 Domain 输入框时，`fields.length > 0` 立即成立，导致抓到旧输入框或重复点击 `Add to allowlist`。
   - 未检查目标域名（`*`）是否已配置、未复用已有的空 Domain 输入框。
   - **修复**：
     - 若检测到目标域名已存在且无 `Gemini API (default)`，直接关闭弹窗跳过，避免重复添加规则。
     - 若存在空的 Domain 输入框，直接复用而不再点击 `Add to allowlist`。
     - 点击 `Add to allowlist` 后严格等待 `fields.length > initialDomainCount`。
     - `set_value` 重试前先调用 `focusWindowAndSettle(windowId, 180)` 重新稳固窗口焦点。
     - 为所有 `waitForScanCondition` 补齐 `signal` 透传，确保 Emergency Stop 可立即中断。
3. **`tryAdvanceApiKeyNonBlocking` 提前误判**（`apps/p2p-remote-controller/src/services/aiStudioBatchRoundRobin.ts`）：
   - 确认 `Select key` 后改为严格等待 `isApiKeySelected(s)`，避免弹窗关闭动画期间背景 `Network` 标题提前触发条件。

---

## 3. 新增 Automation Tracker（全链路可观测）

1. **核心服务**：`apps/p2p-remote-controller/src/services/automationTracker.ts`
   - 提供 `createAutomationRun(workflow)`、`recordAutomationTrace(...)`、`summarizeScanForTracker(scan)`。
   - 记录每次自动化运行的 `runId`、`workflow`、`windowId`、`pass`、`step`、`outcome`（`started | ok | retry | skipped | failed`）、`observedState` 与 `error`。
   - 同步向 `Timeline` 写入 `source: "automation"` 事件，确保即便在 `compact` 导出级别下也能完整看到自动化路径与失败详情。
2. **全流程埋点**：
   - 已接入 `aiStudioBatch.ts`、`aiStudioBatchRoundRobin.ts`、`aiStudioInit.ts`、`aiStudioInitSteps.ts`、`aiStudioAxHelpers.ts`。
3. **UI 与导出集成**：
   - `DebugConsoleTabs.tsx` 新增 **Automation** 标签页（支持 `Issues Only` 过滤、一键复制 Trace、清空）。
   - `logExport.ts` 与 `useLogExportActions.ts` 将 `automation` 事件及 `automationTraces` / `automationIssues` 纳入 `Quick Copy` 与 `Export Timeline JSON`。

---

## 4. 工具链与验证结果

- 工具链：已全局安装 `pnpm`（v12.21.0）并完成依赖安装（`pnpm install --frozen-lockfile`）。
- 检查与构建：
  - `pnpm p2p:check`（`tsc --noEmit && eslint src`）：通过
  - `pnpm --filter p2p-remote-controller test`：9 个测试文件、33 个测试全部通过（含新增 `automationTracker.test.ts` 与 `aiStudioInitSteps.test.ts`）
  - `pnpm p2p:build`：通过
- 已通过 GitHub GraphQL `createCommitOnBranch` 提交已签名 Verified Bot Commit 到 `a2` 分支：`58a8ef95e6914fb88bc1270cc005acf45954bfd5`。
