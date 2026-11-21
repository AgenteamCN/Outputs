# Wave 1 分支 `W1-07` (`feat/consumer-audio-arena`) 交付报告与协议字段清单

- **分支**：`feat/consumer-audio-arena`（基于 `origin/z/agy/rc/multiplayer-recheck-base` `7c80b44`）
- **最新提交**：`a12f21f` (`feat(audio,arena): consume playbackState silenceAll and RosterManifest rebuild (W1-07)`)
- **状态**：已同步至 `origin/feat/consumer-audio-arena`，工作区干净，门禁与单元测试全绿。

---

## 一、实现内容核验

1. **音频消费端：`playbackState !== 'playing'` 即刻静音**
   - `src/game/pipeline/FramePipeline.ts`：新增 `VNextPlaybackState = 'playing' | 'paused' | 'ended'`，在 `FrameContext` 与 `AudioStage.execute()` 中从 `runtime.playbackState ?? runtime.networkReconciler?.playbackState ?? runtime.match?.state?.playbackState` 透传 `playbackState`。
   - `src/audio/RemoteEngineAudioLodController.ts`：新增 `silenceAll()`，清空 `_awakeRemoteIndices`、将所有 `engineSynths` 置为 `dormant = true`、调用 `synth.silence()` 并关闭 `spatialBus`。
   - `src/game/subsystems/GameAudioSubsystem.ts`：新增 `silenceAll()` 统一关停 `engineLodController` 与 `gameAudio`；在 `update(context)` 入口处判定 `playbackState !== 'playing'` 时 1 帧内立即执行 `silenceAll()` 并更新 `lastPlaybackState` 与遥测状态，跳过后续空间音频与引擎合成更新。

2. **Arena 消费端：`RosterManifest` 动态重建 1~6 车、配色、NameTag 与 `ghost` 材质预留**
   - `src/entities/arena/arenaRosterManifestPipeline.ts`：实现 `applyRosterManifestToArena(arena, manifest, options)`：
     - 基于 `metadataHash` 幂等短路（`arena.lastRosterMetadataHash === manifest.metadataHash` 且未设 `force` 时直接跳过）；
     - 遍历 `manifest.cars`（1~6+ 槽位），调用 `vehicleRegistry.ensureCar(carIndex, team, visualId)` 动态创建或替换车身（Fennec/Merc/Octane 等）；
     - 处理 `colorHex`（无自定义颜色时回退 `team` 阵营色）、`paintFinish`、`decalId`、`boostVisual`、`boostAudio`、`isBot`；
     - 消费 `roleFlag: 'player' | 'bot' | 'ghost'`，当 `roleFlag === 'ghost'` 时通过 `vehicleRegistry.setCarGhostOpacity(carIndex, true, ghostOpacity ?? 0.55)` 套用半透明材质并在 `carRoot.userData.isGhost` / `arena.carRoleFlags` 中标记；
     - 隐藏不在当前 `manifest.cars` 中的多余槽位，并输出 `NameTagCarEntry[]` 通过 `options.onUpdateNameTags` 同步 NameTag 与 `focusedCarIndex`。
   - `src/entities/ArenaWorld.ts`：挂载 `carRoleFlags`、`lastRosterMetadataHash` 与 `applyRosterManifest(manifest, options)` 入口。

---

## 二、测试与门禁结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（1429/1429 TS，0 循环依赖，0 越权向上导入，0 类型错误）
- `npx tsx scripts/testConsumerAudioArena.ts`：**3/3 PASS**
  - `[PASS] GameAudioSubsystem immediately silences all synths and gameAudio when playbackState !== playing`
  - `[PASS] AudioStage propagates runtime.playbackState into FrameContext`
  - `[PASS] applyRosterManifestToArena rebuilds 1-6 cars, colors, NameTags, ghost materials, and respects metadataHash`

---

## 三、Wave 2 协议对齐字段清单（`W1-07` 消费诉求）

供 `feat/protocol-schema-alignment` 在设计二进制 `WorldFrame` / `RosterManifest` / `Signal` 时对齐：

### 1. 通道 1（高频 State 帧头 `WorldFrame`）
| 字段名 | 类型 / 位宽建议 | 用途说明 |
|---|---|---|
| `playbackState` | `uint8` / enum (`0=playing, 1=paused, 2=ended`) | 驱动 `AudioStage` -> `GameAudioSubsystem.update()`；非 `playing` 时 1 帧内 `silenceAll()` |
| `metadataHash` | `uint32` (4B HEX) | 与本地 `arena.lastRosterMetadataHash` 比对，不一致时触发异步拉取 `RosterManifest` 并重建车身/NameTag |
| `focusedCarIndex` | `uint8` (0~7) | 驱动主视角引擎音频路由（非空间直通 vs 3D 空间化）与 NameTag 焦点判定 |

### 2. 通道 2（低频清单 `RosterManifest`）
| 字段名 | 类型 / 建议格式 | 用途说明 |
|---|---|---|
| `metadataHash` | `uint32` (4B HEX) | 清单版本号，用于 `applyRosterManifestToArena` 幂等校验 |
| `mode` | `string` / enum | 当前模式上下文（含 `Exhibition` 等） |
| `focusedCarIndex` | `uint8` (optional) | 当前聚焦车号 |
| `cars[].carIndex` | `uint8` (0~7) | 车辆槽位索引（支持 1~6 人局及最多 8 槽） |
| `cars[].name` | `string` | NameTag 显示名称 |
| `cars[].team` | `uint8` (`0 \| 1`) | 阵营（蓝/橙），用于默认涂装与 NameTag 阵营色 |
| `cars[].visualId` | `string` / enum | 车身模型 ID（`fennec`、`octane`、`dominus`、`merc`、`breakout` 等） |
| `cars[].colorHex` | `uint32` / `string` (optional) | 自定义车身颜色（缺省按 `team` 回退） |
| `cars[].paintFinish` | `string` / enum (optional) | 漆面质感（`metallic` / `matte` / `null`） |
| `cars[].decalId` | `string` (optional) | 贴花 ID |
| `cars[].boostVisual` | `string` (optional) | 尾焰特效类型 |
| `cars[].boostAudio` | `string` (optional) | 尾焰音效类型 |
| `cars[].isBot` | `bool` (optional) | Bot 标识（驱动 Boost 粒子等 Bot 区分逻辑） |
| `cars[].roleFlag` | `uint8` / enum (`0=player, 1=bot, 2=ghost`) | 角色标识；`ghost` 自动套用半透明材质并标记 `isGhost` |
| `cars[].ghostOpacity` | `float32` / `uint8` (optional) | 幽灵车透明度（默认 `0.55`） |
| `cars[].isLocalPlayer` | `bool` (optional) | 本地玩家标记 |
