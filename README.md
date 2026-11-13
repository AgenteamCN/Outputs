# Rocket Companion Tauri 修复与版本识别实现报告

## 1. 分支与 `Cargo.toml` 归属说明

- **新建分支**：`z/agy/rc/tauri/icon-and-build-id`（本地最新提交 `43a937d`），基于 `7ea9841`（`origin/z/user/tauri/cargowhatagain1`，即在 `9de427ec2ec1f4d00fa4a191ea00fdf45adb3d2b` 基础上增加了 `macos-private-api` feature）。
- **`Cargo.toml` / `Cargo.lock` 是否应该上传仓库**：**应该上传**。
  - 对于 Rust / Tauri 桌面应用，`Cargo.toml` 与 `Cargo.lock` 是跨平台（macOS / Windows）可复现构建的必需文件。
  - `7ea9841` 在 `src-tauri/Cargo.toml` 中为 `tauri` 启用了 `"macos-private-api"` feature，这与 `src-tauri/tauri.conf.json` 中的 `"macOSPrivateApi": true` 直接对应（缺少该 feature 会导致 macOS 私有 API 编译/校验报错），因此基于 `7ea9841` 推进是正确的。

---

## 2. 模块化设计与改动清单

遵循低耦合、按功能拆分的原则，将新增逻辑拆分为 3 个独立小模块，避免主文件膨胀：

1. **构建期 Commit ID 注入（`build.rs` + `src/build_info.rs`）**
   - `build.rs` 在编译期自动执行 `git rev-parse --short=7 HEAD` 与 `git rev-parse HEAD`（若无 `.git` 目录则回退至 `GITHUB_SHA` 或 `"dev"`），并注入环境变量 `RC_GIT_COMMIT_SHORT` / `RC_GIT_COMMIT_FULL`，同时配置 `cargo:rerun-if-changed` 监听 `.git/HEAD`、`packed-refs` 与 `refs/heads`。
   - `build_info.rs` 提供跨平台一致的纯元数据格式化函数：
     - `format_tray_launch_label()` -> `"Launch (<7位commit>)"`
     - `format_tray_tooltip()` -> `"Rocket Companion (<7位commit>)"`
     - `format_window_title(base)` -> `"Rocket Companion (<7位commit>)"` / `"Rocket Companion — Launch Configurator (<7位commit>)"`
   - 无论 Windows 还是 macOS，只要基于同一 commit 编译，两端显示的短 commit ID 完全一致，方便多人联机测试版本对齐。

2. **Crossbar #1 应用 / Dock / 窗口图标恢复（`src/app_icon.rs` + `Cargo.toml`）**
   - **根因**：在 macOS 上使用 `ActivationPolicy::Accessory` 隐藏 Dock 图标并在打开窗口时切回 `Regular` 时（尤其在 `tauri dev` 未打包或状态切换期间），macOS 会重置应用图标并回退到默认的黑色命令行 `exec` 图标。
   - **修复**：
     - 在 `Cargo.toml` 中仅针对 macOS 引入 `[target.'cfg(target_os = "macos")'.dependencies] objc2 = "0.6"`（`Cargo.lock` 中已包含 `0.6.5`，无额外依赖膨胀）。
     - `app_icon.rs` 内嵌 `icons/128x128@2x.png`，在 macOS 通过 `NSApplication::sharedApplication` 调用 `setApplicationIconImage:` 强制设置 Crossbar #1 图标；在 Windows/Linux 通过 `win.set_icon(...)` 为 `main` 与 `configurator` 两个窗口设置 Crossbar #1 图标。
     - 在启动 `setup()`、`launch_main_window_with_config()`、`open_configurator_window()`、`update_macos_activation_policy()` 及 `RunEvent::Reopen` 切回 `Regular` 时均调用图标同步，确保 Base Game 与 Configurator 均显示标准 Crossbar #1 图标。

3. **原生应用菜单与 `About Rocket Companion`（`src/app_menu.rs`）**
   - **根因**：未自定义 macOS 菜单时，Tauri 默认使用 `Cargo.toml` 的 `package.name`（`rc_tauri_shell`），导致显示 `About rc_tauri_shell`。
   - **修复**：
     - `app_menu.rs` 在 macOS 上构建原生菜单栏（`Rocket Companion` / `Edit` / `Window`），将菜单项命名为 **`About Rocket Companion`**、`Hide Rocket Companion`、`Quit Rocket Companion`。
     - 通过 `AboutMetadata` 设置应用名 `Rocket Companion`、版本 `v1.0.0 (<commit_id>)`、完整 `Build Commit: <full_sha>` 以及 Crossbar #1 图标。
     - Windows/Linux 保持无顶部菜单栏的干净窗口外观，通过托盘 `Launch (<commit_id>)`、托盘 Tooltip 与窗口标题 `Rocket Companion (<commit_id>)` 保持跨平台版本标识一致。

4. **接线集成（`src/tray_manager.rs`、`src/window_manager.rs`、`src/lib.rs`）**
   - `tray_manager.rs`：托盘 `Launch` 项文本更新为 `Launch (<commit_id>)`，托盘 Tooltip 更新为 `Rocket Companion (<commit_id>)`。
   - `window_manager.rs`：新增 `apply_window_titles()`，在启动及每次打开 `main` / `configurator` 窗口时刷新标题与图标。
   - `lib.rs`：在 `setup()` 中依次初始化 `setup_app_menu`、`apply_app_and_window_icons`、`apply_window_titles`，并在 `Reopen` 时确保 macOS Dock 图标同步。

---

## 3. 验证与环境限制说明

- **Git 提交状态**：已提交至本地分支 `z/agy/rc/tauri/icon-and-build-id`（commit `43a937d`）。
- **Push 限制**：当前沙盒环境未配置 GitHub 推送凭据（`GIT_TERMINAL_PROMPT=0 git push` 返回 `could not read Username for 'https://github.com'`），未能自动推送到远端。请在本地/有凭据环境执行：
  ```bash
  git push -u origin z/agy/rc/tauri/icon-and-build-id
  ```
- **Rust 编译验证限制**：当前 Ubuntu 沙盒未安装 `cargo` / `rustc` 及 macOS/Windows 工具链，需在 macOS / Windows 本机运行 `pnpm run tauri:dev` 或 `cargo check` 进行最终编译与界面验证。
