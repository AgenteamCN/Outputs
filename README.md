# P2P Remote Controller 落地交付与技术答疑报告

> **Git 分支**：`z/p2p-remote-controller-monorepo`（基于 `a2`）  
> **Verified Bot Commit**：`45906371512e07a7f7a68f342466ea0bcd21f322` (`verified: true`)  
> **检查结果**：`p2p-remote-controller`（`tsc --noEmit` + `eslint` + `vite build`）与根应用 `agy-preview-instant`（`tsc --noEmit` + `eslint` + `vitest` 13/13 + `vite build`）均通过，零错误。

---

## 一、你的几个核心疑问解答

### 1. 同一个 Desktop Space 下，后台「点点点 + 输入」同时前台正常干别的，到底行不行？
- **后台读取页面内容**：**100% 行**。`AXUIElement` 扫描不会移动物理鼠标，也不会抢占窗口焦点。
- **后台点击（`AXPress`）**：**部分可行**。对于标准 `<button>`、`<a>`、原生表单控件，调用 `kAXPressAction` 可以在不移动鼠标、不切换焦点的情况下触发点击；但对于现代 SPA 里用普通 `<div>` 绑定的自定义点击事件、或必须 Hover 才展开的菜单，后台 `AXPress` 可能无响应。
- **后台输入（`AXValue`）**：**看网页实现**。`AXUIElementSetAttributeValue(..., kAXValueAttribute, ...)` 可以在后台改值，但 React/Vue 等受控组件往往只监听 `input`/`keydown` 事件，不一定会因为 `AXValue` 变化而更新内部状态。如果真遇到不响应的输入框，才需要降级为短暂聚焦后用 `CGEvent` 模拟按键。
- **当前策略**：先把网页元素抓出来，看看你的目标网站到底是标准无障碍组件还是纯自定义 `<div>`，再决定是否需要降级方案。

### 2. 开发期二进制签名变化导致「辅助功能权限」失效 & 本地钥匙串自签名证书怎么玩？
macOS 的 TCC（辅助功能权限）会把权限绑定到应用的**代码签名身份（Code Signing Identity）**。默认 ad-hoc 签名（`-`）在每次重新编译后二进制哈希会变，导致系统认为它是“新程序”而要求重新授权。

**本地钥匙串创建自签名代码签名证书步骤：**
1. 打开 macOS 自带的 **钥匙串访问（Keychain Access）**。
2. 顶部菜单点击：`钥匙串访问 -> 证书助理 -> 创建证书...`
3. 填写信息：
   - **名称**：例如 `AGY Local Dev`
   - **身份类型**：`自签名根证书`
   - **证书类型**：选择 **`代码签名（Code Signing）`**
   - 点击创建（默认保存在「登录」钥匙串）。
4. 在钥匙串列表双击刚建好的 `AGY Local Dev` -> 展开 **信任** -> 将 **代码签名** 改为 **始终信任**。
5. 使用方式：
   - 编译 Swift Sidecar 时设置环境变量：
     ```bash
     export AGY_CODESIGN_IDENTITY="AGY Local Dev"
     ```
   - 打包 `.app` 时也可以将 `apps/p2p-remote-controller/src-tauri/tauri.conf.json` 中的 `bundle.macOS.signingIdentity` 改为 `"AGY Local Dev"`（注意本地自用即可，不要提交个人隐私敏感信息）。
   - 这样每次重编译签名身份固定，TCC 权限就不会频繁失效。
   - *(补充：在 `pnpm p2p:dev` 开发模式下，实际请求权限的宿主通常是你运行命令的终端 App，例如 Terminal / iTerm2 / VS Code，给终端本身授一次辅助功能权限在开发期最省事。)*

### 3. Safari 同一个 Desktop 下独立窗口、不最小化、被遮挡/最大化在后面是否就不会失活？
**是的，你的理解完全准确。**
只要满足三个条件：
1. 从标签栏独立成**单独的 Safari Window**（不要作为后台 Tab 塞在另一个窗口里）；
2. **不最小化**（`minimized == false`）；
3. 留在**同一个 Desktop Space** 下。

即使它被你当前正在工作的前台窗口**完全遮挡**（甚至它自己是最大化状态被盖在下面），Safari/WebKit 依然会把它视为活跃窗口容器，不会像后台 Tab 或最小化窗口那样迅速冻结 JS 与无障碍树。
*(如果极少数长驻网页仍因长时间无交互被系统 App Nap 降频，可在终端执行 `defaults write com.apple.Safari NSAppSleepDisabled -bool YES` 缓解，不需要时再删掉即可。)*

### 4. 不用 Accessibility Inspector，只装了 Xcode 就能直接用 Tauri 跑起来吗？
**是的，完全不需要打开 Xcode GUI。**
只要你安装过 Xcode，在终端确认过一次：
```bash
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```
之后全程只需要命令行即可完成 Swift 编译与 Tauri 调试。

---

## 二、已完成的代码架构与功能清单

1. **Monorepo 工作区改造**
   - `pnpm-workspace.yaml` 新增 `apps/*` 与 `packages/*`。
   - 原应用 `AGY Preview Instant` 保持根目录不变，原有命令（`pnpm dev` / `pnpm build` / `pnpm check`）零影响。
   - 新增根脚本：`pnpm p2p:dev`、`pnpm p2p:build`、`pnpm p2p:build:app`、`pnpm p2p:check`、`pnpm ax-helper:build`。

2. **`packages/macos-ax-helper`（SwiftPM 原生辅助进程，Apple Silicon `arm64`）**
   - `Package.swift` + `Models.swift` + `SafariController.swift` + `AccessibilityEngine.swift` + `JsonRpcServer.swift` + `main.swift`。
   - 通过 NDJSON stdin/stdout 与 Tauri Rust 后端通信，支持：
     - `check_accessibility`：检查并弹窗申请 `AXUIElement` 权限。
     - `get_safari_status` / `launch_safari`：检测 Safari 是否运行，支持后台拉起 Safari。
     - `open_managed_window`：新建专属受控 Safari 窗口并返回稳定 `windowId`（不受标题变化、Redirect、SPA 路由跳转影响）。
     - `scan_window_elements`：通过 `_AXUIElementGetWindow` 精确定位目标窗口内的 `AXWebArea`，抓取可交互元素与文本节点（含 `path`、`role`、`title`、`value`、`actions`、`frame`）。
     - `perform_action`：支持对指定 `path` 节点执行 `press`（`kAXPressAction`）、`focus`、`set_value`（`kAXValueAttribute`）。

3. **`apps/p2p-remote-controller`（全新 Tauri v2 应用）**
   - **独立配置**：`P2P Remote Controller`（`com.antigravity.p2premotecontroller`），独立开发端口 `1421`（可与 `AGY Preview Instant` 同时运行）。
   - **托盘与图标**：使用 Lucide `monitor-cloud` 生成黑白 Template 托盘图标（`tray-monitor-cloud.png`）与独立应用图标（`icon.icns` 等）；托盘右键菜单支持 **Show Window**、**Open DevTools Console**、**Quit**。
   - **Debug 与移动端预览 UI**：
     - 顶部提供权限状态、**Grant AX Permission**、**Console**（直接打开 DevTools）、**Quick Copy**（一键复制完整 JSON 快照）。
     - 支持隔离过滤，默认只扫描由本应用打开的 `managedWindowIds`，也可勾选 `Include All Safari Windows` 全量扫描对比。
     - 支持对抓到的元素直接点击 **Press / Focus / Set Value** 进行实时验证，并在右侧切换 **Raw JSON / Mobile Preview / Shared Presets**。

---

## 三、你在 Mac 上的启动步骤

```bash
git fetch origin
git checkout z/p2p-remote-controller-monorepo
pnpm install
pnpm p2p:dev
```

启动后：
1. 点击 **Grant AX Permission** 授予辅助功能权限；
2. 点击 **Open Managed Safari Window** 打开一个测试网页；
3. 点击 **Scan AXTree** 查看抓到的无障碍节点，点击右上角 **Quick Copy** 把结果贴给我，我们就可以根据真实节点结构继续推进 WebRTC / 私有协议与移动端 PWA 重建。
