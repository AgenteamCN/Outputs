# RocketCompanion 主分支同步、音频计算全量 Offload 与架构演进调查报告

> **执行基准 Commit**: `2cb143b` (`origin/z/agy/rc/rust/t2-sp-octopus-merge`)  
> **合并目标分支**: `origin/alpha` (`1facf97`)  
> **工作分支**: `z/agy/rc/rust/sync-main-audio-investigate`  
> **合并结果 Commit**: `f64c3b8` (GitHub Verified Bot Commit, Signed by App)  
> **验证状态**: `pnpm check` (100% TS, 0 errors), `pnpm build:wasm:check` (Rust WASM 6 crates clean), `node scripts/buildPhysicsWasm.mjs --dev` (compiled in 14.9s)

---

## 1. 主分支同步与冲突裁决复盘 (Merge & Alignment Retrospective)

主分支 `origin/alpha` 在 PR #1 到 PR #6 期间合入了大量关键特性与底层修复。本次通过标准 Git 3-Way Merge（保留完整 DAG 追溯历史，严禁 rebase）将主分支全部改动吸收到 Rust 演进分支，并对冲突进行了严格的架构级裁决。

### 1.1 主分支吸收的核心改动一览
1. **PR #1 (Multiplayer Fixes Octopus)**:
   - 彻底解决工作线程输入饥饿 (`input starvation`)、回滚调和器顿挫死循环 (`reconciler hitch hard-snap loop`)。
   - 修复房主回环 (`host loopback`) 初始时间线同步、多客端在线房间名单 (`roster`) 与 WebRTC 数据通道状态。
   - 优化大厅目标球重置与环形缓冲区内存开销 (`PredictionHistoryBuffer`, `NetworkInputBuffer`)。
2. **PR #2 (Replay Smooth & Viewer Unify)**:
   - 消除 120Hz/30Hz 物理采样率不匹配导致的相位漂移速度偏差 (`phase-drift speed mismatch`)、关键帧定格与最后一帧抖动。
   - 重构 UE3 NetStream 解码器，对齐 Boxcars 字节流协议，新增未知属性投机性位宽探测器 (`RLRCUnknownPropertyBitProber`)。
   - 统一 `.replay` 原生文件与 `.rcs` 容器的解析入口，整合 Replay Viewer 与 RCS 导入器。
3. **PR #3 (Light Trail Effects & Performance Boost)**:
   - 新增 Light Beam 与 Light Wave 粒子拖尾特效、双喷口支持与 Bot 团队专属颜色方案。
   - 新增零泛光 Performance Boost 模式与独立 Active Tuner 浮动调优窗口。
4. **PR #4 (Tauri Configurator Refinements)**:
   - 增加 Crossbar #1 桌面/托盘图标生成流水线，优化预启动配置器 (`Layer1TauriConfiguratorRecipe`)，隐藏开发提示并优化桌面环境启动速度。
5. **PR #5 (UI Rework Fixes)**:
   - 解决 L1/L2 走查项，引入通知中心折叠架构，修复配置器布局抖动与浮动窗口契约。
6. **PR #6 (Hex Thickness Tuning)**:
   - 实现 Driftwoods 球场 GPU 六角丝带宽度与尾翼深度的实时着色器参数缩放。

---

### 1.2 冲突裁决与取舍细节

| 冲突文件 | 冲突点 | 裁决依据与处理方案 |
| :--- | :--- | :--- |
| `package.json` | 脚本冲突（HEAD 包含 `build:wasm*` 编译脚本，alpha 包含 `tauri:*` 与 `repro:*`） | **取并集**。保留 Rust WASM 的 `predev`, `build:wasm`, `build:wasm:dev`, `build:wasm:check`，同时合入主分支的 `tauri:*` 及 `repro:multiplayer-room` 脚本，保障桌面端与 WASM 编译双向可用。 |
| `src/entities/trajectory/BallTrajectorySamplingController.ts` | 轨迹重算时的切片截断预算（HEAD `actualCount` vs alpha `shouldCollapseToCatchup`） | **采纳 alpha 进阶方案**。主分支在非 1v1 网络环境及小球剧烈碰撞重置时，增加了 `shouldCollapseToCatchup` 状态保护，避免预测切片突变导致渲染拉花，逻辑更健全。 |
| `src/network/bot/BotInferenceWorker.ts` | 删除 vs 修改（HEAD 删除该文件，alpha 调整了推理步长） | **彻底删除 (听 Rust 分支)**。Rust 分支已通过 `crates/rc_bot_ai` 的 `BotDirector` 实现了 120Hz 纯 WASM 同步推理，TS 端由 `BotTunnelManager` 与 `BotWeightLoader` 直连，不再需要 Web Worker 进行异步推理。 |
| `src/game/scenario/multiplayerRoomReproHostFlowPipeline.ts` | 依赖导入路径失效（alpha 脚本引用了已被删除的 `ExperimentalBotAgent.ts`） | **修复至新架构**。将 `ensureExperimentalEngine` 的导入切至 `src/ai/BotWeightLoader.ts`，打通回环房间测试。 |
| `scripts/vite/gitHeadAutoRestartPlugin.ts` | 签名定义缺失 (`baselineSignature is not defined`) | **统一复合签名体系**。插件中扩展 `readCompositeGitHeadState()`，同时返回 `gitSignature`（主仓+public子模块 HEAD）与 `rustSignature`（`crates/` 目录源码散列），形成 `${git} [${rust}]` 全量复合签名，使得 Live WASM 热重编与 Vite 客户端重载双向闭环。 |

---

## 2. Replay 架构深度剖析：Rust Boxcars vs RC TypeScript RLRC

### 2.1 为什么不能直接生搬硬套现有的 Rust Boxcars 库？
社区已有的 Rust 库（如 `boxcars`）是一个优秀的**底层比特流反序列化器**，但它在工程定位上与游戏前端运行时存在巨大脱节：
1. **缺乏时序重采样与刚体插值**：Rocket League 原版录像的网络帧是变频记录的（物理帧通常为 30Hz 或 15Hz，且包含网络丢包导致的非均匀时间间隔）。Boxcars 仅仅将 Actor 与 Property 原始数值吐出，而游戏前端需要 120Hz 丝滑物理播放，必须做 Hermite-Slerp 连续样条插值。
2. **缺乏实体生命周期与跨局重建**：录像中的 Car、Ball、BoostPad 在进球回放、加时 Teardown 时会频繁销毁与重生。Boxcars 不维护 Entity Lifecycle Span，若没有上层生命周期管理器，车辆在进球瞬间速度会被直接抹平或造成假死。
3. **缺少音频与事件触发驱动**：Boxcars 并不具备物理碰撞法向量反向检测、撞柱（Goalpost Ping）判定等物理事件生成能力。

### 2.2 RC 在 TypeScript 层对 Replay 做的重大改进
RC 团队在借鉴 Boxcars 逆向结构的同时，进行了系统性的二次研发与进阶重构：
1. **确定性 UE3 NetCache 类层次目录 (`RLRCClassHierarchyCatalog`)**：
   - 取代了早期粗暴的 `parentId` / `cacheId` 猜测继承，构建了包含 235+ 属性与 45+ 属性解码器的静态类型字典。
   - 精确覆盖了 PsyNet NetVersion >= 10 的 8 字节平台 ID、QWordString、DemolishFx 以及 CamSettings 868.34.12 变体。
2. **投机性位流探测器 (`RLRCUnknownPropertyBitProber`)**：
   - 针对不同游戏版本可能出现的未记录私有属性，能够向前做试探性位探测并动态寻找位同步锚点，防止由于单个属性解析失败导致整场录像后续网络流完全错位（Bit-drift desync）。
3. **零相位局部漂移消除 (`Zero-Phase Drift Removal`)**：
   - 彻底解决了 120Hz 客户端物理时钟与录像网络帧之间的累积相位误差，消除回放过程中的偶发顿挫与尾帧抖动。
4. **表面法向反转运动学过滤 (`RLRCBallSurfaceImpactKinematics`)**：
   - 提取了包含 45 度斜角斜坡的地面/墙面反向与角偏转检测，精准区分小球的贴地滑行与真实回弹。

### 2.3 后续 Replay 迁移至 Rust (`crates/rc_replay_codec`) 规划
目前 `crates/rc_replay_codec` 已经包含了 `container.rs`, `seeker.rs`, `delta_transpose.rs`。后续应将 TypeScript 中的 `RLRCNetStreamBitDecoder` 与 `RLRCHermiteSlerpResampler` 彻底移植回 Rust，以利用 SIMD 实现录像的多线程极速预解析与内存紧凑排布。

---

## 3. 音频系统计算 Offload 深度调查与重构方案

### 3.1 核心缺陷根因：“小球滚在地上持续异响”
#### 现象描述
小球在地面上平稳滚动时，系统会高频触发离散的撞击音效（`BALL_WORLD_HIT`），发出类似“嗒嗒嗒”或机关枪般的持续碰撞声。

#### 深度技术根因剖析
在分析 `crates/rc_physics_sim/src/step_and_events.rs` 与 `crates/rc_audio_director/src/incident_classifier.rs` 后，定位出三个连环机制缺陷：

1. **物理引擎接触流形的连续触发特性**：
   - 在 Bullet / RocketSim 中，小球贴地或在斜坡滚动时，小球碰撞球体与场地网格每 tick 都在发生接触穿透解算，物理层每 tick 都会产生 `ArenaEvent::BallHitWorld`。
2. **法向速度与水平滚动速度的退化回退 (Fallback Bug)**：
   - 查看 `crates/rc_audio_director/src/incident_classifier.rs` 第 292-296 行：
     ```rust
     let normal_vel = if ev.normal_rel_vel.is_finite() && ev.normal_rel_vel > 0.0 {
         ev.normal_rel_vel.abs()
     } else {
         ev.speed.abs() // <-- 致命缺陷！
     };
     ```
   - 当小球贴地纯滚动时，其垂直于地面的法向相对速度 `normal_rel_vel ≈ 0`。
   - 代码在此处执行了 `else { ev.speed.abs() }`，**直接将水平滚动速度（例如 1000~1500 uu/s）误当作撞击法向速度！**
   - 此时 `1500 > cfg.ball_ground_hit_threshold (140.0)` 恒成立！
3. **仅靠冷却计时器（Cooldown）无法区分离散碰撞与滚动**：
   - 代码设置了 `cooldown = ms_to_ticks_120hz(cfg.incident_cooldown_ms)`（约 40ms = 5 ticks）。
   - 结果就是：每隔 40ms，水平速度再次通过阈值判定，强行发射一枚离散的 `BALL_WORLD_HIT` 音效 Slot！

---

### 3.2 引擎动力总成（Engine Audio）控制机制现状
当前 `crates/rc_audio_director/src/drivetrain_state.rs` 与 `granular_lookup.rs` 已实现了完备的仿真实时动力学：
- **状态机模型**：6 个前进挡位 + 倒挡 + 空挡 + 超速挡（Overdrive）。
- **参数控制链路**：
  - 输入：油门 `raw_thr`、速度 `fwd_spd`、转向 `steer`、手刹 `hb`、加速 `boost`、接地 `grounded`、时间差 `dt`。
  - 核心物理计算：飞轮转动惯量模拟、红线断油与回弹（Rev-limiter bounce）、换挡转速跌落与回火放炮（Backfire pop）、上墙重力做功负载压低（Wall-deceleration strain dip）、超音速转向啸叫。
  - 输出：粒度合成采样位置、平滑转速比率（`current_rpm / redline_rpm`）、当前挡位、平滑增益等。

---

### 3.3 架构目标：计算全量 Offload 至 Rust，TS 端退化为“纯哑巴播放器 (Dumb Sink)”

理想架构中，TypeScript 与 Web Audio 应当**完全移除非音频专属的逻辑运算**。

```
+-----------------------------------------------------------------------------------+
|                           Rust / WASM (rc_audio_director)                         |
|                                                                                   |
|  [物理/运动学状态]                                                                 |
|   - BallState: 位置、线速度、角速度、接触法向量                                   |
|   - CarState: 挡位状态机、转速、负载、油门开度、悬挂压缩比                         |
|   - CameraState: 听者位置 (Listener Pos)、朝向、视锥裁剪                           |
|                                                                                   |
|  [高负载计算（Rust 完全接管）]                                                     |
|   1. 连续法向反转冲量检测: delta_vn = (v_out - v_in) . n >= threshold             |
|   2. 滚动 vs 回弹状态分类: 接触持续时长 > 3 ticks 且 vn < 40 -> 判定为滚动        |
|   3. 动力总成 RPM 步进与挡位解算                                                  |
|   4. 3D 空间三维声学衰减曲线 (Bézier / Smoothstep / Inverse) 与距离裁剪           |
|                                                                                   |
|  [统一共享内存 C-ABI: AudioFrameCuePod]                                           |
|   - one_shot_edge_mask_lo/hi (64 个离散事件触发位)                                |
|   - 8 个 AudioOneShotSlot: { sound_id, volume, playback_rate, pan_x, pan_y, pan_z}|
|   - 1 个 BallAcousticLoop: { rolling_gain, rolling_rate, flying_gain, flying_rate }|
|   - 8 个 CarAcousticLoop:  { engine_rpm_norm, engine_gain, boost_gain, ... }      |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          | 120Hz 每帧 0 拷贝直接内存映射 (HEAPU32 / HEAPF32)
                                          v
+-----------------------------------------------------------------------------------+
|                        TypeScript / Web Audio (Thin Sink)                         |
|                                                                                   |
|  - 不做任何矢量乘除法、不做阈值计算、不跑冷却倒计时                               |
|  - 遍历 WASM 传来的 8 个 OneShot Slots：                                          |
|      if (slot.volume > 0) -> AudioSlotPool.play(slot.sound_id, slot.params)       |
|  - 连续音频更新：                                                                 |
|      rollingGainNode.gain.value = wasm.ball.rolling_gain                          |
|      rollingSourceNode.playbackRate.value = wasm.ball.rolling_rate                |
|      engineGainNode.gain.value = wasm.car[i].engine_gain                          |
+-----------------------------------------------------------------------------------+
```

#### 小球异响修复算法（Rust 实现）：
小球碰撞事件必须同时满足以下几何与动力学条件：
1. **法向入射速度判定**：在接触前，小球沿表面法向量的反向速度必须超过真实撞击阈值：
   $$\vec{v}_{in} \cdot \vec{n} < -v_{\text{impact\_threshold}} \quad (v_{\text{impact\_threshold}} \ge 160\text{ uu/s})$$
2. **法向速度反转冲量判定**：
   $$\Delta v_n = (\vec{v}_{post} - \vec{v}_{pre}) \cdot \vec{n} \ge \Delta v_{\text{threshold}} \quad (\Delta v_{\text{threshold}} \ge 220\text{ uu/s})$$
3. **滚动抑制**：若小球处于地面接触流形内且连续贴地大于 2 个 tick，强制禁止离散撞击事件触发，仅驱动 `BallAcousticLoopPod.rolling_gain` 连续环境循环音效。

---

## 4. 团队权责划分与协作矩阵 (Team Responsibilities)

为避免后续跨团队、跨模块协作时的相互干扰与边界推诿，建议确立以下清晰的责任矩阵：

```
+---------------------+----------------------------------------------------+--------------------------+
| 团队 (Team)         | 核心职责与专精领域 (Core Focus)                    | 严禁涉足与边界划分       |
+---------------------+----------------------------------------------------+--------------------------+
| 1. Rust 团队        | - 物理模拟底层 (rc_physics_sim / rocketsim)         | - 不编写 UI 组件与 DOM    |
|    (Our Team)       | - 决策 AI 与推理 (rc_bot_ai / SIMD SequentialMlp)  | - 不直接操作 Web Audio   |
|                     | - 轨迹预测与切片数据 (rc_trajectory_mesh)          |   AudioNode 原生接口     |
|                     | - 动力总成、声学定向与判定 (rc_audio_director)     |                          |
|                     | - 录像比特流编解码 (rc_replay_codec)               |                          |
|                     | - 保证 WASM C-ABI 内存排布与数据对齐               |                          |
+---------------------+----------------------------------------------------+--------------------------+
| 2. Rocket Companion | - 场景管线与三维渲染装配 (Three.js / Shader / Emitter)| - 不实现重度数值模拟逻辑 |
|    (TS/Core Team)   | - 网络同步与回滚插值协调器 (Reconciler / Client)   | - 录像解析与声学判定必须 |
|                     | - 薄音频播放接收器 (Web Audio Sink / Howler / Pool)|   调用 Rust WASM 导出    |
|                     | - 调度器与微任务管理 (frameBudgetScheduler)        |                          |
+---------------------+----------------------------------------------------+--------------------------+
| 3. Tauri 团队       | - 原生桌面外壳与生命周期 (src-tauri)               | - 不关心物理和比赛逻辑   |
|    (Desktop Shell)  | - 系统托盘、单实例锁、深色模式配置管理             | - 本地开发验证需提供     |
|                     | - 桌面图标构建工具 (generateTauriCrossbarIcons)     |   Headless / 容器降级容错|
|                     | - 本地文件系统安全沙盒与 IPC 通信                 |                          |
+---------------------+----------------------------------------------------+--------------------------+
| 4. UI 团队          | - 主菜单、HUD、车库、比赛大厅等界面 (ui-core)      | - 严禁在 UI 渲染周期内   |
|    (Frontend UI)    | - 调优面板交互 (ui-tuner / FloatingWindow)         |   触发同步物理/网络计算  |
|                     | - 样式令牌 (ui-system.css / Tailwind)              | - 通过 Zustand Store 解耦|
+---------------------+----------------------------------------------------+--------------------------+
```

---

## 5. 后续重构并行分支规划 (Parallel Branch Strategy)

为了保障后续研发效率最高，避免大爆炸式的巨型分支冲突，建议按照**垂直功能领域**与**C-ABI 契约**切分出 5 条高内聚、低耦合的并行开发分支。

### 5.1 分支切分图谱
```
origin/z/agy/rc/rust/sync-main-audio-investigate (基准锚点: f64c3b8)
   |
   +---> [分支 A] z/agy/rc/rust/audio-offload-incident-fix
   |        - 修复小球滚动法向量与冲量判定，杜绝持续撞击声
   |        - 实现引擎声音参数与距离裁剪纯 Rust 输出
   |
   +---> [分支 B] z/agy/rc/rust/replay-codec-stream-parser
   |        - 将 TS RLRCBitReader 与类继承缓存迁移至 crates/rc_replay_codec
   |        - 实现 Rust 侧 120Hz Hermite-Slerp 插值生成
   |
   +---> [分支 C] z/agy/rc/rust/trajectory-spatial-occlusion
   |        - 轨迹预测的碰撞反弹射线与遮挡计算下沉至 rc_trajectory_mesh
   |        - 优化 CPU 内存传输带宽
   |
   +---> [分支 D] z/agy/rc/tauri/ci-sandbox-decoupling
   |        - 为无 GUI/无 C 编译器的 Linux 容器环境增加 Tauri 构建容错
   |        - 优化 pnpm dev:tauri-http 与 Tray 模式自启动
   |
   +---> [分支 E] z/agy/ui/audio-and-replay-telemetry-hud
            - 针对音频事件与 Replay 时间线对齐提供轻量级可视化 HUD 仪表
```

### 5.2 实施原则
1. **契约先行 (Contract-First)**：先在 `crates/rc_*` 中定义好 POD 结构体与 C-ABI 头信息（如字段偏移、位掩码），一旦契约锁定，Rust 内部优化与 TS 接收器实现互不干扰。
2. **禁止超大单文件**：任何新模块单文件严格遵守 Radar 守护门禁（Soft <= 400L / 12KB，Hard <= 800L / 18KB），按照纯逻辑管道（Pure Pipeline）或领域状态机进行拆分。
3. **保持基准可查**：禁止 rebase，所有并行分支最终通过清晰的 Octopus Merge 或阶段性集成 Pull Request 回流。

---

## 6. 验证与总结

- 本次合并圆满完成主分支所有新特性吸收，消除了潜在的代码漂移风险。
- Radar 架构检测工具验证通过（100% TS，Volume Guardrails 验证合规，0 循环依赖，0 向上越界引用）。
- Rust WASM 6 个 Crates 检查通过，`src/physics/core.wasm` 编译成功。
- 所有代码已通过认证 Bot 签名完整推送到远程目标工作分支 `z/agy/rc/rust/sync-main-audio-investigate`。
