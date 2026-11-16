# RocketCompanion `z/agy/rc/multiplayer-recheck` Step 5 推进与全链路验收报告

## 1. 前序状态检查与关键修复

在推进 Step 5（Linux Headless Chrome 真实多标签页 E2E 验收）之前，检查了分支 `origin/z/agy/rc/multiplayer-recheck` 的 Step 1-4 基线（`91ed3b5`、`9af302d`、`2e76b7d`、`938f846`）以及沙盒运行环境：

1. **工具链与静态资源子模块补全**：
   - 安装 `pnpm`（`12.21.0`）并完成 `pnpm install`。
   - 初始化 `public` 子模块（`RocketCompanionStatics.git`），补全 `public/assets/camera/camera.wasm` 以通过 `ERR_POV_MICROKERNEL_TAMPERED` 启动校验。
2. **发现并修复 BroadcastChannel 可靠控制帧静默丢弃问题**（`src/network/transport/webrtc/webrtcHandshakePipeline.ts`）：
   - 在真实双标签页无外部信令服务器的跨标签页（`BroadcastChannel`）通信中，`transmitReliableControlFrame` 会将 `{ type: 'global_workspace_broadcast', ... }`、`{ type: 'room_activity_switch', ... }` 等控制帧对象通过 `roomChannel.postMessage(messageObj)` 发送。
   - 原 `classifyRoomMessage` 仅处理 `cross_tab_join` / `cross_tab_ack` / `room_answer` / `color_change` / `ping` / `pong` / `p2p_packet`，其余对象一律返回 `{ action: 'ignore' }`，导致 `global_workspace_broadcast` 与 `room_activity_switch` 在跨标签页模式下无法送达 Client。
   - 已在 `classifyRoomMessage` 中补充对 `global_workspace_broadcast`、`room_activity_switch`、`replication_mode_set`、`state_paused_ack`、`room_csr_state_broadcast`、`tunnel_endpoint_advertise`、`peer_kick`、`peer_leave`、`voice_signal`、`diagnostic_fleet_toggle` 的识别并转为 `{ action: 'raw_packet', payload: JSON.stringify(msg) }`，打通完整控制面。
3. **只读 E2E 诊断接口**：
   - 在 `src/game/GameRuntime.ts`、`src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts` 与 `src/ui-core/stores/spectateWorkspaceStore.ts` 中挂载只读 `window.__rcE2E` 访问器，用于 Headless Chrome CDP 精确校验物理姿态、球位置及多 Store 状态。

## 2. Step 5：Linux Headless Chrome 真实双标签页 E2E 验收

新增自动化 E2E 脚本 `scripts/runDevRoomSyncE2E.mjs`（并注册 `package.json` 中的 `test:dev-room-sync-e2e`），在无 GUI 沙盒下通过 SwiftShader WebGL + CDP 双标签页验证完整链路（**Host 与 Client 均使用无 `?reproRole=` 后缀的普通 URL `/?skipShaderPrewarm=1`**）：

1. **Stage 1 — Host Dev UI 建房**：Host 打开 `dev-under-development` 页面点击 `Create Developer Sync Room`，生成房间号并打开 `DevRoomSyncFloatingWindow`。
2. **Stage 2 — 普通无后缀 Client UI 加房**：Client 打开无后缀页面，导航至 `join-room` 输入房间号并提交 `Join`，通过 `BroadcastChannel` 完成建连（`connectedPeers >= 1`）。
3. **Stage 3 — Pure CSR Sync All**：验证 Client 自动进入 `spectate_mirror`、`clientInputEnabled === false`、`followHostWorkspace === true`，以及通过 `DevRoomSyncFloatingWindow` 按钮切换 `arenaPreset`（`throwback` <-> `standard`）实时同步。
4. **Stage 4 — Dual-End Input 门控**：验证 `clientInputEnabled === false` 时 `PredictionReconciler.sampleAndPredictInput` 将输入清零；Host 开启 `Allow Client Driving Input` 后保留输入；关闭后恢复清零。
5. **Stage 5 — Telestrator (`0x05`) 实时同步**：Host 开启 `everyone` 权限后，通过真实 CDP `Input.dispatchMouseEvent`（`mousePressed` / `mouseMoved` / `mouseReleased`）在 Host 与 Client 3D 场地画布上双向绘制笔画并验证 `0x05` 同步计数，随后执行 Host 两步 Clear 验证双端同步清空。
6. **Stage 6 — Replay Watch Together**：Host 切换活动至 `Replay`，验证 Client 保持轻量 CSR `freeplay` 壳且 `replayPlaybackMode === true`；Host 调用 `enterStateOnlyMode(snap, 420)` 并广播 `playbackPaused: true, pausedAtTick: 420, cameraSync`，验证 Client 同步到位。
7. **Stage 7 — Custom Training Pack**：Host 切换活动至 `Training` 并点击 `Reset Shot`，验证 Client 无需本地 `.RCDPack` 文件即同步 `activeRoomActivity === 'multiplayer_training'` 与递增的 `shotResetSeq`。
8. **Stage 8 — Room Match 自动拉入**：Host 点击 `Match`，验证已连接 P2P Peer 自动被拉入 Match 且 `clientInputEnabled === true`（无需二次建房/邀请）。

## 3. 自动化验证结果

全部命令均已执行并通过（退出码 0）：

- `pnpm run check`：GREEN（`1415/1415` TS，`tsc --noEmit` 0 错误，0 循环依赖，0 越层引用）。
- `pnpm run radar:audit`：GREEN（9 个审计目标结构耦合与 Ping-Pong 均在阈值内）。
- `pnpm run test:multiplayer-suite`：ALL PASSED（Test 1 ~ Test 14 全通过）。
- `pnpm run test:dev-room-sync-e2e`：ALL E2E CHECKS PASSED（耗时 `95335ms`）：
  - `runtime_error === 0`
  - `poseAnomalies === 0`
  - `stuckInGoal === false`

## 4. 提交记录

- 分支：`z/agy/rc/multiplayer-recheck`
- 已签名 Bot 提交：`745ab8e1ed27856cf6f4acdfa4a88f59abe43df2`（`verified: true`, `web-flow` 签名，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）。
