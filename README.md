# RocketCompanion `z/agy/rc/multiplayer-recheck` 完整性检验、测试审计与加固报告

## 一、总体检验结论

基于分支 `origin/z/agy/rc/multiplayer-recheck`（原基线 `cfa729b`，本次已补齐加固并提交 `bf6c5e0`，`verified: true`），对报告中列出的 4 类核心问题、额外暴露问题及测试体系进行了全面复核与实测：

1. **4 类核心问题修复完成情况**：
   - **问题 1（Dev Sync Room Freeplay 进球无法触发 Cinematic Goal Replay）**：已在 `e104b50` 修复，**本次补全了 Client 侧观测校验**。`FreeplayGoalAndTouchSubController` 与 `FreeplayMode.skipGoalReplay()` 已区分 `isDevSyncHostBroadcastingMode` 与远程客户端；实测 Host 正常进入 `matchPhase: "replay"`、`isGoalReplayActive: true`，Client 同步收到 `goal` 阶段信号。
   - **问题 2（Replay Watch Together Client 缺失人名、1-6 名单条及积分牌）**：已在 `30ec8e5` 修复。`TelemetryStage._syncEdgeRosterHud()`、`InGameHUD.tsx`、`GoalReplayHudOverlay.tsx` 与 `WorkspaceReplaySyncPayload` 均已打通；实测 3v3 回放双端均渲染 `domScoreboard: true`、完全一致的 `scoreboardText`、以及 `domRosterCards: 6` 与一致的 6 名玩家名条。
   - **问题 3（观战目标 1-6 号位同步与初始对齐）**：`30ec8e5` + `cfa729b` 修复了 `onCameraStateSync` 与 `getSpectateCameraFollowTargetIndex` 绑定，**但审计发现一处遗漏**：`GameCameraSubsystem.resolveFrameCameraMetadata()` 每帧都会用 `ws.hostCameraSync.focusedCarIndex` 覆写 `r.camera.focusedCarIndex`，导致 Client 在 Host 未切换时本地切人会在下一帧被拉回（且目标车尚未可见时会被 `setFocusedCarIndex(0)` 覆盖）。**已在 `bf6c5e0` 修复该遗漏**，并在 `repro:dev-broadcast-v2` 中实测通过（Host 切 2 -> Client 2 -> Client 本地切 4 保持 4 -> Host 切 5 -> Client 跟随 5）。
   - **问题 4（Exhibition 3v3 倒计时 Client 缺失 2 和 1 及音效）**：已在 `e104b50` + `30ec8e5` 修复。`experimentalMultiBotsLifecyclePipeline.ts` 与 `RuntimeKickoffSignalSubController.ts` 已统一为绝对 `targetTick` 契约；实测 Client 完整观测到 `["3", "2", "1", "GO"]` 及 `hostCountdownAudioCount: 3`、`clientCountdownAudioCount: 3`。

2. **报告中提及的“额外暴露问题”核查**：
   - **3v3 回放 `domRosterCards: 10` 问题**：经核查，`InGameHUD` 固定挂载左右各 5 个（共 10 个）`EdgePlayerSlotCard` 槽位，`EdgePlayerSlotCard.tsx` 对 `slot.exists === false` 的空槽位设置 `display: none`。过滤 `getComputedStyle(el).display !== 'none'` 后双端严格为 `6` 张卡片，属预期槽位池设计。
   - **Countdown 音效 Hook 盲区**：`cfa729b` 已将测试 spy 挂载点对齐 `window.__rcE2E?.gameAudio || rt.options?.gameAudio`，实测双端均为 `3` 次倒计时音效触发。

---

## 二、测试系统防篡改审计（`5cec557..HEAD`）

对 `5cec557` 之后的测试文件变更（`30ec8e5`、`cfa729b`）进行了逐行 `git diff` 审计，结论：**未被篡改为假测试**，且整体断言更严格：
1. `scripts/runMultiplayerSyncIssuesRepro.ts`：仅将 `issue-4` 的状态字段名从过时的 `currentFrame`/`roster`/`entries` 对齐到真实 store 结构（`cursorFrame`/`resolvedRoster`/`leftPlayers`+`rightPlayers`），仍直接驱动真实 `TelemetryStage._syncEdgeRosterHud()`。
2. `scripts/runDevBroadcastReproV2.mjs`：
   - 将 Client 窗口改为 `Target.createTarget(..., { newWindow: true })`，消除 Chrome 后台标签页 rAF 节流；
   - 强化了 Bug 2 断言（要求双端 `domRosterCards === 6`、排除占位符名称、且 `host.cardNames` 与 `client.cardNames` 及 `scoreboardText` 完全一致）；
   - 将 Bug 3 Host 切人改为真实 UI 路径 `rv.selectPovSlot(3)`；
   - **本次（`bf6c5e0`）进一步强化**：Bug 1 增加 Client 侧 `goal/replay` 阶段观测校验；Bug 3 增加 Client 本地切人非覆盖校验（切 4 保持 4）+ Host 再切人跟随校验（切 5 跟随 5）。
3. `scripts/runDevSyncReproSuite.mjs`：
   - **已知局限**：Scenario 5~9 在开启第 3 个 SwiftShader WebGL 窗口（Client 2）后仍会触发 180s Watchdog 超时（未完成 Scenario 5~9）；另外 Scenario 3 在 `exp-multi-bots -> replay-viewer` 切换后因缺少 `waitFor` 曾出现一次 `domRosterCards: 0` 竞态（已在 `bf6c5e0` 补充 `waitFor` 并修复 Client 2 窗口隔离，但 3 窗口并发在当前容器下仍需进一步拆分为双窗口或优化启动耗时）。
4. `scripts/runDevRoomSyncE2E.mjs`：Stage 6 仍使用 `enterStateOnlyMode` 注入帧，未加载真实 `.rcreplay` 文件与检查 DOM，实际覆盖由 `repro:dev-broadcast-v2` 承担。

---

## 三、本次提交与验证门禁实测结果

- **提交**：`bf6c5e05226f408ccf516c450ea9c9569dafa282`（`verified: true`，`antigravity-fine-grained-token[bot]`）
- **修改文件**：
  - `src/game/subsystems/GameCameraSubsystem.ts`
  - `scripts/runDevBroadcastReproV2.mjs`
  - `scripts/runDevSyncReproSuite.mjs`

| 验证命令 | 结果 | 说明 |
|---|---|---|
| `pnpm run check` (`radar` + `tsc --noEmit`) | **PASS** | 1425/1425 TS，0 错误，0 架构违规 |
| `pnpm run test:multiplayer-suite` | **PASS (24/24)** | 全部 24 项单元与协议联调测试通过 |
| `pnpm run repro:sync-issues` | **PASS (0/5 reproduced)** | 真实管线 5 项均未复现 |
| `RC_REPLAY_PATH=/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay pnpm run repro:dev-broadcast-v2` | **PASS (reproduced=0/4)** | 使用按真实 RLRC V2 容器生成的 1.1MB 3v3 `.rcreplay` 验证，含本次新增 Client Goal Replay 观测与非覆盖切人校验 |
| `pnpm run test:dev-room-sync-e2e` | **PASS (9/9 stages)** | 73.3s 全阶段通过，`runtime_error=0` |
| `pnpm run repro:dev-sync-bugs` | **部分通过 (S1-S4 PASS, S5 Watchdog)** | S1~S4（含 S3 真实 3v3 回放 DOM）全部通过，S5 启动第 3 个 Chrome WebGL 实例超时 |

---

## 四、建议补充的额外测试路线（后续完善方向）

1. **Coach Handover + 反向 Spectating（Client 控车时 Host 观战）**：验证控制权移交 Client 后，Host 侧视角跟随、HUD 提示与 `authorityEpoch` 栅栏隔离，以及收回控制权后状态无回跳。
2. **Freeplay Goal Replay 跳过投票（Skip Replay）**：验证 Host 与 Client 在 Freeplay Goal Replay 期间按跳过键（`shouldVoteSkipReplay`）时双端同步提前退出回放并重置开球。
3. **Late-Join（中途加入）Replay / Exhibition**：验证房间已处于 Replay 播放中或 Exhibition 倒计时/加时赛中时，新加入的 Client 能立即通过 `ROOM_ACTIVITY_SWITCH` + `GlobalWorkspaceBroadcast` 对齐比分、时钟、Roster 与当前观战目标。
4. **拆分 `runDevSyncReproSuite.mjs` 的 3 窗口并发压力**：将需要 Client 2 的场景拆出独立脚本或复用双窗口，避免 Headless SwiftShader 3 实例并发触发 180s Watchdog。
