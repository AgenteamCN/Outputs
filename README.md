# W1-05 (`feat/mode-replay-watch-together`) 交付报告 & 字段清单

- **分支**：`feat/mode-replay-watch-together`
- **基线**：`origin/z/agy/rc/multiplayer-recheck-base` (`7c80b445f2953076e5356320c6e91bc04da4bba8`)
- **Verified Bot Commit**：`83a591ab2fd1f3a93daf5f8411bff94e03bf6627`（GitHub GraphQL `createCommitOnBranch`，`signature.isValid = true`, `state = VALID`）

---

## 一、W1-05 核心交付物

1. **Wave 1 临时 TS 协议契约**：`src/replay/watch-together/ReplayWatchTogetherTypes.ts`
   - 定义 `ReplayWatchPlaybackState`（`idle | paused | playing | rewinding | forwarding`）、`ReplayWatchPlaybackCommand`、`ReplayWatchRosterManifest`、`ReplayWatchFrameStreamPacket`、`ReplayWatchCursorStatus`。
2. **1~6 车 RosterManifest 与确定性 4B `metadataHash`**：`src/replay/watch-together/ReplayWatchRosterManifest.ts`
   - `buildReplayWatchRosterManifest()`：归一化 1~6 号车的 `carIndex`、`slotKey`、`team`（0/1 固定阵营，杜绝 4/5/6 切焦点跳边）、`roleFlag`（`player | bot | ghost`）、`visualId`、`hitboxId`、`colorSlot`、`colorHex`、`paintFinish`、`boostVisual`、`boostAudio`、`decalId`。
   - `computeReplayMetadataHash()`：基于 FNV-1a 计算 32-bit `metadataHash`（及 `0xXXXXXXXX` 格式化），任何车身、配色、阵营或录像变更即触发新 Hash。
3. **Worker 游标与帧流引擎**：`src/replay/watch-together/ReplayWatchCursorEngine.ts`
   - 纯 Worker 无 DOM/无 rAF 驱动，支持 `load_replay`、`play`、`pause`、`rewind`、`fast_forward`、`seek`、`set_speed`（0.25x / 0.5x / 1.0x / 2.0x 等）、`set_focus`（1~6 视角与自由相机位姿）、`unload`。
   - 无论正放、倒放（Rewind）、暂停或跳帧（Seek），`txSeq` **严格单调递增**，满足 `EstablishedSession` 滑窗反向裁剪与 `TimelineFrameBuffer` 时间轴插值对称性。
   - 自动提取球体状态、1~6 辆车状态、34 个 BoostPad Bitmask（Lo 32b + Hi 2b）、计分板/时钟/加时状态与按 `signalId` 去重的 Channel 3 信号事件（仅正放/Seek 触发，倒放/暂停不误播）。
4. **Worker ActivityKernel 接入**：`src/network/server/activities/ActivityKernel.ts`
   - `ReplayActivityKernel` 挂载 `ReplayWatchCursorEngine`，处理 `replay_watch_command` / `replay_watch_step` 控制消息，并在 `getStateSummary()` 输出 `replayWatch`、`rosterManifest`、`lastFramePacket`。
5. **单元测试与集成测试**：`scripts/testReplayWatchTogether.ts`

---

## 二、验证结果

- `node --experimental-strip-types scripts/testReplayWatchTogether.ts`：**PASS**（覆盖 6 车外观与 `metadataHash` 校验、正放/倍速/倒放/暂停/跳帧、`txSeq` 单调性、4/5/6 焦点阵营锁定、34 BoostPad Bitmask、`RoomActivityOrchestrator` + `ReplayActivityKernel` 集成）。
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1431/1431 TS，0 循环依赖，0 越层引用，0 错误）。
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**28/28 PASS**。

---

## 三、Replay Watch Together 数据包字段清单（供 Wave 2 `feat/protocol-schema-alignment` 汇总）

### 通道 1：高频 State 流（Unreliable，15/30/60Hz）
| 字段名 | 类型/位宽建议 | 说明 |
|---|---|---|
| `schemaHash` | `uint32` (4B HEX) | 协议结构校验指纹 |
| `txSeq` | `uint32` (4B) | 严格单调递增发包序列号（正/倒放/暂停/跳帧均递增，用于滑窗 ACK 裁剪） |
| `metadataHash` | `uint32` (4B) | 当前 `RosterManifest` 哈希；Client 不一致时异步重拉通道 2 |
| `replayFrame` | `uint32` (4B) | 当前整数帧游标 (`0 .. totalFrames - 1`) |
| `fractionalFrame` | `float32` (4B) | 精确浮点游标（用于慢放/倍速子帧对齐） |
| `playbackState` | `uint8` (枚举) | `0=idle, 1=paused, 2=playing, 3=rewinding, 4=forwarding`（`!== playing` 时 Client 音频立即 `silenceAll()`） |
| `playbackSpeed` | `float32` / `uint16` Q8.8 | 倍速（如 `0.25, 0.5, 1.0, 2.0`） |
| `playbackDirection` | `int8` | `-1=倒放, 0=暂停, 1=正放` |
| `focusedCarIndex` | `int8` | `0..5`（`-1` 为自由视角），驱动 Client 顶部 1~6 HUD 高亮与视角切换 |
| `cameraMode` | `uint8` | `0=car, 1=free, 2=ball` |
| `cameraPose` | `6 * float32` (可选 bit) | `eyeThreeJs[3] + targetThreeJs[3]`（FreeCam / Director Cam 时携带） |
| `matchPhase` | `uint8` (枚举) | `playing / kickoff / goal_replay / overtime / ended` |
| `blueScore` / `orangeScore` | `uint8` + `uint8` | 蓝/橙比分 |
| `remainingSeconds` / `overtimeSeconds` | `uint16` + `bool` | 倒计时秒数与加时标记 |
| `ball` | `pos(3f)+vel(3f)+angVel(3f)+rot(quat/mat)` | 球位姿与速度 |
| `cars[0..5]` | `carIndex(u8)+pos(3f)+vel(3f)+angVel(3f)+rot+boost(u8)+flags(u8)` | 1~6 车位姿、Boost 与 `onGround/supersonic/demoed/isBoosting/isFlipping` 位掩码 |
| `boostPadBitmaskLo/Hi` | `uint32 + uint8` (34 bits) | 34 个 BoostPad 激活状态位掩码 |

### 通道 2：低频 Metadata 清单（Reliable 异步推送 / 按 `metadataHash` 按需补拉）
| 字段名 | 类型 | 说明 |
|---|---|---|
| `metadataHash` | `uint32` (4B) | 整个 `RosterManifest` 的 FNV-1a 校验哈希 |
| `replayId` | `string` | 录像文件名 / ID |
| `totalFrames` | `uint32` | 录像总帧数 |
| `tickRateHz` | `uint16` | 录像采样率（默认 `120`） |
| `arenaPreset` | `uint8` | 球场预设（Standard / Throwback 等） |
| `telestratorPermission` | `uint8` | `host_only / everyone` |
| `cars[0..5]` | Array (1~6) | 每辆车：`carIndex (0..5)`, `slotKey (1..6)`, `name`, `team (0\|1)`, `roleFlag (player\|bot\|ghost)`, `isBot`, `visualId`, `hitboxId`, `colorSlot`, `colorHex`, `paintFinish`, `boostVisual`, `boostAudio`, `decalId` |

### 通道 3：瞬时 Signal / Incident（Reliable，按 `signalId` 去重 + ACK）
| 字段名 | 类型 | 说明 |
|---|---|---|
| `signalId` | `uint32` | 唯一信号 ID（去重防重放） |
| `signalType` | `uint16` | 进球、扑救、爆破、触球音效、倒计时滴声、Telestrator 事件 |
| `frameIndex` | `uint32` | 触发帧号 |
| `param1` / `param2` | `int32` / `int32` | 附加参数（如触球车号、得分阵营） |

### 通道 4：Bulk / Asset 大文件流（低优先级独立通道，后期可选）
- 可选 `.rcreplay` 文件预分发（`replayId`, `fileHash`, `totalBytes`, `chunkIndex`），开启后通道 1 仅需同步 `replayFrame` + `playbackState` + `focusedCarIndex` + `cameraPose`。
