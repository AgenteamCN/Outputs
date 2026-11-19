# RocketCompanion 联机与回放同步完整性检验、遗漏修复与测试接力报告

**分支**：`z/agy/rc/multiplayer-recheck`
**最新验证提交**：`dad6dee`（Verified Bot Commit，`signature.state = VALID`）

---

## 一、整体完成情况与遗漏点排查结论

对 `5cec557` 至 `bf6c5e0` 之间的修复提交进行代码审计与真实 Headless Chrome + WebGL + WASM 复现验证后，结论如下：

1. **先前已完成的 4 大核心问题**：
   - **问题 1（Dev Sync Freeplay 进球触发 Cinematic Goal Replay）**：`FreeplayGoalAndTouchSubController.ts` 已正确区分 `isBroadcastingHost` 与远程观战客户端，并广播 `GOAL_REPLAY_WILL_START` / `GOAL_REPLAY_START` / `GOAL_REPLAY_END`。
     - *发现并修复的测试时序遗漏*：`runDevBroadcastReproV2.mjs` 原先使用固定 `1200ms` `setTimeout` 等待，在 SwiftShader CPU 渲染波动下偶发提前采样（`goalResetTicks: 15`），已改为 `waitFor` 轮询（最长 6s）直到 Host 进入 Goal Replay 且 Client 收到 `goal`/`replay` 状态。
   - **问题 2（Replay Watch Together 客户端 HUD / 积分牌 / 名单条）**：`TelemetryStage.ts`、`InGameHUD.tsx`、`GoalReplayHudOverlay.tsx` 已打通，**但实际运行真实 3v3 回放（`rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`）时暴露出 1 个关键遗漏**：
     - **遗漏 Bug A（Client 1-3 / 4-6 队伍左右互换）**：该 3v3 回放中本地玩家 `Player` 在 Orange 队（`localTeam = 1`），Host 侧 `EdgePlayerSlotCard` 左侧 1-3 显示 `Player, Nexto #2, Nexto #3`、右侧 4-6 显示 `Nexto #4, Nexto #5, Nexto #6`。但 `WorkspaceReplaySyncPayload` 未同步 `localTeam`，导致 Client 端 `detectedLocalTeam` 默认回退为 `0`，1-3 与 4-6 两侧名单条完全左右互换。
     - **修复**：在 `GlobalWorkspaceProtocol.ts`、`replayViewerStore.ts`、`replayOnlineSyncHelper.ts`、`devSyncChannelHandlers.ts`、`onlineSessionEnvironmentPipeline.ts` 与 `TelemetryStage.ts` 中补齐 `localTeam` 同步链路，双端 1-6 `cardNames` 现已完全一致。
   - **问题 3（1-6 号位观战目标切换与初始对齐）**：真实浏览器复现时额外暴露出 **3 处隐式状态覆盖遗漏**：
     - **遗漏 Bug B（首次进入 Replay 初始聚焦错位）**：`StandaloneReplayViewerController._initializeLoadedReplaySession()` 调用 `camera.resetView()` 后直接发送 `ROOM_ACTIVITY_SWITCH`，此时尚未将 `rosterSubController.activePovCarIndex`（如 `3`）应用到 `camera.focusedCarIndex` 与 `runtime.replayPovCarIndex`，且 `onlineSessionEnvironmentPipeline.ts` 收到 `ROOM_ACTIVITY_SWITCH` 时也未同步更新 `ctx.camera`，导致 Host 看 3 号车而 Client 初始停留在 0 号车。
     - **遗漏 Bug C（Host 切换 1-6 号位后被 `replayPovCarIndex` 拉回）**：`ReplayViewerRosterSubController.selectPovSlot()` 更新了 `playerCarIndex` 和 `camera.setFocusedCarIndex`，但漏更新 `runtime.replayPovCarIndex`，导致下一帧 `GameCameraSubsystem` 仍按旧 `replayPovCarIndex` 将 Host 视角拉回原目标。
     - **遗漏 Bug D（Late-Join 客户端被握手重置视角）**：`OnlineLobbyDualWorldController.prepareLobbyWorld()` 与 `OnlineMatchReconcilerBinder` 在 Dev Sync Client 握手完成时会无条件把 `camera.focusedCarIndex` 重置为 `localCarIndex` 或 `0`，覆盖掉已同步的 Host 观战目标。
     - **修复**：新增 `ReplayViewerRosterSubController.applyInitialPovToRuntime()` 并在 `selectPovSlot()` 同步更新 `runtime.replayPovCarIndex`；在 `onlineSessionEnvironmentPipeline.ts` 收到 `ROOM_ACTIVITY_SWITCH` 时立即同步 `ctx.camera`；在 `OnlineLobbyDualWorldController.ts` 与 `OnlineMatchReconcilerBinder.ts` 中保护 Dev Sync Client 不被 `localCarIndex` / `0` 覆盖，优先使用 `getSpectateCameraFollowTargetIndex() ?? 0`。
   - **问题 4（Exhibition 3-2-1 倒计时）**：`experimentalMultiBotsLifecyclePipeline.ts` 与 `RuntimeKickoffSignalSubController.ts` 对齐 `targetTick` 后，Client 端稳定观测到 `[3, 2, 1]` 倒计时状态、DOM 文字与 3 次倒计时音效。

---

## 二、测试系统防篡改审计与“接力测试（Checkpoint / Resume）”机制

1. **测试系统防篡改核查**：
   - `scripts/runMultiplayerSyncIssuesRepro.ts`（`repro:sync-issues`）确认未回退为 Mock 注入，仍直接驱动 `FreeplayGoalAndTouchSubController`、`TelemetryStage._syncEdgeRosterHud()` 与 `experimentalMultiBotsLifecyclePipeline` + `RuntimeKickoffSignalSubController` 真实管线。
   - `scripts/runDevBroadcastReproV2.mjs` 中 `bug-3` 原先在同一 `eval` 中同时调用 `rv.selectPovSlot(3)` 与 `camera.setFocusedCarIndex(2)`（两者对 `localTeam=1` 的 3v3 回放语义冲突，slot 3 对应 car 5 而非 car 2），已修正为纯粹调用真实热键入口 `rv.selectPovSlot(3)` 与 `rv.selectPovSlot(6)` 并校验双端 `focusedCarIndex` 严格对齐。
2. **S5 超时定位与断点接力机制（`runDevSyncReproSuite.mjs`）**：
   - **根因**：`runDevSyncReproSuite.mjs` 原 Watchdog 为 180s，且 S5 启动第 3 个 Chrome `newWindow: true` WebGL + SwiftShader + WASM 实例时并发抢占 CPU，且 `CdpClient.send()` 无单次调用超时，一旦 `Runtime.evaluate` 卡死就会阻塞 `waitFor` 直至 Watchdog 强杀。
   - **改造**：
     - 将 Watchdog 调整为 `260_000ms`（4 分 20 秒），为 `CdpClient.send()` 增加 60s 单次超时保护，并将 `Client2` 改为同窗标签页（`newWindow: false`）以降低 3 窗口并发 SwiftShader 压力。
     - 新增 **流式断点记录与接力参数**：每次 `recordScenario()` 立即流式写入 `/tmp/dev-sync-repro-checkpoint.json`（可通过 `RC_REPRO_CHECKPOINT_FILE` 指定）；支持 `--resume` / `RC_REPRO_RESUME=1` 自动从最后一个已完成场景接力续跑，或通过 `--from=<N>` / `RC_REPRO_FROM=<N>` 从指定场景恢复房间与模式状态接力执行（已实测通过 `--resume` / `--from=5` 接力完成 S5 验证）。
3. **新增额外测试路线（`scripts/runDevSyncExtraRoutesE2E.mjs` / `pnpm run repro:dev-extra-routes`）**：
   - 补齐报告中建议的 3 条核心未覆盖路线（采用双窗口轮替控制 SwiftShader 负载）：
     1. `extra-1-freeplay-goal-replay-skip`：Freeplay 进球进入 Goal Replay 后 Host 执行 `skipGoalReplay()`，验证双端同步退出回放并回到 `playing`。
     2. `extra-2-coach-handover-epoch-reclaim`：Coach Handover 开启 `clientInputEnabled=true` -> `authorityEpoch` 递增 -> Client 控车、Host 反向观战 -> Host 收回控制权 `clientInputEnabled=false` -> `authorityEpoch` 再次递增且无状态回滚。
     3. `extra-3-late-join-replay-watch-together`：Host 在 3v3 回放播放中并切换 POV 后，新 Client2 中途加入房间（Late-Join），验证其立即对齐 `matchMode`、`activeRoomActivity`、顶部积分牌、6 名玩家 1-6 顺序与当前观战 `focusedCarIndex`。

---

## 三、测试与构建验证结果汇总

| 检查 / 测试命令 | 结果 | 说明 |
|---|---|---|
| `pnpm run check` | **PASS (GREEN)** | 1425/1425 TS 文件，`tsc --noEmit` 0 错误，Radar 硬边界与 18KB 体积上限全部通过 |
| `pnpm run test:multiplayer-suite` | **PASS (24/24)** | 传输层、协议编解码、Coach Handover、CameraSync、Pure CSR 全通过 |
| `pnpm run repro:sync-issues` | **PASS (0/5 reproduced)** | 真实业务管线不再复现 5 类同步缺陷 |
| `pnpm run test:dev-room-sync-e2e` | **PASS (9/9 stages, 71.1s)** | 真实 Headless Chrome + WebGL + WASM 全流程通过 |
| `RC_REPLAY_PATH=... pnpm run repro:dev-extra-routes` | **PASS (3/3 routes)** | Freeplay Skip、Coach Handover Epoch Reclaim、Late-Join Replay 全部通过（注：最后清理 `/tmp/chrome-...` 临时目录锁文件竞态已加 `try/catch` 保护） |
| `RC_REPLAY_PATH=... pnpm run repro:dev-broadcast-v2` | **bug-1/2/4 PASS_FIXED；bug-3 修复 `replayPovCarIndex` 同步** | 在修复 `localTeam` 与 `OnlineMatchReconcilerBinder` 后，bug-1/2/4 均已通过且 `initialHostFocused === initialClientFocused === 3`；随后针对 `hostAfterSwitch6: 3` 问题补充了 `ReplayViewerRosterSubController.selectPovSlot()` 中 `runtime.replayPovCarIndex` 同步（注：重跑时因 Pixeldrain 对同 IP 触发热链限流返回 403 无法再次拉取 `.rcreplay` 复跑最终一轮，代码层面已完成 `check` + `test:multiplayer-suite` + `repro:sync-issues` + `test:dev-room-sync-e2e` 验证） |
