# Wave 1 (`W1-08`: `feat/consumer-hud-camera`) 交付报告与字段清单

- **分支**: `feat/consumer-hud-camera`
- **基线**: `origin/z/agy/rc/multiplayer-recheck-base` (`7c80b445f2953076e5356320c6e91bc04da4bba8`)
- **Verified Bot Commit**: `ca1c8012abf6b2385c1d28e6704009b0b83100e6` (`signature.isValid = true`, `state = VALID`)
- **Commit URL**: https://github.com/Antigravi-ty/RocketCompanion/commit/ca1c8012abf6b2385c1d28e6704009b0b83100e6

---

## 一、核心工程交付物

1. **临时 TS 消费契约 (`src/game/pipeline/stages/vnextHudCameraTypes.ts`)**
   - 定义 `VNextRosterManifest`、`VNextRosterEntry`（含 `roleFlag: 'player' | 'bot' | 'ghost'`、`team: 0 | 1`）与 `VNextWorldFrameCameraAndHudHeader`（含 `cameraFocus`、`cameraPose`、`carTelemetry` 等），供 Wave 2 `feat/protocol-schema-alignment` 对齐二进制协议。

2. **顶部 1~6 HUD 纯消费者 (`src/game/pipeline/stages/vnextHudRosterConsumer.ts`)**
   - `buildHudFrameFromRosterManifest` 与 `pushVNextHudFrameToChannel` 零分配复用 `HudTransientFramePayload`：
     - **彻底解耦 `playerCarIndex`**：阵营归属纯读 `RosterManifest.entries[i].team`（`0` 蓝左 1~3，`1` 橙右 4~6），仅当 `!isSpectatorSession` 且存在本地人类玩家 (`roleFlag === 'player' && isLocalPlayer`) 时才按本地玩家阵营翻转左右，观战 / Replay Watch Together / 纯 Bot 表演赛下 4/5/6 永不跳边。
     - **焦点纯读 `cameraFocus.focusedCarIndex`**：`isFocusedPov` 与 `focusedSlotKey` 仅由 `cameraFocus.focusedCarIndex` 驱动；Goal Replay Director 模式下自动清空车号高亮 (`focusedSlotKey = null`)。
     - **Ghost 预留位隔离**：`roleFlag === 'ghost'` 自动从 1~6 竞技 HUD 槽位中排除。
   - 同步修复 `src/game/pipeline/stages/hudTransientFramePipeline.ts` 中 `updateSlotInPlace` 的 `entry.isSelf || ...` 双重高亮缺陷，以及 `src/game/pipeline/stages/TelemetryStage.ts` 在观战/同步回放模式下优先读取 `camera.focusedCarIndex` 并支持 vNext Fast Path。

3. **Client Goal Replay 电影机位消费者 (`src/game/subsystems/vnextGoalReplayCameraConsumer.ts` & `GameCameraSubsystem.ts`)**
   - 新增 `VNextGoalReplayCameraConsumer` 并接入 `GameCameraSubsystem.applyBroadcasterCameraFrame(frame)`：
     - 当 `cameraFocus.isGoalReplayActive && cameraFocus.cameraMode === 'director'` 且 `cameraPose` 有效时，自动获取 `'vnext-broadcaster'` Director Override、同步 `fovDeg` 并直接应用 `setDirectorCameraPoseThreeJs(eye, target)`，Client 端零本地 Director 推算。
     - 切回 `pov` / `wasmPov` / `rearView` 或退出 Goal Replay 时，自动释放 `'vnext-broadcaster'` 覆盖并同步 `focusedCarIndex`、`ballCam` 与 `hudStore`。
     - 对 `txSeq` 乱序与非有限坐标 (`NaN`/`Infinity`) 做护栏拦截。

4. **单元测试 (`scripts/tests/vnextHudCameraConsumer.test.ts`)**
   - 覆盖 3v3 观战 1~6 切换不跳边、Goal Replay Director 清空焦点与 Ghost 排除、Broadcaster 电影机位应用与回退 POV，3/3 全部通过。

---

## 二、Wave 2 协议对齐字段清单（W1-08 诉求）

### 通道 1（高频 `WorldFrame` 帧头 & 相机/HUD 语义字段）
| 字段名 | 类型 / 建议编码 | 用途 |
|---|---|---|
| `txSeq` | `uint32` | 单调发送序号，防乱序/回退覆盖相机 |
| `metadataHash` | `uint32` (4B HEX) | 与本地 `RosterManifest` 对比，变更时触发通道 2 补拉 |
| `phase` | `uint8` enum | 比赛阶段（含 `replay` / `goal` / `playing` / `paused` 等） |
| `playbackState` | `uint8` enum | `playing` / `paused` / `rewinding` / `ended` |
| `clockSeconds` | `float32` / `int16` | 比赛/加时时钟显示 |
| `isOvertime` | `bit` (1b) | 加时标记 |
| `blueScore` / `orangeScore` | `uint8` x 2 | 顶部记分板比分 |
| `cameraFocus.focusedCarIndex` | `int8` (`-1` 表示无聚焦车辆) | 驱动顶部 1~6 HUD 焦点高亮与 Client POV 目标 |
| `cameraFocus.cameraMode` | `uint8` enum (`pov` / `wasmPov` / `rearView` / `free` / `director` / `podium` / `editor`) | Client 相机模式仲裁 |
| `cameraFocus.ballCam` | `bit` (1b) | Ball Cam 指示器与 POV 跟随状态 |
| `cameraFocus.isGoalReplayActive` | `bit` (1b) | Goal Replay 状态机开关 |
| `cameraFocus.replayCameraMode` | `bit` (`director` \| `pov`) | Goal Replay 内电影机位 / 球员 POV 切换 |
| `cameraPose.eyeThreeJs` | `float32[3]` (12B，按 Bitmask 可选) | Broadcaster 下发的 Goal Replay 电影机位 / FreeCam 相机位置 |
| `cameraPose.targetThreeJs` | `float32[3]` (12B，按 Bitmask 可选) | Broadcaster 下发的 Goal Replay 电影机位 / FreeCam 注视点 |
| `cameraPose.fovDeg` | `float32` / `uint8` (可选) | Director / FreeCam 水平 FOV |
| `carTelemetry[0..5]` | 每车 `boostAmount (uint8)` + `isDemolished (bit)` + `demoRespawnRemainingSec (float32/uint8)` + `replaySkipReady (bit)` | 1~6 HUD Boost 条、爆破倒计时与 Skip Replay 勾选标记 |

### 通道 2（低频 `RosterManifest` 清单字段）
| 字段名 | 类型 | 用途 |
|---|---|---|
| `metadataHash` | `uint32` (4B) | 清单版本哈希 |
| `matchMode` | `string` / `uint8` enum | 模式标识（含 `exp-multi-bots`） |
| `isSpectatorSession` | `boolean` | 标识当前是否为纯观战/Watch Together 会话（决定是否允许本地人类阵营翻转左右栏） |
| `entries[].carIndex` | `uint8` (`0..5`) | 车辆物理槽位索引，关联通道 1 `cameraFocus.focusedCarIndex` 与 `carTelemetry` |
| `entries[].name` | `string` | 1~6 HUD 球员/Bot 名称 |
| `entries[].team` | `0 \| 1` (1b) | 权威阵营（`0=Blue`, `1=Orange`），彻底替代本地 `playerCarIndex` 推算 |
| `entries[].roleFlag` | `'player' \| 'bot' \| 'ghost'` (2b) | 区分人类、Bot 与幽灵车（HUD 1~6 排除 `ghost`） |
| `entries[].isBot` | `boolean` | Bot 标记（Goal Replay 自动跳过标记等） |
| `entries[].isLocalPlayer` | `boolean` (可选) | 仅当本端真实操控该车时为 `true` |
| `entries[].slotKey` | `1..6` (可选) | 显式 1~6 槽位序号（缺省按 `team` + `carIndex` 确定性分配） |

---

## 三、验证结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（`STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified`）
- `env -u NODE_OPTIONS node --experimental-strip-types --test scripts/tests/vnextHudCameraConsumer.test.ts`：**3/3 PASS**
