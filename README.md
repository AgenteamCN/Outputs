# P2P Remote Controller & Monorepo Workflow Report

- **Repository**: `Antigravi-ty/agyPreviewInstant`
- **Branch**: `z/p2p-remote-controller-monorepo`
- **Verified Bot Commit**: `6f2225c5f333ab404dfe46ff50b887023ad0999a`

---

## 1. 本次已完成修复（可直接拉取测试）

### 1.1 双终端隔离启动 + Scope-Aware Git Hook 自动重启
借鉴 `Antigravi-ty/RocketCompanion` 的 `.githooks` + `tauriDevWatch` 机制，已为本仓库实现**按作用域（Scope-Aware）自动重启**：
- **macOS Terminal.app（持有 TCC Accessibility / Automation 权限）**：
  ```bash
  pnpm p2p:dev:watch
  ```
- **VS Code Integrated Terminal（专注 AGY 开发）**：
  ```bash
  pnpm agy:dev:watch
  ```
- **工作机制**：
  - `.githooks/post-checkout`、`post-merge`、`post-rewrite` 在每次 `git checkout` / `git pull` / `git merge` / `git rebase` 时，通过 `git diff` 分别计算 `agyChanged`（`src/`、`src-tauri/`、`index.html`、`packages/shared-config/` 等）与 `p2pChanged`（`apps/p2p-remote-controller/`、`packages/macos-ax-helper/`、`packages/shared-config/` 等），写入 `.cache/git-checkout-event.json`。
  - `scripts/devWatch.ts` 监听该事件与 HEAD 状态：**只改了 P2P 代码时，AGY 端保持运行、绝不重启或打断正在编辑的上下文**；反之亦然。
  - 已执行 `pnpm hooks:install`（并写入 `package.json` 的 `prepare` 脚本），自动将本地 `core.hooksPath` 配置为 `.githooks`。

### 1.2 P2P Remote Controller UI 布局与交互修复
- **修复顶栏滑动与 200% 宽度溢出**：
  - `.app-shell` 锁定为 `height: 100vh; max-width: 100vw; overflow: hidden;`。
  - `.topbar` 与 `.control-strip` 增加 `flex-shrink: 0`（`.topbar` 设为 `position: sticky; top: 0;`），顶部控制栏不再随列表滚动而隐藏一半。
  - `.main-grid` 由 `1.2fr 1fr` 改为 `minmax(0, 1.2fr) minmax(0, 1fr)`，并为 `.panel`、`.panel-body`、`.element-card` 补齐 `min-width: 0; overflow: hidden;`，彻底解决左侧长文本把页面撑宽到 200%、右侧 Raw JSON 被挤没的问题。
  - 增加 `@media (max-width: 960px)` 响应式布局，窄屏/Pad 下自动切换为上下分栏。
- **默认 URL 更新**：默认目标 URL 从 `https://example.com` 改为 `https://aistudio.google.com`。
- **Regex 正则搜索**：在元素过滤框旁新增 `Regex` 开关，支持对 `role`、`subrole`、`title`、`description`、`value`、`identifier`、`nodeId`、`path` 进行大小写不敏感的正则匹配（例如 `agents|antigravity`），非法正则时输入框高亮红框且不崩溃。
- **窗口管理（Focus Current / Open New / 自动清理已关闭窗口）**：
  - Swift (`AccessibilityEngine.swift` / `JsonRpcServer.swift`) 新增 `focus_window` RPC，Rust (`commands.rs` / `lib.rs`) 新增 `ax_focus_window` 命令，通过 `NSRunningApplication.activate` + `kAXRaiseAction` + `kAXMainAttribute` 将选中 Safari 窗口置前。
  - 顶部按钮调整为 **`Open New`** 与 **`Focus Current`**。
  - 新增 `syncSafariWindows`：每次刷新或遇到 `not found` 错误时，自动从 `managedWindowIds` 与 `selectedWindowId` 中剔除用户手动关闭的窗口。
- **SPA 加载状态与延迟重扫**：
  - Swift `scanWindow` 新增读取 `AXWebArea` 的 `AXLoaded` 属性，并在面板顶部显示 `AXLoaded` / `AXLoading...` 状态徽章。
  - 工具栏新增 `Auto-Rescan` 开关（默认开启）：执行 `Press` / `Focus` / `Set Value` 后等待 `350ms` 再重扫 AXTree，提升对 SPA 懒加载面板的捕获率。

### 1.3 引入 RocketCompanion 风格的代码规范 Radar
- 新增 `scripts/radar.ts`（`pnpm radar`），扫描 `src/`、`apps/p2p-remote-controller/src/`、`packages/shared-config/src/`、`scripts/`：
  - 检查 Emoji 禁用、非英文字符（CJK）禁用、`any` 密度上限（单文件 <= 5）。
  - 统计上下文体积告警（Soft: >400 行或 >12 KB；Hard: >800 行或 >18 KB），并串联 `pnpm --filter p2p-remote-controller check`。

---

## 2. 技术问答与评估

### 2.1 Diagnostic JSON 隐私检查
已核查你粘贴的完整 Diagnostic 输出，**未发现敏感信息泄露**：
- 不含 API Key、Token、Cookie、邮箱、个人域名或公网 IP。
- `executablePath` 仅包含 `/Volumes/Wine/AntigravityGemini/agyPreviewInstant/...`，未暴露 macOS 用户名主目录（`/Users/<name>`）。
- `scanResult` 仅包含 Google AI Studio 公开的模型选择列表文本。

### 2.2 元素列表显示的是“无障碍标签”还是“页面文本”？为什么能直接 Press？
- UI 卡片标题按优先级取：`el.title || el.description || el.value || el.identifier || el.nodeId`（分别对应 macOS AX 的 `AXTitle`、`AXDescription`、`AXValue`、`AXIdentifier`）。
- Google AI Studio 的模型卡片和按钮使用了标准的 HTML `<button>` / `<a>` 与 ARIA 标签，因此 macOS WebKit Accessibility Tree 将其映射为 `AXButton` / `AXLink`，并暴露 `AXPress` 动作。
- **`Press` 与 `Focus` 的区别**：`AXPress` 可在 Safari 处于后台时直接触发点击（适合后台静默自动化）；`AXFocused` 只是设置焦点，并不负责把 Safari 窗口切到最前——置前窗口现在由新增的 `Focus Current`（`kAXRaiseAction`）负责。

### 2.3 SPA 按需加载状态的捕获可能性
- **当前可用（纯 AX 层）**：`AXWebArea` 提供 `AXLoaded`（文档级加载完成状态）和 `AXBusy`。但对于 SPA 点击某个按钮后的**局部异步请求**（如点击 Link API Key 后动态拉取数据），Safari 的 `AXLoaded` 通常仍为 `true`。
- **可行方案**：
  1. **短期（已实现）**：动作后延迟 `350ms` 重扫（`Auto-Rescan`）。
  2. **中期（纯 AX 轮询/稳定判定）**：在自动化工作流步骤中引入 `waitForElement(regex, timeoutMs)`，每 200ms 轮询 AXTree 直到目标控件出现或连续两次扫描节点数稳定。
  3. **终极方案（如需 100% 精确网络/DOM 状态）**：通过 Safari AppleScript `do JavaScript`（需 Safari 开启 *Allow JavaScript from Apple Events*）或配套 Safari WebExtension 监听 `MutationObserver` 与 `fetch` 队列。

### 2.4 TCC 权限方案评估：借用 Terminal.app vs 预编译“长期不变壳子”
- **现阶段（快速迭代期）**：推荐**继续用系统 Terminal.app 运行 `pnpm p2p:dev:watch`**。原因：只要还在频繁改 Rust / Swift 代码，每次重新编译未公证的 debug 二进制都会改变 CDHash，使用固定壳子反而可能因辅助 binary 变更触发 TCC 失效；开发完成后，在 *系统设置 -> 隐私与安全性 -> 辅助功能 / 自动化* 中取消勾选 Terminal 即可收回权限。
- **进入日常使用 / 测试期**：推荐**预编译一个稳定的 `.app` 壳子**（`pnpm p2p:build:app` 并用固定开发者证书或自签名证书签名），把 `macos-ax-helper` 内嵌其中并授予 TCC。只要原生二进制不动，前端 JS/CSS 可以通过远程/本地热更新加载，无需重复授权。

---

## 3. 未来工作步骤（Roadmap）

1. **Step 1：本地拉取并重编 Swift Helper 验证（立即）**
   - 拉取最新 `z/p2p-remote-controller-monorepo` 分支。
   - 运行 `pnpm ax-helper:build`（因为本次新增了 `focus_window` 与 `webAreaLoaded`）。
   - 在 Terminal.app 运行 `pnpm p2p:dev:watch`，在 VS Code 运行 `pnpm agy:dev:watch`，验证 UI 布局、Regex 搜索、`Focus Current` 以及关闭窗口后的自动清理。
2. **Step 2：Automation Workflow 引擎（核心下一步）**
   - **录制模式（Action Recorder）**：监听用户在 P2P 控制器中的点击序列，记录语义签名（`role + title/regex + path`），而不只依赖易变的绝对 `path` 数组。
   - **步骤编排器**：支持 `waitForElement(regex)` -> `press` -> `set_value` -> `press`（例如自动在 Google AI Studio 切换到 `Agents` / `AntiGravity` 并选定模型）。
   - **后台静默执行**：利用 `AXPress` 不抢占前台的特性，在后台完成 Safari 操作，不影响主机正常编码。
3. **Step 3：P2P 远程传输与移动端/平板响应式界面**
   - 将当前右侧 `Mobile Preview` 升级为独立响应式路由/视图，通过现有 MQTT / WebRTC 通道把手机端的点击指令转发给 Mac 后台的 `macos-ax-helper`。
4. **Step 4：按 Radar 指标拆解大文件**
   - 后续迭代中逐步拆分 `apps/p2p-remote-controller/src/App.tsx`（当前 27.4 KB，已触发 Radar 18 KB 硬上限提示）为 `TopBar`、`SafariControlStrip`、`AxElementList`、`InspectorPanel` 等独立模块。
