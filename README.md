# D-Track (V1-Rust) 验证报告

- **仓库**：`Antigravi-ty/RocketCompanion`
- **分支**：`z/agy/rc/multiplayer-recheck`（当前 HEAD：`4ec70dc`，基于 Wave 0 `256577470a3188242e72362d9db2dbceaa5f548b`）
- **轨道目标**：D-Track V1-Rust — Bug 5 (C4) `src-tauri/src/lib.rs` 窗口关闭逻辑与 Tauri Shell 编译/单测/构建验证

---

## 1. 缺失工具链安装

容器初始缺少 `pnpm`、`rustc`、`cargo` 及 Tauri Linux 编译链与系统依赖，已全部补齐：

1. **pnpm**：`npm install -g pnpm@12.21.0` -> `12.21.0`
2. **Rust 工具链**：通过 `rustup` 安装 stable toolchain -> `rustc 1.100.0 (0ef1bc386 2026-11-09)`、`cargo 1.100.0 (3acc79c0c 2026-11-05)`（满足 `src-tauri/Cargo.toml` 的 `edition = "2024"` 与 `rust-version = "1.90"`）
3. **系统编译与 Tauri 依赖**：通过 `apt-get install -y` 安装 `build-essential` (`gcc`/`cc`)、`pkg-config`、`libssl-dev`、`libgtk-3-dev`、`libwebkit2gtk-4.1-dev`、`libayatana-appindicator3-dev`、`librsvg2-dev`、`patchelf`

---

## 2. D-Track Rust 编译与构建验证

（使用 `CARGO_TARGET_DIR=/tmp/rc-tauri-target` 执行，工作区保持无未跟踪构建产物）

| 验证项 | 命令 | 结果 | 详情 |
|---|---|---|---|
| Rust 全目标检查 | `cargo check --all-targets` | **PASS** (`EXIT_CODE=0`) | 0 errors；仅存在 2 条非 Linux 平台相关的 `unused_variables` 警告（`src/window_manager.rs:151` `update_macos_activation_policy` 在 `#[cfg(target_os = "macos")]` 外未加 `_` 前缀） |
| Rust 单元测试 | `cargo test` | **PASS** (`EXIT_CODE=0`) | `2 passed; 0 failed`（`tunnel_manager::tests::peer_routing_table_multi_peer_upsert_and_remove`、`tunnel_manager::tests::bridge_frame_roundtrip_and_udp_loopback`） |
| Tauri 完整构建 | `pnpm exec tauri build --debug --no-bundle` | **PASS** (`EXIT_CODE=0`) | Vite 前端构建 28.70s + Rust 链接构建 37.23s，生成可执行文件 `/tmp/rc-tauri-target/debug/rc_tauri_shell` |

### `src-tauri/src/lib.rs` (Bug 5 / C4) 代码核验

- `on_window_event`（L70–76）：仅监听 `WindowEvent::Destroyed` 以更新 macOS activation policy，不再拦截 `WindowEvent::CloseRequested` 隐藏窗口。
- `RunEvent::ExitRequested`（L153–163）：仅在 `code.is_none()` 且 `tray_only`（`should_start_tray_only_from_env() || state.config.tray_only_mode`）为 `true` 时调用 `api.prevent_exit()` 保留托盘；正常模式下允许应用退出。
- `RunEvent::Exit`（L164–166）：统一调用 `cleanup_all_tunnels(app_handle)` 清理隧道资源。

---

## 3. 全量门禁复核（`4ec70dc`）

| 门禁命令 | 结果 | 关键指标 |
|---|---|---|
| `pnpm run check` | **PASS (GREEN)** | `1429/1429` TS 文件（100.0% TS），`tsc --noEmit` clean，`NameTagScatter=1/1`，`BoolHotspots=7`，`UnscopedCaches=0` |
| `pnpm run radar:audit` | **PASS (GREEN)** | 硬边界、防假拆分、架构债务棘轮全部通过 |
| `pnpm run test:multiplayer-suite` | **PASS** | `26/26` 全部通过（含 Bug 3 / B-Track 与 Bug 4 / C-Track 测试） |

`git status --short` 为空，主仓库无任何改动或残留文件。
