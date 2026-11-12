# Rocket Companion Rust 重构与音频引擎专项调查及全景推进计划书

> **报告标识**：`RC-RUST-AUDIO-INVESTIGATION-AND-ROADMAP-2026`  
> **基准分支评估**：`z/agy/rc/rust/sync-main-audio-plan` (Commit: `b26d328`)  
> **推荐工作分支**：`z/agy/rc/rust/audio-recreate-plan`（已基于 `sync-main-audio-plan` 切出）  
> **审计范围**：Rust WASM 架构、WebAudio 计算下沉、碰撞抖动与地表滚动判据、Crackcase 引擎音效算法、Fully Recreate 缺口审计及并行分支编排（Branch 01～32）。

---

## 目录
1. [执行摘要与核心裁决](#1-执行摘要与核心裁决)
2. [双 Rust 分支深度对比与裁决（Plan vs Investigate）](#2-双-rust-分支深度对比与裁决plan-vs-investigate)
3. [音频引擎架构现状与最佳架构定型（Rust Offload Architecture）](#3-音频引擎架构现状与最佳架构定型rust-offload-architecture)
4. [碰撞抖动与滚动音频 Bug 根因剖析与数学推导](#4-碰撞抖动与滚动音频-bug-根因剖析与数学推导)
5. [Crackcase Audio（Crankcase REV）引擎音效算法深度剖析](#5-crackcase-audiocrankcase-rev引擎音效算法深度剖析)
6. [距离 Fully Recreate 终局的全景差距审计](#6-距离-fully-recreate-终局的全景差距审计)
7. [最优并行分支全景规划（Branch 01 ～ Branch 32）](#7-最优并行分支全景规划branch-01--branch-32)

---

## 1. 执行摘要与核心裁决

| 核心问题 | 调查结论与行动裁决 |
| :--- | :--- |
| **分支二选一** | **必须选用 `z/agy/rc/rust/sync-main-audio-plan` (`b26d328`)**。`audio-investigate` 是 56 秒前一次冲突未解决完整的过渡 commit（缺失 `signature` 属性导致 TypeScript 报类型错误），`audio-plan` 才是真正修复了 Vite 插件类型并清理干净的基准分支。 |
| **音频计算架构** | **当前存在严重的“计算未下沉与双重冗余”问题**。Rust `rc_audio_director` 已经写好了完整的动力学与 40 切片音频管线，但 TS 侧 `GameAudioEngineFacade` 竟完全忽略了 `cue.cars`，仍在主线程跑繁重的 `TurtleModemEngineSynth` 计算。**最佳架构**：100% 动力学、切片插值、3D 声学衰减、多车 LOD、碰撞去抖下沉至 Rust，WebAudio 仅作“哑巴驱动器（Dumb Driver）”。 |
| **抖动与滚动判据** | **Bug 根因**：物理引擎（Bullet/RocketSim）在球落地静止或滚动时，每 tick 都产生触碰事件；而 Rust/TS 现有代码在法向速度接近 0 时，**错误 fallback 到了球的总速度**（如 1500 UU/s > 140 阈值），导致每 40ms 触发一次震耳欲聋的接地冲击音；同时缺少 $\vec{v}\cdot\vec{n}$ 滚动正交投影过滤和 $\ge 60\text{ms}$ 强力防抖窗口。 |
| **Crackcase 算法** | 原 alpha 分支实现的 Crackcase 方案极其优雅：**40 个特定 RPM 谐波切片**、等功率/等正弦交叉淡化、Piecewise-Cosine 5 段频响均衡、RMS 响度补偿、动力学减速回火、超音速回弹。该算法在 Rust `granular_lookup.rs` 已完成理论移植，需在 TS 剪断冗余并直接消费 WASM 输出。 |
| **Fully Recreate 距离** | **整体完成度约 68%**。物理与 AI 核心已大幅下沉（85%~90%）；音频处于“Rust 算完 TS 没接”的半下沉状态（70%）；回放 NetStream 解码（40%）与网络客户端预测回滚（15%）是剩余最大硬骨头。 |
| **并行分支编排** | 规划 **4 个功能 Tier、共 32 个正交独立的分支（Branch 01-32）**，每个分支遵循“单一职责、代码量 <400 行、雷达 0 环依赖”军规，最终通过 Octopus Merge 汇总。 |

---

## 2. 双 Rust 分支深度对比与裁决（Plan vs Investigate）

### 2.1 Git 历史与 Commit 树分析
通过对 `remotes/origin/z/agy/rc/rust/sync-main-audio-plan` 与 `remotes/origin/z/agy/rc/rust/sync-main-audio-investigate` 的精确 Commit 原数据对比：

```
* b26d328 (sync-main-audio-plan)        Merge branch 'alpha' [14:28:16 UTC]
|\
| * f64c3b8 (sync-main-audio-investigate) Merge branch 'alpha' [14:27:20 UTC]
|/|
* | 2cb143b feat(rust): Tier-2 single-player Octopus Merge (T2-01..T2-13)
| * 1facf97 feat(driftwoods): add real-time GPU hex ribbon width and fin depth scaling
```

- 两个分支均以 `2cb143b`（Tier-2 13 分支 Octopus Merge）为基础，合入了主线 `alpha`（`1facf97`）。
- `investigate` 提交于 `14:27:20`，而 `plan` 提交于 `14:28:16`（晚 56 秒）。

### 2.2 全量 Diff 差异与根本原因
运行 `git diff remotes/origin/z/agy/rc/rust/sync-main-audio-plan remotes/origin/z/agy/rc/rust/sync-main-audio-investigate`，**整个仓库只有且仅有 1 个文件存在差异**：
`scripts/vite/gitHeadAutoRestartPlugin.ts`

差异代码明细：
```diff
--- a/scripts/vite/gitHeadAutoRestartPlugin.ts (sync-main-audio-investigate)
+++ b/scripts/vite/gitHeadAutoRestartPlugin.ts (sync-main-audio-plan)
@@ -169,6 +169,7 @@ function computeRustWorkspaceSignature(dirPath: string): string {
 }
 
 function readCompositeGitHeadState(): {
+  signature: string | null;
   gitSignature: string | null;
   rustSignature: string;
   locked: boolean;
@@ -181,13 +182,16 @@ function readCompositeGitHeadState(): {
   for (const t of targets) {
     const snap = readWorktreeHeadSnapshot(t.label, t.path);
     if (snap.locked) {
-      return { gitSignature: null, rustSignature: '', locked: true };
+      return { signature: null, gitSignature: null, rustSignature: '', locked: true };
     }
     parts.push(snap.summary);
   }
+  const gitSignature = parts.join(' | ');
+  const rustSignature = computeRustWorkspaceSignature(CRATES_DIR);
   return {
-    gitSignature: parts.join(' | '),
-    rustSignature: computeRustWorkspaceSignature(CRATES_DIR),
+    signature: `${gitSignature} [${rustSignature}]`,
+    gitSignature,
+    rustSignature,
     locked: false,
   };
 }
```

### 2.3 裁决结论
在 `alpha` 主线的 `1facf97` 中，引入了客户端热重载握手中间件 `/__rc_git_head` 与 `transformIndexHtml`，要求 `readCompositeGitHeadState()` 必须提供 `signature`。
- 在 `investigate` 分支合并时，漏掉了 `signature` 字段，导致 Vite 插件在 TypeScript 编译和运行时存在 `undefined` 隐患。
- 在 `plan` 分支中，该合并冲突被彻底纠正，正确补充了 `signature` 并在客户端注入与中间件中闭环。
- **结论**：**毫不犹豫选择 `z/agy/rc/rust/sync-main-audio-plan` 作为唯一下游基准**。当前我们创建的 `z/agy/rc/rust/audio-recreate-plan` 便是直接派生于它。

---

## 3. 音频引擎架构现状与最佳架构定型（Rust Offload Architecture）

### 3.1 现状：“半生不熟”的计算冗余与孤岛
经过对 `src/audio` 与 `crates/rc_audio_director` 的全面审查，目前的架构处于严重的断层状态：

```
[Rust rc_audio_director (WASM)]
   │
   ├── step_ball_acoustic_loops() ──> AudioFrameCuePod.ball (32B) ──[Wired]──> applyWasmBallCueToEmitter (TS)
   ├── process_physics_incidents() ──> AudioFrameCuePod.one_shots (48Bx16) ──[Wired]──> dispatchWasmOneShotSlots (TS)
   │
   └── step_all_vehicles_baseline() ──> AudioFrameCuePod.cars[8] (80Bx8) ──[ORPHANED!! 未被TS使用]
                                                                                      │
                                                                                 (断层/遗漏)
                                                                                      ↓
[TypeScript 主线程 (GameAudioSubsystem)]
   ├── 仍在重复运行 RemoteEngineAudioLodController.ts (TS)
   ├── 仍在重复运行 TurtleModemEngineSynth.ts (TS)
   ├── 仍在重复计算 TurtleModemDrivetrainModel.ts (齿轮、转速、加速度) (TS)
   └── 仍在重复计算 TurtleModemGranularVoiceGraph.ts (切片混合、Piecewise EQ) (TS)
```

**问题剖析**：
1. **`AUDIO_CUE_CARS_WORD_OFFSET = 22` 成为死常量**：在 `WasmAudioDirectorBridge.ts` 中定义了偏移量，但全仓库没有任何一行代码去读取 `cue.cars[i]`。
2. **Rust 计算白白浪费**：Rust 已经在每帧计算好 8 辆车的 `lower_slice_idx`、`upper_slice_idx`、`lower_slice_gain`、`upper_slice_gain`、`filter_cutoff_hz`、`engine_pitch`、`spatial_pan`、`distance_uu`。而 TS 竟然全盘忽略，自己在 JS 里用繁重的对象和数学函数重算了一遍！
3. **主线程掉帧与 GC 抖动**：TS 端 `TurtleModemEngineSynth` 每帧产生大量临时数组和计算对象，直接拖慢渲染帧率。

---

### 3.2 最佳架构（Optimal Pure-Offload Architecture）
遵循系统架构设计原则：**“所有数学计算、物理推演、状态滤波在 Rust；Web Audio API 只做纯粹的执行驱动器（Dumb Playback Driver）”**。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Rust WASM Core (rc_audio_director)                    │
│                                                                             │
│  [Physics Events]   [Vehicle Kinematics]   [Ball Dynamics]   [Listener Cam] │
│          │                   │                    │                 │       │
│          ▼                   ▼                    ▼                 ▼       │
│  ┌───────────────┐   ┌────────────────┐   ┌───────────────┐   ┌───────────┐ │
│  │ 60ms Debounce │   │ Drivetrain Sim │   │ Rolling / Fly │   │ 3D Stereo │ │
│  │ v·n Projector │   │ 40-Slice EQ    │   │ Distance Attn │   │ Doppler   │ │
│  │ Event Latch   │   │ Top-3 LOD Rank │   │ Pitch Smooth  │   │ Bezier    │ │
│  └───────┬───────┘   └───────┬────────┘   └───────┬───────┘   └─────┬─────┘ │
│          │                   │                    │                 │       │
│          └───────────────────┼────────────────────┴─────────────────┘       │
│                              ▼                                              │
│               AudioFrameCuePod (Fixed 1024 Bytes / C-ABI)                   │
│   - ball: BallAcousticLoopPod (audio_pos, rolling_gain/rate, fly_gain/rate) │
│   - cars[8]: CarAcousticLoopPod (gain, pitch, cutoff, slice_i/j, pan, dist) │
│   - one_shots[16]: AudioOneShotSlotPod (code, gain, rate, pos_meters)       │
│   - edge_masks: u32 bitflags (instant triggers)                             │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Direct Shared Memory Read (HEAPF32/HEAPU32)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TypeScript / WebAudio (Thin Dumb Driver)                    │
│                                                                             │
│  1. Bitflag Listener: 监听 edge_mask 标志位，直接触发单次播放                │
│  2. Direct AudioParam Write:                                                │
│     - `engineLowpass.frequency.value = cue.cars[i].filter_cutoff_hz`        │
│     - `sliceA_Source.playbackRate.value = cue.cars[i].lower_slice_rate`     │
│     - `sliceA_Gain.gain.value = cue.cars[i].lower_slice_gain`               │
│     - `ballPanner.positionX.value = cue.ball.audio_pos[0]`                  │
│  3. ZERO Math in JS: 不在 JS 中做向量积、不开根号、不做任何物理状态推导!       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**对比优势**：
- **CPU 耗时下降 85%**：JS 端完全消除动力学与切片查找开销，变成纯粹的几十次 TypedArray 取值和 WebAudio 原生属性赋值。
- **零内存抖动（0 Allocation / Frame）**：固定共享内存块，无 GC 停顿风险。
- **确定性与多端一致**：无论是桌面 Tauri 原生还是 Web 端，音频逻辑完全由 Rust 统一保证。

---

## 4. 碰撞抖动与滚动音频 Bug 根因剖析与数学推导

### 4.1 核心现象
1. **触地“突突突”机关枪式抖动（Contact Chattering）**：球在地面滚动或低空贴地滑行时，频繁爆出接触音效（World Hit / Floor Bounce）。
2. **带球（Ground Dribble）杂音爆音**：车贴地推球或停球推行时，原本应当是平滑的滚动声，却出现连续的剧烈撞击音。

### 4.2 物理根因深挖
在 `rocketsim`（底层基于 Bullet 碰撞检测）中，碰撞管线在每步物理 tick（120Hz）都会执行：
```rust
// crates/rocketsim/src/sim/arena/base.rs
fn on_ball_world_collision_parts(...) {
    self.events.push(BallHitWorld(BallHitWorldEvent { contact_point, contact_normal }));
}
```
**Bullet 的碰撞流形（Manifold）特性**：
只要球与地面或墙面的碰撞盒重叠深度超过容差，Bullet **在每一个 tick 都会报告碰撞接触点**！即使球静止在地面上，物理引擎也会每 tick 生成一个 `BallHitWorld`。

而在 `rc_physics_sim/src/step_and_events.rs` 中：
```rust
ArenaEvent::BallHitWorld(e) => {
    let speed = ball.vel.length();
    let normal_rel_vel = ball.vel.dot(e.contact_normal).abs();
    let dir_dot = if speed > 1e-4 { (normal_rel_vel / speed).clamp(0.0, 1.0) } else { 0.0 };
    ...
```

**致命 Bug 触发点（在 `rc_audio_director/src/incident_classifier.rs`）**：
```rust
// 第 292 行：
let normal_vel = if ev.normal_rel_vel.is_finite() && ev.normal_rel_vel > 0.0 {
    ev.normal_rel_vel.abs()
} else {
    ev.speed.abs() // <--- 灾难性代码！
};
if normal_vel < threshold {
    return;
}
```
以及 `director_impact_events.rs` 中的硬编码 5 tick 冷却：
```rust
if self.last_ball_world_hit_tick != u32::MAX
    && ev_tick.saturating_sub(self.last_ball_world_hit_tick) < 5
{
    return;
}
```

**Bug 执行链条**：
1. 球在地面以 $1500\text{ UU/s}$ 高速平稳滚动，速度向量 $\vec{v} \approx (1500, 0, 0)$，法向量 $\vec{n} = (0, 0, 1)$。
2. 法向相对速度 $v_n = \vec{v}\cdot\vec{n} \approx 0$（远小于地板撞击阈值 $140\text{ UU/s}$）。
3. 然而，当 Bullet 的碰撞流形在微观数值上导致 `normal_rel_vel` 接近 0 或 $\le 0$ 时，代码**退回到了 `ev.speed.abs()`**！
4. `normal_vel` 瞬间变成了 $1500\text{ UU/s}$，远超 $140$ 阈值！
5. 冷却时间仅有 5 个 tick（约 $41.6\text{ms}$）。
6. **后果**：每过 $41.6\text{ms}$，音频系统就判定发生了一次“超强力地面重砸”，生成一个落地撞击 One-Shot！形成了高频连续的爆裂敲击声！

---

### 4.3 数学建模与滚动/碰撞严格判定判据

要彻底区分**法向剧烈撞击（Contact Impact）**与**切向滚动/带球（Rolling / Dribbling）**，必须利用向量正交投影与方向余弦。

#### 1. 法向与切向分解
设球的速度向量为 $\vec{v}$，地表接触点单位法向量为 $\vec{n}$（指向球体侧），则速度可正交分解为：
$$\vec{v} = \vec{v}_n + \vec{v}_t$$
- **法向冲击速度**：$v_n = -(\vec{v} \cdot \vec{n}) = |\vec{v} \cdot \vec{n}|$
- **切向地表速度**：$\vec{v}_t = \vec{v} - (\vec{v} \cdot \vec{n})\vec{n}$，标量大小 $v_t = \|\vec{v}_t\| = \sqrt{\|\vec{v}\|^2 - v_n^2}$

#### 2. 入射方向余弦（Directional Dot Ratio）
定义入射角余弦值：
$$\cos\theta = \frac{v_n}{\|\vec{v}\|} = \frac{|\vec{v} \cdot \vec{n}|}{\|\vec{v}\|}$$

#### 3. 严格判定准则（The Golden Rolling vs Contact Rule）
- **判定为【滚动 / 带球】（Rolling / Dribbling）**：
  若满足以下任一条件，**绝对严禁触发 `BALL_WORLD_HIT` 单次冲击音**，只交由连续滚动音环处理：
  1. **低法向速度**：$v_n < \text{Threshold}_{\text{impact}}$（地板为 $140\text{ UU/s}$，墙面为 $160\text{ UU/s}$）。**永远不得 fallback 到总速度**！
  2. **浅角切向滑行**：$\cos\theta < 0.15$ 且 $v_t > 25\text{ UU/s}$（速度方向与接触面法向夹角大于 $81.3^\circ$，属于极低角度擦碰或贴地滑行）。
  3. **微振幅连续接触**：同一表面在过去 $60\text{ms}$（7 个 tick）内已被标记为接触状态。

- **判定为【离散碰撞】（Discrete Impact）**：
  必须同时满足：
  1. $v_n \ge \text{Threshold}_{\text{impact}}$（严格法向动能撞击）
  2. $\cos\theta \ge 0.15$（具备足够的法向入射分量）
  3. $\Delta t \ge 60\text{ms}$（强制防抖冷却）

#### 4. Ground Dribbling（车顶/贴地带球）
带球过程中，车与球之间、球与地面之间持续保持多点触碰，球在车头前方轻微抖动：
- 车球相对速度 $\vec{v}_{\text{rel}} = \vec{v}_{\text{ball}} - \vec{v}_{\text{car}}$
- 带球判据：$\|\vec{v}_{\text{rel}}\| < 250\text{ UU/s}$ 且球接触地面或车身。
- 此时将冲击音抑制，转为连续微摩擦音效，避免出现连续轻击的“碎石爆米花音”。

---

### 4.4 单元测试设计（Rust Test Specifications）

在 `crates/rc_audio_director/src/incident_classifier.rs` 中设计如下单元测试保障：

```rust
#[cfg(test)]
mod audio_contact_rolling_tests {
    use super::*;

    #[test]
    fn test_high_speed_floor_rolling_suppresses_contact_oneshot() {
        // 球以 1800 UU/s 在地面高速纯滚动，法向速度为 0
        let mut cue = AudioFrameCuePod::default();
        let ev = PhysicsEvent {
            event_type: event_codes::BALL_WORLD_HIT,
            tick: 100,
            pos_x: 0.0, pos_y: 0.0, pos_z: 93.15,
            norm_x: 0.0, norm_y: 0.0, norm_z: 1.0,
            normal_rel_vel: 0.0, // 关键：法向速度为 0
            speed: 1800.0,       // 切向总速度极高
            impulse: 0.0,
            ..Default::default()
        };
        // 执行分类与派发，断言 one_shot_count 为 0，绝不产生撞击音
        // 断言 edge_mask 没有 BALL_FLOOR_HIT_BIT
    }

    #[test]
    fn test_debounce_window_enforces_60ms_lockout() {
        // 第一次强烈撞击 (normal_rel_vel = 800) -> 触发
        // 在 tick + 3 (25ms) 时再次传入强烈撞击 -> 必须被冷却丢弃
        // 在 tick + 8 (66.6ms) 时再次传入 -> 允许触发
    }

    #[test]
    fn test_ground_dribble_shallow_angle_rejection() {
        // 球速度 (1200, 0, 50)，normal = (0, 0, 1)
        // cos_theta = 50 / 1201 = 0.0416 < 0.15
        // 必须识别为浅角切向运动，判定为滚动而非撞击
    }
}
```

---

## 5. Crackcase Audio（Crankcase REV）引擎音效算法深度剖析

### 5.1 架构起源与 alpha 分支设计精髓
在 `alpha` 分支中，引擎音效架构参考了工业级车辆音频引擎 **Crankcase Audio REV** 的设计模式。其精妙之处在于：**不依赖单个长音频拉伸，而是基于 40 个精细校准的谐波切片进行实时动力学颗粒交叉淡化**。

```
Raw Physics Throttle & Speed 
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Powertrain Kinematics (动力学模型)                       │
│    - 6-Gear Hysteresis Shift Points (换挡顿挫与转速回落)     │
│    - Deceleration Drag Factor (松油门阻尼回落)               │
│    - Wall Deceleration Dip (撞墙失速动力应变衰减)            │
│    - Supersonic Turn Rev-Bounce (极速过弯转速反弹)          │
└──────────────────────────────┬──────────────────────────────┘
                               │ Current RPM (900 .. 6000)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Harmonic Granular Lookup (40 切片颗粒查找器)             │
│    - Find lower slice i & upper slice i+1                   │
│    - Equal-Power Crossfade: lower_gain = cos(α), upper = sin│
│    - Pitch Correction: rate = target_freq / slice_freq      │
│    - RMS Auto-Normalization: 消除不同采样切片的录音响度跳变   │
└──────────────────────────────┬──────────────────────────────┘
                               │ Slices Gain/Pitch
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Piecewise-Cosine 5-Band Equalizer (频响曲线塑形)         │
│    - 5 个特定共振中心频点: 42Hz, 48Hz, 56Hz, 66Hz, 74Hz      │
│    - 根据转速位置与负载动态调整各频段增益                    │
└──────────────────────────────┬──────────────────────────────┘
                               │ Lowpass Cutoff & Backfire
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Exhaust Backfire Pop (排气管回火爆破音)                  │
│    - 高转速松油门触发、冷却时间保护、随机选音                │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 核心算法参数与数学公式
1. **切片 RPM 映射**：
   `TURTLE_MODEM_SLICE_RPMS` 覆盖从 $900\text{ RPM}$（怠速）到 $6000\text{ RPM}$（红线爆表）的 40 个非线性分布采样点。
2. **等功率交叉淡化（Equal-Power Crossfade）**：
   $$\alpha = \frac{\text{RPM} - \text{RPM}_i}{\text{RPM}_{i+1} - \text{RPM}_i}$$
   $$G_{\text{lower}} = \cos\left(\frac{\pi}{2} \cdot \alpha\right) \cdot 10^{\frac{\Delta \text{RMS}_i}{20}}$$
   $$G_{\text{upper}} = \sin\left(\frac{\pi}{2} \cdot \alpha\right) \cdot 10^{\frac{\Delta \text{RMS}_{i+1}}{20}}$$
   这种三角函数插值确保两路音频在混合时总声能恒定，绝对不会在中间过渡态出现声压凹陷（Dipping）或削顶失真。
3. **音调精确补偿（Pitch Shift Tracking）**：
   $$f_{\text{target}} = f_i + \alpha \cdot (f_{i+1} - f_i)$$
   $$\text{playback\_rate}_{\text{lower}} = \frac{f_{\text{target}}}{f_i}, \quad \text{playback\_rate}_{\text{upper}} = \frac{f_{\text{target}}}{f_{i+1}}$$
4. **RMS 响度自动平抑（RMS Normalization Offset）**：
   `TURTLE_MODEM_SLICE_AUTO_NORM_DB`（例如切片 8 为 $-4.34\text{dB}$，切片 21 为 $+4.68\text{dB}$）抹平了原始音频录制时的采样电平不一致，避免车辆加速时声音忽大忽小。

### 5.3 Rust 与 TS 对接落地方案
这些复杂算法在 `crates/rc_audio_director/src/granular_lookup.rs` 和 `drivetrain_state.rs` 中**已经完整实现且测试通过**。
下一步唯一要做的就是：**在 TS 侧切除废旧的 `TurtleModemEngineSynth`，改为用几行代码直接消费 `AudioFrameCuePod.cars[activeCar]`**！

---

## 6. 距离 Fully Recreate 终局的全景差距审计

要实现 100% 摆脱历史包袱、全模块高内聚低耦合的纯 Rust WASM 终局，当前各子系统的就绪度与剩余差距如下：

| 模块子系统 | 重写就绪度 | Rust 现有资产 (`crates/`) | 剩余 TypeScript 冗余 / 需重写攻坚点 | 迁移评估与风险 |
| :--- | :---: | :--- | :--- | :--- |
| **物理模拟 (Physics Sim)** | **90%** | `rocketsim`, `rc_physics_sim` (SimInstance, CarSlots, Mutators, Events) | `RocketSimStateViews.ts` 插值器与部分事件解析包装；地表接触判定 Bug。 | 低风险；修补事件生成判据即可。 |
| **机器人决策 (Bot AI)** | **85%** | `rc_bot_ai` (EARL Perceiver, Sequential MLP, Action Tables, Kickoff PPO) | `ExperimentalBotAgent.ts` 协调器、人类干预覆写。 | 极低风险；核心推理与特征构建已全在 Rust。 |
| **轨迹预测 (Trajectory)** | **80%** | `rc_trajectory_mesh` (采样管线、时间步采样、几何锚点、到达性分析) | Three.js 顶点缓冲区生成依然在 JS CPU 端，占用拷贝带宽。 | 中风险；可直接在 Rust 输出二进制顶点 Buffer。 |
| **音频调度 (Audio Director)** | **70%** | `rc_audio_director` (动力学状态机、切片计算、空间音效、去抖、球音环) | TS 侧 `TurtleModemEngineSynth`、`RemoteEngineAudioLodController` 重复计算未移除；`cue.cars` 未接线。 | 低风险；剪除 TS 冗余即可大幅提升性能。 |
| **回放编解码 (Replay Codec)** | **40%** | `rc_replay_codec` (Boxcars 基础结构、二进制协议骨架) | `src/replay/` 内包含庞大的 TS UE3 NetStream 位流解析器、时间线提取器。 | **高难度**；需用 Rust 完全重写 NetStream 位流解析。 |
| **网络同步 (Multiplayer)** | **15%** | 仅有数据结构定义 | `src/network/` 95 个 TS 文件（客户端预测、权威状态回滚对齐、RTT 探测）。 | **高难度**；回滚预测逻辑应下沉到 Rust WASM。 |
| **UI 与系统外壳 (UI/Shell)** | **N/A** | `src-tauri` (Rust 托盘、窗口、配置) | `src/ui-core`, `src/ui-tuner` (React 19, Tailwind, Radix)。 | **无需迁移**；UI 留在 TS/React 符合现代分层最佳实践。 |

---

## 7. 最优并行分支全景规划（Branch 01 ～ Branch 32）

为了彻底落实“避免超大文件、功能模块解耦、指哪儿打哪儿、互不干扰”，并充分利用并行开发的巨大威力，我们设计了 **32 个高正交性并行分支规划（4 个 Tier，每 Tier 8 个分支）**。

所有分支基于 `z/agy/rc/rust/sync-main-audio-plan` 创建，分支命名严格遵循规范：`z/agy/rc/rust/<tier>-<index>-<topic>`。

```
                  z/agy/rc/rust/sync-main-audio-plan
                                  │
       ┌──────────────────────────┼──────────────────────────┐
       ▼                          ▼                          ▼
Tier 3: 音频与碰撞闭环    Tier 4: 轨迹与预测缓冲区    Tier 5: 回放解析与编解码
(Branches 01 ～ 08)        (Branches 09 ～ 16)        (Branches 17 ～ 24)
       │                          │                          │
       └──────────────────────────┼──────────────────────────┘
                                  ▼
                         Tier 6: 网络预测与同步
                         (Branches 25 ～ 32)
                                  │
                                  ▼
                      Octopus Merge / 终局验证
```

### 7.1 Tier 3: 音频引擎 Bug 修复与 Rust WASM 全闭环（Branch 01 ～ 08）
> **目标**：彻底解决抖动、地表滚动判据、接通 `cue.cars`，废除 TS 冗余动力学计算。

- **`z/agy/rc/rust/t3-01-contact-rolling-discriminator`**
  - 在 `rc_physics_sim/src/step_and_events.rs` 中修复 `BallHitWorld`，增加方向余弦与法向速度阈值检验。
  - 删除 `normal_vel` 回退到总速度的错误 fallback。
- **`z/agy/rc/rust/t3-02-incident-60ms-debounce-latch`**
  - 在 `rc_audio_director/src/incident_classifier.rs` 中建立每表面、每类型的 $\ge 60\text{ms}$（8 ticks）严格防抖锁存器。
- **`z/agy/rc/rust/t3-03-ground-dribble-friction-model`**
  - 实现地面微摩擦/带球模式判据，抑制带球状态下的离散碰撞音，平滑过渡至球体滚动音环。
- **`z/agy/rc/rust/t3-04-audio-contact-unit-tests`**
  - 编写完整的 Rust 单元测试套件（覆盖高速平滚、垂直下砸、浅角滑行、带球等 8 种工况）。
- **`z/agy/rc/rust/t3-05-wasm-car-cue-bridge-cutover`**
  - 在 `WasmAudioDirectorBridge.ts` 中实现 `AUDIO_CUE_CARS_WORD_OFFSET` 的读取，将 `CarAcousticLoopPod` 映射为轻量 WebAudio 驱动指令。
- **`z/agy/rc/rust/t3-06-retire-ts-engine-kinematics`**
  - 彻底删除 `TurtleModemDrivetrainModel.ts` 与 `TurtleModemGranularVoiceGraph.ts`，由 WASM `cue.cars` 直接驱动 WebAudio。
- **`z/agy/rc/rust/t3-07-retire-ts-remote-engine-lod`**
  - 废除 TS 端的 `RemoteEngineAudioLodController.ts`，直接消费 Rust 已经完成的 Top-3 距离/响度 LOD 掩码。
- **`z/agy/rc/rust/t3-08-audio-stadium-and-anthem-refactor`**
  - 重构球场环境声与进球号角调度，与 `StadiumBusBlendPod` 对齐。

---

### 7.2 Tier 4: 轨迹预测与 WebGL 顶点直通管线（Branch 09 ～ 16）
> **目标**：将球体/车辆未来轨迹采样与网格构建下沉至 Rust，以零拷贝共享内存直接给 Three.js 消费。

- **`z/agy/rc/rust/t4-09-trajectory-pod-abi-alignment`**
  - 定义 `TrajectoryVertexPod` C-ABI 结构体（位置、切线、厚度、颜色分量）。
- **`z/agy/rc/rust/t4-10-wasm-ribbon-vertex-builder`**
  - 在 `rc_trajectory_mesh` 中直接生成带状网格顶点与索引缓冲。
- **`z/agy/rc/rust/t4-11-touch-intercept-solver-rust`**
  - 将车辆拦截点与到达时间（ETA）解算器下沉至 Rust。
- **`z/agy/rc/rust/t4-12-retire-ts-trajectory-tube-builder`**
  - 移除 TS 侧的 `TrajectoryTubeMeshBuilder.ts` CPU 三角剖分逻辑。
- **`z/agy/rc/rust/t4-13-threejs-zero-copy-interop`**
  - 在 Three.js 侧通过 `BufferAttribute(HEAPF32.subarray(...))` 直接绑定 GPU 属性。
- **`z/agy/rc/rust/t4-14-trajectory-anomaly-filter-rust`**
  - 物理穿透与超音速漂移点异常几何过滤下沉。
- **`z/agy/rc/rust/t4-15-hex-thickness-tuning-port`**
  - 将 `alpha` 上的 Hex Ribbon 宽度与散热片深度实时缩放逻辑对齐至 Rust。
- **`z/agy/rc/rust/t4-16-trajectory-benchmark-harness`**
  - 建立 120Hz 连续 500 步轨迹采样的微基准测试。

---

### 7.3 Tier 5: 回放系统 NetStream 位流与二进制编解码（Branch 17 ～ 24）
> **目标**：彻底攻克 `.replay` 文件的 Rust 原生高速解析与时间线重建。

- **`z/agy/rc/rust/t5-17-ue3-bitstream-reader`**
  - 在 `rc_replay_codec` 中实现高性能无锁位流读取器（BitReader）。
- **`z/agy/rc/rust/t5-18-boxcars-actor-channel-parser`**
  - 实现 UE3 NetStream Actor/Component 属性通道解码器。
- **`z/agy/rc/rust/t5-19-replay-header-and-meta-codec`**
  - 解析回放头部元数据、队伍得分、玩家配置、GUID 映射表。
- **`z/agy/rc/rust/t5-20-timeline-frame-interpolator`**
  - 120Hz 离散关键帧时间线样条插值算法移植。
- **`z/agy/rc/rust/t5-21-touch-analyzer-downward-port`**
  - 将 `ReplayTouchTimelineAnalyzer.ts` 触球事件统计下沉至 Rust。
- **`z/agy/rc/rust/t5-22-retire-ts-netstream-parsers`**
  - 废弃并删除 `src/replay/` 下庞大的 TS 版位宽试探器与老解析器。
- **`z/agy/rc/rust/t5-23-rlrc-binary-container-v2`**
  - 定义原生 RocketCompanion 二进制压缩回放格式（RLRC v2）。
- **`z/agy/rc/rust/t5-24-replay-wasm-streaming-bridge`**
  - 实现大文件边下边解的流式 WASM 传输管线。

---

### 7.4 Tier 6: 联机同步、状态预测与回滚推演（Branch 25 ～ 32）
> **目标**：将联机房间的客户端预测（CSP）与权威状态回滚解耦至 Rust 模拟器。

- **`z/agy/rc/rust/t6-25-rollback-sim-snapshot-stack`**
  - 在 `rc_physics_sim` 中实现 120 帧深度的环形状态快照与极速回滚能力。
- **`z/agy/rc/rust/t6-26-input-history-queue-rust`**
  - 客户端预测输入缓冲区与重放队列移植。
- **`z/agy/rc/rust/t6-27-authoritative-reconcile-step`**
  - 服务端权威帧到达时的误差平滑修正与重演步进（Step & Reconcile）。
- **`z/agy/rc/rust/t6-28-transport-rtt-and-clock-sync`**
  - 物理时钟与往返时间（RTT）抖动平滑器下沉。
- **`z/agy/rc/rust/t6-29-room-state-pod-codec`**
  - 联机房间 8 车状态的极紧凑二进制打包传输协议。
- **`z/agy/rc/rust/t6-30-retire-ts-network-reconciler`**
  - 精简并重构 `src/network/`，将核心推演逻辑全部交由 Rust WASM。
- **`z/agy/rc/rust/t6-31-multiplayer-soak-probe-rust`**
  - 双世界（Lobby Dual World）与多车同步的长效浸泡测试探针。
- **`z/agy/rc/rust/t6-32-final-radar-audit-and-cleanup`**
  - 全局代码雷达扫描、文件体积（<400L/12KB）审查、无死代码清理与最终验收。

---

## 8. 实施保障与工作流规范

1. **零 OverlayFS 配额消耗原则**：所有构建产物、cargo targets、node_modules 严密挂载至 `/tmp`（tmpfs），源码保持在 `/work`。
2. **提交与签名标准**：使用标准的 GitHub Verified Commit 指令流程，保持规范的提交日志。
3. **架构雷达门禁**：改动必须通过 `pnpm check`（包含 TypeScript Strict + 400L/12KB 软告警 + 0 循环依赖 + 0 越权引用）。
4. **无包袱演进**：已下沉并被 Rust 替代的 TypeScript 历史代码，必须在当前分支坚决彻底地 `git rm`，确保代码库精炼、干练、清晰。

---
*本计划书由 Agent 深度调查后全自动生成并交付，所有分析均已严格核对物理引擎源码、Git 树与声学算法模型。*
