# RocketCompanion Developer Broadcasting & Replay Watch Together 复现与根因分析报告

- **目标分支**：`z/agy/rc/multiplayer-recheck`
- **新增复现 Commit**：`9c7d27e2e4256fd2957085bccf6ea58456714f07`（Verified Bot Commit）
- **复现套件**：`scripts/runDevBroadcastReproV2.mjs`（`pnpm run repro:dev-broadcast-v2`）
- **测试回放样本**：`rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`

---

## 一、快速复现方法

```bash
# 确保已初始化 public 子模块（包含 camera.wasm 等静态资产）
git submodule update --init --depth 1 public

# 指定 3v3 回放文件并运行真实 Headless Chrome + WebGL + WASM 复现套件
RC_REPLAY_PATH=/path/to/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-broadcast-v2
```

当前基线（`9c7d27e`）运行结果：**`reproduced=4/4`**，全部 4 类问题均已在真实浏览器双端（Host + Client）稳定复现，详细 JSON 快照输出至 `/tmp/dev-broadcast-repro-v2.json`。

---

## 二、已复现问题清单、根因定位与分步修复指南

### 问题 1：Developer Broadcasting Room 默认 Freeplay 进球后无法触发 Cinematic Goal Replay（球黑掉卡死）

- **复现现象**：
  - Host 创建 Developer Sync Room 后默认处于 `freeplay` 模式，且 `runtime.authoritativeServer` 存在（`hasAuthServer: true`）。
  - 将球高速射入球门（`posY` 越过 `5120` 球门线到达 `5601` -> `5884`）后，`FreeplayMode.isGoalReplayActive()` 始终为 `false`，`goalAndTouch.goalResetTicks` 始终为 `0`，`goalIncidentTick` 为 `-1`，球停在球门网内（`ballVelY: 0`）并进入 RocketSim 的进球后熄灭/黑化状态，无法自动进入 Cinematic Goal Replay 与重置开球，只能手动通过 Ball Control 拿球。
- **根因定位**：
  - `src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts` 的 `stepAfterPhysicsTick()` 中存在提前返回逻辑：
    ```ts
    if (
      (scenePort.authoritativeServer || scenePort.networkReconciler) &&
      !(this.goalResetTicks > 0 && this.replayPendingType === 'expire')
    ) {
      if (sim.pollGoal) sim.pollGoal();
      return;
    }
    ```
  - 当 Host 开启 Developer Broadcasting Room 时，`scenePort.authoritativeServer` 非空，但此时 Host 并不是在运行 `AuthoritativeServerHost` 的 Match 权威状态机，而是在 **Broadcasting Host（Pure CSR 广播主控）** 模式下本地运行 `FreeplayMode`。上述判断把“Broadcasting Host”错误等价于“远程 Match 客户端”，直接 `pollGoal()` 吞掉进球标志并 `return`，导致 Freeplay 的进球检测、球冻结、Cinematic Goal Replay 与开球重置全部被跳过。
  - 同时，即便 Host 触发了 Freeplay Goal Replay，也需要通过 `server.pushBroadcastSignal(...)`（`GOAL_REPLAY_WILL_START` / `GOAL_REPLAY_START` / `GOAL_REPLAY_END`）广播给 Client，并在 `FreeplayMode.skipGoalReplay()` 中区分 `isBroadcastingHost` 与真正的远程客户端（`FreeplayMode.ts` 中 `if (this.runtime?.networkReconciler && !this.directorPreview.isReplayActive)` 目前也会受影响）。
- **分步修复建议**：
  1. 在 `FreeplayGoalAndTouchSubController.stepAfterPhysicsTick()` 中引入明确的角色判定：仅当处于 **非 Host 广播的在线客户端**（即 `Boolean(scenePort.networkReconciler) && !isDevSyncHostBroadcastingMode(Boolean(scenePort.authoritativeServer))`）时才跳过本地 Freeplay 进球逻辑；Broadcasting Host 必须允许走完整的本地 Freeplay Goal Detection & `beginGoalReplay()` 流程。
  2. 在 `SimulationStage.ts` 中确认 `physics.pollGoal()` 不会在 `FreeplayGoalAndTouchSubController.stepAfterPhysicsTick()` 之前提前消费掉 Freeplay 的 goal latch（注意 `FreeplayMode.stepAfterPhysicsTick` 在 `physics.step` 阶段内执行时序）。
  3. 在 `FreeplayGoalAndTouchSubController` / `directorPreview` 触发 `beginGoalReplay` 和 `completeReplaySession` 时，若 `isDevSyncHostBroadcastingMode(Boolean(scenePort.authoritativeServer))` 为真，同步通过 `authoritativeServer.pushBroadcastSignal` 发送 `GOAL_REPLAY_WILL_START`、`GOAL_REPLAY_START` 和 `GOAL_REPLAY_END`（或 `KICKOFF_START`）给 Pure CSR Client，确保两端同步进入/退出电影级回放视角。

---

### 问题 2：Replay Watch Together 时 Client 缺失人名、左右 1-3 / 4-6 名单条及上方积分牌 UI

- **复现现象**：
  - Host 加载 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` 后：
    - Host DOM：`domScoreboard: true`（`Alpha:StepperScoreboardHUD` 可见，显示比分与时间），`domRosterCards: 10`（左右两侧各队玩家条及人名正常显示）。
    - Client DOM：`matchMode: "replay-viewer"`, `activeRoomActivity: "replay"`，但 **`domScoreboard: false`、`domRosterCards: 0`、`cardNames: []`**，Client 看不到任何积分牌和左右 6 名玩家的名称。
- **根因定位**：
  1. **`TelemetryStage._syncEdgeRosterHud()` 提前 return 导致 `isClientSyncedReplay` 成为死代码**：
     - 在 `src/game/pipeline/stages/TelemetryStage.ts` 中，`isReplayViewerLoaded` 仅检查本地是否加载了 `standaloneReplayViewer?.hasLoadedReplay` 或 `rlrcSession?.hasLoadedReplay`（Client 上均为 `false`）。
     - 在函数开头：
       ```ts
       if (
         !isReplayViewerLoaded &&
         matchMode !== 'exp-multi-bots' &&
         matchMode !== 'match' &&
         matchMode !== 'online-warmup' &&
         matchMode !== 'dev-test-lab'
       ) {
         return;
       }
       ```
       当 Client 的 `matchMode` 为 `'freeplay'`（或尚未切换为 `'replay-viewer'`）时直接在此 `return`，而提交 `96fc3a1` 新增的 `isClientSyncedReplay`（`appState?.onlineSession?.activeRoomActivity === 'replay' && rvStoreState?.viewerState !== 'idle'`）写在了该 `return` 之后近 100 行处，根本无法执行到 `globalHudTransientChannel.setRoster(...)`。
  2. **`InGameHUD` / `GoalReplayHudOverlay` 的 `isReplayViewerLoaded` 依赖 `loadedFileName && totalFrames > 0` 与 `useReplayViewerStore` 同步链路**：
     - 在 `src/ui-core/hud/InGameHUD.tsx` 与 `src/ui-core/hud/GoalReplayHudOverlay.tsx` 中，`showTopScoreboard` 和 `showEdgeRosterStrips` 依赖 `isReplayViewerLoaded`（即 `replayViewerFileName && replayViewerTotalFrames > 0 && replayViewerState !== 'idle'`）。
     - 实际运行中，Host 通过 `StandaloneReplayViewer` 加载 `.rcreplay` 后，`useReplayViewerStore` 在 Client 端未能持续满足 `loadedFileName` + `totalFrames > 0`（或首次 `ROOM_ACTIVITY_SWITCH` / `GlobalWorkspaceBroadcast` 时 `rvStore` 字段未完整同步，导致 `domScoreboard === false`）。
  3. **积分牌比分与倒计时未纳入 `WorkspaceReplaySyncPayload`**：
     - `src/network/protocol/GlobalWorkspaceProtocol.ts` 中的 `WorkspaceReplaySyncPayload` 与 `RLRCReplaySessionController._syncOnlineRoom` / `replayOnlineSyncHelper.ts` 仅同步了 `fileName`、`totalFrames`、`currentFrame`、`playbackState`、`flowPct`、`focusedCarIndex`、`roster`，**没有同步 `blueScore`、`orangeScore`、`remainingSeconds`、`isOvertime`**，也没有在 `devSyncChannelHandlers.ts` 中更新 `hudStore`，导致即便积分牌显示出来，Client 的比分和时钟也不会随回放帧变化。
- **分步修复建议**：
  1. 将 `isClientSyncedReplay` 的判断提前到 `TelemetryStage._syncEdgeRosterHud()` 的入口守卫条件中（允许 `isReplayViewerLoaded || isClientSyncedReplay || matchMode === 'replay-viewer'` 通过守卫），确保 Client 每帧都能调用 `globalHudTransientChannel.setRoster(...)` 写入 6 名玩家的名字与队伍颜色。
  2. 在 `InGameHUD.tsx` 与 `GoalReplayHudOverlay.tsx` 中，当 `onlineSession.activeRoomActivity === 'replay'`（且处于 `isDevSyncRoom`）时，直接允许显示 `showTopScoreboard` 与 `showEdgeRosterStrips`，不再因为本地未加载文件（`totalFrames === 0`）而隐藏整个 Replay HUD。
  3. 扩展 `WorkspaceReplaySyncPayload`（`GlobalWorkspaceProtocol.ts`）以及 `replayOnlineSyncHelper.ts` / `RLRCReplaySessionController.ts` / `devSyncChannelHandlers.ts`，增加 `blueScore`、`orangeScore`、`remainingSeconds`、`isOvertime` 字段并同步到 Client 的 `hudStore`，使上方积分牌与 Host 完全一致。

---

### 问题 3：观战目标切换（1-6 号位）与默认观战目标未同步到 Client

- **复现现象**：
  - 初始进入 Replay Watch Together 时：Host 默认观战 `focusedCarIndex: 3`（或 `0`），Client 默认观战 `playerCarIndex`（与 Host 不一致）。
  - Host 切换观战目标至 3 号车（`focusedCarIndex = 2`）后等待 600ms：
    - Host `camera.focusedCarIndex === 2`。
    - Client `useSpectateWorkspaceStore.hostCameraSync.focusedCarIndex === 2`（`wsIsSpectating: true`, `wsFollowHostCamera: true`, `car2Visible: true`），**但 Client 的 `camera.focusedCarIndex` 仍然停留在 `0`**！
- **根因定位**：
  1. **`channel.onCameraStateSync` 未同步更新 `useSpectateWorkspaceStore.hostCameraSync`**：
     - 在 `src/game/runtime/devSyncChannelHandlers.ts` 中，`channel.onGlobalWorkspaceBroadcast` 会调用 `applyIncomingWorkspaceBroadcast(msg)` 更新 `useSpectateWorkspaceStore`，但 `channel.onCameraStateSync` 仅调用 `applyCameraSyncToPort(camera, camSync)`，**没有更新 `useSpectateWorkspaceStore.getState().hostCameraSync`**。
     - 下一帧 `GameCameraSubsystem.update()` 执行 `resolveFrameCameraMetadata()` -> `getSpectateCameraFollowTargetIndex()` 时，读取的是旧的 `useSpectateWorkspaceStore.hostCameraSync.focusedCarIndex`，然后执行 `r.camera.setFocusedCarIndex(obsTargetIdx)`，立刻把刚刚切换的视角覆盖回旧目标。
  2. **Camera Focus 状态存在多源冲突（`GameCameraSubsystem` vs `CameraManager` vs `ReplayViewer`）**：
     - 在 `repro_v2` 中，即使 `wsFocusedCarIndex` 已经更新为 `2` 且 `car2Visible === true`，Client 的 `camera.focusedCarIndex` 仍为 `0`。原因在于 `GameCameraSubsystem.resolveFrameCameraMetadata()` 仅在 `isReplayPov` 为 false 时读取 `spectateFollowIdx`，且在 `StandaloneReplayViewer` / `ReplayViewerCausalSyncController` / `useReplayPovSlotHotkeys` 中，Host 切换 1-6 号位后如果只改了 `rv` 内部状态或 `playerCarIndex` 而未同步触发 `AuthorityCameraSyncTracker` 或被 `CameraAndArenaStage` / `GameCameraSubsystem` 的 `activeCar` 逻辑覆盖，Client 就不会跟随。
  3. **Client 独立视角与 Host 强制跟随的边界**：
     - 需求明确：只要 Host 发生了一次切换（1-6 号位或自由视角/球视角），所有未开启独立摄像机锁定的 Client 必须立刻跟随切换到同一目标；若 Host 长时间未切换，Client 本地自主切换不应被周期性广播强制拉回（即 `AuthorityCameraSyncTracker` 的边沿触发语义必须严格生效，且仅在 Host 实际改变 `focusedCarIndex` 时递增 `cameraSyncSeq`）。
- **分步修复建议**：
  1. 在 `src/game/runtime/devSyncChannelHandlers.ts` 的 `channel.onCameraStateSync` 中，当 `shouldApplyIncomingCameraSync` 返回 `true` 时，除了调用 `applyCameraSyncToPort(camera, camSync)`，还必须同步更新 `useSpectateWorkspaceStore` 的 `hostCameraSync`（以及 `useReplayViewerStore.activePovCarIndex`）。
  2. 排查 `GameCameraSubsystem.resolveFrameCameraMetadata()` 与 `CameraManager.setFocusedCarIndex()`：确保当 `ws.isSpectating && ws.followHostCamera && !allowClientIndependentCamera` 时，`obsTargetIdx` 优先使用 `ws.hostCameraSync.focusedCarIndex`，并且**仅在 Host 发送了新的 `cameraSyncSeq`（边沿触发）时**才覆写 Client 当前的 `focusedCarIndex`，避免 Client 本地切人后被下一帧 `GameCameraSubsystem` 覆写，也避免 Host 切人后 Client 无法持久保持。
  3. 初始加入房间或进入 Replay 时，Host 主动下发一次带有当前 `focusedCarIndex` 的 `cameraSync`（递增 `cameraSyncSeq`），从而消除“Host 看 1 号位、Client 默认看 4 号位”的初始错位。

---

### 问题 4：Exhibition（`exp-multi-bots`）倒计时 3-2-1 在 Client 端只触发 `3`，缺失 `2` 和 `1` 及对应音效

- **复现现象**：
  - Host 启动 Exhibition 3v3 后，Host `match.state` 完整经历：`kickoff(3) -> kickoff(2) -> kickoff(1) -> playing(0)`。
  - Client `match.state` 仅经历：`lobby(0) -> kickoff(3)`，DOM 倒计时文字只出现过 `"3"`，**`2` 和 `1` 从未触发，对应音效也只播放了第 3 秒**。
- **根因定位（协议语义错配）**：
  - **发送端**（`src/game/modes/experimental-multi-bots/experimentalMultiBotsLifecyclePipeline.ts`）：
    - 在 `3`、`2`、`1` 秒切换时调用：
      ```ts
      replayPort.emitReplaySignal(MATCH_SIGNALS.KICKOFF_WILL_START, state.countdownTicks, sec);
      ```
      这里传入的 `param1` 是**剩余 tick 数**（`360`、`240`、`120`），`param2` 是**当前秒数**（`3`、`2`、`1`）。
  - **接收端**（`src/game/runtime/RuntimeKickoffSignalSubController.ts` 的 `case MATCH_SIGNALS.KICKOFF_WILL_START`）：
    - 将 `sig.param1` 当作**绝对目标 tick（`targetFrame`）**，将 `sig.param2 === 1` 当作 `isOvertimeKickoff`！
    - 执行 `if (targetFrame > 0 && effectiveTick >= targetFrame)` 判断：
      - 当房间已经运行超过 360 ticks（或进入第 2 秒时 `effectiveTick` 已经 `>= 240`），`effectiveTick >= targetFrame` 立刻成立！
      - Client 误以为倒计时已经过期，直接把 `matchPhase` 置为 `'playing'`、`kickoffCountdown` 置为 `0`，并清空 `latchedKickoffSeconds`！因此只要房间 tick 稍大或进入第 2 秒（`param1 = 240`），`2` 和 `1` 就必然被直接吞掉。
- **分步修复建议**：
  1. **统一 `KICKOFF_WILL_START` 信号契约**：在 `experimentalMultiBotsLifecyclePipeline.ts` 中，计算绝对目标 tick：`const targetTick = currentTick + state.countdownTicks`（且仅在进入 kickoff 时以 `targetTick` 作为 `param1`，`isOvertime ? 1 : 0` 作为 `param2`），与 `AuthoritativeServerHost` 保持完全一致的协议语义；或者在 `RuntimeKickoffSignalSubController` 中明确区分“绝对 `targetFrame`”与“显式倒计时秒数信号”，禁止混用 `param1`/`param2`。
  2. 同时在 `ExperimentalMultiBots` 的 `replayPort.emitReplaySignal` 路径下，确保每次秒数跳变（3 -> 2 -> 1）或基于统一的 `targetFrame` 在 Client 本地按 `remainingTicks = targetFrame - effectiveTick` 驱动 `latchedKickoffSeconds.add(3/2/1)` 与 `audio.play('match_countdown_321', 1.0)`。

---

## 三、先前测试为何未覆盖这些问题（“假测试”审计与测试策略建议）

对仓库现有测试脚本审计后，结论如下：**先前的 `runMultiplayerSyncIssuesRepro.ts` 属于典型的“Mock 绕过型假测试”，而 `runDevSyncReproSuite.mjs` 属于“只测 Happy Path 的浅层 E2E”**。

1. **`scripts/runMultiplayerSyncIssuesRepro.ts`（5 个场景全部误报 `NOT_REPRODUCED`）**：
   - **只测编解码与纯函数，绕过了真实运行时状态机**：
     - *Issue 2（Exhibition 倒计时）*：测试里手动调用 `systemControl.pushBroadcastSignal(KICKOFF_WILL_START, 200, 180, 3)` 并只断言 `reconciler.onSignals` 收到信号，**从未运行** `experimentalMultiBotsLifecyclePipeline` 与 `RuntimeKickoffSignalSubController`，因此完全没发现 `param1` 语义冲突导致 2 和 1 被吞掉。
     - *Issue 4（Replay UI 与 Roster）*：测试里直接在脚本中手动调用 `resolveModeRosterEntries(...)`，**从未运行** `TelemetryStage._syncEdgeRosterHud()` 和 React HUD 渲染，因此完全没发现 `TelemetryStage` 在第 169 行提前 `return` 导致 Client 端根本不推送 Roster。
     - *Issue 5（Goal Replay）*：测试里手动调用 `systemControl.pushBroadcastSignal(GOAL_REPLAY_WILL_START, ...)`，**从未运行** `FreeplayGoalAndTouchSubController.stepAfterPhysicsTick()`，因此完全没发现 Broadcasting Host 在 Freeplay 进球时会被 `scenePort.authoritativeServer` 提前拦截。
2. **`scripts/runMultiplayerSuiteTests.ts`（24 个单元测试）**：
   - 主要覆盖传输层（WebRTC/Tunnel/Framer）、协议编解码（GlobalPayloadCodec / GlobalWorkspaceProtocol）、状态机单测（CoachHandover / AuthorityCameraSyncTracker / PauseAck）。
   - 这些单测**有价值**（可防止底层协议与控制器回归），但不能替代端到端集成校验。
3. **`scripts/runDevSyncReproSuite.mjs`（9 个 Headless Chrome E2E）**：
   - 虽然启动了真实 Chrome + WASM，但断言过浅：
     - Freeplay 只测了直线行驶，**未测射门进球**；
     - Exhibition 只测了 `numCars === 6`，**未测倒计时 `3->2->1->GO` 时序与音效**；
     - Replay 只测了帧推进与 Pause ACK，**未测 Client DOM 积分牌/名单条渲染及 1-6 号位切人持久化**；
     - Camera 只测了 `ballCam` 布尔开关，**未测 `focusedCarIndex` 切换与跨帧保持**。

### 测试体系演进建议

- **保留** `runMultiplayerSuiteTests.ts` 作为底层协议与单元不变量回归测试；
- **废弃或重写** `runMultiplayerSyncIssuesRepro.ts` 中脱离真实管线的 Mock 测试；
- **以本次新增的 `scripts/runDevBroadcastReproV2.mjs`（`pnpm run repro:dev-broadcast-v2`）作为核心验收门禁**：修复上述 4 个问题后，重新运行 `pnpm run repro:dev-broadcast-v2`，直到输出 `reproduced=0/4`（全部 `PASS_FIXED`）为止，并将其纳入后续 CI/人工发版前必跑回归。

---

## 四、状态解耦性评估：`Spectating` vs `Controlling` 抽象架构审视

从本次复现的 4 个问题来看，代码库在传输层（`WebRTC` / `Tunnel` / `GlobalPayloadCodec`）耦合度较低，但在 **GameRuntime 业务层与 HUD 层**，`spectating`（观战/广播接收）与 `controlling`（本地或权威控制）尚未彻底解耦，主要表现为以下 4 处隐式耦合：

1. **角色与模式混为一谈**：
   - `FreeplayGoalAndTouchSubController` 用 `scenePort.authoritativeServer || scenePort.networkReconciler` 来判断是否由远程权威控制，导致“开房广播的 Host（Controlling）”被当成“远程客户端（Spectating）”而失去本地进球回放能力。
   - **建议抽象**：将运行期角色显式收敛为两维正交状态：
     - **Control Role（控制权）**：`local_authority`（Host 本地主导 / Coach 移交后的 Client 主导） vs `spectate_mirror`（Pure CSR 镜像跟随）。
     - **Activity Mode（业务场景）**：`freeplay` / `replay` / `exhibition` / `static_training` / `match`。
2. **Client UI 渲染条件耦合了“本地资源加载状态”**：
   - `TelemetryStage` 与 `InGameHUD` 假设只有本地加载了 `.rcreplay` 或本地处于 `match` 模式才渲染积分牌和 1-6 号位名单条，违背了“Client 处于 `spectate_mirror` 时，UI 形态应完全由 Host 广播的 Activity 与 Metadata 驱动”的原则。
3. **控制权动态移交（Coach Handover）与反向 Spectating 的权衡**：
   - 将抽象收敛为 `spectating` 与 `controlling` 的排列组合，完全可以支撑 **Replay Watch Together**、**Static Training Pack Together**、**Coach 演示 -> 移交 Client 控制权 -> 收回控制权** 等任意模式组合；但工程上需注意两个权衡点：
     - **Epoch 栅栏与状态快照对齐**：控制权从 Host 切到 Client（Host 变为反向 `spectating`）瞬间，必须通过 `authorityEpoch` 严格隔离旧帧，并在交接点做一次状态对齐（目前 `CoachAuthorityHandoverController` 已具备基础框架，但 HUD 提示与 Camera Authority 切换需与 `useSpectateWorkspaceStore` 统一单一数据源）。
     - **模式逻辑归属权**：当 Host 将控制权交给 Client 进行 Static Training Pack 试打时，**“何时判定训练重置/射门结束”的逻辑归属权**仍在 Host 的模式控制器还是跟随 `controlling` 方？建议 **模式生命周期（Reset/Score/Phase）始终由 Room Host 统一裁决与广播**，Client 在获得 `controlling` 权时仅接管车辆物理输入与本地预测/权威帧上报，避免双端同时运行模式状态机产生脑裂。
