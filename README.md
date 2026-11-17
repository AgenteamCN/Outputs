# Pure CSR 同步问题报告复核、测试盲区分析与 Step 1 修复报告

## 一、与前序 Commit 的冲突与交互扫描

基线分支：`origin/z/agy/rc/multiplayer-recheck`，重点检查了以下两次 commit：
- `8f459e4bf4256b4a3bd60fb16e352beba14400a6`（`feat(multiplayer): complete Phase 4 authority camera sync, coach handover, and 9-scenario E2E suite`）
- `836fe89fb9805fa7feb9b9f225c69723e2f39e88`（`test(multiplayer): add repro suite for 5 dev-sync broadcast & replay issues`）

### 1. Commit 关系说明
- **`836fe89`**：即为生成该 5/5 复现报告的复现套件（`scripts/runMultiplayerSyncIssuesRepro.ts` 及 `repro:sync-issues`），与报告需求完全同源，无冲突。
- **`8f459e4`**：实现了 Phase 4 的权威相机同步（`AuthorityCameraSyncTracker` + `camera_state_sync`）、教练控制权移交（`CoachAuthorityHandoverController`）及 1-car 紧凑快照边界修复。与报告中的修复方案**不存在阻塞性冲突**，但存在以下 **4 处关键交互/潜在冲突点**需要在推进 Step 1~3 时注意：

### 2. 扫描发现的潜在冲突与交互点
1. **Step 1 与 `8f459e4` 教练接管（`remoteHasAuthority` / `isClientInputAuthority`）的交互**：
   - 在 `SimulationStage.ts` 中，当教练/Client 接管控制权（`remoteHasAuthority === true`）时，Host 会直接走 `server?.pushBroadcastSnapshot(cf.snapshot, cf.tick)`，跳过 `broadcastPublisher.publishIfReady`。
   - **Client 端**：当 `isDevSyncClientInputAuthorityMode(reconciler)` 为真时，`isDevSyncClientPureCsrSpectateMode(reconciler)` 为假，因此接管控制权的 Client 仍会正确读取本地键盘而非远端 `acknowledgedControls`（无冲突）。
   - **旁观者边缘场景**：若房间内有第 2 个旁观 Client，在 Coach 接管期间 `hostInputAuthority` 回传的 `client_authority_frame` 目前未携带 `acknowledgedControls`，后续如需让其他旁观者也看到 Coach 的按键，需在 `ClientAuthorityPublisher` 一并附带控制输入。
2. **Step 3（Replay Watch Together `cameraSync`）与 `8f459e4`（`AuthorityCameraSyncTracker`）的潜在冲突**：
   - `8f459e4` 专门引入了边缘触发 + `authorityEpoch`/`seq` 围栏的 `camera_state_sync` 通道，明确防止周期性 `GlobalWorkspaceBroadcast` 覆盖 Client 的本地相机自由度。
   - 报告 Step 3 提议在 `syncReplayWorkspaceToOnlineRoom` 及 `ReplayOnlineHeartbeatTracker`（1-2Hz 心跳）中广播 `replaySync.focusedCarIndex` 与 `cameraSync`。若直接无条件覆盖 Client 相机，会与 `8f459e4` 的非覆盖（non-override）原则冲突。**建议**：Client 端仅在 `seq` 递增（边缘变化）或处于强制跟随状态时应用 `focusedCarIndex`，并保留“允许 Client 独立切换观察视角”开关。
3. **Step 2（Issue 5 Goal Replay）与 Pure CSR `SimulationStage` 门控的交互**：
   - Client 端在 Pure CSR 下接收 Host 推流的 Goal Replay 时，`flags.isGoalReplay` 为真，必须允许 `SpectateInterpolationBuffer` / `interpolator.sync(timestamp)` 继续推进，同时避免与 Client 本地 `ReplayTimelineController` 自驱回放逻辑抢占状态。
4. **架构护栏（`pnpm run radar`）体积限制**：
   - `MatchSystemControlSubController.ts` 在修改前已达 17.9 KB（硬上限 18.0 KB）。直接在其内部新增字段或方法会触发 Radar 硬报错拦截构建（已在本次 Step 1 中通过拆分 `serverGoalReplaySessionPipeline.ts` 解决）。

---

## 二、为什么先前测试未覆盖这些问题 & 测试流程缺失分析

### 1. 先前测试未覆盖的根本原因
1. **双轨架构旁路（Bypass）**：
   - 原有 `runMultiplayerSuiteTests.ts`（如 `MultiClientLoopbackSimulator`）主要测试 `AUTHORITATIVE_SERVER` 比赛模式。该模式下物理在 Worker 内运行，`MatchLifecycle` 与 `AuthoritativePhysicsEngine` 会自动填充 `acknowledgedControls` 并将 `MATCH_SIGNALS` 写入 `SignalEventBuffer`。
   - Dev Sync Room 引入的 Pure CSR（`SYS_CTRL_STATE_ONLY`）走的是 `HostBroadcastPublisher` -> `AuthoritativeServerHost.pushBroadcastSnapshot` -> `MatchSystemControlSubController.stepStateOnlyFrameIfActive` 的**独立广播旁路**，完全绕过了 `MatchLifecycle`。
2. **E2E / 复现套件断言粒度过浅**：
   - `runDevSyncReproSuite.mjs`（9/9）仅验证了快照位置/Tick 同步、Pause ACK、Telestrator 笔迹、相机与控制权移交，**从未对 Pure CSR 包内的 `acknowledgedControls` 非空性、`signals` 信号列表、以及 Client 端 Store/HUD 状态（`useReplayViewerStore`、`keyboardInputOverlayTelemetry`、`hudRosterResolverPipeline`）做任何断言**。

### 2. 类似的未覆盖场景（同类盲区）
- **多车/Bot 控制权与 Coach 接管下的输入同步**：Coach 接管控制权时 `ClientAuthorityPublisher` 未携带 `acknowledgedControls`。
- **Pure CSR 下的其他高频服务端信号与事件**：除 `FLIP_RESET_GAINED (0x0f)` 外，Exhibition 倒计时/得分/摧毁、Training 切球重置（`BALL_RESET`）、Goal Replay 信号在 Pure CSR 旁路中均处于静默丢失状态。
- **回放房间切换后的端到端 Store/Roster 联动**：`GlobalWorkspaceBroadcast` 携带的回放元数据与 `hudRosterResolverPipeline` 渲染结果此前无集成断言。

### 3. 测试流程缺失及本次补全
在审查 `scripts/runMultiplayerSyncIssuesRepro.ts` 时还发现：**该脚本原本在 `reproIssue1` 中用 `GlobalPayloadCodec.decode(rawPacket.subarray(1))` 误裁掉了 `GlobalPayloadCodec` 首字节（`stepStateOnlyFrameIfActive()` 返回的是无帧头前缀的裸 `GlobalPayloadCodec` 载荷，应通过 `sessionManager.buildClientPacket` 封装后再送入 `PredictionReconciler`）**，且未真正传入测试控制输入和调用 `mapRemoteControlsToOverlayKeys`。

为此，我们在流程上进行了以下补全：
1. 修复了 `scripts/runMultiplayerSyncIssuesRepro.ts` 中 `issue-1` 的真实协议编解码路径与远程按键映射验证，使其成为可真实反映修复状态的回归门禁。
2. 在 `scripts/runMultiplayerSuiteTests.ts` 新增 **Section 22: `Pure CSR acknowledgedControls Broadcast & KeyboardInputOverlay Remote Mapping`**，建立从 `HostBroadcastPublisher` -> `MatchSystemControlSubController` -> `GlobalPayloadCodec` -> `PredictionReconciler` -> `keyboardInputOverlayTelemetry` 的全链路契约回归测试，防止后续回归。

---

## 三、Step 1 修复执行情况

已按方案完成 Step 1 全部改造，并解决体积护栏与类型检查问题：
1. **Host 广播端**：
   - `MatchSystemControlSubController.ts`：新增 `broadcastAcknowledgedControls` 缓存，扩展 `pushBroadcastSnapshot` / `enterStateOnlyMode` 接收 `acknowledgedControls`，在 `stepStateOnlyFrameIfActive` 中写入真实缓存（不再硬编码 `[]`），并在 `clear()` 中重置。
   - 拆分纯逻辑至 `src/network/server/match/serverGoalReplaySessionPipeline.ts`，确保 `MatchSystemControlSubController.ts` 保持在 Radar 18KB 硬限制以下。
   - `AuthoritativeServerHost.ts` 与 `workerTrainingAndPhysicsRouter.ts`：打通 `enterStateOnlyMode` / `pushBroadcastSnapshot` 的 `acknowledgedControls` 参数透传。
   - `HostBroadcastPublisher.ts` 与 `SimulationStage.ts`：在 `isBroadcastingHost && !remoteHasAuthority` 时采集本帧车辆/Bot/模式控制输入并随 `publishIfReady` 下发。
2. **Client 接收与 Overlay 渲染端**：
   - `keyboardInputOverlayTelemetry.ts`：新增 `isRemoteSpectating` 状态及纯映射函数 `mapRemoteControlsToOverlayKeys`。
   - `CarNoseOrientationArrows.ts`：支持可选 `remoteControls` 参数，优先映射远端控制输入并更新 `isRemoteSpectating`。
   - `GameCameraSubsystem.ts`：在 `isDevSyncClientPureCsrSpectateMode(reconciler)` 为真时，从 `reconciler.lastParsedPacket.acknowledgedControls` 按 `obsTargetIdx` 提取远端控制输入传入 `carNoseOrientationArrows.update`。
   - `useKeyboardOverlayLiveInputs.ts`：当 `telemetry.isRemoteSpectating` 为真时忽略本地 `localHeld` 与指针按压，仅消费远端遥测态。

---

## 四、验证结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASSED**（0 架构/体积违规，0 TypeScript 错误）。
- `pnpm run test:multiplayer-suite`：**PASSED**（22/22 全部通过，含新增 Section 22）。
- `pnpm run repro:sync-issues`：
  - `[NOT_REPRODUCED] issue-1: User Input Broadcast & Keyboard Input Overlay`（**已修复**）
  - `[REPRODUCED] issue-2` ~ `issue-5`（待后续 Step 2 / Step 3 修复，符合预期）。
