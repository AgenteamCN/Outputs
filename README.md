# RocketCompanion Rust 分支同步与音频/重放/并行架构深度调研报告

**基准提交 (HEAD)**: `2cb143b4a794f440c98a90e64ab2985fe5fdb9a7` (Tier-2 Octopus Merge)  
**合并主分支 (alpha)**: `1facf976a06b38f56958c052b372a68c2ed9a9ca` (PR #1 ~ #6)  
**当前工作分支**: `z/agy/rc/rust/sync-main-audio-plan`  
**GitHub Verified Bot Commit**: `b26d3287295714c16f4f5f69d373498b963d6e70` (GPG Signed, `verified: true`)  

---

## 目录
1. [主分支同步与冲突解决总结](#1-主分支同步与冲突解决总结)
2. [工程验证与构建状态](#2-工程验证与构建状态)
3. [重放 (Replay) 体系深度调研：TS 实现与 Boxcars 对比](#3-重放-replay-体系深度调研ts-实现与-boxcars-对比)
4. [音频引擎架构深度剖析与 Rust 离载方案](#4-音频引擎架构深度剖析与-rust-离载方案)
   - 4.1 滚地异常异响根本原因与表面法向量/冲量判定重构
   - 4.2 引擎动力总成 (Powertrain) 与粒子粒度合成离载
   - 4.3 终极音频架构：Rust 物理驱动 + WebAudio 哑端渲染
5. [高效并行分支重建方案 (Tier-3 架构路线)](#5-高效并行分支重建方案-tier-3-架构路线)
6. [跨团队职责与边界划分矩阵](#6-跨团队职责与边界划分矩阵)

---

## 1. 主分支同步与冲突解决总结

主分支（`origin/alpha`）近期的核心演进（PR #1 ~ PR #6）集中在 UI 重构、Tauri 桌面集成、多人房间复现机制、Boxcars 重放对齐以及性能加速调优器。本次已按照无历史包袱、先进技术优先的原则完成完整合并。

### 1.1 冲突解决与代码取舍
在 `git merge origin/alpha --no-commit` 过程中，定位并解决了以下三个冲突点及相关依赖项：

1. **`package.json` 脚本融合**
   - **HEAD (Rust WASM)**: 保留 `build:wasm`、`build:wasm:dev`、`build:wasm:check` 以及 build/test 过程中的 `--if-missing` WASM 自动前置检查。
   - **origin/alpha**: 吸纳 Tauri v2 体系专属脚本 (`tauri:dev`, `tauri:dev:tray`, `tauri:build`, `tauri:icons`) 以及多人房间复现脚本 `repro:multiplayer-room`。
   - **结果**: 完美整合两端命令，开发者既能调试 Tauri 托盘与桌面窗口，也能无缝构建 Rust WASM 内核。

2. **`src/entities/trajectory/BallTrajectorySamplingController.ts`**
   - **取舍**: 采纳主分支（`origin/alpha`）中关于连续增量追帧与全量重算下的窗口坍缩逻辑（`isIncWindow ? (shouldCollapseToCatchup ? Math.min(...) : stepVisible) : actualCount`），避免了旧版本在剧烈击球碰撞或重置时预测曲线突跳的视觉伪影。

3. **`src/network/bot/BotInferenceWorker.ts` (Modify/Delete 冲突)**
   - **决策**: 执行 `git rm src/network/bot/BotInferenceWorker.ts`。
   - **原因**: HEAD 在 Tier-2 阶段（commit `454ce0f`）已将所有 Heuristic / Kickoff PPO / Botpack / EARL Perceiver 推理完全切至 Rust 侧 `crates/rc_bot_ai` 的 `BotDirector`，移除了独立 WebWorker 开销。该 TS 文件的删除符合架构精简和全面 Rust 化的方向。

4. **主分支引入的潜在缺陷修复**
   - **`src/game/scenario/multiplayerRoomReproHostFlowPipeline.ts`**: 主分支代码仍尝试从已废弃的 TS 文件导入 `ensureExperimentalEngine`，已纠正为从 `src/ai/BotWeightLoader.ts` 导入。
   - **`scripts/vite/gitHeadAutoRestartPlugin.ts`**: 主分支在重构 git head 监听时遗漏了 `baselineSignature` 变量声明，导致客户端拉取 `/__rc_git_head` 接口时 500 报错并引发死循环刷新。本次已补齐 `baselineSignature` 的完整生命周期跟踪与复位。

---

## 2. 工程验证与构建状态

所有改动均在沙盒规范下严格验证：

- **Radar 架构守护与全量类型检查**:
  ```bash
  $ pnpm check
  ======================================================================
   ROCKET COMPANION GEN-3 SEMANTIC ARCHITECTURE RADAR
  ======================================================================
   TypeScript & Rust   : 1311/1311 TS (100.0% TS) | 69 custom Rust files + rocketsim.tree-hash
   TypeScript Strict   : PASS ("tsc" --noEmit)
   Hard Boundaries     : 0 Circular Deps | 0 Value Re-Exports | 0 Upward Imports | Any<=5
   STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified.
  ======================================================================
  ```
- **Rust WASM 目标空间检查**:
  ```bash
  $ pnpm build:wasm:check
  [WASM-BUILD] Running cargo check (--target wasm32-unknown-unknown --workspace)...
  [WASM-BUILD] [OK] Workspace check passed in 26.48s.
  ```
- **生产打包构建 (Vite + Release WASM)**:
  ```bash
  $ pnpm build
  [WASM-BUILD] [OK] Wrote src/physics/core.wasm (913.5 KB, 182 exports, profile=release) in 17.94s.
  ✓ 3689 modules transformed.
  ✓ built in 27.54s
  ```

---

## 3. 重放 (Replay) 体系深度调研：TS 实现与 Boxcars 对比

### 3.1 现存 Rust 库 (`boxcars`) 的局限性
`boxcars` 是社区成熟的 Rocket League `.replay` 解析库，但在生产运行时存在以下致命痛点：
1. **未识别属性立即崩溃 (Strict Schema Failure)**:
   Psyonix 在每季更新（如新载具 hitbox、新平台标识符、定制突变器）时都会在 UE3 NetStream 中新增或扩展属性结构。`boxcars` 遇到未知类网络缓存或未识别 bit 位时直接返回反序列化错误，导致新版本录像完全无法回放。
2. **定频帧时间漂移与抖动**:
   `boxcars` 仅做原始帧数据抽取，对 Rocket League 录像中常见的变频网络包（从 30Hz、60Hz 到 120Hz 不等）、丢包跳帧、关键帧冻结（Keyframe Freezes）缺乏物理平滑合成层。

### 3.2 RocketCompanion TypeScript 引擎的领先创新
主分支在 `src/replay/rlrc/` 中重新实现的 NetStream 解析器相比 `boxcars` 具有代际优势：
1. **投机式位宽探测器 (`RLRCUnknownPropertyBitProber.ts`)**:
   - 具备自适应 **Auto-Probing Bit Skimmer** 机制。
   - 当遇到未知属性时，并不中断解析，而是利用预设的候选位宽步长（`STRUCT_EXPANSION_CANDIDATE_DELTAS`）进行前向投机试探，结合后续帧头的时间戳与增量合法性（`isValidRLRCFrameHeader`）反向锁定未知字段位宽，实现对未知新版本录像的“自愈容错解析”。
2. **动态类继承网络缓存构建 (`RLRCClassHierarchyCatalog.ts`)**:
   - 能够基于 UE3 类继承链做动态拓扑回退，自动继承父类的属性映射，免除全量硬编码。
3. **120Hz Hermite-Slerp 连续动力学合成 (`RLRCHermiteSlerpMathPipeline.ts`)**:
   - 将稀疏或不均匀的网络流刚体位置与四元数通过带张力约束的 Hermite 三次样条及球形线性插值（SLERP）平滑到 120Hz 物理基准，消除了最后单帧抖动与跳变。

### 3.3 结论与后续重构方案
- **保持算法逻辑，下沉编译内核**:
  TS 版本的**容错探测状态机与平滑数学公式**经过实战验证极为优秀，不应退回裸版 `boxcars`。后续应在 `crates/rc_replay_codec` 中将这套自愈探测（Bit Prober）与 Hermite-Slerp 管道移植为纯 Rust 实现，从而兼顾**纳秒级解析性能**与**超强容错自愈能力**。

---

## 4. 音频引擎架构深度剖析与 Rust 离载方案

### 4.1 滚地异常异响根本原因与表面法向量/冲量判定重构

#### 4.1.1 异响根因定位
经排查 `crates/rc_physics_sim/src/step_and_events.rs` 与 `crates/rc_audio_director/src/director_impact_events.rs`，发现两大致病点：
1. **Bullet 接触流形持续触发 (Continuous Manifold Emission)**:
   当球在地面上静止或滚动时，RocketSim Bullet 物理引擎在每一步（120Hz）都会生成 `ArenaEvent::BallHitWorld`。
2. **法向速度判定失效与过短冷却**:
   - 在 `incident_classifier.rs` 中：若 `ev.normal_rel_vel <= 0`，代码直接降级为使用全速 `ev.speed.abs()`。当球高速滚地时（如 1200 uu/s），全速远超地面碰撞阈值（140 uu/s）。
   - 在 `director_impact_events.rs` 中：地面击打冷却仅为 `5 ticks`（即 41.6ms，每秒触发 24 次！）。
   - **现象**: 球贴地高速滚动时，系统每 40ms 就把它判定为一次剧烈砸地碰撞，连续派发 `0x04` `BALL_WORLD_HIT` 单发音效（One-shot Thud），与原本的连续滚地音效（`ball_loops.rs`）产生严重相位叠加和“机关枪”式爆音。

#### 4.1.2 修复与数学方案
1. **法向内向速度严格投影**:
   碰撞单发音效必须仅由垂直于接触表面指向内部的法向撞击驱动：
   $$v_n = -(\vec{v}_{\text{ball}} \cdot \vec{n}_{\text{surface}})$$
   当球贴地滚动时，$v_n \approx 0$，切向速度 $\vec{v}_t = \vec{v} - (\vec{v} \cdot \vec{n})\vec{n}$。
   切向滚动仅激发 `BallAcousticLoopPod.rolling_gain`，绝不允许激发生硬的撞击单发音效。
2. **接触状态边沿检测 (Hysteresis Contact Latching)**:
   在 Rust 侧为球维护接触状态标志位 `was_in_surface_contact`：
   - 仅当从离地变为贴地（上升沿 `false -> true`），或者发生强烈的正向法向冲量跳变（$\Delta v_n > 250\text{ uu/s}$，如被车撞击）时，才允许产生一次性地面撞击音效。
   - 处于持续滚动状态时（`was_in_surface_contact == true`），完全屏蔽 `BALL_WORLD_HIT` 的地面分发。
3. **冷却窗口调整**:
   将地面离散撞击冷却从 5 ticks 扩大至 20 ticks (约 166ms)。

---

### 4.2 引擎动力总成 (Powertrain) 与粒子粒度合成离载

#### 4.2.1 现状分析
目前系统中存在**双重动力总成计算**：
- TS 侧 `TurtleModemEngineSynth.ts` 仍在逐帧执行换挡、RPM 上升/回落、超转速断油、排气回火模拟。
- Rust 侧 `crates/rc_audio_director/src/drivetrain_state.rs` 也维护了一套完整的 8 辆车动力学状态机。
这导致计算冗余，且 TS 垃圾回收（GC）会导致引擎音频微卡顿。

#### 4.2.2 离载控制机制
将所有动力学与声学插值 100% 转移至 Rust WASM：
1. **Rust 输出 POD 扩展**:
   `CarAcousticLoopPod` 结构体直接包含：
   - `rpm`: 连续物理平滑 RPM
   - `gear`: 当前挡位 (-1, 0, 1..6)
   - `engine_gain`: 距离衰减与负荷平滑后的总增益
   - `pitch_ratio`: 基础音高比率
   - `slice_weights[0..4]`: 粒度引擎切片的 4 个交叉淡入淡出权重
   - `doppler_pitch_scalar`: 3D 空间多普勒频移
   - `pan`: 双耳空间方位角
2. **离散回火/超音速音爆边沿触发**:
   通过 64 位标志掩码（`AudioFrameCuePod.one_shot_edge_mask_lo/hi`）输出标志位，音频前端无需计算何时触发回火，只需按位感知触发。

---

### 4.3 终极音频架构：Rust 物理驱动 + WebAudio 哑端渲染

```
   [ RocketSim 120Hz 物理世界 ]
               │
               ▼
┌────────────────────────────────────────────────────────┐
│      crates/rc_audio_director (Rust WASM 内部)         │
│  - 纯法向冲量检测 (v_n = -v·n) 与地面滚动迟滞锁存器     │
│  - 8 车 Turtle Modem 动力学 / 换挡 / 4-Slice 权重计算  │
│  - 3D 双耳空间多普勒频移、距离曲线与遮挡衰减           │
│  - 离散事件环形缓冲区消费、碰撞优先级裁决 (Top 8 Slots)│
└────────────────────────────────────────────────────────┘
               │
               │ (零拷贝共享内存 AudioFrameCuePod)
               ▼
┌────────────────────────────────────────────────────────┐
│            TypeScript WebAudio 哑端渲染器              │
│  - 不执行任何物理方程、不计算向量、不保存状态机        │
│  - 循环层: 读 rolling_gain / engine_gain 写入 GainNode │
│  - 挡位层: 读 slice_weights 设置 AudioBufferSourceNode │
│  - 单发层: 读 edge_mask / one_shot_slots 派发播放音效  │
└────────────────────────────────────────────────────────┘
```

该架构将所有计算密集型任务完全卸载到 Rust 侧，TS 侧变为纯粹的参数写入管道，彻底根绝因主线程 GC 或浮点运算卡顿带来的音频撕裂。

---

## 5. 高效并行分支重建方案 (Tier-3 架构路线)

为了以最高效率推进后续重构，防止分支间产生大面积合并冲突，推荐采用**契约先行 (Contract-First) 的并行分支拓扑**：

### 5.1 分支规划与命名规范 (共 4 个业务领域，8 个并行分支)
所有分支均以 `z/agy/rc/rust/*` 命名，基于当前的 `z/agy/rc/rust/sync-main-audio-plan` 签出：

```
z/agy/rc/rust/sync-main-audio-plan (基准: b26d328)
  │
  ├── [Group A: 音频导演重塑]
  │     ├── z/agy/rc/rust/t3-01-audio-incident-hysteresis  (法向内向冲量/滚地迟滞/地面单发冷却)
  │     └── z/agy/rc/rust/t3-02-audio-engine-synth-offload (WASM 输出 4-slice 权重，TS 哑端化)
  │
  ├── [Group B: 重放解码移植]
  │     ├── z/agy/rc/rust/t3-03-replay-prober-rust         (将 TS 投机位宽探测器移植至 rc_replay_codec)
  │     └── z/agy/rc/rust/t3-04-replay-hermite-slerp-wasm  (120Hz 动力学重构与平滑在 Rust 完成)
  │
  ├── [Group C: 物理回滚与多人同步]
  │     ├── z/agy/rc/rust/t3-05-physics-rollback-resim     (256 槽快照环原生回滚重模拟)
  │     └── z/agy/rc/rust/t3-06-multiplayer-packet-c-abi   (网络输入/状态包零拷贝二进制编解码)
  │
  └── [Group D: Tauri 桌面与工程自动化]
        ├── z/agy/rc/rust/t3-07-tauri-dev-rebuild-daemon   (完善 crates 改动检测与 Tauri 热重载)
        └── z/agy/rc/rust/t3-08-crossbar-tray-refinements  (Tauri 托盘/菜单/全屏配置器无缝联调)
```

### 5.2 为什么 8 个分支并行效率最高？
1. **领域正交 (Orthogonal Domains)**:
   - `rc_audio_director` 只读物理事件，不写物理状态。
   - `rc_replay_codec` 是纯离线/流式解码器，不影响实时竞技场。
   - `rc_physics_sim` 回滚机制仅影响预测同步层。
   - `src-tauri` 是独立桌面壳层。
   4 个方向互不修改彼此源码文件，冲突概率降至极低。
2. **C-ABI 接口契约锁定**:
   在开发前首先锁定 `POD` 结构体布局，只要内存定义不变，前后端便能完全解耦并行开发。
3. **最终收敛机制**:
   各分支研发完成后，参照本次合并经验，进行二次 Octopus Merge，形成稳定的 Stage 3 里程碑。

---

## 6. 跨团队职责与边界划分矩阵

针对用户关心的“具体问题归属哪一个团队”，清晰的边界定义如下：

| 领域 / 问题项 | 负责团队 | 协作团队 | 交付标准与边界划分 |
| :--- | :--- | :--- | :--- |
| **音频计算与事件判定** | **Rust 团队** | RocketCompanion 团队 | 表面法向投影、滚地判定、碰撞冲量阈值、动力学换挡与多普勒计算全部在 `crates/rc_audio_director` 内完成；向外只提供只读 POD 缓冲区。 |
| **WebAudio 声音回放** | **RocketCompanion 团队** | UI 团队 | 负责根据 Rust 导出的标志位和增益参数调用浏览器 WebAudio API 发声，封装轻量 Facade，不写物理判断。 |
| **重放位宽探测与平滑** | **Rust 团队** | RocketCompanion 团队 | 将 `RLRCUnknownPropertyBitProber` 与 Hermite 插值从 TS 移植到 `crates/rc_replay_codec`，提供高吞吐 WASM 解析器。 |
| **Tauri 桌面应用与托盘** | **Tauri 团队** | RocketCompanion 团队 | 负责 `src-tauri` 窗口生命周期、系统托盘右键菜单、跨平台二进制打包以及本地机器配置存储。 |
| **UI 页面与调优器 (Tuner)** | **UI 团队** | RocketCompanion 团队 | 负责 `src/ui-core`、`src/ui-tuner` 浮动窗口、参数滑动条、CSS 视觉效果，严禁涉及后端物理或 WASM 细节。 |
| **工程构建与热重载插件** | **RocketCompanion 团队** | Tauri 团队 / Rust 团队 | 维护 `scripts/vite/gitHeadAutoRestartPlugin.ts` 和 `devLauncher.ts`，确保无论改动 Rust、TS 还是 Tauri 都能精准触发热重载与编译。 |

---

## 7. 交付完成情况

1. **主分支同步**: 已将 `origin/alpha` (PR #1 ~ #6) 全部特性与修复合并至工作分支 `z/agy/rc/rust/sync-main-audio-plan`。
2. **冲突解决与代码自愈**:
   - 修复了 `package.json` 的双端构建与运行脚本冲突。
   - 修复了 `BallTrajectorySamplingController.ts` 曲线采样追帧逻辑。
   - 删除了被 Rust 完全取代的 `BotInferenceWorker.ts`。
   - 修复了 `multiplayerRoomReproHostFlowPipeline.ts` 的旧引用报错。
   - 修复了 `gitHeadAutoRestartPlugin.ts` 的 `baselineSignature` 运行时未定义异常。
3. **全线绿灯通过**:
   - `pnpm check` (100% TS, 0 errors, Radar Green)
   - `pnpm build:wasm:check` (Rust crates check clean)
   - `pnpm build` (生产模式 WASM 与 Vite 打包成功)
4. **GitHub Verified Commit**:
   - 生成由 GitHub App Bot GPG 自动签名的 Merge Commit: `b26d3287295714c16f4f5f69d373498b963d6e70`，已成功推送到远程 `origin/z/agy/rc/rust/sync-main-audio-plan`。
