# RocketCompanion 全链路同步（Sync All / CSR Broadcast）扫描与推进方案

## 一、已完成的仓库操作与基线验证

1. **已将最新 `alpha`（`8b6b0a2b91036d03c05886c0ddbe0e5f1b403f87`）合并至 `z/agy/rc/multiplayer-recheck`（`e6bb6378`）并完成 Verified Bot Push**：
   - 合并提交：`0fb08304e80343c37880c8ccb6f0f5c4d6bf8d2c`（`verified: true`，`antigravity-fine-grained-token[bot]` 自动签名，0 冲突）。
   - 确认了先合并 `alpha` 的必要性：PR #20（`2fb9052`）对 Floating Window 做了结构性重构，强制 `PresetConfig`（`iconName` / `iconColor` / `title`）与 `SURFACE_META`、`floatingSurfaceRegistry.ts` 保持 1:1 一致，并由 `scripts/radar/radarUiAndRepoGuards.ts` 静态门禁强制校验。
2. **工具链与门禁实测全绿**：
   - 已修复沙盒 `pnpm` shim 问题，安装 `pnpm@10.28.2` 并完成 `pnpm install`。
   - 合并后树上实测通过：
     - `pnpm run check`：PASS
     - `pnpm run radar:audit`：PASS
     - `pnpm run test:multiplayer-suite`：10/10 ALL PASS

---

## 二、核心策略判断：先验证"完全同步（Sync All / Pure CSR）"，还是先全做完再切分？

**明确结论：必须采用"先跑通并验证 Pure CSR 完全同步（Sync All），再按模式逐步开放特定用户输入"的路线。**

原因如下：
1. **Pure CSR（只读渲染广播）是所有上层联机模式的数学超集**：
   - 在 Pure CSR 模式下，Host 是唯一物理与状态权威（`SNAPSHOT` + `GlobalWorkspaceBroadcast` + `TELESTRATOR_SYNC`），Client 全程处于 `clientInputEnabled = false` 的纯渲染态。
   - 一旦 Pure CSR 跑通并验证通过：
     - **Replay Watch Together** = Pure CSR + `StandaloneReplayViewerController` 注入 `enterStateOnlyMode`；
     - **Coaching / 教学观战** = Pure CSR + Telestrator 画画；
     - **Custom Training Pack** = CSR + `training_shot_reset` + 按需开启 `clientInputEnabled`；
     - **Room Match 自动拉入** = 复用已建好的 P2P `DataChannel`，切换 `activityId: 'match'` 并开放 `clientInputEnabled`。
2. **如果先把所有模式与输入逻辑全堆上去再整体验证**，一旦出现画面抖动、状态不同步或模式跳变，很难区分是 `PredictionReconciler` 预测回滚问题、`GameModeManager.setMode()` 本地副作用问题，还是传输层同步问题。
3. **先验证 Sync All 再逐层解禁输入**，每一步都可以独立在 `test:multiplayer-suite` 和 Headless Chrome 中卡点验收，风险完全可控。

---

## 三、底层 Signaling 与数据包解析是否需要改动？

### 1. 底层 Signaling：不需要改动
- `WebSocketSignalingClient`、`SupabaseRealtimeSignalingClient`、`SignalingProtocolHandler`、`HostSignalingNegotiationSubController`、`ClientJoinNegotiationSubController` 均与房间内具体活动（Freeplay / Replay / Training / Match）完全解耦。
- 普通 Client 通过 `Layer3JoinRoomRecipe` -> `onJoinOnlineWarmup` 输入 Room Code 即可完成 SDP/ICE/Relay/Tunnel 建连，无需任何 URL 后缀或信令协议改动。

### 2. 底层二进制数据包解析：核心编解码器无需大改，仅需两处增量接线
- `TunnelProtocol.ts` 已完整定义 `PHYSICS_SNAPSHOT (0x01)`、`USER_INPUT (0x02)`、`RELIABLE_CONTROL (0x03)`、`PING_PONG_16B (0x04)`、`TELESTRATOR_SYNC (0x05)`、`CSR_STATE_BROADCAST (0x20)`、`ROOM_ACTIVITY_SWITCH (0x21)`。
- `GlobalPayloadCodec.ts` 已包含 `stateSnapshot`（球、车、34 个 Boost Pad 位掩码、确认控制帧）、`signals`（进球/爆炸/Boost/跳跃/触球等音效与特效事件）与 `allUserRtt`。
- **仅需两处轻量接线**：
  1. **打通 `TELESTRATOR_SYNC (0x05)` 的传输层收发**：`TelestratorWireCodec.ts` 和 `ReliableTunnelFramer.ts` 已就绪，但 `DataChannelTransport.ts`、`dataChannelFramingPipeline.ts` 和 `P2PWebRTCChannel.ts` 目前缺少 `sendTelestratorBytes` / `onTelestratorBytes` 的收发分发。
  2. **扩展 `GlobalWorkspaceProtocol.ts`（JSON 控制帧）**：在 `GlobalWorkspaceBroadcastPayload` 中补充 `clientInputEnabled: boolean`、`followHostWorkspace: boolean` 与可选的 `arenaPreset` / `cameraSync` 字段，复用已有的 `sendGlobalWorkspaceBroadcast` 通道下发。

---

## 四、Drawing（Telestrator 画画）同步现状与隐藏坑点

1. **现有基础**：`TelestratorWireCodec.ts` 已实现 `STROKE_COMMIT`、`STROKE_HASH_ACK`、`STROKE_UNDO`、`STROKE_CLEAR`、`STROKE_DRAFT`、`HOST_PERMISSION_SYNC` 的紧凑二进制编解码；`TelestratorOverlaySubsystem` 当前仅在 `telestratorOverlayHelpers.ts` 中做本地自环编解码校验（`verifyTelestratorStrokeWire`）。
2. **扫描发现的 3 个关键坑点**：
   - **权限与相机同步门控卡死 Client 绘制**：`telestratorOverlayHelpers.ts` 的 `canTelestratorDrawNow()` 中硬编码了 `if (!st.isCameraSynced) return false;`。在 Sync All 房间中，若未将 Client 的 `isCameraSynced` 置为 `true`（或在 `everyone` 权限下放行），Client 即使拿到权限也无法落笔。
   - **双 Store 未联动**：`useTelestratorStore`（`allowClientDrawing` / `isHost` / `isCameraSynced`）与 `useSpectateWorkspaceStore`（`telestratorPermission`）是两个独立 Store，收到 `GlobalWorkspaceBroadcast` 时必须同步更新两者。
   - **3D 世界坐标特性**：Telestrator 笔画基于相机射线投射到场地 3D 几何体上（只要 Host 与 Client 加载相同场地，不同分辨率下天然对齐，无需屏幕坐标换算）。

---

## 五、全面扫描发现的其他遗漏点（关键风险清单）

除原报告已列出的点外，本次全量扫描还发现以下 **6 个关键遗漏**：

1. **`applyClientRoomActivitySwitch()` 模式 ID 不匹配与本地文件加载冲突**（`src/game/runtime/onlineSessionEnvironmentPipeline.ts:305-325`）：
   - 该函数将 `'multiplayer_training'` 映射为 `'training'`、将 `'replay'` 映射为 `'replay'`，但 `GameModeManager.ts` 注册的模式 ID 实际上是 `'static-training-pack'` 和 `'replay-viewer'`，导致静默回退到 `'freeplay'`。
   - 更重要的是：Client 本地**没有** Host 的 `.RCDPack` / `.RCSPack` / `.RCReplay` 文件，若切入本地模式会尝试加载不存在的文件或覆盖物理状态。因此 Client 在 CSR 房间中应保持轻量化 CSR 接收态（仅由 Host `SNAPSHOT` + `stateSummary` 驱动画面与 HUD），同时当 `activityId === 'match'` 时需避免 `store.setMatchMode('match')` 意外触发本地 `BotMatchMode` AI。
2. **场地 / 地图预设（Arena / Throwback Stadium）未同步**：
   - `GlobalPayloadCodec` 的物理快照不包含地图 ID。如果 Host 在 Under Development 中切到 `throwback` 球场或特定训练包球场，Client 若仍停留在 `standard` 球场，碰撞体与 Boost Pad 位置将错位。需在 `ROOM_ACTIVITY_SWITCH` / `GlobalWorkspaceBroadcast` 中携带 `arenaPreset`。
3. **Host 视角 / 相机同步（`FOLLOW_HOST_CAMERA`）尚未实现**：
   - `ReplicationBusProtocol.ts` 定义了 `FOLLOW_HOST_CAMERA` 标志位，但当前没有任何路径在广播 Host 的相机模式（BallCam / CarCam / FreeCam）或观战目标车索引。在 Replay 观看或教学演示时，若不同步相机目标，Client 视角会停在原地。
4. **Workspace 镜像默认未自动激活**：
   - `spectateWorkspaceStore.ts` 默认 `spectateMode: 'spectate_state_only'` 且 `isSpectating: false`。如果不由 Host 房间策略触发 `spectate_mirror`（同时利用已有的 `localSnapshotBeforeMirror` 在退房时无损恢复 Client 本地浮窗），Client 收到 `GlobalWorkspaceBroadcast` 后不会自动打开/同步 Host 的浮窗和菜单路由。
5. **Room Match 自动拉入已连入 P2P Client 的机制缺失**：
   - 当 Host 将一个已建立 P2P 连接的 Sync 房间直接切到 `Match` 时，不需要重新走 Signaling 邀请（`DataChannel` 已连通），但目前 `MatchRosterPolicyState` 会等待 Client 手动发 `arena_loaded_ack` 与 `match_ready_ack`。需要在 `MatchActivityKernel.onMount` 或 `OnlineMatchControlBridge.switchRoomActivity('match')` 中支持自动为已连接的 P2P Peer 分配蓝/橙队槽位并触发自动 Ready/Start。
6. **Replay Takeover 已有现成 `enterStateOnlyMode` 桥接**（`src/replay/ReplayTakeoverSubController.ts:169`）：
   - `ReplayTakeoverSubController` 已实现 `server.enterStateOnlyMode?.(snap, param1)`，且 `simulationNetworkReconcilePipeline.ts` 已处理 `reconciler.replayPlaybackMode`。Replay CSR 只需补齐 `StandaloneReplayViewerController` 在线房间播放时的逐帧推流与暂停心跳同步即可，工作量比原报告预估更小。

---

## 六、调整后的 5 步推进方案（单分支 `z/agy/rc/multiplayer-recheck` 串行收拢）

### Step 1：Pure CSR "Sync All" 房间闭环 + 对齐 `alpha` 新规范的 `DevRoomSyncFloatingWindow`
- 在 `Layer3DevUnderDevelopmentRecipe.tsx` 新增 "Create Developer Sync Room" 入口，调用 `bridge.onHostOnlineWarmup` 并打开 `DevRoomSyncFloatingWindow`（严格遵循 `alpha` `2fb9052` 的 `PresetConfig` + `SURFACE_META` + `floatingSurfaceRegistry` 1:1 规范）。
- 扩展 `GlobalWorkspaceProtocol.ts` 与 `spectateWorkspaceStore.ts`（`clientInputEnabled`、`followHostWorkspace`、`arenaPreset`），在 `HostSessionController` / `MultiplayerSessionManager` 接入 `broadcastGlobalWorkspace`。
- 实现前后双端 Input 门控（`PredictionReconciler.sampleAndPredictInput` + `HostWorkerMessageRouter` `clientInput` 清零）。
- 修复 `applyClientRoomActivitySwitch()`，确保 Client 在 CSR 房间中不触发本地文件加载或 `BotMatchMode` 抢占。
- **验收**：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite`（新增 Test 11 验证 Sync All + Input 门控 + Workspace 广播）。

### Step 2：Telestrator（0x05）全链路实时同步
- 在 `DataChannelTransport.ts`、`dataChannelFramingPipeline.ts`、`P2PWebRTCChannel.ts` 接入 `TELESTRATOR_SYNC (0x05)` 收发。
- 在 `HostSessionController.ts` 实现权限校验（`host_only` / `everyone`）与全房间转发 + `STROKE_HASH_ACK`。
- 联动 `useTelestratorStore` 与 `useSpectateWorkspaceStore`，修复 `isCameraSynced` 在 CSR 房间中的绘制门控。
- **验收**：`test:multiplayer-suite` 新增 Test 12（双端 `STROKE_DRAFT` / `STROKE_COMMIT` / `STROKE_UNDO` / `STROKE_CLEAR` 与权限拦截）。

### Step 3：Replay Watch Together（Pure CSR + 相机同步）
- 将 `StandaloneReplayViewerController` 播放/暂停/倍速/拖拽时间轴接入 `authoritativeServer.enterStateOnlyMode(snap, tick)` 与 `GlobalWorkspaceBroadcast`（`playbackPaused`、`pausedAtTick`、相机目标）。
- **验收**：`test:multiplayer-suite` 新增 Test 13（Replay 帧注入 -> Client CSR 插值同步误差 < 1e-3 + 暂停/拖拽同步）。

### Step 4：按需开放用户输入（Custom Training Pack 教练模式 & Room Match 自动拉入）
- **Custom Training Pack**：扩展 `MultiplayerTrainingActivityKernel` 支持 `training_shot_reset`（服务端重置球/车位姿）与 `stateSummary` 题目元信息同步；Client 无需本地 `.RCDPack` 文件即可同步渲染并按 Host 开关参与击球训练。
- **Room Match 自动拉入**：在 `switchRoomActivity({ activityId: 'match' })` 时自动将当前房间已建连的 P2P Peer 分配至蓝/橙队并触发自动 `arena_loaded_ack` / 开局，无需重新建房或二次邀请。
- **验收**：`test:multiplayer-suite` 新增 Test 14。

### Step 5：Linux Headless Chrome 真实多标签页 E2E 验收
- 扩展 `scripts/runMultiplayerRoomRepro.mjs`（或新增 `runDevRoomSyncE2E.mjs`），在无 GUI 沙盒下用双标签页 Headless Chrome 跑通：Host Dev UI 建房 -> 普通无后缀 Client UI 加房 -> Pure CSR Sync All -> Input 门控 -> Telestrator 画画 -> Replay 同步 -> Training Pack -> Match 自动拉入。
- **硬断言**：`runtime_error === 0`、`poseAnomalies === 0`、`stuckInGoal === false`。
