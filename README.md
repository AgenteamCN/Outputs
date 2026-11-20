# Wave 1 (`W1-01`) — `feat/worker-authority-bus` 交付报告

- **分支**：`feat/worker-authority-bus`（基于 `origin/z/agy/rc/multiplayer-recheck-base` `7c80b44`）
- **Verified Bot Commit**：`d46b6746b4333a480b8309f8b5711a0b32237692` (`signature.isValid = true`, `state = VALID`)

## 1. 核心交付内容

1. **`StateFrameSlidingWindow` (`src/network/server/StateFrameSlidingWindow.ts`)**
   - 实现通道 1 高频 State 流的 **10 帧滑窗**（`DEFAULT_SLIDING_WINDOW_CAPACITY = 10`）。
   - 每帧分配严格单调递增的 `txSeq`（`uint32`），**完全解耦 `simTick` / `replayFrameIndex`**，确保正放、暂停、倒放（Rewind）、跳帧（Seek）对称一致。
   - 实现按 Session 裁剪：`getUnackedFramesForSession(lastAckedTxSeq)`（好网仅发 1 帧增量，弱网自动补发滑窗内未确认尾帧），以及全员确认全局裁剪 `pruneAckedByAllSessions(minAckedTxSeq)`。

2. **`WorkerAuthorityBus` (`src/network/server/WorkerAuthorityBus.ts`)**
   - 统一托管 **Match / Freeplay / Exhibition / Replay** 四大模式的 Worker 侧步进与广播推流循环：
     - 独立 `setTimeout` 定时器循环（脱离主线程 `requestAnimationFrame`，免疫 Host 切后台/最小化时的 1Hz rAF 节流）。
     - 固定物理步进（默认 120Hz）+ 可配置通道 1 广播频率（`15Hz / 30Hz / 60Hz / 120Hz`）。
     - 内置 Replay 游标控制（`playing` / `paused` / `rewinding` / `seeking`、倍速、总帧数、FPS），倒放/跳帧时 `txSeq` 依然严格单调递增。
   - 每次推流自动写入 `StateFrameSlidingWindow` 并按 `EstablishedSessionManager` 记录的 `lastAckedTxSeq` 分发裁剪后的帧批次（跳过虚拟 Bot 通道）。

3. **`PeerSession` & `EstablishedSessionManager` Ack 追踪**
   - `src/network/server/PeerSession.ts`：新增 `lastAckedTxSeq` 与单调保护方法 `_recordTxSeqAck(ackedTxSeq)`（乱序旧包不回退）。
   - `src/network/server/EstablishedSessionManager.ts`：新增 `recordClientTxSeqAck`、`getSessionLastAckedTxSeq`、`getMinHumanAckedTxSeq`（仅统计人类 Session，忽略 `bot` / `bot_*`）。

4. **Worker 协议与路由接入**
   - `src/network/protocol/HostWorkerProtocol.ts`：新增 `WORKER_BUS_CONFIGURE`、`WORKER_BUS_REPLAY_CONTROL`、`WORKER_BUS_TX_SEQ_ACK`、`CHANNEL1_STATE_FRAME_BATCH`。
   - `src/network/server/workerActivityAndInputRouter.ts` & `workerReplayAndDiagnosticsRouter.ts`：路由上述 Worker Bus 控制与 `txSeq` Ack 消息。
   - `src/network/server/AuthoritativeServerWorker.ts`：初始化 `WorkerAuthorityBus`，并在 `tick()` 与 `destroy()` 中接入滑窗记录与生命周期清理。

5. **Wave 1 独立单测 (`scripts/testWorkerAuthorityBus.ts`)**
   - 覆盖：10 帧滑窗容量与淘汰、好网 1 帧增量/弱网多帧补发、倒放与跳帧 `txSeq` 单调对称性、多 Session 差异化 Ack 裁剪与乱序防回退、Worker `setTimeout` 无 rAF 连续推流、4 模式步进与 Replay 正/倒放/暂停/Seek、Worker 路由集成。

---

## 2. 提交给 Wave 2 (`feat/protocol-schema-alignment`) 的通道 1 数据包字段清单

`W1-01`（Worker Authority Bus & 通道 1 滑窗分发）需要通道 1（State Stream）与上行 Ack 包携带以下字段：

### 下行：通道 1 `WorldFrame` 包头与帧语义字段
| 字段名 | 建议类型 | 字节数 | 说明 |
|---|---|---|---|
| `schemaHash` | `uint32` | 4B | 协议结构哈希（HEX，如 `0x57313031` 占位），收包首行校验防静默错位 |
| `txSeq` | `uint32` | 4B | 严格单调递增传输序号（滑窗裁剪与丢包补发唯一依据） |
| `metadataHash` | `uint32` | 4B | 当前 `RosterManifest` + 房间上下文哈希，变化时触发通道 2 重拉 |
| `activityMode` | `uint8` (enum) | 1B | `0=match`, `1=freeplay`, `2=exhibition`, `3=replay` |
| `playbackState` | `uint8` (enum) | 1B | `0=playing`, `1=paused`, `2=rewinding`, `3=seeking` |
| `phase` | `uint8` / string | 1B | 房间/比赛阶段（kickoff / active / goal_scored / goal_replay / overtime / ended 等） |
| `clockSeconds` | `float32` | 4B | 比赛/自由模式时钟秒数 |
| `simTick` | `uint32` | 4B | 物理仿真 Tick |
| `replayFrameIndex` | `uint32` | 4B | Replay 模式当前帧索引（非 Replay 模式为 `0`） |
| `focusedCarIndex` | `uint8` | 1B | 当前相机焦点车号（1~6 车索引） |
| `timestampMs` | `float64` / `uint32` | 4B~8B | 权威端发包时间戳 |
| `sectionFlags` | `uint16` | 2B | 位掩码标识本帧包含的 Section（球、1~6 车位姿/速度/Boost、BoostPad Bitmask、相机位姿等） |

### 上行：Client 回报 Ack 字段（可 Piggyback 于输入包或独立心跳包）
| 字段名 | 建议类型 | 字节数 | 说明 |
|---|---|---|---|
| `ackedTxSeq` | `uint32` | 4B | Client 最新收到的通道 1 `txSeq`，供 `EstablishedSessionManager` 反向裁剪滑窗 |

---

## 3. 验证结果

- `env -u NODE_OPTIONS node --experimental-strip-types scripts/testWorkerAuthorityBus.ts`：**5/5 全部 PASS**
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 越界引用）
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**28/28 全部 PASS**
