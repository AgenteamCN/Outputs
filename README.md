# agyPreviewInstant 架构雷达升级与代码拆分规划报告

## 一、核心问题诊断：为什么 `App.tsx` 膨胀到 3,870 行却没被雷达拦住？

### 1. 原版雷达的盲区
原版 `scripts/radar.ts`（123 行）**确实扫描了** `apps/p2p-remote-controller/src/`，但在机制上存在两个致命缺口：
1. **硬上限默认不阻断**：虽然定义了 `HARD_LINE_LIMIT = 800` 和 `HARD_BYTE_LIMIT = 18 KB`，但超标文件仅作为 `[radar] Hard context-budget notices` 打印出来，**只有传入 `--strict` 参数时才会计入 `errors` 并退出非零**。
2. **缺少架构级检测维度**：没有循环依赖检测、没有 `AGENTS.md` 规定的单向依赖层级校验、没有结构耦合/防伪拆分（Anti-Fake-Split）分析、没有 React God-Component（Hook 爆炸）度量，也没有“棘轮上限（Ratchet）”来阻止架构债继续恶化。

### 2. 不只是 Remote Controller，AGY 主端同样复杂度偏高
用升级后的雷达全量扫描 **190 个文件**后，结果非常明确：**Remote Controller（P2P）的 `App.tsx` 是最严重的单点巨石，但 AGY 主端（`src/`）同样积累了大量高体积、高耦合文件**，并非只有 Remote Controller 偏高：

- **硬超标文件（> 800 行或 > 18 KB）共 21 个**：
  - **P2P Remote Controller（4 个）**：
    - `apps/p2p-remote-controller/src/App.tsx`：**3,870 行 / 140.7 KB**（包含 **48 个 `useState`、15 个 `useRef`、16 个 `useEffect`、944 行 JSX**，以及 450+ 行的 `handleEmqxDecision`、280+ 行的 `handleInitialize`、7 个混杂领域）
    - `apps/p2p-remote-controller/src/components/PocketDeckView.tsx`：683 行 / 23.9 KB
    - `apps/p2p-remote-controller/src/services/aiStudioInit.ts`：766 行 / 22.5 KB
    - `apps/p2p-remote-controller/src/components/EmqxBridgePanel.tsx`：603 行 / 21.6 KB
  - **AGY 主端 `src/`（16 个）**：
    - `src/stores/telemetryStore.ts`：1,103 行 / 32.4 KB
    - `src/utils/draftRegexDuplicate.ts`：1,138 行 / 30.6 KB
    - `src/services/tauriBackendClient.ts`：1,121 行 / 30.5 KB
    - `src/components/drafts/BatchDuplicateConfigPanel.tsx`：835 行 / 26.7 KB
    - `src/components/layout/AppTopHeader.tsx`：784 行 / 24.4 KB
    - `src/utils/telemetryMessageIngestion.ts`：796 行 / 24.0 KB
    - 其余 10 个超 18 KB 文件（`taskDiagnosticsExport.ts`、`LanAccessSection.tsx`、`taskStepCondenser.ts`、`HookModificationSection.tsx`、`presetStore.ts`、`AppearanceSection.tsx`、`TaskHistorySidebar.tsx`、`themeStore.ts`、`EnvironmentTable.tsx`、`StorageBreakdownTables.tsx`）
  - **脚本（1 个）**：`scripts/syncPrivateProfile.mjs`（862 行 / 23.0 KB）
- **软超标文件（> 400 行或 > 12 KB）**：共 **34 个**。
- **循环依赖**：**0 个**（当前保持干净，雷达已锁定为硬性 0 容忍）。
- **单向依赖违规（`AGENTS.md` 规定 `types/config -> utils -> services -> stores -> hooks/components`）**：共 **5 处**：
  - `[agy]` `src/utils/rawPromptCopy.ts` (L1:utils) -> `src/services/tauriBackendClient.ts` (L2:services)
  - `[agy]` `src/utils/taskExportHydrator.ts` (L1:utils) -> `src/stores/environmentStore.ts` (L3:stores)
  - `[agy]` `src/utils/taskExportHydrator.ts` (L1:utils) -> `src/stores/taskStore.ts` (L3:stores)
  - `[agy]` `src/utils/taskExportHydrator.ts` (L1:utils) -> `src/stores/telemetryStore.ts` (L3:stores)
  - `[p2p]` `apps/p2p-remote-controller/src/types/emqxBridge.ts` (L0:types) -> `apps/p2p-remote-controller/src/utils/logExport.ts` (L1:utils)
- **孤立模块**：`src/utils/draftRegexEngine.ts`（0 个传入引用，需确认是否为死代码或待接入）。
- **当前雷达直接拦截的 4 项基线违规**：
  1. `src/utils/githubAuthBlock.ts:9` 单行超 400 字符（423 字符）
  2. `apps/p2p-remote-controller/src/App.tsx:179` 含中文字符 `"允许来自 Apple 事件的 JavaScript"`（用于匹配中文版 Safari 报错，违反 `AGENTS.md` 全英文规则）
  3. `apps/p2p-remote-controller/src/App.tsx:1043` 多语句压缩行（6 处）
  4. `apps/p2p-remote-controller/src/utils/quotaAlertParser.test.ts:7` 单行超 400 字符（611 字符）

---

## 二、已完成：工具链补齐 & 雷达增强（参考 RocketCompanion `746126d`）

参考 `RocketCompanion` 分支 `origin/z/agy/rc/multiplayer-recheck` 的最新提交 `746126d`（`chore(radar): add deterministic arch-debt ratchets and audit guards for Wave 0`），结合 `agyPreviewInstant` 的双端结构（AGY 主端 + P2P Remote Controller）与 `AGENTS.md` 规范，已完成以下增强并提交至 `a2` 分支（Verified Bot Commit `bdcb9cb`）：

1. **工具链安装**：安装 `pnpm@12.21.0` 并完成 `pnpm install --frozen-lockfile`，验证 `pnpm check` 与 `pnpm p2p:check` 均可运行。
2. **模块化雷达架构（`scripts/radar.ts` + `scripts/radar/*`）**：
   - `scripts/radar/radarConstantsAndFs.ts`：体积阈值、目录遍历、相对路径解析、Git 变更文件识别、AGY/P2P 子系统统计。
   - `scripts/radar/radarAstScanners.ts`：英文/CJK/Emoji 校验、单行 400 字符上限、禁止 `Part1`/`Helper1` 等机械拆分命名、禁止多语句压缩行、禁止纯值 re-export 门面（`export ... from`）、`any` 密度上限（<= 5）。
   - `scripts/radar/radarBoundaryAndCycles.ts`：DFS 循环依赖检测（硬性 0 容忍）、跨应用隔离（禁止 `src/` 与 `apps/p2p-remote-controller/` 直接互导）、**单向依赖层级校验**（`types/config -> utils -> services -> stores -> hooks/components`，棘轮上限锁定在当前 5 处，任何新增向上依赖立即报错）。
   - `scripts/radar/radarStructuralCoupling.ts`：移植并扩展了结构耦合与防伪拆分分析（Host 穿透、Pass-Through 代理、React God-Component Hook 密度检测，棘轮上限锁定在 90 Hooks），并支持 `--audit` 输出重构处方（`EXTRACT_CUSTOM_HOOK` / `SPLIT_VIEW_SUBCOMPONENTS` / `PURE_PIPELINE` / `OWN_STATE_SUBCONTROLLER`）。
   - `scripts/radar/radarArchDebtGuards.ts`：移植 Wave 0 棘轮守卫——硬体积超标文件数棘轮（锁定 `21/21`，禁止新增巨石文件）、多布尔标志组合分支热点检测、无 `dispose/clear/reset` 钩子的模块级 `Map/Set` 缓存检测、孤立模块检测。

常用命令：
- `pnpm radar --skip-tsc --audit`：快速全量架构审计与重构处方输出（约 400ms）
- `pnpm radar --strict`：严格模式（所有超 800L/18KB 文件均视为错误）

---

## 三、分阶段拆分与重构计划（等待 AGY 维修完成后按 Wave 执行）

> 严格遵循 `docs/read_this_only_if_you_are_spliting_up_files.md`：**按内聚边界与窄腰契约拆解，严禁拆出共享内部状态的 `*Helper.ts`**。

### Wave 1：优先拆分 `apps/p2p-remote-controller/src/App.tsx`（与 AGY 主端隔离，可独立开工）
目标：将 `App.tsx`（3,870 行 / 90 Hooks）压降至 **<= 350 行** 的纯壳层编排器，并为后续**独立第三方 Safari Remote Controller（Debug Console）及纯 AX 无障碍 UI 的 Mobile Viewport 页面**预留清晰边界。

1. **提取独立 Custom Hooks / 子控制器（`apps/p2p-remote-controller/src/hooks/`）**：
   - `useSafariWindowManager.ts`：托管窗口列表、持久化、`syncSafariWindows`、`mergeWindowMetadata`、AX 扫描与点击/输入动作分发。
   - `useNetworkRecorder.ts`：Recorder 注入（`ensureWindowInjected`）、Window Probe 定时器、后台轮询（`handlePollNetRecorder`）、429 Quota Alert 去重与状态管理。
   - `useEmqxBridge.ts`：EMQX 连接生命周期、心跳与流量统计、审批队列（`pendingApprovals`）、以及将 450 行的 `handleEmqxDecision` 拆分为纯命令路由（`services/emqxCommandRouter.ts`）+ 动作执行回调。
   - `usePocketDeckBridge.ts`：LAN Sharing 状态、PocketDeck Snapshot 组装与 `handlePocketDeckAction` 桥接（同时为未来独立 Mobile Viewport AX 页面提供统一只读 Snapshot + Action Dispatch 契约）。
2. **提取纯业务服务与流程管道（`apps/p2p-remote-controller/src/services/`）**：
   - 将 `handleInitialize`（280+ 行）下沉到 `services/aiStudioInit.ts` 或独立的 `services/aiStudioInitRunner.ts`，通过窄进度回调更新 UI。
   - 修复 `types/emqxBridge.ts -> utils/logExport.ts` 的 L0->L1 逆向依赖（将 `LogExportLevel` 移至 `types/`）。
3. **拆分 UI 视图组件（`apps/p2p-remote-controller/src/components/`）**：
   - `PermissionAutomationBar.tsx`（顶部权限/Automation/Safari 状态条）
   - `ManagedWindowSidebar.tsx`（左侧窗口列表、别名编辑、缺失窗口重开）
   - `AxElementInspector.tsx`（AX 元素过滤、正则搜索、快捷动作行）
   - `DebugConsoleTabs.tsx`（JSON / Timeline / Console / Network / Presets 右侧面板容器）

### Wave 2：修复 AGY 主端单向依赖违规与死代码（待 AGY 当前维修合并后）
1. 解除 `src/utils/rawPromptCopy.ts` 对 `tauriBackendClient.ts` 的 L1->L2 依赖（改为参数注入或移入 `services/`）。
2. 解除 `src/utils/taskExportHydrator.ts` 对 3 个 `stores/` 的 L1->L3 依赖（改为接收只读数据快照参数，成为纯函数管道）。
3. 核查并清理孤立模块 `src/utils/draftRegexEngine.ts`。
4. 清理基线 4 项违规（`githubAuthBlock.ts` 与 `quotaAlertParser.test.ts` 长行换行、`App.tsx` 多语句行与本地化错误匹配策略）。

### Wave 3：AGY 主端巨石文件按领域拆解
1. **Store 与 Service 层**：
   - `src/stores/telemetryStore.ts`（1,103 行）-> 拆出纯归档/索引/分片计算管道至 `utils/`，Store 仅保留状态与动作。
   - `src/services/tauriBackendClient.ts`（1,121 行）-> 按后端命令域（`credentialsClient.ts`、`taskIpcClient.ts`、`lanIpcClient.ts`、`telemetryIpcClient.ts`）拆分，无跨模块共享状态。
2. **纯计算与 UI 层**：
   - `src/utils/draftRegexDuplicate.ts`（1,138 行）-> 拆分为规则解析、正则匹配预览、批量生成三个独立纯管道。
   - `src/components/drafts/BatchDuplicateConfigPanel.tsx`（835 行）、`src/components/layout/AppTopHeader.tsx`（784 行）、`src/components/layout/TaskHistorySidebar.tsx`（613 行 / 26 Hooks）-> 按子视图槽位拆分为独立展示组件与 `useTaskSidebar*` Hook。

### Wave 4：收紧雷达棘轮
- 每完成一个 Wave，同步下调 `scripts/radar/radarArchDebtGuards.ts` 与 `radarBoundaryAndCycles.ts` 中的棘轮上限：
  - `HARD_LAYER_VIOLATION_LIMIT`：`5 -> 0`
  - `HARD_VOLUME_CEILING_FILES_LIMIT`：`21 -> 10 -> 0`
  - `HARD_REACT_HOOK_COUNT_LIMIT`：`90 -> 30 -> 20`
