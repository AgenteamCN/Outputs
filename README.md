# Step 1 完成报告：Pure CSR "Sync All" 房间闭环 + DevRoomSyncFloatingWindow

- **目标分支**：`z/agy/rc/multiplayer-recheck`
- **Verified Bot Commit**：[`91ed3b584f8611c2e3a0dd8313a0c90b4ea6ed41`](https://github.com/Antigravi-ty/RocketCompanion/commit/91ed3b584f8611c2e3a0dd8313a0c90b4ea6ed41) (`signature.isValid: true`)

## 1. 核心改动清单

### 1.1 `GlobalWorkspaceProtocol` & `spectateWorkspaceStore` 扩展与端到端广播
- [GlobalWorkspaceProtocol.ts](src/network/protocol/GlobalWorkspaceProtocol.ts)：在 `GlobalWorkspaceBroadcastPayload` 与 `sanitizeGlobalWorkspaceBroadcast` 中新增 `clientInputEnabled: boolean`、`followHostWorkspace: boolean` 与可选 `arenaPreset?: 'standard' | 'throwback'`。
- [spectateWorkspaceStore.ts](src/ui-core/stores/spectateWorkspaceStore.ts)：新增 `clientInputEnabled`、`followHostWorkspace`、`arenaPreset` 状态与 setter；`applyHostWorkspaceBroadcast` 在收到 `followHostWorkspace: true` 时自动进入 `spectate_mirror`（捕获 `localSnapshotBeforeMirror`），并在关闭镜像或离房时无损恢复本地浮窗与菜单路由。
- [runtimeSpectateWorkspaceBridge.ts](src/ui-system/core/runtimeSpectateWorkspaceBridge.ts)：镜像创建缺失浮窗时按 `PRESET_WINDOWS` 还原真实标题与尺寸，新增 `resetSpectateWorkspaceOnLeave`、`syncClientInputEnabledUi`、`syncArenaPresetUi` 并通过 [runtimeUiPort.ts](src/game/runtimeUiPort.ts) 与 [runtimeEngineUiPort.ts](src/ui-system/core/runtimeEngineUiPort.ts) 暴露给引擎层，保持零边界违规。
- [HostSessionController.ts](src/network/transport/HostSessionController.ts) & [MultiplayerSessionManager.ts](src/network/transport/MultiplayerSessionManager.ts)：新增 `broadcastGlobalWorkspace` 与 `activeGlobalWorkspace` 缓存，新 Peer 连入 `onConnected` 时自动重放当前全局工作区状态。
- [OnlineMatchControlBridge.ts](src/game/runtime/OnlineMatchControlBridge.ts)、[runtimeOnlineBridgeHandlers.ts](src/ui-system/core/runtimeOnlineBridgeHandlers.ts)、[appStoreTypes.ts](src/ui-core/stores/appStoreTypes.ts)：打通 `onBroadcastGlobalWorkspace`、`onSetClientInputEnabled`、`onSwitchRoomActivity` UI -> 引擎桥接。

### 1.2 前后双端 Input 门控（Pure CSR）
- **Client 端**：[PredictionReconciler.ts](src/network/client/PredictionReconciler.ts) 新增 `clientInputEnabled` 与 `setClientInputEnabled(enabled)`；[reconcilerSessionPipeline.ts](src/network/client/reconcilerSessionPipeline.ts) 的 `commitPredictedLocalStep` 在 `clientInputEnabled === false` 时将控制输入强制归零为 `NEUTRAL_CAR_CONTROLS`。
- **Server 端**：[NetworkDataHandler.ts](src/network/server/NetworkDataHandler.ts) 新增 `clientInputEnabled` 门控（保留 `acks`、可靠指令与时间戳处理，但屏蔽非 `host`/非 `bot_` 远端驾驶输入）；[HostWorkerProtocol.ts](src/network/protocol/HostWorkerProtocol.ts)、[AuthoritativeServerHost.ts](src/network/server/AuthoritativeServerHost.ts)、[HostWorkerMessageRouter.ts](src/network/server/HostWorkerMessageRouter.ts) 与新拆分的 [workerActivityAndInputRouter.ts](src/network/server/workerActivityAndInputRouter.ts) 接入 `SET_CLIENT_INPUT_ENABLED` 指令与 `ROOM_ACTIVITY_SWITCH` 联动。

### 1.3 修复 `applyClientRoomActivitySwitch()`
- [onlineSessionEnvironmentPipeline.ts](src/game/runtime/onlineSessionEnvironmentPipeline.ts)：修复 `multiplayer_training` / `replay` / `match` 模式 ID 映射与本地文件/Bot AI 抢占冲突，使 Client 在在线 CSR 房间中保持轻量化 `'freeplay'` CSR 接收壳，仅同步 `activeRoomActivity`、`replayPlaybackMode`、`arenaPreset` 与 `clientInputEnabled`。

### 1.4 `DevRoomSyncFloatingWindow` 与 `Layer3DevUnderDevelopmentRecipe` 入口
- [DevRoomSyncFloatingWindow.tsx](src/ui-core/panels/dev/DevRoomSyncFloatingWindow.tsx) & [devFloatingPresets.ts](src/ui-core/stores/devFloatingPresets.ts)：严格遵循 alpha `2fb9052` 的 `PresetConfig` + `SURFACE_META` + `floatingSurfaceRegistry` 1:1 规范，提供房间状态、Follow Host Workspace、Allow Client Driving Input、Telestrator Permission、Arena Preset 同步与手动广播/停房按钮。
- [Layer3DevUnderDevelopmentRecipe.tsx](src/ui-core/recipes/Layer3DevUnderDevelopmentRecipe.tsx)：新增 **"Developer Sync Room (Pure CSR)"** -> **"Create Developer Sync Room"** 入口，一键配置 Pure CSR 默认策略、打开 `DevRoomSyncFloatingWindow` 并调用 `bridge.onHostOnlineWarmup`。

## 2. 自动化验收结果

- `pnpm run check` (`radar` + `tsc --noEmit`)：**PASS** (`STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified`)
- `pnpm run radar:audit`：**PASS** (`STATUS: GREEN`)
- `pnpm run test:multiplayer-suite`：**PASS**（11/11 全部通过，新增 **Test 11: Pure CSR Sync All, Dual-End Input Gating & Workspace Broadcast** 覆盖广播/迟到重放、Workspace 镜像与离房恢复、Client/Server 双端输入门控与 `applyClientRoomActivitySwitch`）
