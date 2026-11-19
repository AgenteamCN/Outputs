# agyPreviewInstant Batch B 验收、Octopus Merge、棘轮收紧与 Batch C 并行规划报告

## 一、Octopus Merge 检验与执行结果

**结论：3 条 Batch B 分支零冲突完成 Octopus Merge，全量编译、Lint、单测与架构雷达全部通过。**

- **合并提交**：[`4a10d76`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/4a10d765bcac60544479dcb965a49359b04972a1)（Verified Bot GPG-Signed Octopus Merge on `a2`）
- **合入分支**：
  - `origin/refactor/p2p-hooks-shell` (`4a97cc3`)
  - `origin/refactor/agy-stores-slim` (`4c72644`)
  - `origin/refactor/agy-ui-hotspots` (`71fbaac`)
- **全量验证结果**：
  - `pnpm check`（`tsc --noEmit` + `eslint src` + `vitest run` 2 files / 16 tests）：**PASS**
  - `pnpm p2p:check`（`tsc --noEmit` + `eslint src`）：**PASS**
  - `pnpm radar`（228 files 全量扫描）：**PASS**

---

## 二、Batch B 各分支完成质量评估

| 分支 | 核心成果 | 指标变化 | 质量评价与遗留注意点 |
| :--- | :--- | :--- | :--- |
| **`refactor/p2p-hooks-shell`** (`4a97cc3`) | `App.tsx` 抽壳 + 接入 Batch A 5 个视图/路由模块 + 抽离 8 个 Hooks / 2 个 Service / 1 个 Utils | `App.tsx`: `3,912L / 90 Hooks` → **`332L / 18 Hooks`**；清零 `App.tsx` 静默 catch | **达标且干净**。需注意新抽出的 `useSafariWindowManager.ts`（`524L / 18.0 KB / 26 Hooks`）与 `useNetworkRecorder.ts`（`486L / 17.9 KB / 21 Hooks`）紧贴 18 KB 硬边界，后续加新功能前建议微调。 |
| **`refactor/agy-stores-slim`** (`4c72644`) | 抽离 `telemetrySnapshotMerge.ts`、`TelemetrySyncCoordinator`、`TelemetryBrokerCoordinator`、`presetPersistence.ts`、`themeStorageAndDom.ts` | `telemetryStore.ts`: `1,103L` → **`399L / 15.5 KB`**；`presetStore.ts`: `622L` → **`344L / 12.6 KB`**；`themeStore.ts`: 清掉 **18 处空 catch** 并退出硬超标 | **优秀**。3 个核心 Store 全部退出硬超标清单，职责清晰无跨层违规。 |
| **`refactor/agy-ui-hotspots`** (`71fbaac`) | 拆解 3 个 UI 巨石为独立展示子组件与 `useTaskSidebar*` Hooks | `BatchDuplicateConfigPanel.tsx` (`835L`)、`AppTopHeader.tsx` (`784L`)、`TaskHistorySidebar.tsx` (`613L / 26 Hooks`) **全部降至 400L / 12KB Soft Limit 以下** | **极佳**。3 个热点全部一次性消化到健康区间。 |

**全局债务变化（Batch A 后 → Batch B 合并后）**：
- 硬超标文件（`> 800L` 或 `> 18 KB`）：`19` → **`12`**（初始 `21`）
- 静默 `catch`（空 `catch {}` + 静默 `.catch(() => ...)`）：`56 + 50` → **`31 + 23`**（初始 `66 + 60`）
- 最大单文件 React Hook 数：`90` → **`27`**（`useEmqxBridge.ts`）
- 循环依赖 / 单向层违规 / 孤立模块 / 多布尔热点 / 无生命周期缓存：**全部保持 `0`**

---

## 三、第三阶段（棘轮收紧）已完成

参考 `RocketCompanion` 的软硬阈值与棘轮风格，已直接在 `a2` 完成第三阶段棘轮收紧并提交 Verified Bot Commit [`06e2ccb`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/06e2ccb34533e1dff093fdc12280a49820bc8a26)：

| 棘轮常量 | 所在文件 | 收紧前 | 收紧后 | 说明 |
| :--- | :--- | :---: | :---: | :--- |
| `HARD_VOLUME_CEILING_FILES_LIMIT` | `scripts/radar/radarArchDebtGuards.ts` | `19` | **`12`** | 锁定当前 12 个，禁止新增任何巨石文件 |
| `HARD_EMPTY_CATCH_LIMIT` | `scripts/radar/radarArchDebtGuards.ts` | `56` | **`31`** | 锁定当前 31 处 |
| `HARD_SILENT_PROMISE_CATCH_LIMIT` | `scripts/radar/radarArchDebtGuards.ts` | `50` | **`23`** | 锁定当前 23 处 |
| `HARD_BOOLEAN_FLAG_CLUSTER_LIMIT` | `scripts/radar/radarArchDebtGuards.ts` | `25` | **`15`** | 对齐 RocketCompanion |
| `HARD_COMPOUND_FLAG_IF_LIMIT` | `scripts/radar/radarArchDebtGuards.ts` | `25` | **`12`** | 对齐 RocketCompanion |
| `HARD_REACT_HOOK_COUNT_LIMIT` | `scripts/radar/radarStructuralCoupling.ts` | `90` | **`28`** | 当前最大 27（`useEmqxBridge.ts`） |
| `HARD_HOST_PENETRATION_LIMIT` | `scripts/radar/radarStructuralCoupling.ts` | `80` | **`55`** | 当前最大 52（`taskMutationSlice.ts`） |
| `HARD_PASSTHROUGH_PROXY_LIMIT` | `scripts/radar/radarStructuralCoupling.ts` | `30` | **`20`** | 防止出现代理膨胀 |
| `HARD_LAYER_VIOLATION_LIMIT` | `scripts/radar/radarBoundaryAndCycles.ts` | `0` | **`0`** | 保持归零 |

---

## 四、剩余超大文件清单（12 个 Hard Limit + 41 个 Soft Limit 概览）

### 1. 剩余 12 个 Hard Limit 文件（`> 800L` 或 `> 18 KB`）
- **P2P 端（2 个）**：
  1. `apps/p2p-remote-controller/src/components/PocketDeckView.tsx`（683L / 23.9 KB）— 含 16 Hooks + 4 处静默 catch
  2. `apps/p2p-remote-controller/src/services/aiStudioInit.ts`（766L / 22.5 KB）— 含 11 处静默 `.catch(() => null)`
  *(注：`EmqxBridgePanel.tsx` 603L / 21.5 KB 仍超 18KB，计入 P2P 第 3 个待拆视图)*
- **AGY Utils 纯管道（3 个）**：
  3. `src/utils/telemetryMessageIngestion.ts`（796L / 24.0 KB）
  4. `src/utils/taskDiagnosticsExport.ts`（718L / 22.3 KB）
  5. `src/utils/taskStepCondenser.ts`（739L / 20.8 KB）— 含 2 处空 catch
- **AGY Settings & UI 组件（5 个）**：
  6. `src/components/settings/sections/LanAccessSection.tsx`（705L / 21.6 KB）
  7. `src/components/settings/sections/HookModificationSection.tsx`（639L / 20.7 KB）
  8. `src/components/settings/sections/AppearanceSection.tsx`（677L / 18.9 KB）
  9. `src/components/environments/EnvironmentTable.tsx`（485L / 18.5 KB）
  10. `src/components/settings/sections/StorageBreakdownTables.tsx`（503L / 18.3 KB）
- **脚本（1 个）**：
  11. `scripts/syncPrivateProfile.mjs`（862L / 23.0 KB）

### 2. Soft Limit（`> 400L` 或 `> 12 KB`）共 41 个（仅列需关注的边缘热点）
- `apps/p2p-remote-controller/src/hooks/useSafariWindowManager.ts`（524L / 18.0 KB / 26 Hooks）
- `apps/p2p-remote-controller/src/hooks/useNetworkRecorder.ts`（486L / 17.9 KB / 21 Hooks）
- `src/utils/taskSimplifiedExport.ts`（619L / 17.9 KB）
- `src/utils/telemetryStateResolver.ts`（573L / 17.6 KB）

---

## 五、新功能开发 vs 后续拆分：决策建议与 Batch C 并行计划

### 1. 当前版本是否足够支持新功能开发？
**完全足够，建议“新功能开发”与“Batch C 剩余拆分”同步并行，无需等待 Batch C 结束！**
原因如下：
1. **核心阻塞已解除**：最大的单点巨石 `apps/p2p-remote-controller/src/App.tsx` 已从 3,870 行压至 332 行，P2P 与 AGY 主端双端硬隔离，独立页面/新功能（如独立 Debug Console、Mobile Viewport AX 页面）可直接基于 `usePocketDeckBridge` 的只读 Snapshot + Action Dispatch 契约开发。
2. **雷达棘轮已兜底**：`pnpm radar` 已锁死 `HardVolume <= 12`、`LayerViolations == 0`、`Cycles == 0`、`ReactHooks <= 28`、`SilentCatches <= 31+23`。只要开发新功能时遵守单文件 `<= 400L / 12KB` 且不新增静默 catch，就不会引入新架构债。

### 2. Batch C：最高效 5 条零冲突并行清理计划（可与新功能分支同步推进）

| 并行分支 | 独占文件范围（零交集） | 核心任务 | 预期收益 |
| :--- | :--- | :--- | :--- |
| **1. `refactor/p2p-ui-and-init`** | `apps/p2p-remote-controller/src/components/{PocketDeckView,EmqxBridgePanel}.tsx`、`services/aiStudioInit.ts` | 拆分 PocketDeck 与 EMQX 面板子视图，将 `aiStudioInit.ts` 按步骤拆分为纯脚本构建器并清理 11 处 `.catch(() => null)` | P2P 端 HardVolume **清零**；SilentCatches `-15` |
| **2. `refactor/agy-telemetry-and-export`** | `src/utils/{telemetryMessageIngestion,taskDiagnosticsExport,taskStepCondenser}.ts` | 将遥测消息摄取、诊断导出、步骤压缩拆分为独立纯转换管道，并清理 `taskStepCondenser.ts` 2 处空 catch | AGY Utils HardVolume **清零**；SilentCatches `-2` |
| **3. `refactor/agy-settings-sections`** | `src/components/settings/sections/{LanAccessSection,HookModificationSection,AppearanceSection,StorageBreakdownTables}.tsx` | 按设置卡片/表单区块拆分为独立子组件 | Settings 面板 HardVolume **清零** |
| **4. `refactor/agy-env-and-scripts`** | `src/components/environments/EnvironmentTable.tsx`、`scripts/syncPrivateProfile.mjs` | 拆出 EnvironmentTable 行/弹窗子组件；将 `syncPrivateProfile.mjs` 拆出模板/路径解析模块 | 全仓 HardVolume **12 → 0 彻底清零** |
| **5. `fix/silent-catch-cleanup`** | `lanPairingUrl.ts`、`networkRecorderScript.ts`、`mqttWebSocketSubscriber.ts`、`environmentStore.ts`、`requestQuotaStore.ts`、`taskExecutionSlice.ts`、`taskHydrationCoordinator.ts`、`formatters.ts`、`emqxBridgeUtils.ts`、`TaskComposerView.tsx`、`TaskInputSnapshotPanel.tsx`、`LanQrScannerModal.tsx` | 集中清理剩余 37 处零散空 `catch` / 静默 `.catch`（改为结构化日志或显式错误上浮） | 全仓 SilentCatches **接近或完全归零** |

**Parallel Rule**：新功能分支只需避免修改上述 Batch C 占用的 12 个大文件，即可与 Batch C 5 条分支 **100% 并发开发**，合入时继续沿用 Octopus Merge + Verified Bot Commit 即可。
