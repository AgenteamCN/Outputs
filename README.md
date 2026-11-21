# agyPreviewInstant (`a2`) 修复与验证报告

- **Verified Bot Commit**: `f2e95a4` (`Antigravi-ty/agyPreviewInstant`, branch `a2`)
- **验证结果**: `tsc --noEmit`、`eslint src/`、`vitest run`（14 个测试文件，51 个用例）全部通过。
- **构建提示**: 本次包含 `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift` 改动，macOS 端拉取 `a2` 后请执行 `pnpm p2p:build:app`。

## 修复内容
1. **修复 P2P 下 LLM 刷新与 Direct Scan AX 丢失数据**
   - `apps/p2p-remote-controller/src/hooks/useCellularP2pHost.ts`: `pocket_deck_action` 在执行 `scan` / `select_window` / `open_new_window` 时保留并返回 `elements`。
   - `apps/p2p-remote-controller/src/components/PocketDeckView.tsx`: `sendAction` 合并快照而非整包覆盖，避免清空已拉取的 `elements`、`consoleLogs` 和 `timeline`。
2. **修复 `Send & Run` 正则匹配失败 (`No element matched /^Run$|.../i`)**
   - `apps/p2p-remote-controller/src/hooks/usePocketDeckBridge.ts`: `targetRegex` 改为分别匹配 `title`、`description`、`value`、`identifier`，不再将 `el.role`（如 `AXButton `）拼接到字符串开头导致 `^...$` 锚点失效。
3. **Pocket Deck Header 亮暗色对齐、状态徽章与实时流量**
   - `apps/p2p-remote-controller/src/components/PocketDeckHeader.tsx`:
     - 设置菜单弹窗背景从不存在的 `var(--bg-elevated, #161b22)` 改为 `var(--surface)`，对齐亮/暗色模式。
     - `#commit · Xm ago` 在 Cached 模式下显示黄色（`var(--warning)`），Live（LAN/P2P）下显示灰色（`var(--text-muted)`）。
     - `EMQX` 徽章：已连接绿色（`ok`）、连接中黄色（`warn`）、断开/异常红色（`err`）。
     - `P2P` 激活时显示绿色徽章并实时展示累计流量（如 `P2P · 14.2 KB`）。
4. **Pocket Deck Bridge UI 优化**
   - `apps/p2p-remote-controller/src/components/PocketDeckView.tsx`: 在 `All` 按钮上方增加 `↓` 一键滚动到底部按钮；为滚动容器启用 `overflowAnchor: "none"` 并在 textarea 自动增高时保持贴底，消除输入换行时的页面抖动。
5. **Models (3+2) 加载状态与 Force URL**
   - `apps/p2p-remote-controller/src/components/PocketDeckLlmControlTab.tsx`: 切换模型或点击 `Force URL` 时显示 `Loading...` / `Ready` 状态徽章，并自动轮询 `scan` 校验目标模型与输入框就绪，同时提供 `Reload` 按钮。
6. **EMQX International Pop Up 跳板模式**
   - `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift`: `popup` 模式改为打开 `https://cloud-intl.emqx.com/console/deployments?type=broker` 跳板页，等待域名 commit 后执行 `window.open(..., 'popup=yes,...')` 并置 `opener = null`，通过 `beforeIds` 差集定位新 popup `windowId` 后关闭跳板页。
   - `useSafariWindowActions.ts`、`useSafariBatchOperations.ts`、`usePocketDeckBridge.ts`: `Open New`、`+3 Win` 和 `Open New (Mobile)` 默认传入 `{ mode: "popup" }`。
7. **P2P Hot Update 现状核查**
   - `apps/p2p-remote-controller/src/services/lanService.ts` 的 `triggerManualPocketDeckUpdate()` 目前仅调用 `ServiceWorkerRegistration.update()`（依赖源站 HTTP 可达），尚未实现通过 WebRTC DataChannel 传输 `dist/` 分片的纯 P2P 双缓冲热更新。
