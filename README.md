# Wave 1 — W1-09 (`feat/bulk-asset-channel`) 交付报告

- **分支**：`feat/bulk-asset-channel`
- **Verified Bot Commit**：`283b3af96662c87b5a54d4225abbf5fd7f64e4e8` (`signature.isValid = true`, `state = VALID`)
- **Author**：`antigravity-fine-grained-token[bot]` (`Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)

---

## 一、核心实现内容

1. **Channel 4 二进制分片协议（`src/network/protocol/BulkAssetProtocol.ts`）**
   - 定义 `BULK_FRAME_MAGIC = 0x424b` (`'BK'`)、`BULK_PROTOCOL_VERSION = 1`。
   - 支持 4 类帧：`CHUNK (0x01)`、`COMPLETE_ACK (0x02)`、`HASH_MISMATCH_NACK (0x03)`、`CANCEL (0x04)`。
   - 内置同步零依赖 32-bit FNV-1a `computeBulkFileHash` 与 `formatBulkFileHashHex`（如 `0x8f3a19c4`）。
   - 预留 4 类资产类型：`rewinder_slice`、`ghost_trajectory`、`replay_file`、`training_pack`。

2. **第 4 通道独立传输层（`src/network/transport/BulkAssetChannel.ts`）**
   - **Chunk 分片**：默认 `16384B` 分片（可配），首片携带 UTF-8 JSON 元数据。
   - **`bufferedAmount` 背压限流**：高水位 `262144B` 暂停发送、低水位 `65536B` 恢复（监听 `onbufferedamountlow` + `16ms` 轮询兜底），单次 pump 上限 `8` chunks，防止打爆 SCTP 发送缓冲或阻塞通道 1/2/3。
   - **完整 `fileHash` 校验**：接收端重组完成后计算 FNV-1a 校验和；一致则回传 `COMPLETE_ACK` 并触发 `onAssetReceived`，不一致则丢弃并回传 `HASH_MISMATCH_NACK` 触发 `onTransferError`。

3. **物理隔离的 `game_bulk` DataChannel 接入**
   - `src/network/transport/webrtc/HandshakeSession.ts`：Host 端创建第 3 条独立有序可靠通道 `pc.createDataChannel('game_bulk', { ordered: true })`。
   - `src/network/transport/webrtc/DataChannelTransport.ts` & `dataChannelFramingPipeline.ts`：识别 `game_bulk` 并绑定至 `bulkChannel`，与 `game_net` / `game_ctrl` 物理隔离。
   - `src/network/transport/P2PWebRTCChannel.ts`：暴露 `bulkChannel` getter 供上层直接调用。

4. **`FreeplayRewinderController` 切片传输接入**
   - `src/replay/rewinderBulkSliceCodec.ts`：紧凑二进制编码 `RewinderFrameEntry[]`（`compactSnap` 与 `renderState` 零拷贝 `Float32Array` 字节流 + 控制/事件尾）。
   - `src/replay/FreeplayRewinderController.ts`：新增 `exportRewinderSliceForBulk`、`sendRewinderSliceViaBulkChannel`、`importRewinderSliceFromBulk`。

---

## 二、Wave 1 → Wave 2 协议字段诉求清单

供 Wave 2 `feat/protocol-schema-alignment` 汇总对齐：

1. **Channel 4 `CHUNK` 二进制包头（28 Bytes）**
   - `magic: uint16` (`0x424b`)
   - `version: uint8` (`1`)
   - `frameType: uint8` (`0x01=CHUNK | 0x02=COMPLETE_ACK | 0x03=HASH_MISMATCH_NACK | 0x04=CANCEL`)
   - `transferId: uint32`
   - `assetKind: uint8` (`1=rewinder_slice | 2=ghost_trajectory | 3=replay_file | 4=training_pack`)
   - `flags: uint8` (bit0=compressed 预留)
   - `chunkIndex: uint32`
   - `totalChunks: uint32`
   - `totalBytes: uint32`
   - `fileHash: uint32` (FNV-1a 32-bit)
   - `metadataLength: uint16` (仅 chunk 0 携带，其余为 0)
   - 紧随其后：`metadataBytes` + `chunkPayloadBytes`

2. **Channel 4 资产 Metadata 字段诉求**
   - `RewinderSliceBulkMetadata`: `startFrame`, `endFrame`, `cursorFrame`, `targetFrame`, `frameCount`, `totalRewinderFrames`, `arenaPreset?`
   - `GhostTrajectoryBulkMetadata`: `carIndex`, `playerName`, `team`, `visualId?`, `colorHex?`, `startTick`, `endTick`, `tickCount`, `roleFlag: 'ghost'`
   - `ReplayFileBulkMetadata`: `fileName`, `format ('rcreplay' | 'rlrc')`, `totalFrames`, `durationSeconds`, `arenaPreset?`, `schemaHash?`
   - `TrainingPackBulkMetadata`: `packId`, `packName`, `shotCount`, `schemaVersion`

3. **与通道 1/2/3 的交叉引用诉求**
   - 后期若启用 Replay“传文件 + 仅同步 `frameIndex`”模式，通道 2 `RoomManifest` 需携带 `replayFileHash` (4B) 以校验本地是否已通过通道 4 缓存完整 `.rlrc` / `.rcreplay` 文件。

---

## 三、自动化验证结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1431/1431 TS files，0 errors，0 circular deps，体积护栏全绿）。
- `env -u NODE_OPTIONS node --experimental-strip-types scripts/testBulkAssetChannel.ts`：**PASS**。
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**29/29 PASS**（含新增 Section 29 Channel 4 全链路测试）。
