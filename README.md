# RocketCompanion Multiplayer / Developer Sync Room 全面审计与修复报告

- **仓库 / 分支**: `Antigravi-ty/RocketCompanion` (`z/agy/rc/multiplayer-recheck`)
- **最新 Verified Bot Commit**: `971ef3d8d4d7bef42b4ca9c05e1c487f335b5e26` (`verified: true`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)
- **前序基线同步**: `dad6dee1f78b4ec99aba1f84c094400e67fbd473`（含 `localTeam` / `activePovCarIndex` / `replayPovCarIndex` 同步、`scripts/runDevSyncExtraRoutesE2E.mjs` 与 `runDevSyncReproSuite.mjs` checkpoint/resume 支持）

---

## 一、原有 4 类问题修复完备性审计与遗漏根因修复

### 1.1 问题 1–4 审查结论
对 `5cec557..dad6dee` 提交链路与运行时代码核查结果：
1. **Freeplay 进球无法触发 Cinematic Goal Replay**：已正确区分 Broadcasting Host 与远程 Pure CSR Client，允许 Broadcasting Host 走完整本地进球检测与 `beginGoalReplay()`，并通过 `GOAL_REPLAY_WILL_START` / `GOAL_REPLAY_START` / `GOAL_REPLAY_END` 同步至 Client。
2. **Replay Watch Together Client 缺失人名、1-6 名单条与上方积分牌**：已将 `isClientSyncedReplay` 提前至 `TelemetryStage._syncEdgeRosterHud()` 守卫入口，并在 `InGameHUD.tsx` / `GoalReplayHudOverlay.tsx` 与 `WorkspaceReplaySyncPayload` 中同步 `blueScore`、`orangeScore`、`remainingSeconds`、`isOvertime` 与 `localTeam`。
3. **1-6 号位观战目标切换与默认视角同步**：
   - 在 `dad6dee` 中已同步初始 `activePovCarIndex` 与 `runtime.replayPovCarIndex`。
   - **本次额外定位并修复了两个隐藏时序/覆盖 Bug（commit `971ef3d`）**：
     1. **`src/game/runtime/signalWorldTransitionPipeline.ts`（`focusLocalPlayerCarInArenaAndCamera`）**：当 Pure CSR Client 处于 `spectateMirrorMode` 时，`KICKOFF_START` / 信号状态转换会调用 `focusLocalPlayerCarInArenaAndCamera`，此前无条件执行 `camera?.setFocusedCarIndex?.(idx)`（默认 `0`），在 3-2-1 开球倒计时结束瞬间把 Host 已同步的 `focusedCarIndex`（如 2 号车）重置回 `0`。现改为在 `spectateMirrorMode` 下优先保留 `getSpectateCameraFollowTargetIndex()`，不覆盖观战目标。
     2. **`src/game/modes/shared/GoalReplaySessionCoordinator.ts`（`endGoalReplaySession`）**：在模式切换（如 `BotMatchMode.onLeave`）触发 `endGoalReplaySession(..., { restoreLocalCarFocus: true })` 时，同样会无条件把 `scenePort.camera?.setFocusedCarIndex?.(localCarIdx)` 重置回 `0`，导致 `GameCameraSubsystem.ts` 中 `localCamFocusedIdx`（`0`）锁死覆盖 `spectateFollowIdx`（`2`）。现改为在 `spectateMirrorMode` 下保留 `getSpectateCameraFollowTargetIndex()`。
4. **Exhibition 3-2-1 倒计时同步**：Host 与 Client 均稳定经历 `3 -> 2 -> 1 -> GO` 倒计时状态与音频触发。

---

## 二、测试体系防篡改审计与新增覆盖路线

1. **防篡改审计**：
   - 核查 `5cec557..dad6dee` 改动：`5cec557` 移除了 `runMultiplayerSyncIssuesRepro.ts` 中绕过真实逻辑的 mock 检查，改为验证真实通道、HUD 状态、音频计数与 DOM 渲染；`runDevBroadcastReproV2.mjs` 与 `runDevSyncReproSuite.mjs` 均在真实 Headless Chrome + WebGL (SwiftShader) + WASM 下运行，无伪造通过断言。
2. **新增核心 E2E 覆盖路线（`scripts/runDevSyncExtraRoutesE2E.mjs`）**：
   - `extra-1-freeplay-goal-replay-skip`：Freeplay Goal Replay Skip 双端同步退出（Host 触发 `skipGoalReplay()` -> Host 与 Client 同步回到 `playing`）。
   - `extra-2-coach-handover-epoch-reclaim`：Coach Handover + 反向观战 + `authorityEpoch` 递增栅栏与 Host 回收控制权（`epochBefore=1 -> epochHandover=2 -> epochReclaim=3`，Client 实际驾驶位移 `distClientDrove=454`）。
   - `extra-3-late-join-replay-watch-together`：Late-Join Client 在 Host 已加载并播放 3v3 `.rcreplay` 后加入房间，立即对齐 Replay 积分牌（`001:251BO5OVERTIME`）、6 人名条（`1PlayerYOU0x .. 6Nexto #6YOU0x`）及 `focusedCarIndex=2`。

---

## 三、全量验证结果（所有改动提交后实测）

1. **架构雷达 & TypeScript 严格检查**（`pnpm run check`）：
   - **PASS**（1425/1425 TS 文件，0 循环依赖，0 越层引用，`tsc --noEmit` 零错误）。
2. **Multiplayer 单元测试矩阵**（`pnpm run test:multiplayer-suite`）：
   - **24/24 PASS**。
3. **Sync Issues 真实链路复现套件**（`pnpm run repro:sync-issues`）：
   - **0/5 reproduced**（全部 `NOT_REPRODUCED`）。
4. **Dev Broadcast Repro V2（真实 3v3 `.rcreplay` + Headless Chrome）**（`pnpm run repro:dev-broadcast-v2`）：
   - **4/4 PASS_FIXED**（`reproduced=0/4`：`bug-1-freeplay-goal-replay`、`bug-2-replay-ui-and-roster`、`bug-3-camera-player-switch-sync`、`bug-4-exhibition-countdown-321`）。
5. **Dev Room Sync E2E（9 阶段完整生命周期）**（`pnpm run test:dev-room-sync-e2e`）：
   - **ALL E2E CHECKS PASSED**（68.4s，`runtime_error=0`，`poseAnomalies=0`，`stuckInGoal=false`）。
6. **Extra Routes E2E**（`node scripts/runDevSyncExtraRoutesE2E.mjs`）：
   - **3/3 PASS**（Freeplay Goal Replay Skip、Coach Handover Epoch Reclaim、Late-Join Replay Watch Together）。
