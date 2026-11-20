# 本轮修复总结与 Real-Time 测试前评估报告

## 一、Live Chat Worker (`/help`) 评估

已请求 `GET https://agentlivechatchannel.maboco-mark.workers.dev/help`，成功返回协议文档与隔离频道 `ch_a5f16e78`：
- **整体够格**：文档结构完整，明确给出了 `/cmd`、`/chat/send`、`/chat/poll`、`/mqtt/rpc` 的请求体格式、`waitMs`（默认/上限 `300000ms`）、`stream: true`（NDJSON 心跳防断流）、`mirrorToChat`、`returnOnFirstMessage` 以及 `supportedCommands` 列表和 `quickStart` 示例。
- **后续可评估微调的点**：
  1. `supportedCommands` 目前包含 `link_all_api_keys`、`initialize_all_windows`、`scan_window`、`reload_window`、`perform_action` 等 23 个指令；若后续移动端需要 `load_all_api_keys`、`refresh_ax_tree`、`reload_with_preset` 等新指令，需同步扩充 Worker 白名单或支持透传。
  2. 多请求并发测试 LLM/Agent 状态（如 `thinking`/`loading`）时，建议后续在 `/chat/poll` 中关注消息游标或去重 ID，避免长轮询边界漏收或重复。

---

## 二、本轮已完成的代码修复（Verified Bot Commit）

- **Commit**: [`c97475f`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/c97475fbc5ae3d8c4ea77140a5afcd1535cb1008)（分支 `a2`，`signature.isValid: true`）
- **验证结果**：
  - `pnpm radar -- --audit`：**PASS**（0 报错，0 循环依赖，0 静默 catch，修改文件均满足体积/耦合红线）
  - `apps/p2p-remote-controller` 单测 (`pnpm test`)：**7 个测试文件 / 27 个用例全部通过**

### 1. AGY Environment Manager 分组与排序默认值持久化
- **修改文件**：`src/utils/environmentStateMerge.ts`、`src/stores/environmentStore.ts`
- **改动内容**：新增 `readEnvironmentGroupBy` / `writeEnvironmentGroupBy`（`agy_env_group_by_v1`）与 `readEnvironmentOrderPriority` / `writeEnvironmentOrderPriority`（`agy_env_order_priority_v1`），在 `useEnvironmentStore` 初始化及每次切换 `Group By`（如 `Size Tier` / `Date`）和 `Order`（如 `Time -> Size` / `Size -> Time` / `Date -> Size`）时自动写入 `localStorage`，关闭弹窗或重启应用后保持上次选择。

### 2. `Initialize` 自动点击 `No thanks` 优化
- **修改文件**：`apps/p2p-remote-controller/src/services/aiStudioInitSteps.ts`、`apps/p2p-remote-controller/src/services/aiStudioInitExistingWindow.ts`
- **改动原因与内容**：
  - 原先 `dismissCookieBanner` 仅在初次扫描时检查一次且未先聚焦按钮/窗口，若 Cookie 弹窗异步延迟渲染或窗口未激活则会被跳过。
  - 现已支持 `AXButton` / `AXLink` / `AXMenuItem`，点击前先执行 `focusWindowBestEffort` 与元素 `focus`，并在 `runExistingWindowInitialize` 中传入 `waitForBannerMs = 1200` 等待异步弹窗出现后再点击关闭。

### 3. `Link All Keys` 改为 `Load All Keys`
- **修改文件**：`apps/p2p-remote-controller/src/components/SafariControlStrip.tsx`、`apps/p2p-remote-controller/src/services/aiStudioBatch.ts`
- **改动内容**：
  - UI 按钮重命名为 `Load All Keys`（底层 `link_all_api_keys` 事件与命令名保持兼容）。
  - `linkAllSafariApiKeys` 在检查每个窗口前会先尝试关闭 `No thanks` 弹窗；若 API Key 未处于已选择状态，会同时匹配 `Link API Key` 按钮以及 `No API key selected`（`AXPopUpButton` / `AXButton`），聚焦并触发 `AXPress` 以展开/加载密钥列表。

### 4. `Init All` 桌面范围开关 + 非阻塞轮询（Round-Robin）算法
- **修改文件**：
  - `apps/p2p-remote-controller/src/hooks/useSafariBatchOperations.ts`
  - `apps/p2p-remote-controller/src/services/aiStudioBatchRoundRobin.ts`（新增）
  - `apps/p2p-remote-controller/src/services/aiStudioBatch.ts`
  - `apps/p2p-remote-controller/src/components/SafariControlStrip.tsx`（从 `PermissionAutomationBar.tsx` 拆分）
  - `apps/p2p-remote-controller/src/hooks/useLogExportActions.ts`（从 `App.tsx` 拆分）
  - `apps/p2p-remote-controller/src/App.tsx`
- **改动内容**：
  - **桌面范围开关**：工具栏新增 `Current Desktop`（默认） / `All Windows` 切换开关。选择 `Current Desktop` 时，按 `w.visible !== false && !w.minimized` 过滤窗口列表，并同步作用于 `Scan All`、`Load All Keys` 与 `Init All`。
  - **Round-Robin 非阻塞初始化**：`initializeAllSafariWindows` 改为多轮队列调度（最多 5 轮）。跳过 `ready` 与 `busy` 窗口后，逐个窗口检查：若页面仍在加载或等待 API Key 列表，刷新一次 AXTree 复检后仍未就绪则立即放回队尾并切换下一个窗口，避免单个慢窗口阻塞其余 9 个窗口的初始化。

---

## 三、重点说明：`Scan All` 为什么经常显示 `partial`？

在 `apps/p2p-remote-controller/src/services/aiStudioWindowState.ts` 的 `classifyAiStudioWindowState` 中，窗口就绪状态分为 `loading`、`uninitialized`、`partial`、`ready`、`busy` 五档：
1. **已判定通过的条件**：
   - `apiKeyLinked`：检测到 `AXPopUpButton` 标题包含 `API key selected`（且不含 `No API key selected`），同时不存在 `Link API Key` 按钮。
   - `modelSelected`：检测到 `Select Antigravity agent harness model` 按钮文本匹配 `/gemini-4-hydrogen/i`。
2. **导致频繁显示 `partial` 的核心原因**：
   - 代码中 `networkConfigured = networkVerified || (previous?.networkConfigured ?? false)`。
   - AI Studio 主页面上的 Network 按钮在**未初始化（默认 Gemini API 规则）**和**已初始化（`*` 规则）**两种状态下，AXTree 标题都只显示 `"1 rule"`，**不点开 Network Configuration 弹窗就无法从 AXTree 区分内部规则是否为 `*`**。
   - 因此，`Scan All` 执行纯被动扫描（`scanSafariWindow`）时不会点开 Network 弹窗；只要当前窗口在本次会话内存 `windowReadiness` 中没有留下 `networkVerified = true` 的记录（例如重启 App、手动初始化、或新接管的窗口），`networkConfigured` 就会回落为 `false`，最终被归类为 `partial`（`API key linked; needs network`）。

---

## 四、后续 Real-Time 检测与 Pocket Deck / E2EE 需 Double Check 清单

1. **现场 Double Check 项（Mac 实机测试）**：
   - **多桌面 `visible` 字段准确性**：切换多个 macOS Space 后，确认 `get_safari_status` 返回的 `visible` 是否准确将非当前桌面窗口标为 `false`。
   - **`No thanks` 实机节点属性**：若某些新开页面仍未自动点掉 `No thanks`，通过 Live Chat 抓一份 AXTree 确认其 `role`/`title` 是否发生变化。
   - **Round-Robin 节奏参数**：10 窗口并发 `+3 Win` 后执行 `Load All Keys` -> `Init All`，观察 `maxPasses = 5` 与 `600ms`/`900ms` 等待间隔是否足够，或是否需要调整轮次上限。
2. **下一阶段 Pocket Deck & E2EE 联调规划**：
   - **缓存与双模式切换**：完善 Service Worker 离线资源缓存策略，优先探测局域网直连，失败时自动切换 WebRTC P2P（通过 E2EE EMQX 通道交换 ICE）或 EMQX Relay。
   - **鉴权与操作审批**：引入 UUID Access Token + TOTP，在 P2P 建连与每次关键 Action Approval 时校验。
   - **移动端扩展指令**：拆分 `Refresh Page` vs `Refresh AX Tree`、`Reload Current` vs `Reload Default/Preset URL`，并针对轮播图等动态 DOM 引入基于元素 `name` 正则的动态路径匹配。
