# 验证与状态报告：Pure CSR 广播、Coach 接管输入同步与 Replay Watch Together

## 1. 仓库与分支状态

- 仓库：`https://github.com/Antigravi-ty/RocketCompanion`
- 分支：`z/agy/rc/multiplayer-recheck`（已与 `origin/z/agy/rc/multiplayer-recheck` 同步，工作区干净）
- 最新提交（均为 GitHub 自动 GPG 签名的 Verified Bot Commit，`Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）：
  - `4907669` — `fix(multiplayer): complete Step 1 Pure CSR and Coach handover input broadcast`
  - `aba3e1f` — `fix(multiplayer): complete Step 2 Pure CSR signal sync and Step 3 replay watch together`

## 2. 关键需求核查

1. **Step 1 + Coach 接管多客户端输入广播**：
   - `SimulationStage.ts` 在 `isBroadcastingHost && !remoteHasAuthority` 时采集本帧车辆/Bot/模式控制输入并通过 `HostBroadcastPublisher.publishIfReady` 下发；在 `isClientInputAuthority` 时通过 `ClientAuthorityPublisher` 附带 `acknowledgedControls` 发往 Host，Host 再通过 `server?.pushBroadcastSnapshot(cf.snapshot, cf.tick, cf.acknowledgedControls)` 广播给所有客户端。
   - `hostCrossTabAndAuthority.ts` 校验并缓存 `latestAcknowledgedControls`。
   - `GameCameraSubsystem.ts` 在 Pure CSR 观战客户端（如 Client Bravo）与 Host/Coach 观战远端权威客户端（如 Client Alpha）时，分别读取 `reconciler.lastParsedPacket.acknowledgedControls` 与 `hostAuthority.latestAcknowledgedControls`，传入 `CarNoseOrientationArrows.update` 更新 `keyboardInputOverlayTelemetry`。
   - `useKeyboardOverlayLiveInputs.ts` 在 `isRemoteSpectating` 为真时忽略本地按键，仅消费远端遥测态。

2. **Step 2（Pure CSR 信号转发、Training 重置、Goal Replay & Cinematic Camera）**：
   - `MatchSystemControlSubController`、`AuthoritativeServerHost` 与 `workerTrainingAndPhysicsRouter` 已打通 `pushBroadcastSignal`，支持在 State-Only 模式下将主线程信号写入 `SignalEventBuffer`。
   - `SimulationStage.ts` 在检测到 `FLIP_RESET_GAINED` (0x0f) 时通过 `pushBroadcastSignal` 广播；`StaticTrainingPackMode.ts` 同步 `trainingPackSummary` 并发送 `BALL_RESET`；`simulationNetworkReconcilePipeline.ts` 在收到 `BALL_RESET` / `ROUND_STARTED` 时重置 `SpectateInterpolationBuffer`。
   - `GoalReplaySessionCoordinator.ts` 与 `ExperimentalMultiBotsReplayController.ts` 在进入/结束进球回放时广播 `GOAL_REPLAY_START` / `GOAL_REPLAY_END`，使 Pure CSR 客户端复用 `RuntimeGoalReplaySignalSubController` 触发 Cinematic Director 镜头并保持 `SpectateInterpolationBuffer` 推进。

3. **Step 3（Replay Watch Together 与边沿触发相机非覆盖）**：
   - `GlobalWorkspaceProtocol.ts` 已扩展 `WorkspaceReplaySyncPayload`（`fileName`、`totalFrames`、`currentFrame`、`playbackState`、`flowPct`、`focusedCarIndex`、`roster`）。
   - `replayOnlineSyncHelper.ts` 与 `ReplayOnlineHeartbeatTracker` 在切房、状态变更及心跳时广播 `replaySync` 与边沿触发的 `cameraSync`（仅在相机变化或切房时 `bumpLocalSeq()`）。
   - `devSyncChannelHandlers.ts` 通过 `sharedAuthorityCameraSyncTracker.shouldApplyIncomingCameraSync(...)` 保持边沿触发与非覆盖约束，并将 `replaySync` 同步至 `useReplayViewerStore`。
   - `onlineSessionEnvironmentPipeline.ts` 与 `hudRosterResolverPipeline.ts` 在 `replay` 活动切换时同步 `useReplayViewerStore` 并优先使用 `replaySync.roster` 解析队伍与名称。

## 3. 实测验证结果

所有测试与静态检查均在 `/workspace/RocketCompanion` 实跑通过：

- `pnpm run repro:sync-issues`：**0/5 reproduced**（issue-1 ~ issue-5 全部 `[NOT_REPRODUCED]`）
- `pnpm run test:multiplayer-suite`：**PASS**（Sections 1–24 全部通过，包含 Section 22、23、24）
- `pnpm run check`（`pnpm run radar && pnpm run typecheck`）：**PASS**（`STATUS: GREEN`，`1425/1425 files`，`tsc --noEmit` 零错误）
- `pnpm run build`：**PASS**（`vite build` 成功）
