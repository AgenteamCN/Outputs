# 路 2 定位与修复报告：Replay Watch Together NameTag 全量同步 & 边沿触发视角同步（Bug 2）

## 1. 核心结论（确定性）

1. **Client `nameTagSystem` 底层 6 车 NameTag 数据与队伍色已完整同步，无产品代码 Bug**：
   - Host 进入 Replay Watch Together 并广播 `replaySync.roster` 后，Client 端通过 `devSyncChannelHandlers.ts`（`applyClientReplayActivitySync` 与 `onGlobalWorkspaceBroadcast`）调用 `syncReplayRosterTeamsAndNameTags` -> `projectMatchRoster`（`src/game/runtime/MatchRosterProjection.ts`），将全部 6 辆车（`carIndex: 0..5`）的 `name`、`username`、`team`、`isBot` 写入 `getNameTagRuntimeState().carEntries`，并同步更新 `physics.setCarTeam` 与 `arena.setCarTeam`（`arena.carTeams === [0, 0, 0, 1, 1, 1]`）。
   - `VehicleNameTagSystem.update`（`src/entities/arena/VehicleNameTagSystem.ts` L157-160）直接读取 `getNameTagRuntimeState().carEntries` 与 `arena.carTeams` 渲染所有车辆的 NameTag。
2. **Replay 刚加载进入 Watch Together 时已补发初始视角边沿事件，边沿触发（Edge-Triggered）同步链路完整**：
   - Host 在 `StandaloneReplayViewerController._initializeLoadedReplaySession`（`src/replay/StandaloneReplayViewerController.ts` L281-294）先调用 `rosterSubController.applyInitialPovToRuntime(this.runtime)`，再调用 `_emitStateTransition(0, true, true, loadedFileName)`。
   - `syncReplayWorkspaceToOnlineRoom`（`src/replay/replayOnlineSyncHelper.ts` L156-180）在 `opts.switchActivity && opts.active` 时执行 `sharedAuthorityCameraSyncTracker.bumpLocalSeq()`，将递增后的 `cameraSyncSeq` 随 `cameraSync` 广播给 Client。
   - Client 端 `AuthorityCameraSyncTracker.shouldApplyIncomingCameraSync`（`src/game/pipeline/stages/AuthorityCameraSyncTracker.ts` L110-163）仅在 `(epoch, seq)` 推进或初次收到新 key 时返回 `true`；在 Host 未切换视角期间（即使每 500ms 发送心跳），Client 的本地视角切换不会被 Host 心跳覆盖；一旦 Host 发生新的视角切换（`SimulationStage.ts` L411 `pollLocalCameraChange` 或 `ReplayOnlineHeartbeatTracker.maybeSync`），`cameraSyncSeq` 递增并立即同步给 Client。
3. **原 `runDevBroadcastReproV3.mjs` 断言耦合问题**：
   - 原脚本通过比较 `nts._visibleCards2D + nts._offscreenIndicators2D` 校验 NameTag，受 POV 聚焦车排除与屏幕视锥裁剪影响（实测 Host `tagsCount=4`、Client `tagsCount=3`），无法代表底层 6 车全量 NameTag 状态，且未测试边沿触发视角同步。

---

## 2. 代码与链路核查明细

### 2.1 6 车 NameTag 数据同步链路
- **Host 广播**：`src/replay/replayOnlineSyncHelper.ts`（L141-153）将 `rvStore.resolvedRoster` 映射为 `replaySync.roster`（含 `carIndex`、`name`、`team`、`isBot`），随 `room_activity_switch` 与 `global_workspace_broadcast` 发送。
- **Client 接收**：`src/game/runtime/devSyncChannelHandlers.ts`
  - `applyClientReplayActivitySync`（L130-135）：在 `room_activity_switch` 时调用 `syncReplayRosterTeamsAndNameTags`。
  - `onGlobalWorkspaceBroadcast`（L283-289）：在 `global_workspace_broadcast` 时调用 `syncReplayRosterTeamsAndNameTags`。
- **底层状态写入**：`src/game/runtime/MatchRosterProjection.ts`（L36-92）
  - 对每辆车调用 `physics?.setCarTeam(entry.carIndex, entry.team)` 和 `arena?.setCarTeam(entry.carIndex, entry.team)`。
  - 调用 `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode: roster.length > 1, carEntries })`，写入完整 6 车 `carEntries`。

### 2.2 边沿触发视角同步链路
- **初始进入边沿**：`StandaloneReplayViewerController._initializeLoadedReplaySession` -> `_emitStateTransition(0, true, true, loadedFileName)` -> `syncReplayWorkspaceToOnlineRoom(..., { switchActivity: true })` -> `sharedAuthorityCameraSyncTracker.bumpLocalSeq()`。
- **运行期边沿检测**：
  - `SimulationStage.ts`（L411-420）：`sharedAuthorityCameraSyncTracker.pollLocalCameraChange(camera, authEpoch, localPeerId)` 检测 `isFreeCam:ballCam:focusedCarIndex` 变化并发送 `camera_state_sync`。
  - `ReplayOnlineHeartbeatTracker.maybeSync`（`src/replay/replayOnlineSyncHelper.ts` L202-236）：检测 `cameraChanged` 时同样 `bumpLocalSeq()` 并广播。
- **Client 非覆盖保证**：`AuthorityCameraSyncTracker.shouldApplyIncomingCameraSync` 记录 `lastAppliedEpoch`、`lastAppliedSeq`、`lastAppliedKey`（均为上一次**已应用**的远端状态而非 Client 本地状态），因此 Host 2Hz 心跳重复发送相同 `(seq, key)` 时返回 `false`，Client 可自由本地切视角。

---

## 3. 本次修改与验证

### 3.1 修改文件
1. `src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`
   - 从 `../runtimeUiPort.ts` 导入 `getNameTagRuntimeState`，在 `window.__rcE2E` 上暴露 `getNameTagState: () => getNameTagRuntimeState()`，便于 E2E 直接核验底层 `carEntries`。
2. `scripts/runDevBroadcastReproV3.mjs`（Bug 2 用例）
   - 直接校验 Host 与 Client 的 `getNameTagState().carEntries`（6 辆车 `carIndex: 0..5`、`name`、`team` 全量一致）及 `arena.carTeams === [0, 0, 0, 1, 1, 1]`，同时保留 `tagsCount > 0` 渲染检查。
   - 新增 3 步边沿触发视角同步 E2E 断言：
     1. Host 切换 `focusedCarIndex = 2, ballCam = false` -> Client 在 5s 内跟随至 `(2, false)`；
     2. Client 本地切换 `focusedCarIndex = 4` 并等待 1.2s（Host 保持不动）-> Client 保持在 `4` 不被心跳覆盖；
     3. Host 再次切换 `focusedCarIndex = 5, ballCam = true` -> Client 在 5s 内切回 `(5, true)`。

### 3.2 实测结果
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN / PASS**（1430/1430 TS，0 循环依赖，0 类型错误）。
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**ALL MULTIPLAYER SUITE TESTS PASSED**。
- `pnpm run repro:dev-broadcast-v3`（Bug 2 实测输出摘要）：
  - `host.numCars: 6`, `host.arenaCarTeams: [0,0,0,1,1,1]`, `host.tagsCount: 4`
  - `client.numCars: 6`, `client.arenaCarTeams: [0,0,0,1,1,1]`, `client.tagsCount: 3`
  - `host.carEntries` 与 `client.carEntries` 均为完整 6 车（`Ducky_Blue_1..3` team `0`，`Ducky_Orange_1..3` team `1`），`carEntryMismatches: []`, `teamMismatch: false`
  - `edgeSync`: `edgeSyncOk: true`
    - `clientAfterHostEdge1`: `{"focusedCarIndex":2,"ballCam":false,"isFreeCam":false}`
    - `clientAfterLocalSwitch`: `{"focusedCarIndex":4,"ballCam":false,"isFreeCam":false}`
    - `clientAfterHostEdge2`: `{"focusedCarIndex":5,"ballCam":true,"isFreeCam":false}`
  - 结果：`[PASS_FIXED] bug-2-replay-nametag-and-team-desync`。

---

## 4. Git 提交与分支信息

- 新建分支：`z/agy/rc/track2-bug2-nametag-edgesync`（基于 `origin/z/agy/rc/multiplayer-recheck-base` @ `02adc3d`）
- Verified Bot Commit：`826162b5baf63ab34c9c8dbb90d3d56917030da3`（`verified: True`）
- Commit Message：`test(e2e): verify 6-car NameTag carEntries and edge-triggered camera sync in Replay Watch Together`
