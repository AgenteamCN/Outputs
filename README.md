# RocketCompanion vNext — Wave 1 `W1-04` (`feat/mode-freeplay-spectate`) 交付报告与字段清单

- **分支**: `feat/mode-freeplay-spectate`
- **基线**: `origin/z/agy/rc/multiplayer-recheck-base` (`7c80b445f2953076e5356320c6e91bc04da4bba8`)
- **Verified Bot Commit**: `0585d24019b29dd3c72f621d9acc14af165e1396` (`signature.isValid: true`, `state: VALID`)
- **验证状态**:
  - `node --experimental-strip-types scripts/testFreeplaySpectateStream.ts`: **5/5 测试套件全绿**
  - `pnpm run check` (`radar` + `tsc --noEmit`): **STATUS: GREEN**（1433/1433 TS 文件，0 循环依赖，0 体积硬上限违规）
  - `npx prettier --check`: **通过**

---

## 一、W1-04 实现摘要

1. **Freeplay 统一帧流控制器（Worker-compatible, zero-DOM）**
   - `src/network/server/activities/freeplaySpectateTypes.ts`：定义 Wave 1 临时 TS 接口（Ch1 `FreeplayStateFramePayload`、Ch2 `FreeplayRosterManifest`、Ch3 `FreeplaySignalEvent`、Ch4 `FreeplayRewinderSliceManifest`）。
   - `src/network/server/activities/freeplaySnapshotReader.ts`：纯快照提取（球体、1~8 车位姿/四元数/速度/Boost/状态标志、BoostPad Bitmask）与 FNV-1a `metadataHash` / `fileHash` 计算。
   - `src/network/server/activities/freeplayGoalReplayStreamPipeline.ts`：纯 Goal Replay（含普通进球与手动 `KeyU` 过期球）帧提取与 Director/POV `cameraPose` 生成。
   - `src/network/server/activities/FreeplayRewinderStreamSubController.ts`：Rewinder 子状态机（`idle` / `recording` / `stopped` / `rewinding` / `forwarding` / `paused-scrub` / `preroll-countdown`）及 Channel 4 历史切片清单生成。
   - `src/network/server/activities/FreeplaySpectateStreamController.ts`：统一编排 Freeplay 自由态（`live`）、Goal Replay（`goal_replay`，支持慢放倍率与跳过）及 Rewind（`rewind`，**倒放期间 `sourceTick` / `replayCursorFrame` 递减而 `txSeq` 严格单调递增**）。

2. **Worker Kernel 与 Rewinder 权限打通**
   - `src/network/server/activities/ActivityKernel.ts`：`FreeplayActivityKernel` 挂载 `FreeplaySpectateStreamController`，处理 `freeplay_reset_kickoff`、`freeplay_expire_ball`、`freeplay_goal_scored`、`freeplay_skip_goal_replay`、`freeplay_set_unlimited_boost`、`freeplay_set_replay_camera_mode`、`freeplay_rewinder_command` 控制消息，并输出统一 `getStateSummary`。
   - `src/replay/FreeplayRewinderController.ts`：放开 Hosting / Loopback Authority 在线 Freeplay 下使用 Rewinder 的限制，仅拦截非 Host 的远端观战客户端。

3. **自动化测试**
   - `scripts/testFreeplaySpectateStream.ts`：覆盖 Live 帧流与 `metadataHash` 变更、Goal Replay 慢放/机位/跳过、手动 Ball Expire、Rewind 倒放严格单调 `txSeq` + 暂停静音语义 + 3-2-1 Preroll 干预 + Ch4 切片清单、以及 `FreeplayActivityKernel` 控制消息集成。

---

## 二、Wave 2 协议汇总字段需求清单（W1-04 Freeplay）

供 Wave 2 `feat/protocol-schema-alignment` 统一二进制 Codec 定稿使用：

### Channel 1：高频 State 流 (`FreeplayStateFramePayload`)
| 字段名 | 类型 / 建议二进制编码 | 说明 |
|---|---|---|
| `txSeq` | `uint32` | 严格单调递增传输序号（**倒放/跳帧时绝不回退**，用于 `EstablishedSession` 滑窗裁剪） |
| `schemaHash` | `uint32` (4B HEX) | 协议结构校验哈希（Wave 1 临时占位 `0x57313034`） |
| `metadataHash` | `uint32` (4B) | 当前 `RosterManifest` + Freeplay 上下文 FNV-1a 哈希 |
| `sourceTick` | `int32` | 物理 / 回放 / Rewinder 源 Tick（倒放时递减） |
| `subMode` | `uint8` enum | `0=live`, `1=goal_replay`, `2=rewind` |
| `playbackState` | `uint8` enum | `0=playing`, `1=paused`, `2=rewinding`（非 `playing` 时客户端触发音频 `silenceAll()`） |
| `playbackRate` | `float32` | 播放倍率（`1.0` 正常、`0.25/0.5` Goal Replay 慢放、`-1.0/-2.0` 倒放、`0` 暂停） |
| `phase` | `uint8` enum | `0=playing`, `1=goal`, `2=replay`, `3=countdown`, `4=lobby` |
| `clockSecondsRemaining` | `float32` | `preroll-countdown` 剩余秒数或 `goal_replay` 剩余秒数 |
| `unlimitedBoost` | `bitflag` (1 bit) | Freeplay 无限 Boost 开关 |
| `boostPadMask` | `uint32` / `uint64` | 34 个 BoostPad 激活状态位掩码 |
| `focusedCarIndex` | `uint8` | 当前聚焦车号（0~7） |
| `cameraMode` | `uint8` enum | `0=player`, `1=director`, `2=pov` |
| `cameraPose` | `optional (7 x float32)` | `goal_replay` / `rewind` 时下发的权威相机位姿（`posX/Y/Z`, `lookAtX/Y/Z`, `fov`） |
| `replayCursorFrame` | `int32` | 回放/Rewinder 当前帧游标（`live` 时为 `-1`） |
| `replayTotalFrames` | `uint32` | 回放/Rewinder 总帧数（`live` 时为 `0`） |
| `rewinderState` | `uint8` enum | `idle / recording / stopped / rewinding / forwarding / paused-scrub / preroll-countdown` |
| `ball` | `struct` | `pos(3f)`, `vel(3f)`, `angVel(3f)`, `isExpired(1bit)` |
| `cars[]` | `1..6 x struct` | `carIndex(u8)`, `team(1bit)`, `pos(3f)`, `vel(3f)`, `quat(4f)`, `boost(f32/u8)`, `isSupersonic/isBoosting/onGround/isDemolished(bitflags)` |
| `goalReplayMeta` | `optional struct` | `replayType('goal'|'expire')`, `scoringTeam(u8)`, `scorerCarIndex(u8)`, `scorerName(str)`, `ballSpeedKph(f32)` |

### Channel 2：低频 Metadata 清单 (`FreeplayRosterManifest`)
- `metadataHash (uint32)`、`activityId ('freeplay')`、`arenaPreset ('standard' | 'throwback')`、`unlimitedBoost (bool)`、`goalReplayEnabled (bool)`、`telestratorPermission ('host_only' | 'everyone')`
- `cars[]`: `carIndex`, `name`, `team (0|1)`, `visualId`, `colorHex`, `paintFinish`, `decalId`, `isBot`, `roleFlag ('player' | 'bot' | 'ghost')`

### Channel 3：瞬时 Signal / Incident (`FreeplaySignalEvent`)
- `signalId (uint32)`、`txSeq (uint32)`、`sourceTick (int32)`、`kind`:
  `car_ball_hit` | `goal_scored` | `ball_expired` | `goal_replay_start` | `goal_replay_end` | `kickoff_reset` | `flip_reset_gained` | `rewinder_state_change` | `countdown_tick`
- 可选负载：`carIndex`, `team`, `ballSpeedKph`, `rewinderState`, `countdownSeconds`

### Channel 4：Bulk / Asset 切片清单 (`FreeplayRewinderSliceManifest`)
- `fileHash (string/uint32)`、`totalFrames (uint32)`、`startTick (int32)`、`endTick (int32)`、`cursorFrame (uint32)`、`byteLength (uint32)`、`chunkCount (uint32)`
