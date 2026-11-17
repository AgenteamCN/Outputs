# RocketCompanion Dev-Sync 多人状态同步问题复现与修复方案

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **基准分支**: `origin/z/agy/rc/multiplayer-recheck` (基于 `8f459e4`)
- **复现测试提交**: `836fe89` (`test(multiplayer): add repro suite for 5 dev-sync broadcast & replay issues`, Verified Bot Commit)
- **新增复现脚本**: `scripts/runMultiplayerSyncIssuesRepro.ts`（可通过 `pnpm run repro:sync-issues` 运行）

---

## 一、复现结果总览

运行 `pnpm run repro:sync-issues` 已在真实协议编解码与状态机路径下 **5/5 稳定复现**全部问题：

1. `[REPRODUCED] issue-1: User Input Broadcast & Keyboard Input Overlay`
   - `HostBroadcastPublisher.publishIfReady` 与 `MatchSystemControlSubController.stepStateOnlyFrameIfActive` 未采集并下发 `acknowledgedControls`（硬编码 `[]`）。
   - Client 端 `CarNoseOrientationArrows.update` 与 `useKeyboardOverlayLiveInputs` 仅监听本地键盘/鼠标事件，未读取被观战车辆（`obsTargetIdx`）的远端 `acknowledgedControls`。
2. `[REPRODUCED] issue-2: Exhibition Countdown, Incidents & Training Sync`
   - `MatchSystemControlSubController` 与 `AuthoritativeServerHost` 缺少将 Host 主线程信号转发至 `SignalEventBuffer` 的 `pushBroadcastSignal` 接口。
   - Exhibition（`experimentalMultiBotsLifecyclePipeline` / `ExperimentalMultiBotsGoalIncidentTracker`）仅调用本地 `replayPort.emitReplaySignal`，Training（`StaticTrainingPackMode`）未联动 `OnlineMatchControlBridge` / `BALL_RESET`，导致 Client 收不到 `3-2-1-GO` 倒计时、incident 与训练球重置事件。
3. `[REPRODUCED] issue-3: Flip Reset Visual & Audio Sync`
   - Host 端 `CarActionEventSynthesizer` 在本地触发了 `FLIP_RESET_GAINED` (`0x0f`) 视觉与音频，但未在 Pure CSR 广播中将其作为服务端信号转发给 `AuthoritativeServerHost`；Client 端 `RuntimeParticipantAndCombatSignalSubController.handleCombatIncidentSignal` 已具备处理 `0x0f` 并播放 `FlipResetVisual` 与音频的能力，但收不到信号。
4. `[REPRODUCED] issue-4: Replay Watch Together UI, Target Player, Names & Colors`
   - Client 收到 `room_activity_switch('replay')` 后，`createOnlineSessionBridgePort().applyClientRoomActivitySwitch` 将 `modeManager` 切回 `'freeplay'`，且未同步 `useReplayViewerStore` 状态，导致 Replay UI / 状态栏完全未进入。
   - `syncReplayWorkspaceToOnlineRoom` 仅发送 `playbackPaused` / `pausedAtTick` / `cameraSync`，未携带回放元数据（文件名、总帧数、当前帧、倍速）及回放 Roster（每辆车的 `carIndex`、真实 `name`、`team` 颜色）。
   - Client `hudRosterResolverPipeline` 在 `isReplayViewerLoaded === false` 时退化为 `freeplay` 默认规则（`Player` / `Necto` 与 `carIndex % 2` 猜测队伍），直接导致**名称不统一、队伍颜色反转**，且未能锁定同步 Host 当前观察的目标车辆。
5. `[REPRODUCED] issue-5: Goal Replay & Cinematic Camera Trigger`
   - Exhibition / Freeplay 下的 `ExperimentalMultiBotsReplayController` / `GoalReplaySessionCoordinator` 仅在本地发射 `GOAL_REPLAY_WILL_START`，未通过网络广播给 Client；Client 端 `RuntimeGoalReplaySignalSubController` 收不到 `GOAL_REPLAY_WILL_START` (`0x87`) / `GOAL_REPLAY_START` (`0x88`)，因此无法进入 `phase = 'replay'` 与 `acquireGoalReplayCameraOverride`（Cinematic / Director 镜头）。

---

## 二、分步修复方案（建议按顺序推进）

### Step 1: 打通 Pure CSR 广播的 `acknowledgedControls` 与 `KeyboardInputOverlay` 远端映射（解决 Issue 1）
1. **Host 广播端**:
   - 扩展 `HostBroadcastPublisher.publishIfReady` 与 `AuthoritativeServerHost.pushBroadcastSnapshot` / `MatchSystemControlSubController.pushBroadcastSnapshot` 参数，接收 `acknowledgedControls?: Array<{ carIndex: number; controls: CarControlsInput }>` 并缓存在 `MatchSystemControlSubController` 中。
   - 在 `MatchSystemControlSubController.stepStateOnlyFrameIfActive` 中不再硬编码 `acknowledgedControls: []`，而是写入缓存的 `acknowledgedControls`（由 `GlobalPayloadCodec` 使用 `InputPacketCodec.packTickControls` 编码，每车仅占 4 字节）。
   - 在 `SimulationStage` 中，当 `isBroadcastingHost` 为真时采集本帧车辆控制输入（本地玩家 `runtime.playerControls`、Bot 控制、或回放当前槽位 `acknowledgedControls`）传入 `broadcastPublisher.publishIfReady`。
2. **Client 接收与 Overlay 渲染端**:
   - 在 `GameCameraSubsystem.syncArenaCameraMetadata` / `update` 中调用 `carNoseOrientationArrows.update` 时，判断是否处于远端观战状态（`isDevSyncClientPureCsrSpectateMode` 或观看他人/回放）。
   - 若为观战远端车辆 `obsTargetIdx`，从 `reconciler.lastParsedPacket?.acknowledgedControls`（或回放当前帧 `acknowledgedControls`）查出对应 `carIndex === obsTargetIdx` 的 `controls`，按键盘语义映射为虚拟按键态：
     - `wPressed`: `controls.throttle > 0.1 || controls.pitch < -0.1`
     - `sPressed`: `controls.throttle < -0.1 || controls.pitch > 0.1`
     - `aPressed`: `controls.steer < -0.1 || controls.yaw < -0.1`
     - `dPressed`: `controls.steer > 0.1 || controls.yaw > 0.1`
     - `qPressed`: `controls.roll < -0.1`
     - `ePressed`: `controls.roll > 0.1`
     - `slidePressed`: `Boolean(controls.handbrake)`
     - `boostPressed`: `Boolean(controls.boost)`
     - `jumpPressed`: `Boolean(controls.jump)`
   - 在 `useKeyboardOverlayLiveInputs` 中，当处于观战模式时忽略本地 `window` 的 `keydown`/`mousedown`（`localHeld`），仅消费由 `CarNoseOrientationArrows` 更新的 `telemetry.*Pressed`。

### Step 2: 为 `AuthoritativeServerHost` 增加主线程信号转发通道（同时解决 Issue 2、3、5 的传输瓶颈）
1. **Worker / Server 通道**:
   - 在 `MatchSystemControlSubController` 新增 `pushBroadcastSignal(signalType: number, frameId: number, param1 = 0, param2 = 0, replaceTypes?: number[])`，内部在 `isStateOnlyMode` 下将其写入 `MatchCommandQueueSubController` / `SignalEventBuffer`（带适当冗余帧数，如 30~120 ticks）。
   - 在 `AuthoritativeServerHost` 与 `workerTrainingAndPhysicsRouter` 暴露对应的 `pushBroadcastSignal` 消息路由。
2. **Issue 3 (Flip Reset)**:
   - 在 `SimulationStage` 中，当 `isBroadcastingHost` 为真且 `dispatchCarActionTickEffects` 检测到 `FLIP_RESET_GAINED`（`tickRes.flipResetVisualCarIndices` 或 `tickRes.events` 中的 `0x0f`）时，调用 `server.pushBroadcastSignal(MATCH_SIGNALS.FLIP_RESET_GAINED, currentTick, carIndex, resetSubType)`。
   - Client 端 `RuntimeParticipantAndCombatSignalSubController.handleCombatIncidentSignal` 已完整支持 `0x0f`，收到信号后会自动触发 `getFlipResetVisual(targetCarIdx).play()` 与 `audio.handleEvent`。
3. **Issue 2 (Exhibition 倒计时/Incidents 与 Training)**:
   - 在 `ExperimentalMultiBotsMode`（`experimentalMultiBotsLifecyclePipeline`、`ExperimentalMultiBotsGoalIncidentTracker`）发射 `KICKOFF_WILL_START`、`KICKOFF_START`、`SCORE_UPDATED`、`OVERTIME_START`、`MATCH_ENDED`、`PODIUM_START` 时，若处于 `isDevSyncHostBroadcastingMode`，同步调用 `server.pushBroadcastSignal(...)`。
   - 在 `StaticTrainingPackMode` 切球或重置回合时，通过 `OnlineMatchControlBridge.syncTrainingPackActivity` 更新 `trainingPackSummary`（含 `shotResetSeq`）并发送 `BALL_RESET` 信号；Client 端在收到 `shotResetSeq` 变化或 `BALL_RESET` 时清空 `SpectateInterpolationBuffer` 残余轨迹，避免球/车瞬移插值拖影。
4. **Issue 5 (Goal Replay & Cinematic Camera)**:
   - Host 端在 `ExperimentalMultiBotsReplayController` 与 `FreeplayGoalAndTouchSubController` 进入/结束进球回放时，若处于 `isDevSyncHostBroadcastingMode`，向 `server` 推送 `GOAL_REPLAY_WILL_START`、`GOAL_REPLAY_START`、`GOAL_REPLAY_WILL_END`、`GOAL_REPLAY_END`。
   - Client 端需检查 `SimulationStage`：在 `isPureCsrClient` 模式下，即使 `flags.isGoalReplay` 为真，若正在由 Host Pure CSR 推流（而非 Client 本地 `ReplayTimelineController` 自驱回放），仍需允许 `interpolator.sync(timestamp)` / `SpectateInterpolationBuffer` 推进，同时保持 `RuntimeGoalReplaySignalSubController` 触发的 `startGoalReplayDirectorSession`（`acquireGoalReplayCameraOverride`）接管 Cinematic Director 镜头。

### Step 3: 完善 Replay Watch Together 的 UI 状态、目标视角与 Roster 同步（解决 Issue 4）
1. **扩展 `GlobalWorkspaceBroadcastPayload`**:
   - 在 `GlobalWorkspaceProtocol` 中新增可选的 `replaySync` 字段：
     - `fileName`, `totalFrames`, `currentFrame`, `playbackState` (`'playing' | 'paused' | 'ended'`), `flowPct`
     - `focusedCarIndex`（与 `cameraSync.focusedCarIndex` 对齐）
     - `roster: Array<{ carIndex: number; name: string; team: 0 | 1; isBot?: boolean }>`
2. **Host 端同步发送**:
   - 让 `StandaloneReplayViewerController` 与 `RLRCReplaySessionController` 统一通过 `syncReplayWorkspaceToOnlineRoom`（及 `ReplayOnlineHeartbeatTracker`）定时/事件驱动广播 `replaySync` 与 `cameraSync`（确保切换观察对象、暂停、Seek、倍速时立即广播）。
   - 确保 Host 切换观察车辆时同时更新 `camera.focusedCarIndex` 与 `replayPovCarIndex`。
3. **Client 端同步应用**:
   - `applyClientRoomActivitySwitch('replay')` 与收到 `GlobalWorkspaceBroadcastPayload.replaySync` 时，同步写入 Client 端只读镜像状态（如 `useReplayViewerStore` / `onlineSession.replaySyncState`），让 `GoalReplayHudOverlay` 与 Replay 状态栏正常显示（先保证双方视角、UI 状态、名称、队伍颜色完全一致，再在此基础上开放“允许 Client 独立切换观察视角”开关）。
   - 在 `hudRosterResolverPipeline` 与 `ArenaWorld` 车体材质/NameTag 同步中，当 `activeRoomActivity === 'replay'` 且存在 `replaySync.roster` 时，优先使用 `replaySync.roster` 解析蓝/橙两队名单与车体颜色，禁止回退到 `freeplay` 的 `carIndex % 2` 猜测逻辑。

---

## 三、验证方式

1. 复现套件（当前应返回 5/5 REPRODUCED）：
   ```bash
   pnpm run repro:sync-issues
   ```
2. 现有静态检查与多人回归套件（当前全部通过）：
   ```bash
   pnpm run check
   pnpm run test:multiplayer-suite
   ```
