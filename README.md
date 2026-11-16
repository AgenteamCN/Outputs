# RocketCompanion Multiplayer 推进报告（Alpha Merge + Step 3 Replay Watch Together）

## 1. 分支合并与前序状态检查
- **Alpha 分支同步（零冲突合并）**：
  - 将 `origin/alpha`（`ff3076a`，包含 PR #22 git-hook-auto-reload 与 PR #23 tuner-startup-optimization，共 6 个提交、15 个文件变更）合并至 `z/agy/rc/multiplayer-recheck`。
  - 合并提交：`49f216e9f17c515eb7a3b2e83d22f550481b61d6`（GitHub Bot Verified Signature: `true`）。
- **Step 1 & Step 2 完备性检查**：
  - Step 1（Pure CSR Sync All、双端输入门控、`DevRoomSyncFloatingWindow`）已在 `91ed3b5` 完成，Test 11 通过。
  - Step 2（Telestrator `0x05` 全链路实时同步、双 Store 联动与权限门控）已在 `9af302d` 完成，Test 12 通过。

## 2. Step 3：Replay Watch Together（Pure CSR + 相机同步）实现
- **提交记录**：`2e76b7d5ffaa979eb6870c1211b5a2751caddcd6`（GitHub Bot Verified Signature: `VALID`）。
- **核心改动**：
  1. **相机同步协议扩展**：
     - `src/network/protocol/GlobalWorkspaceProtocol.ts`：新增 `WorkspaceCameraSyncPayload`（`followHostCamera`、`isFreeCam`、`ballCam`、`focusedCarIndex`、`freeCamPosition`、`freeCamLookAt`）及 `sanitizeCameraSync` 校验。
     - `src/ui-core/stores/spectateWorkspaceStore.ts`：新增 `followHostCamera`、`hostCameraSync`、`setFollowHostCamera`、`setPlaybackState`。
     - `src/ui-system/core/runtimeSpectateWorkspaceBridge.ts` & `src/game/runtimeUiPort.ts` & `src/ui-system/core/runtimeEngineUiPort.ts`：新增 `captureCameraSyncFromPort` / `applyCameraSyncToPort` 桥接。
  2. **Host 广播与 Client 相机应用**：
     - `src/game/runtime/OnlineMatchControlBridge.ts`：在 `broadcastGlobalWorkspace` 自动采集 `domain.camera` 的 `cameraSync`。
     - `src/game/runtime/RuntimeSceneBootstrap.ts` & `src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`：Client 收到 `GlobalWorkspaceBroadcast` 时通过 `applyCameraSyncToPort(camera, payload.cameraSync)` 同步 POV 车位、BallCam 与 FreeCam 位姿。
  3. **首帧 `SYS_CTRL_STATE_ONLY` + Snapshot 同包捕获修复**：
     - `src/network/client/reconcilerInboundPacketParser.ts` & `src/network/client/reconcilerSessionPipeline.ts`：新增 `isReplayPlaybackMode()` 动态读取，确保当 `SYS_CTRL_STATE_ONLY` 与首帧 `stateSnapshot` 在同一数据包到达时，该首帧不会被漏出 `drainedReplayPackets`。
  4. **Replay Viewer 在线推流与暂停/拖拽/相机心跳**：
     - 新增 `src/replay/replayOnlineSyncHelper.ts`：提供 `syncReplayFrameToAuthoritativeServer`（统一走 `authoritativeServer.enterStateOnlyMode(snap, tick)`，兼容 Worker 与直连模式）、`syncReplayWorkspaceToOnlineRoom` 及节流 + 相机变化即时穿透的 `ReplayOnlineHeartbeatTracker`。
     - 接入 `src/replay/ReplayViewerCausalSyncController.ts`、`src/replay/StandaloneReplayViewerController.ts` 与 `src/replay/ReplayScoreboardSynchronizer.ts`：覆盖加载、播放、暂停、Seek、Scrub、POV 切换及退出清理（自动恢复物理与切回 `freeplay`）。

## 3. 自动化验证结果
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1415/1415 TS 文件，0 严格类型错误，0 循环依赖/越界引用，所有文件体积在硬限制内）。
- `pnpm run test:multiplayer-suite`：**13/13 PASS**，新增 **Test 13** 验证：
  - `MatchSystemControlSubController.enterStateOnlyMode` -> `PredictionReconciler` 首帧同步进入 `replayPlaybackMode` -> `applyReplayFrameToEngine` 插值误差 `< 1e-3`。
  - `GlobalWorkspaceBroadcast` 暂停/Tick 状态（`playbackPaused`、`pausedAtTick`）、POV 切换与 FreeCam 位姿同步，以及 `ReplayOnlineHeartbeatTracker` 节流与相机变化即时同步。
