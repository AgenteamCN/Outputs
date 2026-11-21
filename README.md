# OverlayFS 547.77 MB 膨胀原因与 `env:help` 排查报告

## 1. 现象与实测数据

- **沙箱整体磁盘（OverlayFS `/`）**：`df -h` 显示 `/` 已用 **566 MB**（对应面板显示的 `547.77 MB / 16,657 files`）。
- **仓库目录（`/workspace`）**：`du -sh /workspace` 为 **30 MB**（相比干净基线 28 MB 仅增加 2 MB）。
- **系统目录**：`/usr` 占 3.6 GB（含基础镜像 + 新增写入），`/var` 占 76 MB。

## 2. 根本原因分析

1. **为什么 OverlayFS 变成了 547.77 MB？**
   - 执行 `bash scripts/env/bootstrap.sh --with-rust` 时，脚本第 6 步检测到缺少 `webkit2gtk-4.1` 和 `gcc`，自动运行了：
     `apt-get install -y --no-install-recommends gcc libc6-dev pkg-config libwebkit2gtk-4.1-dev libgtk-3-dev librsvg2-dev libssl-dev`
   - 该命令拉取并解压了 **209 个 Ubuntu `.deb` 包**（包括 `libwebkit2gtk-4.1-dev`、`libgtk-3-dev`、`gstreamer1.0-plugins-good`、`libicu-dev`、`gcc-13`、`binutils` 等），全部写入 `/usr` 与 `/var/lib/dpkg`，直接向 OverlayFS `/` 增加了约 **536 MB / 16,000+ 个文件**。
   - 相比之下，`node_modules`、`public/`、`pnpm-store`、`rustup-home`、`cargo-home`、`cargo-target` 均已正确放在 `/tmp/rc-cache`（tmpfs），未占用 OverlayFS。
   - **架构区分**：仅编译 `rc_core` WASM（`wasm32-unknown-unknown`）只需要 `rustup` + `cargo`（放在 `/tmp`），跑宿主机 `cargo test` 只需要最小 `gcc` + `libc6-dev`，并不需要 400+ MB 的 `libwebkit2gtk-4.1-dev` / `libgtk-3-dev`（Tauri 桌面端依赖）。

2. **为什么 `pnpm run env:check` 之前显示 `OverlayFS size: 30 MB`？**
   - `scripts/env/checkEnv.mjs` 调用 `tmpfsGuard.mjs` 仅统计 `/workspace` 目录的体积（30 MB），未统计 `apt-get install` 写入 `/usr` 和 `/var` 的 OverlayFS 增量，因此显示为 `30 MB (baseline: 28 MB, budget: <= 100 MB)`。

3. **`pnpm run env:help` 是否可用？**
   - 原版 `alpha` 分支确实没有 `scripts/env/*` 与 `env:*` 脚本（它们原本只在 `integration/vnext-core` 上）。
   - **但我们在建立新分支 `feat/mvp1-rust-core-render-split-7e3a9c` 时，已将 `scripts/env/*` 与 `env:help` / `env:check` / `env:doctor` / `env:setup` / `env:setup:rust` 一并合入 `package.json`**。
   - 实测在 `/workspace/RocketCompanion` 目录下运行 `pnpm run env:help` 可正常输出帮助信息（在 `/` 根目录直接跑 `pnpm` 会因找不到 `package.json` 而报错）。

## 3. 后续优化建议

- 若后续仅做 WASM（`crates/rc_core`）与 Web 端开发，可在 `bootstrap.sh` 中将 `--with-rust` 拆分为 `--with-rust-wasm`（仅装 `rustup`/`cargo` 到 tmpfs，跳过 `webkit2gtk`/`gtk3` apt 安装）与 `--with-tauri`（需要桌面端打包时再装 GTK/WebKitGTK 依赖），即可将 OverlayFS 严格控制在 <= 100 MB 预算内。
- 若需修复当前环境快照的 547 MB 占用，需要从干净容器快照重建环境并仅执行 `--with-rust-wasm`（或 `bootstrap.sh` 不带 `--with-rust` + 手动将 `rustup` 装到 `/tmp/rc-cache`）。
