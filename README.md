# agyPreviewInstant 修复与功能增强交付报告

**Commit**: `f8c547eb6d5ec2d138ac375402e932780d9000c9` (`a2`, Verified Bot Commit `VALID`)  
**GitHub**: https://github.com/Antigravi-ty/agyPreviewInstant/commit/f8c547eb6d5ec2d138ac375402e932780d9000c9

---

## 一、核心改动汇总

### 1. 按 `windowId` 隔离 Network 状态 + 轻量探针（解决状态反复横跳）
- **`apps/p2p-remote-controller/src/types/ax.ts` & `apps/p2p-remote-controller/src/App.tsx`**:
  - 将原先全局单份的 `netRecording` / `netSnapshot` / `netVerifyStatus` / `netVerifyDetail` 重构为按 `windowId` 隔离的 `netStateByWindow: Record<number, WindowNetState>`。
  - EMQX 远端指令（`net_record_start` / `net_record_stop` / `net_record_poll` / `reload_window` / `close_window`）与本地 UI 操作均只读写目标 `windowId` 对应条目，不再互相覆盖。
- **`apps/p2p-remote-controller/src/services/networkRecorderScript.ts`**:
  - 新增 `buildNetworkRecorderProbeScript()`，只返回 `installed` / `recording` / `activeStreams` / `entriesCount` / `interactionsCount` / `readyState` / `href`，不序列化 `entries` 大数组。
  - 移除 URL 变化和 `AXLoadComplete` 时强行降级为 `unchecked` 的逻辑，改为防抖 `scheduleWindowProbe()`：若脚本仍存活则保持 `Injection Verified` 绿标并刷新计数；仅当确认脚本丢失时才触发后续处理。

### 2. `Open New` 全自动注入 + 切 Tab 感知 + Auto-Inject 闭环
- **`apps/p2p-remote-controller/src/App.tsx`**:
  - 通过 `Open New`（`handleOpenManagedWindow`）打开的所有新窗口默认开启 `autoInject: true`，并立即调度 `ensureWindowInjected()`（带 `injectingWindowsRef` 并发防重入保护，若 `readyState === "loading"` 则每 350ms 重试最多 5 次，一旦进入 `interactive`/`complete` 立即注入，跳过 `about:blank` / `safari-resource:` 等内部页）。
  - Network 面板顶部新增当前选中窗口的 `Auto-Inject` 开关与 `windowId=` 标识。
- **`packages/macos-ax-helper/Sources/MacosAxHelper/` (`SafariController.swift`, `Models.swift`, `AccessibilityEngine.swift`)**:
  - `listWindowsWithSource()` 新增返回 `activeTabIndex`（`index of current tab of w`）、`zOrder`、`isFrontWindow`、`lastFocusedAtMs`。
  - 前端 `syncSafariWindows()` 检测到同一 `windowId` 下 `url` 或 `activeTabIndex` 变化时，自动触发 `scheduleWindowProbe()` 与自动重注。

### 3. `Init & Record` 更名为 `Initialize` + 最近一次使用窗口检测 + 明确错误暴露
- **`packages/macos-ax-helper/Sources/MacosAxHelper/AxObserverManager.swift` & `SafariController.swift`**:
  - `AxObserverManager` 监听 `kAXFocusedWindowChangedNotification` 与 `kAXMainWindowChangedNotification`，记录 `lastFocusedWindowId` 与 `lastFocusedTimestampMs`。
  - `SafariController.getStatus()` 新增返回 `mostRecentWindowId` 与 `mostRecentSource`（优先 `ax_focused`，其次 Safari Z-order 最前的 `front_window`）。
- **`apps/p2p-remote-controller/src/App.tsx` (`handleInitialize`)**:
  - 按钮已更名为 `Initialize`：
    - 若 Safari 已有打开窗口：自动选中 `mostRecentWindowId`（最近一次使用的 Safari 窗口），纳入受管并开启 `autoInject: true`，执行扫描与注入校验。
    - 若 Safari 未运行或无窗口：启动并打开目标 URL，轮询期间若检测到「允许来自 Apple 事件的 JavaScript」未开启或重定向至 `accounts.google.com` 登录页，立即中止并直接在顶部 `error-banner` 与 Network 状态卡暴露明确提示，不再静默空转 12 秒。

### 4. 自定义 Tab 别名 + URL Path 与 Title 双行展示
- **`apps/p2p-remote-controller/src/types/ax.ts` & `apps/p2p-remote-controller/src/App.tsx`**:
  - `ManagedTargetRecord` 新增持久化字段 `alias?: string`。
  - 窗口列表芯片（`.window-chip`）改为双行展示：
    - **第一行**：`[Alias]`（如有）+ `formatUrlPath(win.url)`（即 Domain 之后的 Path + Query + Hash）。
    - **第二行**：`#windowId` + 页面 `Title` + Tab 数量 + `[Recent]` / `[Auto]` 标记。
  - 窗口列表下方新增 `Rename / Alias` 内联编辑栏，支持随时设置、修改或清空自定义别名。

---

## 二、验证结果
- `pnpm --filter p2p-remote-controller check`（`tsc --noEmit && eslint src`）：**0 errors**（6 个既有 `react-hooks/exhaustive-deps` warning）。
- `pnpm --filter p2p-remote-controller build`（`tsc && vite build`）：**构建成功**（`dist/assets/index-BvqE-4B_.js`）。
- 注：沙箱为 Linux x86_64 环境，Swift helper（`packages/macos-ax-helper`）需在 macOS 本地随 Tauri 构建时一并编译验证。
- 已通过 EMQX Bridge（`agy_p2p_v1/chat`）向客户端发送本次更新通知。
