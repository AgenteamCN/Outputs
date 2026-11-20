# 任务完成报告：Live Chat `/help` 渠道隔离、Safari/AI Studio QoL 升级与 Hooks 架构拆分

## 一、Live Chat System (`agentlivechatchannel.maboco-mark.workers.dev`) 升级与验证

已对 Cloudflare Worker `agentlivechatchannel` 完成在线测试与更新部署：

1. **新增 `GET /help` 与 `POST /help` 端点**（只需携带 `x-agy-request-id` 请求头即可调用）：
   - 自动随机生成隔离的 `channel` ID（如 `ch_b3912eb5`，也可通过 `?channel=` 或请求体指定）。
   - 返回该通道对应的 MQTT Topic 模板（`agy_p2p_v1/cmd/<channel>/<command>`、`agy_p2p_v1/resp/<channel>/<requestId>`、`agy_p2p_v1/chat/<channel>`）。
   - 返回全部可用端点说明（`/help`、`/version`、`/cmd`、`/chat/send`、`/chat/poll`、`/mqtt/rpc`）、完整的 `supportedCommands` 列表（含新增的批量 Safari/AI Studio 命令）以及预填好 `channel` 的 `quickStart` curl 示例。
   - `/version` 端点同步更新，在 `endpoints` 中列出 `"/help"`。
2. **线上实测结果**：
   - `GET /help` 成功返回 `{"ok": true, "channel": "ch_b3912eb5", ...}`。
   - 使用分配到的 `ch_b3912eb5` 调用 `POST /chat/send` 成功发布消息（`{"ok": true}`）。

---

## 二、Quality of Life 更新（`apps/p2p-remote-controller`）

### 1. 自动恢复焦点（`focusRestore.ts`）
- 新增 `apps/p2p-remote-controller/src/utils/focusRestore.ts`，封装 `restoreCallerFocus()` 与 `withCallerFocusRestored()`（基于 `@tauri-apps/api/window` 的 `getCurrentWindow().setFocus()`）。
- 在以下所有会触发 Safari 窗口聚焦的链路中默认启用焦点自动回切：
  - UI 操作：`handleFocusCurrentWindow`、`handleInitAiStudio`、`handleLinkAllApiKeys`、`handleInitializeAllWindows`、`handleOpenMultipleWindows`。
  - 远程 EMQX 指令：`focus_window`、`initialize_aistudio`、`dispatch_aistudio_task`、`open_safari_windows`、`perform_action`（当 `focusFirst=true` 时）。
- 用户点击按钮或云端下发指令后，Safari 完成动作会立刻把焦点切回 Remote Controller，减少手动切换窗口的打断感。

### 2. AI Studio 初始化 Emergency Stop（紧急中止）
- 为 `waitForScanCondition`、`dismissCookieBanner`、`ensureApiKeyLinked`、`selectHarnessModel`、`configureNetworkAllowlist`、`initializeAiStudioWindow`、`dispatchAiStudioTask` 及批量操作全部接入 `AbortSignal`。
- 当 `Link API Key` 或模型/网络配置卡住时，UI 工具栏会显示红色 **Emergency Stop** 按钮（`handleEmergencyStopInit`），同时支持远程 EMQX 命令 `emergency_stop_initialize`，可立即抛出 `AbortError` 中止当前轮询与等待。

### 3. 多窗口扫描、状态分级颜色标识与一键批量操作
- **纯状态分类器**（`src/services/aiStudioWindowState.ts`）：通过扫描结果自动判定窗口状态：
  - `ready`（绿色）：页面已加载 + API Key 已连接 + 目标模型已选中 + Network 已配置。
  - `busy`（紫色 `[Running]`）：检测到 `Stop` / `Cancel` / `Interrupt` 等运行中按钮或已下发任务。
  - `loading`（黄色 `[Loading]`）：`webAreaLoaded === false` 或元素过少（`<= 10`）。
  - `partial`（橙色 `[Partial]`）：API Key 已连接，但模型或 Network 尚未配置。
  - `uninitialized`（红色 `[Uninit]`）：未连接 API Key（存在 `Link API Key` 按钮）或需登录 Google。
- **批量操作服务**（`src/services/aiStudioBatch.ts` & `src/services/aiStudioBatchDispatch.ts`）：
  - **Scan All**（`scan_all_windows`）：扫描所有可见 Safari 窗口并更新颜色状态。
  - **Link All Keys**（`link_all_api_keys`）：遍历窗口，跳过已连接 / `busy` / `loading` 窗口，自动为未连接窗口聚焦并点击 `Link API Key`，结束后切回焦点。
  - **Init All**（`initialize_all_windows`）：扫描后跳过 `ready` 与 `busy` 窗口，只对未就绪窗口执行 `initializeAiStudioWindow` 并标记为 `ready`。
  - **+3 Win**（`open_safari_windows`）：显式批量新建 Safari 窗口。
  - **批量下发容量检查**（`check_dispatch_capacity` & `batch_dispatch_tasks`）：下发 N 个任务前先校验可用 `ready`（且非 `busy`）窗口数量；若不足，明确返回缺口数量与当前状态分布（而不擅自弹窗）。
- **Dispatch 快速路径优化**（`aiStudioInit.ts` & `aiStudioAxHelpers.ts`）：
  - 修复了此前每次 `dispatch` 都无条件重新点击模型选择器并打开 Network 弹窗的问题：
    1. `selectHarnessModel` 先检查 `isHarnessModelSelected(scan, modelRegex)`，若已匹配直接跳过点击。
    2. 当窗口已处于 `ready`（`assumeNetworkReady: true`）且 API Key 与模型均已校验通过时，直接跳过 Network 配置弹窗，直接输入 Prompt 并点击 Run。

---

## 三、Hooks 架构重构与“Hooks 数量堆叠”科普

### 1. 本次重构落地（Clean-Before-Extend）
- **工具链**：已全局安装 `pnpm`（`12.21.0`）并完成依赖安装。
- **`useEmqxBridge.ts` 拆分**（原 400 行 / 12.5KB / 27 个 Hooks）：
  - 抽取纯 TS 消息解析器：`src/utils/emqxMessageParser.ts`（0 Hooks）。
  - 拆分连接管理子 Hook：`src/hooks/useEmqxConnection.ts`（连接状态、客户端实例、流量统计、发布到 AGY）。
  - 拆分审批与聊天子 Hook：`src/hooks/useEmqxApprovalsAndChat.ts`（待审批队列、聊天持久化、决策执行）。
  - `src/hooks/useEmqxBridge.ts` 退化为薄编排层，仅负责组合两个子 Hook 并保持对外接口不变（已从 Radar Top-20 耦合热点列表中彻底移除）。
- **`useSafariWindowManager.ts` 拆分**：
  - 抽取 `src/hooks/useSafariBatchOperations.ts`，承接 `handleInitialize`、`handleInitAiStudio`、Emergency Stop 以及全部批量扫描/Link Key/Init All/多窗口打开逻辑。

### 2. 科普：“Hooks 数量堆叠在同一个文件/Hook 里”是什么意思？
很多人容易把“Hooks 数量太多”误解成“文件行数太长”，其实在 React 中它有更具体的运行时含义：

1. **React 的底层机制（Hook 链表）**：
   - 无论是组件还是自定义 Hook（如 `useEmqxBridge`），内部每次调用 `useState`、`useEffect`、`useRef`、`useCallback`、`useMemo`，React 都会在当前组件 Fiber 节点的 `memoizedState` 上按**调用顺序**串联成一个单向链表。
   - **自定义 Hook 并不会创建独立的作用域隔离**——调用 `useEmqxBridge()` 的组件（如 `App.tsx`），会把 `useEmqxBridge` 里面的 27 个 Hook 全部平铺进 `App` 自己的 Hook 链表里。
2. **为什么“堆叠”是真实隐患（如 `Hooks=27(S:8/E:6)`）？**
   - **正交状态互相连坐重渲染**：`useEmqxBridge` 同时管“WS 连接/流量统计（`trafficStats`）”、“审批队列（`pendingApprovals`）”、“聊天记录（`chatMessages`）”。每收到一条聊天消息或字节数跳动，`setTrafficStats` 就会触发调用方重渲染，进而重新执行所有 27 个 Hook 的闭包与依赖比对。
   - **副作用（`useEffect`）纠缠与闭包陈旧**：6 个 `useEffect` 和多个 `ref` 同步代码混在一起，极易出现“为了绕过 ESLint `exhaustive-deps` 而不断加 `useRef` 镜像”的恶性循环（比如 `emqxConfigRef`、`emqxStateRef`、`getCommandDepsRef`）。
   - **难以做纯逻辑单测**：把协议解析、状态机和 React `useState`/`useEffect` 绑死在一起后，想测试“收到一条 `cmd` 消息怎么路由”就必须挂载整个 React 组件树；拆成纯 TS 函数/Store（0 Hooks）+ 小粒度子 Hook 后，纯 TS 逻辑可以直接用 Vitest 毫秒级单测。
3. **判断标准（后续如何自己做架构判断）**：
   - **纯数据转换 / 状态机 / 解析逻辑** -> 放进纯 TS `utils/` 或 `services/` 或 Zustand Store（**0 Hooks**）。
   - **单一职责的 React 状态适配** -> 每个自定义 Hook 控制在 **8~15 个 Hooks 以内**；一旦看到 `radar --audit` 中 `Hooks >= 24`（尤其是 `E >= 5` 个 `useEffect`），就说明它已经把多个正交子域揉在了一起，适合按职责拆成子 Hook + 纯 TS 模块。

---

## 四、检查与提交状态

- `pnpm p2p:check`（`tsc --noEmit` + `eslint src`）：**PASS**
- `pnpm --filter p2p-remote-controller test`（6 个测试文件，24 条用例，含新增 `aiStudioWindowState.test.ts`）：**PASS**
- `pnpm radar -- --audit`：**PASS**（`HardVolume=0/0 | LayerViolations=0 | BoolHotspots=0 | UnscopedCaches=0 | Orphans=0 | SilentCatches=0+0`）
- **Verified Bot Commit**：分支 `a2`，Commit `b730f9b52b3610f8c6bc371c0d479afcaecf28d9`（`signature.isValid: true`，`Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）。
