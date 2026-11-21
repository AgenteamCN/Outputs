# RocketCompanion MVP-1 架构收敛与实施计划

## 1. 基线选择与共识
- **主底座**：基于 `1f9ca7dc9348ad7617c3b1f579519bc6e54c1a67`（`feat/mvp1-rust-core-render-split-7e3a9c`，从干净 `alpha` 分叉），拉出新分支 `feat/mvp1-rust-core-split-v2`。
- **上游隔离**：保持 `crates/rocketsim` 100% 原汁原味、零魔改，确保后续可低冲突同步 `ZealanL/RocketSim` 上游更新。

## 2. 核心实施清单
1. **环境与 OverlayFS 预算修复 (`<= 100 MB`)**
   - 修正 `scripts/env/bootstrap.sh`：将 `libwebkit2gtk-4.1-dev` / `libgtk-3-dev` 等 Tauri Linux 桌面依赖从 `--with-rust` 中剥离，仅在显式传入 `--with-tauri` 时安装；`--with-rust` 仅将 `rustup`、`cargo` 与 `wasm32-unknown-unknown` 工具链安装至 `/tmp/rc-cache`（0 MB OverlayFS 增量）。
   - 在 `scripts/env/checkEnv.mjs` 中补充提示/检查，避免误用 `apt-get` 膨胀 OverlayFS。
2. **Rust 双 Crate 架构拆分 (`rc_core` + `rc_core_wasm`)**
   - `crates/rc_core`：纯 Rust 核心库（`rlib`），封装 `SimCore` 实例（物理步进、120Hz `CleanPovCameraKernel`、坐标转换、`RenderFrame`、`TelemetryRingBuffer`），去除全局静态单例与 WASM 裸指针胶水，支持 Linux 下原生多线程 `cargo test` 及未来 Tauri Native / Dedicated Server 直接复用。
   - `crates/rc_core_wasm`：极薄 `cdylib` 外壳，负责将 `SimCore` 导出为浏览器 Worker 调用的 C ABI WASM 函数，更新 `scripts/buildRcCoreWasm.mjs` 与 `crates/Cargo.toml`。
3. **`http://localhost:3000` 临时快捷入口**
   - 在主入口页面添加一个显眼的临时快捷按钮，一键跳转至 `?mvp1=1` 测试台。
4. **MVP-1 渲染与场景完善（最小心智负担）**
   - 接入 `alpha` 现有的真实球场、Octane 车身、球 `.glb` 模型加载（保留几何体兜底防加载失败黑屏）。
   - 确保球正常生成与同步，调整 Car / Ball Camera 参数与对齐。
   - 借鉴轻量 Boost Pad 绘制逻辑：利用 `RenderFrame` 已输出的 64 位 `boost_pads_lo / boost_pads_hi` 掩码，在 34 个精确场地坐标上绘制高亮/变暗 Hitbox 指示器（大包/小包区分，无需复杂 `.glb` 绑定）。
5. **键盘绑定修正**
   - `W / S / A / D / Q / E`：油门/俯仰/转向/偏航/左右滚转（移除 `ArrowLeft`/`ArrowRight` 对转向的误绑）
   - `ShiftLeft`：Power Slide（Handbrake）
   - `Space`：Car / Ball Camera 切换
   - `ArrowLeft` & `Period (.)`：Boost
   - `ArrowRight` & `Slash (/)`：Jump
6. **Linux `cargo test` 空中翻滚 & Stall（无效翻滚）验证**
   - 在 `crates/rc_core` 中新增单元测试，验证空中 `W/S/A/D/Q/E` 翻滚，以及同时按下 `Q + D + Jump`、`A + E + Jump` 时能否正确触发 RocketSim 的 Stall（`has_flipped` 触发、垂直高度保持且无水平加速）。

## 3. 验证与交付
- 运行 `cargo test --manifest-path crates/Cargo.toml -p rc_core`、`node scripts/buildRcCoreWasm.mjs`、`pnpm run build`（或 `tsc --noEmit`）以及 Headless Chrome `?mvp1=1` 冒烟验证。
- 使用 GitHub Verified Bot Commit 提交至新分支，并将最终交付报告 POST 至备份网关。
