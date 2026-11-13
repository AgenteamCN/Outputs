# RocketCompanion (`b88102a`) 架构重构与并行分支全景评估报告

> **基准 Commit**：`b88102a60527e50892564b15b7ff2e1acee401c0`（已合并 `origin/alpha` 最新特性，`pnpm run check` 全绿：1316 TS 文件 / 69 自定义 Rust 文件，0 Radar Error，0 `tsc` Error）  
> **评估范围**：Tier 3–6 Rust 下沉计划（16 分支版） + 主干 Next-Era Multiplayer 架构蓝图（Track A–F 22 分支版） + Drawing Sandbox / WebRTC Voice / Dynamic Training Pack (RCD) / CSR Follow Host 最新业务需求。

---

## 一、 核心结论

1. **原 16 分支 Rust 下沉计划（Tier 3–6）整体方向正确，但直接照搬会与 Next-Era Multiplayer 发生重复建设**：
   - **Tier 3（音频）** 与 **Tier 4（轨迹）** 与主干新需求几乎正交，可立即开工，且 Tier 4 可进一步合并为 **1 个分支**。
   - **Tier 5（UE3 `.replay` 解析）** 在 `b88102a` 中尚未在 TS 侧接入 `rc_replay_codec`（TS 侧仍有 `src/replay/rlrc/parser/` 21 个文件在主线程解析），参考 `boxcars` 下沉完全可行，建议精简为 **3 个分支**。
   - **Tier 6（联机回滚与协议）** 在 `b88102a` 中 Rust 侧 `reconciler_exports.rs` 已就绪，但 TS 侧 `src/network/` **0 调用**；若先按旧 `MatchDomainState` 协议接回滚，随后又要配合 `RoomReplicationBus` 推翻重写，属于典型浪费。**必须将 Tier 6 与 Track D/E/F 合并统一设计**。
2. **主干 Track A–F（22 分支）与您的最新需求存在 4 处重要需求裁剪与 4 处仓库红线约束**，本报告已全部对齐修正。
3. **最终建议**：将两大规划与新需求去重融合，拆分为 **4 个 Wave、共 16 个高内聚低耦合并行分支**（全部以 `z/agy/rc/rust/*` 命名），既能保证无文件冲突的高效并行，又避免先做后改的返工。

---

## 二、 `b88102a` 代码库深度扫描核验（Ground-Truth Audit）

### 1. Tier 3：音频引擎（100% 核验属实，两条碰撞路径必须同时修）
- **Live 路径漏修确认**：`crates/rocketsim_wasm/src/lib.rs` 绑定运行的是 `AudioDirectorInstance`（`crates/rc_audio_director/src/director_impact_events.rs::handle_ball_world_or_post_hit`），而非 `incident_classifier.rs::process_physics_incidents`。
  - `director_impact_events.rs` 存在：`normal_speed = ev.normal_rel_vel.max(ev.impulse)`、硬编码 `< 5` ticks（41.6ms）冷却、完全未使用 `crates/rc_physics_sim/src/step_and_events.rs` 已写入 `custom_float` 的 `dir_dot`（$\cos\theta$）。
  - `incident_classifier.rs` 存在：`ev.normal_rel_vel <= 0.0` 时错误 fallback 到 `ev.speed.abs()`。
- **半下沉现状确认**：`src/audio/WasmAudioDirectorBridge.ts` 仅定义了 `AUDIO_CUE_CARS_WORD_OFFSET = 22`，全仓库无任何读取代码；而 Rust `AudioFrameCuePod`（1112B）已包含 `cars: [CarAcousticLoopPod; 8]` 与 `stadium_bus: StadiumBusBlendPod`（16B）。
- **TS 侧清理目标确认**：`src/audio/`（26 个文件）中**不存在** `RemoteEngineAudioLodController.ts` 与 `TurtleModemDrivetrainModel.ts`。实际需瘦身/下线 TS 动力学计算的是：`TurtleModemEngineSynth.ts` (472L)、`TurtleModemGranularVoiceGraph.ts` (525L)、`turtleModemSynthAcousticPipeline.ts` (522L)、`VehicleAudioManager.ts` (459L)、`AudioMenuPreviewPlayer.ts` (282L) 与 `wasmAudioOneShotDispatcher.ts` (306L)。
- **Golden Sample 自动化验收方案**：由于物理与 `AudioDirectorInstance` 均确定性，可在测试脚本中固定一组 Bot 对战输入序列（如 30 秒 2v2），分别记录旧版 TS 合成器与新版 WASM `AudioFrameCuePod` + `AudioOneShotSlotPod` 的 **AudioParam 目标曲线（pitch/gain/filter）** 与 **OneShot 触发日志（tick, event_type, sub_type, final_gain, playback_rate）** 进行对齐比对，无需对比声波波形。

### 2. Tier 4：轨迹预测（WASM 已接入，只需审计收口与基准）
- 扫描确认 `src/entities/trajectory/BallTrajectoryTubeRenderer.ts` 与 `src/entities/ReachabilityVisualizer.ts` **已经**调用 `_physics_buildTrajectoryMesh` / `_physics_getTrajectoryMeshHeaderPtr` 等 `trajectory_exports.rs` 零拷贝接口。
- 无需拆 2 个分支，合并为 **1 个分支**，重点审计 `ballTrajectoryGeometryPipeline.ts` (256L) 与 `BallTrajectorySamplingController.ts` (469L) 是否仍有残留 TS 几何拷贝，并补齐 120Hz 连续采样基准测试。

### 3. Tier 5：UE3 `.replay` 解析（Rust 导出尚未使用，TS 侧 21 个文件待下沉）
- `crates/rocketsim_wasm/src/exports/replay_exports.rs` 已导出 `_physics_decodeImportedReplayStream`、`_physics_encodeReplayExportStream`、`_physics_seekReplayTime` 等接口，但 **TS 侧目前 0 调用**。
- `src/replay/rlrc/parser/` 下仍有 **21 个 TS 文件**（`RLRCBitReader.ts`、`RLRCNetStreamBitDecoder.ts`、`RLRCClassNetCacheBuilder.ts`、`RLRCPropertyAttributeDecoder.ts`、`RLRCUnknownPropertyBitProber.ts` 等）在主线程解析 UE3 `.replay`，外加 `ReplayTouchTimelineAnalyzer.ts` (457L)。
- 结合 `nickbabcock/boxcars` 思路与现有 `RLRCUnknownPropertyBitProber` 动态探测机制下沉至 `rc_replay_codec` 完全可行，且与网络/沙盒完全正交。

### 4. Tier 6 & Next-Era Multiplayer（必须合并设计，避免双重返工）
- `crates/rocketsim_wasm/src/exports/reconciler_exports.rs` 已完整导出 `_physics_reconcileRollbackToTick`、`_physics_stepReconcilerRange`、`_physics_getReconcilerControlsRingPtr`，但 **`src/network/` 目前 0 调用**，`reconcilerRollbackPipeline.ts` 与 `ReconcilerRollbackStepper.ts` 仍在 TS 侧运行。
- `src/network/server/match/` 中的 `MatchDomainState.ts` (330L)、`MatchPhaseLifecycleSubController.ts` (477L)、`MatchSystemControlSubController.ts` (439L) 确实存在“Match 状态机与练车/暂停同步职责耦合”的问题。

### 5. `scripts/radar.ts` 与 `src/ui-core/AGENTS.md` 硬性红线
1. **UI 分层目录禁令**：`src/ui-core/` 强制按 `config -> tokens -> stores -> primitives -> layout -> compounds -> navigation -> (hud|charts|viewport|loading) -> (panels|recipes) -> ui-tuner` 单向分层，**严禁**新建 `src/ui-core/telestrator/`、`src/ui-core/audio/`、`src/ui-core/training/`、`src/ui-core/overlay/` 目录。
2. **Transport 隔离禁令 (`[TRANSPORT ISOLATION VIOLATION]`)**：`src/network/transport/` 禁止 import `src/network/` 与 `src/ui-core/config/` 之外的任何模块（绝不能反向 import `src/audio/voice/`）。
3. **代码卫生**：0 `index.ts` 重导出、0 中文注释/字符串/日志、0 Emoji、严禁 `soccer`/`soccar` 命名；单文件软上限 `400L / 12KB`，硬上限 `800L / 18KB`。

---

## 三、 新需求专项技术评估与待定夺清单

### 1. Drawing Sandbox（3D 战术画笔 + 视角锁定）
- **贴地/贴墙实现（CMF vs GLB）**：建议对**低面数碰撞 Proxy / 解析场馆几何**（`Z=0` 草皮 + 侧壁/四分之一圆弧/天花板解析求交 + 球门框网格 Raycast）做射线检测，比直接对高面数渲染 GLB 求交性能高一个数量级，且与物理边界 100% 贴合。
- **屏幕空间粗细与视角同步一致性**：笔画以**世界空间 3D 锚点序列**存储（支持实线 / 虚线 / 箭头、3 种 Preset、100%/70%/50% 透明度、6 种预设色：深灰/白/红/黄/亮绿/暗绿、`Ctrl/Cmd+Z` 撤销、一键清空），渲染层采用屏幕空间固定像素宽度的 Ribbon 着色器。这样在相同相机 Pose 下，各端视觉粗细与形状完全一致；自由移动视角后笔画稳固锚定在场地上。
- **网络同步（Hash + ACK）**：建议绘制结束时直接发送紧凑的 Float32 关键点序列（经过距离采样与简化后通常仅数百字节），并附带 `strokeId + fnv1aHash` 做状态对账与 ACK 重传，比“先发 Hash 等 ACK 再发笔画”减少 1 个 RTT 延迟。
- **Camera Direction Adjust（持久偏转 + 互斥锁）**：在 `CameraManager.ts` / `CleanPovCameraKernel.ts` 增加持久 `povAngularOffset`（Yaw/Pitch），支持 `1–6` 车辆视角、`7` 自由视角、两侧状态指示器、一键 Restore 归零（按 `1–6` 自动归零）；与画笔状态严格互斥（转视角时禁画，画画时禁转视角）。
- **Floating Window 形态限制（需 UI 团队配合）**：当前 `floatingWindowRegistry.tsx` 与 `Panel.tsx` 默认渲染 Header/Title 与标准横向宽度，尚不支持“无 Title、无 Footer、超窄纵向纯图标竖栏 + 自适应左右侧 Hover Bubble 提示”。**先行方案**：先在 Developer Settings 挂载入口并实现一个紧凑纵向工具栏窗口（预设 3 套粗细先给默认值），将 Floating Window Schema 扩展（`hideHeader` / `hideFooter` / `verticalCompact` / `hoverBubble`）列为 UI 团队对接项。

### 2. WebRTC Audio Channel（原生语音通话）
- **需求裁剪**：按最新要求，**移除** 3D 空间语音衰减（`PannerNode`）与车顶说话指示器，保持原声清晰通话。
- **降噪方案评估与库推荐**：
  - **输入降噪**：优先开启浏览器原生 WebRTC APM（`echoCancellation: true, noiseSuppression: true, autoGainControl: true`），零包体积开销；若在复杂机械键盘/风扇环境下仍不够用，推荐直接引入成熟的 **`@sapphi-red/web-noise-suppressor`（基于 RNNoise WASM + AudioWorklet，约 100KB）**，切勿手写降噪算法。
  - **输出降噪**：对端输入端已做过一次 APM；若本地仍需对远端音轨二次降噪，可将远端 `MediaStream` 接入同一套 RNNoise `AudioWorkletNode` 后再接 `GainNode`（支持 `0% ~ 250%` 增益，并串联轻量 `DynamicsCompressorNode` 防止 250% 增益爆音）。
- **Player ID 音量与权限策略**：
  - 默认在 **Developer Settings** 中处于**关闭**状态，绝不在启动时静默请求权限；每次用户手动开启开关时，显式调用 `navigator.mediaDevices.getUserMedia` 请求麦克风权限。
  - 音量映射优先绑定账号系统（`CloudProfileSync` / `credentialBundleTypes`）的持久化 `PlayerId`（匿名/离线时 fallback 到房间分配的 `PeerId`），并作为房间同步状态的一部分。

### 3. Dynamic Training Pack（RCD Pack）
- **成功判定器**：大幅简化，仅保留 **Goal / Timeout** 两种判定。
- **数据架构建议（避免链表指针冲突）**：
  - 将从 Replay `intervene` 前截取的 4 秒完整状态保存为独立的 **`StateSession`**（含完整帧序列、初始快照、来源 Replay 元数据），存放在独立的 `captured-from-replay` 存储区（IndexedDB）。
  - **关键设计**：不要把 `prevSessionId / nextSessionId` 硬编码在 `StateSession` 内部，而是放在 **`RCDPack` 的有序引用清单中**（或 Pack 内的链表节点）。这样同一个 `StateSession` 才能被安全地 `Duplicate to` 多个 Pack 而不发生指针覆盖；执行 `Delete` 时提供二次确认（Double Check）并按引用计数清理孤立 Session。
- **Intervene 接管时序**：倒计时期间严格播放原 Replay 状态；在真正接收用户输入的一瞬间（或提前 0.5s 静默预热车辆悬挂/模型）执行每辆车的个性化接管策略（`disappear` / `as previous` / 指定 Bot Model 接管）。
- **待产品/团队定夺的开放问题**：
  1. 从零基于静态发球录制多车动态动作（逐车叠加录制时序）的交互流程与编辑器状态机；
  2. Online Mode 下多端对“接管哪一辆车”的权限申请、冲突仲裁与实时交换协议；
  3. RCD Pack 中是否允许对可控车的 Spawn Position 或球位置施加极小 Variation（与幂等性之间的取舍）。

### 4. Next-Era Multiplayer（统一总线 + CSR Follow Host）
- **动态模式标志位与带宽裁剪**：在 `RoomReplicationBus` 帧头引入 `ActivityModeFlags`，使同一底层总线按模式裁剪载荷：
  - `Match`：仅同步物理 State + User Input（跳过画笔通道）；GoalReplay 期间继续复用 Jump 按键边沿锁定技巧；
  - `Training`：省略比分与比赛倒计时字段；
  - `Replay / Sandbox`：按需挂载稀疏帧（Frozen + 2Hz Ping-Pong）与画笔同步通道。
- **Completely Follow Host（CSR 模式）**：Client 仅接收状态流与 Host 相机/活动摘要，按本地屏幕分辨率与渲染比例进行 Client-Side Rendering；在 Notification 区域常驻最低优先级提示条，支持一键开启/退出完全跟随 Host。

---

## 四、 推荐并行分支拆分方案（4 Wave / 16 分支）

所有分支均基于 `b88102a`（或 `origin/alpha` 最新基线）拉出，命名统一为 `z/agy/rc/rust/*`，各分支修改文件高度正交，可将合并冲突降至最低：

### Wave 1：零阻塞并行启动（7 分支，互不冲突）
| 分支名 | 核心职责与落位 | 关键验收标准 |
| :--- | :--- | :--- |
| `z/agy/rc/rust/t3-01-ball-contact-rolling-and-debounce` | 同时修复 `director_impact_events.rs` 与 `incident_classifier.rs`：移除 `ev.speed` / `impulse` 误用、接入 `custom_float` (`dir_dot < 0.15` 浅角抑制)、地面/墙面冷却 ≥ 60ms（8 ticks）、地面带球低速抑制、补齐 Rust 单测 | `cargo test -p rc_audio_director` 全绿 |
| `z/agy/rc/rust/t3-02-wasm-car-cue-and-bus-cutover` | 在 `WasmAudioDirectorBridge.ts` 接入 `AUDIO_CUE_CARS_WORD_OFFSET = 22` 读取 `cars[8]`，对齐 `StadiumBusBlendPod` 与 `wasmAudioOneShotDispatcher.ts`，清理 TS 混音衰减计算 | WASM 8 车环绕音频与总线参数直驱 WebAudio |
| `z/agy/rc/rust/t3-03-retire-ts-turtle-modem-and-verify` | 下线 `TurtleModemEngineSynth.ts`、`TurtleModemGranularVoiceGraph.ts`、`turtleModemSynthAcousticPipeline.ts` TS 动力学与切片计算；改造 `VehicleAudioManager.ts` & `AudioMenuPreviewPlayer.ts` 为 Dumb Driver；编写 Golden Sample 对比测试 | 音频行为对齐 Golden Sample，`pnpm check` 通过 |
| `z/agy/rc/rust/t4-01-trajectory-audit-and-benchmark` | 审计 `src/entities/trajectory/` 零拷贝直读完整性，清理残留 TS 几何包装，建立 120Hz 连续采样与渲染性能基准 | 0 TS 侧轨迹重计算，`pnpm check` 通过 |
| `z/agy/rc/rust/sandbox-01-drawing-telestrator-core` | 新建 Drawing Sandbox（6 车场景）；`src/entities/telestrator/` 实现场地 Raycast、Screen-space Ribbon、实线/虚线/箭头、Ctrl+Z；`src/camera/` 实现 `1-7` 视角 + Direction Adjust 持久偏转与互斥；Developer Settings 挂载入口与紧凑竖栏工具面板 | 沙盒可进入、画笔与视角互斥正常、`pnpm check` 通过 |
| `z/agy/rc/rust/voice-01-webrtc-native-channel` | `src/audio/voice/` 实现 `IVoiceAudioBridge` 与原生 WebRTC 音轨桥接（从上层注入 `RTCPeerConnection`，不碰 `transport/` 隔离红线）；支持 Player ID `0-250%` 音量、输入/输出降噪选项；Developer Settings 默认关闭，开启时显式请求权限 | 满足 `[TRANSPORT ISOLATION]`，`pnpm check` 通过 |
| `z/agy/rc/rust/training-01-rcd-pack-and-session-capture` | `src/game/modes/dynamic-training/` 实现 RCD Pack Schema、Replay `intervene` 前 4s `StateSession` 独立捕获与 IndexedDB 存取、Goal/Timeout 判定、接管瞬间/提前 0.5s 模型切换、Pack 预览/Move/Duplicate/Delete(Double Check) | 可从 Replay 捕获并运行 RCD Pack，`pnpm check` 通过 |

### Wave 2：UE3 `.replay` NetStream Rust 化（3 分支，按序收敛）
| 分支名 | 核心职责与落位 |
| :--- | :--- |
| `z/agy/rc/rust/t5-01-ue3-bitstream-and-actor-parser` | 参考 `boxcars` 在 `crates/rc_replay_codec` 实现 UE3 BitReader、头部解码与 Actor/Property 通道解码器 |
| `z/agy/rc/rust/t5-02-touch-timeline-and-streaming-bridge` | 将 `ReplayTouchTimelineAnalyzer.ts` 下沉至 Rust，并打通大文件 `.replay` 流式 WASM 解码桥接 |
| `z/agy/rc/rust/t5-03-retire-ts-netstream-parsers` | 分批 `git rm` `src/replay/rlrc/parser/` 中已被 Rust 替代的 21+ TS 位流解析器并验证回放基线 |

### Wave 3：Next-Era Multiplayer 协议与 WASM Reconciler（3 分支，先锁协议再并行）
| 分支名 | 核心职责与落位 |
| :--- | :--- |
| `z/agy/rc/rust/net-01-replication-bus-and-pod-codec` | 锁定 `RoomReplicationBus` 双模协议（Continuous 120Hz vs Sparse 2Hz Ping-Pong）、`ActivityModeFlags` 带宽裁剪、`ROOM_ACTIVITY_SWITCH` 信令与 8 车状态二进制协议 |
| `z/agy/rc/rust/net-02-wasm-reconciler-and-clock-sync` | 让 `src/network/client/` 全面切换消费 `_physics_reconcileRollbackToTick` 与 `_physics_stepReconcilerRange`，移除 TS 重复回滚推导，并将 RTT 平滑与时钟同步下沉至 `rc_physics_sim` |
| `z/agy/rc/rust/net-03-webrtc-dual-channel-and-sync-router` | `DataChannelTransport.ts` 落地 Unreliable/Reliable 双信道分流；`src/network/sync/` 实现实体级权威路由与画笔 Stroke Hash/ACK 校验通道 |

### Wave 4：上层 Kernel 拆解、CSR 跟随与终局验收（3 分支）
| 分支名 | 核心职责与落位 |
| :--- | :--- |
| `z/agy/rc/rust/net-04-room-orchestrator-and-kernels` | 解耦 `AuthoritativeServerHost` 与 `MatchDomainState`，落地可热插拔的 `FreeplayKernel`、`MatchKernel`、`SandboxKernel` |
| `z/agy/rc/rust/net-05-spectator-csr-and-intervene-handoff` | 多人 Replay 观战室 + `Completely Follow Host` (CSR) 模式与 Notification 常驻最低优先级提示 + 4-3-2-1-0 接管倒计时与多端交接 |
| `z/agy/rc/rust/net-06-multiplayer-soak-and-final-radar` | 扩展 `runMultiplayerRoomRepro.mjs` 覆盖模式热切换与 CSR 跟随压测；执行 `pnpm run check`、`radar:audit` 与 `test:baseline` 终局验收 |

---

## 五、 建议立即启动顺序

1. **第一批立即起跑**：`Wave 1` 的 7 个分支（互相 0 文件冲突，可直接并行开发）。
2. **同步对齐**：由 UI 团队同步评估 Floating Window 竖栏无标题/无页脚/Hover Bubble 扩展；产品侧确认 RCD Pack 的“从零多车录制”与“联机接管权交换”规则。
3. **待您确认本方案后**：即可按上述分支列表直接拉取对应 `z/agy/rc/rust/*` 分支开始编码落地与提交。
