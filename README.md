# RocketCompanion Rust 重构分支深度诊断、复现路径与架构评估报告

> **基准环境**：仓库 `Antigravi-ty/RocketCompanion` | 基准 Commit `eb68dfcd0f2caa7a85af97f301626296afba4a37` (`origin/z/agy/rc/rust/fix-keyboard-dodge-air-steer`) | 本地诊断分支 `z/agy/rc/rust/diag-repro-report` | 对照基准分支 `origin/alpha` (`8d19408`)

---

## 一、 工具链验证与 `tmpfs` 一键恢复方案

### 1.1 当前工具链实测状态
在沙盒中通过 `/work/fix.sh --ref eb68dfcd0f2caa7a85af97f301626296afba4a37 --new-branch z/agy/rc/rust/diag-repro-report --no-check` 初始化后，所有重型依赖与构建产物均已隔离至 `/tmp` (`tmpfs`)，OverlayFS (`/work/RocketCompanion`) 保持源码纯净：

| 检查项 | 命令 | 实测结果 | 备注 |
| :--- | :--- | :--- | :--- |
| **Rust 工具链** | `rustc --version && cargo --version` | **PASS** (`rustc 1.100.0`, `cargo 1.100.0`) | 通过 `/usr/local/bin/{cargo,rustc,rustup}` 自动恢复至 `/tmp/rustup` 与 `/tmp/cargo-home` |
| **TS 架构雷达与类型检查** | `pnpm check` (`radar.ts` + `tsc --noEmit`) | **PASS** (0 循环依赖, 0 越界导入, 1333/1333 TS) | 发现 463 个文件处于 12KB..18KB / 400..800L 软提示区间 |
| **WASM Workspace Check** | `pnpm build:wasm:check` | **PASS** (12.67s) | 7 个自定义 crate (`rocketsim`, `rc_*`, `rocketsim_wasm`) 全部通过 |
| **WASM Dev 编译** | `pnpm build:wasm:dev` | **PASS** (16.80s) | 输出 `/tmp/rc-wasm-target/core.wasm` (1162.2 KB, 198 exports) |
| **Native `cargo test`** | `cargo test --workspace` | **FAIL** (`linker 'cc' not found`) | 沙盒未预装 `gcc`/`clang`；`wasm32-unknown-unknown` 使用内置 `rust-lld`，不受影响 |
| **Ducky 3v3 全生命周期 E2E** | `node scripts/runScenarioBaseline.mjs --scenario=online_3v3_ducky_full_lifecycle` | **PASS** (30.4s, 2400 ticks, 22/22 断言通过) | Headless Chrome + WebGL2 (SwiftShader) + WASM Ducky 3v3 实跑通过 |
| **全量基线套件** | `pnpm test:baseline` | **RED** (`multiplayer_6car_loadout_stress` 失败) | 捕获到真实物理回归：`phantomDoubleJumps=1, hasAirJumped=false` |

### 1.2 `tmpfs` 临时工具链一键恢复建议
目前 `/work/fix.sh` 已经完美处理了 `pnpm`、`node_modules`、`public` 静态资产、`rustup`、`cargo` 以及 `/tmp/rc-wasm-target` 的 `tmpfs` 映射。为了打通 **Native Rust 单元测试 (`cargo test`)** 与 **WASM 体积优化 (`wasm-opt`)**，建议在恢复脚本中追加两项轻量级 `tmpfs` 引导：
1. **Native 链接器 (`lld` / `clang` 提取至 `/tmp`)**：在无 root 权限或避免污染 OverlayFS 的前提下，通过 `apt-get download lld clang` + `dpkg -x *.deb /tmp/llvm-sysroot` 并在 `CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=/tmp/llvm-sysroot/usr/bin/ld.lld`（或 wrapper script）下运行 `cargo test --workspace`，即可在不污染系统盘的情况下跑通全部 Rust 单元测试（包括 `rc_audio_director/src/director_impact_tests.rs` 等）。
2. **`wasm-opt` / `wasm-pack`**：`buildPhysicsWasm.mjs` 已内置纯 `cargo build --target wasm32-unknown-unknown` + 导出检查，无需强依赖 `wasm-pack`；若需 release 体积裁剪，可将 `binaryen` (`wasm-opt`) 解压至 `/tmp/bin/wasm-opt`。

---

## 二、 Ducky Model 可行性验证与双轨自动化复现路径

### 2.1 Ducky Model 现状确认：已在 Rust (`rc_bot_ai`) 原生跑通
经查证，**SafeTensors 模型加载与推理已经完全下沉至 Rust WASM (`crates/rc_bot_ai`)**，并非在 TypeScript 中计算：
- 模型文件：`public/assets/models/experimental/botpack_9700488/ducky_policy.safetensors`（仅 **1.6 MB**）。
- Rust 侧实现：`crates/rc_bot_ai/src/safetensors_store.rs` 直接解析二进制 SafeTensors，`mlp_obs_extended.rs` 构建 `DuckyObs_V2_01(3)` (295 floats)，`sequential_mlp.rs` 使用 `DUCKY_SPEC` 执行 SIMD MLP 推理，`bot_director.rs` 统一调度。
- **Node.js 纯 WASM 实测**：通过 `/tmp/test_ducky_wasm.mjs` 直接加载 `core.wasm`、`public/arenamesh/*.cmf` 与 `ducky_policy.safetensors`，`_physics_loadBotWeights(7, ...)` 成功返回 `18` 个张量，`_physics_isBotPolicyLoaded(7) === 1`。
- **Headless Chrome 实测**：`online_3v3_ducky_full_lifecycle` 场景已使用 3 个 Ducky Bot 跑通 3v3 完整对局（Lobby → Prepare → Kickoff → Playing → Goal → Director Replay → Overtime → Golden Goal → `.RCReplay` 往返 → Podium → Replay Viewer → Freeplay），2400 ticks 零控制台报错。

### 2.2 为什么现有测试没抓到 Flip Reset / 进球多音效 / Replay 闪烁？
核心原因在于：**现有的 `online_3v3_ducky_full_lifecycle` 断言只检查了流程状态机（`phase === 'replay'`、`blueScore >= 2`、`camera_no_nan_poses`）**，而没有检查：
1. **音频单帧/单事件播放次数**（例如每次进球 `sfx_goal_poof` / `goalScored` 被 `AudioSlotPool` 和 `gameAudio.play` 实际触发了几次）；
2. **Flip Reset 的物理接触与事件一致性**（车轮接触球体 `UserInfoType::Ball` 次数 vs `0x0F` 事件发射次数 vs 连续 Reset 场景）；
3. **Cinematic Replay 期间车辆 Mesh 坐标的帧间震荡度**（目前只测了 Camera 位姿平滑度，未测 Replay 期间每辆车 `car[i].group.position` 的帧间位移反转/抖动）。

### 2.3 推荐的自动化复现与 Golden Standard 对比方案

建议构建**双轨自动化复现链路**：

#### 轨道 A：Headless Chrome 全链路探针（抓音频、相机、Mesh 闪烁）
复用 `scripts/runScenarioBaseline.mjs` + `soakAudioAndHeapHooks.ts`，新增一个专用诊断场景（或扩展 `online_3v3_ducky_full_lifecycle`）：
- **Ducky 双模式驱动**：
  - **Bot 填充模式**：`configureBotSlot(1..5, policyId=7, ...)` 填充 5 个 Ducky Bot。
  - **接管用户输入模式**：对 `car[0]` 启用 Ducky 接管（在 `SimulationStage` 前通过 `_physics_stepBotInferenceMask(0x3F, ...)` 覆盖本地 `playerControls`，或使用预设 `.RCDPack` / `.RCReplay` 注入特定空中 Flip Reset 切片并由 Ducky 接管）。
- **三大问题自动断言探针**：
  1. **进球音效计数探针**：在 `AudioSlotPool.playBuffer` 与 `gameAudio.play` 上挂载计数器（按 `sfx_goal_poof` / `goal_explosion` 统计），每次进入 `goal` phase 至下一次 `kickoff` 之间，**严格断言 `goalAudioCount === 1`**（当前分支实测会 `>= 2`）。
  2. **Replay 车辆闪烁探针**：在 `phase === 'replay'` 且 `cameraMode === 'director'` 期间，每帧记录 `arena.cars[i].group.position`，计算相邻两帧位移向量点积 `dot(delta_t, delta_{t-1})`。若出现连续大幅反向跳变（`|delta| > 30uu` 且点积为负），立即判定为**车辆闪烁/双源抢写**。
  3. **Flip Reset 探针**：构造“空中单跳 → 4 轮触球 → 不翻转直接二次触球 / 快速连 Reset”合成切片，统计 `wheels_with_contact` 含 `Ball` 的上升沿次数 vs `0x0F` 事件数 vs `0x0F` 音频槽位触发数。

#### 轨道 B：Node.js + WASM 极速差分测试（以 `origin/alpha` 为 Golden Standard）
将 `origin/alpha`（C++ WASM + 原版 TS）与当前 `rust` 分支视为两个黑盒（对我们自身是白盒）：
- 使用完全相同的开球种子 `resetKickoff(42)` 与相同的 Ducky 控制序列（或录制一份 `.RCReplay` 控制流），分别在 `origin/alpha` 与 `rust` 分支回放 3v3 10,000 ticks。
- **可接受差异（白名单）**：随机种子不同导致的开球阵型差异、浮点微扰导致的 1~2 tick 碰撞时间差。
- **不可接受差异（红线）**：相同触球状态下的 Flip Reset 判定分歧、单次进球的事件发射数量 > 1、High-RPM 音频增益突变、Replay 帧间状态跳变。

---

## 三、 四大核心问题源码级根因诊断

### 3.1 问题一：Flip Reset 时灵时不灵（`0x0F` 判定与多重防抖冲突）

用户提到“旧版本通过车辆轮胎接触点的物体指针判断它是不是球”。在 Rust 版本中，虽然保留了 `UserInfoType::Ball` 检查，但**在 `car_modifiers.rs`、`car_action_events.rs`、`rc_audio_director` 与 TS `CarActionDebounceRegistry.ts` 中叠加了四层相互矛盾的限制**：

1. **连续 Flip Reset 被条件锁死（`crates/rc_physics_sim/src/car_modifiers.rs:181-189`）**：
   ```rust
   let flip_reset_this_tick = !st.is_demoed
       && is_on_ground
       && !pre.is_on_ground
       && (has_ball_or_car_wheel || is_ceiling_contact)
       && (pre.has_jumped
           || !pre.had_air_flip_avail
           || pre.has_double_jumped
           || pre.has_flipped)
       && st.has_flip_or_jump();
   ```
   - **缺陷 A**：如果车辆已经获得过一次 Flip Reset（或从天花板/门框掉落未按跳跃），此时 `has_jumped == false`、`has_flipped == false`、`has_double_jumped == false`、`had_air_flip_avail == true`，上述 `&& (...)` 条件直接为 **`false`**！导致**未消耗掉上一次 Flip 的情况下，第二次四轮触球无法触发 Flip Reset 事件与音效**。
   - **缺陷 B**：`is_on_ground` 在 `crates/rocketsim/src/sim/car/base.rs:1048` 定义为 `num_wheels_in_contact >= 3`。如果车辆高速蹭球，第 T 帧只有 2 轮射线命中球体、第 T+1 帧另外 2 轮命中（或车体先发生刚体弹开导致同一帧内未能凑齐 3 轮 raycast 命中），`is_on_ground` 根本不会变成 `true`，Flip Reset 直接丢失。
2. **物理层与音频层冷却时间不一致（12 ticks vs 60 ticks）**：
   - `crates/rc_physics_sim/src/car_action_events.rs:169` 使用 **12 ticks (100ms)** 冷却。
   - `crates/rc_audio_director/src/director_vehicle_events.rs:185` 与 `vehicle_incident_cues.rs:344` 却使用 **60 ticks (500ms)** 冷却！
   - 若 0.5 秒内完成快速 Double Flip Reset，物理层发出了 `0x0F`，但音频导演**静默吞掉了第二次音效**。
3. **TS 侧双重拦截与墙上时钟污染（`src/game/pipeline/stages/CarActionEventSynthesizer.ts:226` & `CarActionDebounceRegistry.ts:320`）**：
   - `CarActionEventSynthesizer.ts:226` 存在硬拦截：`if (networkReconciler && !networkReconciler.lobbyDualWorldMode) return;`——**在线/联机模式下本地预测的 `FLIP_RESET_GAINED` 会被直接丢弃**。
   - `CarActionDebounceRegistry.ts:321` 同时检查 `tickDelta < 60 || timeDeltaMs < 500`，使用了 `performance.now()` 墙上时钟，在倍速快进或帧率波动时行为不可预测。
4. **回滚/Seek 后的单调性假设失效**：
   - `director_vehicle_events.rs:184` 使用 `ev_tick.saturating_sub(dt_state.last_flip_reset_tick) < 60`。若发生网络回滚或 Replay 往回拖拽（`ev_tick < last_flip_reset_tick`），`saturating_sub` 结果为 `0 < 60`，导致后续 Flip Reset 音效被误杀。

---

### 3.2 问题二：进球触发的音频明显多于一次（多链路并发 + 回滚重触发）

当前架构中，**单次进球会同时激活 Rust 音频导演与多处 TS 业务层代码**，导致 `sfx_goal_poof.m4a`（`goalScored`）被重复播放多次：

1. **Rust 音频导演链路**：
   - `crates/rc_physics_sim/src/car_modifiers.rs:279` 检测到进球后发射事件 `0x0E`。
   - `crates/rc_audio_director/src/director_impact_events.rs:403` (`handle_goal_scored`) 生成 `AudioOneShotSlotPod { event_type: 0x0E }`。
   - `src/audio/wasmAudioOneShotDispatcher.ts:253` 消费该槽位并调用 `pool.playBuffer(incBufs.goalScored, ...)`。
2. **TS 业务层并行重复播放（至少 5 个独立入口未与 WASM 解耦）**：
   - `src/game/runtime/RuntimeGoalReplaySignalSubController.ts:104`：收到 `SCORE_UPDATED` 信号时直接调用 `audio?.play('sfx_goal_poof', 1.0)`。
   - `src/game/modes/freeplay/freeplayTouchAndGoalPipeline.ts:342`：Freeplay 进球记录时直接调用 `audio.play('sfx_goal_poof', 1.0)`。
   - `src/game/modes/experimental-multi-bots/ExperimentalMultiBotsGoalIncidentTracker.ts:210`：Bot 对局进球时直接调用 `audio?.play?.('sfx_goal_poof', 1.0)`。
   - `src/game/modes/static-training/StaticTrainingSimulationController.ts:365`：训练模式进球时直接调用 `audio?.play?.('sfx_goal_poof', 1.0)`。
   - `src/replay/ReplayFrameEventDispatcher.ts:221`：回放事件分发时再次调用 `gameAudio?.play?.('sfx_goal_poof', 1.0)`。
3. **去重窗口过短 + 状态回滚重触发**：
   - `wasmAudioOneShotDispatcher.ts:15` 的 `SAME_FRAME_DEDUP_MS` 仅 **28ms**（约 1.6 帧）。
   - 网络回滚或状态恢复（`state_sync.rs:179`）会将 `goal_scored_flag` 恢复为 `0.0`，且 `director.rs:313` 在 `current_tick < last_sim_tick` 时会将 `last_goal_scored_tick` 重置为 `u32::MAX`——当客户端重算跨过球门线时，`0x0E` 会再次发射并突破 28ms 窗口，导致同一粒进球播放 2~4 次以上爆炸音效。

---

### 3.3 问题三：Cinematic Replay Camera 期间车辆来回闪烁

在 Cinematic Goal Replay (`phase === 'replay'`, `cameraMode === 'director'`) 期间，车辆网格位置存在**多写入源抢占（Multi-Writer Race）**：

1. **`SimulationStage` 与 `GameplayModeStage` 同帧双写 `interpolator`**：
   - 在 `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts:105` 中，`reconciler.reconcile(now)` **在 `replayPlaybackMode` 下依然被无条件调用**。
   - 紧接着第 168 行调用 `applyReplayFrameToEngine(physics, pkt.stateSnapshot, interpolator)`，以 `prevFrameBuffer = null, alpha = 1.0` 写入 `interpolator`。
   - 随后在 `GameplayModeStage` 中，`networkStreamedGoalReplayCoordinator.ts` → `ReplayTimelineController.stepPlayback` → `executeReplayPlaybackStep` (`ReplayDirectorStatePipeline.ts:389`) 又会依据 `cursorFloat` 与 `alpha` 再次调用 `applyStepFrameAndControlsToEngine` 改写 `interpolator`。两组不同的 `(prevState, currState, alpha)` 在相邻帧/同帧间交替生效，导致车辆在两帧位置之间来回跳变。
2. **`PresentationStage` 仍叠加联机平滑偏移 (`smoothOffset`)**：
   - `src/game/pipeline/stages/PresentationStage.ts:128-137` 在 `reconciler` 存在时，**未判断 `isGoalReplay` / `replayPlaybackMode`**，每帧继续执行 `reconciler.updateSmoothing(dt)` 并将 `smoothOffset` 强行加到 `activeCarMesh.group.position` 上，使主角车辆在回放坐标系中发生位置撕裂与闪烁。

---

### 3.4 问题四：高转速引擎音频“滋啦滋啦”爆音（对比 `origin/alpha`）

- Commit `52b8173` 已针对该问题做了显著优化：75ms 断油回弹（rev-limiter bounce）、95-RPM 切片锚点间距、余量缩放（headroom scaling）与 `setTargetAtTime` 音频率平滑，整体表现已优于 `origin/alpha`。
- **残留风险点**：高转速下若 `TurtleModemGranularVoiceGraph` 的多切片循环点未对齐过零点（zero-crossing）或 RPM 在 95-RPM 锚点边界高频抖动，仍可能引发微弱相位干涉爆音；建议在 `rc_audio_director` 侧对输出给 WebAudio 的 RPM 增加轻量级施密特触发器/低通滤波。

---

## 四、 `origin/alpha` vs `rust` 分支功能差异与未迁移清单

对比 `d9e7498`（`rust` 分支最后一次同步 `alpha` 的公共祖先）之后 `origin/alpha` 的提交历史（至 `8d19408`），以下功能已在主分支落地但**尚未在 `rust` 分支重建/同步**：

| 主分支 PR / Commit | 功能模块 | 缺失/待重建内容说明 |
| :--- | :--- | :--- |
| **PR #19 (`bc1e9a2`)** | 文件扩展名统一 | 统一 `.RCReplay`、`.RLRCReplay`、`.RCGR`、`.RCDPack`、`.RCSPack`（移除 `.gz` 后缀约定） |
| **PR #18 (`3e8ce00`, `f70fbf3`, `abeab8f`, `3da79fa`, `5aab062`)** | **Dynamic Training Takeover** | Replay Viewer 捕获 4s 预卷 `.RCDPack`、双标记捕获、接管 HUD 与 3s 超时音效、修复槽位错乱与稀疏阵容、修复 120Hz 物理双步进、暂停时冻结 Boost Pad |
| **PR #17 (`3b3b3cc`)** | 渲染与后处理 | 修复半透明遮罩遮挡 Hex Glow 的问题，将护盾六边形网格默认不透明度设为 `0.03` |
| **PR #16 (`8484cf9`)** | 通知系统 UI | 增加黄色铃铛强制折叠开关与退出菜单自动展开逻辑 |
| **Telestrator (`da722e3`, `2f38585`, `5b1ee5c`)** | 3D 战术画笔 | CMF 射线检测（消除球场 Mesh 回退卡顿）、圆角端点/连接、视向箭头、实时预设重建、Dump Last Stroke |
| **PR #14 (`db84803`, `cf5cfee`)** | Track D 联机总线 | `RoomReplicationBus` 与 `ActivityOrchestrator`（`rust` 的 Wave 3/4 已部分实现，需与 `alpha` 最终契约对齐核验） |

---

## 五、 Rust vs TypeScript 下沉评估与架构解耦建议

### 5.1 模块下沉边界建议

| 子系统 | 当前归属 | 建议归属 | 权衡理由 (Trade-off) |
| :--- | :--- | :--- | :--- |
| **物理仿真 & 碰撞** | Rust (`rocketsim`, `rc_physics_sim`) | **保持 Rust** | 120Hz 核心计算与内存安全已完全落地 |
| **Bot AI & SafeTensors 推理** | Rust (`rc_bot_ai`) | **保持 Rust** | Ducky/Nexto/Ripple 等已全部使用 Rust SIMD 推理，无需 TS 介入 |
| **音频事件判定 & 空间计算** | Rust (`rc_audio_director`) + TS 混杂 | **100% 收口至 Rust** | 必须**删除 TS 侧所有独立 `audio.play('sfx_goal_poof')` 与 `CarActionDebounceRegistry` 重复防抖**，TS 仅保留薄 WebAudio 播放壳 |
| **网络协议编解码、状态差分、Jitter Buffer 与预测回滚循环** | TS (`src/network/`, 110 文件) | **核心状态机下沉 Rust** | WebRTC DataChannel / WebSocket I/O 留给 TS；二进制包序列化、压缩、256 槽位快照回滚 (`13c078e` 基础已就绪) 与平滑插值下沉 Rust，消除 GC 抖动 |
| **Replay 导演相机轨迹规划** | TS (`src/replay/`, 86 文件) | **中期评估下沉 Rust** | `ReplayDirectorGazeSolver` 涉及全帧扫描与平滑滤波，下沉 Rust 可与 `rc_replay_codec` 共享内存，避免大量 JS 对象分配 |
| **UI / Three.js 渲染 / WebAudio API** | TS | **保持 TS** | 浏览器 DOM/WebGL2/AudioContext 原生绑定，留在 TS 维护成本最低 |

### 5.2 代码组织与防大文件建议
- 当前 Rust 侧最大文件为 `sequential_mlp.rs` (548 行)，整体拆分健康。
- TS 侧仍有 463 个文件处于 400~800 行区间，且存在像 `sfx_goal_poof` 这样分散在 6 个不同文件中的“隐式耦合”。后续修复应优先**收口事件源**（Single Source of Truth），而不是在 TS 侧继续打补丁式防抖。

---

## 六、 待确认与后续权衡讨论的问题 (Questions for Discussion)

1. **Flip Reset 判定语义**：在物理上，是否严格要求与 Rocket League 原版一致（必须 **4 轮同时**触球才算 Reset，还是沿用 RocketSim 的 `>= 3` 轮）？另外，若车辆**尚未消耗上一次 Flip** 时再次四轮触球，是否应该重新触发 `0x0F` 音效与视觉特效？
2. **联机音频单一数据源**：是否同意彻底移除 TS 侧（`RuntimeGoalReplaySignalSubController`、`freeplayTouchAndGoalPipeline`、`ReplayFrameEventDispatcher` 等）对 `sfx_goal_poof` 的直接调用，全部统一由 `rc_audio_director` 的 `0x0E` 槽位（配合状态机锁死“单回合仅响一次”）驱动？
3. **主分支 PR #16~#19 同步优先级**：在修复上述 3 个核心 Bug 后，是否优先将 `origin/alpha` 的 PR #18 (Dynamic Training Takeover) 与 PR #19 (文件扩展名统一) 合并/重构进 Rust 分支，还是先完成网络通道 Rust 化？
4. **Native `cargo test` 需求**：是否需要在 `/work/fix.sh` 中集成轻量级 `/tmp` `lld` 链接器，使沙盒与 CI 能够直接运行 `cargo test --workspace` 来保护 `rc_physics_sim` 与 `rc_audio_director` 的边界用例？
