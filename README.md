# Rocket Companion Rust 线头清理与规划审查报告

## 1. 三个分支线头的对比裁决与云端清理

对仓库内三个待决提交进行了完整父提交、文件差异及构建门禁核查（三者共同父提交均为 `2cb143b` Tier-2 Octopus Merge 与 `1facf97` alpha PR #6）：

| 提交 / 分支 | 状态与决策 | 核查事实 |
| :--- | :--- | :--- |
| `b26d3287295714c16f4f5f69d373498b963d6e70`<br>(`z/agy/rc/rust/sync-main-audio-plan`) | **保留（唯一基准）** | 正确保留了 PR #1 多人开球防重入修复（`RuntimeKickoffSignalSubController.ts`）、PR #6 Vite 插件 `signature` 修复（`gitHeadAutoRestartPlugin.ts`）以及 Tier-2 Rust WASM 开球管线（`kickoff_ppo.rs`、`kickoff_v1_v2_obs.rs`、`kickoffEpisodePrimitives.ts` 等）。实测 `pnpm check`（GREEN，1311/1311 TS，`tsc --noEmit`，0 环依赖）与 `pnpm build:wasm:check`（7 个 crates 全部通过）均通过。 |
| `ec3e076688954389e8afd983e0a1fe71e3a1a8e2`<br>(`z/agy/rc/rust/alpha-merge-and-audio-arch`) | **已从云端删除（HTTP 204）** | 合并冲突解决存在多处严重回退：在 `RuntimeKickoffSignalSubController.ts` 丢失了 PR #1 多人联机修复；在 `kickoff_ppo.rs`、`kickoff_v1_v2_obs.rs`、`kickoffEpisodePrimitives.ts` 等文件回退了 Tier-2 Rust WASM 改造；将 `replayViewerBaselineScenario.ts` tick 从 485 回退到 450；且在仓库根目录直接提交了 `ARCHITECTURE_MERGE_AND_AUDIO_REPORT.md`。 |
| `f64c3b85875e8444a133a194be26b8ece8a202f7`<br>(`z/agy/rc/rust/sync-main-audio-investigate`) | **已从云端删除（HTTP 204）** | 与 `b26d328` 仅差 1 个文件（`scripts/vite/gitHeadAutoRestartPlugin.ts`），缺失 `signature` 属性导致 Vite 插件存在类型与运行时隐患。 |

> 注：`b26d328` 当前仅落后 `origin/alpha`（`f5e8c9f`，PR #7 `refactor(radar)`）1 个非功能性提交，后续可直接合入 `alpha`。

---

## 2. 原规划书（`RC-RUST-AUDIO-INVESTIGATION-AND-ROADMAP-2026`）合理性审查

### 2.1 准确且合理的核心结论
1. **音频 `cue.cars` 断层属实**：`src/audio/WasmAudioDirectorBridge.ts:77` 定义了 `AUDIO_CUE_CARS_WORD_OFFSET = 22`，但全仓无任何读取；`crates/rc_audio_director` 每帧计算了 8 辆车的切片、增益、滤波和 LOD，TS 侧 `GameAudioSubsystem.ts` 与 `TurtleModemEngineSynth.ts` 仍在主线程重复计算。
2. **Rust Pure-Offload 方向正确**：所有动力学、切片插值、3D 衰减、LOD 留在 Rust，WebAudio 退化为纯执行驱动器（Dumb Driver）。
3. **碰撞抖动的物理根因正确**：Bullet 在球贴地静止或滚动时每 tick 触发 `BallHitWorld`，必须依靠法向速度 $v_n$、方向余弦 $\cos\theta = |\vec{v}\cdot\vec{n}|/|\vec{v}|$ 与 $\ge 60\text{ms}$ 防抖锁存来区分撞击与滚动/带球。

### 2.2 与代码库事实不符 / 不合理之处（必须修正）
1. **运行时碰撞音频 Bug 定位偏差**：
   - 原报告将 `crates/rc_audio_director/src/incident_classifier.rs:292` 的 `ev.speed.abs()` fallback 当作运行时真凶。
   - **事实**：`AudioDirectorInstance::consume_event_buffer()` 实际调用的是 `director_impact_events.rs` 的 `handle_ball_world_or_post_hit()`，**并未调用 `incident_classifier.rs`**（后者目前仅被 `lib.rs` 单元测试调用）。
   - `director_impact_events.rs` 虽未 fallback 到 `ev.speed`，但它**完全忽略了 `step_and_events.rs:124` 已经算好并写入 `custom_float` 的 `dir_dot`（$\cos\theta$）**，且使用硬编码 `< 5` tick（41.6ms）冷却、缺乏按表面锁存与带球抑制；同时 `step_and_events.rs` 在球贴地滚动时每 tick 都向 256 槽位事件环写入无效 `BallHitWorld`。需要同步修复 `director_impact_events.rs` 与 `step_and_events.rs`，并清理/统一 `incident_classifier.rs` 的重复逻辑。
2. **TS 音频删除清单与实际文件不符**：
   - 原报告要求“彻底删除 `TurtleModemDrivetrainModel.ts` 与 `TurtleModemGranularVoiceGraph.ts`”以及“废除 `RemoteEngineAudioLodController.ts`”。
   - **事实**：`TurtleModemDrivetrainModel.ts` 与独立的 `RemoteEngineAudioLodController.ts` **在 Tier-2 已经不存在**（`RemoteEngineAudioLodController` 是 `TurtleModemEngineSynth.ts:349` 内的类）；而 `TurtleModemGranularVoiceGraph.ts` 是 **WebAudio 40 切片节点图（`AudioBufferSourceNode` + `GainNode` + `BiquadFilterNode`）**，不能整文件删除，而应保留其作为 Dumb WebAudio 节点图，由 `cue.cars[i]` 直接喂入参数，裁撤 `TurtleModemEngineSynth.ts` 与 `turtleModemSynthAcousticPipeline.ts` 中的动力学/RPM/切片查找/LOD 计算。
3. **Tier 4（轨迹预测）几乎整组与 Tier-2 重复，且 `t4-15` 张冠李戴**：
   - 原报告规划 `t4-09`..`t4-13`（定义 Vertex POD、Rust 生成带状网格与索引、拦截求解器、删除 `TrajectoryTubeMeshBuilder.ts`、Three.js 零拷贝绑定）。
   - **事实**：`TrajectoryTubeMeshBuilder.ts` 已不存在；`crates/rc_trajectory_mesh` 已经实现了 `tube_extruder.rs`、`intercept_solver.rs`、`sampler_and_anomaly.rs`、`bounce_and_anchors.rs`，`src/physics/core.js` 已导出 `_physics_getTrajectoryPositionsPtr` 等全部零拷贝指针。
   - `t4-15-hex-thickness-tuning-port` 将 PR #6（`DriftwoodsHexGeometryProcessor.ts` 中球场六边形墙面 `Woods_Field_Walls_Hexes` 的 GPU 着色器属性）误认为轨迹丝带宽度，属于概念混淆。
4. **Tier 5（回放）与 Tier 6（联机）低估了现有 Rust 资产**：
   - `crates/rc_replay_codec` 已实现 `container.rs`（RLRC 容器）、`delta_transpose.rs`、`seeker.rs`（Hermite-Slerp）、`anomaly.rs`；真正缺失的是 `src/replay/`（106 个 TS 文件）中的 **UE3 `.replay` NetStream 位流读取与 Actor/Property 通道解码**。
   - `crates/rc_physics_sim/src/reconciler.rs` 已实现 **256 帧快照环形缓冲（`SNAPSHOT_RING_CAPACITY = 256`）与 1024 帧多车控制环（`CONTROLS_RING_CAPACITY = 1024`）**，并非原报告所说的“仅有数据结构定义（15%）”。真正缺的是把 `src/network/`（95 个 TS 文件）中的预测协调、RTT/时钟平滑、房间状态编解码与 `reconciler.rs` 接通并裁撤 TS 冗余。
5. **32 分支规划过于碎片化**：为凑 4×8=32 把已完成的轨迹/回滚基础设施重复列入，会带来不必要的合并与维护开销。

---

## 3. 修正后的并行推进计划（3 个 Tier / 16 个正交分支）

所有分支基于 `z/agy/rc/rust/sync-main-audio-plan`（`b26d328`）创建，遵循 `<400L`、`0 环依赖`、`pnpm check` + `pnpm build:wasm:check` 门禁。

### Tier 3：音频闭环与碰撞防抖（6 个分支）
1. `z/agy/rc/rust/t3-01-contact-rolling-gate`：在 `rc_physics_sim/src/step_and_events.rs` 对 `BallHitWorld` 增加法向速度与 `dir_dot` 门控，避免滚动时每 tick 污染事件环。
2. `z/agy/rc/rust/t3-02-director-debounce-and-unify`：在 `rc_audio_director/src/director_impact_events.rs` 接入 `ev.custom_float`（`dir_dot`）与 $\ge 60\text{ms}$ 表面防抖锁存，清理 `incident_classifier.rs` 中的 `ev.speed` fallback 与重复分类路径。
3. `z/agy/rc/rust/t3-03-ground-dribble-friction-model`：实现车-球相对速度 $< 250\text{ UU/s}$ 贴地带球摩擦判据，抑制离散碰撞音并平滑过渡到球体滚动音环。
4. `z/agy/rc/rust/t3-04-audio-contact-unit-tests`：覆盖高速平滚、垂直砸地、浅角滑行、带球、60ms 防抖窗口的 Rust 单元测试（直接测试 `AudioDirectorInstance` 运行时路径）。
5. `z/agy/rc/rust/t3-05-wasm-car-cue-bridge-cutover`：在 `WasmAudioDirectorBridge.ts` 接入 `AUDIO_CUE_CARS_WORD_OFFSET`，将 `CarAcousticLoopPod` 直接映射驱动 `TurtleModemGranularVoiceGraph`（保留为 Dumb WebAudio Driver）。
6. `z/agy/rc/rust/t3-06-retire-ts-engine-synth-and-lod`：裁撤 `TurtleModemEngineSynth.ts` / `turtleModemSynthAcousticPipeline.ts` 中的 TS 动力学、切片查找与 `RemoteEngineAudioLodController` 计算，对齐 `StadiumBusBlendPod`。

### Tier 4：回放 UE3 NetStream 下沉（5 个分支）
7. `z/agy/rc/rust/t4-01-ue3-bitstream-reader`：在 `rc_replay_codec` 中实现无锁 UE3 `BitReader`。
8. `z/agy/rc/rust/t4-02-netstream-actor-channel-decoder`：实现 UE3 NetStream Actor/Component 属性通道解码器。
9. `z/agy/rc/rust/t4-03-replay-header-meta-and-touch-stats`：解析 `.replay` 头部元数据、玩家/GUID 与触球时间线统计。
10. `z/agy/rc/rust/t4-04-replay-wasm-streaming-bridge`：实现流式 WASM 解码管线，对接现有 `container.rs` / `seeker.rs`。
11. `z/agy/rc/rust/t4-05-retire-ts-netstream-parsers`：废弃并移除 `src/replay/` 下冗余的 TS 位宽试探器与老解析器。

### Tier 5：联机预测与状态同步下沉 + 终局审计（5 个分支）
12. `z/agy/rc/rust/t5-01-reconciler-wasm-bridge-cutover`：将 `src/network/` 客户端预测与回滚逻辑接通至已有的 `rc_physics_sim/src/reconciler.rs`（256 帧快照环 + 1024 控制环）。
13. `z/agy/rc/rust/t5-02-rtt-clock-and-smoothing-rust`：物理时钟、RTT 抖动平滑与视觉纠偏衰减下沉至 Rust。
14. `z/agy/rc/rust/t5-03-room-state-pod-codec`：联机房间 8 车状态紧凑二进制打包编解码。
15. `z/agy/rc/rust/t5-04-retire-ts-network-reconciler`：精简重构 `src/network/`（95 文件），移除 TS 侧重复预测/回滚状态机。
16. `z/agy/rc/rust/t5-05-multiplayer-soak-and-radar-audit`：双世界/多车浸泡测试、`pnpm check` 与 `pnpm build:wasm:check` 全量验收。
