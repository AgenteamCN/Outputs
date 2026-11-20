# RocketCompanion 联机与观战架构推倒重写方案（vNext）及 4 波次并行执行计划

> **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base`
> **代码核查依据**：`src/network/protocol/GlobalPayloadCodec.ts`、`src/network/protocol/ReplicationBusProtocol.ts`、`src/network/protocol/GlobalWorkspaceProtocol.ts`、`src/network/server/EstablishedSessionManager.ts`、`src/network/sync/ReplicationStateInterpolator.ts`、`src/network/sync/spectateLookaheadPipeline.ts`
> **决策结论**：经新一轮 Live Chat 深度对齐，彻底放弃在旧观战/Lookahead 补丁上继续修补，确立**“Single Player = Loopback Multiplayer + 4 通道统一传输 + 4 波次发散-收敛-反向对齐-Octopus 合流”**的推倒重写架构与并行工程落地方案。

---

## 一、现状代码根因速览

1. **双协议割裂与模式特判泛滥**
   - 底层已有紧凑的二进制 `GlobalPayloadCodec.ts`（含 `PAYLOAD_FLAGS` 位掩码及 `RESERVED_3..7`），但上层又另起了一套 JSON 层的 `GlobalWorkspaceProtocol.ts`（承载 `replaySync`、`cameraSync`、`playbackPaused`、`authorityEpoch`）。
   - `ReplicationBusProtocol.ts` 的 `ACTIVITY_IDS` 甚至未包含 `Exhibition`（`exp-multi-bots`），导致 Exhibition 的 6 人 Roster 与 Goal Replay 完全脱节。
2. **旧观战管线强耦合 Lookahead 与单调 Tick 假设**
   - `spectateLookaheadPipeline.ts`、`spectatePhysicsLookaheadPipeline.ts` 与 `ReplicationStateInterpolator.ts` 假设 `tick` 单调递增并强行外推，导致 Replay 倒放（Rewind）反复清空缓冲并剧烈抖动，Freeplay 观战与本地物理预测打架。

---

## 二、核心架构设计（第一性原理 + 4 大关键 Trade-off 决策）

### 第一性原理
- **Single Player = 自己连自己的 Multiplayer（Loopback Authority）**：音频、HUD 1~6、NameTag、Cinematic Goal Replay 相机、粒子与 3D 渲染不再区分“单机直读”和“联机观战”，全部统一消费 Worker Authority 输出的状态流。
- **Spectating 的本质**：Client 消费 Authoritative Server 已经确定计算完、从 Frame Buffer 推出来的**即时回放流（Instant Replay Stream）**，Client 端零物理模拟、零 Lookahead 瞎猜。

### 四通道数据模型与二进制协议设计

| 通道 | 定位与传输特性 | 承载内容 | 核心护栏与工程决策 |
|---|---|---|---|
| **通道 1：高频 State 流** | Unreliable / 15·30·60Hz 必发，丢包不阻塞、下一帧覆盖 | 球与 1~6 辆车位姿/速度/Boost、BoostPad Bitmask、帧语义头（`playbackState`、`phase`、`clock`、`focusedCarIndex`、`cameraPose`）、**4B `schemaHash` (HEX)**、**4B `metadataHash`**、**`txSeq`** | **滑窗 + `EstablishedSession` 反向裁剪**：Server 维护最近 ~10 帧窗口，Client 回报已收 `txSeq` (`ack`)，Server 按 Session 裁剪已确认帧（弱网自动补发、好网仅发 1 帧增量）。**必须用单调 `txSeq` 而非 `tick`/`replayFrame` 做裁剪**，确保倒放/跳帧完全对称。 |
| **通道 2：低频 Metadata 清单** | Reliable 异步推送 / 按需补拉（不阻塞通道 1） | **`RosterManifest`**（1~6 号车 `name`、`team`、`visualId` 车身、`colorHex`、涂装、`isBot`、**`roleFlag: player \| bot \| ghost`**）+ 房间/模式上下文（含 `Exhibition`、球场预设、Replay 总帧数、Telestrator 权限） | **彻底不用死等 ACK 卡主流程**：每帧通道 1 永久携带 `metadataHash`（4 字节）。进房、**Multiplayer 中途换人**、切模式、换录像时 Hash 变化，Client 发现 `frame.metadataHash !== localHash` 立即异步拉取全量并重建车辆/HUD/名卡。 |
| **通道 3：瞬时 Signal / Incident** | Reliable / 去重 + ACK | 进球、扑救、爆破、触球音效事件、倒计时滴声、Telestrator 画笔落笔/清屏 | 沿用并扩展 `GlobalPayloadCodec` 的 `SIGNAL` Section，按 `signalId` 去重与 ACK，驱动音频与特效事件。 |
| **通道 4：Bulk / Asset 大文件流** | 低优先级独立 Reliable DataChannel（Chunk 分片 + `fileHash` + `bufferedAmount` 背压限流） | `FreeplayRewinderController` 历史切片（Intervene From Here）、Training Pack / Ghost Car 录制轨迹、后期可选 `.rlrc` Replay 预分发 | 与通道 1/2/3 物理隔离，防止大文件引发队头阻塞（Head-of-Line Blocking）。后期若针对 Replay 开启“传文件 + 仅同步 `frameIndex`”极低带宽模式，只需在 Worker 端接入本地文件读帧源，下游渲染/音频/HUD 一行代码不改。 |

### 关键架构细节护栏
1. **包头 `schemaHash`（4 字节合法 HEX，如 `0x8F3A19C4`）**：由二进制字段结构表 / 构建 Commit 自动计算，替代易错的人肉 `v1/v2` 版本号；两端协议字段只要差 1 bit，收包第一行立即报错拦截，杜绝静默错位解析。
2. **FPV Dual Controller（Coach–Student 绝对主从移交广播权）**：
   - Coach 优先级绝对大于 Student：Coach 操作中或停手 `< 300ms` 时，Host 跑物理并 Broadcast，Student 本地输入直接忽略。
   - **300ms 迟滞护栏**：Coach 停手 `>= 300ms` **且 Student 确实有非零输入**时，Student 拿过广播权实现操作者 0 延迟。
   - **抢回覆盖护栏**：Coach 一旦重新输入，无需等待 Student 确认，直接以本地最新收到的权威快照为起点递增 `authorityEpoch` 强行覆盖广播；Student 收到更高 `authorityEpoch` 瞬间停算并向 Coach 对齐（优先保 Coach 绝对丝滑）。
3. **Ghost Car（幽灵车）前瞻预留**：静态录制轨迹走通道 2/4 一次性下发按 `tick` 查表渲染；实时双车对练幽灵车只需在 `RosterManifest` 中预留 `roleFlag = ghost`，渲染层自动套用半透明材质并关闭碰撞。
4. **纯时间轴渲染缓冲（`TimelineFrameBuffer`）**：彻底删除 Lookahead；局域网 60Hz+ 支持直通，15/30Hz 降频时以**接收时间轴（arrivalMs）**做双帧线性/四元数插值，正放、慢放、暂停、倒放行为 100% 一致。

---

## 三、4 波次并行执行计划（节奏 C：先发散 -> 汇总对齐 -> 反向同步 -> Octopus 合流）

> **核心工程逻辑**：第 0 分钟将“清理旧代码”与“业务模块开发”同时铺开实现零等待；Wave 2 收口定稿二进制协议；Wave 3 让各业务分支**反向合并（Reverse Sync）**底座并在各自小分支内部消化冲突；Wave 4 一次 `git merge -s octopus` 干净收口。
> *（注：若人力紧张想省总工时，可让 `chore/strip-legacy-spectate` 先行快速完成，其余分支再从它分叉以减少一次反向合并。）*

### Wave 1：第 0 分钟全员同步发散（10 条分支并行开工，0 等待）

| 分支名称 | 职责范围 | Wave 1 独立交付物 |
|---|---|---|
| **`chore/strip-legacy-spectate`** | 专职删除 `spectateLookaheadPipeline.ts`、`spectatePhysicsLookaheadPipeline.ts`、`ReplicationStateInterpolator.ts` 单调 tick/外推耦合逻辑、`SimulationStage.ts` 与 `hudStore.ts` 中的观战临时补丁 | 干净的空壳接口与无旧补丁代码树 |
| **`feat/worker-authority-bus`** | 将 Match / Freeplay / Exhibition / Replay 步进与发包循环下沉至 Web Worker（脱离主线程 1Hz rAF 节流），实现 `txSeq` 10 帧滑窗 + `EstablishedSession` ack 裁剪 | Worker 定时器推流 + 滑窗裁剪单测 |
| **`feat/timeline-frame-buffer`** | 实现 `TimelineFrameBuffer`（60Hz 直通 + 15/30Hz 基于接收时间轴双帧插值，天然兼容正放/倒放/暂停） | 正放/倒放/慢放/暂停插值单测 |
| **`feat/mode-match-exhibition`** | 补齐 `Exhibition` (`exp-multi-bots`) 纳入统一架构：输出 6 人 `RosterManifest`，并与 `Match` 共用 `GoalReplay` 状态机与电影机位输出 | Exhibition 6 人清单 + Goal Replay 输出 + 字段清单 |
| **`feat/mode-freeplay-spectate`** | Freeplay 自由态接入 Worker 广播、Goal Replay 与 Rewind 状态输出 | Freeplay 统一帧流输出 + 字段清单 |
| **`feat/mode-replay-watch-together`** | Replay Watch Together 接入 Worker 控制游标（正/倒放、倍速、暂停、跳帧）及 1~6 车外观/焦点输出 | Replay 帧流输出 + 字段清单 |
| **`feat/mode-training-fpv-dual`** | Training Pack 接入与 FPV Dual Controller（Coach 绝对抢占、300ms 且 Student 有输入移交、`authorityEpoch` 覆盖护栏、Ghost 预留位） | 双控仲裁单测 + 字段清单 |
| **`feat/consumer-audio-arena`** | `GameAudioSubsystem` 消费 `playbackState !== 'playing'` 立即 `silenceAll()`；`Arena` 消费 `RosterManifest` 动态重建 1~6 车身（`visualId`）、配色与 NameTag（含 `ghost` 材质预留） | 音频静音与车身动态重建 + 字段清单 |
| **`feat/consumer-hud-camera`** | 顶部 1~6 HUD 纯读 `RosterManifest.team` + `cameraFocus`（彻底解耦本地 `playerCarIndex`）；Client 相机统一消费 Broadcaster 下发的 Goal Replay 电影机位 | HUD 1~6 与电影机位消费 + 字段清单 |
| **`feat/bulk-asset-channel`** | 实现第 4 通道（Chunk 分片 + `fileHash` 校验 + 背压限流），接入 `FreeplayRewinderController` 切片/快照传输，预留 Ghost 轨迹与 Replay 文件传输 | 第 4 通道分片传输单测 |

*注：Wave 1 期间，8 条业务分支各自用临时 TS Interface 跑通本地逻辑，并提交一份“本模块需要往数据包里塞哪些字段”的清单。*

---

### Wave 2：协议汇总闸口 & 干净底座合流（收敛 -> `integration/vnext-core`）

1. **专人开立 `feat/protocol-schema-alignment`**：
   - 收集 Wave 1 全部业务分支提交的字段诉求；
   - 统一设计并定稿最终二进制协议：包头 `4B schemaHash (HEX)` + `txSeq` + `4B metadataHash` + Bitmask Section Flags + `WorldFrame` / `RosterManifest` / `Signal` 编解码器（`DataView` 零分配直读）。
2. **合流形成干净底座**：
   - 将 `chore/strip-legacy-spectate`（已删旧补丁）与 `feat/protocol-schema-alignment`（最终二进制 Codec）合并为 **`integration/vnext-core`**。

---

### Wave 3：反向对齐（Reverse Sync —— 在各自小分支内部消化冲突）

- Wave 1 的所有业务/消费/通道分支，统一执行 **`git merge integration/vnext-core`**（反向对齐底座）；
- 各分支在自己内部完成：
  1. 删除 Wave 1 临时 TS Payload 字段，接入最终二进制 `WorldFrame` / `RosterManifest` / `Signal` Codec；
  2. 清理本模块内残留的旧调用点；
  3. 跑绿本分支单元测试。

---

### Wave 4：Octopus 总合流 & 自动化门禁验收

- Wave 3 全部绿灯对齐后，执行 **`git merge -s octopus ...`** 将全部业务分支一次性干脆合入集成分支；
- 运行全量门禁回归：
  - `pnpm run check`
  - `pnpm run test:multiplayer-suite`（设 `RC_SUITE_WATCHDOG_MS=280000`）
  - `pnpm run repro:dev-broadcast-v3`（更新为新协议断言，稳定 0 复现）

---

## 四、下一轮 Real Device Test 专项验收清单

1. **Replay Watch Together（6 人局录像）**
   - [ ] **车身与配色**：1~6 号车所有车身型号（Fennec/Merc/Octane 等）与颜色在 Host 和 Client 两端 100% 一致，无 1/4 号重复。
   - [ ] **顶部 1~6 HUD 聚焦与阵营**：Host 依次切换 1~6 视角时，Client 顶部 HUD 焦点同步高亮对应车号，4/5/6 始终留在对应阵营栏，不跳边。
   - [ ] **暂停音频**：Host 按暂停后，Client 所有引擎/Boost/环境音效在 1 帧内完全静音；恢复播放后正常恢复。
   - [ ] **正放 & 倒放（Rewind）**：Host 连续倒放（1x / 2x）时，Client 球与车平滑倒退，零抖动；停下后画面静止无微颤。
2. **Freeplay & Host 切后台压力测试**
   - [ ] Host 在浏览器切 Tab 或最小化窗口 10 秒，Client 观战球/车依然保持设定广播率平滑运动，无 1Hz 掉帧。
   - [ ] Freeplay 进球触发 Goal Replay，Client 同步进入电影机位且车辆正常运动。
3. **Exhibition（6 人 Bot 表演赛）**
   - [ ] 进房后 Client 场内 6 个 NameTag 与顶部/记分板名字、阵营颜色与 Host 完全一致（不再显示默认 4 名卡）。
   - [ ] 任意一方进球后，Client 与 Host 同步进入 Cinematic Goal Replay，球与 6 辆车正常回放动作。
4. **FPV Dual Controller（Coach–Student 联调）**
   - [ ] Coach 操作时，Student 按键被忽略；Coach 停手 `>= 300ms` 且 Student 有输入时 Student 接管广播（0 延迟操作）；Coach 一旦重新输入立即强占回控制权，无卡顿。
5. **Multiplayer 中途换人 & 降频带宽 & Bulk/Telestrator**
   - [ ] 房间中途换人/Bot 替补时，`metadataHash` 触发自动重拉，Client 车身/颜色/名卡瞬间同步更新。
   - [ ] 手动切至 `30Hz` / `15Hz` 时，`TimelineFrameBuffer` 时间轴插值画面流畅；滑窗裁剪在模拟丢包下无卡顿。
   - [ ] `FreeplayRewinder` 第 4 通道切片传输与 Telestrator 画画标注在各模式下同步正常。
