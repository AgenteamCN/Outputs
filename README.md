# Rocket Companion (RC) 下一代架构演进与并行开发方案审查报告

**基准分支与 Commit**：`origin/alpha` (`f5e8c9f`)  
**审查工作分支**：`z/agy/rc/arch-review-alpha`  
**工程健康度基线**：`pnpm check`（Radar Gen-3 + `tsc --noEmit`）**GREEN**（1350/1350 TS 文件，0 JS 引擎文件，0 循环依赖，0 向上引用）

---

## 一、 核心结论先行（TL;DR）

1. **50 个分支过多、过碎，建议收敛为 6 条 Track 共 22 个高内聚分支。**
   原计划为了凑满 50 个分支，将大量“本应内聚在 1 个模块”的逻辑强行按单文件切分，且**未充分盘点 `origin/alpha` 已有代码**——当前 Alpha 分支实际上已经完成了原方案中约 **35%～40%** 的底层能力（如 `MatchSystemControlSubController`、`RLRCInterveneTakeoverSubController`、`StaticTrainingPackMode`、`FreeCameraController`、`globalPhysicsSnapshotSectionCodec` 等）。盲目按 50 分支从零开干，不仅会**重复造轮子**，还会触发 Radar 2.0 的“伪拆分 / Pass-Through Proxy / Host Penetration”结构告警，并大幅推高合并期接口对齐成本。
2. **推翻“Match 与 State Sync 拆成两套平行体系”的做法，改为【统一房间复制总线 + 可热插拔模式内核】。**
   针对“整体捏成一个 vs 拆成 Match 和 State”的架构纠结，以及“开发者开房后动态切换 Match / Replay / Sandbox 并 Broadcast 给所有 Client”的核心诉求，最优解是**纵向分层、正交解耦**：
   - **底层“捏成一个”**：沉淀统一的 `RoomReplicationBus`（只负责 WebRTC 双模状态分发、可靠信令、权威归属与输入回传，**完全不感知**上层是比赛、练车、看录像还是开发者沙盒）。
   - **上层“按模式拆分且可热切换”**：将 `Match`、`Freeplay`、`ReplayWatch`、`DynamicTraining`、`DevSandbox` 抽象为可热插拔的 **Room Activity Kernel**，由 Host 通过单条广播信令在同一个房间连接内动态切换，无需断开重连。
3. **代码具体实现交由各子团队裁决，主干只锁定接口契约（Port / Wire Protocol）与 Radar 规范。**

---

## 二、 深度架构研判：解决“Match vs State”纠结与动态 Broadcast 模式

### 1. 当前 Alpha 分支的真实痛点
通过深度扫描 `src/network/server/match/` 与 `src/game/modes/`，当前最大的问题不是缺少功能，而是**职责倒挂**：
- `MatchDomainState.ts` 和 `MatchPhaseLifecycleSubController.ts` 既要管比赛计分/加时/Podium，又通过 `lobbyMode`、`isTrainingMode`、`trainingDisableGoalReset` 等布尔开关硬塞了练车场逻辑，还挂载了 `MatchSystemControlSubController.ts`（负责 `SYS_CTRL_STATE_ONLY` 停帧录像同步）。
- 结果就是：**想做一个“不计分的自由练车”或“开发者沙盒广播”，必须拖着整套 Match 状态机一起跑。**

### 2. 推荐的新抽象：Unified Room Replication Bus + Hot-Swappable Activity Kernel

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                Host Room Orchestrator (房主动态调度器)                     │
│  广播信令: ROOM_ACTIVITY_SWITCH (activityId, seed/payload, authorityMask) │
└───────┬──────────────────┬──────────────────┬──────────────────┬────────┘
        ▼                  ▼                  ▼                  ▼
  [Freeplay Kernel]  [Match Kernel]    [Replay & Intervene] [Dev Sandbox /
  (无计分/无限气/     (倒计时/比分/      (Host控时间轴/       Future CSR
   FIFO滚动池)        GoalReplay/加时)   4-3-2-1-0接管)       Broadcast]
        │                  │                  │                  │
        └──────────────────┴──────────────────┴──────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│             Unified Room Replication Bus (唯一底层同步总线)                │
│  1. 双模状态广播: Continuous (60/120Hz 快照) <-> Sparse (冻结帧 + 2Hz Ping) │
│  2. 实体权威路由: Host权威 / Peer本地预测 / Replay只读 / 接管交接           │
│  3. 多路复用信道: Unreliable Snapshot Channel + Reliable Control Channel  │
│  4. 零历史包袱: 不含任何计分、倒计时、进球重置等业务语义                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**这一抽象如何完美满足您的长远设想：**
- **动态变换、不断线切换**：开发者（Host）建房后，所有 Client 接入 `RoomReplicationBus`。Host 先拉起 `Replay` 模式带大家复盘；复盘完毕发一个 `ROOM_ACTIVITY_SWITCH('match')`，全员无缝切入 2v2 对战；打完一局发一个 `ROOM_ACTIVITY_SWITCH('freeplay')` 或 `ROOM_ACTIVITY_SWITCH('sandbox')`，全员瞬间回到自由练车或开发者自定义的测试沙盒。
- **为未来 Client-Side Rendering / 全局状态广播铺路**：只要把“当前房间正在运行的 Activity ID + 序列化场景/UI 状态摘要”也视为 `RoomReplicationBus` 的一个可靠信令载荷，未来甚至可以将菜单状态、视角焦点、实验室参数一并广播给所有 Client，上层业务与底层同步彻底零耦合。

---

## 三、 原 50 分支计划书深度审计（基于 Alpha 源码盘点）

### 1. 已有资产 vs 真正缺口对照表

| 原计划 Track | Alpha (`f5e8c9f`) 已存在的相关实现 | 原计划中的重复/错位项 | 真正的增量缺口 |
| :--- | :--- | :--- | :--- |
| **Track 1: State Sync & Protocol** (`branch01-10`) | • `globalPhysicsSnapshotSectionCodec.ts`（稀疏 448-float 编码 + activeCarMask）<br>• `GlobalPayloadCodec.ts` & `sessionSignalWireCodec.ts`<br>• `DataChannelTransport.ts`（已含基础 Ping/Pong）<br>• `PredictionReconciler.ts`<br>• `RLRCHermiteSlerpResampler.ts`（Hermite+Slerp 算法已有） | • `branch03/04/08` 大量与现有 Codec 和 Resampler 重叠，另起炉灶会产生两套编码器<br>• `branch10` 指定的 `*.test.ts` 与 `node scripts/radar.ts` **无法运行**（仓库无 vitest/jest，且 TS 脚本需 `--experimental-strip-types` 或 `pnpm check`） | 1. **Sparse/Freeze 降频态**：在 `SYS_CTRL_STATE_ONLY` / 暂停态下真正切断 120Hz 物理广播，切至 `STATE_PAUSED_ACK` + 2Hz 16B Ping-Pong。<br>2. **双 DataChannel 显式分流**：区分 Unreliable 快照信道与 Reliable 信令信道。<br>3. **独立基准验证**：应接入 `src/game/scenario/` 而非虚构 `.test.ts`。 |
| **Track 2: Multiplayer Freeplay** (`branch11-18`) | • `MatchDomainState.ts` 已有 `lobbyMode`、`trainingDisableGoalReset`、`trainingUnlimitedBoost`<br>• `MatchPhaseLifecycleSubController.ts` 已有 `enterLobbyMode()` 与 `startTrainingMode()`（已配 `COUNTUP_FIFO`）<br>• `LobbyGhostCarCoordinator.ts` 已有碰撞模式切换<br>• `autonomousTrainingAndMatrixScenario.ts` 已测通部分训练流 | • `branch11-15` 是在已经臃肿的 `Match` 子控制器上继续打补丁，与“彻底解耦”的目标背道而驰<br>• `branch18` 的 `runMultiplayerRoomRepro.mjs --mode=freeplay` 目前脚本仅识别 `2p \| 2p2ai \| 3p3ai \| both \| all` | 1. **剥离而非打补丁**：将 Freeplay 抽成独立 `ActivityKernel`，不依赖 `Match` 计分/时钟。<br>2. **多人一键重置球 & 碰撞热切换**的房间级广播。<br>3. 为 `runMultiplayerRoomRepro.mjs` 补齐 `freeplay` 与模式热切换压测分支。 |
| **Track 3: Dynamic Training Pack** (`branch19-28`) | • `src/game/modes/static-training/`（含 `StaticTrainingPackMode.ts`、`staticTrainingVariationMath.ts`、`StaticTrainingShotEditor.ts`）<br>• `src/replay/ReplayBufferSliceExtractor.ts`（已支持切片提取） | • `branch22`（变体微扰）和 `branch20`（切片提取）已有高度可复用的前序基建，无需重头写数学库 | 1. `.rctrain` 动态切片资产 Schema 与 IndexedDB Worker。<br>2. **DynamicTrainingMode**（Lead-in 切片回放 -> 介入点 -> 接管 -> 多维成功判定）。<br>3. 时间轴预览与标注 UI（`src/ui-core/training/`）。 |
| **Track 4: Replay Spec & Takeover** (`branch29-36`) | • `MatchSystemControlSubController.ts`（已实现 `SYS_CTRL_STATE_ONLY 0xA0` / `INTERVENE_WILL_RESUME 0xA1` / `PHYSICS_RESUME 0xA2`）<br>• `RLRCInterveneTakeoverSubController.ts`（已有 480 帧 pre-roll 与 4 种 non-POV 行为）<br>• `SignalAudioEngine.ts`（已有 `training_countdown` / `match_start_go`）<br>• `useInGameHudChoreography.ts`（已有 `useTakeoverPrerollChoreography`） | • `branch32/33/34/35` 把**已存在的核心状态机和 HUD/音频管线**当成全新开发任务列出<br>• `branch36` 同样写了不存在的 `*.test.ts` | 1. **多人观战室房间模型**（当前系统控制偏单机/主从原型，缺多人观战席位、Host 授权指定学员接管 `InterveneRoleAssignment`）。<br>2. **时间轴 Lockstep 信令**（Host Seek/Speed/Pause 广播与 Client 对齐）。 |
| **Track 5: 3D Telestrator & Free Cam** (`branch37-44`) | • `CameraManager.ts` + `FreeCameraController.ts`（Free Cam 已完整实现，支持 WASD/I/M/Shift/鼠标拖拽/FOV） | • `branch43/44` 需基于现有 `CameraManager` 扩展，不可另起控制器 | 1. **3D Telestrator（完全空白，最适合独立并行）**：`src/entities/telestrator/`（地表/侧壁 Raycasting、Ribbon 网格、空中下垂虚线与阴影圈、时空淡出）+ `src/ui-core/telestrator/` + 笔画压缩协议。<br>2. **Coach POV Follow** 与 C 键视角循环扩展。 |
| **Track 6: WebRTC Voice & Integration** (`branch45-50`) | • `WebRTCConnection.ts`（已管理 `RTCPeerConnection` 生命周期）<br>• `scripts/radar.ts`（已有 Gen-3 架构与耦合审计） | • `branch49`（`mergeStagingTool.ts`）在 20 余个清晰垂直分支下属于过度工程，且可能与现有 `radar.ts` 重复 | 1. **WebRTC Voice（完全空白，最适合独立并行）**：`IVoiceAudioBridge` + `WebRtcNativeVoiceBridge`（挂载到现有 `WebRTCConnection` 的 `RTCPeerConnection.addTrack`）+ 可选 `VoiceSpatialPanner` + 静音/音量条 UI。<br>2. **全链路集成守卫脚本**（串行跑 `pnpm check`、`test:baseline`、`repro`）。 |

### 2. 工程规范与路径/命名问题清单
1. **测试路径与命令错误**：仓库当前没有任何 `vitest`/`jest` 依赖，所有 `*.test.ts` 均无运行器；自动化验证必须落入 `src/game/scenario/`（由 `pnpm test:baseline` 驱动）或 `scripts/runMultiplayerRoomRepro.mjs`（Headless Chrome）。
2. **Radar 2.0 红线约束**：
   - 严禁出现 `index.ts` 桶式重导出（0 Value Re-Exports）。
   - 严禁在 `src/` 或 `scripts/` 中出现任何中文注释、中文日志或 Emoji（`checkLanguageAndEmoji` 会直接报错拦截）。
   - 严禁使用 `soccer`/`soccar` 命名（`AGENTS.md` 硬性要求）。
   - 单文件严格控制在 400 行 / 12KB 以下（超 800 行 / 18KB 直接阻塞），且不能为了凑行数做无意义的 `Part1`/`Helper1` 水平拆分。
3. **跨层引用检查**：
   - `src/entities/telestrator/` 只能依赖 `three` 与通用数学/物理常量，**绝不能反向 import `src/game/` 或 `src/network/`**。
   - `src/audio/voice/` 只通过窄接口（Port）接入 `WebRTCConnection` 的 `RTCPeerConnection`，不直接耦合 `MultiplayerSessionManager`。

---

## 四、 重构后的高效率并行开发蓝图（6 Track / 22 分支）

以下方案**不预设具体代码实现**，仅明确各分支的**职责边界、依赖关系与独立验收标准**，具体实现由各团队自行裁决。

### 阶段一：可 100% 零阻塞并行的独立子系统（优先启动）

#### Track A：3D 空间战术画笔与自由视角扩展（4 分支）
- `z/agy/rc/branch01-telestrator-3d-core`：`src/entities/telestrator/` — 场馆表面 Raycasting（`Z=0` 草皮 + 6 面侧壁/天花板）、贴地 Ribbon 渲染、空中航迹下垂虚线（Drop Stems）与地面阴影圈。
- `z/agy/rc/branch02-telestrator-timeline-and-wire`：`src/entities/telestrator/` + `src/network/protocol/` — 笔画时空绑定（按 Replay Tick 锚定/淡出）与 Float32 世界坐标序列网络编解码。
- `z/agy/rc/branch03-telestrator-ui-toolbar`：`src/ui-core/telestrator/` — 悬浮工具栏（画笔/箭头/战术圈/下垂虚线开关/颜色/清空）。
- `z/agy/rc/branch04-camera-coach-follow`：`src/camera/` — 扩展 `CameraManager` 与 `FreeCameraController`，增加 C 键循环与“锁定跟随导师主视角（Coach POV Follow）”。

#### Track B：纯原生 WebRTC 语音通话（3 分支）
- `z/agy/rc/branch05-voice-bridge-and-webrtc`：`src/audio/voice/` — 定义 `IVoiceAudioBridge` 抽象防腐层，并实现基于 `RTCPeerConnection.addTrack`（Opus + 内置 AEC/NS）的 `WebRtcNativeVoiceBridge`。
- `z/agy/rc/branch06-voice-spatial-panner`：`src/audio/voice/` — 基于 Web Audio `PannerNode` 的 3D 空间语音衰减（可开关）。
- `z/agy/rc/branch07-voice-chat-ui`：`src/ui-core/audio/` — 麦克风开/关、音量电平条、说话者车顶指示状态。

#### Track C：Dynamic Training Pack 单机引擎（4 分支）
- `z/agy/rc/branch08-dynamic-pack-schema-and-extractor`：`src/game/modes/dynamic-training/` + `src/replay/` — `.rctrain` 资产 Schema，复用并增强 `ReplayBufferSliceExtractor` 提取动态切片与介入点元数据。
- `z/agy/rc/branch09-dynamic-pack-variation-and-judge`：`src/game/modes/dynamic-training/` — 复用 `staticTrainingVariationMath` 思路实现动态切片初速度/角度/自旋微扰，以及多维度成功判定器（进球/扑救/限时/球速）。
- `z/agy/rc/branch10-dynamic-training-mode-runner`：`src/game/modes/dynamic-training/` — 实现 `DynamicTrainingMode`（Lead-in 播放 -> 介入倒计时 -> 物理接管 -> 判定结算 -> 成绩持久化）。
- `z/agy/rc/branch11-dynamic-training-editor-ui`：`src/ui-core/training/` — 训练包切片打点与时间轴预览 UI + IndexedDB 异步存取 Worker。

---

### 阶段二：核心网络与房间模式重构（建议先锁协议，再并行）

#### Track D：统一房间复制总线与双模同步（4 分支）
- `z/agy/rc/branch12-replication-bus-protocol`：`src/network/protocol/` + `src/network/sync/` — 定义 `RoomReplicationBus` 双模协议（Continuous 120Hz vs Sparse/Frozen + `STATE_PAUSED_ACK` + 2Hz 16B Ping-Pong）与 `ROOM_ACTIVITY_SWITCH` 热切换信令。
- `z/agy/rc/branch13-webrtc-dual-channel-mux`：`src/network/transport/webrtc/` — 在 `DataChannelTransport` 中落地双 DataChannel 分流（Unreliable 快照信道 vs Reliable 控制/信令信道）与自适应码率降频。
- `z/agy/rc/branch14-sync-authority-and-interpolator`：`src/network/sync/` — 实体级物理权威路由（Host / Peer / Replay 只读）与客户端稳态插值衔接（复用 `RLRCHermiteSlerpResampler` 核心数学）。
- `z/agy/rc/branch15-room-activity-orchestrator`：`src/network/server/` — 将 `AuthoritativeServerHost` / `HostWorkerMessageRouter` 从硬编码 `Match` 中解放，支持在不断开房间的情况下动态挂载/切换上层 `ActivityKernel`。

#### Track E：上层模式内核拆解（Freeplay / Match / Sandbox）（3 分支）
- `z/agy/rc/branch16-freeplay-activity-kernel`：`src/network/server/activities/` — 彻底从 `MatchPhaseLifecycleSubController` 剥离出的多人自由练车内核（无计分、无进球冻结、无限气、ROLLING FIFO、一键重置球、Ghost/Solid 碰撞热切换）。
- `z/agy/rc/branch17-match-and-sandbox-kernels`：`src/network/server/activities/` — 保留纯粹的 `Match` 对战内核，并新增 `DevSandboxKernel`（支持开发者房间广播自定义状态/实验场模式切换）。
- `z/agy/rc/branch18-room-switch-repro-suite`：`scripts/runMultiplayerRoomRepro.mjs` — 增加 `--mode=freeplay` 与动态模式热切换（Freeplay -> Match -> Replay -> Freeplay）的 Headless Chrome 多端压测。

#### Track F：Replay 多人观战室与远程接管（4 分支）
- `z/agy/rc/branch19-spectator-room-and-timeline`：`src/network/server/spectator/` — 多人观战室域模型与 Host 时间轴 Lockstep 信令（Seek / PlayRate / Pause 触发总线进入 Sparse 2Hz Ping-Pong 模式）。
- `z/agy/rc/branch20-intervene-role-and-handoff`：`src/network/server/spectator/` — 接管角色授权（Host 自演或指定特定 Client 学员接管），桥接现有 `MatchSystemControlSubController` 与 `RLRCInterveneTakeoverSubController`。
- `z/agy/rc/branch21-takeover-countdown-ux`：`src/audio/` + `src/ui-core/overlay/` — 复用并扩展现有 `SignalAudioEngine` 与 `useInGameHudChoreography`，对齐 4-3-2-1-0 音梯与多人观战提示 HUD。
- `z/agy/rc/branch22-full-integration-audit`：`src/game/scenario/` + `scripts/` — 补齐动态训练、观战接管场景测试，并沉淀一键串行审计入口（`pnpm check` + `test:baseline` + `repro`）。

---

## 五、 合并策略与各团队交付红线

1. **合入节奏**：
   - **Track A / B / C** 完全独立，开发完成并通过 `pnpm check` 后即可随时合入主干，无需等待网络层。
   - **Track D** 锁定 `RoomReplicationBus` 与 `ROOM_ACTIVITY_SWITCH` 协议后，**Track E / F** 立即基于该契约并行开发，最后通过 `--no-ff` 按 Track 顺序收敛。
2. **交付前强制自检**：
   - 任何分支提交前必须通过 `pnpm run format:check && pnpm run check`（0 Radar Error，0 `tsc` Error）。
   - 禁止在仓库中引入中文注释/字符串、Emoji、`index.ts` 桶文件及向后兼容垫片。
   - 不保留死代码，该删的历史耦合文件直接删除（依赖 Git 历史回溯）。
