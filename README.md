# Track C (`abeab8fa` / Dynamic Training Pack) 深度排查、修复与合并评估报告

**工作分支**：`z/agy/rc/track-c-audit`（基于 `abeab8fa0462382d53bac7ac94b2d0360d6013b6`）  
**已推送 Verified Commit**：[`0401e7a205e1681b987a376c9d1450af088c42eb`](https://github.com/Antigravi-ty/RocketCompanion/commit/0401e7a205e1681b987a376c9d1450af088c42eb) (`signature.isValid: true`)

---

## 一、实测 Bug 根因定位与已落地的修复

### 1. 导出 `1.RCDPack` 在 frame 2775「完全没对上」的根因（已修复）
- **根因**：对用户提供的 `1.RCDPack` 与 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz` 逐帧比对发现：
  - 球的位置完全一致（frame `2772` 均位于 `[-2020.60, 2259.54, 668.63]`）。
  - 但 `1.RCDPack` 中 `roster` 与每帧 `cars` 数组顺序为 `[3, 4, 5, 0, 1, 2]`（`ReplayViewerRosterSubController.resolvedRoster` 将 Player 所在队伍排在前面），而 `DynamicTrainingSliceExtractor.ts` 直接按 `roster` 顺序写入 `cars` 并透传给 `hydrateRLRCFramesFromDynamicSession`。
  - 下游 `RLRCInterveneTargetFrameApplier` / `RLRCInterveneNonPovSyncPipeline` / `RLRCIntervenePrerollHistoryInjector` 全部按 **数组下标 `frame.cars[slot]`** 取车。结果 **Slot 0 (`Nexto #4`) 拿到了 Car 3 (`Player`) 的位置/速度/朝向，Slot 3 (`Player`) 拿到了 Car 0 (`Nexto #4`) 的数据**，整场 6 辆车全部错位。
- **修复**（`src/replay/DynamicTrainingSliceExtractor.ts`）：
  1. `extractDynamicSessionFromReplayFrames` 与 `extractDynamicSessionFromRLRCFrames` 提取前强制按 `slotIndex` 升序排列 `roster` 与 `cars`。
  2. `hydrateRLRCFramesFromDynamicSession` 按 `car.slotIndex` 建立槽位索引数组 `cars[slot]`（即使加载之前已导出的 `1.RCDPack` 旧文件也能自动纠正错位）。
  - **实测验证**：用 `1.RCDPack` + `.rcreplay.gz` 跑 Node 校验，6 辆车 `slotIndex` 全部对齐 `0..5`，位置误差 `0.000000`。

### 2.「3-2-1 倒计时莫名其妙」与「倒计时结束后速度突然飞快」的根因（已修复）
- **根因**：双重步进 + 帧率耦合：
  1. `SimulationStage.ts` 的 `CUSTOM_STEP_MODES` 集合中**遗漏了 `'dynamic-training-pack'`**，导致 `SimulationStage` 的 120Hz `interpolator.update()` 不断调用 `physics.step(1)`。
  2. 与此同时，`DynamicTrainingMode.update(context)` 又在 **渲染帧率（60/120/144Hz）** 下调用 `this.interveneController.stepTick(...)`，内部再次调用 `physics.step(1)`。
  3. 结果：倒计时阶段渲染线程按 60Hz 拉回 pre-roll 帧而 `SimulationStage` 仍在 120Hz 偷跑物理（导致画面抖动、3 秒倒计时被拉长到 ~6 秒）；进入 `intervening` 后两边同时调 `physics.step(1)`，物理流速直接翻倍（2x+ 飞快）。
- **修复**：
  - `src/game/pipeline/stages/SimulationStage.ts`：将 `'dynamic-training-pack'` 加入 `CUSTOM_STEP_MODES`，并在 `interpolator.update` 回调中路由到 `onStepPhysicsTick()`。
  - `src/game/modes/dynamic-training/DynamicTrainingMode.ts`：将 pre-roll 倒计时与 `intervening` 物理步进从 `update()` 迁入 `onBeforePhysicsTick()` / `onStepPhysicsTick()`，由 120Hz 累加器单源确定性驱动。

### 3. 只有「接管前 4 秒」导致后续 `as-previous` 失控：两段式标签区间（已修复）
- **问题**：原实现仅截取 `[t0 - 480 + 1, t0]`，`t0` 之后没有帧数据，导致 `as-previous`（`follow-state`）在接管后第 1 帧就因 `frames[followIdx]` 为 `undefined` 而失去原始轨迹。
- **修复**：
  - `dynamicTrainingPackTypes.ts` / `DynamicTrainingSliceExtractor.ts`：为 `DynamicTrainingStateSession` 新增 `interveneFrameOffsetInSession` 与 `originalMovementEndFrame120Hz`，支持截取 `[max(0, t0 - 479), max(t0, tEnd)]`。
  - `DynamicTrainingMode.ts`：`startShot` 改以 `session.interveneFrameOffsetInSession` 作为接管锚点（兼容旧 4 秒切片）。
  - `replayViewerStore.ts` / `StandaloneReplayViewerController.ts` / `ReplayViewerFloatingWindow.tsx`：新增 **`Tag Intervene From`** 与 **`Tag Movement End`** 两个打标按钮及状态存储，导出时按双标签区间切片。

### 4. RCDPack 悬浮窗改用标准 UI 组件 & 禁用 `Optional POV Car Spawn Variation`（已修复）
- `DynamicTrainingPackManagerSection.tsx` & `DynamicTrainingShotEditorCard.tsx`：将原生 `<select>` / `<button>` / `<input type="checkbox">` 替换为标准 `DropdownMenu`、`Button`、`SegmentedSwitch`、`Checkbox`。
- 将 `Optional POV Car Spawn Variation` 设为 `disabled` 并标注为 **`Optional POV Car Spawn Variation (under evaluation)`**。

### 5. 回放暂停/冻结时地面 Boost Pad 动画抖动（已修复）
- **根因**：`src/entities/BoostPadSystem.ts` 在 `!isActive` 时每帧都会 `padCooldownTimers[i] -= safeDt`，当与 `stateBuffer` 中静态的 `bufferCooldown` 偏差超过 `0.5s` 时又强行拉回 `bufferCooldown`，形成每 0.5 秒循环一次的充能环抖动。
- **修复**：检测 `EXTENDED_STATE_OFFSETS.IS_SIMULATION_FROZEN` 或 `bufferCooldown` 与上一帧完全一致（静态冻结）时，停止递减 `padCooldownTimers[i]` 并直接锁定为 `bufferCooldown`。

---

## 二、关于「500% 回放倍速是否会每秒发 600 个包」的结论
- **不会，且网络广播频率由 Track D/F 管，不由 Replay 倍速决定**：
  - `ReplayTakeoverSubController` / `RLRCReplaySessionBridgePipeline` 在 500% 倍速下是在 **120Hz tick 内将游标步长放大为 `5.0` 帧/tick**（跳帧推进），tick 频率本身仍是 120Hz，并非跑到 600Hz。
  - 联机房间的网络快照广播频率由 Track D 的 `RoomReplicationBus`（Continuous 60/120Hz 或 Sparse 2Hz Ping-Pong）与 `AuthoritativeServerHost` 的固定发送节拍控制，与本地回放倍速彻底解耦。

---

## 三、如何检验 Track C 是否具备合并条件（Merge Readiness Checklist）

建议按以下 4 步验收，全部通过即可合并到 `alpha`：
1. **静态与架构门禁**（本次已通过）：
   - `pnpm run check`（Radar 2.0 + `tsc --noEmit`）0 Error。
   - `pnpm run build` 生产构建通过。
2. **数据与槽位一致性回归**（本次已通过）：
   - 对 `1.RCDPack` 与真实 3v3 `.rcreplay.gz` 验证 `frame.cars[slot].slotIndex === slot` 且全员 6 车坐标误差 `0.000000`。
3. **浏览器交互实测（待您本地端到端确认）**：
   - 在 Replay Viewer 加载 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz`：
     - 暂停在含正在充能 Boost Pad 的帧，确认地面 Boost Pad 充能环**完全静止不抖动**。
     - 在 `2775` 打 `Tag Intervene From`，在后续节点（如 `3015`）打 `Tag Movement End`，点击 `Put into Dynamic Training Pack (.RCDPack)`。
     - 点击 Play：确认 6 辆车位置/车型/队伍与回放完全一致、3-2-1 倒计时为正常 1:1 实时秒数、倒计时结束后物理流速正常 1x。
     - 测试 `5275` 场景下的 5 种接管模式（`player` / `as-previous` / `disappear` / `no-input` / `model`）。
4. **后续仍可作为 Track C2 / Track F 联动项（不阻塞本次基础合并）**：
   - `tEnd`（`originalMovementEndFrame120Hz`）之后，原本处于 `as-previous` 的车辆在序列结束后的默认行为目前会保持最后一帧控制输入/惯性滑行；后续可按需增加「`tEnd` 后自动转为 `no-input` 或 `model`」的显式选项。
   - `RLRCReplayToRcsImporter` 如需同样的双标签按钮，可复用 `useReplayViewerStore` 的同款双标签模式。
