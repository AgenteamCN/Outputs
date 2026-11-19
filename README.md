# agyPreviewInstant 并行拆分提速方案、Interaction ID 发送故障定位与雷达静默 Catch 升级报告

## 一、核心结论：原计划能行，但必须从“串行 Wave”改为“双端隔离 + 多分支并发”

原计划（Wave 1 → Wave 4）在**架构边界**上完全正确，但按原节奏串行推进有三个明显瓶颈：
1. **Wave 1 过重**：把 `apps/p2p-remote-controller/src/App.tsx` 的 4 个 Hook、4 个 UI 组件和 2 个 Service 打包在同一个大分支里，合入周期长，期间只要继续修 Bug 或加新功能，分支就会频繁冲突。
2. **两头没有真正并行**：AGY 主端（`src/`）与 P2P 端（`apps/p2p-remote-controller/src/`）在雷达里已是**零互导硬隔离**，完全可以同时开工，不需要等 P2P 拆完再动 AGY。
3. **线上 Bug 没被优先止血**：当前 `a2` 最新功能提交 [`3781b45`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/3781b45dbd795111af8ae263ca2193312cb471ef) 的 Interaction ID 同步链路存在“漏接 + 静默吞错”问题，必须先用 1 个极小分支止血，再铺开拆分。

---

## 二、故障定位：为什么 `remote controller interaction ID sent to AGY` 发不出去且不报错？

检查 [`3781b45`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/3781b45dbd795111af8ae263ca2193312cb471ef) 与当前 `a2` 代码后，定位到 **3 个直接原因**：

1. **核心漏接：正常轮询路径根本没有调用 `processPolledNetworkSnapshot`**
   - `processPolledNetworkSnapshot`（负责 API Key 脱敏、429 告警检测、以及 `autoSyncToAgy` 自动调用 `publishInteractionToAgy`）目前**只接在了 `apps/p2p-remote-controller/src/App.tsx:1918`**（EMQX 远程指令 `poll_net_recorder` 分支）中。
   - 而日常使用的手动/后台自动轮询 `handlePollNetRecorder`（`apps/p2p-remote-controller/src/App.tsx:2259-2300`），以及窗口初始化后的轮询（`2585-2600`、`2737-2753`）全部**直接写了 `parsed` 进 state，没有经过 `processPolledNetworkSnapshot`**。因此正常轮询抓到 `v1_` Interaction ID 后根本不会自动发送给 AGY。
2. **发送失败时静默返回或异常未被接住**：
   - 自动同步调用 `publishInteractionToAgy(interaction, windowId, false)`（`App.tsx:1454-1489`）时，若 EMQX 未连接（`!client || emqxStateRef.current !== "connected"`），函数直接 `return false`，没有任何日志或状态提示（只有 `manual === true` 才调用 `setErrorMessage`）。
   - 手动点击 `Send to AGY`（`App.tsx:3773`）调用 `publishInteractionToAgy(item, selectedWindowId, true)` 时，`client.publish(topic, payload, 1, false)`（`apps/p2p-remote-controller/src/services/emqxClient.ts:254-268`）在 WebSocket 未就绪时会 `throw new Error("EMQX is not connected")`，而 `publishInteractionToAgy` 外层没有 `try/catch` 将错误写入 `errorMessage` 或 Bridge 消息流；同时 `errorMessage` 渲染在页面最顶部（`App.tsx:3189`），在 Network 面板底部操作时也极易被忽略。
3. **静默 `catch` 掩盖了上下文错误**：
   - `handlePollNetRecorder`（`App.tsx:2311`）在 `silent === true` 时吞掉了轮询异常；初始化轮询多处使用 `.catch(() => null)`（`App.tsx:2588, 2740, 2763`）与空 `catch {}`（`App.tsx:2597, 2749`）；
   - AGY 接收端 `src/stores/telemetryStore.ts:276` 同样存在空 `catch {}` 吞掉 payload 解析异常。

---

## 三、已完成：工具链补齐 & 雷达升级（禁止静默 `catch`）

1. **工具链补齐**：已在环境安装 `pnpm@12.21.0` 并完成 `pnpm install --frozen-lockfile`。`pnpm p2p:check` 验证通过（0 errors；AGY 主端 `pnpm check` 因 `src/config/agentDefaults.ts` 依赖生成的 `./resolvedProfile.json`，需先执行 profile 同步）。
2. **雷达升级（Verified Bot Commit [`c3f9e15`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/c3f9e155d0765bae3e4568a124c63cad4192ac18) 已合入 `a2`）**：
   - 修改 `scripts/radar.ts` 与 `scripts/radar/radarArchDebtGuards.ts`，新增 **Silent-Catch 棘轮守卫**：
     - 扫描全仓源码（排除 `scripts/` 与 `*.test.*`）的**空 `catch {}` 块**（含仅含注释的 `catch`，当前基线 `66/66`）和**静默 Promise 吞错**（`.catch(() => {})` / `.catch(() => null | undefined | false | ...)`，当前基线 `60/60`）。
     - **任何新增的静默 `catch` 都会立即触发 `[SILENT-CATCH RATCHET]` 或 `[SILENT-PROMISE-CATCH RATCHET]` 并阻断退出（exit 1）**。
     - 运行 `pnpm radar --skip-tsc --audit` 会直接打印 `[E]` 类静默吞错热点文件及精确行号，方便逐个清理。

---

## 四、高效率多分支并行拆分计划（6 条并行分支，文件零冲突）

核心策略：**按文件所有权切分分支**。谁独占哪些文件就只改哪些文件，把对 `apps/p2p-remote-controller/src/App.tsx` 的修改集中到“止血分支”和“最终抽壳分支”，其余分支**100% 并发开发、互不冲突**。

### 第一阶段（Batch A）：4 条分支立即并发（预计 1~2 小时内各自独立合入）

| 并行分支 | 责任范围（独占文件，零冲突） | 核心任务 | 验收标准 |
| :--- | :--- | :--- | :--- |
| **1. `fix/p2p-interaction-sync`**<br>*(最高优先级，先合)* | `apps/p2p-remote-controller/src/App.tsx`（仅修 Bug 与基线）、`quotaAlertParser.test.ts`、`types/emqxBridge.ts` | 1. 将 `processPolledNetworkSnapshot` 接入 `handlePollNetRecorder` 与初始化轮询；<br>2. 给 `publishInteractionToAgy` 补齐 `try/catch` 与可见错误提示；<br>3. 清掉 P2P 侧 3 项雷达基线报错（中文字符、多语句行、超长行）与 1 处 L0→L1 逆向依赖。 | 手动/自动轮询可触发 Interaction ID 同步与 429 告警；`pnpm p2p:check` 通过。 |
| **2. `refactor/p2p-views-and-router`**<br>*(与分支 1 完全并行)* | **仅新建文件**：`apps/p2p-remote-controller/src/components/{PermissionAutomationBar,ManagedWindowSidebar,AxElementInspector,DebugConsoleTabs}.tsx`、`services/emqxCommandRouter.ts` | 先把纯 Props 展示组件与 `handleEmqxDecision` 的纯命令路由抽成独立模块，**此分支不改 `App.tsx`**，彻底避免与分支 1 冲突。 | 新模块通过 `pnpm p2p:check`，每文件 `<= 350` 行、无静默 `catch`。 |
| **3. `fix/agy-layers-and-baseline`**<br>*(AGY 主端，零交集)* | `src/utils/rawPromptCopy.ts`、`src/utils/taskExportHydrator.ts`、`src/utils/githubAuthBlock.ts`、`src/utils/draftRegexEngine.ts` | 1. 解除 4 处 `utils -> services/stores` 逆向依赖（改为纯数据快照参数注入）；<br>2. 修复 `githubAuthBlock.ts:9` 超长行；<br>3. 核查并清理/接入孤立模块 `draftRegexEngine.ts`。 | AGY 侧 `LayerViolations` 清零，`Orphans` 清零。 |
| **4. `refactor/agy-clients-and-utils`**<br>*(AGY 主端，零交集)* | `src/services/tauriBackendClient.ts`（1,121L）→ 4 个域客户端；`src/utils/draftRegexDuplicate.ts`（1,138L）→ 3 个纯管道 | 按命令域拆分 IPC 客户端（`credentials` / `task` / `lan` / `telemetry`），按步骤拆分正则批量复制纯计算管道。 | 单文件全部降至 `<= 400` 行，无新静默 `catch`。 |

### 第二阶段（Batch B）：分支 1、2 合入后，3 条分支继续并发收口

| 并行分支 | 责任范围（独占文件，零冲突） | 核心任务 | 验收标准 |
| :--- | :--- | :--- | :--- |
| **5. `refactor/p2p-hooks-shell`** | `apps/p2p-remote-controller/src/App.tsx` + 新建 `hooks/{useSafariWindowManager,useNetworkRecorder,useEmqxBridge,usePocketDeckBridge}.ts` + 接入分支 2 的视图组件 | 将 `App.tsx`（3,870L / 90 Hooks）压至 `<= 350` 行纯编排壳层，同时顺手清理 `App.tsx` 内的 34 处静默 `catch`。 | `App.tsx <= 350` 行、`Hooks <= 20`，`pnpm p2p:check` 与 `pnpm radar` 通过。 |
| **6. `refactor/agy-stores-slim`** | `src/stores/telemetryStore.ts`（1,103L）、`src/stores/presetStore.ts`（622L）、`src/stores/themeStore.ts`（625L） | 将纯归档/索引/分片计算下沉到 `src/utils/`；顺手清理 `themeStore.ts` 的 18 个空 `catch`。 | Store 文件均 `<= 400` 行，Silent-Catch 显著下降。 |
| **7. `refactor/agy-ui-hotspots`** | `src/components/drafts/BatchDuplicateConfigPanel.tsx`（835L）、`src/components/layout/AppTopHeader.tsx`（784L）、`src/components/layout/TaskHistorySidebar.tsx`（613L / 26 Hooks） | 按子视图槽位拆分为独立展示组件与 `useTaskSidebar*` Hook。 | UI 巨石文件全部退出硬超标清单。 |

### 第三阶段（棘轮收紧）
每合入一条重构分支，同步下调 `scripts/radar/radarArchDebtGuards.ts` 与 `radarBoundaryAndCycles.ts` 中的棘轮上限：
- `HARD_LAYER_VIOLATION_LIMIT`: `5 → 0`
- `HARD_VOLUME_CEILING_FILES_LIMIT`: `21 → 10 → 0`
- `HARD_EMPTY_CATCH_LIMIT` / `HARD_SILENT_PROMISE_CATCH_LIMIT`: `66+60 → 30+20 → 0`
- `HARD_REACT_HOOK_COUNT_LIMIT`: `90 → 30 → 20`
