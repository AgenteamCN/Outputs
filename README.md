# Rocket Companion Rust 重构与音频引擎规划审计及分支裁决报告

## 1. 三线头基准分支裁决与云端清理结果

针对提供的三个待决策 Commit：
- `ec3e076688954389e8afd983e0a1fe71e3a1a8e2`（`origin/z/agy/rc/rust/alpha-merge-and-audio-arch`）
- `b26d3287295714c16f4f5f69d373498b963d6e70`（`origin/z/agy/rc/rust/sync-main-audio-plan`）
- `f64c3b85875e8444a133a194be26b8ece8a202f7`（`origin/z/agy/rc/rust/sync-main-audio-investigate`）

### 最终裁决：**仅保留 `b26d328` (`z/agy/rc/rust/sync-main-audio-plan`)，已从云端删除其余两个分支**

1. **`b26d328` (`z/agy/rc/rust/sync-main-audio-plan`) — 底子最正常（唯一保留）**
   - 正确合并了 `2cb143b`（Tier-2 Octopus Merge）与 `alpha`（`1facf97`）。
   - 完整保留了 `alpha` 主线在 `src/game/runtime/RuntimeKickoffSignalSubController.ts` 中修复网络回放倒带与开球死锁的关键提交（`39cdc82`、`815f67d`）。
   - 正确修复了 `scripts/vite/gitHeadAutoRestartPlugin.ts` 的 `signature` 属性冲突。
   - 实测 `pnpm check` 全绿：`1311/1311 TS`，`tsc --noEmit` 通过，0 循环依赖、0 越权引用。
2. **`f64c3b8` (`z/agy/rc/rust/sync-main-audio-investigate`) — 已从 GitHub 云端删除**
   - 合并 `alpha` 时遗漏 `scripts/vite/gitHeadAutoRestartPlugin.ts` 的 `signature` 字段，存在编译与热重载运行时缺陷。
3. **`ec3e076` (`z/agy/rc/rust/alpha-merge-and-audio-arch`) — 已从 GitHub 云端删除**
   - 合并质量存在 4 项严重问题：
     - **回退了 `alpha` 修复**：`src/game/runtime/RuntimeKickoffSignalSubController.ts` 被错误覆盖回 `2cb143b` 合并前版本，直接丢失了 `39cdc82` 与 `815f67d` 修复；
     - **夹带无关改动**：在 merge commit 中混入了 `crates/rc_bot_ai/src/kickoff_ppo.rs`、`kickoff_v1_v2_obs.rs`、`kickoffEpisodePrimitives.ts`、`kickoffGhostPrecomputePipeline.ts`、`replayViewerBaselineScenario.ts` 等非合并改动；
     - **破坏 Vite 插件状态更新**：将 `gitHeadAutoRestartPlugin.ts` 中的 `let baselineSignature` 改为 `const baselineSignature` 并删去了重置赋值；
     - **污染仓库根目录**：将 `ARCHITECTURE_MERGE_AND_AUDIO_REPORT.md`（257 行）直接提交进了主仓库。

> 注：原报告中声明“推荐工作分支 `z/agy/rc/rust/audio-recreate-plan`（已基于 `sync-main-audio-plan` 切出）”，经核查远端尚不存在该分支，后续应直接基于 `z/agy/rc/rust/sync-main-audio-plan`（`b26d328`）切出。

---

## 2. 原规划报告（`RC-RUST-AUDIO-INVESTIGATION-AND-ROADMAP-2026`）合理性审计

对照 `b26d328` 真实代码库逐项核验后，原报告**在 Tier 3 音频问题诊断上基本准确，但在代码路径定位上有遗漏，且 Tier 4 / Tier 5 / Tier 6 存在大量与现有代码严重不符的“重复规划”和“文件名幻觉”**，需要重构。

### 2.1 核验准确的部分
- **音频“半下沉”架构现状属实**：`src/audio/WasmAudioDirectorBridge.ts`（第 77 行）确实定义了 `AUDIO_CUE_CARS_WORD_OFFSET = 22`，但全仓库无任何读取代码；TS 侧 `TurtleModemEngineSynth.ts`、`TurtleModemGranularVoiceGraph.ts`、`turtleModemSynthAcousticPipeline.ts` 与 `VehicleAudioManager.ts` 仍在主线程重复计算 40 切片混合与动力学，而 Rust `rc_audio_director` 已算完所有 `cars[8]` 数据。
- **碰撞抖动根因方向正确**：`crates/rc_audio_director/src/incident_classifier.rs`（第 292–296 行）确实存在 `ev.normal_rel_vel <= 0.0` 时错误 fallback 到 `ev.speed.abs()` 的问题，导致高速平滚时触发高阈值地板撞击音。

### 2.2 失真或不合理的部分（必须修正）
1. **碰撞抖动漏掉了真正运行的 `AudioDirectorInstance` 路径**：
   - `crates/rocketsim_wasm/src/lib.rs` 实际绑定运行的是 `AudioDirectorInstance`（`crates/rc_audio_director/src/director_impact_events.rs` 中的 `handle_ball_world_or_post_hit`），而非 `incident_classifier.rs` 的 `process_physics_incidents`。
   - `director_impact_events.rs` 的真实问题是：使用 `ev.normal_rel_vel.max(ev.impulse)`、硬编码 `< 5` ticks（41.6ms）冷却，且**完全没有使用** `crates/rc_physics_sim/src/step_and_events.rs`（第 124 行）已经算好并写入 `custom_float` 的 `dir_dot`（$\cos\theta$）来做浅角滚动过滤。两条路径必须同时修复。
2. **Tier 3（分支 01~08）存在不存在的文件名且拆分过碎**：
   - 原计划点名要删除 `RemoteEngineAudioLodController.ts` 与 `TurtleModemDrivetrainModel.ts`，但 `src/audio/`（共 26 个文件）中**根本不存在这两个文件**。实际需清理/瘦身的是 `TurtleModemEngineSynth.ts`、`TurtleModemGranularVoiceGraph.ts`、`turtleModemSynthAcousticPipeline.ts`、`VehicleAudioManager.ts` 以及依赖 `TurtleModemEngineSynth` 的 `AudioMenuPreviewPlayer.ts`。
   - 将同一组滚动判据、防抖和单元测试硬拆为 `t3-01`~`t3-04` 四个分支属于过度碎片化，合并为一个内聚分支即可。
3. **Tier 4（轨迹预测）几乎全部重复规划**：
   - `crates/rc_trajectory_mesh/src/` 已经完整实现 `tube_extruder.rs`、`intercept_solver.rs`、`sampler_and_anomaly.rs`、`bounce_and_anchors.rs`、`ground_reachability.rs`、`aerial_reachability.rs`，且 `crates/rocketsim_wasm/src/exports/trajectory_exports.rs` 已导出零拷贝顶点/法线/UV/索引指针。
   - 原报告中点名要移除的 `TrajectoryTubeMeshBuilder.ts` 在 `src/` 中不存在；而 `t4-15-hex-thickness-tuning-port` 属于 Driftwoods 球场 GPU Hex Ribbon 着色器缩放特性，与轨迹网格毫无关系。
4. **Tier 5（回放系统）混淆了自研二进制流与 UE3 `.replay` 解析**：
   - `crates/rc_replay_codec/src/` 已经实现 `container.rs`、`delta_transpose.rs`、`seeker.rs`、`anomaly.rs` 及 `replay_exports.rs`，原计划的 `t5-20`、`t5-23` 属于重复建设。
   - 真正仍留在 `src/replay/`（106 个 TS 文件）的硬骨头是 **UE3 `.replay` NetStream 位流/Actor 通道解析** 及 `ReplayTouchTimelineAnalyzer.ts`。
5. **Tier 6（网络预测与回滚）严重低估 Rust 现有资产**：
   - 原报告称“Multiplayer 完成度 15%，仅有数据结构定义”，与代码库严重不符。
   - `crates/rc_physics_sim/src/reconciler.rs` 与 `crates/rocketsim_wasm/src/exports/reconciler_exports.rs` 已经实现了 `CONTROLS_RING_CAPACITY`、`ReconcileControlEntryPod`、`ReconcileOutcomePod`、`_physics_reconcileRollbackToTick` 和 `step_reconciler_range`。因此 `t6-25`~`t6-27` 已在 Rust 侧完成，真正缺口是 `src/network/`（95 个 TS 文件）全面切换消费这些 WASM 导出并下线 TS 冗余逻辑。

---

## 3. 修正后的精简并行分支规划（16 分支，基于 `b26d328`）

将原虚胖的 32 分支精简为 **4 个 Tier、16 个高内聚正交分支**：

### Tier 3：音频引擎全下沉与碰撞抖动根治（5 分支）
1. `z/agy/rc/rust/t3-01-ball-contact-rolling-and-debounce`
   - 同时修复 `director_impact_events.rs`（live 路径）与 `incident_classifier.rs`：删除 `ev.speed` fallback；接入 `ev.custom_float`（`dir_dot < 0.15` 浅角抑制）；将地板/墙面碰撞冷却统一提升至 `≥ 60ms`（8 ticks）；加入地面带球低相对速度抑制；补充覆盖高速平滚、浅角擦碰、带球、60ms 防抖的 Rust 单元测试。
2. `z/agy/rc/rust/t3-02-wasm-car-cue-bridge-cutover`
   - 在 `WasmAudioDirectorBridge.ts` 接入 `AUDIO_CUE_CARS_WORD_OFFSET = 22`，读取 `AudioFrameCuePod.cars[8]` 并直接映射至 WebAudio `AudioParam`。
3. `z/agy/rc/rust/t3-03-retire-ts-turtle-modem-synth`
   - 下线 `TurtleModemEngineSynth.ts`、`TurtleModemGranularVoiceGraph.ts`、`turtleModemSynthAcousticPipeline.ts` 中的 TS 动力学与切片计算，将 `VehicleAudioManager.ts` 与 `AudioMenuPreviewPlayer.ts` 改造为纯 Dumb Driver。
4. `z/agy/rc/rust/t3-04-stadium-anthem-thin-driver`
   - 对齐 `StadiumBusBlendPod` 与 `wasmAudioOneShotDispatcher.ts`，清理 TS 侧残留混音与衰减计算。
5. `z/agy/rc/rust/t3-05-audio-radar-and-e2e-verify`
   - 音频子系统体积/边界/无死代码清理，确保 `pnpm check` 与音频行为验收通过。

### Tier 4：轨迹预测 TS 侧零拷贝收口（2 分支）
6. `z/agy/rc/rust/t4-01-trajectory-ts-zero-copy-audit`
   - 核验 Three.js 侧是否已 100% 直读 `trajectory_exports.rs` 共享内存指针，清理残留 TS 几何包装代码。
7. `z/agy/rc/rust/t4-02-trajectory-120hz-benchmark`
   - 建立 120Hz 连续采样与零拷贝渲染基准测试。

### Tier 5：UE3 `.replay` NetStream Rust 化（5 分支）
8. `z/agy/rc/rust/t5-01-ue3-bitstream-reader`：在 `rc_replay_codec` 中实现 UE3 无锁 `BitReader` 与头部元数据解码。
9. `z/agy/rc/rust/t5-02-ue3-actor-channel-parser`：实现 UE3 NetStream Actor/Component 属性通道解码器。
10. `z/agy/rc/rust/t5-03-touch-timeline-analyzer-rust`：将 `src/replay/ReplayTouchTimelineAnalyzer.ts` 下沉至 Rust。
11. `z/agy/rc/rust/t5-04-replay-wasm-streaming-bridge`：实现大 `.replay` 文件流式 WASM 解码桥接。
12. `z/agy/rc/rust/t5-05-retire-ts-netstream-parsers`：分批 `git rm` `src/replay/`（106 TS 文件）中已被替代的位流解析器。

### Tier 6：联机回滚全面切换至 `rc_physics_sim::reconciler`（4 分支）
13. `z/agy/rc/rust/t6-01-ts-network-reconciler-cutover`：让 `src/network/` 全面切换消费 `_physics_reconcileRollbackToTick` 与 `step_reconciler_range`，移除 TS 侧重复状态回滚推导。
14. `z/agy/rc/rust/t6-02-rtt-and-clock-sync-rust`：将 RTT 抖动平滑与物理时钟同步下沉至 `rc_physics_sim`。
15. `z/agy/rc/rust/t6-03-room-state-pod-codec`：联机房间 8 车状态紧凑二进制协议收口。
16. `z/agy/rc/rust/t6-04-multiplayer-soak-and-final-radar`：双世界/多车浸泡测试与全局 `pnpm check` 终局验收。
