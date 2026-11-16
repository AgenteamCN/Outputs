# RocketCompanion Multiplayer Step 1–5 Verification & Step 5 E2E Report

- **Branch**: `z/agy/rc/multiplayer-recheck`
- **Commits**:
  - `745ab8e`: `feat(multiplayer): implement Step 5 Headless Chrome multi-tab E2E & fix BroadcastChannel control relay`
  - `0719bf9`: `fix(multiplayer): buffer early GlobalWorkspaceBroadcast & preserve hideFooter on mirror` (`verified: true`, GPG signed as bot)

## 1. 前序基础状态检查（Step 1–4）与补充修复

1. **Step 1–4 基线完备性**：
   - `91ed3b5`（Step 1 Pure CSR Sync All & `DevRoomSyncFloatingWindow`）、`9af302d`（Step 2 Telestrator 0x05 全链路同步）、`2e76b7d`（Step 3 Replay Watch Together + 相机同步）、`938f846`（Step 4 Custom Training Pack & Room Match 自动拉入）均已在 `z/agy/rc/multiplayer-recheck` 就绪。
2. **Headless Chrome 真实多标签页联调中发现并修复的遗漏点**：
   - **CrossTab BroadcastChannel 控制帧路由缺失**（`src/network/transport/webrtc/webrtcHandshakePipeline.ts`）：`classifyCrossTabRoomMessage` 原先未识别 `global_workspace_broadcast`、`room_activity_switch`、`replication_mode_set`、`state_paused_ack`、`room_csr_state_broadcast`、`tunnel_endpoint_advertise`、`peer_kick`、`peer_leave`、`voice_signal`、`diagnostic_fleet_toggle` 等可靠控制帧对象，导致跨标签页通信时直接被忽略。已补全路由至 `raw_packet` 交由 `DataChannelTransport` 分发。
   - **早到 `GlobalWorkspaceBroadcast` 竞态丢包**（`src/network/transport/P2PWebRTCChannel.ts`）：若 Host 在 `onConnected` 立即下发 `activeGlobalWorkspace`，而 Client 尚未挂载 `onGlobalWorkspaceBroadcast` 回调，首帧广播会被静默丢弃。已在 `P2PWebRTCChannel` 增加 `pendingGlobalWorkspaceBroadcast` 缓冲并在挂载回调时立即回放。
   - **Client 镜像浮窗缺少 `hideFooter` 导致运行时 Console Error**（`src/ui-system/core/runtimeSpectateWorkspaceBridge.ts`）：`applyWorkspaceStateLocally` 调用 `floatingStore.spawnWindow` 时未透传 `preset` 元数据与显式 `hideFooter: preset?.hideFooter ?? false`，触发 `[FLOATING WINDOW SCHEMA ERROR]`。已补全透传与显式 `hideFooter` 默认值。

## 2. Step 5：Linux Headless Chrome 真实双标签页 E2E 验收

- **测试脚本**：`scripts/runDevRoomSyncE2E.mjs`（`pnpm run test:dev-room-sync-e2e`）
- **覆盖流程（无 `?reproRole` 快捷后缀）**：
  1. **Stage 1**：Host Dev UI 创建 Developer Sync Room（`1408`）并打开 `DevRoomSyncFloatingWindow`。
  2. **Stage 2**：普通无后缀 Client UI（`/?skipShaderPrewarm=1`）通过 Play -> Join Room 输入房间码连入。
  3. **Stage 3**：Pure CSR Sync All & `arenaPreset` 同步校验（`spectate_mirror`、`followHostWorkspace: true`、`clientInputEnabled: false`）。
  4. **Stage 4**：双端 Input 门控校验（`PredictionReconciler.sampleAndPredictInput` 归零，Host 开关切换 ON/OFF 实时同步）。
  5. **Stage 5**：Telestrator（`0x05`）双向实时同步与权限门控（`host_only` / `everyone`、双 Store 联动、`isCameraSynced` 放行）。
  6. **Stage 6**：Replay Watch Together 同步（`replay` 活动切换、Pure CSR 轻量壳、`playbackPaused` / `pausedAtTick` / `cameraSync` 同步）。
  7. **Stage 7**：Custom Training Pack 同步（`multiplayer_training` 切换、无本地 `.RCDPack` 依赖、`Reset Shot` 同步）。
  8. **Stage 8**：Room Match 自动拉入（`switchRoomActivity('match')` 自动分配已连入 P2P Client 槽位并开放 `clientInputEnabled: true`）。

## 3. 自动化验收结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1415/1415 TS，0 边界违规）
- `pnpm run radar:audit`：**GREEN**
- `pnpm run test:multiplayer-suite`：**ALL PASS**（14/14）
- `pnpm run test:dev-room-sync-e2e`：**ALL E2E CHECKS PASSED**（90152ms）
  - `runtime_error === 0`
  - `poseAnomalies === 0`
  - `stuckInGoal === false`
