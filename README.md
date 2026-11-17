# RocketCompanion Developer Sync Room 诊断、Tauri 合并与修复报告

## 1. 仓库与分支操作结果
- **Tauri 更新合并**：已将 `origin/z/agy/tauri`（`78d0e0239792da9c518a62b54bba3581434adbe9`）无冲突合并至 `z/agy/rc/multiplayer-recheck`（原基线 `74fe936e5eb11b5c3ffa2382b18460f6a8df029c`）。
- **Verified Bot Commits**（均已通过 GitHub App 自动 GPG 签名，`verified: true`）：
  - Merge Commit: `a78fc05c0a28c92ee3ac0a542117e0efae90b663`（合并 `origin/z/agy/tauri`）
  - Fix Commit: `3918fae6ca21e21c2ed3af0f91617b287073a9cd`（架构与同步修复 + 接口增强 + E2E 断言加固）

---

## 2. 总体架构诊断：为什么会出现这些“离谱”现象？

你判断得非常准确——**核心问题是设计思路错位，而非底层传输（WebRTC / BroadcastChannel / Tunnel）连接故障**。

早期实现把 `Developer Sync Room` 当成了 **Multiplayer-based（多人实体化对抗房间）**，再在其上打“禁输入 / 观战”的补丁；而实际使用场景是 **Spectating-based（主机状态广播 / 导演观战模式，仅在显式切入 Match / 允许画画等场景下才临时拉起交互）**：

1. **Freeplay 中 Client 强行生成第二辆车并挡路、视角锁死在自己车上**：
   - **原因 A（服务端误判为非训练/自动开赛房间）**：`MatchRosterPolicyState` 在 `isDevSyncRoom: true` 且 `lobbyMode: false` 时，将 `isTrainingMode` 默认设为 `false`、`autoStartWhenFull` 默认设为 `true`；同时 `MatchRosterSubController.assignPlayerSlot` 把 `state === 'playing'` 视为正式比赛，导致 Client 加入时被分配了一个实体车位（`car 1`）并插进场地中央/开球位，直接阻挡 Host（`car 0`）。
   - **原因 B（Client 根本没收到/没处理 `handshake_response`）**：`MatchRosterPolicyState.buildHandshakeJson` 在 `handshake_response` JSON 中附带了 `serverTick: this.matchTick`，而客户端 `reconcilerInboundPacketParser.ts` 先判断 `if (typeof obj.serverTick === 'number')`，后判断 `else if (obj.type === 'handshake_response')`！导致所有 `handshake_response` 均被当成普通物理帧吞掉，Client 一直停留在 `joinOnlineServer` 预创建的双车默认状态（`numCars: 2, playerCarIndex: 1, focusedCarIndex: 1, isDevSyncRoom: false`）。
   - **原因 C（`applyRosterSnapshot` 时序反转）**：`OnlineMatchReconcilerBinder.handleClientHandshakeResponse` 先调 `applyRosterSnapshot` 后调 `setOnlineSession`，导致首次同步名册时 `isDevSyncRoom` 仍为 `false`，Client 名册与视角目标错乱。

2. **3v3（6 车）Replay 回放中“有轨迹/Eat Boost，但车消失 / 只剩 2 辆车”**：
   - **原因 A（Snapshot 车数偏移量读错）**：`SIM_OFFSETS.NUM_CARS` 在协议中是索引 **`2`**（索引 `0` 是 `TICK`，索引 `1` 是 `GOAL`），但 `MatchSystemControlSubController` 的 `loadSimSnapshot` 和 `stepStateOnlyFrameIfActive` 读取了 `snap[1]`（`GOAL`），导致 `headerCars / snapCarCount` 经常读成 `0`，`stateOnlyCarMask` 回退为 `physicsManager.getActiveCarMask()`（仅 `1` 或 `3`，即 1~2 辆车），Client 只收到前 1~2 辆车的状态，6 车轨迹与吃球/吃包事件却照常渲染。
   - **原因 B（Host 回放自回声覆盖）**：Host 开启 `replayPlaybackMode` 后，`simulationNetworkReconcilePipeline.ts` 仍对非 Goal Replay 的 `drainedReplayPackets` 执行 `applyReplayFrameToEngine`，用 Worker 回传的 2 车包覆盖了 Host 本地正在播放的 6 车回放物理状态。

3. **为什么之前的自动化测试显示“全部通过”？**
   - `scripts/runMultiplayerSuiteTests.ts`（15 项）全部基于内存 Mock（`mockSim`、`mockPhysicsManager` 等），没有接入真实的 `RocketSim` WASM、`AuthoritativeServerWorker`、`PredictionReconciler` 报文解析器和 `AppStore` 状态。
   - `scripts/runDevRoomSyncE2E.mjs` 虽然跑了真实双标签页 Headless Chrome，但存在**断言盲区**：
     - Stage 2/3 只检查了 `connectedPeers >= 1` 和球坐标距离 `< 50`，**从未检查** `physics.numCars`、`playerCarIndex`、`focusedCarIndex`、`isDevSyncRoom` 是否真为 `1` 车观战状态；
     - Stage 6（Replay）只注入了单车状态下移动的球位置，**从未测试** 3v3（6 车）回放时的双端 `numCars === 6`。

4. **Telestrator（一起画画）与 Training Pack 状态确认**：
   - **Telestrator**：底层与 UI 均已实现。`TelestratorWireCodec`（`0x05`）+ `hostTelestratorRelay` + `telestratorStore.allowClientDrawing` + `activeGlobalWorkspace.telestratorPermission === 'everyone'` 已打通；当 Host 开启 `All Clients: ON`（或 `telestratorPermission: 'everyone'`）时，Client 可直接同步绘制并双向广播笔画。
   - **Training Pack**：通过 `ROOM_ACTIVITY_SWITCH`（`multiplayer_training`）与 `training_shot_reset` 同步元数据与重置动作；Client 默认作为旁观视角跟随 Host 的 `car 0` 演练。

---

## 3. 本次已完成的核心修复与新增能力

1. **Spectating-based 默认模型固化（消除 Freeplay 幽灵车 & 修复握手）**：
   - `src/network/client/reconcilerInboundPacketParser.ts`：调整分支顺序，优先处理 `roster_update` 与 `handshake_response`，修复 `handshake_response` 因携带 `serverTick` 被误判吞掉的根本 Bug。
   - `src/network/server/match/MatchRosterPolicyState.ts` & `HostWorkerMessageRouter.ts`：`isDevSyncRoom && !lobbyMode` 时强制 `isTrainingMode = true`、`state = MATCH_STATES.PLAYING`、`autoStartWhenFull = false`，并在旁观握手包中携带 `initialSnapshot`。
   - `src/network/server/match/MatchRosterSubController.ts`、`matchRosterSlotAssignmentPipeline.ts`、`matchControlPacketPipeline.ts`、`ActivityKernel.ts`：非比赛态加入 Dev Sync Room 的非 Host 客户端统一分配为 `'spectator'`；从 Match 退出回到 Freeplay/Replay/Training 时自动降级回 `'spectator'`。
   - `src/game/runtime/OnlineMatchReconcilerBinder.ts`、`OnlineRosterSyncCoordinator.ts`、`onlineLobbyTransitionPipeline.ts`、`GameCameraSubsystem.ts`：先写入 `onlineSession.isDevSyncRoom` 再应用名册；Client 旁观态保持 `numCars = 1`、`playerCarIndex = 0`、`focusedCarIndex = 0`，彻底消除场地中央的幽灵车并锁定 Host 视角。

2. **6 车（3v3）Replay 同步修复**：
   - `src/network/server/match/MatchSystemControlSubController.ts`：改用 `SIM_OFFSETS.NUM_CARS`（索引 `2`）读取快照车数，在 `loadSimSnapshot` 中提前扩容 `pm.sim` 槽位，在 `stepStateOnlyFrameIfActive` 中生成正确的 `stateOnlyCarMask`（6 车时为 `0x3f`）。
   - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`：当 `server && !flags.isGoalReplay` 时跳过 `applyReplayFrameToEngine`，防止 Host 自我回声覆盖本地回放。

3. **仅广播 State 接口（菜单状态可选剥离）**：
   - 在 `GlobalWorkspaceProtocol.ts`、`spectateWorkspaceStore.ts`、`runtimeSpectateWorkspaceBridge.ts` 及 `DevRoomSyncFloatingWindow.tsx` 新增 `menuStateIncluded` / `broadcastMenuState` 开关（默认保持 `true` 以便早期联调，关闭后仅广播相机/播放/活动状态，不再同步菜单与浮窗）。

4. **Client 常驻状态通知与快捷退出**：
   - 在 `runtimeSpectateWorkspaceBridge.ts` 中通过 `QuickNotificationSystem` 推送常驻通知（`id: 'dev-sync-room-client'`, `type: 'ongoing'`），提供 **Stop Syncing / Resume Sync** 与 **Exit Room** 操作按钮，支持通过右上角小铃铛图标收起为纯净画面，离开房间时自动销毁。

5. **自动化测试加固**：
   - 在 `scripts/runDevRoomSyncE2E.mjs` 的 Stage 2 增加了 Freeplay 双端 `numCars === 1`、`client.playerCarIndex === 0`、`client.camera.focusedCarIndex === 0`、`isDevSyncRoom === true` 硬断言；在 Stage 6 增加了 6 车（3v3）Replay 快照双端 `numCars === 6` 硬断言。

---

## 4. 最终验证结果
- `pnpm run check`（Radar 架构边界 + TypeScript `tsc --noEmit`）：**PASS（0 错误）**
- `pnpm run test:multiplayer-suite`（15 项联机协议与状态机测试）：**PASS（15/15）**
- `pnpm run test:dev-room-sync-e2e`（双标签页真实 Headless Chrome E2E，含 Freeplay 1 车观战与 6 车 Replay 断言）：**PASS（9/9 阶段全部通过，`runtime_error=0`, `poseAnomalies=0`, `stuckInGoal=false`）**
