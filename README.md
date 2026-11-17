# RocketCompanion Dev Sync Room — Phase 1–3 真实完成度核查、漏扫补充与 Phase 4 拆分可执行实施计划

> **基准分支**：`origin/z/agy/rc/multiplayer-recheck`（已包含最新修复提交 `4a3cc3f`）  
> **Linux Headless 工具链状态**：已完成安装与验证（`pnpm@12.21.0`、`public` 子模块、`pnpm install`、`google-chrome --headless=new` + SwiftShader WebGL + CDP）。

---

## 1. Phase 1–3 真实完成情况核查与补充修复

### 1.1 代码与测试实测结果

在 `z/agy/rc/multiplayer-recheck` 上实测完整验证套件：

| 验证命令 | 结果 | 说明 |
| :--- | :--- | :--- |
| `pnpm run check` (`radar` + `tsc --noEmit`) | **PASS** | 全量 TS 类型检查与架构雷达规则通过 |
| `pnpm run test:multiplayer-suite` | **PASS** | 全部单元 / Loopback 测试通过 |
| `pnpm run test:dev-room-sync-e2e` | **PASS** | 9 个阶段 E2E 通过（HTTP 模式） |
| `pnpm run repro:dev-sync-bugs -- --report-only` | **4/6 PASS, 2 REPRODUCED_BUG** | 场景 1–4 通过；场景 5（多客户端）、场景 6（控制权移交）按预期复现 |

### 1.2 深度扫描发现并已补齐的 Phase 1 & Phase 3 隐蔽缺陷（已提交 `4a3cc3f`）

虽然 Phase 1–3 主体提交（`a4ab863`、`24c34f6`、`0caec73`）已合入，但逐行核查发现 **2 个真实边界 Bug**（未在原有 E2E 中暴露）：

1. **Phase 1 缺陷：`HostBroadcastPublisher` 在 `preserveEngineTick: true` 模式下 tick 冻结**
   - **根因**：`StandaloneReplayViewerController` 在播放与暂停拖拽（scrub）回放帧时，调用 `_applyFrameAtCursor(..., { preserveEngineTick: true })`，导致 `physics.getHeaderView().tickCount` 保持冻结（例如始终为 `0`）。
   - **后果**：
     1. 回放播放时 `HostBroadcastPublisher` 始终发送 `serverTick = 0`，客户端 `SpectateInterpolationBuffer` 不断覆盖 `frames[0]`，无法形成插值队列。
     2. 暂停状态下 Host 拖拽回放进度条或切换 Training Pack 题目时，`currentTick !== lastPublishedTick` 为 `false`，Host 拒绝广播新帧。
   - **已修复**（`src/game/pipeline/stages/HostBroadcastPublisher.ts`）：新增轻量快照签名 `computeBroadcastSignature(snap)`，当 `engineTick` 未变化但快照签名变化时，自动推进单调 `nextStreamTick` 并在暂停态下立即触发单帧同步。

2. **Phase 3 缺陷：`Pause ACK` 的 `peerId` 不匹配导致 Host 无法进入 `pausedIdle`**
   - **根因**：客户端 `PredictionReconciler.sendPauseAck` 发送的 `peerId` 为 `this.channel.peerId || 'client'`，而 Host 端 `HostPauseAckController.unackedPeers` 中存储的是 Host 分配的真实 `activePeerId`（如 `peer_xxx`）。`HostSessionController` 收到 ACK 后未强制用 `activePeerId` 覆盖，导致 `unackedPeers.delete('client')` 永远删不掉 `activePeerId`，Host 持续重试 5 秒直至超时。
   - **已修复**（`src/network/transport/HostSessionController.ts`）：在 `ch.onPauseStateAck` 中强制将 ACK 的 `peerId` 规范化为当前通道的 `activePeerId`。

---

## 2. 原始报告未覆盖的盲区扫描（已纳入后续计划）

1. **Cross-Tab `BroadcastChannel` 共享通道串扰**：本地多标签页调试使用共享的 `BroadcastChannel("p2p_room_<roomId>")`。如果多个 Client 连接同一房间，所有广播消息必须携带 `senderId` / `targetPeerId` 路由过滤，否则 Client1 与 Client2 的握手、状态快照、输入包会互相污染。
2. **迟到客户端（Late-Joiner）初始状态同步**：Client2 在 Host 已经进入 Exhibition 3v3 或已暂停 Replay 后才加入房间时，必须在连接建立瞬间收到 Host 当前的 `GlobalWorkspaceBroadcast`、`PAUSE_STATE_SYNC`（若处于暂停态）以及最新完整物理快照。
3. **动态 Training Pack（Dynamic Training Pack）与比赛事件同步**：原报告仅验证了静态 Training Pack 初始站位。需要补充验证切题（Next Shot）、重置（Reset Shot）、Goal 得分/开球重置、Demo 重生、Boost Pad 冷却在 Pure CSR 广播下是否完整镜像。
4. **回放推进与倍速/拖拽端到端测试**：`repro:dev-sync-bugs` 场景 3 仅校验了 `hostCars=6, clientCars=6, hostCursor=0`，未实际验证回放 cursor 推进、暂停 -> 客户端 ACK -> 0Hz -> 拖拽单帧同步 -> 恢复播放。
5. **Telestrator (`0x05`) 多客户端扇出**：需验证 `Host <-> Client1 <-> Client2` 三方之间的画笔同步及 `host_only` / `everyone` 权限隔离。
6. **客户端接管期间的相机解耦**：当 `clientInputEnabled=true` 且由 Student 驾驶时，Student 需要控制自己的本地相机（BallCam/CarCam），不应被 Host 的 `cameraSync` 强行覆盖。

---

## 3. 关于 Phase 4 难度的评估与拆分建议

**结论：原 Phase 4 确实过大，强烈建议拆分为 4 个独立小步骤（4A / 4B / 4C / 4D）。**

- **为什么原 Phase 4 太重？** 它把两个完全正交的系统绑在一起：
  1. **N-Peer 传输与信令多路复用**（解决 Bug C：Multi-Client）
  2. **仿真/输入权威移交与抢占状态机**（解决 Bug D：Input Handover + FPV Master/Slave）
- **拆分后是否还能独立验收？** **完全可以，而且验收更清晰！** 每个子步骤都有明确的单点验收门禁（例如 4A 对应 `repro:dev-sync-bugs` 场景 5；4B 对应场景 6），不会出现“改了一半无法测试”的僵局。

---

## 4. 详细可执行实施计划（Step-by-Step Implementation Plan）

### Phase 4A：多客户端房间扇出与 Cross-Tab/WebRTC 通道隔离（修复 Bug C）

**目标**：让 Host 支持同时连接 2+ 个客户端（`Client1`、`Client2`...），所有客户端同时收到 `GlobalWorkspaceBroadcast`、`BroadcastFrame`、`PAUSE_STATE_SYNC` 和 `Telestrator`。

1. **修复 Host 仅预热单通道导致的连接阻塞**
   - **文件**：`src/network/transport/HostSessionController.ts`
   - **改动**：
     - 不要只依赖单一 `'peer_1'` 预热通道。当收到来自未知 `senderId` 的 `cross_tab_join` 或信令 `join` 时，自动调用 `_createPeerChannel(senderId, peerName, 'spectator', metadata)` 动态创建并注册新的 `P2PWebRTCChannel`。
     - 确保 `maxBroadcastChannels` 默认值足够大（例如 `>= 8`），`maxInputChannels` 仅限制 Match 模式下的玩家槽位，不限制 Dev Sync Room 观战/广播通道。
2. **修复 `BroadcastChannel` 共享通道的 Peer 定向路由**
   - **文件**：`src/network/transport/webrtc/CrossTabSignaling.ts`、`src/network/transport/webrtc/HandshakeSession.ts`、`src/network/transport/webrtc/DataChannelTransport.ts`、`src/network/transport/webrtc/webrtcHandshakePipeline.ts`
   - **改动**：
     - 在 `CrossTabSignaling` / `HandshakeSession` 中为每个控制消息（`cross_tab_join`、`cross_tab_ack`、`room_offer`、`room_answer`）强制附带 `senderId` 和 `targetPeerId`。
     - 接收端 `HandshakeSession` 仅处理 `targetPeerId === localPeerId` 或广播给 Host 的握手包；已连接的 `P2PWebRTCChannel` 忽略来自其他 `senderId` 的控制与状态包。
     - Host 的 `DataChannelTransport` 在 `roomBroadcastChannel` 上发送 `BroadcastFrame` 时，带 `fromPeerId = host` 标识，所有 Client 的 `DataChannelTransport` 均接收并解码。
3. **迟到客户端（Late-Joiner）首帧与工作区补发**
   - **文件**：`src/network/transport/HostSessionController.ts`、`src/game/pipeline/stages/HostBroadcastPublisher.ts`
   - **改动**：
     - 在 `onPeerConnected` 回调中，若 `this.activeGlobalWorkspace` 非空，立即向新连入的 `ch` 单播一次 `sendGlobalWorkspaceBroadcast(this.activeGlobalWorkspace)`。
     - 若 `this.pauseAckController.paused` 为 `true`，将新 `peerId` 加入 `unackedPeers` 并立即单播 `PAUSE_STATE_SYNC` + 最近一帧快照（`HostBroadcastPublisher.forcePublishSnapshotOnce()`）。
4. **Phase 4A 验收门禁**：
   - `pnpm run check` PASS
   - `pnpm run test:multiplayer-suite` PASS（新增 1 Host + 2 Client 广播测试）
   - `pnpm run repro:dev-sync-bugs` 中 **Scenario 5 (`Multi-Client (2+ Clients) Dev Sync Room`) 变为 `PASS`**（`hostConnectedPeers >= 2`，`client2Mode === hostMode`）。

---

### Phase 4B：显式客户端输入移交（修复 Bug D — 基础版）

**目标**：当 Host 开启 `clientInputEnabled = true` 时，指定 Client（默认当前首个连接的 Client 或选定 `targetPeerId`）可以 0ms 延迟本地驾驶，Host 实时观战。

1. **定义权威移交控制消息与状态字段**
   - **文件**：`src/network/protocol/GlobalWorkspaceProtocol.ts`、`src/network/transport/webrtc/dataChannelControlFraming.ts`、`src/network/transport/webrtc/DataChannelTransport.ts`
   - **改动**：
     - 在 `GlobalWorkspaceBroadcastPayload` 中新增：
       - `authorityEpoch: number`（单调递增版本号）
       - `activeInputPeerId: string | null`（`null` 表示 Host 控制，非空表示该 `peerId` 拥有物理控制权）
     - 新增控制帧类型：
       - `control_authority_grant`: `{ type: 'control_authority_grant', authorityEpoch, targetPeerId, snapshot?: number[] }`
       - `control_authority_reclaim`: `{ type: 'control_authority_reclaim', authorityEpoch }`
       - `client_authority_snapshot`: Client 拥有控制权时回传给 Host 的物理快照帧（复用 `saveCompactState()` 二进制格式，携带 `authorityEpoch`）。
2. **Client 端接管本地物理模拟与状态回传**
   - **文件**：`src/game/pipeline/stages/SimulationStage.ts`、`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`、`src/game/runtime/RuntimeInputPoller.ts`
   - **改动**：
     - 新增判定函数 `isDevSyncClientInputAuthority(reconciler)`：当 `isDevSyncRoom && !isHosting && clientInputEnabled === true && activeInputPeerId === localPeerId` 时为 `true`。
     - 当 `isDevSyncClientInputAuthority` 为 `true` 时：
       - Client **关闭** `isPureCsrClient` 与 `SpectateInterpolationBuffer`；
       - Client **本地运行** `interpolator.update(...)` -> `pollInputs()` -> `physics.step(1)`（0ms 输入延迟）；
       - Client 在 `SimulationStage` 末尾以 60Hz 将 `physics.saveCompactState()` 通过 `client_authority_snapshot` 发送给 Host；
       - Client 暂时屏蔽 Host 的 `cameraSync` 覆盖（允许学生自由使用本地球/车视角）。
3. **Host 端临时切换为 Pure CSR 接收学生画面（并转发给其他观战端）**
   - **文件**：`src/game/pipeline/stages/SimulationStage.ts`、`src/game/pipeline/stages/HostBroadcastPublisher.ts`、`src/network/transport/HostSessionController.ts`
   - **改动**：
     - 当 Host 处于 `activeInputPeerId !== null` 时：
       - Host 暂停本地 `physics.step(1)`，改用 `SpectateInterpolationBuffer` 接收来自 `activeInputPeerId` 的 `client_authority_snapshot` 并调用 `applyReplayFrameToEngine()` 更新画面；
       - Host 若连接了其他 Client（如 Client2），将收到的快照继续扇出给其他观战端。
4. **Phase 4B 验收门禁**：
   - `pnpm run check` PASS
   - `pnpm run repro:dev-sync-bugs` **全部 6 个场景 PASS (`reproducedBugs === 0`)**，Scenario 6 中 Client 按下 `KeyW` 后 Host 的 `car0Pos` 产生 > 50 单位位移。

---

### Phase 4C：FPV Master/Slave 教练 300ms 抢占（Override）

**目标**：在教练（Host）与学员（Client）共驾模式下，教练随时按下任意驾驶按键即可立即夺回控制权，教练停手 300ms 后自动交还给学员。

1. **新增 FPV 抢占状态机**
   - **文件**：`src/game/runtime/CoachAuthorityHandoverController.ts`（新建纯逻辑控制器，便于单元测试）
   - **核心逻辑**：
     - 模式枚举：`authorityMode: 'host_only' | 'client_only' | 'fpv_master_slave'`。
     - 当 `authorityMode === 'fpv_master_slave'` 时，每帧检测 Host 本地输入 `hasActiveInput(playerControls)`（`|throttle| > 0.05 || |steer| > 0.05 || jump || boost || handbrake`）：
       - **若 Host 有输入且当前 `activeInputPeerId !== null`**：Host 立即将 `authorityEpoch += 1`，`activeInputPeerId = null`，发送 `control_authority_reclaim`，并从最新收到的 Client 快照无缝恢复本地 `physics.step()`（丢弃旧 `authorityEpoch` 的在途 Client 帧）。
       - **若 Host 持续无输入 > 300ms 且存在目标学员 `targetPeerId`**：Host 将 `authorityEpoch += 1`，`activeInputPeerId = targetPeerId`，发送 `control_authority_grant`（附带当前 `saveCompactState()`），平滑移交控制权。
2. **Phase 4C 验收门禁**：
   - 在 `scripts/runMultiplayerSuiteTests.ts` 新增 `CoachAuthorityHandoverController` 单元测试（覆盖：输入抢占、300ms 空闲移交、旧 `authorityEpoch` 丢包防护）。

---

### Phase 4D：回放推进/拖拽、动态训练包与 Telestrator 全链路加强

**目标**：补齐原报告未覆盖的边缘场景 E2E 验证。

1. **升级 `scripts/runDevSyncReproSuite.mjs`**：
   - 场景 3 增加：**实际播放推进**（验证 `hostCursor > 0` 且 `clientBall` 随动）+ **暂停 -> ACK -> `pausedIdle` -> 拖拽进度条同步 -> 恢复播放**。
   - 新增场景 7：**Dynamic Training Pack 切题与重置同步**（Host 切题/重置，Client 车球位置误差 < 10）。
   - 新增场景 8：**Telestrator (`0x05`) 1 Host + 2 Client 双向绘制与权限门禁**。
2. **Phase 4D 验收门禁**：
   - `pnpm run check` + `pnpm run test:multiplayer-suite` + `pnpm run test:dev-room-sync-e2e` + `pnpm run repro:dev-sync-bugs` 全绿。

---

### Phase 5（保持独立解耦）：正式“比赛观战模式” & 可选“100ms 空闲 Necto 接管”

1. **Match Spectating Mode**：复用 `BroadcastSpectatePipeline`，关闭 `devFloatingWindows`、`clientInputEnabled = false`、`followHostWorkspace = false`（观战者保留本地视角与记分板切换权）。
2. **100ms Idle Necto Takeover**：仅在 `RuntimeInputPoller.ts` 内作为可选本地输入源实现（`idleBotTakeoverMs = 100`，默认关闭），与网络权威状态机完全解耦。
