# AGY Preview Instant — 诊断、分支合并与状态机修复报告

## 1. Live Chat 实测诊断结论（频道 `ch_83079527`）

通过对 5 个典型 Safari 窗口的实时 `scan_all_windows` 与 `scan_window` 采集，确认了全部异常的根因：

| 窗口 ID | 实际状态 | 原判定 | AX 树实测根因 |
| :--- | :--- | :--- | :--- |
| `#63133` | `uninitialized` | `partial` | Google AI Studio 新对话默认已选中 `API key selected`，且旧逻辑因 `modelSelected=false` & `network=false` 判为 `partial`；实际上尚未执行初始化，应为 `uninitialized`。 |
| `#63135` | `ready` | `partial` | 新版 UI 的 `AXButton`（`Select Antigravity agent harness model`）**不暴露**已选模型名称，且 Network 替换默认规则后仍显示 `1 rule`，导致 `modelSelected` 和旧规则数检测全部失效。 |
| `#63137` | `done` | `partial` | URL 为 `/prompts/<id>`（非 `new_chat`），无 `Build with Agents` 标题，无 `Stop` 按钮；原状态机缺少 `done` 状态。 |
| `#62597` | `busy` (`Running`) | `busy` | 有 `Stop` 按钮（无 `Run` 按钮）。但 `scanAllSafariWindows` 之前传入了 `markedBusy: prev?.hasActiveTask ?? false`，导致任务结束后仍永久卡在 `busy`。 |
| `#63170` | 含左侧侧边栏 | `partial` | AX 树中存在 `AXButton` `title="Collapse navigation menu"`（`path=[2]`）。已通过 `perform_action` (`AXPress`) 实测成功收起。 |

---

## 2. 已完成的修改（已合并推送至 `origin/a2`，Verified Commit `dd6718a`）

1. **分支合并**：
   - 将 `origin/feat/llm-ax-regex-and-parallel-approvals` (`e6be509`) 与 `origin/a2` (`96e824b`) 零冲突合并，保留双父节点提交历史（`dd6718a`，GitHub Bot GPG `verified: true`）。
2. **窗口状态机重构（`aiStudioWindowState.ts`, `aiStudioBatch.ts`, `aiStudioBatchRoundRobin.ts`, `types/ax.ts`）**：
   - 新增 `done` 状态：非 `busy` 且为 `/prompts/<id>`（非 `new_chat`）时判定为 `done`，`init_all` 轮询队列自动跳过 `done` 窗口。
   - **修复 `running` (`busy`) 卡死**：移除 `scanAllSafariWindows` 中错误的 `markedBusy: prev?.hasActiveTask ?? false`，仅以实时 AX 树 `Stop` 按钮为准。
   - **改用 `networkConditionAdjusted` 标志位判 `ready`**：不再依赖 `AXButton.title` 中的模型名或 Network `1 rule`/`2 rules` 数量；初始化完成后置 `true`，进入 `running` / 非 `new_chat` URL 时清空；未初始化的新窗口统一归为 `uninitialized`。
3. **初始化流程加固（`aiStudioInit.ts`, `aiStudioInitSteps.ts`）**：
   - 新增 `collapseNavigationSidebar`：初始化时如检测到 `Collapse navigation menu` 按钮则自动点击收起左侧侧边栏。
   - **修复 `Link API Key modal did not appear` -> `Element path out of bounds` 级联报错**：将等待弹窗超时从 `8s` 延长至 `18s`（适配 lazy loading），并在失败时直接重扫校验或抛错终止，不再把过期的旧 `scan` 传给 `selectHarnessModel`。
   - **加固 Network Allowlist 弹窗**：等待 `Add to allowlist` 与 `Save`/`Cancel` 按钮同时渲染完成（`12s`）再操作，避免 `saveBtn=missing` 时误点击，并增加一次 `Add to allowlist` 自动重试。
4. **UI 与可观测性改进**：
   - **删除旧版纯 `Initialize` 按钮**：仅保留 `Init AI Studio` 与 `Init All`（`SafariControlStrip.tsx`, `PermissionAutomationBar.tsx`, `App.tsx`, `useSafariBatchOperations.ts`）。
   - **修复 `[Recent]` 标签被截断问题**：在 `ManagedWindowSidebar.tsx` 中将 `[Recent]` 与 `[Auto]` 紧跟在 `#{windowId}` 之后显示，不再放在长标题末尾。
   - **新增 `Observation` 窗口判定日志 & `Developer Tools` 收拢**：在 `automationTracker.ts` 中记录每次窗口状态判定的完整信号（`apiKey`, `netFlag`, `running`, `promptUrl`, `buildWithAgents`, `sidebarOpen`, `elementCount`, `url`），并在 `DebugConsoleTabs.tsx` 中收拢至 `Developer Tools` 栏下（`Observation`, `Automation`, `Timeline`, `Raw JSON`, `Console`）。

---

## 3. 关于 Worker 与控制命令架构的说明

- **控制命令不需要改 Worker**：`agentlivechatchannel` Worker 仅负责透传 MQTT JSON 载荷（`agy_p2p_v1/cmd/<channel>/<command>` -> `agy_p2p_v1/resp/<channel>/<requestId>`）及提供 `/help` 列表；所有命令解析、裁剪参数（如 `level`, `regexFilter`, `maxChars`）和响应格式均在客户端 `p2p-remote-controller` 的 `emqxCommandRouter.ts` / `emqxCommandHandlers.ts` 中自行定义与处理。后续如需增加简短输出或新控制指令，直接改客户端即可，无需改动 Worker 架构。

---

## 4. 验证结果

- `pnpm --filter p2p-remote-controller test -- --run`：**11 个测试文件、41 项测试全部通过**。
- `pnpm --filter p2p-remote-controller build` (`tsc && vite build`)：**通过**。
- `pnpm --filter p2p-remote-controller exec eslint src`：**0 errors, 0 warnings**。
- `pnpm run lint`（根目录）：**0 errors**。
- `pnpm run env:check`：OverlayFS 占用 `8 MB`（预算 `<= 100 MB`）。
