# RocketCompanion 联机与观战架构重写方案及并行执行计划

> **基线分支**：`z/agy/rc/multiplayer-recheck-base` (`2fc5fc0`)
> **决策结论**：经 Live Chat 多轮对齐，停止在旧观战/插值补丁上继续叠加修复（“在屎上雕花”），转为**保留现有核心资产与测试套件、彻底剥离旧观战补丁、按“Single Player = 自己连自己的 Multiplayer（Loopback Authority）”第一性原理重写同步链路**。

---

## 一、Live Chat 核心共识

1. **Single Player = Localhost Multiplayer（统一数据链路）**
   - 单人模式本质是“连接本地 Loopback Authority、且不允许外部 Peer 加入的 Multiplayer”。
   - 音频（`GameAudioSubsystem`）、顶部 HUD、NameTag、Cinematic Goal Replay 相机、粒子特效与 3D 渲染，**不再区分“本地单机直读路径”和“联机观战路径”**，全部统一消费来自 Authority 的同一套状态流。
2. **先做通无复杂前瞻的干净状态流，再按需叠加纯时间轴插值**
   - 局域网 / 60Hz+ 直连下，不依赖 Lookahead 外推即可流畅渲染。
   - 面向外网上下行带宽受限场景（15Hz / 30Hz / 60Hz 自适应降频），在干净状态流之上叠加**基于接收时间轴的纯双帧插值**（不假设 `tick` 单调递增，正放、慢放、暂停、倒放行为完全一致）。
3. **FPV Dual Controller（Coach–Student 双控模型）**
   - Coach 与 Student 类似于双遥控器接入同一个 Game Core：Coach 优先级 > Student。
   - Coach 有输入时立即抢占并屏蔽 Student；Coach 空闲超过 200–300ms 时允许 Student 接管输入（或按需移交广播权威，操作者延迟最低），切换瞬间附带完整物理快照防瞬移。
4. **广播下沉至 Web Worker**
   - 所有模式的权威步进与广播发包均由 Web Worker 驱动，彻底免疫浏览器切后台 Tab 时 `requestAnimationFrame` 被节流到 1Hz 的问题，从而**彻底不需要 Client 端 Lookahead 瞎猜外推**。

---

## 二、Real Device Test 6 大问题与代码级根因复盘

| # | Real Device Test 实测现象 | 代码级根因 | 重写后的根除方式 |
|---|---|---|---|
| **1** | **Replay Watch Together：Host 暂停后 Client 音频不停** | `GameAudioSubsystem.triggerGameAudioEvents` 仅检查 `isSignalAudioMutedInGoalReplay(r)`，未检查 Replay 暂停态 / `playbackState === 'paused'` | `WorldFrame` 携带语义字段 `playbackState: 'playing' \| 'paused' \| 'scrubbing'`，音频层收到非 `playing` 立即 `silenceAll()` |
| **2** | **Replay Watch Together：车身全错、1/4 号颜色冲突** | `ReplayViewerRosterSubController` 中 `arena.ensureCar(e.carIndex, e.team, e.visualId)` 与 `setCarLoadout` 仅在 Host 本地执行；`WorkspaceReplaySyncPayload.roster` 未下发完整车身模型与配色并重建 Client 车辆 | 统一 `RosterManifest`（含 `carIndex, team, name, visualId, colorHex, paintFinish, boostVisual`），附带短 Hash/版本号随每帧校验 |
| **3** | **Replay Watch Together：正放正常、倒放 Client 剧烈抖动** | `ReplicationStateInterpolator` 写死了 `tick < last.tick - 12` 触发 `this.reset()` 清空缓冲、`tick > lastPushedTick` 门控 `stalled` 检测，且 `spectateLookaheadPipeline` 按正向速度外推 | 彻底删除 `spectateLookaheadPipeline` 与 tick 单调假设；插值缓冲按**到达时间轴（arrival/renderTime）**工作，倒放与正放完全对称 |
| **4** | **Replay Watch Together：顶部 1~6 HUD 恒焦 4 号，切 4/5/6 会先移到左侧再回右侧** | `hudStore` / HUD 槽位映射与 `playerCarIndex` 局部覆盖耦合，且车号与阵营左右栏位绑定逻辑混乱 | HUD 只读 `RosterManifest` 的 `team` 归属与 `WorldFrame.cameraFocus.focusedCarIndex`，彻底解耦本地 `playerCarIndex` |
| **5** | **Freeplay Spectate 剧烈抖动** | Freeplay / Match / Replay 强行共用 `ReplicationStateInterpolator` + `SimulationStage` 模式特判，Client 本地预测与 Host 广播流打架 | Client（非权威端）关闭本地物理模拟与 Lookahead，纯渲染 Worker 广播帧流 |
| **6** | **Exhibition（6 人）：全员名卡用默认 4 名卡、Goal Replay 进不去/不动** | `Match` 有 `MatchRosterSubController` 广播 `roster_update`，但 `exp-multi-bots` 只写了本地 `useExperimentalMultiBotsStore`；Goal Replay 机位与 `isGoalReplay` 状态未走统一广播 | Exhibition 与 Match 共用同一套 `RosterManifest` 广播与 `GoalReplay` 状态机（Broadcaster 统一输出 `phase: 'goal-replay'` + 相机位姿） |

---

## 三、资产盘点：保留 vs 删除 vs 重写

### 1. 保留资产（Reuse）
- **各模式本地玩法核心**：Match、Freeplay、Exhibition (`experimental-multi-bots`)、ReplayViewer、Training Pack 本身。
- **Telestrator 画画功能**：独立模块，设计干净，实测稳定，直接保留权限与画笔同步通道。
- **底层 P2P 传输与信令**：`MultiplayerSessionManager` / WebRTC DataChannel 建连与房间管理。
- **回归测试与 E2E 门禁**：`pnpm run check`、`pnpm run test:multiplayer-suite`、`pnpm run repro:dev-broadcast-v3`（更新为新协议断言）。

### 2. 彻底删除/剥离的包袱（Remove）
- `src/network/sync/spectateLookaheadPipeline.ts`
- `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
- `src/network/sync/ReplicationStateInterpolator.ts` 中所有与“tick 单调递增假设 / 物理外推 / 多模式特判”强耦合的逻辑
- `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` 与 `SimulationStage.ts` 中分散的观战/Replay 临时补丁分支
- `ReplayViewerRosterSubController.ts` / `replayOnlineSyncHelper.ts` / `hudStore.ts` 中互打补丁的局部状态同步

### 3. 全新重写模块（Rewrite）
- **Worker 权威广播总线**（`HostWorker` / `ReplayWorker` 统一发包，不挂主线程 rAF）
- **双层统一同步协议**：`RosterManifest`（低频+Hash校验+ACK） + `WorldFrame`（高频状态+语义+相机）
- **纯时间轴渲染平滑器**：`TimelineFrameBuffer`（支持 60Hz 直通与 15/30Hz 降频双帧插值，兼容正/倒放）
- **FPV Dual Controller 仲裁器**：Coach 优先抢占 + 200–300ms 空闲移交 + 状态快照对齐

---

## 四、目标架构设计（Clean-Slate Architecture）

```text
+-------------------------------------------------------------------+
|  Input Sources (FPV Dual Controller)                              |
|  [Coach / Host Input] --(priority > Student)--> [Authority Arbiter]|
|  [Student Input] -----(allowed if Coach idle >= 250ms)-----+      |
+------------------------------------------------------------|------+
                                                             v
+-------------------------------------------------------------------+
|  Web Worker Authority (Match / Freeplay / Exhibition / Replay)    |
|  - 免疫浏览器后台 1Hz rAF 节流                                      |
|  - 统一输出两类包：                                                  |
|    1) RosterManifest (manifestHash, cars[]: id/team/name/body/color)|
|    2) WorldFrame (manifestHash, authorityEpoch, playbackState,    |
|       phase, tick, clock, ball, cars[], cameraFocus/cinematicPose)|
+-------------------------------------------------------------------+
          | (Single Player: Loopback)        | (Multiplayer: 15/30/60Hz)
          v                                  v
+-------------------------------------------------------------------+
|  Unified Client Renderer (Pure Presentation, Zero Physics Guess)  |
|  - Hash 校验器：frame.manifestHash !== localHash -> 拉取 Manifest   |
|  - TimelineFrameBuffer：60Hz 直通 / 15-30Hz 按接收时间轴双帧插值     |
|  - 下游统一消费者：                                                  |
|    * 3D Arena & Cars (按 Manifest 动态重建车身/颜色/Boost)          |
|    * GameAudioSubsystem (playbackState !== 'playing' -> 静音)      |
|    * HUD 1~6 & NameTags (纯读 Manifest.team + cameraFocus)         |
|    * Camera (Cinematic Goal Replay 由 Broadcaster 推位姿，可选自由视角)|
+-------------------------------------------------------------------+
```

### 关键细节防坑说明（5 个核心护栏）
1. **`manifestHash` 每帧随路校验**：不仅靠进房 ACK，每帧 `WorldFrame` 附带短 `manifestHash`（如 32-bit FNV-1a）；任何重连、丢包或模式切换导致 Hash 不一致时，Client 自动请求全量 `RosterManifest`。
2. **`authorityEpoch` 防双写竞争**：Coach 与 Student 切换输入/广播权时递增 `authorityEpoch`；Coach 拥有无条件抢占权，Student 仅在 `coachIdleMs >= 250` 且收到最新物理快照后才可推进模拟。
3. **Goal Replay 权威统一**：Broadcaster 在 Match / Freeplay / Exhibition 中统一执行 Goal Replay 状态机与 Cinematic Camera 计算，通过 `WorldFrame` 下发 `phase: 'goal-replay'` 与 `cameraPose`，Client 照单渲染即可。
4. **音频与特效语义化**：`GameAudioSubsystem` 与粒子系统不再根据“两帧坐标差 / 本地 dt”盲算状态，而是直接读取 `WorldFrame.playbackState`、`WorldFrame.phase` 与车辆 `boost/supersonic` 标志位。
5. **15/30/60Hz 降频兼容**：`TimelineFrameBuffer` 以 `arrivalMs`（接收时间戳）为横轴做线性/四元数球面插值，不依赖 `tick` 递增，因此 `playbackSpeed` 为负（倒放）、为 0（暂停）、为 0.5（慢放）时行为完全一致。

---

## 五、并行分支任务拆解（Parallel Execution Plan）

为最快完成推倒重写，建议基于当前基线切出 **4 条并行工作流（Workstream）**，最后合流联调：

| 并行分支 / 责任人 | 核心任务范围 | 关键产出与验收标准 | 依赖关系 |
|---|---|---|---|
| **Branch A：`feat/protocol-manifest-worldframe`**<br>（协议与 Worker 广播底座） | 1. 定义极简 `RosterManifest`（含 `manifestHash` + ACK）与 `WorldFrame` 协议<br>2. 将 Match / Freeplay / Exhibition / Replay 的广播发送全部下沉到 Web Worker 定时器 | - 协议层单测全绿<br>- Host 切后台 Tab 时广播频率稳定在设定值（15/30/60Hz），无 1Hz 掉帧 | 独立可并行，作为 B/C/D 的契约底座（Day 1 先定接口） |
| **Branch B：`feat/clean-timeline-renderer`**<br>（清理 Lookahead + 纯时间轴渲染） | 1. 删除 `spectateLookaheadPipeline`、`spectatePhysicsLookaheadPipeline` 及旧 `ReplicationStateInterpolator` 耦合代码<br>2. 实现 `TimelineFrameBuffer`（支持直通与接收时间轴双帧插值，支持正/倒放/暂停） | - 正放、倒放、慢放、暂停单测全绿<br>- Freeplay / Replay Watch Together 倒放零抖动 | 依赖 Branch A 接口定义 |
| **Branch C：`feat/unified-consumers-hud-audio`**<br>（音频 / HUD / 车身 / Exhibition 统一消费） | 1. `GameAudioSubsystem` 接入 `playbackState === 'paused'` 立即静音<br>2. `Arena` 消费 `RosterManifest` 重建全员车身（`visualId`）与颜色<br>3. 顶部 1~6 HUD 与 NameTag 纯读 `RosterManifest` + `cameraFocus`<br>4. 打通 Exhibition (`exp-multi-bots`) 的 Manifest 广播与 Goal Replay 状态机 | - Replay 1~6 车身/颜色/HUD 两侧完全一致<br>- Exhibition 6 人名卡与 Goal Replay 100% 同步<br>- Host 暂停即静音 | 依赖 Branch A 接口定义 |
| **Branch D：`feat/fpv-dual-controller-and-e2e`**<br>（FPV 双控 + 测试套件适配） | 1. 实现 Coach–Student `authorityEpoch` 抢占与 250ms 空闲移交（带快照）<br>2. 对齐 `test:multiplayer-suite` 与 `repro:dev-broadcast-v3` 验收门禁（解除 Bug 2/3 测试耦合） | - `test:multiplayer-suite` 全绿<br>- `repro:dev-broadcast-v3` 稳定 0 复现<br>- Training Pack 双控抢占单测通过 | 与 A/B/C 并行开发，合流后统一跑门禁 |

---

## 六、下一轮 Real Device Test 专项验收清单

重写合流后，在真实设备（Desktop / Tauri 双端局域网 + 模拟 30Hz 降频）按以下清单逐项打勾验收：

1. **Replay Watch Together（6 人局录像）**
   - [ ] **车身与配色**：1~6 号车所有车身型号（Fennec/Merc/Octane 等）与颜色在 Host 和 Client 两端 100% 一致，无 1/4 号重复。
   - [ ] **顶部 1~6 HUD 聚焦与阵营**：Host 依次切换 1/2/3/4/5/6 视角时，Client 顶部 HUD 焦点同步高亮对应车号，且 4/5/6 始终留在右侧蓝/橙对应阵营栏，不会跳边。
   - [ ] **暂停音频**：Host 按暂停后，Client 所有引擎/Boost/环境音效在 1 帧内完全静音；恢复播放后正常恢复。
   - [ ] **正放 & 倒放（Rewind）**：Host 连续倒放（1x / 2x）时，Client 球与车平滑倒退，无剧烈抖动；停下后画面静止无微颤。
2. **Freeplay & Host 切后台压力测试**
   - [ ] Host 在浏览器切到其他 Tab 或最小化窗口 10 秒，Client 观战球/车依然保持 60Hz（或设定广播率）平滑运动，无 1 秒一卡。
   - [ ] Freeplay 进球触发 Goal Replay，Client 同步进入电影机位且车辆正常运动。
3. **Exhibition（6 人 Bot 表演赛）**
   - [ ] 进房后 Client 场内 6 个 NameTag 与顶部/记分板名字、阵营颜色与 Host 完全一致（不再显示默认 4 名卡）。
   - [ ] 任意一方进球后，Client 与 Host 同步进入 Cinematic Goal Replay，球与 6 辆车正常回放动作。
4. **FPV Dual Controller（Coach–Student 联调）**
   - [ ] Coach 操作时，Student 按键被屏蔽；Coach 停手 >250ms 后 Student 可接管车辆操作，Coach 一旦重新输入立即无感抢回控制权，无瞬移。
5. **降频带宽与 Telestrator 回归**
   - [ ] 将广播率手动切至 `30Hz` / `15Hz` 时，开启时间轴插值后观战画面依然流畅。
   - [ ] Telestrator 画画标注与清屏在各模式下同步正常。
