# RocketCompanion 联机同步与 Pure CSR 全链路深度复核检验报告

**目标仓库**：`Antigravi-ty/RocketCompanion`  
**目标分支**：`origin/z/agy/rc/multiplayer-recheck`  
**复核基线**：`0719bf9` -> **修复提交**：[`9e889cb`](https://github.com/Antigravi-ty/RocketCompanion/commit/9e889cbc89d6299af8ac50a5157071caf812f69a)（Verified Bot Commit）

---

## 一、总体结论：是否可以直接进入 Real Device Test / Roadwash Test？

**结论**：**修复前存在 5 处实质性缺口（不可直接进真机）；现已全部补齐并完成自动化闭环验证，当前状态可以进入 Real Device / Roadwash Test。**

此前 `0719bf9` 虽然已跑通 `test:multiplayer-suite`（14 项单元测试）和初版 `test:dev-room-sync-e2e`，但在逐项 **Detail Check** 中发现：原 E2E 仅校验了 UI Store 状态字面量（如 `activeRoomActivity`、`spectateMode`），未断言 **Host 与 Client 的真实物理位置收敛**，掩盖了 Client 仍停留在 `lobbyDualWorldMode`、Replay 帧注入类型不匹配、以及逆向状态切换（`Match -> Replay -> Freeplay -> Leave Room`）残留状态等关键问题。

---

## 二、Detail Check 发现的缺口与已落地的修复

### 1. [P0] Pure CSR（`spectate_mirror`）下 Client 未退出 `lobbyDualWorldMode`，导致球/车物理未同步 Host
- **问题根因**：Client 通过 `onJoinOnlineWarmup` 建连时默认进入 `lobbyDualWorldMode = true`，`reconcilerRollbackPipeline` 会持续保留 Client 本地沙盒的球与 0 号车而忽略 Host 快照。原实现仅在 Match `MATCH_STARTED` 信号到来时退出该模式，导致 Pure CSR / Replay / Training 模式下 Client 球位与 Host 完全不一致（复核初测时 Host `ballPos=[0,0,161]` 而 Client `ballPos=[207,13,94]`）。同时，`reconcilerInboundPacketParser` 的 `isStaleLobbyPhysicsInMatch` 会在退出 `lobbyDualWorldMode` 后误拦截带有 `0xc0` 诊断车掩码的 Host 快照。
- **修复内容**：
  - 在 `PredictionReconciler` 与 `reconcilerSessionPipeline` 新增 `spectateMirrorMode` 状态及 `setSpectateMirrorMode()`；
  - 在 `RuntimePostprocessingAndSessionBootstrap.ts`（收到 `followHostWorkspace: true` 广播时）和 `OnlineSessionCoordinator.ts`（收到 `replay` / `multiplayer_training` / `match` 活动切换时）自动调用 `setLobbyDualWorldMode(false)` 并隐藏 Lobby Ghost；
  - 在 `reconcilerInboundPacketParser.ts` 中对 `spectateMirrorMode` 与 `replayPlaybackMode` 放行 `isStaleLobbyPhysicsInMatch` 过滤。

### 2. [P1] `stadiumPreset`（Standard / Throwback）切换未同步至 `AuthoritativeServerWorker` 物理权威端
- **问题根因**：`ArenaEnvironmentTunerBridge.ts` 切换 `stadiumPreset` 时仅重载了主线程的渲染场景与本地预测物理，未通知 `AuthoritativeServerWorker` 重载对应球场碰撞体。
- **修复内容**：
  - 在 `AuthoritativeServerHost.ts` 新增 `setArenaPreset(manifestUrl, arenaPresetIndex)`，在 `workerTrainingAndPhysicsRouter.ts` 新增对应 `setArenaPreset` 指令重载 Worker 内 RocketSim 网格与预设索引，并在 `ArenaEnvironmentTunerBridge.ts` 中联动调用。

### 3. [P1] 复杂逆向工况切换（`Match -> Replay -> Freeplay/Lobby -> Leave Room`）存在陈旧信号抢占与状态残留
- **问题根因**：
  1. Host 从 `Match` 切回 `Replay` 或 `Freeplay` 后，在途/缓冲的 `KICKOFF_WILL_START` 信号仍会触发 `RuntimeKickoffSignalSubController` 将 Client 的 `matchMode` 强行拉回 `'match'`；
  2. Host 从 `Replay` 切回 `Freeplay` 后，在途的 `SYS_CTRL_STATE_ONLY` 信号仍会触发 `RuntimeSignalDomainSubController` 将 Client 的 `replayPlaybackMode` 重新置为 `true` 并冻结物理；
  3. Client 退房时 `resetSpectateWorkspaceOnLeave()` 未将 `spectateMode` 重置回 `'spectate_state_only'`。
- **修复内容**：
  - `OnlineMatchControlBridge.ts` 与 `onlineSessionEnvironmentPipeline.ts` 在切换至非 `match` 活动时主动调用 `resetRuntimeMatchStateToFreeplay` 与 `setMatchMode('freeplay')`；
  - `RuntimeKickoffSignalSubController.ts` 对非 `match` 活动屏蔽陈旧 `KICKOFF_WILL_START`；
  - `RuntimeSignalDomainSubController.ts` 在 `freeplay` / `multiplayer_training` / `match` 活动下屏蔽陈旧 `SYS_CTRL_STATE_ONLY`；
  - `runtimeSpectateWorkspaceBridge.ts` 的 `resetSpectateWorkspaceOnLeave()` 显式调用 `ws.setSpectateMode('spectate_state_only', undefined, applyWorkspaceStateLocally)` 恢复本地浮窗与路由。

### 4. [P1] E2E 测试脚本自身缺陷与覆盖面补全（`scripts/runDevRoomSyncE2E.mjs`）
- **问题根因**：
  - Stage 6 原先向 `enterStateOnlyMode` 传入了 `{ ball, cars }` 普通对象而非 `Float32Array` 紧凑快照，导致服务端静默忽略；且 `cameraSync` 使用了 `{ mode: 'ballcam', targetCarIndex: 0 }` 错误字段；
  - 缺少对 Host/Client 真实 `ballPos` 收敛（距离 `< 50`）的硬断言，也未覆盖 `Match -> Replay -> Freeplay -> Leave Room` 逆向工况。
- **修复内容**：
  - Stage 3（Pure CSR）、Stage 6（Replay Watch Together）、Stage 8（Room Match Auto-Pull）均新增 **Host/Client `ballPos` 真实收敛断言**；
  - Stage 6 改用 `phys.setBallState(...)` + `phys.saveCompactState()` 生成真实 `Float32Array` 快照，并使用 `WorkspaceCameraSyncPayload` 真实字段 `{ followHostCamera: true, ballCam: true, focusedCarIndex: 0 }`；
  - 新增 **Stage 9**：完整验证 `Match -> Replay -> Freeplay (Lobby) -> Leave Room` 逆向切换与本地工作区无损恢复。

---

## 三、工具链与环境补充说明

- 沙盒内初始缺少 `pnpm` 与 `public/` 静态资源子模块（导致首次 E2E 报 `ERR_POV_MICROKERNEL_TAMPERED`）：已安装 `pnpm 12.21.0`，并通过认证初始化 `public`（`RocketCompanionStatics`）子模块后全链路恢复正常。

---

## 四、自动化验收结果（提交 `9e889cb` 后实测）

| 检查项 | 命令 | 结果 | 关键指标 |
| :--- | :--- | :--- | :--- |
| 静态架构雷达 + TS 严格检查 | `pnpm run check` | **PASS** | 1415/1415 TS 文件，0 编译错误，0 架构违规 |
| 架构深度审计 | `pnpm run radar:audit` | **PASS** | Anti-Fake-Split / 边界检查全部通过 |
| 联机单元测试套件 | `pnpm run test:multiplayer-suite` | **PASS** | Test 1 ~ Test 14 全部通过 |
| Linux Headless Chrome 双标签 E2E | `pnpm run test:dev-room-sync-e2e` | **PASS** | 耗时 `96864ms`，Stage 1 ~ Stage 9 全通过；`runtime_error=0`，`poseAnomalies=0`，`stuckInGoal=false` |

**E2E 关键收敛实测数据**（来自 `/tmp/dev-room-sync-e2e-report.json`）：
- `pureCsrSyncAll.ballSyncDist`：`0`（`lobbyDualWorldMode: false`，`spectateMode: "spectate_mirror"`）
- `telestrator`：`hostStrokesAfterClientDraw: 2`，`clientStrokesAfterHostDraw: 1`，双向同步与双端 Clear 一致
- `replayWatchTogether`：`pausedAtTick: 420`，`ballPos: [120, -350, 240]`，误差 `dist: 0`
- `customTrainingPack`：`resetSeqIncremented: true`
- `roomMatchAutoPull`：`clientInputEnabled: true`，`replayPlaybackMode: false`，误差 `dist: 0`
- `reverseTransitionsAndLeave`：`Match -> Replay -> Freeplay -> Leave Room` 全部正常恢复

---

## 五、进入 Real Device / Roadwash Test 前的注意事项

1. **同机双标签 vs 真机跨设备传输差异**：Headless Chrome E2E 在同源双标签下会自动激活 `BroadcastChannel` 极速通道；真机跨设备将走真实 **Signaling + WebRTC DataChannel（`RTCDataChannel`）**。代码层已确认 `TELESTRATOR_SYNC (0x05)`、`GlobalWorkspaceBroadcast`、`ROOM_ACTIVITY_SWITCH (0x21)` 均已接入 `DataChannelTransport` 真实分片/控制信道，但仍建议在 Roadwash Test 重点观察弱网/丢包下的首次状态同步耗时与 WebRTC ICE 建连稳定性。
2. **后续工况拓展建议**：当前 `Match -> Replay -> Freeplay/Lobby -> Leave Room` 基础逆向链路已打通；若后续需要支持「在 Replay 中直接拉起特定 Tick 的 Match 接管（Takeover）」或「带局部进度条拖拽的 Replay 连续推流」，可在 `StandaloneReplayViewerController` 与 `DevRoomSyncFloatingWindow` 上进一步增量接线。
