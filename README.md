# 修复报告：纯键盘 W + Jump 无法触发 Front Dodge 及 CarConfigC 内存偏移错位

## 分支与提交信息
- **基准 Commit**: `52b81739fc648597bc5e7368e6b122aa32fe6ad5`
- **工作分支**: `z/agy/rc/rust/fix-keyboard-dodge-air-steer`
- **Verified Commit**: `eb68dfcd0f2caa7a85af97f301626296afba4a37` (`signature.isValid: true, state: "VALID"`)
- **修改文件**: [`src/physics/RocketSimVehicleAndArenaSpecs.ts`](file:///work/RocketCompanion/src/physics/RocketSimVehicleAndArenaSpecs.ts)

## 根因定位
1. **为什么之前单测/WASM 直调能过，但浏览器实战完全没有翻滚**：
   - 在 Rust 端 `crates/rc_physics_sim/src/layout.rs` 中，`#[repr(C)] CarConfigC` 的 `[f32; 3]` 为 **12 字节（3 个 float）**，而非旧版 C++ RocketSim `Vec` 的 16 字节对齐（4 个 float）。
   - `CarConfigC` 的实际 float 索引布局为：
     - `0..2`: `hitbox_size [f32; 3]`
     - `3..5`: `hitbox_pos_offset [f32; 3]`
     - `6`: `front_wheels.wheel_radius`
     - `7`: `front_wheels.suspension_rest_length`
     - `8..10`: `front_wheels.connection_point_offset [f32; 3]`
     - `11`: `back_wheels.wheel_radius`
     - `12`: `back_wheels.suspension_rest_length`
     - `13..15`: `back_wheels.connection_point_offset [f32; 3]`
     - `16`: `three_wheels + _pad`
     - `17`: `dodge_deadzone`
   - `loadRocketSimWasmModule()` 初始化时调用了 `applyCalibratedCarSuspensionConfigsToWasm()`，该函数仍使用旧版 C++ 偏移 `fIdx + 9` 和 `fIdx + 17`：
     - `fIdx + 9` 写坏了 `front_wheels.connection_point_offset.y`（25.9 → ~39.24）；
     - `fIdx + 17` 直接把 `dodge_deadzone`（默认 `0.5`）覆盖成了后轮悬架长度（约 `36.74`）；
     - 且真正的 `front_wheels.suspension_rest_length`（7）和 `back_wheels.suspension_rest_length`（12）从未被写入。
   - 由于 `|pitch| + |yaw| + |roll| <= 3.0 < 36.74`，所有车辆的二段跳都会判定为 `is_flip_input = false`，导致纯键盘（以及手柄）完全无法触发任何方向翻滚（Front/Back/Side/Diagonal Dodge），全部退化为 Double Jump。
   - 绕过 `loadRocketSimWasmModule()` 直接加载 `core.wasm` 的测试不会触发 `applyCalibratedCarSuspensionConfigsToWasm()`，因此未能暴露该问题。

## 修复内容
- 在 [`src/physics/RocketSimVehicleAndArenaSpecs.ts`](file:///work/RocketCompanion/src/physics/RocketSimVehicleAndArenaSpecs.ts) 中新增并导出 `CAR_CONFIG_C_OFFSETS`，将 `applyCalibratedCarSuspensionConfigsToWasm()` 的写入偏移修正为：
  - `CAR_CONFIG_C_OFFSETS.FRONT_SUS_REST` (`7`)
  - `CAR_CONFIG_C_OFFSETS.BACK_SUS_REST` (`12`)
- 保留 `front_wheels.connection_point_offset` 与 `dodge_deadzone = 0.5` 不被覆盖，并让 `step_and_events.rs` 中的悬架配置同步逻辑恢复正常工作。

## 验证结果
1. **浏览器端真实键盘 & 随机状态矩阵验证（Headless Chrome + Vite + `core.wasm`）**：
   - 检查 7 种车型配置（Hitbox 0..6）：`dodgeDeadzone` 均为 `0.5`，`frontOffY` 未被破坏，`frontSus` / `backSus` 与校准值一致。
   - 真实 `KeyboardEvent`（`KeyW` + `Slash` single jump → 松手 → `Slash` double jump）：修复前 `velZ` 上升且 `isFlipping: false`（Double Jump）；修复后 `velY` 获得前向加速脉冲、`angVel` 产生前翻角速度且 `isFlipping: true`（Front Dodge）。
   - 随机初始速度/朝向下的 10 组键盘输入矩阵（`none`、`W`、`S`、`A`、`D`、`Q`、`E`、`W+A`、`W+D`、`S+A`）及空中 `air steering`（`W/S/A/D/Q/E`）全部通过（`ALL PASSED: true`）。
2. **仓库工程检查**：
   - `pnpm check` (`radar` + `tsc --noEmit`): PASS (GREEN)
   - `pnpm build:wasm:check` (`cargo check --target wasm32-unknown-unknown --workspace`): PASS
