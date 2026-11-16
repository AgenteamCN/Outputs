# Step 2 完成报告：Telestrator (`0x05`) 全链路实时同步

- **仓库 / 分支**：`Antigravi-ty/RocketCompanion` (`origin/z/agy/rc/multiplayer-recheck`)
- **Commit**：[`9af302dee13724e35a2f841296b2979f15727e9d`](https://github.com/Antigravi-ty/RocketCompanion/commit/9af302dee13724e35a2f841296b2979f15727e9d) (`verified: true`, Bot Commit + `Co-authored-by: Mark`)

---

## 一、前序基础（Step 1）核查结果

- `91ed3b5` 已完整落地 Step 1（Pure CSR Sync All、前后双端输入门控、`GlobalWorkspaceProtocol` 扩展、`applyClientRoomActivitySwitch` 修复、`DevRoomSyncFloatingWindow` 与 Test 11）。
- 在此基础上直接推进 Step 2，并补齐报告中未扫描到的 `STROKE_UNDO` / `STROKE_CLEAR` 二进制编解码、Host 对 Client `clear(all)` 越权降级、以及无头环境下 `runtimeUiPort` 未绑定 Store 时的可选链防护。

---

## 二、Step 2 核心改动清单

### 1. 二进制协议与传输层接线（`TELESTRATOR_SYNC = 0x05`）
- `src/network/protocol/TelestratorWireCodec.ts`
  - 新增 `getTelestratorWireMsgType(bytes)` 消息类型识别。
  - 新增 `encodeTelestratorUndo` / `decodeTelestratorUndo`（`STROKE_UNDO = 3`）与 `encodeTelestratorClear` / `decodeTelestratorClear`（`STROKE_CLEAR = 4`）。
- `src/network/transport/webrtc/dataChannelFramingPipeline.ts`
  - 在 `DataChannelTransportEvent` 新增 `'telestrator_sync'` 事件。
  - 新增 `transmitTelestratorEnvelope(...)`：使用 `encodeTunnelEnvelope(TUNNEL_PACKET_TYPES.TELESTRATOR_SYNC, bytes, ...)` 封装后，优先走 WebRTC `controlDataChannel`（不可靠草稿可走主 `dataChannel`），回退 `ReliableTunnelFramer` / Rust UDP `TunnelTransport` / `BroadcastChannel`。
- `src/network/transport/webrtc/DataChannelTransport.ts`
  - 新增 `onTelestratorBytes` 回调与 `sendTelestratorBytes(bytes, reliable)` 方法；在 `handleIncomingMessage` 中解包 `TUNNEL_PACKET_TYPES.TELESTRATOR_SYNC` 并分发事件与回调。
- `src/network/transport/P2PWebRTCChannel.ts`
  - 暴露并绑定 `onTelestratorBytes` 与 `sendTelestratorBytes`。

### 2. Host 权限校验、转发与 `STROKE_HASH_ACK`
- `src/network/transport/hostTelestratorRelay.ts`（新增）
  - 实现 `handleHostInboundTelestratorBytes`：
    - `host_only` 模式下拦截未授权 Client 的绘制请求，并回传 `HOST_PERMISSION_SYNC(false)`。
    - 校验通过的 `STROKE_COMMIT` 立即向发送方回传 `STROKE_HASH_ACK`。
    - 拦截 Client 发送的 `STROKE_CLEAR(scope: 'all')` 并降级为 `scope: 'own'`，防止 Client 清空 Host 或其他人的笔画。
    - 屏蔽 Client 伪造的 `HOST_PERMISSION_SYNC` / `STROKE_HASH_ACK`。
    - 将合法报文转发给房间内其余已连接 Peer（`STROKE_DRAFT` 走非可靠，`COMMIT/UNDO/CLEAR` 走可靠）并投递至 Host 本地回调。
- `src/network/transport/HostSessionController.ts` & `src/network/transport/MultiplayerSessionManager.ts`
  - 在 `_createPeerChannel` 接入 `handleHostInboundTelestratorBytes`，新增 `broadcastTelestratorBytes`、`setTelestratorAllowedClientIds` 及 `onTelestratorBytes` 全双工回调接线。

### 3. 双 Store 联动、绘制门控修复与运行期闭环
- `src/ui-core/stores/telestratorStore.ts` & `src/game/subsystems/telestratorOverlayHelpers.ts`
  - 修复 `canLocalDrawInTelestrator` 与 `canTelestratorDrawNow`：`allowClientDrawing === true`（`everyone` 权限）时直接放行，不再被 `!isCameraSynced` 卡死。
  - 在 `telestratorOverlayHelpers.ts` 新增 `applyRemoteTelestratorWireBytes`（含对未绑定 `runtimeUiPort` 的可选链防护）。
- `src/ui-core/stores/spectateWorkspaceStore.ts` & `src/ui-system/core/runtimeSpectateWorkspaceBridge.ts`
  - 联动 `useSpectateWorkspaceStore` 与 `useTelestratorStore`：Host 切换 `telestratorPermission`、Client 收到 `GlobalWorkspaceBroadcast`（`applyHostWorkspaceBroadcast` / `applyIncomingWorkspaceBroadcast`）以及离房清理（`resetSpectateWorkspaceOnLeave`）时同步更新 `allowClientDrawing`、`isCameraSynced` 与 `isHost`。
- `src/entities/telestrator/TelestratorManager.ts`
  - 新增 `undoRemoteStroke(authorId, strokeId?)`，支持按 `strokeId` 精确撤销远端笔画。
- `src/game/subsystems/TelestratorOverlaySubsystem.ts`、`src/game/GameRuntime.ts`、`src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`
  - 接入 `setNetworkTransport` 与 `handleRemoteTelestratorBytes`：`STROKE_DRAFT`（50ms 节流）、`STROKE_COMMIT`、`STROKE_UNDO`、`STROKE_CLEAR` 实时双向收发，并在 `onHostServer` / `onJoinServer` / `onStopServer` 同步 `useTelestratorStore` 角色与 `isCameraSynced`。

---

## 三、自动化验证结果

1. **`pnpm run check`**（`radar.ts` + `tsc --noEmit`）：**GREEN**（1414/1414 TS 文件，0 循环依赖，0 越级导入，所有文件均在 18KB 硬上限内）。
2. **`pnpm run radar:audit`**：**GREEN**。
3. **`pnpm run test:multiplayer-suite`**：**ALL 12 TESTS PASSED**（含新增 **Test 12: Telestrator (0x05) Full-Link Sync, Dual-Store Linkage & Permission Gating**，覆盖 Undo/Clear 编解码、双 Store 联动与门控修复、`host_only` 拦截、`everyone` 下 Draft/Commit/Undo/Clear 全链路同步、`STROKE_HASH_ACK` 与 Client `clear('all')` 越权降级）。
