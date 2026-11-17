# Step 1 完成与测试套件补全报告

## 1. 提交信息
- **分支**: `z/agy/rc/multiplayer-recheck`
- **Commit**: [`490766995ebe25950598f3f4a60dc600454c9731`](https://github.com/Antigravi-ty/RocketCompanion/commit/490766995ebe25950598f3f4a60dc600454c9731) (`isValid: true`, `wasSignedByGitHub: true`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)

## 2. Step 1 实现总览

### 2.1 Host Pure CSR `acknowledgedControls` 采集与广播
- **`src/network/server/match/serverGoalReplaySessionPipeline.ts` & `MatchGoalReplaySubController.ts`**: 拆分纯服务端回放会话初始化与步进逻辑，确保 `MatchSystemControlSubController.ts` 保持在 Radar 18KB 硬限制以下。
- **`src/network/server/match/MatchSystemControlSubController.ts`**: 新增 `broadcastAcknowledgedControls` 缓存；扩展 `pushBroadcastSnapshot` / `enterStateOnlyMode` 接收 `acknowledgedControls`；`stepStateOnlyFrameIfActive` 写入真实缓存（不再硬编码 `[]`）；`clear()` 重置缓存。
- **`src/network/server/match/matchLifecyclePipeline.ts`**: 扩展 `LifecyclePhysicsSnapshotPort.acknowledgedControls` 类型兼容 `CarControlsInput`。
- **`src/network/server/AuthoritativeServerHost.ts` & `src/network/server/workerTrainingAndPhysicsRouter.ts`**: 打通 `enterStateOnlyMode` / `pushBroadcastSnapshot` 的 `acknowledgedControls` 透传。
- **`src/game/pipeline/stages/HostBroadcastPublisher.ts` & `src/game/pipeline/stages/SimulationStage.ts`**: Host 广播时（`isBroadcastingHost && !remoteHasAuthority`）采集本帧车辆/玩家/Bot 控制输入传入 `broadcastPublisher.publishIfReady`。

### 2.2 教练接管（Coach Handover）多端控制输入同步（Client Alpha -> Coach/Host -> Client Bravo）
- **`src/game/pipeline/stages/HostBroadcastPublisher.ts` & `src/game/pipeline/stages/SimulationStage.ts`**: `ClientAuthorityPublisher.publishIfReady` 在 `isClientInputAuthority` 模式下随 `client_authority_frame` 附带 `acknowledgedControls`。
- **`src/network/protocol/GlobalWorkspaceProtocol.ts` & `src/network/transport/hostCrossTabAndAuthority.ts`**: `ClientAuthorityFramePayload` 与 `HostInputAuthorityTracker.ingestFrame` 解析并校验 `acknowledgedControls`，并维护 `latestAcknowledgedControls`。
- **`src/game/pipeline/stages/SimulationStage.ts`**: Host 在 `remoteHasAuthority` 期间消费远端权威帧时，将 `cf.acknowledgedControls` 一并透传至 `server?.pushBroadcastSnapshot(cf.snapshot, cf.tick, cf.acknowledgedControls)`，使 Pure CSR 广播包将控制输入同步至所有旁观 Client（如 Client Bravo），同时 Coach（Host）端也可通过 `latestAcknowledgedControls` 实时显示。

### 2.3 Client 接收与 `KeyboardInputOverlay` 远端映射
- **`src/ui-core/config/keyboardInputOverlayTelemetry.ts`**: 新增 `isRemoteSpectating` 状态与纯映射函数 `mapRemoteControlsToOverlayKeys`。
- **`src/entities/arena/CarNoseOrientationArrows.ts`**: `update()` 支持可选 `remoteSpectate` 参数，处于远端观战时按 `mapRemoteControlsToOverlayKeys` 映射并更新 `isRemoteSpectating`。
- **`src/game/subsystems/GameCameraSubsystem.ts`**: 在 `isDevSyncClientPureCsrSpectateMode(reconciler)` 为真时从 `reconciler.lastParsedPacket.acknowledgedControls` 提取 `obsTargetIdx` 对应控制输入；在 Host 观战远端权威客户端时从 `hostAuthority.latestAcknowledgedControls` 提取控制输入，传入 `carNoseOrientationArrows.update`。
- **`src/ui-core/panels/dev/keyboard-input-overlay/useKeyboardOverlayLiveInputs.ts`**: 当 `telemetry.isRemoteSpectating` 为真时忽略本地键盘/鼠标 `localHeld` 与 `setPointerHeld`，仅消费远端遥测态。

## 3. 测试套件补全
- **`scripts/runMultiplayerSyncIssuesRepro.ts`**: 修复 `issue-1` 的真实协议编解码路径（去除对 `GlobalPayloadCodec` 载荷的误裁切、使用 `sessionManager.buildClientPacket` 封装、注入测试 `acknowledgedControls` 并验证 `mapRemoteControlsToOverlayKeys` 映射）。
- **`scripts/runMultiplayerSuiteTests.ts`**: 新增 **Section 22 (`Pure CSR acknowledgedControls Broadcast, Coach Handover Multi-Client Input & KeyboardInputOverlay Remote Mapping`)**，覆盖 Host->Client Pure CSR 输入广播与 Client Alpha->Coach(Host)->Client Bravo 教练接管多端输入同步全链路回归验证。

## 4. 验证结果
- `pnpm run check`: **PASS** (`1425/1425` TS, `tsc --noEmit` clean, Radar GREEN)
- `pnpm run test:multiplayer-suite`: **PASS** (`22/22` sections passed)
- `pnpm run repro:sync-issues`:
  - `[NOT_REPRODUCED] issue-1: User Input Broadcast & Keyboard Input Overlay`（已修复）
  - `issue-2` ~ `issue-5` 保持 `[REPRODUCED]`（待后续 Step 2 / Step 3 推进）
