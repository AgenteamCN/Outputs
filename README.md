# W1-03 (`feat/mode-match-exhibition`) 交付报告与协议字段清单

- **分支**：`feat/mode-match-exhibition`
- **基线提交**：`7c80b445f2953076e5356320c6e91bc04da4bba8` (`origin/z/agy/rc/multiplayer-recheck-base`)
- **已推送 Verified Bot Commit**：`c5f8a4da0c48b2501ab0205150e95518dc571813` (`verified: true`)

---

## 一、核心交付物摘要

1. **Exhibition (`exp-multi-bots`) 纳入统一协议与活动内核**
   - `src/network/protocol/ReplicationBusProtocol.ts`：在 `ACTIVITY_IDS` 中补齐 `EXHIBITION: 'exp-multi-bots'`，并在 `ACTIVITY_DEFAULT_PACKET_FLAGS` 中配置 `STATE_SNAPSHOT | USER_INPUT | CLOCK_AND_SCORE | FOLLOW_HOST_CAMERA`。
   - `src/network/server/activities/ActivityKernel.ts`：新增 `ExhibitionActivityKernel` 并注册到 `createDefaultActivityKernelRegistry()`。
   - `src/network/sync/SyncAuthorityRouter.ts`：`applyActivitySwitch` 纳入 `ACTIVITY_IDS.EXHIBITION` 的权威路由。

2. **Exhibition 与 Match 共用 6 人 `RosterManifest`（通道 2）**
   - 新增 `src/network/protocol/MatchExhibitionRosterManifest.ts`：
     - 定义 `MatchExhibitionRosterManifest` 与 `RosterManifestEntry`（含 `roleFlag: 'player' | 'bot' | 'ghost'` 幽灵车预留位、`visualId`、`hitboxId`、`colorSlot`、`colorHex`、`paintFinish`、`boostVisual`、`boostAudio`、`anthemId`）。
     - 实现确定性 32-bit FNV-1a `computeRosterMetadataHash`（4 字节 `metadataHash`），任何 1~6 号车变更或模式上下文变化均会触发 Hash 更新。
     - 提供 `buildExhibitionRosterManifest` 与 `buildMatchRosterManifest`。
   - 接入 `ExperimentalMultiBotsRosterController.ts`（`getRosterManifest()`）与 `MatchRosterSubController.ts`（`getRosterManifest()`）。

3. **Match 与 Exhibition 共用 `GoalReplay` 状态机与电影机位输出**
   - 新增 `src/replay/UnifiedGoalReplayStateMachine.ts`：
     - 统一 `GoalReplayPhase`（`'idle' | 'active' | 'finished'`）、`playbackState`（`'playing' | 'paused'`）、触球时间轴分析、戏剧性慢放（`computeDramaticSlowMoFactor`）与导演/POV 双模式相机输出 `GoalReplayCameraPoseOutput`（`mode`, `eye`, `target`, `up`, `fovDeg`, `focusedCarIndex`）。
   - 接入 `ExperimentalMultiBotsReplayController.ts` 与 `MatchGoalReplaySubController.ts`，统一通过 `getGoalReplayFrameOutput()` 输出标准化的 `GoalReplayStreamFrameOutput`。

4. **自动化单测与架构门禁验证**
   - 新增 `scripts/testMatchExhibitionUnified.ts`：`env -u NODE_OPTIONS node --experimental-strip-types scripts/testMatchExhibitionUnified.ts` -> `[testMatchExhibitionUnified] PASS`。
   - `pnpm run check`（`radar` + `tsc --noEmit`）：1430/1430 TS 文件通过，0 循环依赖，0 越层引用，全绿通过。

---

## 二、Wave 2 协议汇总字段清单（W1-03 诉求）

### 通道 1（高频 State 流）
| 字段名 | 类型 / 字节建议 | 说明 |
|---|---|---|
| `metadataHash` | `uint32` (4B) | 由 `MatchExhibitionRosterManifest` 计算，用于 Client 检测中途换人 / Bot 替补 / 切模式并触发通道 2 补拉 |
| `phase` | `uint8` / enum | 比赛阶段：`lobby \| prepare \| kickoff \| playing \| goal_scored \| replay \| paused \| ended \| podium` |
| `playbackState` | `uint8` / enum | `playing \| paused`（Goal Replay 暂停时供音频子系统 1 帧静音） |
| `clockSecondsRemaining` | `float32` / `int16` | 比赛倒计时秒数（含加时标记 `isOvertime`） |
| `blueScore` / `orangeScore` | `uint8` + `uint8` | 蓝队 / 橙队比分 |
| `focusedCarIndex` | `int8` | 当前相机聚焦车号（0~5） |
| `cameraMode` | `uint8` (1bit) | `0 = director`（电影机位）/ `1 = pov`（第一/跟随视角） |
| `cameraPose.eye` | `float32[3]` (12B) | Goal Replay 电影机位 / POV 相机位置（Unreal Units） |
| `cameraPose.target` | `float32[3]` (12B) | Goal Replay 相机注视目标点 |
| `cameraPose.fovDeg` | `float32` (4B) | 相机视场角（默认 Director FOV 或 POV 110°） |
| `goalReplayProgress` | `uint16` / `float32` | Goal Replay 当前进度 `[0, 1]` 与 `slowMoFactor` |

### 通道 2（低频 Metadata 清单：`RosterManifest`）
| 字段名 | 类型 | 说明 |
|---|---|---|
| `manifestVersion` | `uint32` | 单调递增清单版本号 |
| `metadataHash` | `uint32` | 清单内容 32-bit FNV-1a 校验哈希 |
| `activityId` | `string` / enum | `'match' \| 'exp-multi-bots'` |
| `arenaPreset` | `string` | 球场预设名 |
| `matchDurationSeconds` | `uint16` | 比赛时长配置 |
| `telestratorAllowed` | `boolean` | 是否允许画笔标注 |
| `entries[0..5]` | `RosterManifestEntry[]` | 1~6 号车：`carIndex`, `keyNumber`, `team(0\|1)`, `teamSlotIndex`, `name`, `roleFlag('player'\|'bot'\|'ghost')`, `isBot`, `botType`, `visualId`, `hitboxId`, `colorSlot`, `colorHex`, `paintFinish`, `boostVisual`, `boostAudio`, `anthemId` |

### 通道 3（瞬时 Signal / Incident）
| 信号 | 参数 | 说明 |
|---|---|---|
| `GOAL_REPLAY_WILL_START` | `scoringTeam`, `scorerCarIndex` | 进球回放即将开始 |
| `GOAL_REPLAY_START` | `replayTicks`, `packedMeta` | 进球回放开始（携带得分队、进球者、球速） |
| `GOAL_REPLAY_END` | `0`, `0` | 进球回放结束，切回 Kickoff 或 Podium |
| `SCORE_UPDATED` / `ROUND_STARTED` | `blueScore`, `orangeScore` | 比分与回合同步 |
