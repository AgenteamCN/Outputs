# Rocket Companion Tauri 修复与版本识别实现报告

## 1. 分支与 Verified Bot Commit 推送结果

- **分支**：`z/agy/rc/tauri/icon-and-build-id`（基于 `7ea9841` / `origin/z/user/tauri/cargowhatagain1`，其父提交为 `9de427ec2ec1f4d00fa4a191ea00fdf45adb3d2b`）
- **Verified Commit**：`3fa2386800094c276f9b76b27f16b24c4920c265`（`signature.isValid: true`, `state: VALID`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）
- **链接**：https://github.com/Antigravi-ty/RocketCompanion/commit/3fa2386800094c276f9b76b27f16b24c4920c265
- **`Cargo.toml` / `Cargo.lock` 说明**：应该上传仓库。`7ea9841` 启用了 `"macos-private-api"` feature，与 `tauri.conf.json` 中的 `"macOSPrivateApi": true` 配套必需。

---

## 2. 模块化实现内容

1. **构建期 Commit ID 注入（`build.rs` + `src/build_info.rs`）**
   - `build.rs` 在编译期读取 `git rev-parse --short=7 HEAD` 与完整 SHA（回退 `GITHUB_SHA` / `"dev"`），并配置 `cargo:rerun-if-changed`。
   - `build_info.rs` 统一格式化托盘 `Launch (<commit_id>)`、托盘 Tooltip `Rocket Companion (<commit_id>)` 与窗口标题 `Rocket Companion (<commit_id>)` / `Rocket Companion — Launch Configurator (<commit_id>)`，确保 Windows 与 macOS 同一提交产物显示一致。

2. **Crossbar #1 应用 / Dock / 窗口图标恢复（`src/app_icon.rs` + `Cargo.toml`）**
   - 在 macOS 通过 `objc2` 调用 `NSApplication::sharedApplication.setApplicationIconImage:` 设置 `icons/128x128@2x.png`，修复 `ActivationPolicy::Accessory` <-> `Regular` 切换或 `tauri dev` 运行时回退为黑色命令行 `exec` 图标的问题。
   - 在 Windows/Linux 为 `main` 与 `configurator` 窗口调用 `win.set_icon(...)`。

3. **macOS 原生菜单与 `About Rocket Companion`（`src/app_menu.rs`）**
   - 替换默认的 `About rc_tauri_shell`，提供 `About Rocket Companion` 菜单项及原生 About 弹窗（展示 `Rocket Companion`、`v1.0.0 (<commit_id>)`、`Build Commit: <full_sha>` 与 Crossbar #1 图标）。

---

## 3. 验证结果

- `pnpm check`（`radar` + `tsc --noEmit`）：**通过（GREEN，0 错误）**。
- `cargo check`：在沙盒内已通过 `cargo` 下载并开始编译依赖，因当前 Ubuntu 沙盒未安装 `cc` 链接器而终止于 `linker 'cc' not found`；Rust 源码需在 macOS / Windows 本机运行 `pnpm run tauri:dev` 或 `cargo check` 完成最终编译验证。
