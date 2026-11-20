# W1-07 (`feat/consumer-audio-arena`) Wave 1 交付报告与字段需求清单

- **分支**：`feat/consumer-audio-arena`（基于 `z/agy/rc/multiplayer-recheck-base` `7c80b44`）
- **Verified Bot Commit**：`a12f21f60952e2c697094baa82e4024e81a11130`（GitHub GraphQL `createCommitOnBranch`, `signature.state = VALID`）

---

## 一、已完成工作摘要

### 1. `GameAudioSubsystem` 消费 `playbackState !== 'playing'` 立即 `silenceAll()`
- **`src/audio/RemoteEngineAudioLodController.ts`**：新增 `silenceAll()` 方法，立即将所有引擎合成器置为 `dormant = true`、调用 `synth.silence()` 并关闭 `spatialBus`（区别于 `resetAll()`，暂停时不触发重生音效）。
- **`src/game/pipeline/FramePipeline.ts`**：定义 `VNextPlaybackState = 'playing' | 'paused' | 'ended'`，在 `FrameContext` 与 `FramePipelineRuntimePort` 中新增 `playbackState` 字段，并在 `AudioStage.execute()` 中从 `runtime.playbackState` / `networkReconciler.playbackState` / `match.state.playbackState` 同步写入 `context.playbackState`。
- **`src/game/subsystems/GameAudioSubsystem.ts`**：
  - 新增 `silenceAll()` 方法，同时静音 `engineLodController`（全部主/远端引擎合成器）与 `gameAudio`（`ballEngine`、`vehicleManager`、`AudioSlotPool`）。
  - 在 `update(context)` 入口检测 `playbackState !== 'playing'`，在 **1 帧内**立即执行 `this.silenceAll()` 并更新遥测后提前返回；恢复 `'playing'` 后下一帧自动唤醒引擎与球声更新。
  - 修复 `updateVehicles` 分支统一使用解析后的 `audio` 门面（`r.options?.gameAudio || gameAudio`）。

### 2. `Arena` 消费 `RosterManifest` 动态重建 1~6 车身、配色与 NameTag（含 `ghost` 材质预留）
- **`src/entities/arena/arenaRosterManifestPipeline.ts`**：定义临时 Wave 1 `RosterManifest` / `RosterManifestCarEntry` / `RosterCarRoleFlag` TS 接口，并实现 `applyRosterManifestToArena(arena, manifest, options)`：
  - **`metadataHash` 幂等拦截**：当 `manifest.metadataHash === arena.lastRosterMetadataHash` 时直接跳过冗余重建（支持 `options.force` 强制刷新）。
  - **1~6 车动态重建与换装**：按 `carIndex` 调用 `vehicleRegistry.ensureCar(carIndex, team, visualId)`，自动处理新增槽位、`visualId` 车身模型热切换与阵营同步。
  - **配色与 Loadout**：支持自定义 `colorHex`（含清空自定义色回退阵营默认色）、`paintFinish`、`boostVisual`、`boostAudio`、`isBot`。
  - **`ghost` 材质预留（`roleFlag === 'ghost'`）**：自动调用 `vehicleRegistry.setCarGhostOpacity(carIndex, true, ghostOpacity ?? 0.55)` 套用半透明材质并标记 `carRoot.userData.isGhost = true` / `roleFlag = 'ghost'`；切回 `'player'` / `'bot'` 时恢复不透明材质。
  - **空缺槽位隐藏**：缩容（如 6 车 -> 2 车）时自动隐藏未在清单中的槽位（`visible = false`）。
  - **NameTag 数据输出**：生成标准 `NameTagCarEntry[]`（`carIndex`、`name`、`team`），通过 `onUpdateNameTags` 回调与返回值 `nameTagEntries` 交由上层 `MatchRosterProjection`（SSOT）写入，保持 `NameTagScatter=1/1` 护栏。
- **`src/entities/ArenaWorld.ts`**：新增 `carRoleFlags`、`lastRosterMetadataHash` 与 `applyRosterManifest(manifest, options)` 公开方法，并导出相关类型。

---

## 二、测试与门禁验证

1. **单元测试**：`env -u NODE_OPTIONS node scripts/runTsWithVite.mjs scripts/testConsumerAudioArena.ts`
   - `[PASS] GameAudioSubsystem immediately silences all synths and gameAudio when playbackState !== playing`
   - `[PASS] AudioStage propagates runtime.playbackState into FrameContext`
   - `[PASS] applyRosterManifestToArena rebuilds 1-6 cars, colors, NameTags, ghost materials, and respects metadataHash`
2. **全量静态门禁**：`pnpm run check`（`radar` + `tsc --noEmit`）
   - `STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified.`
   - `NameTagScatter=1/1`、`0 Circular Deps`、`0 Value Re-Exports`、`0 Upward Imports`、`Any<=5`。

---

## 三、Wave 2 协议对齐字段需求清单（供 `feat/protocol-schema-alignment` 汇总）

### 通道 1（高频 State 流，每帧头部/语义字段）
| 字段名 | 类型 / 建议位宽 | 消费方 | 用途说明 |
|---|---|---|---|
| `playbackState` | enum (`'playing' \| 'paused' \| 'ended'`, 2 bits) | `GameAudioSubsystem` / `AudioStage` | `!== 'playing'` 时 1 帧内立即触发 `silenceAll()`，恢复 `'playing'` 后无缝恢复引擎与球声 |
| `metadataHash` | `uint32` (4 Bytes) | `ArenaWorld.applyRosterManifest` | 与本地 `lastRosterMetadataHash` 比对，仅在变化时触发通道 2 `RosterManifest` 异步拉取与重建 |
| `focusedCarIndex` | `uint8` (3~4 bits, `0..7`) | `VehicleNameTagSystem` / `AudioListener` | 决定主视角引擎合成器（非空间化）与 NameTag 单字符/距焦判断 |

### 通道 2（低频 `RosterManifest` 清单，按 `metadataHash` 变更触发）
| 字段名 | 类型 | 消费方 | 用途说明 |
|---|---|---|---|
| `metadataHash` | `uint32` (4 Bytes) | `ArenaWorld` | 清单版本校验与幂等缓存键 |
| `cars[].carIndex` | `uint8` (`0..5`, 预留 `0..7`) | `ArenaVehicleRegistry` / `NameTag` | 车辆槽位索引 |
| `cars[].name` | `string` (UTF-8) | `VehicleNameTagSystem` | 场内 NameTag 展示名 |
| `cars[].team` | `0 \| 1` (1 bit) | `ArenaVehicleRegistry` / `NameTag` | 阵营（0 蓝 / 1 橙），决定默认车身色与 NameTag 阵营色 |
| `cars[].visualId` | `string` / `uint8` 枚举 | `ArenaVehicleRegistry` | 车身模型 ID（`fennec`、`octane`、`merc` 等），触发动态建车或热换模型 |
| `cars[].colorHex` | `uint32` RGB / `null` | `ArenaVehicleLoadoutManager` | 自定义车身配色；为 `null` 时回退阵营默认色 |
| `cars[].paintFinish` | `'metallic' \| 'matte' \| null` | `ArenaVehicleLoadoutManager` | 漆面质感 |
| `cars[].decalId` | `string \| null`（预留） | `ArenaVehicleLoadoutManager` | 涂装标识预留 |
| `cars[].boostVisual` | `string \| null` | `VehicleBoostEmitter` | 氮气粒子外观预设 |
| `cars[].boostAudio` | `string \| null` | `GameAudioEngineFacade` | 氮气音效变体 |
| `cars[].roleFlag` | `'player' \| 'bot' \| 'ghost'` (2 bits) | `ArenaWorld` / `ArenaVehicleRegistry` | 角色标识；`'ghost'` 时自动应用半透明幽灵车材质并标记 `isGhost` |
| `cars[].ghostOpacity` | `float32` / `uint8`（可选，默认 `0.55`） | `ArenaVehicleLoadoutManager` | 幽灵车透明度 |
