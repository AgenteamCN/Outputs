# 音频触发频率 & 车辆引擎音效衔接修复报告

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **基准分支**: `origin/z/agy/rc/rust/wave-1-4-octopus` (`84f20d2`)
- **工作分支**: `z/agy/rc/rust/fix-audio-trigger-and-engine`
- **Verified Bot Commit**: `d43b36fbddd8463f5eeaa0e746472f063e6a5bf2` (`signature: VALID`)

---

## 一、问题根因定位

### 1. 球体平滚 / 擦墙音频触发频率爆炸（"已经烂完了 / 耳朵受不了"）
通过 WASM 物理引擎（`core.wasm`）实测平滚（`pos=(0,0,93.15), vel=(900,0,0)`）与调用链排查，定位到两个叠加问题：
1. **TypeScript 旁路了 Rust `rc_audio_director` 的过滤与冷却**：
   - RocketSim 在球平滚或贴墙滑动时，每帧都会向 `PhysicsEventBuffer` 写入 `BALL_WORLD_HIT (0x04)`（120 ticks 内产生 64 次 `normalRelVel=0.00, dirDot=0.000` 的事件）。
   - Rust 侧 `rc_audio_director`（`director_impact_events.rs`）正确过滤掉了这些零法向速度/切向滑动事件（`wasmOneShotBallWorld=0`）。
   - **但** `src/game/pipeline/stages/CarActionEventSynthesizer.ts` 仍在 `physics.readEvents(...)` 中调用 `gameAudio.handleEvent(event, camera)`，而 `GameAudioEngineFacade.incidentEngine.handleEvent` 没有任何 `normalRelVel` 阈值、`dirDot` 过滤、冷却时间和空间衰减，导致直接以满音量 ~64~120Hz 疯狂触发 `groundHit[0]`！
2. **球体连续滚动 / 下落循环音源被置空（Stub）**：
   - `GameAudioEngineFacade.ts` 中的 `this.ballEngine` 被替换为了空壳 stub（`_startLoops: () => {}`，`rollingGain: null`，`ballPanner: null`），导致 `WasmAudioDirectorBridge.ts` 的 `applyWasmBallCueToEmitter` 无法驱动任何 WebAudio 节点，球平滚与空中下落没有连续音效。

### 2. 车辆引擎音频不连续（切换切片/换挡有断层与咔哒声）
对比旧版内核（`origin/alpha`）的 `TurtleModemGranularVoiceGraph.ts` 与 `TurtleModemEngineSynth.ts`，发现 `rust` 分支重写时丢失了关键的 WebAudio 参数平滑与去相关机制：
1. **`TurtleModemGranularVoiceGraph.ts` 全部改为硬跳变写入**：
   - 移除了 `applySmoothedAudioParamValue` / `stepExponentialSmoothedValue`，导致 `onGain.gain`、`offGain.gain`、`playbackRate`、`idleGain.gain`、`lowpassNode.frequency` 每帧直接阶跃赋值。
   - 当 RPM 跨过谐波切片边界（如 slice 19 -> 20，2990 RPM -> 4520 RPM）时，旧切片被瞬间置 `0`，新切片瞬间跳变，产生明显的爆音与不连续感。
   - `delayNode.delayTime` 完全未被更新（恒为 `0`），导致 `lowerSlice` 与 `upperSlice` 交叉淡入淡出时发生相位抵消 / 梳状滤波。
2. **`TurtleModemEngineSynth.ts` 主输出增益未平滑**：
   - `outputNode.gain.value = targetGain` 直接阶跃写入，启停/切换状态时产生瞬态跳变。

---

## 二、修复内容

1. **禁止 TS 旁路重放 WASM 音频事件**（`src/game/pipeline/stages/CarActionEventSynthesizer.ts`）
   - 当 `physics?.module?._physics_stepAudioDirector` 可用时，不再调用 `gameAudio.handleEvent(event, camera)`，统一由 Rust `rc_audio_director` 进行阈值/法向点积/冷却/空间衰减裁剪后通过 `dispatchWasmOneShotSlots` 播放。
2. **恢复球体连续音频 WebAudio 图**（新增 `src/audio/BallAudioEmitter.ts`，接入 `src/audio/GameAudioEngineFacade.ts`）
   - 恢复 `BallAudioEmitter`（3D `PannerNode`、`rollingGain`/`flyingGain`、`rollingSource`/`flyingSource`），由 WASM `applyWasmBallCueToEmitter` 零拷贝驱动。
3. **加固 Rust 侧球-墙碰撞防抖与切向剔除**（`crates/rc_audio_director/src/director_impact_events.rs`）
   - 将 `BALL_WORLD_HIT` / `BALL_GOALPOST_HIT` 最小冷却提高至 `>= 80ms`（`>= 10 ticks @ 120Hz`），并将切向滑动剔除阈值 `dir_dot` 提高至 `0.18`，防止曲面坡道接缝微弹跳引发高频碎响。
4. **恢复颗粒合成引擎音频平滑与相位去相关**（`src/audio/TurtleModemGranularVoiceGraph.ts`、`src/audio/TurtleModemEngineSynth.ts`）
   - 在 `TurtleModemGranularVoiceGraph.applyWasmCue` 中恢复 `applySmoothedAudioParamValue` / `stepExponentialSmoothedValue`：平滑 `lowpassNode.frequency`、`idleSource.playbackRate`、`idleGain.gain`、各切片 `onGain`/`offGain`（遵循 `cfg.gainSmoothingTimeMs`）、`playbackRate`（`15ms`）及 `delayNode.delayTime`（`40ms`，应用 `cfg.decorrelationDelayMs`）。
   - 修复切片离开激活区间时跳过平滑直接归零的问题（保留 `smoothedOnGain`/`smoothedOffGain > GAIN_DEADBAND` 期间继续平滑衰减至 0）。
   - 在 `TurtleModemEngineSynth.applyWasmCarCue` 中平滑 `outputNode.gain` 并传递帧 `dtSec` 给 `voiceGraph.applyWasmCue`。

---

## 三、验证结果

- `pnpm run build:wasm:dev`：WASM 编译通过（`core.wasm` 1160.2 KB, 198 exports）。
- `pnpm run build:wasm:check`：Rust workspace (`wasm32-unknown-unknown`) 检查通过。
- `pnpm run check`（`radar` + `tsc --noEmit`）：`STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified.`
- `pnpm run build`：Vite 生产构建通过。
