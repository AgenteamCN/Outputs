# RocketCompanion 多人同步与 Replay Watch Together 补全及验证报告

## 一、提交与分支状态
- **仓库**：`Antigravi-ty/RocketCompanion`
- **分支**：`z/agy/rc/multiplayer-recheck`
- **最新 Verified Bot Commit**：[`96fc3a1`](https://github.com/Antigravi-ty/RocketCompanion/commit/96fc3a1409a72bf018db8803790f59d448e79008)（`signature.isValid: true`, `state: VALID`，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

## 二、逐项审查与补全内容

### 1. Step 1（Coach 接管多端输入广播 & Pure CSR `acknowledgedControls`）
- 确认 `4907669` 已打通：
  - `ClientAuthorityPublisher`（`src/game/pipeline/stages/HostBroadcastPublisher.ts`）在 Client Alpha 接管时采集并附带 `acknowledgedControls`。
  - `HostInputAuthorityTracker`（`src/network/transport/hostCrossTabAndAuthority.ts`）校验并缓存 `acknowledgedControls`。
  - `SimulationStage.ts` 在 `remoteHasAuthority` 下将 `cf.acknowledgedControls` 经 `server.pushBroadcastSnapshot` 转发至 Pure CSR 广播，使 **Coach (Host)** 与 **Client Bravo** 均能实时看到 Client Alpha 的 `KeyboardInputOverlay` 按键态。

### 2. Step 2（Pure CSR Goal Replay & Cinematic Director Camera 复用与解耦）
- 确认 `aba3e1f` 已复用原有 `MATCH_SIGNALS.GOAL_REPLAY_WILL_START` / `GOAL_REPLAY_START` / `GOAL_REPLAY_END` 与 `RuntimeGoalReplaySignalSubController`（`startGoalReplayDirectorSession`）驱动 Cinematic Director 镜头。
- **发现并修复关键交互缺陷**：
  - `src/game/runtime/RuntimeGoalReplaySignalSubController.ts`：此前 `handleGoalReplayStart` 无条件调用 `reconciler.setReplayPlaybackMode?.(true)`，导致 Pure CSR 观战模式（`isDevSyncClientPureCsrSpectateMode(reconciler)`）在下一帧被判定为 `false`，进而中断 `SpectateInterpolationBuffer` 推流与 `acknowledgedControls` 读取。
  - 修复：在 `isDevSyncClientPureCsrSpectateMode(reconciler)` 为真时跳过 `setReplayPlaybackMode(true)` 与 `setReplayRecordingActive(true)`，保持 Pure CSR 帧插值与 Cinematic Director 镜头并行工作（将原有 match 自驱回放与 Pure CSR 观战流解耦）。

### 3. Step 3（Replay Watch Together 状态、Roster、3D 队伍颜色与边沿相机同步）
- **Client 端 `TelemetryStage` 门控修复**（`src/game/pipeline/stages/TelemetryStage.ts`）：
  - 此前 `src.replayViewerState` 被 `isReplayViewerLoaded` 门控，而 Client 端未加载本地 `.replay` 文件（`isReplayViewerLoaded === false`），导致 `hudRosterResolverPipeline` 中的 `hasSyncedReplayRoster` 恒为 `false`。
  - 修复：新增 `isClientSyncedReplay` 判定（`!isReplayViewerLoaded && activeRoomActivity === 'replay' && rvStoreState.viewerState !== 'idle'`），从 `useReplayViewerStore` 透传 `replayViewerState`、`replayViewerActivePovCarIndex` 与 `replayViewerResolvedRoster`。
- **3D 车体材质与 NameTag 队伍同步**（`src/game/runtime/devSyncChannelHandlers.ts`、`src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`）：
  - Client 收到 `replaySync.roster` 时同步调用 `physics?.setCarTeam` 与 `arena?.setCarTeam`，确保 3D 车体颜色与 NameTag 与回放 Roster 完全一致。
- **`RLRCReplaySessionController` 统一接入在线房间同步**（`src/replay/rlrc/runtime/RLRCReplaySessionController.ts`）：
  - 接入 `ReplayOnlineHeartbeatTracker` 与 `syncReplayWorkspaceToOnlineRoom`，在加载、激活/退出、播放/暂停、Seek、切换 POV 槽位及播放步进时同步 `replaySync` 与边沿触发的 `cameraSync`。
- **边沿相机同步与 Client 独立观察视角开关**（`src/ui-core/stores/appStoreTypes.ts`、`src/game/pipeline/stages/AuthorityCameraSyncTracker.ts`、`src/game/runtime/devSyncChannelHandlers.ts`）：
  - 新增 `onlineSession.allowClientIndependentCamera` 开关；开启时 `AuthorityCameraSyncTracker.shouldApplyIncomingCameraSync` 与 `replaySync.focusedCarIndex` 均不覆盖 Client 本地视角，默认关闭时保持边沿触发同步。

### 4. 测试套件补全（`scripts/runMultiplayerSuiteTests.ts`）
- **Section 23**：补充 Pure CSR `GOAL_REPLAY_START` 下 `reconciler.replayPlaybackMode === false` 且 `isDevSyncClientPureCsrSpectateMode(reconciler) === true` 断言。
- **Section 24**：补充 `bindDevSyncClientChannelHandlers` 在 `allowClientIndependentCamera: true` 下不覆盖 Client 视角，以及 `arena.setCarTeam` / `physics.setCarTeam` 同步回放队伍颜色的断言。

## 三、验证结果
- `pnpm run check`（Radar 1425/1425 TS + `tsc --noEmit`）：**PASS (GREEN)**
- `pnpm run repro:sync-issues`：**0/5 issues reproduced（全部修复）**
- `pnpm run test:multiplayer-suite`：**24/24 PASS**
