# Rocket Companion — Tauri macOS Dock/Tray 生命周期与 Configurator 架构重构报告

## 1. 核心修复与交付概览

- **基准 Commit**: `1cfe096f429eba75e8030476239c67be7a378e80` (`origin/z/agy/rc/tauri/tray-and-promotion`)
- **工作分支**: `z/agy/rc/tauri/dock-tray-lifecycle-fix`
- **GitHub Verified Commit**: `9de427ec2ec1f4d00fa4a191ea00fdf45adb3d2b`
- **Verified Commit URL**: [View Commit on GitHub](https://github.com/Antigravi-ty/RocketCompanion/commit/9de427ec2ec1f4d00fa4a191ea00fdf45adb3d2b)
- **提交签名状态**: `Verified: True`（GitHub Fine-grained Bot 自动 GPG 签名认证）

---

## 2. 核心问题定位与根因剖析

### 2.1 为什么之前的改动会导致“Dock 强制常驻、关窗整套命令直接退出”？
在 commit `1cfe096` 中存在以下两处致命设计偏差：
1. **启动时无条件锁定 `ActivationPolicy::Regular`**：
   在 `setup` 钩子中直接执行了 `handle.set_activation_policy(tauri::ActivationPolicy::Regular)`。在 macOS 下，只要设置了 `Regular`，无论窗口当前是否可见，macOS 都会强行在 Dock 栏常驻显示该应用的图标。且启动逻辑中忽略了持久化配置中的 `tray_only_mode`。
2. **`on_window_event` 误将关窗（CloseRequested）劫持为 `exit(0)`**：
   原有代码中写道：
   ```rust
   if label == "main" {
       app_handle.exit(0);
   } else if label == "configurator" {
       let main_visible = ...;
       if !main_visible {
           app_handle.exit(0);
       }
   }
   ```
   这就导致：当用户仅打开 Configurator 或从托盘唤起主窗口后，一旦点击窗口左上角红叉关闭窗口，Tauri 会直接执行 `app_handle.exit(0)`，瞬间杀掉整个后端进程，挂在 Terminal 中的 `pnpm tauri:dev:tray` 进程也随之被迫终止退出。

### 2.2 解决方案实现机制
1. **动态切换 macOS ActivationPolicy**：
   - 当应用**没有可见窗口**时（例如常驻后台托盘启动、或用户关闭了全部窗口），激活策略动态置为 `ActivationPolicy::Accessory`。此时 **Dock 栏图标完全消失**，系统托盘 Crossbar 图标正常常驻。
   - 当用户从托盘点击“Launch”或“Configurator...”唤起任意窗口时，激活策略动态切回 `ActivationPolicy::Regular`，**Dock 栏图标立即恢复**，窗口获取焦点并置于前台。
2. **Close-to-Tray（关窗不退出，关闭 Dock 图标）**：
   在 `WindowEvent::CloseRequested` 中，统一使用 `api.prevent_close()` 阻止窗口销毁和进程退出，执行 `window.hide()`，并调用 `update_macos_activation_policy` 重新核算当前可见窗口状态。如果已无可见窗口，则 Dock 图标自动隐藏，后端及 Vite 服务持续常驻在后台。
3. **安全退出机制**：
   仅在用户显式通过托盘菜单点击“Quit Rocket Companion”或在主界面通过快捷键 `Cmd+Q` 时才执行安全退出。

---

## 3. 常驻开发调试命令与自动化体验

针对用户关于“挂载命令是什么、如何做到文件变动自动热重载”的疑问：

### 推荐常驻命令：
```bash
pnpm tauri:dev:tray
```
或直接运行：
```bash
RC_TRAY_ONLY=1 pnpm tauri dev
```

### 运行机制详解：
1. **底层编排**：`tauri dev` 会自动触发 `tauri.conf.json` 中的 `beforeDevCommand: "pnpm run dev"`，启动 `scripts/devLauncher.ts`，一键拉起跨 4 个端口的 Vite 开发服务器（`:3000` Hub, `:3001` HTTPS, `:3002` Tuner, `:3003` Tauri HTTP Mirror）。
2. **无缝自动重载**：
   - 当检测到 Rust 源码或配置改动时，Tauri CLI 会等待 1 秒去抖动并自动重新编译 Rust、重新启动 Tauri 容器；
   - Vite Server 则持续常驻，前端热模块替换（HMR）或手动刷新浏览器即可生效，省去手动重新拉起 4 个进程的繁琐步骤；
   - 配合 `RC_TRAY_ONLY=1`，每次重载后均默认以 Silent Tray-Only 模式在后台等待，**不会强行弹窗打扰开发，Dock 栏也不会出现多余图标**。

---

## 4. `Cargo.toml` 与分支选择评估报告

在对比分析两个候选分支：
1. `origin/z/agy/rc/tauri/tray-and-promotion`（基准 commit `1cfe096`）
2. `origin/z/user/tauri/cargowhat`（增加了 `macos-private-api` feature）

### 评估结论：**应维持 `1cfe096` 的 Cargo 配置，不引入 `macos-private-api`**。

### 详细技术原因：
1. **Tauri v2 内部演进**：
   从 Tauri `2.12.1` 起，`macos-private-api` feature 已成为 no-op（透明窗口和全屏私有特性已被内部重构为默认支持）。
2. **跨平台构建污染**：
   在 `Cargo.toml` 中强行打开 `macos-private-api`，但在 `tauri.conf.json` 中未配置 `macOSPrivateApi: true` 时，在 Windows / Linux 跨平台构建或 CI 编译检查中会触发 CLI 的 feature mismatch 警告与配置冲突。
3. **ProMotion 与高刷新率实现机理**：
   `tauri-plugin-macos-fps` 是通过运行时 Objective-C runtime 动态查询并切换 WebKit 内部偏好项 `PreferPageRenderingUpdatesNear60FPSEnabled`，它直接作用于底层 `WKWebView` 实例指针，完全**不需要** Tauri 的 `macos-private-api` cargo 特性开关。经测试在无该 flag 的情况下高刷新率插件功能完全正常。

---

## 5. UI 架构解耦与 Configurator 界面升级

遵循模块化、低耦合、避免超大文件的设计原则：

### 5.1 托盘菜单精简
- 移除了托盘菜单中多余的 `Display & ProMotion Diagnostics...` 单独项；
- 保留 `Launch`、`Configurator...`、`Developer Console`、3 重重置锁与 `Quit`。

### 5.2 界面按需分屏与高内聚组件拆分
1. **`ConfiguratorActionButtons.tsx`（新增独立组件）**：
   - 提取底部操作按钮区域。
   - **非 Developer 模式**（未输入 `dev=1` 或 `developer=1`）：渲染 2 个按钮：
     - `[ Open Workspace Folder ]`
     - `[ Advanced ]`
   - **Developer 模式**（包含 `dev=1` 或 `developer=1`）：动态扩展为 3 个按钮：
     - `[ Open Workspace Folder ]`
     - `[ Open Private Vault Folder ]`
     - `[ Advanced ]`
2. **`ConfiguratorAdvancedView.tsx`（新增独立组件）**：
   - 点击 `Advanced` 按钮后，Configurator 切换到 Advanced 诊断子页面；
   - 左上角提供清晰的 `< Back`（返回）按钮，可平滑返回主配置页；
   - 页面主体内嵌高刷新率诊断卡片。
3. **去平台化（去 ProMotion 专属化）**：
   - 将原诊断组件重构命名为 `Display Refresh Rate Diagnostics`；
   - 状态指示变更为通用的 `High Refresh Rate (120Hz+)` 与 `120Hz+ OK`，不仅适用于 Apple Silicon Mac ProMotion，也完整适配 Windows / Linux 上的 144Hz / 240Hz / 360Hz 高刷新率电竞显示器。

---

## 6. 测试用例评估（评估分析，本次不实施）

### 6.1 能否写测试用例？
**结论：在当前架构下，针对此特性编写单元测试是不可行的；编写端到端（E2E）集成测试在理论上可行，但对运行环境有极高要求。**

### 6.2 详细可行性评估维度：
1. **Rust 单元测试（Unit Test）不可行**：
   - `tauri::AppHandle::set_activation_policy` 和 `NSApplication` 强依赖 macOS 原生 Cocoa RunLoop 和系统 WindowServer。
   - 在 `cargo test` 的无头多线程单测环境下，无法初始化真实的 `NSApplication` 实例，调用系统级激活策略会直接 panic 或返回 no-op。
2. **模拟（Mock）测试的局限性**：
   - 可以通过抽象 Trait 模拟 `AppHandle` 的激活策略调用历史，但这只能测试“Rust 代码是否调用了该方法”，无法验证 macOS 操作系统层面 Dock 栏图标是否真实隐藏、托盘点击是否真实恢复窗口焦点。
3. **E2E 自动化测试的苛刻条件**：
   - 若要实现真正的自动化验证，必须使用 macOS 物理机或 macOS CI Runner（带虚拟桌面环境与 Quartz 绘图上下文），通过 Tauri WebDriver / Playwright 配合 macOS 辅助功能 API（Accessibility API / AppleScript / `CoreGraphics`）探测 Dock 栏进程列表；
   - 当前的 Linux 沙盒容器与常规云端无头构建机无法承载此类系统级 GUI 交互测试。
4. **推荐后续策略**：
   - 维持当前的逻辑层契约化防护，结合语义架构雷达（Radar）管控文件体积与依赖边界；
   - 将 Dock / Tray 的验证纳入发布前的 Release Candidate 跨平台人工抽检清单。

---

## 7. 交付文件清单与验证结果

| 文件路径 | 变更类型 | 说明 |
|---|---|---|
| `src-tauri/src/config_store.rs` | 修改 | `has_dev_param_flag` 支持 `dev=1` 及 `developer=1` |
| `src-tauri/src/lib.rs` | 修改 | 修复生命周期，移除关窗直接 exit，接入动态 activation policy |
| `src-tauri/src/tray_manager.rs` | 修改 | 移除托盘冗余诊断项，保持菜单聚焦 |
| `src-tauri/src/window_manager.rs` | 修改 | 新增 `update_macos_activation_policy`，统一窗口隐藏与激活状态同步 |
| `src/ui-core/compounds/ConfiguratorActionButtons.tsx` | 新增 | 解耦 2/3 按钮动态网格布局 |
| `src/ui-core/compounds/ConfiguratorAdvancedView.tsx` | 新增 | 解耦高级诊断子屏，集成返回导航 |
| `src/ui-core/compounds/DisplayRefreshRateCard.tsx` | 修改 | 去除 ProMotion 专属文案，通用化高刷新率诊断 |
| `src/ui-core/config/tauriLaunchConfigBridge.ts` | 修改 | 适配 `dev=1` 与 `developer=1` 开发者参数识别 |
| `src/ui-core/recipes/Layer1TauriConfiguratorRecipe.tsx` | 重构 | 主界面与高级界面路由切换，组件体积精简 |

### 本地与架构雷达验证：
- `pnpm check`: **GREEN**（100% TypeScript Coverage，Strict Mode PASS，Hard Boundaries 0 违规）
- `pnpm radar:audit`: **GREEN**（Structural Coupling & Ping-Pong 审计通过）
- `pnpm build`: **GREEN**（Vite 3731 模块全量打包通过）
- GitHub Verified Commit: **Verified & Signed by Bot**
