# RocketCompanion MVP-1 实施完成报告

- **分支**：`feat/mvp1-rust-core-render-split-7e3a9c`（基于干净 `alpha` 建立，未引入 `integration/vnext-core` 状态补丁）
- **Verified Bot Commit**：`1f9ca7dc9348ad7617c3b1f579519bc6e54c1a67`（`verified: true`）

## 1. 核心架构落地

1. **纯净 `crates/rocketsim` + 自研 `crates/rc_core`（WASM）**
   - `crates/rocketsim` 保持与上游一致（`7ee65828fe1c6efdeb22cf77e9f5df2ff7238971`），零修改。
   - 新建 `crates/rc_core`（`crates/rc_core/src/lib.rs`、`camera_kernel.rs`、`coords.rs`、`render_frame.rs`、`telemetry.rs`）：
     - 120Hz 固定步进一车（Octane）一球物理仿真，通过 `/arenamesh/manifest.json` + 16 个 `.cmf` 分片初始化场地碰撞体。
     - 将 `CleanPovCameraKernel` 完整迁移至 Rust（`CleanPovCameraKernel` in `camera_kernel.rs`），支持 Car Cam / Ball Cam 平滑二次方过渡、地面/空中混合、速度与超音速 FOV 动态变化。
     - 仅在 `coords.rs` 导出边界做一次 UE3 Z-up -> Three.js Y-up 坐标与四元数转换。
     - 输出定长 140 字节二进制 `RenderFrame`（含 `magic`、`tick`、`teleport_seq`、`flags`、球/车/Camera 位姿与速度、Boost 与 Boost Pad 位掩码）。
     - 内置 240 tick（2 秒）`TelemetryRingBuffer` 黑匣子环形缓冲，支持一键导出 JSON 遥测快照。

2. **Web Worker 与统一输入（TS -> Worker）**
   - `src/mvp1/SimCoreWorker.ts`：在 Dedicated Web Worker 中加载 `src/mvp1/rc_core.wasm`，以 120Hz 固定步进，每次从 WASM 线性内存拷贝切片出 `ArrayBuffer` 并 `Transfer` 至主线程，避免 `memory.grow` 导致的视图失效。
   - `src/mvp1/InputController.ts`：基于 `KeyboardEvent.code`（`KeyW/A/S/D/Q/E`、`ShiftLeft`、`Space`、`Period`、`Slash`、`ArrowLeft/Right`、`KeyR`、`KeyT`）输出标准 `CarControlInput`。

3. **Pure Thin Renderer + 同屏双画布对比验证**
   - `src/mvp1/FrameSource.ts`：`BufferedFrameSource` 实现固定 2-tick 缓冲插值（`lerp` / `slerp`），不退化为 passthrough；检测 `teleportSeq` 跳变时强制硬切位姿并触发瞬移信号。
   - `src/mvp1/ArenaRenderer.ts`：纯哑巴 Three.js 渲染器，加载 `/assets/vehicles/Octane.glb` 与 geodesic ball，支持 `SpeedTrail` 丝带并在 `teleported` 时截断重置，杜绝拉丝。
   - `src/mvp1/Mvp1App.ts` + `mvp1.html`（亦可通过 `/?mvp1=1` 访问）：同屏实例化两个 `ArenaRenderer`：
     - **左屏**：120Hz Direct。
     - **右屏**：刻意降频为 60Hz 并注入 5~25ms 随机抖动。

## 2. 自动化验证结果

- **Rust 单元测试**（`cargo test -p rc_core`）：3/3 通过（`render_frame_is_140_bytes`、`camera_kernel_produces_finite_poses_and_transitions`、`telemetry_ring_buffer_wraps_cleanly`）。
- **WASM 构建**（`pnpm run build:rc-core`）：`wasm32-unknown-unknown` 编译成功，产出 `src/mvp1/rc_core.wasm`（463 KB）。
- **架构门禁与类型检查**（`pnpm run check`）：`radar` 1397/1397 TS 100% PASS，`tsc --noEmit` PASS。
- **Headless Chrome WebGL2 E2E 验证**（`http://127.0.0.1:3003/mvp1.html`）：
  - `CONSOLE_ERRORS: []`（零报错）。
  - 模拟 `KeyW` + `ShiftLeft` 加速：`speedUu` 从 `0` 增至 `853`，`boost` 从 `33` 消耗至 `20`，`camFov` 从 `77.6` 增至 `79.5`。
  - 触发 `toggleBallCam()` 与 `resetKickoff()`：`ballCam` 切换为 `false`，`teleportSeq` 从 `1` 递增至 `2`，`boost` 复位为 `33`。
  - 双画布均正常渲染（`leftRenderCount: 7`, `rightRenderCount: 4`），遥测导出返回 128 条采样。
- **OverlayFS 状态**（`pnpm run env:check`）：`30 MB`（预算 `<= 100 MB`）。
