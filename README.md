# RocketCompanion Dev Sync Room 合并与诊断报告

## 1. 分支合并与环境验证结论

- **Tauri 更新合并情况**：`origin/z/agy/tauri`（`78d0e0239792da9c518a62b54bba3581434adbe9`）已通过合并提交 `a78fc05` 无冲突并入 `z/agy/rc/multiplayer-recheck`，随后仓库最新提交 `3918fae`（`fix(dev-sync): enforce spectating-based freeplay/replay sync and add menu broadcast + client notification`）已在此基础上落地核心修复。
- **在最新 HEAD（`3918fae`）上的实际校验结果**：
  1. 静态检查：`pnpm run check`（`radar` + `tsc --noEmit`）**PASS**（1415/1415 TS 文件，0 类型错误，0 架构边界违规）。
  2. 单元测试：`pnpm run test:multiplayer-suite` **PASS**（15/15 项全绿）。
  3. 无头浏览器双标签 E2E：`RC_VITE_ROLE=rc-http pnpm run test:dev-room-sync-e2e` **PASS**（9/9 阶段全绿，耗时 70832ms，`runtime_error=0`，`poseAnomalies=0`，`stuckInGoal=false`）。
     > 注：在无头 Chrome 中若使用默认 HTTPS（`basicSsl` 自签证书），会因 `net::ERR_CERT_VERIFIER_CHANGED` 导致 Microkernel 资源加载失败；使用 `RC_VITE_ROLE=rc-http`（`127.0.0.1` HTTP Secure Context）可正常启动并完成全链路校验。

---

## 2. 为什么之前的自动化测试显示通过，但实际使用完全不对？

### 2.1 `scripts/runMultiplayerSuiteTests.ts`（15 项测试）属于纯内存 Mock 单元测试
- 测试中使用的是空壳 `mockSim`（`getStateSnapshot: () => new Float32Array(0)`）以及内存直连回调替代真实 `RTCDataChannel`。
- 该套件**从未启动** `GameRuntime`、`core.wasm`、`EntityManager`、`OnlineLobbyDualWorldController`、`ReplayTimelineController` 或 `StaticTrainingPackMode`，因此无法暴露实体错乱、WASM 内存越界或回放/训练模式冲突。

### 2.2 `scripts/runDevRoomSyncE2E.mjs` 在 `74fe936` 版本存在明显断言盲区
在 `3918fae` 补强之前，`runDevRoomSyncE2E.mjs` 存在以下漏测：
1. **Stage 3（Freeplay / Pure CSR）**：仅断言 Host 与 Client 的 `ballPos` 距离 `< 50`，**未检查**场上总车辆数（是否多生成了 1 号实体车）、未检查 Client 的 `playerCarIndex` 是否仍为 `1`、未检查相机是否聚焦在 0 号车。
2. **Stage 6（Replay Watch Together）**：**未加载真实 3v3（6 车）回放**，仅点击 `Replay` 按钮后手动调用 `phys.setBallState` 与 `server.enterStateOnlyMode(snap, 420)`，且只断言了 `playbackPaused` 和 `ballPos`。
3. **Stage 7（Custom Training Pack）**：仅直接调用 `broadcastRoomActivitySwitch({ activityId: 'multiplayer_training' })` 与 `training_shot_reset` 元数据同步，**未实际进入 `StaticTrainingPackMode` 并驱动物理帧**。
4. 此外，在未合并 `3918fae` 的 `74fe936 + 78d0e02` 上实测 `test:dev-room-sync-e2e`，**Stage 9（Match -> Replay 反向切换）直接复现了客户端崩溃**：`EntityManager.hotSwapVehicle` -> `RocketSimPhysicsEngine.setCarHitbox` -> `core.wasm` `RocketSim::Arena::AddCar` 抛出 `RuntimeError: memory access out of bounds`，导致客户端卡在 `match` 状态超时失败。

---

## 3. 总体设计诊断与各模式问题根因

核心设计矛盾在于：**Developer Sync Room 本质上是“单向状态广播 + 观战同步（Spectating-based）”，但早期实现直接复用了“多人对战大厅（Multiplayer-based）”的实体初始化与对账管线。**

### 3.1 Freeplay：客户端多生成一辆车在场地中央、看不到 Host 动作且干扰 Host
- **根因**：
  1. 客户端调用 `OnlineSessionCoordinator.joinOnlineServer` 时，无条件执行 `OnlineLobbyDualWorldController.prepareClientJoinLobbyWorld`：强行将 `world.playerCarIndex` 设为 `1`、开启 `arena.isLobbyMode = true`、生成 1 号车、执行 `isolateNonLocalLobbyCars(1)` **把 Host 的 0 号车隐藏**，并把客户端镜头聚焦到 1 号车。
  2. 客户端发出的 `join` 请求默认 `role: 'player'`，Host 侧 `MatchRosterPolicyState.assignPlayerSlot`（及 `matchControlPacketPipeline`）为其分配了物理车位（`entityId = 1`）并在 Host 的 `AuthoritativeServer` 物理世界中实体化，导致场中多出一辆车直接阻挡/干扰 Host。
  3. 即使收到 `isDevSyncRoom: true` 的 `handshake_response`，后续状态同步仍保留了双车/双世界残留状态。
- **当前状态（`3918fae`）**：已改为默认 `spectator` / 观战广播模式，Client 不再在 Host 侧或本地生成多余实体车，保持 1 车视角跟随 Host。

### 3.2 Replay Viewing（3v3 六人回放：Host 车消失只剩轨迹/Boost、Client 只剩 2 辆车）
- **根因**：
  1. **Host 自回环覆写与强制减车**：Host 在 Dev Sync Room 中同时运行 `AuthoritativeServer` 和自身的 `networkReconciler`（见 `simulationNetworkReconcilePipeline.ts`）。进入 Replay 查看器后，Host 的 `networkReconciler.replayPlaybackMode` 未置为 `true`，每帧仍执行 `reconciler.reconcile(now)`，接收 `AuthoritativeServer` 发回的 2 车快照；`NetworkTelemetryCollector` 随即执行 `while (sim.numCars > serverCarCount) sim.removeCar(...)`，**把 Host 刚生成的 6 辆回放车强行删减回 2 辆并覆盖物理状态**。与此同时，JS 层的 `ReplayTimelineController` / `ReplayViewerCausalSyncController` 仍在渲染 6 辆车的尾迹和 Boost Collect 特效，从而出现“能看到轨迹和吃包特效，但看不到车”。
  2. **服务端 `stateOnlyMode` 未按快照车辆数扩容**：`MatchSystemControlSubController.loadSimSnapshot` 在恢复快照时，`AuthoritativeServer` 内部 `sim` 仍停留在 2 车槽位，导致下发给 Client 的快照也只包含 2 辆车。
- **当前状态（`3918fae`）**：已在 `simulationNetworkReconcilePipeline.ts` 中跳过 Host 在回放模式下的自回环对账，并在 `MatchSystemControlSubController` 中读取 `SIM_OFFSETS.NUM_CARS` 以支持 6 车（3v3）状态同步；`runDevRoomSyncE2E.mjs` Stage 6 也已补充 6 车断言并通过验证。

### 3.3 菜单状态广播开关与 Client 常驻退出/暂停通知
- **设计定位**：菜单路由（`menuRoute`）和浮窗列表（`openFloatingWindows`）属于 UI 工作区镜像，不属于世界状态（World State）。
- **当前状态（`3918fae`）**：
  - `GlobalWorkspaceProtocol`、`spectateWorkspaceStore`、`runtimeSpectateWorkspaceBridge` 与 `DevRoomSyncFloatingWindow` 已新增 `broadcastMenuState` / `menuStateIncluded` 选项（早期测试默认 `true`），保留了“仅广播世界状态、不同步菜单”的开关接口。
  - `OnlineRosterSyncCoordinator` 已为 Dev Sync Room Client 接入基于 `QuickNotificationStore` 的常驻通知（`persistent`），提供 **Stop Syncing**（停止跟随同步）与 **Exit Room**（退出房间）按钮，并支持点击右上角铃铛图标收起以获得纯净画面。

---

## 4. 仍需推进的下一步验证与收尾事项

1. **Custom Training Pack（`StaticTrainingPackMode`）实机与 E2E 闭环**：
   - 目前 E2E Stage 7 仍仅测试 `multiplayer_training` 活动切换与 `training_shot_reset` 元数据，尚未真正进入 `StaticTrainingPackMode` 并校验 Host 切球/重置时 Client 的实时画面同步。
   - 需重点排查 `staticTrainingSessionPipeline.ts` 中的 `teardownRuntimeNetworkForStaticTraining`（确保在 `isDevSyncRoom` 下不会意外拆掉房间连接），并将 `StaticTrainingPackMode` 的物理状态接入 `authoritativeServer.enterStateOnlyMode` 广播。
2. **Telestrator“一起画画”与“不广播菜单”的解耦**：
   - 目前 `telestratorPermission === 'everyone'` 已允许 Client 绘图（`setAllowClientDrawing(true)`），但若关闭了菜单/浮窗广播（`broadcastMenuState = false`），Host 开始画画时 Client 不会自动激活 Telestrator 画板遮罩。
   - 建议在 `GlobalWorkspaceBroadcastPayload` 中增加独立的 `telestratorOverlayActive`（仅当 `telestratorPermission === 'everyone'` 时同步），使 Client 无需依赖菜单广播即可直接一起画画。
3. **真实回放文件与 Tauri 客户端人工回归**：
   - 建议在 Tauri 桌面端加载真实 3v3 `.RCReplay` 文件复核 6 车网格、NameTag、镜头切换及窗口重开后的音频恢复（`a78fc05`）。
