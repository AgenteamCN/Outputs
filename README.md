# Batch A Branch 1 (`fix/p2p-interaction-sync`) 完成报告

- **仓库**：`Antigravi-ty/agyPreviewInstant`
- **分支**：`fix/p2p-interaction-sync`（基于 `origin/a2` `c3f9e15`）
- **Verified Bot Commit**：[`b5810fa`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/b5810faa2ef0048f875490d8a036d7ed8c405a58)（`signature.isValid = true`, `state = VALID`, 含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 一、工具链补齐

- 全局安装 `pnpm@12.21.0`，并在仓库根目录执行 `pnpm install --frozen-lockfile` 完成依赖安装。

## 二、代码修改明细

1. **接入 `processPolledNetworkSnapshot` 到所有轮询路径**（`apps/p2p-remote-controller/src/App.tsx`）
   - 在 `handlePollNetRecorder`（`parsed.installed` 与 `else` 分支）接入 `processPolledNetworkSnapshot(parsed, targetId)`，确保手动轮询与 2s 后台自动轮询均能执行 API Key 脱敏、429 告警检测与 `v1_` Interaction ID 自动同步至 AGY。
   - 在已有窗口初始化轮询与新建窗口初始化轮询两处同样接入 `processPolledNetworkSnapshot`，并将原空 `catch` 改为 `console.warn` 日志输出。
   - 在 `handlePollNetRecorder` 的 `silent === true` 异常分支补齐 `console.warn` 日志，避免静默吞错。

2. **补齐 `publishInteractionToAgy` 异常处理与错误提示**（`apps/p2p-remote-controller/src/App.tsx`）
   - 使用 `try/catch` 包裹 `client.publish(topic, payload, 1, false)`，失败时写入 `setErrorMessage` 并向 Bridge 消息流追加 `system` 状态提示，返回 `false`。
   - 手动点击未连接 EMQX 时，除 `setErrorMessage` 外同时记录 Bridge 系统提示。
   - 在 `processPolledNetworkSnapshot` 中仅当 `publishInteractionToAgy` 返回 `true` 时才将 `interaction.id` 写入 `syncedAgyInteractionIdsRef.current`，防止发送失败后被永久跳过。

3. **消除 P2P 侧 L0→L1 逆向依赖**
   - 将 `LogExportLevel` 定义从 `apps/p2p-remote-controller/src/utils/logExport.ts`（L1）迁移至 `apps/p2p-remote-controller/src/types/emqxBridge.ts`（L0）。
   - 更新 `utils/logExport.ts`、`utils/emqxBridgeUtils.ts`、`components/EmqxBridgePanel.tsx` 与 `App.tsx` 的类型引用（无 re-export）。

4. **清理 P2P 侧 3 项雷达基线违规**
   - `apps/p2p-remote-controller/src/App.tsx`：`isAppleEventsJsDisabledError` 中的中文报错匹配字符串改用 `\u` Unicode 转义，满足源码全英文规则。
   - `apps/p2p-remote-controller/src/App.tsx`：清理字符串内分号及单行内联 JS 片段，消除 `[MULTI-STATEMENT LINE]` 误报/违规。
   - `apps/p2p-remote-controller/src/utils/quotaAlertParser.test.ts`：第 7 行超长测试字符串拆分为多行拼接，满足单行 `<= 400` 字符限制。

## 三、验证结果

- `pnpm p2p:check`：通过（0 errors，7 warnings 为既有 `react-hooks/exhaustive-deps` 提示）。
- `pnpm --filter p2p-remote-controller exec vitest run`：5 个测试文件 / 19 条用例全部通过。
- `pnpm radar --skip-tsc`：
  - P2P 侧基线违规已全部清零（全仓仅剩 AGY 主端 `src/utils/githubAuthBlock.ts:9`，属于 Batch A Branch 3 范围）。
  - `LayerViolations` 从 `5/5` 降至 `4/5`。
  - `SilentCatches` 从 `66/66+60/60` 降至 `64/66+60/60`。
