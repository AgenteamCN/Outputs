# P2P Remote Controller：`Init & Record`、Network 状态抖动与 Safari 注入机制深度排查及优化方案

> 基于仓库 `Antigravi-ty/agyPreviewInstant`（最新提交 `bc10143` / `be2a1d4` / `6a5813c`）源码分析

---

## 一、核心结论速览

1. **`Init & Record` 为什么没能打开新页面并初始化到对应 Model / Network Settings，且控制台不报错？**
   - **它根本不包含任何 Model 选择或 Settings 配置逻辑**。代码中 `Init & Record` 只是把 `DEFAULT_TARGET_URL` 写死为 `https://aistudio.google.com/prompts/new_chat?model=antigravity-preview-09-2026`，然后执行「启动 Safari → 新建窗口 → 轮询等加载 → 注入 Network Recorder」。页面上的 Model 下拉框、Run Settings 等完全没有被操作。
   - **控制台不报错的原因**：轮询等待页面加载期间（Step 3/5），`scanSafariWindow(...).catch(() => null)` 和 `executeSafariJavaScript(...).catch(() => "")` 把所有异常静默吞掉了；进度与错误信息只写进了右侧 **Network Tab 的状态条（`netVerifyDetail`）**，没有打到明显错误弹窗。若 Safari 未开启「允许来自 Apple 事件的 JavaScript」或页面重定向到登录页，它只会默默空转 12 秒（24 × 500ms）后失败。
2. **Network 面板为什么一会儿提示 `Not Installed`（未注入），一会儿又变成 `Unchecked` / `Injection Failed`？**
   - **根本原因**：前端的 `netRecording`、`netSnapshot`、`netVerifyStatus`、`netVerifyDetail` 是**全局唯一一份状态**（没有按 `windowId` 隔离），同时受到 **URL 变化检测**、**`AXLoadComplete` 事件**、**手动切换窗口**、**EMQX 远端指令** 四个来源的交叉覆盖。
   - 特别是 AI Studio 这类 SPA 会频繁通过 `history.pushState` 改 URL 或局部触发 `AXLoadComplete`，此时注入其实还活着，但前端逻辑会直接把状态强行降级为 `Unchecked`，导致界面状态反复横跳。
3. **当前的 Injection 机制针对谁？PID、Window ID 还是 Tab ID？能否自动检测与 Auto-Injection？**
   - **注入粒度**：Safari 在 macOS 上是单主进程（共享同一个 **PID**）；注入指令通过 AppleScript `first window whose id is <windowId>` 的 `current tab` 执行，因此它是**精确针对单个受管 `windowId` 的当前激活 Tab**，**不会**对所有 Safari 窗口注入，也**不追踪 Tab ID**（同窗口切 Tab 就会丢失注入）。
   - **能否自动检测与 Auto-Injection？** **完全可以**。现有 `AXObserver`（监听 `AXLoadComplete` / `AXTitleChanged` / `AXFocusedWindowChanged`）+ 轻量级 JS 探针完全足够支撑「无感自动检测 + 自动重注入」，彻底消除手动点 `Inject & Record Net` 和状态误报。

---

## 二、问题 1 深度解析：`Init & Record` 的实际工作流程与失效原因

### 1.1 当前真实工作流程（5 步）

入口在 `apps/p2p-remote-controller/src/App.tsx` 的 `handleQuickInitAndRecord`（第 1742–1912 行）：

1. **Step 1/5（检查并启动 Safari）**：若 `!safariStatus?.running`，调用 `launchSafari()`（`SafariController.swift` 使用 `NSWorkspace.shared.openApplication` 且 `config.activates = false` 后台启动）。
2. **Step 2/5（打开新窗口）**：读取输入框 `targetUrl`（默认 `https://aistudio.google.com/prompts/new_chat?model=antigravity-preview-09-2026`），调用 `openManagedSafariWindow(resolvedTarget)`。
   - Swift 侧（`SafariController.swift:185-224`）执行 AppleScript：
     ```applescript
     tell application "Safari"
         make new document with properties {URL:"..."}
         delay 0.2
         set w to front window
         ...
     end tell
     ```
   - 拿到 `windowId` 后加入 `managedTargets`（存 `localStorage`）并设为 `selectedWindowId`。
3. **Step 3/5（轮询等待页面加载）**：最多循环 24 次、每次间隔 500ms（最长约 12 秒）：
   - 调用 `scanSafariWindow(created.windowId).catch(() => null)`
   - 调用 `executeSafariJavaScript(created.windowId, "({readyState:document.readyState,href:location.href,title:document.title})")` `.catch(() => "")`
   - 若 `attempt >= 3` 且页面仍停留在 `about:blank`，尝试一次 `window.location.replace(resolvedTarget)`。
   - 退出条件：`scan?.webAreaLoaded && hostMatched && lastReadyState === "complete"`（或 `attempt >= 16` 且为 `interactive/complete`）。
4. **Step 4/5（注入 Network Recorder）**：调用 `executeSafariJavaScript` 注入 `buildNetworkRecorderInstallScript()`，在页面 `window.__agyNetRecorder` 挂载状态并 hook `window.fetch` 与 `XMLHttpRequest.prototype.open/send`。
5. **Step 5/5（初始轮询与状态同步）**：调用 `buildNetworkRecorderPollScript()` 拉取一次初始快照，设置 `netRecording = true`、`netVerifyStatus = "verified"`，最后调用 `getSafariStatus()` 刷新窗口列表。

### 1.2 为什么会出现你遇到的现象？

| 现象 | 源码根因定位 |
| :--- | :--- |
| **没有初始化到对应的 Model 和 Network Settings** | `handleQuickInitAndRecord` 里**没有任何操作页面 DOM 或 AX 控件的代码**。所谓的「初始化到对应 model」目前仅靠 URL 里的 `?model=antigravity-preview-09-2026` 参数；如果页面发生登录跳转、重定向或 AI Studio 默认忽略/覆盖了 query 参数，系统不会去点击 Model 选择器，也不会去配置右侧 Settings 或 `Presets`（`sharedPresets` 目前在前端仅作 JSON 只读展示，见 `App.tsx:2715`）。 |
| **没有成功打开新页面 / 一直停在空白或旧状态** | 1) 若 Safari 刚启动或未处于激活态，`make new document` 后 `delay 0.2` 立刻取 `front window`，可能抓错窗口或拿到尚未创建完成的文档。<br>2) 若未在 Safari 开启 **「开发 → 允许来自 Apple 事件的 JavaScript」**，或 macOS Automation 权限受阻，`openManagedWindow` / `executeJavaScript` 会失败。 |
| **控制台没有报错，不知道它在干什么** | 1) Step 3/5 轮询里使用了 `.catch(() => null)` 和 `.catch(() => "")`（`App.tsx:1787, 1799, 1824`），把底层报错全部静默吞掉了，导致 24 次循环默默跑完 12 秒。<br>2) 所有进度提示只更新在 `netVerifyDetail`（即右侧 **Network Tab 顶部的小卡片**里），若你当前没有盯着那个卡片，或流程在最后被其它事件覆盖，看起来就像「什么也没发生也没有报错」。 |

---

## 三、问题 2 深度解析：Network 状态为什么反复横跳（一会儿提示未注入，一会儿又提示其他）？

在 `App.tsx` 中，Network 录制状态由 4 个**全局单份** React state 控制（第 174–179 行）：
- `netRecording`
- `netSnapshot`
- `netVerifyStatus`（`"unchecked" | "verified" | "not_installed" | "failed"`）
- `netVerifyDetail`

它们没有按 `windowId` 隔离，同时被以下 **5 个异步源** 互相踩踏：

1. **SPA 路由 `pushState` 触发 URL 变更误降级（`App.tsx:257-260`）**
   - 只要 `syncSafariWindows` 检测到当前选中窗口的 `activeWin.url !== lastSelectedUrlRef.current`，就会把 `netVerifyStatus` 从 `verified` 强行降级成 `unchecked`，并提示 `URL changed to ...; click Poll / Verify Net to confirm recorder.`。
   - 实际上 AI Studio 在新建对话、发送消息后通过 `history.pushState` 改变 URL 时，**页面并没有刷新，`window.__agyNetRecorder` 依然正常工作**，这是纯误报。
2. **`AXLoadComplete` 通知误降级（`App.tsx:481-484`）**
   - 监听到 Safari 发出 `AXLoadComplete` 时，直接把 `netVerifyStatus` 改成 `unchecked`（`Page AXLoadComplete detected; click Poll / Verify Net to confirm recorder.`）。
   - 问题在于子 iframe 加载或某些动态节点更新也会触发 `AXLoadComplete`，导致刚刚还是绿标 `Injection Verified`，下一秒就变成黄标 `Unchecked`。
3. **切换选中窗口不隔离、不校验**
   - 窗口 A 注入成功后是 `verified`；你在左侧窗口列表点击切换到窗口 B，UI 不会自动清空或重新探针，而是继续显示窗口 A 的 `verified`；等你点 `Poll / Verify Net` 或 `Clear Net Buffer` 时，才突然跳变成 `Not Installed`。
4. **EMQX 远端指令与本地 UI 串台（`App.tsx:1300-1358`）**
   - 远端通过 EMQX 发来 `net_record_start` / `net_record_poll` 时，即便它操作的是另一个 `req.windowId`，也会直接覆盖本地 UI 的 `netSnapshot` 和 `netVerifyStatus`。
5. **纯内存注入在真实页面跳转/刷新后必然丢失**
   - `window.__agyNetRecorder` 挂载在当前页面 JS 上下文中。一旦发生真实导航（登录跳转、页面刷新、或用户在同窗口切换到另一个 Tab），旧上下文销毁，下一次 Poll 就会立刻报 `Not Installed`。

---

## 四、问题 3 深度解析：当前 Injection 机制（PID / Window ID / Tab ID）

### 4.1 架构分层关系

1. **PID 层（进程级）**：
   - `SafariController.swift:15-17` 通过 `NSRunningApplication.runningApplications(withBundleIdentifier: "com.apple.Safari").first` 获取 Safari 的唯一 `pid`。
   - `AxObserverManager.swift` 对这一个 `pid` 注册全局 `AXObserver`（监听窗口创建、销毁、标题变化、焦点变化、`AXLoadComplete`、`AXLiveRegionChanged`）。
2. **Window ID 层（窗口级，当前核心句柄）**：
   - 窗口标识使用的是 Safari 的窗口 ID（AppleScript `id of w`，对应 macOS `CGWindowID`）。
   - **不是针对所有 Safari 窗口注入**：前端通过 `managedTargets` 区分「受管窗口」与「Safari 其它窗口」，且**只有主动调用注入时，才会对指定的单个 `windowId` 执行 `do JavaScript`**。
3. **Tab ID 层（缺失层）**：
   - `SafariController.swift:263-264` 的实现是：
     ```applescript
     set targetWin to first window whose id is \(windowId)
     set jsResult to do JavaScript "\(escaped)" in current tab of targetWin
     ```
   - 当前**没有 Tab ID 概念**。它永远作用于目标 `windowId` 的 **`current tab`**。如果你在同一个 Safari 窗口里新建/切换了 Tab，虽然 `windowId` 没变，但 `current tab` 已经换成了另一个页面，注入状态自然就变成了 `Not Installed`。

---

## 五、完整优化与重构方案（详细落地方案）

### 方案 A：按 `windowId` 隔离状态 + 轻量探针（彻底解决 Network 状态乱跳）

1. **数据结构改造（`App.tsx`）**
   - 将全局的 `netRecording / netSnapshot / netVerifyStatus / netVerifyDetail` 替换为：
     ```ts
     interface WindowNetState {
       verifyStatus: "unchecked" | "verified" | "not_installed" | "failed";
       verifyDetail: string;
       recording: boolean;
       snapshot: NetworkRecorderSnapshot | null;
       lastTabUrl: string;
       lastProbedAt: number;
       autoInject: boolean; // 该窗口是否开启自动注入
     }
     // Map<windowId, WindowNetState>
     const [netStateByWindow, setNetStateByWindow] = useState<Record<number, WindowNetState>>({});
     ```
   - 界面展示时只读取 `netStateByWindow[selectedWindowId]`；EMQX 远端操作 `targetWin` 时只更新 `netStateByWindow[targetWin]`，彻底杜绝串台。

2. **增加轻量级探针脚本（`networkRecorderScript.ts`）**
   - 新增 `buildNetworkRecorderProbeScript()`，只做极小开销检查（不序列化 `entries` 大数组）：
     ```js
     (function(){
       var r = window.__agyNetRecorder;
       return {
         installed: Boolean(r && r.installed),
         recording: Boolean(r && r.recording),
         activeStreams: r ? r.activeStreams : 0,
         entriesCount: r ? r.entries.length : 0,
         interactionsCount: r ? r.interactions.length : 0,
         readyState: document.readyState,
         href: String(location.href || "")
       };
     })()
     ```

3. **用「静默探针」取代「强行降级为 Unchecked」**
   - 当 `AXLoadComplete` 触发、或 `syncSafariWindows` 发现 URL 变化、或用户在 UI 切换 `selectedWindowId` 时：
     - **不要**直接把状态改写为 `unchecked`！
     - 改为防抖 250ms 后自动调用一次 `probe`：
       - 若返回 `installed: true` → **保持 `Injection Verified` 绿标**，并更新 `entriesCount` / `href`，用户完全无感。
       - 若返回 `installed: false` → 此时才真正确认页面已刷新/切 Tab 导致脚本丢失：
         - 若该窗口开启了 `autoInject`，直接自动重新注入；
         - 若未开启 `autoInject`，再准确显示 `Not Installed` 并提示原因。

---

### 方案 B：自动检测与 Auto-Injection 机制（解决手动频繁重注入）

1. **全局 + 单窗口双层开关**
   - 工具栏或 Network 面板增加 `Auto-Inject` 开关（默认对通过 `Init & Record` / `Open New` 创建的受管窗口开启 `autoInject = true`，对非受管窗口默认关闭，避免干扰用户日常浏览的 Safari 窗口）。

2. **事件驱动 + 兜底的自动注入闭环**
   - **触发器 1（`AXObserver` 事件）**：收到 `AXLoadComplete` 或 `AXTitleChanged` 且 `ev.windowId` 在 `autoInject` 列表中。
   - **触发器 2（窗口/Tab 变化）**：`getSafariStatus` 发现受管窗口的 `url` 或 `activeTabIndex` 发生变化。
   - **防重入与状态机保护**：
     - 维护 `injectingWindowsRef = useRef<Set<number>>(new Set())`，防止同一窗口并发多次注入。
     - 先跑 `probe`：如果 `href` 为 `about:blank` 或 `safari-resource:` 等内部页，跳过注入。
     - 如果 `installed === false` 且 `readyState === "interactive" || readyState === "complete"`，立即调用 `buildNetworkRecorderInstallScript()`，并在成功后更新 `netStateByWindow[wid]` 为 `verified`。
     - 如果 `readyState === "loading"`，延迟 350ms 重试（最多 5 次），一旦进入 `interactive` 立刻注入（比等 `complete` 更早，能抓到更多页面初始化请求）。

3. **Tab 切换感知（`SafariController.swift` 小幅增强）**
   - 在 `listWindowsWithSource`（`SafariController.swift:283-312`）中多返回一列 `index of current tab of w`（当前激活 Tab 序号）和 `URL of current tab of w`。
   - 前端即可精确感知「同一 `windowId` 下用户切换了 Tab」，立即触发一次 `probe` + `autoInject`。

4. **（可选进阶）首字节 100% 不漏抓的终极形态**
   - 说明一下 AppleScript `do JavaScript` 的物理上限：它只能在页面文档创建之后执行，如果页面在 `interactive` 之前就发出了最早的 `fetch`，仍可能漏掉最初 1~2 个请求。
   - 若未来需要 `document_start` 级别零漏抓，可增加一个极简的 **Safari Web Extension（Content Script `run_at: "document_start"`）** 仅负责把 `networkRecorderScript` 注入目标域名，而 `p2p-remote-controller` 继续通过 AppleScript 读写 `window.__agyNetRecorder`。现阶段用上面的 **AXObserver + Probe + Auto-Inject** 即可零额外安装成本解决 95% 的痛点。

---

### 方案 C：`Init & Record` 流程重构（真正打通 开窗 → Model → Settings → 录制）

建议将 `handleQuickInitAndRecord` 重构为清晰的 **3 阶段 Pipeline**，并补齐错误暴露与业务初始化：

1. **阶段 1：权限前置校验 + 稳定开窗（不再静默吞错）**
   - 执行前先检查 `automation?.authorized`；如果是 `launchSafari()` 刚拉起 Safari，等待 500ms 确认进程就绪再调用 `openManagedSafariWindow`。
   - **打开 Safari「允许来自 Apple 事件的 JavaScript」检测**：在首次 `executeSafariJavaScript` 若返回包含 `"Allow JavaScript from Apple Events"` / `"允许来自 Apple 事件的 JavaScript"`，立刻中止轮询，弹出明确的 `errorMessage` 指引用户去 Safari 菜单栏勾选，而不是傻等 12 秒。
   - **识别登录/重定向页**：轮询时如果检测到 `lastHref` 跳转到了 `accounts.google.com`，立即在 `netVerifyDetail` 和 `errorMessage` 提示「Safari 窗口需要登录 Google 账号，请先完成登录后再继续」，停止无意义等待。

2. **阶段 2：提前注入 + 开启 Auto-Inject**
   - 只要 `hostMatched` 且 `readyState === "interactive" || readyState === "complete"`，**立即注入** `networkRecorderScript`（不必死等 `AXLoaded && complete`，因为 AI Studio 的流式/长连接常拖住 `complete`），同时将该窗口标记为 `autoInject = true`。
   - 这样就算后续 SPA 内部发生二次跳转，Auto-Injection 机制也会自动接管重注。

3. **阶段 3：可配置的页面初始化动作（Model & Network Settings）**
   - 目前缺少的正是这一步。建议在 `Init & Record` 旁（或 `Presets` Tab）增加一个可选的 **Init Profile（初始化预设）**，复用项目在 `6a5813c` 已实现的 `wait_for_element` + `perform_action`（支持 `targetRegex`、`focusFirst`、`doubleCheck`）能力：
     1. **Model 校验与切换**：
        - 读取当前期望的 `model`（可从 `targetUrl` 的 `?model=` 参数解析，或从 `sharedPresets.modelPresets` 选择）。
        - 页面就绪后，通过 `executeSafariJavaScript` 或 AX `scanSafariWindow` 检查当前页面已选中的 Model 名称是否与目标一致；若不一致，自动定位 Model 选择器按钮点击展开并选择目标 Model。
     2. **Network / Run Settings 初始化**：
        - 检查右侧 Settings 面板状态，按预设（例如 Temperature、Thinking/Search 开关、Network Recorder `maxBodyChars` 等）依次执行校验与切换。
     3. **全程可观测性**：
        - 每一步（开窗、页面就绪、注入校验、Model 切换、Settings 应用）都写入 `recordTimelineEvent` 和 `consoleLogs`，并在失败时直接展示在顶部 `error-banner`，不再「无报错静默失败」。

---

## 六、建议实施优先级

1. **P0（立即止血，解决当前最大痛点）**：
   - 将 `netVerifyStatus / netRecording / netSnapshot` 改为按 `windowId` 隔离的 Map；
   - 移除 `URL changed` 和 `AXLoadComplete` 时直接强行降级为 `unchecked` 的逻辑，改为调用轻量 `probe`；
   - 修复 `Init & Record` 轮询中 `.catch` 静默吞错的问题，明确暴露 Apple Events JS 权限报错与登录页跳转提示。
2. **P1（体验质变）**：
   - 上线 **Auto-Detect + Auto-Inject** 闭环（含 `SafariController.swift` 返回 `activeTabIndex` 感知同窗切 Tab）。
3. **P2（功能闭环）**：
   - 在 `Init & Record` 中接入 **Stage 3（Model & Run Settings 自动初始化 Pipeline）**，并与 `Presets` Tab 联动。
