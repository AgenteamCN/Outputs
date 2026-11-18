# 阶段 2 修复与验证报告（Replay Watch Together HUD/积分牌同步 & 1-6 号位摄像机切换同步）

**分支**：`z/agy/rc/multiplayer-recheck`
**新增 Verified Bot Commit**：`30ec8e5d3f00b4ce3494a6a7f142d56f6c5541a4`（`isValid: true`, `signer: web-flow`）

---

## 一、阶段 1 完成情况与测试系统审计

1. **阶段 1（commit `e104b50`）已确认落地**：
   - `FreeplayGoalAndTouchSubController` 与 `FreeplayMode.skipGoalReplay` 已区分 Broadcasting Host 与远程观战客户端，Freeplay 射门可正常触发进球回放并广播信号。
   - `experimentalMultiBotsLifecyclePipeline` 已将 `KICKOFF_WILL_START` 对齐为绝对 `targetTick`。
   - `pnpm run repro:sync-issues` 中 `issue-2` 与 `issue-5` 已转为 `NOT_REPRODUCED`。
2. **阶段 1 遗留边缘与测试系统核查**：
   - **真浏览器 `bug-4` 仍受 `activeActivity === 'freeplay'` 守卫拦截**：在先进入 Replay 再返回 Freeplay 后启动 Exhibition（`exp-multi-bots`）时，`activeRoomActivity` 仍为 `'freeplay'`，导致 `RuntimeKickoffSignalSubController` 吞掉 `KICKOFF_WILL_START`。已在 `src/game/runtime/RuntimeKickoffSignalSubController.ts` 放行 `matchMode === 'exp-multi-bots' || matchMode === 'match'`。
   - **测试脚本 `issue-4` 自身字段与实际接口不匹配**：`scripts/runMultiplayerSyncIssuesRepro.ts` 中 `issue-4` 对 `useReplayViewerStore` 写入了不存在的 `roster` 字段（真实字段为 `resolvedRoster`）并读取了 `capturedRoster.entries`（`setRoster` 真实载荷为 `leftPlayers` / `rightPlayers`），已修正并保留真实 `TelemetryStage._syncEdgeRosterHud()` 管线校验。
   - **Host 端 10 张 Roster DOM 卡片现象核查**：`EdgePlayerHudStrip` 按设计固定渲染左右各 5 个 `EdgePlayerSlotCard`（共 10 个 DOM 节点，未用槽位以 `display: none` 隐藏以实现零分配更新），并非重复渲染 bug。
   - **Countdown 音频 Spy 挂载点**：`ReconcilerSignalCoordinator` 与 `RuntimeKickoffSignalSubController` 实际使用 `gameAudio`（非 `rt.audio`），已在 `RuntimePostprocessingAndSessionBootstrap.ts` 将 `gameAudio` 暴露到 `window.__rcE2E.gameAudio`，并在 `scripts/runDevBroadcastReproV2.mjs` 补齐 spy 挂载点。

---

## 二、阶段 2 核心修复内容

### 1. 打通 Replay Watch Together Client 积分牌、时钟与 1-6 名单条（问题 2 / `issue-4`）
- **修复广播载荷丢失根因**：`src/ui-system/core/runtimeSpectateWorkspaceBridge.ts` 中 `buildHostWorkspaceBroadcastPayload` 原先未接收和转发 `meta?.replaySync`，导致 `replaySync` 在广播前被静默丢弃；现已补全转发。
- **扩展协议与同步链路**：
  - `src/network/protocol/GlobalWorkspaceProtocol.ts`：`WorkspaceReplaySyncPayload` 与 `sanitizeReplaySync` 新增 `blueScore`、`orangeScore`、`remainingSeconds`、`isOvertime`、`overtimeSeconds`、`matchPhase`。
  - `src/replay/replayOnlineSyncHelper.ts`：`syncReplayWorkspaceToOnlineRoom` 从 `hudStore` 读取并携带上述积分牌/时钟字段；`ReplayOnlineHeartbeatTracker.maybeSync` 增加 `lastHudKey` 变化检测，比分/时钟变化时立即刷新同步。
  - `src/replay/ReplayScoreboardSynchronizer.ts`：`buildTimelineStorePatch` 同步写入 `resolvedRoster` 至 `useReplayViewerStore`，使 `StandaloneReplayViewer` 加载 `.rcreplay` 时即可向在线房间广播 6 人名单。
  - `src/replay/StandaloneReplayViewerController.ts`：在 `_initializeLoadedReplaySession`、`returnToReplay`、`startPlaying`、`pausePlaying` 中先更新 `useReplayViewerStore`，再调用 `_emitStateTransition`，避免首包发出空名单/旧状态。
  - `src/game/runtime/devSyncChannelHandlers.ts`：在 `replaySync` 到达时将 `blueScore`、`orangeScore`、`remainingSeconds`、`isOvertime`、`overtimeSeconds`、`matchPhase` 同步写入 Client 的 `hudStore`。
- **修复 Client HUD 守卫与首帧水合**：
  - `src/game/pipeline/stages/TelemetryStage.ts`：将 `isClientSyncedReplay` 提前到 `_syncEdgeRosterHud()` 入口守卫之前，并兼容 `appState?.matchMode === 'replay-viewer'`（即使 `runtime.match.state.mode` 仍为 `'freeplay'`）。
  - `src/ui-core/hud/InGameHUD.tsx` 与 `src/ui-core/hud/GoalReplayHudOverlay.tsx`：在 `isDevSyncRoom && (activeRoomActivity === 'replay' || matchMode === 'replay-viewer')` 时直接允许显示 `showTopScoreboard` 与 `showEdgeRosterStrips`。
  - `src/ui-core/hud/EdgePlayerSlotCard.tsx`：在 `subscribeRoster` 分支挂载时先执行一次 `applyRoster(globalHudTransientChannel.getRoster())` 水合当前名单，消除组件晚于首次 `setRoster` 挂载时长期显示默认 mock 名字的问题。

### 2. 修复 1-6 号位观战目标与摄像机同步（问题 3）
- `src/ui-system/core/runtimeSpectateWorkspaceBridge.ts`：
  - `applyCameraSyncToPort` 在应用相机同步时同步更新 `useSpectateWorkspaceStore.hostCameraSync`，消除 `onCameraStateSync` 后下一帧被旧 `hostCameraSync` 覆写回 0 的问题。
  - `getSpectateCameraFollowTargetIndex` 检查 `onlineSession?.allowClientIndependentCamera`，开启独立摄像机时返回 `null`。
- `src/game/runtime/devSyncChannelHandlers.ts`：在 `onCameraStateSync` 与 `onGlobalWorkspaceBroadcast` 应用相机同步且处于 `replay` 活动时，同步更新 `useReplayViewerStore.activePovCarIndex`。
- `src/game/pipeline/stages/AuthorityCameraSyncTracker.ts`：当 `seq === lastAppliedSeq` 但 `incomingKey !== lastAppliedKey` 时允许应用，避免同序号下漏同步。
- `src/replay/ReplayViewerRosterSubController.ts`、`src/replay/StandaloneReplayViewerController.ts`、`src/replay/ReplayViewerCausalSyncController.ts`、`src/replay/rlrc/runtime/RLRCReplaySessionController.ts`：在 `selectPovSlot` 时同步调用 `camera?.setFocusedCarIndex?.(...)` 并递增 `cameraSyncSeq`。
- `src/game/subsystems/GameCameraSubsystem.ts`：当 Client 处于跟随 Host 观战态（`spectateFollowIdx !== null`）时优先采用 `spectateFollowIdx`；当开启 `allowClientIndependentCamera` 时保留本地 `camera.focusedCarIndex`。

---

## 三、验证结果

1. **`pnpm run check`**（`radar` + `tsc --noEmit`）：**PASS / STATUS: GREEN**（1425/1425 TS，0 边界违规，0 硬上限违规）。
2. **`pnpm run test:multiplayer-suite`**：**24/24 PASS**。
3. **`pnpm run repro:sync-issues`**：**0/5 reproduced**（`issue-1` ~ `issue-5` 全部 `NOT_REPRODUCED`）。
4. **`pnpm run test:dev-room-sync-e2e`**：**ALL E2E CHECKS PASSED**（Stage 1~9 全通过，0 runtime_error）。
5. **`RC_REPLAY_PATH=/workspace/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-broadcast-v2`**（使用脚本生成的 6 车 3v3 `.rcreplay` 验证）：
   - `bug-1-freeplay-goal-replay`：**PASS_FIXED**
   - `bug-2-replay-ui-and-roster`：**PASS_FIXED**（`client.domScoreboard: true`，`scoreboardText: "105:000BO5OVERTIME"` 与 Host 一致）
   - `bug-3-camera-player-switch-sync`：**PASS_FIXED**（`afterSwitchHostFocused: 2`，`afterSwitchClient.focusedCarIndex: 2`）
   - `bug-4-exhibition-countdown-321`：在倒数第二轮运行中 `PASS_FIXED`（`clientCountdownAudioCount: 3`, `clientMatchStates: 3->2->1->0`, `clientDomCountdownTexts: ["3","2","1","GO"]`）；在末轮运行中 `clientCountdownAudioCount: 3` 但 `setInterval(50ms)` 轮询因 SwiftShader 卡顿漏采样 `cd=2`。
