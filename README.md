# RocketCompanion — 高转速引擎音频噼啪声 & 翻滚/Flip Reset 缺失修复报告

- **仓库**: `Antigravi-ty/RocketCompanion`
- **工作分支**: `z/agy/rc/rust/audio-flip-reset-fix`（基于 `origin/z/agy/rc/rust/fix-audio-trigger-and-engine` `d43b36f`）
- **已验证提交 (Verified Bot Commit)**: [`52b81739fc648597bc5e7368e6b122aa32fe6ad5`](https://github.com/Antigravi-ty/RocketCompanion/commit/52b81739fc648597bc5e7368e6b122aa32fe6ad5)（`signature.isValid: true`，`Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

---

## 一、问题根因与修复详情

### 1. 高转速引擎音效“噼里啪啦 / 滋啦滋啦”爆音与毛刺

**根因分析：**
1. **50 Hz 限速器锯齿震荡 + 过激空转响应**：`rev_limiter_bounce_duration = 0.02s`（50 Hz 锯齿调制）配合 `air_rev` 飞轮敏捷度 `55.0`，导致空转或高转速时 RPM 在限速区间高频抖动。
2. **高转速切片锚点过密**：`granular_lookup.rs` 在 `5766 ~ 5873 RPM` 区间存在间隔仅 `7 RPM` 的密集切片（且相邻切片 `auto_norm_db` 存在 `2.3 dB` 跳变），微小的 RPM 抖动就会在 6 个缓冲区间剧烈切换。
3. **跨切片叠加削波 (Clipping)**：`auto_norm_db`、`eq_db`、`freq_tilt_db` 与等功率正弦交叉淡入淡出（+3 dB）叠加后超出 `0 dBFS`。
4. **WebAudio 块边界拉链噪声 & 动态延迟多普勒爆音**：`TurtleModemGranularVoiceGraph` 原先每帧直接写入 `AudioParam.value`（产生 128-sample 块边界台阶拉链噪声），且在主/次切片切换时动态调制 `DelayNode.delayTime` 引发多普勒音高抖动与梳状滤波噼啪声；同时周期循环缓冲区的淡入淡出窗口过短。

**修复改动：**
- `crates/rc_audio_director/src/config_pod.rs` & `src/ui-core/config/engineAudioConfig.ts`：将 `rev_limiter_bounce_drop` 调整为 `140.0`，`rev_limiter_bounce_duration` 从 `0.02s` 调整为 `0.075s`。
- `crates/rc_audio_director/src/drivetrain_state.rs`：将 `air_rev` 飞轮敏捷度限制在 `cfg.flywheel_agility.max(20.0).min(28.0)`。
- `crates/rc_audio_director/src/granular_lookup.rs`：
  - `total_db` 钳位至 `(-15.0, 4.0)`；
  - 强制主引擎切片锚点最小间隔 `>= 95.0 RPM`（跳过 `5766..5873` 过密簇）；
  - 为 `master` 增益引入 `0.78` 余量系数防止交叉淡入淡出削波。
- `src/audio/turtleModemBufferConditioning.ts` & `src/audio/turtleModemSynthAcousticPipeline.ts`：将 `conditionPeriodicLoopBuffer` 周期对齐交叉淡入淡出窗口扩展为 `Math.min(Math.floor(L / 4), P * 3)`，并将纯缓冲区 DSP 预处理逻辑拆分为独立模块 `turtleModemBufferConditioning.ts`（确保所有文件符合 Radar 体积上限）。
- `src/audio/audioParamSmoothingPipeline.ts`、`src/audio/TurtleModemGranularVoiceGraph.ts`、`src/audio/TurtleModemEngineSynth.ts`：
  - 新增并应用基于 `AudioParam.setTargetAtTime` 的 `applyAudioRateSmoothedParam`，在音频采样率级实现无拉链噪声平滑（覆盖 `lowpassNode.frequency`、`idleSource.playbackRate`、`idleGain.gain`、`onGain/offGain`、`onSource/offSource.playbackRate` 及 `outputNode.gain`）；
  - 移除活跃切片上的动态 `DelayNode.delayTime` 调制；
  - 切片从静音唤醒时立即对齐 `playbackRate`，避免从旧速率拉扯。

---

### 2. 翻滚 (Dodge `0x08`) 与 Flip Reset (`0x0F`) 缺失

**根因分析：**
1. **`car_modifiers.rs` Flip Reset 判定条件自相矛盾**：原逻辑要求 `!is_on_ground && !pre.is_on_ground && now_air_flip_avail`。在 RocketSim 中，当 3+ 个车轮接触球体或天花板时，`st.is_on_ground()` 为 `true`（接触结束离开后 `wheels_with_contact()` 又已清空），导致接触帧与离球帧均无法满足条件，`flip_reset_this_tick` 恒为 `false`。
2. **`state_sync.rs` 紧凑状态恢复未刷新接触与地面标志**：`restore_compact_state` 恢复状态后未调用 `arena.refresh_car_contact_state(car_id)`，且 `set_car_state_from_slice` 混用了恢复前的旧 `cs.is_on_ground()`，导致回放/跳转恢复后的首帧地面跳跃被误判为空中翻滚或丢失 Flip Reset 触发。
3. **事件路由断链**：
   - `rc_audio_director/src/director_vehicle_events.rs` 中 `handle_flip_reset_gained` 仅接收 `sub_type == 1`，忽略了 `flags` 中编码的对象类型；
   - `rc_audio_director/src/director_impact_events.rs` 缺少 `0x3E` (`sub_type 1..=3`) 到 `handle_car_jump_family` 的路由；
   - `src/audio/VehicleAudioManager.ts` 错误地匹配 `t === 0x05` 而非 `t === 0x3e`。

**修复改动：**
- `crates/rc_physics_sim/src/car_modifiers.rs`：检测 `is_on_ground && !pre.is_on_ground` 且车轮接触 `UserInfoType::Ball | UserInfoType::Car` 或满足天花板接触 (`st.pos.z > 1800.0 && st.rot_mat.z_axis.z < -0.35`)，结合前置已起跳/无空中翻滚条件，正确触发 `flip_reset_this_tick`。
- `crates/rc_physics_sim/src/state_sync.rs`：在 `restore_compact_state` 中使用快照 `car_arr[19]` (`is_on_ground`) 与 `car_arr[22]` (`has_flip_or_jump`) 精确恢复 `has_jumped`/`has_double_jumped`/`has_flipped`，并调用 `arena.refresh_car_contact_state(car_id)`。
- `crates/rc_audio_director/src/director_vehicle_events.rs`、`crates/rc_audio_director/src/director_impact_events.rs`、`src/audio/VehicleAudioManager.ts`：打通 `0x0F` 与 `0x3E` (`0x06`/`0x07`/`0x08`) 的音频事件路由与播放。

---

## 二、回放复现与构建验证

1. **回放复现（`goal-replay-frame-4184-idx-1309-2026-11-13T08-17-11.rcgr.gz`）**：
   - 窗口 `680..810`：
     - Frame `684` (tick `3559`) 成功触发物理事件 `0x0F` (`FLIP_RESET_GAINED`) 与音频事件 `0x0F` (`subType: 1, gain: 1, pitch: 1`)；
     - Frame `796` (tick `3671`) 成功触发物理事件 `0x08` (`CAR_DODGE`) 与音频事件 `0x08` (`subType: 3, gain: 1, pitch: 0.97`)。
   - 窗口 `1085..1115`：
     - Frame `1086` (tick `3961`) 成功触发 `0x06` (`CAR_JUMP`)；
     - Frame `1100` (tick `3975`) 成功触发 `0x08` (`CAR_DODGE`)。
   - 高转速空转限速测试（120..240 ticks）：切片平稳过渡（1s 内仅 5 次平滑切换，跳过密集簇）。
2. **代码检查与构建**：
   - `pnpm run check`：GREEN（`tsc --noEmit` 与 Radar 架构/体积检查全部通过）
   - `pnpm run build:wasm:check`：通过
   - `pnpm run build`：通过（Vite + WASM 生产构建成功）
