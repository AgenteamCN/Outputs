# RocketCompanion Multiplayer Step 4 推进报告

- **仓库 / 分支**: `Antigravi-ty/RocketCompanion` @ `z/agy/rc/multiplayer-recheck`
- **Verified Bot Commit**: `938f846b7ad7f32f63ad49c57d4b779703d4019b` (`signature.isValid: true`, `state: VALID`, author: `antigravity-fine-grained-token[bot]`)
- **Commit 链接**: https://github.com/Antigravi-ty/RocketCompanion/commit/938f846b7ad7f32f63ad49c57d4b779703d4019b

---

## 一、前序基础状态核查（Steps 1–3）

基于 `origin/z/agy/rc/multiplayer-recheck`（基线 `2e76b7d`）检查并实测：
- **Step 1** (`91ed3b5`): Pure CSR Sync All、双端 Input 门控与 `DevRoomSyncFloatingWindow` 已就绪。
- **Step 2** (`9af302d`): Telestrator (`0x05`) 全链路实时同步与权限/双 Store 联动已就绪。
- **Step 3** (`2e76b7d`): Replay Watch Together（Pure CSR 帧注入、暂停心跳、相机同步）已就绪。
- 基线 `pnpm run check`、`pnpm run radar:audit` 与 `pnpm run test:multiplayer-suite`（Tests 1–13）均通过。

---

## 二、Step 4 核心实现详情

### 1. Custom Training Pack 教练模式与无本地 `.RCDPack` 同步
- **`src/network/server/activities/MultiplayerTrainingPolicy.ts`**:
  - 新增 `TrainingShotMetadata`、`TrainingShotResetPayload`、`extractTrainingShotMetadata()` 与 `applyTrainingShotResetToSim()`。
  - 服务端收到 `training_shot_reset` 时直接重置球位姿（`setBallState`）与单车/全车/多车位姿（`setCarState`），并同步重置 `resetGoalLatch()` 与 Boost Pads，递增 `shotResetSeq`。
- **`src/network/server/activities/ActivityKernel.ts` (`MultiplayerTrainingActivityKernel`)**:
  - `onMount` 提取并保存 `shotMetadata`（`trainingPackId`/`packId`、`packName`、`shotIndex`、`totalShots`、`shotName`、`shotDescription`），若携带初始位姿则立即执行服务端重置。
  - `onControlMessage` 新增 `training_shot_reset` 与 `training_state_summary_update` 分发。
  - `getStateSummary` 透出完整 `shotMetadata` 供 `CSR_STATE_BROADCAST` 与 `stateSummary` 同步。
- **Worker / Host 通道打通**:
  - `src/network/protocol/HostWorkerProtocol.ts`: 新增 `ACTIVITY_CONTROL`。
  - `src/network/server/RoomActivityOrchestrator.ts`: 新增 `handleControlMessage()` 委派当前激活的 `ActivityKernel.onControlMessage`。
  - `src/network/server/workerActivityAndInputRouter.ts`: 新增 `MSG.ACTIVITY_CONTROL` 路由并回传最新 `csr` 摘要。
  - `src/network/server/AuthoritativeServerHost.ts`: 新增 `sendActivityControl()` 与 `resetTrainingShot()`。
- **Bridge & Client 无本地文件渲染同步**:
  - `src/ui-core/stores/appStoreTypes.ts`、`src/game/runtimeDomainPortTypes.ts`、`src/ui-system/core/runtimeOnlineBridgeHandlers.ts`: 扩展 `onlineSession.trainingPackSummary` 与 `bridge.onResetTrainingShot`。
  - `src/game/runtime/OnlineMatchControlBridge.ts`: 实现 `resetTrainingShot()` 同步服务端、更新本地 Store 并广播 `ROOM_ACTIVITY_SWITCH`。
  - `src/game/runtime/onlineSessionEnvironmentPipeline.ts` (`applyClientRoomActivitySwitch`): Client 保持轻量 CSR 接收壳，解析 `trainingPackSummary` 并按 Host 开关同步 `reconciler.setClientInputEnabled(...)`，无需本地 `.RCDPack` 文件。

### 2. Room Match 自动拉入已建连 P2P Peer
- **`src/network/server/activities/ActivityKernel.ts` (`MatchActivityKernel` + `autoPullConnectedPeersForMatch`)**:
  - 在 `MatchActivityKernel.onMount` 时自动收集 `roster.players`、`roster.spectators`、`sessionManager.sessions` 及 `stateSummary.connectedPeers` 中的非 Bot 人类 Peer。
  - 自动为尚未分配槽位的 Peer 分配玩家槽位（或从 `spectator` 切换为 `player`），并在蓝队（0）/橙队（1）之间均衡分配（若多人集中在同一队则自动交替分队）。
  - 自动将所有已连接人类 Peer 标记为 `ready = true` 与 `arenaLoaded = true`，随后直接触发 `startMatch(duration)` 进入 `KICKOFF`，无需重新建房、二次邀请或等待手动 `arena_loaded_ack`。
- **`src/network/server/workerActivityAndInputRouter.ts` & `src/game/runtime/OnlineMatchControlBridge.ts`**:
  - 切换 `activityId: 'match'` 时自动将已连接 P2P Peer 传入 `stateSummary` 并同步标记 `HostSessionController.recordPeerReadyState(peerId, { ready: true, arenaLoaded: true })`。
  - 默认自动开放 `clientInputEnabled: true`（服务端 `dataHandler` + 广播 `ROOM_ACTIVITY_SWITCH` / `GlobalWorkspaceBroadcast`），Client 侧 `applyClientRoomActivitySwitch` 在 `match` 下保持轻量 CSR 壳（不误触发本地 `BotMatchMode`）并解锁输入。
- **`src/ui-core/panels/dev/DevRoomSyncFloatingWindow.tsx`**:
  - 新增 `Room Activity Mode` 快捷切换栏（`Freeplay` / `Training` / `Replay` / `Match`）及训练模式下的 `Reset Shot` 按钮。

---

## 三、自动化验收结果

1. **`pnpm run check`** (`radar` + `tsc --noEmit`): **PASS (GREEN)** — `1415/1415` TS 文件，0 循环依赖，0 越层导入，所有文件体积在硬上限（800L/18KB）内。
2. **`pnpm run radar:audit`**: **PASS (GREEN)** — 无违规耦合。
3. **`pnpm run test:multiplayer-suite`**: **14/14 PASS**
   - 新增 **Test 14**: `14. Custom Training Pack (training_shot_reset + Metadata Sync) & Room Match Auto-Pull`，覆盖：
     - `multiplayer_training` 切换 + 无本地 `.RCDPack` 的 `trainingPackSummary` 同步；
     - `ACTIVITY_CONTROL` (`training_shot_reset`) 重置服务端球/车位姿、`resetGoalLatch`、Boost Pads 与 `shotResetSeq`；
     - Host 开关 `clientInputEnabled` 对 Client 输入的门控与解锁；
     - 已建连 P2P Peer（含 CSR `spectator` 与仅在 `sessionManager` 中的 Peer）一键切入 `match` 后自动分至蓝/橙队、自动 `ready/arenaLoaded`、立即进入 `KICKOFF` 并解锁输入，且 Client 不触发 `BotMatchMode`。
