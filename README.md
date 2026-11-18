# Step 1 核验与架构确认报告（分支 `z/agy/rc/multiplayer-recheck`）

## 1. 当前仓库状态与 Step 1 验证结果

分支 `origin/z/agy/rc/multiplayer-recheck` 当前最新提交为 `4907669`（`fix(multiplayer): complete Step 1 Pure CSR and Coach handover input broadcast`），已包含 Step 1 的全部实现与测试套件补全，工作区干净无需重复提交。

### 实测验证结果
- **`pnpm run check`（`radar` + `tsc --noEmit`）**：**PASS**（1425/1425 TS 文件，0 循环依赖 / 0 越界导入，体积护栏全部通过）。
- **`pnpm run test:multiplayer-suite`**：**22/22 PASS**，其中 **Section 22** 完整覆盖：
  1. Host Pure CSR `acknowledgedControls` 采集与 `GlobalPayloadCodec` 编解码；
  2. Client 观战端 `PredictionReconciler` -> `GameCameraSubsystem` -> `CarNoseOrientationArrows` -> `useKeyboardOverlayLiveInputs` 远端按键映射；
  3. **Coach 接管多端输入广播（Client Alpha -> Coach/Host -> Client Bravo）**：
     - Client Alpha 通过 `ClientAuthorityPublisher.publishIfReady` 附带 `acknowledgedControls`；
     - Coach/Host 端 `HostInputAuthorityTracker.ingestFrame` 写入 `latestAcknowledgedControls`，`GameCameraSubsystem` 在 `isHostSpectatingRemoteAuthority` 为真时直接消费 `hostAuthority.latestAcknowledgedControls` 渲染 Alpha 按键；
     - `SimulationStage` 将 `cf.acknowledgedControls` 通过 `server.pushBroadcastSnapshot(...)` 转发至 Pure CSR 广播，Client Bravo 同步收到并正确映射 Alpha 的输入。
- **`pnpm run repro:sync-issues`**：
  - `issue-1`：**`[NOT_REPRODUCED]`**（已修复）。
  - `issue-2` ~ `issue-5`：**`[REPRODUCED]`**（4/5，待后续 Step 2 / Step 3 推进）。

---

## 2. 针对架构问题的回复

### (1) Step 2：Cinematic Goal Replay Camera 能否直接复用现有 Match Replay 广播逻辑并拆分重组？

**结论：完全可以，而且非常契合“Spectate 传输层”与“Match 状态/镜头层”的正交拆分。**

从现有代码链路来看：
1. **镜头与 HUD 表现层（可直接复用）**：
   - Client 端 `RuntimeGoalReplaySignalSubController` 已经完整支持消费 `MATCH_SIGNALS.GOAL_REPLAY_WILL_START (0x87)` 与 `GOAL_REPLAY_START (0x88)`，并在收到信号后自动切入 `phase = 'replay'`，调用 `startGoalReplayDirectorSession`（`acquireGoalReplayCameraOverride`）接管 Cinematic / Director 镜头。
2. **帧流与状态来源层（正交拆分）**：
   - **Authoritative Match 模式**：由 Worker 内的 `MatchGoalReplaySubController` 负责从 `replayBuffer` 回放帧并向 `SignalEventBuffer` 写入 `GOAL_REPLAY_*` 信号。
   - **Pure CSR Spectate 模式（Freeplay / Exhibition Host 广播）**：Host 主线程的 `ExperimentalMultiBotsReplayController` / `FreeplayGoalAndTouchSubController` **本身已经在本地倒带并产出回放帧**，只需通过 `HostBroadcastPublisher` 持续推流帧快照，并通过 `server.pushBroadcastSignal(GOAL_REPLAY_WILL_START / GOAL_REPLAY_START / GOAL_REPLAY_WILL_END / GOAL_REPLAY_END, ...)` 把信号透传进 Pure CSR 广播包即可——**无需在 Worker 里套一层 `MatchGoalReplaySubController` 的物理回放逻辑**。
3. **唯一需要解除耦合的门控点**：
   - 在 `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`（以及 `SimulationStage.ts`）中，目前当 `flags.isGoalReplay` 为真时会门控住部分插值/同步分支。拆分后，只要判断当前处于 `isPureCsrSpectate`（由 Host 推流回放帧，而非 Client 本地 `ReplayTimelineController` 自驱回放），即使 `flags.isGoalReplay === true`，仍允许 `SpectateInterpolationBuffer` 继续采样推进位置与姿态，同时交由 `RuntimeGoalReplaySignalSubController` 驱动 Cinematic Director 镜头。
   - 这样一来，系统就清晰拆分为两个正交维度：**帧传输来源（`Authoritative Match` vs `Pure CSR Spectate`） × 表现相位（`Live` / `Goal Replay` / `Replay Viewer`）**，后续功能都可以直接按状态组合切换。

### (2) Step 3：边沿切换（Edge-Triggered Camera Sync）与多端 Coach 输入广播
- **Step 3 保持边沿切换**：`AuthorityCameraSyncTracker` 已建立基于 `authorityEpoch` + `seq` 的边沿触发与非覆盖原则（Non-Override）。后续在 Step 3 同步 `replaySync.focusedCarIndex` / `cameraSync` 时，仅在 `seq` 递增（发生切换事件）或处于强制跟随模式下应用远端目标，不让 1-2Hz 心跳覆盖 Client 本地相机自由度。
- **Coach 接管多端输入广播**：已在 `4907669` 打通 `Client Alpha -> Host/Coach (latestAcknowledgedControls + pushBroadcastSnapshot) -> Client Bravo`，后续若需进一步增强多 Client 切换时的连续性，可在 `SimulationStage` 中当 `remoteHasAuthority && frames.length === 0` 时保持上一帧缓存或显式复用 `hostInputAuthority.latestAcknowledgedControls`（当前 `MatchSystemControlSubController` 已缓存 `broadcastAcknowledgedControls`，因此中间空帧也会持续下发）。
