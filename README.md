# Wave 1 `W1-09` (`feat/bulk-asset-channel`) 交付报告

- **分支**：`feat/bulk-asset-channel`（基于 `origin/z/agy/rc/multiplayer-recheck-base` @ `7c80b445f2953076e5356320c6e91bc04da4bba8`）
- **Verified Bot Commit**：`e697bcae02e2c0308aec3745d30c26800f55d346`（GitHub GraphQL `createCommitOnBranch`, `signature.state = VALID`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

---

## 1. 核心交付内容

1. **第 4 通道二进制协议与临时 TS 接口（`src/network/protocol/BulkAssetProtocol.ts`）**
   - 定义 `BULK_FRAME_MAGIC = 0x424b` (`'BK'`) 与 32 字节定长包头（`MANIFEST` / `CHUNK` / `COMPLETE_ACK` / `ABORT`）。
   - 实现同步 64-bit FNV-1a 校验函数 `computeBulkFileHash64`（拆分为 `hi`/`lo` 双 `Uint32` + 16 位 hex 字符串），零外部依赖，兼容 Worker / Node / Browser。
   - 预留 `BulkAssetKind`：`rewinder_slice`、`ghost_trajectory`、`replay_file`、`training_pack`，并提供 `GhostTrajectoryBulkDescriptor` 与 `ReplayFileBulkDescriptor`。

2. **第 4 通道可靠分片传输与背压限流（`src/network/transport/webrtc/BulkAssetChannel.ts` + `bulkAssetAssemblyPipeline.ts`）**
   - 独立绑定 `RTCDataChannel`（label: `game_bulk`, `{ ordered: true }`），与 `game_net` / `game_ctrl` 物理隔离，避免队头阻塞。
   - 基于 `bufferedAmount` 实现高低水位背压限流（默认 `highWaterMarkBytes = 256 KiB`、`lowWaterMarkBytes = 64 KiB`、`maxChunksPerBurst = 8`），支持 `onbufferedamountlow` 事件驱动与定时轮询兜底。
   - 接收端重组后执行 `fileHash` 校验：一致则回传 `COMPLETE_ACK` 并触发 `onAssetReceived`；不一致则立即发送 `ABORT(HASH_MISMATCH)` 并丢弃脏分片。

3. **WebRTC 传输层接线**
   - `src/network/transport/webrtc/HandshakeSession.ts`：在 `createOfferToken` 中与 `game_net`、`game_ctrl` 并行创建 `game_bulk` DataChannel。
   - `src/network/transport/webrtc/DataChannelTransport.ts`：`bindDataChannel` 识别 `game_bulk` 并路由至 `this.bulkChannel`，不污染主状态通道与控制通道。
   - `src/network/transport/P2PWebRTCChannel.ts`：暴露 `readonly bulkAssetChannel: BulkAssetChannel`。

4. **`FreeplayRewinderController` 历史切片 / Intervene-From-Here 接入**
   - `src/replay/rewinderBulkSliceCodec.ts`：二进制编解码 `RewinderFrameEntry[]`（`compactSnap`、`renderState`、`hadTouch`、`lastTouchTick`、`controls`、`acknowledgedControls`、`incidents`、`signals`、`audioEvents`）及游标/切片元数据（`REWINDER_SLICE_MAGIC = 0x5257534c`）。
   - `src/replay/FreeplayRewinderController.ts`：新增 `exportRewinderSliceBytes`、`sendRewinderSliceToBulkChannel`、`applyIncomingRewinderSliceBytes`。

---

## 2. 提交给 Wave 2 (`feat/protocol-schema-alignment`) 的字段诉求清单

| 所属通道/对象 | 字段名 | 类型 / 字节 | 用途说明 |
|---|---|---|---|
| **Channel 4 Header** (`BulkAssetFrame`) | `magic` | `uint16` (2B, `0x424B`) | 识别第 4 通道 Bulk 帧 |
| | `frameType` | `uint8` (1B) | `1=MANIFEST`, `2=CHUNK`, `3=COMPLETE_ACK`, `4=ABORT` |
| | `assetKind` | `uint8` (1B) | `1=rewinder_slice`, `2=ghost_trajectory`, `3=replay_file`, `4=training_pack` |
| | `transferId` | `uint32` (4B) | 传输会话唯一 ID，支持并发交错传输 |
| | `seqOrChunkIndex` | `uint32` (4B) | `CHUNK` 为分片序号；`MANIFEST`/`COMPLETE_ACK` 为 `totalChunks`；`ABORT` 为 `reasonCode` |
| | `totalBytes` | `uint32` (4B) | 资产总字节数（或当前分片字节数） |
| | `fileHashHi` / `fileHashLo` | `uint32 + uint32` (8B) | 64-bit FNV-1a 完整性校验值 |
| | `chunkSizeOrReserved` | `uint32` (4B) | `MANIFEST` 携带分片大小；`CHUNK` 携带字节偏移 |
| | `bodyByteLength` | `uint32` (4B) | 紧随包头的载荷长度 |
| **Channel 4 Manifest Meta** | `fileName`, `roomId`, `startFrame`, `endFrame`, `cursor`, `tickRate`, `frameCount`, `carIndex`, `durationSec` | UTF-8 JSON (变长) | 描述 Rewinder 切片、Ghost 轨迹、Replay 文件元信息 |
| **Channel 1 / 2 预留联动字段** | `replayFileHash` (可选) | `uint64` / `8B` | 后期开启“Channel 4 预分发 `.rcreplay.gz`/`.rlrc` + Channel 1 仅同步 `replayFrameIndex`”低带宽模式时，用于校验客户端本地已缓存的录像文件一致性 |

---

## 3. 验证结果

- `pnpm run check` (`radar` + `tsc --noEmit`)：**PASS**（1432/1432 TS 文件，0 循环依赖，0 越界导入，0 Volume HardError）
- `env -u NODE_OPTIONS node scripts/runTsWithVite.mjs scripts/testBulkAssetChannel.ts`：**7/7 PASS**
  1. Chunked transfer & 64-bit `fileHash` verification
  2. Corrupted chunk rejected via `fileHash` mismatch `ABORT`
  3. `bufferedAmount` backpressure throttling & low-watermark resume
  4. `FreeplayRewinderController` slice encode / Channel-4 / apply roundtrip
  5. Reserved Ghost Trajectory & Replay File Channel-4 transfers
  6. `DataChannelTransport` physical isolation (`game_bulk` vs `game_net` / `game_ctrl`)
  7. Size limit guard & transfer abort
