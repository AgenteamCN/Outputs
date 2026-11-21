# Wave 1 Branch `W1-03` (`feat/mode-match-exhibition`) 交付报告

- **分支**：`feat/mode-match-exhibition`
- **基线提交**：`7c80b445f2953076e5356320c6e91bc04da4bba8` (`origin/z/agy/rc/multiplayer-recheck-base`)
- **交付提交**：`b288c51808673617c24c8749a9af79ff7a38af55` (`verified: true`, `state: VALID`)

---

## 一、核心改动与交付物

1. **Exhibition (`exp-multi-bots`) 纳入统一 Activity 架构**
   - `src/network/protocol/ReplicationBusProtocol.ts`：新增 `ACTIVITY_IDS.EXHIBITION = 'exhibition'`、默认数据包标志位 `ACTIVITY_DEFAULT_PACKET_FLAGS[ACTIVITY_IDS.EXHIBITION]`（含 `STATE_SNAPSHOT | USER_INPUT | CLOCK_AND_SCORE | FOLLOW_HOST_CAMERA`），以及 `normalizeActivityId()`（将 `'exp-multi-bots'` 自动归一化为 `'exhibition'`）。
   - `src/network/server/activities/ActivityKernel.ts`：新增并注册 `ExhibitionActivityKernel`，在 `createDefaultActivityKernelRegistry()` 中注册 `ACTIVITY_IDS.EXHIBITION`。
   - `src/network/server/RoomActivityOrchestrator.ts` & `src/network/sync/SyncAuthorityRouter.ts`：`switchActivity()` 自动归一化 `'exp-multi-bots'`，并在 `SyncAuthorityRouter` 中将 `EXHIBITION` 的球权威归属设为 `HOST`。

2. **Wave 1 临时 TS 契约与 6 人 `RosterManifest` 构建器**
   - `src/game/modes/shared/matchExhibitionVNextContract.ts`：定义 `RosterManifest`、`RosterManifestCarEntry`（含 `roleFlag: 'player' | 'bot' | 'ghost'`、`visualId`、`colorHex`、`paintFinish`、`boostVisual`、`boostAudio`、`anthemId` 等）、`GoalReplayStateOutput`、`CameraPoseOutput` 及 `MatchExhibitionFrameHeader`。
   - 实现 FNV-1a 32-bit `computeRosterManifestHash()` 与 `formatMetadataHashHex()`，以及 `buildExhibitionRosterManifest()` / `buildMatchRosterManifest()`。
   - `src/game/modes/experimental-multi-bots/ExperimentalMultiBotsRosterController.ts`：新增 `latestRosterManifest` 与 `getRosterManifest()`，并在 `publishSpawnedCarsTelemetry()` 中同步生成 6 人 `RosterManifest`，补齐 `projectRosterCarEntries()` 的 `isBot` 与 `isLocalPlayer` 字段。

3. **`Match` 与 `Exhibition` 共用 `GoalReplay` 状态机与电影机位输出**
   - `src/game/modes/shared/GoalReplayStateMachine.ts`：实现统一状态机 `GoalReplayStateMachine`（`idle` -> `goal_scored` -> `goal_replay` -> `idle`）与 `extractGoalReplayCameraPose()`（自动将 `DirectorCameraRuntimeState` 的 Unreal 坐标 `(eyeX, eyeY, eyeZ)` 转换为 Three.js `(eyeX, eyeZ, eyeY)`，并兼容 POV 模式从 `cameraManager.camera` 提取机位）。
   - `src/game/modes/experimental-multi-bots/ExperimentalMultiBotsReplayController.ts` & `src/game/modes/ExperimentalMultiBotsMode.ts`：接入 `GoalReplayStateMachine`，并暴露 `getRosterManifest()`、`getGoalReplayState()`、`getFrameHeader()`。
   - `src/game/modes/BotMatchMode.ts`：接入同一 `GoalReplayStateMachine`，在 `replay` 阶段同步推进并暴露 `getGoalReplayState()`。

---

## 二、Wave 2 协议汇总字段清单（`W1-03` 诉求）

供 `feat/protocol-schema-alignment` 汇总进最终二进制协议：

### 通道 1（高频 State 流帧语义头）
- `activityId`: `enum` (`match` | `exhibition` | ...)
- `metadataHash`: `uint32` (4B，用于触发客户端按需重拉 `RosterManifest`)
- `phase`: `enum` (`kickoff` | `playing` | `goal_scored` | `goal_replay` | `podium` | `ended`)
- `playbackState`: `enum` (`playing` | `paused` | `rewinding` | `fast_forward`)
- `clockSecondsRemaining`: `float32` / `uint16`
- `isOvertime`: `bitflag` (`1 bit`)
- `blueScore`: `uint8`
- `orangeScore`: `uint8`
- `focusedCarIndex`: `uint8` (`0..5`)
- `goalReplay`:
  - `active`: `bitflag` (`1 bit`)
  - `cameraMode`: `enum` (`director` | `pov`)
  - `povCarIndex`: `int8` (`-1` 表示无 / Director)
  - `scoringTeam`: `uint8` (`0 | 1`)
  - `scorerCarIndex`: `uint8`
  - `progress`: `float32` (`0..1`)
  - `frameIndex`: `uint32`
  - `totalFrames`: `uint32`
- `cameraPose`（电影机位 / POV 机位统一输出，Three.js 世界坐标）：
  - `valid`: `bitflag` (`1 bit`)
  - `eye`: `float32[3]`
  - `target`: `float32[3]`
  - `up`: `float32[3]`
  - `fovDeg`: `float32`

### 通道 2（低频 `RosterManifest` 清单）
- `metadataHash`: `uint32`
- `activityId`: `string / enum`
- `modeId`: `string` (`match` | `exp-multi-bots`)
- `arenaId`: `string`
- `matchDurationSec`: `uint16`
- `telestratorPermission`: `enum` (`host_only` | `everyone`)
- `cars[1..6]`:
  - `carIndex`: `uint8`
  - `keyNumber`: `uint8`
  - `name`: `string`
  - `team`: `uint8` (`0 | 1`)
  - `teamSlotIndex`: `uint8`
  - `roleFlag`: `enum` (`player` | `bot` | `ghost`)
  - `isBot`: `boolean`
  - `botType`: `string`
  - `visualId`: `string`
  - `hitboxId`: `string`
  - `colorSlot`: `uint8`
  - `colorHex`: `string`
  - `paintFinish`: `string`
  - `boostVisual`: `string`
  - `boostAudio`: `string`
  - `anthemId`: `string | null`

---

## 三、验证结果

- `env -u NODE_OPTIONS node --experimental-strip-types scripts/tests/testModeMatchExhibitionW1_03.ts`：PASS（覆盖 `EXHIBITION` Activity 注册与路由、`exp-multi-bots` 归一化、6 人 `RosterManifest` 与确定性 `metadataHash`、`GoalReplayStateMachine` 与 `cameraPose` 坐标转换）
- `pnpm run check`（`radar` + `tsc --noEmit`）：GREEN（1430/1430 TS，0 错误，0 循环依赖，0 越界导入）
