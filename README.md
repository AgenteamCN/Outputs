# AGY Preview Instant — Batch C 合并后全层核查与下一步路线评估报告

> 核查基准：`origin/a2` @ `abc9018`（Batch C Octopus Merge）  
> 验证环境：已安装 `pnpm 12.21.0`，通过 `node scripts/syncPrivateProfile.mjs` 生成 `resolvedProfile.json` 后全量跑通检查与构建。

---

## 一、棘轮是否锁紧？—— 尚未锁紧（需立即修补）

经过对 `scripts/radar/radarArchDebtGuards.ts`、`scripts/radar/radarBoundaryAndCycles.ts` 与 `scripts/radar.ts` 的源码核查：

1. **已锁紧的指标**：
   - `HARD_LAYER_VIOLATION_LIMIT = 0`（层级越界）
   - `Cycles = 0`（循环依赖）
   - `0 Re-Exports`（值重导出）、`Any <= 5`、`BoolHotspots = 0`、`UnscopedCaches = 0`、`Orphans = 0`
2. **尚未锁紧的指标（关键遗留）**：
   在 `scripts/radar/radarArchDebtGuards.ts` 中，虽然 Batch C 已将实际债务全面清零，但**棘轮上限常量仍停留在旧值**：
   - `HARD_VOLUME_CEILING_FILES_LIMIT = 12`（当前实际 `0`，未锁紧至 `0`）
   - `HARD_EMPTY_CATCH_LIMIT = 31`（当前实际 `0`，未锁紧至 `0`）
   - `HARD_SILENT_PROMISE_CATCH_LIMIT = 23`（当前实际 `0`，未锁紧至 `0`）
3. **额外门禁缺口**：
   - `scripts/radar.ts` 中仅在传入 `--strict` 时才会把 `hardNotices`（>800L / >18KB）直接作为 error 阻断；默认 `pnpm radar` 依赖 `HARD_VOLUME_CEILING_FILES_LIMIT`。因此只要该常量还是 `12`，后续改动仍然可以新增最多 12 个超标大文件而不报错。
   - `pnpm radar` 默认只串联了 `pnpm --filter p2p-remote-controller check`，未串联主仓库 `pnpm check`。

**结论**：**必须立即将上述 3 个棘轮上限全部改回 `0`**，否则无法真正阻止后续代码出现回归。

---

## 二、为什么 `HostPen = 52`、`ReactHooks = 27`？是否正常？

通过 `pnpm radar -- --audit` 定位到两个最高值文件并逐行核查：

1. **`HostPen = 52 <= 55`：`src/stores/task/taskMutationSlice.ts`（`Host=52(52R/0W)`，393L）**
   - **原因**：`scripts/radar/radarStructuralCoupling.ts` 使用正则 `HOST_ACCESS_RE` 匹配 `(this.)?(host|owner|parent|ctrl|runtime|engine|store|state).<prop>`。该文件是 Zustand store slice，包含 9 个 mutation action，大量使用 `set((state) => ({ tasks: state.tasks.map(...), activeView: state.activeView, ... }))`。
   - **是否正常**：**属于统计口径导致的正常现象（假阳性偏高）**。这 52 次全部是对 Zustand 自身 `state` 形参的只读属性访问（`52R / 0W`），不存在外部对象穿透或非法状态写入。
2. **`ReactHooks = 27 <= 28`：`apps/p2p-remote-controller/src/hooks/useEmqxBridge.ts`（`Hooks=27(S:8/E:6)`，400L）**  
   以及紧随其后的 **`useSafariWindowManager.ts`（`Hooks=26(S:8/E:3)`，523L / 18.0KB）**
   - **原因**：两者均为 P2P 端的核心编排型 Custom Hook，分别集中了 8 个 `useState`、3~6 个 `useEffect` 以及大量 `useRef`/`useCallback`。
   - **是否正常**：**在 Custom Hook 语义上合理，但已逼近硬性红线，属于真实隐患**。`useEmqxBridge.ts` 距离 `ReactHooks <= 28` 仅剩 **1 个 hook** 的余量；`useSafariWindowManager.ts` 的体积 **18.0KB 已贴死 18.0KB 硬上限**。后续只要在这两个文件里新增一个状态或回调，就会直接触发雷达报错。

---

## 三、全量验证与软性约束（Anti-Gaming）人工审查

### 1. 全量命令行实测结果（全部 PASS）
- `pnpm radar`：PASS（275 files，Cycles=0，LayerViolations=0/0，Hard=0/12，SilentCatches=0/31+0/23）
- `node scripts/syncPrivateProfile.mjs && pnpm check`：PASS（`tsc --noEmit` 通过，`eslint src` 0 errors / 18 warnings，`vitest run` 2 files / 16 tests 通过）
- `pnpm p2p:check`：PASS
- `pnpm --filter p2p-remote-controller test`：PASS（5 files / 19 tests 通过）
- `pnpm build` & `pnpm p2p:build`：PASS

### 2. 软性约束与 Anti-Gaming 审查
- **规则绕过检查**：全仓 `src/` 与 `apps/p2p-remote-controller/src/` 下 **`eslint-disable` = 0、`@ts-ignore` = 0、`@ts-expect-error` = 0、`as any` = 0**，无通过注释关闭检查的痕迹。
- **虚假碎片化检查**：20 行以下的非入口文件仅 1 个（`src/components/settings/credentials/credentialFieldStyles.ts`，16L 纯样式常量），未出现“为躲避行数限制而拆出大量 10 行空壳包装文件”的现象。
- **横向切分与回调乒乓检查**：
  - `src/stores/taskStore.ts` 通过 `registerPresetDefaultDraftSync`（来自 `workspaceDraft.ts`）注册回调，属于为了避免 `taskStore <-> workspaceDraft` 静态循环依赖而设计的标准回调注册，未发现同层无意义的互调死循环。
- **仍需注意的软性债务**：
  1. **贴线大文件**：`useSafariWindowManager.ts`（18.0KB / 523L）、`taskSimplifiedExport.ts`（17.9KB / 618L）、`useNetworkRecorder.ts`（17.9KB / 485L）、`telemetryStateResolver.ts`（17.6KB / 572L）均已接近 18KB 硬上限。
  2. **18 条 ESLint Warning**：主要为 13 个文件触发 `max-lines > 450`、4 处 `react-hooks/exhaustive-deps`（`TaskInspectorView.tsx`、`TaskOutputViewer.tsx`、`TaskTelemetryPanel.tsx`、`StorageManagementSection.tsx`）以及 `stepTextCondenser.ts` 中 1 处 `no-useless-assignment`。

---

## 四、下一步开发路线与架构建议

### 1. P0：锁紧棘轮 + 顺手拆减 P2P 贴线 Hook
- 将 `radarArchDebtGuards.ts` 中的 `HARD_VOLUME_CEILING_FILES_LIMIT`、`HARD_EMPTY_CATCH_LIMIT`、`HARD_SILENT_PROMISE_CATCH_LIMIT` 全部锁紧为 `0`。
- 顺手将 `useSafariWindowManager.ts`（18.0KB / 26 hooks）和 `useEmqxBridge.ts`（400L / 27 hooks）中的纯轮询/状态逻辑下沉到独立子模块或 `services/`，为后续功能留出安全余量。

### 2. P1：打通 AGY <-> P2P 全链路任务下发与双向通信
通过核查 `useEmqxBridge.ts`、`emqxCommandHandlers.ts`、`aiStudioDispatchSteps.ts` 与 AGY `telemetryStore.ts`：
- **现状**：
  - `P2P -> AGY` 单向已通：P2P 发布 `${prefix}/agy/import_task`，AGY 订阅 `+/agy/import_task` 并自动导入 `Task History`。
  - `EMQX -> P2P` 命令已备齐：`emqxCommandRouter.ts` 已支持 `dispatch_aistudio_task`、`initialize_aistudio`、`scan_window`、`perform_action`、`net_record_poll` 等。
  - **缺失环节**：**AGY 侧尚未实现 P2P Command Client**（AGY 的 `MqttWebSocketSubscriber` 目前只订阅遥测与 `import_task`，未向 `cmd/` 发指令、也未订阅 `resp/`）。
- **任务下发已知 Edge Cases（需在验证时重点修复）**：
  1. `handleDispatchAiStudioTaskCommand` 在 `enterPromptAndRun` 点击 `Run` 后只 `sleep(1000)` 并做**单次** `buildNetworkRecorderPollScript`，既未轮询等待 `CreateInteraction` 的 `interactionId` 真正出现，也未调用 `ctx.processPolledNetworkSnapshot(...)`，导致下发回执经常拿不到 `interactionId`。
  2. `enterPromptAndRun` 使用 AX `set_value` 写入 `AXTextArea`，部分前端受控组件若未收到合成键盘/输入事件会导致 `Run` 按钮保持 disabled；此外还需处理“上一轮仍在运行（页面显示 `Stop` 而非 `Run`）”、“条款/登录弹窗拦截”等边界情况。
- **落地路径**：修补 P2P 下发后轮询捕获 `interactionId` 的闭环 -> 在 AGY 侧增加轻量 `p2pCommandClient`（复用现有 EMQX 连接） -> AGY 直接下发初始化/任务并把返回的 `interactionId` 写入 `Task History`，同时支持通过 EMQX 与 Agent 实时对话（`chat` topic 已有基础结构）。

### 3. P2：自动刷新与有用按钮提取
- `emqxCommandHandlers.ts` 已经内置 `targetRegex` 与 `regexFilter` 能力。后续只需把高频控件（`Run`、`Stop`、模型选择、Network Rule、Accept/Retry）整理成“AX 精确属性优先 + Regex 兜底”的匹配规则表，配合窗口状态自动刷新即可，无需重写底层协议。

### 4. P3：Quality of Life（后台无感 Focus 恢复 & 自身无障碍）
- **Focus 瞬间恢复（强烈建议优先做，成本极低）**：
  - 目前 `initializeAiStudioWindow` 和 `dispatchAiStudioTask` 都会调用 `focusWindowBestEffort(windowId)` 抢占前台焦点。
  - 在 `packages/macos-ax-helper`（Swift）中，可在执行 `focus_window` 前记录 `NSWorkspace.shared.frontmostApplication?.processIdentifier`，并在操作完成后（或暴露 `restore_focus` 命令在 `try ... finally` 中）调用 `NSRunningApplication(processIdentifier: prevPid)?.activate(options: [])` 将焦点切回 AGY 或原前台窗口，从而实现“准后台”静默完成初始化与下发。
- **自身 UI 无障碍标签**：为 AGY 与 P2P 的核心操作按钮、输入框补齐语义化 `aria-label`（与 `data-testid`），后续无论是键盘无障碍还是用 AX 工具反向驱动/自动化测试都会非常顺畅。

### 5. P4：双模式（API Mode vs AX/Browser Mode）可行性评估
- 鉴于当前官方 API 拥堵严重（10 分钟任务耗时 2 小时），双模式具备很高的实用价值，**且不需要做 UI 大改版**。
- **建议方案**：在架构上抽象统一的 `TaskDispatchTransport`（`api` | `ax_browser`），UI 上仅在 Composer 发送区旁或设置中增加一个轻量的模式切换开关（类似 Light/Dark 或模型选择 Pill），底层路由到现有的 `taskExecutionSlice`（API）或 `p2pCommandClient`（EMQX + Safari AX），下游 `Task History` 与 `Telemetry` 展示完全复用现有逻辑。
