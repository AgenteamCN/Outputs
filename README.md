# Dynamic Training Pack 修复报告（Slot 串号 / 名字显示 / 非对称阵型 / 3s Timeout 音效）

- **仓库**: `Antigravi-ty/RocketCompanion`
- **基准 Commit**: `f70fbf369eb1c4a3d72013bf2f30925908acb9e7`
- **工作分支**: `z/agy/rc/fix-replay-slot-and-timeout`（未提交 PR，待验证）
- **Verified Commit**: `96dcecb7bfcf50c574c4ed9ca9cb0c5bebdd91a7` (`verified: true`)

---

## 1. 根因定位：为什么六个 Slot 会全部串号

使用实际提供的 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz` 与 `Captured From Replay.RCDPack` 排查后，确认原因如下：

1. **Replay 导出顺序与 `slotIndex` 不一致**：
   - 在 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz` 中，`Player` 在 Team 1（`carIndex: 3`）。`ReplayViewerRosterResolver.ts` 将本方队伍（`localTeam`）排在前面，导致 `resolvedRoster` 顺序为 `[3, 4, 5, 0, 1, 2]`。
   - `DynamicTrainingSliceExtractor.ts` 按 `roster` 顺序写入 `frame.cars`，于是 `Captured From Replay.RCDPack` 的每一帧里 `cars[0].slotIndex === 3`、`cars[3].slotIndex === 0`。
2. **运行时按数组下标 `cars[slot]` 错位读取**：
   - 以下文件全部直接用 `frame.cars[slot]`（数组下标）而不是匹配 `car.slotIndex === slot`：
     - `RLRCInterveneTargetFrameApplier.ts` (`applyTargetFrameToPhysicsAndArena`)
     - `RLRCInterveneNonPovSyncPipeline.ts` (`stepNonPovCarsBeforePhysics`、`syncFollowStateNonPovAfterStep`)
     - `RLRCInterveneFrameStateBuilder.ts` (`packRLRCFrameToGameStateBuffer`)
     - `RLRCIntervenePrerollHistoryInjector.ts` (`resolveValidIntervenePovSlotIndex`、`buildFrameAcknowledgedControls`、`detectBallTouchIncidentBetweenFrames`)
   - 结果导致 `slot 0` 读了 `slotIndex 3` 的车、`slot 3` 读了 `slotIndex 0` 的车，切换任意 slot 都会位置与队伍错乱；在 1v1 / 2v2 / 非对称对局中更会出现越界或空槽错位。
3. **POV 切换未互斥**：
   - `dynamicTrainingPackStore.ts` 的 `updateShotPerCarTakeover` 在将某辆车设为 `'player'` 时，未把原 `'player'` 槽位降级为 `'as-previous'`，可能导致多辆车同时处于 `'player'`。

---

## 2. 本次修改内容

1. **按 `car.slotIndex` 精确索引（兼容历史已导出的 `.RCDPack` 与非连续槽位）**
   - 在 `src/replay/rlrc/runtime/RLRCInterveneFrameStateBuilder.ts` 新增 `findRLRCCarBySlot(cars, slotIndex)`，并在以下模块全部替换为按 `slotIndex` 查找：
     - `src/replay/rlrc/runtime/RLRCInterveneFrameStateBuilder.ts`
     - `src/replay/rlrc/runtime/RLRCInterveneTargetFrameApplier.ts`
     - `src/replay/rlrc/runtime/RLRCInterveneNonPovSyncPipeline.ts`
     - `src/replay/rlrc/runtime/RLRCIntervenePrerollHistoryInjector.ts`（含 `resolveValidIntervenePovSlotIndex` 与 `detectBallTouchIncidentBetweenFrames` 返回真实 `car.slotIndex`）
   - 在 `src/replay/DynamicTrainingSliceExtractor.ts` 中将新提取的 `compactRoster` 和 `frame.cars` 按 `slotIndex` 升序规范化输出，同时保留仅实际参战车辆（支持 1v1、2v2、2v1、1v0 等任意阵型）。
2. **显示 1-based Slot 编号、队伍色点与玩家名称**
   - `src/ui-core/config/dynamicTrainingPackTypes.ts`：`DynamicTrainingStateSessionSummary` 增加可选 `roster` 字段并在 `summarizeDynamicStateSession` 中填充。
   - `src/ui-core/panels/dev/dynamic-training/DynamicTrainingShotEditorCard.tsx`：优先使用 `sessionSummary.roster` 渲染实际存在的车辆槽位（不再依赖固定 6 槽或猜号），展示 `#1..#6` 编号 + 蓝/橙队伍圆点 + `userName`。
   - `src/ui-core/stores/dynamicTrainingPackStore.ts`：`updateShotPerCarTakeover` 保证单人训练中仅有一个 `'player'` POV 槽位，切换时自动将旧 `'player'` 槽位切回 `'as-previous'`。
   - `src/game/modes/dynamic-training/DynamicTrainingMode.ts`：`startShot` 解析有效 `player` POV 槽位并在 `enterInterveneAtFrame` 后同步 `runtime.playerCarIndex` 与 `arena.playerCarIndex`。
3. **Timeout 最后 3 秒倒计时音效**
   - `src/game/modes/dynamic-training/DynamicTrainingMode.ts`：在 `onAfterPhysicsTick` 中参考 `StaticTrainingSimulationController` 的模式，当 `remainingSeconds` 进入 `3s / 2s / 1s` 时触发 `training_countdown` 音效，超时结算时触发 `training_ball_expire` 音效（并在 `onEnter` / `startShot` / `stopShot` 时重置计数状态）。
4. **Headless / 非 UI 挂载保护**
   - `src/replay/rlrc/runtime/RLRCInterveneEnvPortBuilder.ts`：对 `useReplayToRcsImporterStore.getState()?.updateLiveTelemetry?.(...)` 增加可选链保护。

---

## 3. 测试与验证结果

1. **格式与静态检查**：
   - `pnpm exec prettier --check`（本次修改的 10 个文件）：**PASS**
   - `pnpm run check`（`radar` + `tsc --noEmit`，1364/1364 TS 文件）：**PASS**
   - `pnpm run build`：Exit 0，构建通过
2. **Ducky / 真实 Replay & RCDPack 自动化测试（Node + esbuild 打包实测）**：
   - **测试 1（真实 `Captured From Replay.RCDPack`）**：对乱序 `cars`（`[3,4,5,0,1,2]`）遍历切换 POV `0..5`，验证 `packRLRCFrameToGameStateBuffer`、`setCarState` 位置、`addCarAtSlot` 队伍、`playerCarIndex` 全部 1:1 精确对应，无串号。
   - **测试 2（真实 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz`）**：解析 20107 帧并在 23.2s 提取 session，验证 `roster` 按 `0..5` 排序且完整保留 `Nexto #4..#6 / Player / Nexto #2..#3` 名称与队伍。
   - **测试 3（1v1 / 2v2 / 非对称 2v1 / 单人 1v0）**：验证非连续 `slotIndex`（如 `[0,3]`、`[1,3,5]`、`[2]`）下仅激活真实车辆、其余槽位正确停用与隐藏，且无效 `povSlotIndex` 能回退到有效 `car.slotIndex` 而非数组下标。
   - **测试 4（Timeout 3 秒倒计时音效）**：5s 超时实测触发 3 次 `training_countdown`（3s/2s/1s）与 1 次 `training_ball_expire`。

---

## 4. 关于 Multiplayer 期间“谁是 Player”的建议

- **建议等后续联机房间/玩家槽位基础设施完成后再接 Multiplayer 的多人 `player` 选择，本次先把单人 Concept 彻底调通即可**：
  - 当前单人模式的 `DynamicTrainingShotRef` 已具备 `perCarTakeover`（每车规则）与 `povSlotIndex` 模型；但目前引擎输入管线（`DynamicTrainingMode.readPlayerControls()` / `runtime.playerCarIndex`）和网络同步层默认只有一个本地 `player`。
  - 后续接 Multiplayer 时，只需在房间分配好每个连接玩家对应的 `slotIndex`（例如 `slot -> peerId` 映射）后，将 `perCarTakeover` 中多个 `'player'` 槽位绑定到对应联机玩家即可，无需推翻现有 `.RCDPack` 格式和 `slotIndex` 逻辑。
