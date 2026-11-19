# Batch A 验收评审、Octopus Merge 结果与 Batch B 执行规划

## 一、Batch A 四条分支逐一评审结论

四个分支整体完成度很高，严格遵守了文件所有权隔离与单向依赖规范，**无需重做**，仅在跨分支合入时存在 1 处小冲突与 4 处跨分支导入对齐（已在合并时一并修复并验证通过）：

1. **`origin/fix/p2p-interaction-sync` (`b5810fa`) — 优秀**
   - 将 `processPolledNetworkSnapshot` 正确接入 `handlePollNetRecorder` 与初始化轮询，彻底打通 Interaction ID 自动同步与 429 告警链路。
   - 为 `publishInteractionToAgy` 补齐 `try/catch` 与可见错误提示，并修复了“未连接时仍将 `interaction.id` 写入 `syncedAgyInteractionIdsRef` 导致后续无法重发”的隐性 Bug。
   - 将 `LogExportLevel` 移至 `types/emqxBridge.ts`，消除了 P2P 侧唯一的 L0→L1 逆向依赖；清除了 CJK 字面量、多语句行与测试超长行 3 项雷达基线报错。

2. **`origin/refactor/p2p-views-and-router` (`83a327a`) — 良好（已自动拆分细粒度子组件）**
   - 仅新建文件、零改动 `App.tsx`，完全避免了与分支 1 的合并冲突。
   - 主动将大面板进一步细分为 `PermissionAutomationBar.tsx` (344L)、`QuotaAlertsBanner.tsx` (88L)、`ManagedWindowSidebar.tsx` (166L)、`AxElementInspector.tsx` (139L)、`DebugConsoleTabs.tsx` (337L)、`DebugConsoleNetworkTab.tsx` (252L)、`emqxCommandRouter.ts` (274L)、`emqxCommandHandlers.ts` (343L)，全部控制在 350 行以内。
   - **唯一跨分支对齐点**：由于基于合并前的 `a2` 切出，`DebugConsoleTabs.tsx`、`PermissionAutomationBar.tsx`、`emqxCommandHandlers.ts` 仍从 `../utils/logExport` 导入 `LogExportLevel`（已在合入时同步改为 `../types/emqxBridge`）。
   - **预期状态**：上述新增模块在 Batch B 分支 5 接入 `App.tsx` 前会被雷达标记为 `Orphans=5`（仅审计提示，不阻断）。

3. **`origin/fix/agy-layers-and-baseline` (`c5637a2`) — 优秀**
   - 彻底解除 AGY 主端 4 处 `utils -> services/stores` 逆向依赖：`rawPromptCopy.ts` 改为 `GithubAuthBlockProvider` 参数注入，`taskExportHydrator.ts` 改为接收只读数据快照参数。
   - 修复 `githubAuthBlock.ts:9` 超长行，并清理确认无引用的孤立模块 `src/utils/draftRegexEngine.ts`（-558L）。

4. **`origin/refactor/agy-clients-and-utils` (`10167da`) — 优秀**
   - 将 `tauriBackendClient.ts`（1,120L）按域拆分为 `tauriIpcTransport.ts` (256L)、`credentialsClient.ts` (189L)、`taskIpcClient.ts` (266L)、`lanIpcClient.ts` (109L)、`telemetryIpcClient.ts` (164L)。
   - 将 `draftRegexDuplicate.ts`（1,137L）拆分为 `draftDuplicateRules.ts` (365L)、`draftRegexEvaluation.ts` (393L)、`draftBatchGenerator.ts` (351L)。
   - 两个巨石文件直接从硬超标清单清出，且同步消除了 10 个空 `catch` 与 10 个静默 Promise `catch`。

---

## 二、Octopus Merge 与棘轮收紧结果

已完成全部 4 条分支向 `a2` 的 **Verified Bot Octopus Merge**，并同步收紧雷达棘轮：

- **Octopus Merge 提交**：`82380d1`（`Merge Batch A branches into a2 (Octopus Merge)`，GitHub Bot 签名 `verified: true`）
  - 解决 `src/utils/rawPromptCopy.ts` 唯一合并冲突（保留依赖倒置后的无 `services` 导入版本）。
  - 更新 `src/components/composer/ComposerActionBar.tsx` 导入路径为 `../../services/credentialsClient`。
  - 更新 P2P 侧 3 个新文件从 `../types/emqxBridge` 导入 `LogExportLevel`。
- **棘轮收紧提交**：`e95e292`（`chore(radar): tighten Batch A arch-debt and layer ratchets`，GitHub Bot 签名 `verified: true`）
  - `HARD_LAYER_VIOLATION_LIMIT`: `5 → 0`
  - `HARD_VOLUME_CEILING_FILES_LIMIT`: `21 → 19`
  - `HARD_EMPTY_CATCH_LIMIT`: `66 → 56`
  - `HARD_SILENT_PROMISE_CATCH_LIMIT`: `60 → 50`

### 全量验证结果（基于合并后代码）
- `pnpm check`（`tsc --noEmit` + `eslint src` + `vitest run`）：**通过**（2 test files, 16 passed）
- `pnpm p2p:check`（`tsc --noEmit` + `eslint src`）：**通过**（0 errors, 7 pre-existing hook warnings in `App.tsx`）
- `pnpm --filter p2p-remote-controller test`：**通过**（5 test files, 19 passed）
- `pnpm radar --audit`：**通过**（203 files, `Cycles=0`, `LayerViolations=0/0`, `HardVolume=19/19`, `SilentCatches=56/56+50/50`, `Orphans=5`）

---

## 三、Batch B 执行规划（可直接并发启动）

原 Batch B 规划**不需要大改**，3 条分支文件所有权完全零重叠，可立即基于最新 `a2`（`e95e292`）并发开工：

| 并行分支 | 责任范围（独占文件，零冲突） | 核心任务 | 验收标准 |
| :--- | :--- | :--- | :--- |
| **5. `refactor/p2p-hooks-shell`** | `apps/p2p-remote-controller/src/App.tsx` + 新建 `hooks/{useSafariWindowManager,useNetworkRecorder,useEmqxBridge,usePocketDeckBridge}.ts` + 接入 Batch A 的 5 个新模块 | 1. 将 `App.tsx`（3,912L / 90 Hooks）压降至 `<= 350` 行纯编排壳层；<br>2. 接入 `PermissionAutomationBar`、`ManagedWindowSidebar`、`AxElementInspector`、`DebugConsoleTabs`、`emqxCommandRouter`；<br>3. 清理 `App.tsx` 内剩余 32 处静默 `catch`（5 empty + 27 promise）。 | `App.tsx <= 350L`、`Hooks <= 20`、`Orphans` 归零；`pnpm p2p:check`、`p2p test` 与 `pnpm radar` 通过。 |
| **6. `refactor/agy-stores-slim`** | `src/stores/telemetryStore.ts`（1,103L）、`src/stores/presetStore.ts`（622L）、`src/stores/themeStore.ts`（625L）+ 新建 `src/utils/` 纯计算模块 | 1. 将纯归档/索引/分片计算下沉至 `src/utils/`；<br>2. 清理 `themeStore.ts` 内 18 处空 `catch {}`。 | 3 个 Store 均 `<= 400L`，`SilentCatches` 显著下降，`pnpm check` 与 `pnpm radar` 通过。 |
| **7. `refactor/agy-ui-hotspots`** | `src/components/drafts/BatchDuplicateConfigPanel.tsx`（835L）、`src/components/layout/AppTopHeader.tsx`（784L）、`src/components/layout/TaskHistorySidebar.tsx`（632L / 26 Hooks）+ 对应子组件 / `src/hooks/` | 按子视图槽位拆分为独立展示组件与 `useTaskSidebar*` Hook。 | 3 个 UI 巨石文件全部退出硬超标清单，`pnpm check` 与 `pnpm radar` 通过。 |

### Batch B 合入后的下一轮建议（Batch C 预告）
Batch B 完成后，预计硬超标文件将从 `19` 个降至 `~11` 个，届时可再收紧一轮棘轮（`HardVolume: 19 -> 11`、`ReactHooks: 90 -> 25`、`SilentCatches` 同步下调），并按需开启 Batch C 收尾剩余中大型文件（P2P 侧 `PocketDeckView.tsx` / `aiStudioInit.ts` / `EmqxBridgePanel.tsx`，AGY 侧 `telemetryMessageIngestion.ts` / `taskDiagnosticsExport.ts` / `LanAccessSection.tsx` 等）。
