# RocketCompanion MVP-1 实施交付报告

## 分支与提交信息
- **新分支**：`feat/mvp1-rust-core-split-v2`（基于 `1f9ca7dc9348ad7617c3b1f579519bc6e54c1a67`）
- **Verified Bot Commit**：[`0deab37c9b218cdf43bd127f673d22d6cd2e2a52`](https://github.com/Antigravi-ty/RocketCompanion/commit/0deab37c9b218cdf43bd127f673d22d6cd2e2a52)（`VALID` 签名）
- **前置计划备份**：[`623e96030f7778f3a7e0d3de2e56bad7a701de95`](https://github.com/AgenteamCN/Outputs/commit/623e96030f7778f3a7e0d3de2e56bad7a701de95)

---

## 本次完成内容

1. **环境脚本与 OverlayFS 预算修复 (`<= 100 MB`)**
   - 修改 `scripts/env/bootstrap.sh`：将 Linux Tauri 桌面依赖（`libwebkit2gtk-4.1-dev`、`libgtk-3-dev` 等）隔离到 `--with-tauri`；默认 `--with-rust` 仅将 `rustup`、`cargo` 与 `wasm32-unknown-unknown` 安装至 `/tmp/rc-cache`（tmpfs）。
   - 实测 `node scripts/env/checkEnv.mjs --quick`：`OverlayFS size: 30 MB (baseline: 30 MB, budget: <= 100 MB)`。

2. **Rust 双 Crate 架构拆分 (`rc_core` + `rc_core_wasm`)**
   - `crates/rc_core`（`rlib`）：纯 Rust 核心库，导出 `SimCore` 实例与 `CarControls`，去除全局静态单例与 WASM 裸指针胶水，方便 Linux 并发 `cargo test` 及后续 Tauri Native / Dedicated Server 直接复用。
   - `crates/rc_core_wasm`（`cdylib`）：极薄 WASM C ABI 外壳，更新 `crates/Cargo.toml` 与 `scripts/buildRcCoreWasm.mjs`，重新编译生成 `src/mvp1/rc_core.wasm`。
   - `crates/rocketsim` 保持 100% 原汁原味、零魔改。

3. **Linux `cargo test` 空中翻滚 & Stall（无效翻滚）验证**
   - 在 `crates/rc_core/src/lib.rs` 新增 `aerial_directional_flips_and_stalls_work_in_rocketsim`，加载 `public/arenamesh/mesh_*.cmf` 后验证：
     - `W / S / A / D / Q / E` 空中翻滚均正常触发 `has_flipped` 与对应方向加速度；
     - `Q + D + Jump` 与 `A + E + Jump` 均正确触发 RocketSim Stall（`has_flipped = true` 且水平速度增量 `< 50 uu/s`）。

4. **MVP-1 按键绑定修正**
   - 修改 `src/mvp1/InputController.ts` 与 `mvp1.html`：
     - `W / S / A / D / Q / E`：油门/俯仰/转向/偏航/左右滚转（移除 `ArrowLeft`/`ArrowRight` 对转向的误绑）
     - `ShiftLeft`：Power Slide（Handbrake）
     - `Space`：Car / Ball Camera 切换
     - `ArrowLeft` & `Period (.)`：Boost
     - `ArrowRight` & `Slash (/)`：Jump
     - `R`：Reset Kickoff，`T / F10`：Export Telemetry

5. **真实场地 / 球模型与 Boost Pad Hitbox 绘制**
   - 修改 `src/mvp1/ArenaRenderer.ts`：
     - 接入 `loadDriftwoodsStadium()` 加载真实球场模型（保留线框场地兜底）；
     - 接入 `loadBallAsset()` 加载经典球模型（保留几何体兜底）；
     - 依据 `RenderFrame` 的 64 位 `boost_pads_lo / boost_pads_hi` 掩码，在 34 个 RocketSim Soccar 精确坐标上绘制大包（半径 `208`）/ 小包（半径 `144`）Hitbox 圆柱指示器，实时反映刷新/冷却状态。

6. **`http://localhost:3000` 临时快捷入口**
   - 在 `src/main.ts` 非 `?mvp1=1` 页面右下角注入临时悬浮按钮 `MVP-1 Rust Testbed →`，点击即可一键跳转 `?mvp1=1`。

---

## 本地启动与验证命令

### 1. 切换分支并启动（Mac / 本地开发环境）
```bash
git fetch origin feat/mvp1-rust-core-split-v2
git checkout feat/mvp1-rust-core-split-v2
pnpm install
pnpm run tauri:dev:tray
```
或者直接用网页开发服务器启动：
```bash
pnpm run dev
```
打开 `http://localhost:3000`，点击右下角 **`MVP-1 Rust Testbed →`**（或直接访问 `http://localhost:3000/?mvp1=1`）。

### 2. 已通过的自动化验证
- `cargo test --manifest-path crates/Cargo.toml -p rc_core`：**4 passed, 0 failed**
- `node scripts/buildRcCoreWasm.mjs`：**成功产出 `src/mvp1/rc_core.wasm`**
- `pnpm exec tsc --noEmit`：**0 errors**
- `node scripts/env/checkEnv.mjs --quick`：**OverlayFS 30 MB (`<= 100 MB`)**
