# Dynamic Training Pack 修复与验证报告

**分支**：`z/agy/rc/fix-replay-slot-takeover`  
**基准 Commit**：`f70fbf369eb1c4a3d72013bf2f30925908acb9e7`  
**最新 Verified Commit**：`3e8ce00a6cf2e51970e165c0998e34888e3a54e9`（尚未提交 PR，等待你本地验证）

---

## 1. 两个 Commit 对比：`0401e7a` vs `f70fbf3`

| 维度 | `0401e7a` | `f70fbf3`（本分支基准） | 本次吸收与决策 |
| :--- | :--- | :--- | :--- |
| **120Hz 双步进修复** | 在 `SimulationStage.ts` 特判 `dynamic-training-pack` 并新增 `onStepPhysicsTick()` | 复用标准 `GameMode` 生命周期 `onBeforePhysicsTick()` + `onAfterPhysicsTick()` | 保留 `f70fbf3`，无额外分支特判，耦合更低 |
| **BoostPad 暂停冻结** | 直接移除 `isBufferFrozen` 逻辑 | 保留 `safeDt === 0` 与 `isBufferFrozen` 判断，暂停时精确冻结 | 保留 `f70fbf3` |
| **Slot 串号修复** | 在 `DynamicTrainingSliceExtractor` 中对 `cars` 与 `roster` 按 `slotIndex` 排序 | 未排序，导致 `[3,4,5,0,1,2]` 顺序直接按数组下标读取 | **吸收并升级**：不仅排序，还用 `findCarBySlot` + 稀疏槽位归一化彻底根治 |
| **Bridge & Importer** | 缺少 `runtimeDynamicTrainingBridge.ts` 与 `replayToRcsImporterStore.ts` 完整接线 | 完整保留并支持两阶段截取与桥接 | 保留 `f70fbf3` |

---

## 2. 根因定位与修复明细

### 2.1 Slot 串号 & 稀疏/非对称阵容（1v1 / 2v2 / 非对称）
- **根因**：在 `Captured_From_Replay.RCDPack` 中，`session.roster` 与 `frame.cars` 的顺序为 `[3, 4, 5, 0, 1, 2]`（先橙队后蓝队）。`DynamicTrainingSliceExtractor.ts` 未按 `slotIndex` 排序或归一化，同时 `RLRCInterveneTargetFrameApplier.ts`、`RLRCInterveneNonPovSyncPipeline.ts`、`RLRCIntervenePrerollHistoryInjector.ts`、`RLRCInterveneFrameStateBuilder.ts` 全部直接使用 `frame.cars[slot]` 按数组下标取值，导致 0-5 号位与物理槽位错位；在 1v1 / 2v2 / 非对称对局中，`cars[3]` 甚至会越界为 `undefined`。
- **修复**：
  - `src/replay/DynamicTrainingSliceExtractor.ts`：在提取和回填（`hydrateRLRCFramesFromDynamicSession`）时，统一对 `roster` 与 `cars` 按 `slotIndex` 升序排列，并将每帧 `cars` 归一化为 `0..5` 槽位索引数组（缺失槽位自动填充 `entityActive: false` 占位，兼容已有历史 `.RCDPack` 文件）。
  - `src/replay/rlrc/runtime/RLRCInterveneFrameStateBuilder.ts`：新增并导出 `findCarBySlot(cars, slot)`，并在上述 4 个 RLRC 接管运行时文件中全部替换 `frame.cars[slot]`；同时修复 `resolveValidIntervenePovSlotIndex` 与 `detectBallTouchIncidentBetweenFrames` 返回真实 `car.slotIndex` 而非数组下标。
  - `src/game/modes/dynamic-training/DynamicTrainingMode.ts`：新增 `resolveEffectivePovSlot()`，当稀疏对局中 `povSlotIndex` 不存在时安全回退至有效活跃槽位。

### 2.2 全员设为 Player 仍不接收用户输入
- **根因**：`DynamicTrainingMode.readPlayerControls()` 调用了不存在的 `r?.input?.getPlayerControls?.()`，导致每帧永远返回 `DEFAULT_NEUTRAL_CAR_CONTROLS`。实际上 `SimulationStage.ts` 在每帧物理步前通过 `this.runtime.pollInputs()` 将输入写入 `this.runtime.playerControls`。
- **修复**：`DynamicTrainingMode.readPlayerControls()` 优先读取 `this.runtime.playerControls`；同时为 `createRLRCInterveneEnvPort` 增加独立 `onPhaseChanged` 回调，避免污染 `useReplayToRcsImporterStore`。
- **关于 Multiplayer 规划**：本次按你要求保持单机 POV 接管概念跑通（非 POV 的 `player` 槽位暂按 `follow-state` 处理），待后续多人房间与远端输入同步基础设施就绪后，再将非本地 `player` 槽位接入远端输入流即可。

### 2.3 3-2-1 倒计时缺少 "TAKE OVER IN" UI
- **根因**：`GoalReplayHudOverlay.tsx` 中的 `useGoalReplayAndTakeoverHudState` 与 `GoalReplayHudOverlay` 仅订阅了 `ReplayViewer`、`ReplayToRcsImporter` 和 `Rewinder`，未订阅 `useDynamicTrainingPackStore` 的 `playPhase === 'preroll-countdown'`。
- **修复**：在 `src/ui-core/hud/GoalReplayHudOverlay.tsx` 新增 `useDynamicTrainingPrerollHud()`，将 Dynamic Training 的倒计时状态、倒计时秒数与 POV 队伍颜色（Blue / Orange）接入 `TakeoverVignetteOverlay` 与 `TakeoverCountdownHUD`。

### 2.4 Timeout 最后 3 秒倒计时音效
- **修复**：在 `DynamicTrainingMode.onAfterPhysicsTick()` 中监听 `judge.remainingSeconds` 进入 `3 / 2 / 1` 秒区间时触发 `training_countdown` 音效；超时触发 `training_ball_expire`，进球触发 `sfx_goal_poof`。

### 2.5 Slot 显示名称、编号（#1~#6）与队伍标识
- **修复**：
  - `src/ui-core/config/dynamicTrainingPackTypes.ts`：在 `DynamicTrainingStateSessionSummary` 与 `summarizeDynamicStateSession()` 中携带按 `slotIndex` 排序的 `roster`。
  - `src/ui-core/panels/dev/dynamic-training/DynamicTrainingShotEditorCard.tsx`：基于 `sessionSummary.roster` 渲染实际存在的车辆（1v1/2v2/非对称只显示对应车辆），每行显示队伍色点、`#1~#6` 编号（悬停提示 `Slot 0~5`）、玩家名 `userName` 以及当前 `POV` 徽标。

---

## 3. 验证结果

1. **架构与类型检查（`pnpm check`）**：
   - `radar`（1364/1364 TS，0 循环依赖，0 越级引用，体积合规）：**PASS**
   - `tsc --noEmit`：**PASS**
   - `pnpm build`（Vite 生产构建）：**PASS**
2. **自动化实测（已使用真实 `Captured_From_Replay.RCDPack` + 1v1/2v2/1v3 稀疏对局 + 按键接管模拟）**：
   - `[PASS] 1. Captured_From_Replay.RCDPack slot hydration & ordering verified`（`[3,4,5,0,1,2]` 乱序数据正确归一化为 `0..5`，Slot 3 精确对应 `Player` 且 boost=69.17，Slot 0 对应 `Nexto #4` 且 boost=100）
   - `[PASS] 2. Sparse/asymmetric Ducky tests (1v1, 2v2, 1v3) verified`（仅激活对应上场槽位，无越界与串号）
   - `[PASS] 3. All-player takeover, 3s countdown, jump/front-flip input, and 3s timeout audio verified!`（全员 `player` + 3s 倒计时正常进入 `intervening`，`throttle`/`steer`/`jump`（Single Jump）与 `pitch: -1 + jump`（Front Flip）均精确送达 POV 槽位，最后 3 秒触发 3 次 `training_countdown` 与超时 `training_ball_expire`）
