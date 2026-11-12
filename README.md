# Rocket Companion (RC) 下一代架构演进与高并行 50 分支开发计划书

> **基准代码 Commit**: `origin/alpha` (`f5e8c9f`)  
> **分支命名规范**: `z/agy/rc/branch<01-50>-<kebab-case-feature-name>`  
> **设计原则**: 零历史包袱、模块强解耦、文件规模遵循 Radar 软限制（<400L/12KB）与硬限制（<800L/18KB）、无循环引用、去中心化并行推行。

---

## 一、核心技术决策与架构重构研判

### 1. 状态同步层（State Synchronization）与 Match 逻辑彻底解耦
- **现状剖析**：当前多人联机栈与物理权威计算深度绑定在 `MatchPhaseLifecycleSubController` 与 `MatchDomainState` 的重度比赛状态机中（开球倒计时、比分统计、进球庆祝重置、加时赛判定等）。
- **重构研判**：
  - **Match 本质只是一种上层游戏模式**：将 Match 裁剪掉计分、倒计时与进球重置后，直接退化为高吞吐、零打扰的多人自由练车场（Multiplayer Freeplay / Lobby Playground）。
  - **State Synchronization 提升为独立底层总线**：无论上层是自由练车、比赛对战、Replay 观战还是训练包练习，底层的核心都是“实体状态的三维空间分发与权威插值”。
  - **双模同步协议（Continuous vs Sparse Ping-Pong）**：
    1. **连续广播模式（Continuous Broadcasting）**：在自由练车、实时对战或接管后实时模拟时，以 60Hz/120Hz 权威步进广播 Hermite 紧凑帧（`globalPhysicsSnapshotSectionCodec`）。
    2. **稀疏/暂停低带宽模式（Sparse Present / Ping-Pong Keepalive）**：在 Replay 暂停复盘、导师讲解停帧或训练包等待介入时，**彻底冻结 120Hz 连续物理广播**，仅通过一次性 `STATE_PAUSED_ACK` 锁定帧号与状态摘要。停帧期间仅发送 16 字节轻量 RTT 心跳包（Ping-Pong 空包），带宽消耗从数百 KB/s 暴跌至 < 0.5 KB/s，杜绝无谓的带宽浪费与 CPU 序列化损耗。

### 2. Dynamic Training Pack 与 State Synchronization 的完全解耦
- **解耦逻辑**：Dynamic Training Pack 绝不应与网络同步层强绑定。
  - **数据本质**：Dynamic Training Pack 是一段包含特定车辆/球体初始运动切片（RLRC Replay Track Slice）及“介入触发点（Intervention Takeover Frame）”的静态资产结构（`.rctrain` / `.rlrc`）。
  - **单机先行**：Dynamic Training Pack 的切片提取器（Extractor）、本地单人运行器（Runner）、速度/角度抖动发生器（Variation Math）和进球/防守命中判定器（Judge Pipeline）可以在单机环境下独立闭环开发与单元测试。
  - **顺水推舟**：当 State Synchronization 基础就绪后，多人联机训练包只需挂接“广播加载切片 -> 倒计时 -> 选点接管”这三条信令，无需重写任何训练包核心逻辑。因此，Training Pack 团队与 State Sync 团队可以**100% 并行推进，零互锁阻断**。

### 3. 画笔（Telestrator）独立工程化方案
- **真 3D 空间场馆吸附（Surface Decal Raycasting）**：摒弃任何 2D Canvas 屏幕画线方案，彻底杜绝不同分辨率、视口宽高比与镜头旋转造成的线条漂移错位。
- **空间纵深投影（Aerial Trajectory Depth Drop Stems）**：空间挑球和空中航迹绘制时，沿航迹采样点向下投射半透明垂直虚线直达 $Z=0$ 草皮，并在草皮处投影阴影落点圈，形成直观的三维空间深度暗示。
- **独立分支开发**：画笔完全处于独立的 `src/entities/telestrator/` 和 `src/ui-core/telestrator/`，数据结构仅依赖 Three.js 世界坐标系，后续与任何联机/回放分支联合无障碍。

### 4. 纯原生 WebRTC 语音通话（Voice Chat）与 Rust 零等待策略
- **绝不干等 Rust**：Rust 团队目前音频管线尚在攻坚。前端利用浏览器原生成熟的 WebRTC P2P 音频轨道（`addTrack(MediaStreamTrack)`）即可获得顶级 Opus 编码、动态抖动缓冲（Jitter Buffer）、声学回声消除（AEC）与降噪能力。
- **抽象防腐层（VoiceAudioBridge）**：提供统一的抽象音频接口，当前挂载 WebRTC Native Audio；未来 Rust 原生音频就绪后，底层无缝热插拔，上层业务层零改动。

---

## 二、50 个高并行独立工作分支全景图 (branch01 - branch50)

所有分支均从 `origin/alpha` 最新 Commit 直接分叉，严格保持物理目录和代码职责隔离，确保多 Agent / 多人协同推进时零合并冲突。

```
                                  [origin/alpha]
                                        │
    ┌──────────────┬──────────────┬─────┴────────┬──────────────┬──────────────┐
    ▼              ▼              ▼              ▼              ▼              ▼
[Track 1]      [Track 2]      [Track 3]      [Track 4]      [Track 5]      [Track 6]
State Sync &   Multiplayer    Dynamic Train  Replay Spec &  3D Telestrator WebRTC Voice &
Protocol       Freeplay       Pack Engine    Takeover       & Free Cam     Integration
(branch01-10)  (branch11-18)  (branch19-28)  (branch29-36)  (branch37-44)  (branch45-50)
```

### Track 1: 状态同步底层总线与双模传输协议 (Branches 01–10)
*核心目标：沉淀与 Match 状态机解耦的轻量级实体状态广播总线，支持 Continuous 连续传输与 Sparse Ping-Pong 低带宽心跳。*

| 分支编号 | Git 分支名 | 目标核心模块 / 文件路径 | 职责与产出物 | 独立验证命令 |
| :--- | :--- | :--- | :--- | :--- |
| **branch01** | `z/agy/rc/branch01-state-sync-types` | `src/network/sync/stateSyncProtocolTypes.ts` | 定义全局 StateSync 双模协议规范（Continuous / Sparse / Keyframe / PingPong 报文定义） | `pnpm check` |
| **branch02** | `z/agy/rc/branch02-ping-pong-heartbeat` | `src/network/sync/SparseHeartbeatPacer.ts` | 实现停顿/暂停态 16 字节 Ping-Pong RTT 探测与单调时钟漂移计算器 | `pnpm check` |
| **branch03** | `z/agy/rc/branch03-continuous-snapshot-streamer` | `src/network/sync/ContinuousSnapshotStreamer.ts` | 剥离 Match 依赖的 60Hz/120Hz 高吞吐连续位姿压缩序列化器 | `pnpm check` |
| **branch04** | `z/agy/rc/branch04-hermite-trajectory-interpolator` | `src/network/sync/HermiteTrajectoryInterpolator.ts` | 客户端本地 120Hz 三次 Hermite 样条位置与 Slerp 姿态连续平滑插值器 | `pnpm check` |
| **branch05** | `z/agy/rc/branch05-sparse-keyframe-broadcaster` | `src/network/sync/SparseKeyframeBroadcaster.ts` | 稀疏帧触发器：Scrub 拖动、Seek、暂停帧瞬时 Keyframe + ACK 确认交付 | `pnpm check` |
| **branch06** | `z/agy/rc/branch06-bandwidth-budget-pacer` | `src/network/sync/BandwidthBudgetPacer.ts` | 自适应动态码率调节器（网络拥塞与高延迟下平滑降频至 30Hz/15Hz） | `pnpm check` |
| **branch07** | `z/agy/rc/branch07-sync-authority-router` | `src/network/sync/SyncAuthorityRouter.ts` | 实体级物理权威路由表（区分 Host 权威、Peer 权威与回放录像只读实体） | `pnpm check` |
| **branch08** | `z/agy/rc/branch08-delta-bitmask-pipeline` | `src/network/sync/deltaBitmaskPipeline.ts` | 静止实体状态抑制与增量变化位图编码（仅传输发生运动变化的车辆/球） | `pnpm check` |
| **branch09** | `z/agy/rc/branch09-sync-channel-multiplexer` | `src/network/sync/SyncChannelMultiplexer.ts` | WebRTC DataChannel 多路复用分流器（控制信令信道 vs 高频数据信道） | `pnpm check` |
| **branch10** | `z/agy/rc/branch10-state-sync-e2e-bench` | `src/network/sync/tests/StateSyncHarness.test.ts` | 独立无依赖的 StateSync 吞吐、丢包重传与 RTT 自动化基准测试套件 | `node scripts/radar.ts && pnpm typecheck` |

---

### Track 2: 精简版 Match 与多人自由练车场 (Branches 11–18)
*核心目标：将 Match 系统裁剪解耦为极简多人自由练车模式，关闭进球中断，挂载 60s/90s COUNTUP_FIFO 滚动回放池。*

| 分支编号 | Git 分支名 | 目标核心模块 / 文件路径 | 职责与产出物 | 独立验证命令 |
| :--- | :--- | :--- | :--- | :--- |
| **branch11** | `z/agy/rc/branch11-freeplay-match-options` | `src/network/server/match/MatchDomainState.ts` | 增加 `freeplayModeActive`、`disableGoalReset`、`disableMatchClock` 策略配置项 | `pnpm check` |
| **branch12** | `z/agy/rc/branch12-lobby-physics-streamline` | `src/network/server/match/matchLifecyclePipeline.ts` | 重构 `configureLobbyPhysicsArena`，移除计分触发器与比赛锁定，开启无限气与自由穿行 | `pnpm check` |
| **branch13** | `z/agy/rc/branch13-fifo-ringbuffer-wiring` | `src/network/server/match/MatchPhaseLifecycleSubController.ts` | 自由练车下强制配置 `REPLAY_BUFFER_MODES.COUNTUP_FIFO`（7200/10800 ticks） | `pnpm check` |
| **branch14** | `z/agy/rc/branch14-goal-reset-bypass` | `src/network/server/match/MatchGoalReplaySubController.ts` | 当 `disableGoalReset=true` 时，进球仅触发音效/特效，球体自动反弹或回位，不冻结物理 | `pnpm check` |
| **branch15** | `z/agy/rc/branch15-multiplayer-ball-reset-signal` | `src/network/server/match/MatchCommandQueueSubController.ts` | 增加多人一键重置球（`BALL_RESET`）或开球点轮换信令分发机制 | `pnpm check` |
| **branch16** | `z/agy/rc/branch16-ghost-car-coordinator-tuning` | `src/network/server/LobbyGhostCarCoordinator.ts` | 调优自由练车多车碰撞模型（支持 Ghost Passthrough 半透明与物理碰撞热切换） | `pnpm check` |
| **branch17** | `z/agy/rc/branch17-multiplayer-freeplay-mode` | `src/game/modes/multiplayer-freeplay/MultiplayerFreeplayMode.ts` | 封装上层独立模式 `MultiplayerFreeplayMode`，对外暴露统一开房与入房生命周期 | `pnpm check` |
| **branch18** | `z/agy/rc/branch18-repro-multiplayer-freeplay` | `scripts/runMultiplayerRoomRepro.mjs` | 在 Headless Chrome 自动化回归套件中增加 `--mode=freeplay` 压测验证入口 | `node scripts/runMultiplayerRoomRepro.mjs --mode=freeplay --speed=2` |

---

### Track 3: Dynamic Training Pack 独立引擎与资产管线 (Branches 19–28)
*核心目标：脱离网络层在单机上完整跑通 Dynamic Pack 切片提取、介入点定义、空间容差判定与变体微调。*

| 分支编号 | Git 分支名 | 目标核心模块 / 文件路径 | 职责与产出物 | 独立验证命令 |
| :--- | :--- | :--- | :--- | :--- |
| **branch19** | `z/agy/rc/branch19-dynamic-pack-schema` | `src/game/modes/dynamic-training/dynamicTrainingSchema.ts` | 定义 Dynamic Training Pack 紧凑二进制格式（包含 BaseReplayRef、ShotClipList） | `pnpm check` |
| **branch20** | `z/agy/rc/branch20-replay-slice-extractor` | `src/replay/ReplayBufferSliceExtractor.ts` | 增强现有切片提取器，支持根据 Tick 区间截取车辆、球体运动轨迹与刚体参数 | `pnpm check` |
| **branch21** | `z/agy/rc/branch21-shot-intervene-metadata` | `src/game/modes/dynamic-training/ShotInterveneMetadata.ts` | 定义介入点元数据（InterveneTick、TakeoverCarId、LeadInSeconds、SuccessTrigger） | `pnpm check` |
| **branch22** | `z/agy/rc/branch22-shot-variation-engine` | `src/game/modes/dynamic-training/DynamicShotVariationMath.ts` | 动态射门变体引擎：支持对切片初始速度（±5%）、角度（±3°）、自旋做确定性微扰 | `pnpm check` |
| **branch23** | `z/agy/rc/branch23-shot-success-evaluator` | `src/game/modes/dynamic-training/DynamicSuccessEvaluator.ts` | 多维度成功判定器（进球、门线扑救、停球入框、飞行限时、最高球速限制） | `pnpm check` |
| **branch24** | `z/agy/rc/branch24-dynamic-training-runner` | `src/game/modes/dynamic-training/DynamicTrainingMode.ts` | 实现单人 Dynamic Training Pack 核心模式控制器（切片回放 -> 倒计时 -> 接管） | `pnpm check` |
| **branch25** | `z/agy/rc/branch25-dynamic-shot-editor-pipeline` | `src/game/modes/dynamic-training/dynamicShotEditorPipeline.ts` | 训练包编辑器数据管道：打点标注切片起始帧、介入帧与结束帧 | `pnpm check` |
| **branch26** | `z/agy/rc/branch26-shot-timeline-preview-widget` | `src/ui-core/training/DynamicTimelinePreviewWidget.tsx` | 时间轴预览组件：可视化展示 Lead-in 轨迹、介入接管时刻与预期进球航迹 | `pnpm check` |
| **branch27** | `z/agy/rc/branch27-dynamic-pack-storage-worker` | `src/game/modes/dynamic-training/DynamicPackStorageWorker.ts` | IndexedDB / 本地文件系统的 Dynamic Training Pack 异步存取与压缩解压 Worker | `pnpm check` |
| **branch28** | `z/agy/rc/branch28-dynamic-training-baseline-test` | `src/game/scenario/dynamicTrainingBaselineScenario.ts` | 编写完整的 Dynamic Training 单机自动化场景回归用例 | `node scripts/runScenarioBaseline.mjs` |

---

### Track 4: Replay 联机观战室与远程选点介入接管 (Branches 29–36)
*核心目标：实现 Host 掌控时间轴、Client 本地 Hermite 插值的极低带宽观战室，以及 4-3-2-1-0 音阶接管倒计时。*

| 分支编号 | Git 分支名 | 目标核心模块 / 文件路径 | 职责与产出物 | 独立验证命令 |
| :--- | :--- | :--- | :--- | :--- |
| **branch29** | `z/agy/rc/branch29-spectator-room-model` | `src/network/server/spectator/SpectatorRoomDomainState.ts` | 定义 Replay 观战房间数据域（HostClientSession、ActiveReplaySha256、PlayState） | `pnpm check` |
| **branch30** | `z/agy/rc/branch30-timeline-lockstep-signaling` | `src/network/server/spectator/SpectatorTimelineMessenger.ts` | Host 广播时间轴信令（`SEEK_TO`, `PLAY_RATE`, `PAUSE_AT`）与 Client 接收管线 | `pnpm check` |
| **branch31** | `z/agy/rc/branch31-intervene-role-assignment` | `src/network/server/spectator/InterveneRoleAssignmentSubController.ts` | 介入角色授权器：Host 指定自己演示接管或授权特定 Client（学员）接管 | `pnpm check` |
| **branch32** | `z/agy/rc/branch32-non-pov-car-behaviors` | `src/network/server/spectator/nonPovCarBehaviorPipeline.ts` | 实现介入后非主视角车辆行为（`disappear`、`trajectory-replay`、`no-input` 惯性） | `pnpm check` |
| **branch33** | `z/agy/rc/branch33-takeover-countdown-audio-ladder` | `src/audio/TakeoverCountdownAudioLadder.ts` | 4-3-2-1-0 倒计时分级音效管线（第4秒静音Settle，3/2秒Tick，1秒警示音，0秒GO音） | `pnpm check` |
| **branch34** | `z/agy/rc/branch34-sys-ctrl-handoff-state-machine` | `src/replay/rlrc/runtime/RLRCInterveneTakeoverSubController.ts` | 打通 `SYS_CTRL_STATE_ONLY` -> `SYS_CTRL_INTERVENE` -> `SYS_CTRL_PHYSICS_RESUME` 全链路 | `pnpm check` |
| **branch35** | `z/agy/rc/branch35-hud-intervene-overlay` | `src/ui-core/overlay/HudTakeoverCountdownOverlay.tsx` | 3D 视口内醒目的 3-2-1-GO 倒计时 HUD 动画组件与视觉焦点引导圈 | `pnpm check` |
| **branch36** | `z/agy/rc/branch36-spectator-takeover-e2e` | `src/network/server/spectator/tests/SpectatorTakeoverE2E.test.ts` | 观战室多端载入、时间轴同步与接管交接 E2E 验证用例 | `pnpm check` |

---

### Track 5: 3D 空间场馆投影战术画笔与自由视角 (Branches 37–44)
*核心目标：实现基于草皮与球场侧壁的 3D Raycasting 吸附画笔、空中航迹下垂虚线以及 Free Cam 自由视角。*

| 分支编号 | Git 分支名 | 目标核心模块 / 文件路径 | 职责与产出物 | 独立验证命令 |
| :--- | :--- | :--- | :--- | :--- |
| **branch37** | `z/agy/rc/branch37-surface-raycasting-engine` | `src/entities/telestrator/ArenaSurfaceRaycaster.ts` | 视口光线投射器：计算鼠标/触控与球场地表（Z=0）、侧壁和天花板的三维交点 | `pnpm check` |
| **branch38** | `z/agy/rc/branch38-decal-ribbon-renderer` | `src/entities/telestrator/DecalRibbonRenderer.ts` | Three.js 贴地战术标线网格渲染器（Ribbon Strip 网格，支持抗锯齿贴地偏移） | `pnpm check` |
| **branch39** | `z/agy/rc/branch39-aerial-depth-drop-stems` | `src/entities/telestrator/AerialDepthDropStems.ts` | 空中航迹纵深投影器：从三维空中点向下投射半透明垂直虚线及地面阴影圈 | `pnpm check` |
| **branch40** | `z/agy/rc/branch40-telestrator-stroke-protocol` | `src/network/protocol/telestratorStrokeWireCodec.ts` | 3D 战术笔画网络编解码器（Float32 绝对三维世界坐标序列压缩传输协议） | `pnpm check` |
| **branch41** | `z/agy/rc/branch41-telestrator-temporal-timeline` | `src/entities/telestrator/TelestratorStrokeTimeline.ts` | 笔画时空绑定器：笔画绑定回放 Tick，播放离开后 2 秒平滑淡出或锚定在关键帧 | `pnpm check` |
| **branch42** | `z/agy/rc/branch42-telestrator-ui-toolbar` | `src/ui-core/telestrator/TelestratorFloatingToolbar.tsx` | 悬浮战术工具栏（画笔、箭头、战术圈、下垂虚线开关、清空、颜色选择器） | `pnpm check` |
| **branch43** | `z/agy/rc/branch43-free-cam-hotkey-cycler` | `src/camera/CameraManager.ts` | 视角循环轮换（C 键：POV -> Ball Cam -> Director Orbit -> Free Cam 自由视角） | `pnpm check` |
| **branch44** | `z/agy/rc/branch44-coach-view-follow-mode` | `src/camera/FreeCameraController.ts` | 学员“锁定跟随导师主视角（Coach POV Follow）”与一键脱离自由观察机制 | `pnpm check` |

---

### Track 6: 原生 WebRTC 语音通话与多分支集成收敛 (Branches 45–50)
*核心目标：零依赖快速落地原生 WebRTC Opus 语音通话，建设回归测试网，为多分支 Octopus / No-FF 合并护航。*

| 分支编号 | Git 分支名 | 目标核心模块 / 文件路径 | 职责与产出物 | 独立验证命令 |
| :--- | :--- | :--- | :--- | :--- |
| **branch45** | `z/agy/rc/branch45-voice-bridge-interface` | `src/audio/voice/voiceAudioBridgeTypes.ts` | 定义 `VoiceAudioBridge` 跨平台抽象接口（输入流采集、增益控制、远程音轨挂载） | `pnpm check` |
| **branch46** | `z/agy/rc/branch46-webrtc-native-voice-p2p` | `src/audio/voice/WebRtcNativeVoiceBridge.ts` | 实现纯前端 WebRTC P2P 音频流（`RTCPeerConnection.addTrack`，Opus，内置 AEC） | `pnpm check` |
| **branch47** | `z/agy/rc/branch47-voice-spatial-panner` | `src/audio/voice/VoiceSpatialPanner.ts` | 基于 Web Audio PannerNode 的 3D 空间环绕语音（支持选开近场车距衰减） | `pnpm check` |
| **branch48** | `z/agy/rc/branch48-voice-control-ui-widget` | `src/ui-core/audio/VoiceChatControlWidget.tsx` | 麦克风静音/开麦切换、音量指示计动画条与说话者车顶指示高亮光标 | `pnpm check` |
| **branch49** | `z/agy/rc/branch49-octopus-merge-staging` | `scripts/mergeStagingTool.ts` | 多分支无冲突预检查脚本（基于 AST 结构比对，识别潜在重命名冲突与悬挂引用） | `node scripts/mergeStagingTool.ts --all` |
| **branch50** | `z/agy/rc/branch50-full-integration-watchdog` | `scripts/runFullIntegrationAudit.mjs` | 全局集成守卫脚本：串行执行 Radar、TypeScript、Baseline、Repro 与音频管线审计 | `node scripts/runFullIntegrationAudit.mjs` |

---

## 三、关键子系统技术细节与实现规范

### 1. 状态同步总线（State Synchronization）双模设计

#### (1) 连续广播模式（Continuous Broadcasting）
- **触发条件**：实时联机对战（Match Playing）、自由练车活跃状态（Freeplay Live）、或 Replay 接管后（Takeover Resumed）。
- **时序与流控**：
  - Host 物理引擎在 120Hz 权威步进。
  - 编码管线复用 `globalPhysicsSnapshotSectionCodec`，将当前 Tick 的车辆与球体三维物理快照以紧凑二进制形式打入 WebRTC Unreliable DataChannel（避免 TCP 重传阻塞）。
  - 客户端通过 `HermiteTrajectoryInterpolator` 维护 3 帧微缓冲（~25ms），消除网络抖动，呈现丝滑 120Hz 画面。

#### (2) 稀疏/暂停低带宽模式（Sparse Present / Ping-Pong Keepalive）
- **触发条件**：Host 暂停 Replay 复盘、暂停自由练车、或进入介入准备倒计时。
- **协议状态流**：
  ```
  Host (导师/主机)                               Client (学员/观战端)
     │                                                    │
     │─── [0x21 STATE_FREEZE] (Tick: 14200, StateHash) ───>│ (冻结画面，停顿物理)
     │<── [0x22 STATE_PAUSED_ACK] (AckTick: 14200) ───────│ (确认对齐)
     │                                                    │
     │   ══════════ 切换至稀疏心跳模式 (2Hz 频率) ══════════   │
     │─── [0x01 PING] (LocalTs: 1718000.12) ─────────────>│ (超轻 16 字节)
     │<── [0x02 PONG] (LocalTs, EchoTs: 1718000.12) ──────│ (精准 RTT 监控)
     │                                                    │
     │─── [0x23 SEEK_TO] (TargetTick: 15600) ────────────>│ (拖动进度条，单包触发)
     │<── [0x22 STATE_PAUSED_ACK] (AckTick: 15600) ───────│ (瞬时定位并渲染单帧)
  ```
- **核心价值**：导师暂停视频一讲就是 3 分钟，在此期间不需要传输哪怕一帧 3D 位姿数据。仅靠 2Hz 的 16 字节 Ping-Pong 空包保持连接和测量延迟，节省超过 99% 的带宽与算力。

---

### 2. 精简版 Match（多人自由练车场 / Multiplayer Freeplay）

只需在 Match 状态机中做极简改动，即可复用已验证的千行联机网络栈，实现多人同场自由飞车：
```typescript
// 伪代码方案：在 MatchPhaseLifecycleSubController 中引入 Freeplay 配置
if (d.freeplayModeActive) {
  // 1. 关闭计分与倒计时锁死
  d.disableMatchClock = true;
  // 2. 进球不中断比赛，不进入进球回放
  d.disableGoalReset = true;
  // 3. 卸载重度网络舰队诊断探针，避免额外 CPU 开销
  if (dc) {
    dc.enabled = false;
    if (pm) dc.releasePhysics(pm);
  }
  // 4. 将回放缓冲区切入 60s/90s 滚动模式，与单人模式体验完全对齐
  rb?.configureMode(REPLAY_BUFFER_MODES.COUNTUP_FIFO, 7200);
  // 5. 解除物理输入锁死，开启全场无限气
  pm?.setInputMask(false);
  pm?.sim?.setUnlimitedBoost?.(true);
}
```

---

### 3. Dynamic Training Pack 独立切片与接管生命周期

```
  [完整录像 .rcreplay.gz]
            │
            ▼ (Extractor 按照起止 Tick 裁剪)
   [Dynamic Shot Slice]
   ├── Lead-in 轨迹 (Tick 0..120): 初始车辆/球体运动姿态
   ├── Intervene Takeover 点 (Tick 120): 接管瞬间
   └── Expected Trajectory (Tick 120..360): 原始防守/射门参考轨迹
            │
            ▼ (进入 Dynamic Training Mode)
   [阶段 A: 观察与同步 (480..361 Ticks / 4秒)]
   - 物理保持状态回放或静止，视角与车辆悬挂平滑归位 (Settle)
   - 音频绝对静音，HUD 提示进入准备状态
            │
            ▼
   [阶段 B: 倒计时分级音梯 (360..0 Ticks / 3秒)]
   - Tick 360: 触发常规 Tick 音效，HUD 显式显示 "3"
   - Tick 240: 触发常规 Tick 音效，HUD 显式显示 "2"
   - Tick 120: 触发高频警示音，HUD 显式显示 "1"，两端准备发车
            │
            ▼
   [阶段 C: 接管触发 (Tick 0 / GO)]
   - 播放开球音效 match_start_go
   - 广播 SYS_CTRL_PHYSICS_RESUME
   - 学员本地实时输入解除锁死，权威物理引擎瞬间接管
            │
            ▼
   [阶段 D: 动态判定]
   - 判定器检测球体是否进入球门或目标触发区，记录耗时与球速并保存成绩
```

---

### 4. 3D 空间场馆投影战术画笔（Surface Raycasting Telestrator）

#### (1) 地表与侧壁投射数学
- 摄像机发射射线 $R(t) = \mathbf{P}_{cam} + t \mathbf{D}_{view}$。
- 与草皮地表求交：地表平面方程为 $Z = 0$，交点参数 $t = -\mathbf{P}_{cam}.z / \mathbf{D}_{view}.z$。
- 与侧壁及球门网格求交：通过 Three.js BVH 树或预定义的 6 面场馆包围盒（$|X| \le 4096$, $|Y| \le 5120$, $0 \le Z \le 2048$）求出首个碰撞点。
- 采样点直接保存为 RocketSim 绝对三维坐标（`x, y, z`），杜绝 2D 像素坐标。

#### (2) 空中航迹深度下垂虚线（Aerial Drop Stems）
- 当导师在空中绘制抛物线时，取空中笔画采样点序列 $P_i = (x_i, y_i, z_i)$。
- 针对每个关键采样点：
  - 绘制一条从 $(x_i, y_i, z_i)$ 垂直连线至 $(x_i, y_i, 0)$ 的半透明虚线（Drop Stem）。
  - 在 $(x_i, y_i, 0.5)$ 地面处渲染一个带柔和衰减阴影的投影圈（Ground Footprint Circle）。
- **感知收益**：人类视觉系统能依据地面阴影圈与垂直下垂线，立即在脑海中建立精确的空中高度感知与三维空间纵深。

---

### 5. 原生 WebRTC 语音通话接口设计 (VoiceAudioBridge)

```typescript
export interface IVoiceAudioBridge {
  readonly isMuted: boolean;
  readonly isSpeaking: boolean;
  
  initializeLocalMicrophone(): Promise<MediaStream>;
  attachToPeerConnection(peerConnection: RTCPeerConnection): void;
  detachFromPeerConnection(peerConnection: RTCPeerConnection): void;
  setMicrophoneMute(muted: boolean): void;
  setRemotePeerVolume(peerId: string, volume: number): void;
  getRemotePeerAudioEnergy(peerId: string): number;
  dispose(): void;
}
```
现阶段用 `WebRtcNativeVoiceBridge` 实现（仅需 150 行代码即可挂载）。未来若 Rust 原生底层音频管线完善并经过压力测试，只需提供一个实现相同接口的 `RustNativeVoiceSink` 即可平滑切换，完全不阻塞业务迭代。

---

## 四、多分支并行执行、代码隔离与合并准则

1. **绝对分支隔离，拒绝跨轨污染**：
   - 每个分支只能修改自己规划清单内的文件。
   - 禁止在功能分支中引入未在计划内的跨子系统引用。
2. **严守 Radar 规范**：
   - 避免产生单个文件超过 400 行（软上限）或 800 行（硬上限）。
   - 禁止循环依赖、禁止向下层逆向引用上层、禁止使用裸露的 `index.ts` 桶式重新导出。
3. **分阶段验证与 Octopus / No-FF 合并策略**：
   - **单轨独立验证**：各 Track 内部的小分支（如 Track 1 的 branch01-10）在各自完成后，先通过 `pnpm check` 验证。
   - **同轨顺序 No-FF 合并**：同一轨内的分支按依赖顺序逐步合入轨汇总分支（如 `z/agy/rc/track1-state-sync`）。
   - **跨轨最后 Octopus 合并**：当 Track 1 到 Track 6 全部具备独立绿色测试通过标志后，使用 Octopus Merge 或清晰的 No-FF PR 最终收敛回 `origin/alpha`。

---
*本计划书由自动化架构审计系统基于当前仓库 alpha 分支（Commit f5e8c9f）现场分析生成。*
