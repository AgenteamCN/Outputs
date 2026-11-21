# Wave 1 深度核查评估 & Wave 2 协议对齐落地报告

## 一、Wave 1 全分支 Commit 深度核查结论

对 `origin/z/agy/rc/multiplayer-recheck-base` (`7c80b44`) 上的 10 条 Wave 1 分支逐一核查 `git log` / `git diff --stat` / 关键接口源码后，结论如下：

1. **整体代码质量评估：无需推倒大改版**
   - `W1-00` (`chore/strip-legacy-spectate`) 已合入基线 `7c80b44`。
   - `W1-01` ~ `W1-09` 全部干净基于 `7c80b44` 分出，职责边界清晰，均配有独立单元测试脚本（`scripts/test*.ts`）。
   - **唯一明显瑕疵在 `W1-09` (`feat/bulk-asset-channel`)**：存在两次提交（`e697bca` 与 `283b3af`）。第二次提交新建了 `src/network/transport/BulkAssetChannel.ts` 并重写了 `BulkAssetProtocol.ts`，但遗留了首次提交的 `src/network/transport/webrtc/BulkAssetChannel.ts` 与 `src/network/transport/webrtc/bulkAssetAssemblyPipeline.ts` 两个废弃文件，Wave 3 对齐时需顺手删除。

2. **能否跳过 Wave 2/3 直接 Octopus Merge？—— 不能，原因有三：**
   - **存在 3 个跨分支重叠文件**，直接 `git merge -s octopus` 会冲突中止：
     - `src/network/server/activities/ActivityKernel.ts`（同时被 `W1-03`、`W1-04`、`W1-05` 修改）
     - `src/replay/FreeplayRewinderController.ts`（同时被 `W1-04`、`W1-09` 修改）
     - `scripts/runMultiplayerSuiteTests.ts`（同时被 `W1-02`、`W1-06`、`W1-09` 修改）
   - **协议尚未统一**：`W1-01`~`W1-08` 各自定义了临时 TS 接口（如 `WorkerChannel1StateFrame`、`TimelineFrameHeader`、`MatchExhibitionRosterManifest`、`freeplaySpectateTypes`、`trainingFpvDualVNextTypes`、`vnextHudCameraTypes`），彼此之间尚未接入统一的二进制 `WorldFrame` / `RosterManifest` Codec 与 `schemaHash` 校验。
   - **按原定 Wave 2 -> Wave 3 -> Wave 4 推进最稳妥**：先合流干净底座 `integration/vnext-core`，再在各分支内反向对齐并消化 3 个重叠文件，最后 Octopus Merge 一次过门禁。

---

## 二、本次已完成的 Wave 2 工作（已推送远端）

已在 `Antigravi-ty/RocketCompanion` 创建并推送（Verified Bot Commit `9b5d855`，GPG `VALID`）：
- `origin/feat/protocol-schema-alignment`
- `origin/integration/vnext-core`

### 核心交付物
1. **`src/network/protocol/VNextSchemaHash.ts`**
   - 基于通道 1/2 二进制字段布局表自动计算 4 字节确定性 `schemaHash`：
     - Channel 1 (`VNEXT_CHANNEL1_SCHEMA_HASH`): `0x49D215C8`
     - Channel 2 (`VNEXT_CHANNEL2_SCHEMA_HASH`): `0x9063ED98`
   - 两端字段只要差 1 bit，`decode` 第一行立即抛错拦截。
2. **`src/network/protocol/VNextWorldFrameCodec.ts`**（通道 1 高频状态流）
   - `DataView` 零分配编解码，统一涵盖：`schemaHash`、`txSeq`、`metadataHash`、`activityMode`、`playbackState`、`phase`、`simTick`、`replayFrameIndex`、`replayTotalFrames`、`clockSeconds`、`playbackSpeed`、`blueScore`/`orangeScore`、`focusedCarIndex`、`cameraMode`、`timestampMs`。
   - 按位掩码可选携带：`fpvAuthority`（`authorityEpoch` / `activeInputPeerSlot` / `authorityMode`，供 W1-06）、`trainingHeader`（`shotIndex` / `shotResetSeq` 等，供 W1-06）、`cameraPose`（`eye` / `target` / `fovDeg`，供 W1-02/04/08），并内嵌复用 `GlobalPayloadCodec`（`PHYSICS` / `SIGNAL` / `ALL_RTT`）。
3. **`src/network/protocol/VNextRosterManifestCodec.ts`**（通道 2 低频清单）
   - 统一 `VNextRosterManifest` 编解码与确定性 32-bit FNV-1a `computeRosterMetadataHash()`，覆盖 1~6 号车外观/涂装/Boost/Anthem、`roleFlag: 'player' | 'bot' | 'ghost'`、`ghostOpacity`、`arenaPreset`、`telestratorAllowed` 与 `trainingMeta`。
4. **`src/network/protocol/ReplicationBusProtocol.ts`**
   - 补齐 `ACTIVITY_IDS.EXHIBITION = 'exp-multi-bots'` 及其默认 `ACTIVITY_DEFAULT_PACKET_FLAGS`。
   - 新增 `decodeVNextChannelPacket()` 统一按 `schemaHash` 分发通道 1/2 数据包。
5. **`scripts/testVNextProtocolAlignment.ts`**
   - 验证 `schemaHash` 错位拦截、Channel 1/2 完整往返编解码、`metadataHash` 变更灵敏度与 `EXHIBITION` 活动 ID。

### 验证结果
- `node --experimental-strip-types scripts/testVNextProtocolAlignment.ts`：**PASS** (`Ch1=0x49D215C8, Ch2=0x9063ED98`)
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1431/1431 TS，0 错误，0 循环依赖，0 孤儿模块）

---

## 三、后续 Wave 3 & Wave 4 详细规划

1. **Wave 3：各业务分支反向对齐 `origin/integration/vnext-core`**
   - 9 条分支分别执行 `git merge origin/integration/vnext-core`：
     - **W1-01 (`worker-authority-bus`)**：将 `WorkerChannel1StateFrame.payloadBytes` 与 `WAVE1_CHANNEL1_SCHEMA_HASH` 替换为 `VNextWorldFrameCodec` / `VNEXT_CHANNEL1_SCHEMA_HASH`。
     - **W1-02 (`timeline-frame-buffer`)**：`TimelineFrameHeader` / `TimelineCameraPose` 对齐 `DecodedVNextWorldFrame` 字段。
     - **W1-03 (`mode-match-exhibition`)** & **W1-04 (`mode-freeplay-spectate`)** & **W1-05 (`mode-replay-watch-together`)**：统一输出 `VNextRosterManifest`（复用 `computeRosterMetadataHash`）与 `VNextWorldFrameEncodeInput`。
     - **W1-06 (`mode-training-fpv-dual`)**：`fpvAuthority` / `trainingHeader` / `trainingMeta` 接入 `VNextWorldFrameCodec` 与 `VNextRosterManifestCodec`。
     - **W1-07 (`consumer-audio-arena`)** & **W1-08 (`consumer-hud-camera`)**：直接消费 `DecodedVNextWorldFrame` 与 `VNextRosterManifest`（删除各自临时 `RosterManifest` / `vnextHudCameraTypes` 重复定义）。
     - **W1-09 (`bulk-asset-channel`)**：删除 `e697bca` 遗留的 `src/network/transport/webrtc/BulkAssetChannel.ts` 与 `bulkAssetAssemblyPipeline.ts` 死文件。
   - **重叠文件收敛建议**：将 `ActivityKernel.ts`、`FreeplayRewinderController.ts`、`runMultiplayerSuiteTests.ts` 的 3 处增量修改在合流分支或指定先行分支中统一协调一次，避免 Octopus Merge 冲突。

2. **Wave 4：Octopus Merge & 全量门禁**
   - 执行 `git merge -s octopus` 合入全部 Wave 3 绿灯分支；
   - 运行 `pnpm run check`、`RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`、`pnpm run repro:dev-broadcast-v3`。
