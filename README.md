# Track D (`z/agy/rc/track-d-replication-bus`) 架构进度审计与测试指南

## 一、 一句话结论

**当前 Track D（commit `cf5cfee`）处于「核心协议、双通道传输基建与编排器骨架已完成，但尚未接入主游戏循环与端到端收发链路」的状态。**

- **如果作为「Phase 1 契约与底层基建 PR」合入 `alpha`**：**可以**。代码为纯增量（14 个文件，+1193 行，0 删除），`pnpm run check`（Radar 2.0 + `tsc --noEmit`）、`prettier --check`（针对 Track D 14 个文件）与 `pnpm run build` 均已通过；唯一在运行时会实际触发的新逻辑是在 WebRTC 握手时额外创建了一条 `game_ctrl` 有序 DataChannel。
- **如果期望合入后就已经能跑通「不断线热切换 `Freeplay <-> Match <-> Replay`、动态包标志位省带宽、Client 端自动跟随切换」**：**还不行**。目前 `src/game/`、`src/ui-core/`、`src/ui-system/` 以及 `scripts/` 中没有任何代码调用 Track D 新增的 API，属于“造好了发动机和变速箱，但还没接上油门和车轮”。

---

## 二、 Track D 到底改了什么？（基于 `cf5cfee` 代码逐文件核对）

| 模块层 | 改动文件 | 实际实现内容 | 当前接线状态 |
| :--- | :--- | :--- | :--- |
| **协议层** | `src/network/protocol/ReplicationBusProtocol.ts`<br>`src/network/protocol/HostWorkerProtocol.ts` | 定义 `continuous` / `sparse` 双模、`ACTIVITY_IDS`（`match` / `freeplay` / `replay` / `sandbox`）、8 位 `REPLICATION_PACKET_FLAGS` 动态包标志位、`RoomActivitySwitchPayload` / `StatePausedAckPayload` / `CsrStateBroadcastPayload`、以及 16 字节二进制 `encodeSparsePingPong16B` / `decodeSparsePingPong16B`（Magic `0x52`）。 | 仅被 Track D 内部文件引用 |
| **传输层（双 DataChannel + 自适应降频）** | `src/network/transport/webrtc/HandshakeSession.ts`<br>`src/network/transport/webrtc/DataChannelTransport.ts`<br>`src/network/transport/webrtc/dataChannelFramingPipeline.ts`<br>`src/network/transport/webrtc/AdaptiveReplicationPacer.ts`<br>`src/network/transport/P2PWebRTCChannel.ts` | 1. 握手时除 `game_net`（无序不可靠）外，新增 `game_ctrl`（`ordered: true` 可靠控制信道）。<br>2. 新增 `sendReliableControl`、`sendRoomActivitySwitch`、`sendStatePausedAck`、`sendCsrStateBroadcast`、`setReplicationMode`、`sendSparsePingPong16B`、`shouldSendAdaptiveSnapshot`。<br>3. `AdaptiveReplicationPacer` 实现 120Hz -> 60Hz -> 30Hz 拥塞降频与 Sparse 2Hz 节拍。<br>4. 收到 16B Ping 自动回 Pong；收到控制帧推入 `transportEvents`。 | **已进入握手流程**：`game_ctrl` 会在建连时创建并绑定；但 `sendRoomActivitySwitch`、`shouldSendAdaptiveSnapshot` 等发送方法尚无外部调用方，且 `transportEvents` 中的新事件尚无 Client 消费 |
| **同步与权威层** | `src/network/sync/SyncAuthorityRouter.ts`<br>`src/network/sync/ReplicationStateInterpolator.ts` | 1. `SyncAuthorityRouter` 管理实体级权威（`host` / `peer` / `replay_readonly` / `takeover_handoff`）及接管倒计时交接。<br>2. `ReplicationStateInterpolator` 复用 `RLRCHermiteSlerpMathPipeline` 做连续插值，并在 `sparse` 模式下锁定末帧。 | 独立类，尚无外部调用方 |
| **服务端编排层** | `src/network/server/RoomActivityOrchestrator.ts`<br>`src/network/server/activities/ActivityKernel.ts`<br>`src/network/server/HostWorkerMessageRouter.ts`<br>`src/network/server/AuthoritativeServerHost.ts`<br>`src/network/server/workerReplayAndDiagnosticsRouter.ts` | 1. 抽象 `RoomActivityKernel` 接口及 4 个内置 Kernel（`Match` / `Freeplay` / `Replay` / `Sandbox`）。<br>2. `AuthoritativeServerHost` 新增 `switchActivity()` 等方法向 Worker 发命令；`HostWorkerMessageRouter` 路由到 `RoomActivityOrchestrator` 并回发消息给 Host 主线程。 | **断在最后一公里**：Worker 回发给 Host 主线程的 `ROOM_ACTIVITY_SWITCH` 等消息，尚未被转发到 `P2PWebRTCChannel` 广播给 Client；`buildWorkerRoomStateSummary` 也尚未填充 `activeActivityId` / `activityEpoch` / `replicationMode` |

---

## 三、 在当前 Linux 环境已验证的结果

在 `origin/z/agy/rc/track-d-replication-bus` (`cf5cfee`) 上实测：

1. **`pnpm run check`（Radar 2.0 + `tsc --noEmit`）**：**通过（Exit 0）**，1359/1359 TS 文件，0 循环依赖，0 越层引用，0 类型错误。
2. **`prettier --check`（Track D 14 个文件）**：**通过（Exit 0）**。
3. **`pnpm run build`**：**通过（Exit 0）**，产物正常生成（29.73s）。
4. **`node scripts/runMultiplayerRoomRepro.mjs --mode=2p --max-duration-ms=65000`**：**可在 Linux 无头 Chrome 环境直接运行**，能拉起 Vite + Host/Client 双标签页完成 WebRTC 建连、`lobby -> match` 流程。
5. **`pnpm run test:baseline`**：11 个场景中 9 个通过，但 `multiplayer_6car_loadout_stress` 失败、`roster_combinations_matrix_exploration` 触发 `core.wasm` `memory access out of bounds`（与 Track D 纯 TS 增量无直接关系，但合入前需知悉）。

---

## 四、 怎么测试？能直接提 PR 给 `alpha` 吗？

### 1. 现有测试脚本的盲区
`scripts/runMultiplayerRoomRepro.mjs` 目前只覆盖 `2p` / `2p2ai` / `3p3ai` 的 **Match + Goal Replay** 流程，**完全没有测试**：
- `--mode=freeplay` 独立练车场联机；
- `game_ctrl` 第二条 DataChannel 是否成功 `open`；
- `ROOM_ACTIVITY_SWITCH` 热切换、`sparse` 2Hz 16B Ping-Pong、`REPLICATION_PACKET_FLAGS` 裁剪或 `AdaptiveReplicationPacer` 降频。

因此：**只跑通现有的 Match 联机测试，只能证明“Track D 没把旧的 Match 搞坏”，不能证明 Track D 的新能力已经跑通。**

### 2. 真机 / 多设备必测项（Real Device Checklist）
在向 `alpha` 提 PR 前，建议至少在真实设备（或双浏览器/跨设备）上验证以下 3 项：
1. **双 DataChannel 跨端建连（最关键）**：`HandshakeSession.ts` 现在会同时创建 `game_net` 和 `game_ctrl`。请在真实网络/跨设备环境下建房与加入，确认两条 DataChannel 都能正常 `open`，没有因为多开一条有序信道导致 ICE/SCTP 协商失败或卡顿。
2. **现有 Match 回归**：2 人联机打完一局（含 Lobby、Kickoff、进球回放、跳回放），确认无新增掉线、黑屏或同步回滚异常。
3. **（若要验证 Track D 新功能本身）**：需先补齐「Host Worker 回复 -> `P2PWebRTCChannel` 广播 -> Client 消费 `transportEvents`」这最后一公里接线，否则真机上无法触发动态模式切换与包裁剪。

### 3. 建议补充的测试 Case（可直接在 Linux CI / 本地跑）
1. **Track D 协议与编排器单元测试 / Smoke 脚本**（不依赖浏览器）：验证 `RoomActivityOrchestrator` 在 `freeplay -> match -> replay -> sandbox` 之间热切换时 `epoch` 递增、`packetFlags` 切换、`SyncAuthorityRouter` 权威转移、`encodeSparsePingPong16B`/`decodeSparsePingPong16B` 编解码与 `AdaptiveReplicationPacer` 降频行为。
2. **扩展 `scripts/runMultiplayerRoomRepro.mjs`**：增加 `game_ctrl` 通道 `isControlChannelOpen` 探针，以及 `--mode=freeplay`（自由练车房，验证无计分/无限气/重置球）与活动热切换探针（待 Host->Client 广播接线完成后即可端到端跑通）。

---

## 五、 下一步建议（PR 策略）

- **方案 A（推荐，分步合入）**：先将当前 Track D 作为 **“Phase 1：ReplicationBus 协议 + 双通道传输 + Orchestrator 骨架”** 提 PR 合入 `alpha`（rebase 最新 `alpha` 后确认 `pnpm run check` 与真机双通道建连正常即可），避免长期分叉冲突。
- **方案 B（端到端闭环后再合入）**：在当前分支补齐「Host->Client `room_activity_switch` 广播转发 + Client 消费 `transportEvents` + 快照循环接入 `shouldSendAdaptiveSnapshot()` + `runMultiplayerRoomRepro.mjs` 的 `freeplay` 与热切换探针」后，再一次性提 PR。
