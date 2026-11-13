# Dynamic Training 两个 Commit 对比与玩家无输入问题排查报告

## 一、`0401e7a` vs `f70fbf3` 区别对比

这两个 Commit 均从同一父提交 `abeab8f`（`origin/z/agy/rc/track-c-dynamic-training`）分叉：
- `0401e7a205e1681b987a376c9d1450af088c42eb`（`origin/z/agy/rc/track-c-audit`）
- `f70fbf369eb1c4a3d72013bf2f30925908acb9e7`（`origin/z/agy/rc/track-c-fix-and-audit`）

主要差异如下：

| 维度 | `0401e7a` (`track-c-audit`) | `f70fbf3` (`track-c-fix-and-audit`) |
| --- | --- | --- |
| **120Hz 物理步进管线** (`SimulationStage.ts` / `DynamicTrainingMode.ts`) | 在 `SimulationStage.ts` 中把 `dynamic-training-pack` 特判到与 `kickoff-training` 相同的分支，由 `DynamicTrainingMode.onStepPhysicsTick()` 手动调用 `sim?.step?.(1)`；倒计时阶段在 `onBeforePhysicsTick()` 里调用 `stepTick()` 并返回 `false`。 | 保持 `SimulationStage.ts` 统一的 `isCustomStepMode` 路径（`physics.step(1)` + `onAfterPhysicsTick()`），`DynamicTrainingMode` 遵循标准的 `onBeforePhysicsTick()` / `onAfterPhysicsTick()` 生命周期。 |
| **`playerCarIndex` 同步** (`DynamicTrainingMode.ts`) | **缺失**。未在 `loadAndPlayShot()` 中同步 `runtime.playerCarIndex` 和 `runtime.arena.playerCarIndex`。 | **已修复**。在 `loadAndPlayShot()` 中将 `runtime.playerCarIndex` 与 `runtime.arena.playerCarIndex` 同步为 `shot.povSlotIndex`。 |
| **暂停时 BoostPad 冻结** (`BoostPadSystem.ts`) | **未真正修复**。仅新增了一个未使用的 `EXTENDED_STATE_OFFSETS` 导入；`safeDt` 仍为 `dt > 0 ? ... : 0.016`，`dt === 0` 时会回退到 `0.016` 继续倒计时。 | **已真正修复**。改为 `dt >= 0`（允许 `0`），并增加 `isBufferFrozen` 判断，在暂停时冻结本地 `padCooldownTimers` 并同步 `bufferCooldown`。 |
| **双标记截取与桥接** (`ReplayViewerFloatingWindow.tsx` / `runtimeDynamicTrainingBridge.ts` / `RLRCReplaySessionController.ts`) | 在 `replayViewerStore` 中加了标记状态，但 `runtimeDynamicTrainingBridge.ts` 未把选项透传给 `extractCurrentDynamicSession()`，且未覆盖 `RLRCReplaySessionController`（`.replay` 导入器）。 | 完整实现了 `ReplayViewerFloatingWindow` 的双标记操作与 `runtimeDynamicTrainingBridge.ts` 参数透传（同时支持 `StandaloneReplayViewerController` 和 `RLRCReplaySessionController`）。 |
| **Slot 索引排序与稀疏补齐** (`DynamicTrainingSliceExtractor.ts`) | **独有重要修复**。在 `toCompactFrame`、`compactRoster` 与 `hydrateRLRCFramesFromDynamicSession` 中按 `slotIndex` 排序，并在反序列化时按 `0..maxSlot` 补齐缺失槽位，避免车位索引错乱。 | 未包含该排序与稀疏槽位补齐逻辑。 |

---

## 二、哪个更好？应该基于谁？

**结论：应该基于 `f70fbf3`（`origin/z/agy/rc/track-c-fix-and-audit`），并将 `0401e7a` 中 `src/replay/DynamicTrainingSliceExtractor.ts` 的 slot 排序与稀疏槽位补齐逻辑合入。**

理由：
1. `f70fbf3` 的 `SimulationStage` / `DynamicTrainingMode` 生命周期接入更符合项目现有架构，且正确同步了 `playerCarIndex`。
2. `f70fbf3` 真正修好了 `BoostPadSystem.ts` 的暂停冻结，而 `0401e7a` 的该项改动实际上无效。
3. `f70fbf3` 的双标记截取在 UI、Bridge 和 `RLRCReplaySessionController` 三端是打通的。
4. 唯一需要从 `0401e7a` 吸收的是 `DynamicTrainingSliceExtractor.ts` 中对 `roster` / `cars` 按 `slotIndex` 排序与缺位填充的防错乱修复。

---

## 三、点击 Play 有倒计时音效，但完全动不了、任何玩家输入都无效的原因

### 1. 直接根因：`DynamicTrainingMode.readPlayerControls()` 读取了不存在的方法
在两个 Commit 的 `src/game/modes/dynamic-training/DynamicTrainingMode.ts` 中，`readPlayerControls()` 写法如下：

```ts
private readPlayerControls(): RLRCSynthesizedCarControls {
  const r = this.runtime as unknown as {
    input?: { getPlayerControls?: () => Partial<RLRCSynthesizedCarControls> } | null;
  };
  const raw = r?.input?.getPlayerControls?.();
  if (!raw) return DEFAULT_NEUTRAL_CAR_CONTROLS;
  ...
}
```

**`GameRuntime` 上根本不存在 `runtime.input.getPlayerControls()`。**
实际运行流程是：
1. `SimulationStage` 每帧先执行 `this.runtime.pollInputs()`（`RuntimeInputPoller.ts`），把键盘/手柄输入写入 `this.runtime.playerControls` 并调用 `physics.setControls(playerCarIndex, car0Controls)`。
2. 紧接着 `SimulationStage` 调用 `DynamicTrainingMode.onBeforePhysicsTick()` -> `readPlayerControls()`。
3. 由于 `r?.input?.getPlayerControls?.()` 恒为 `undefined`，`readPlayerControls()` **永远返回全 0 / 全 `false` 的 `DEFAULT_NEUTRAL_CAR_CONTROLS`**。
4. 随后 `interveneController.stepBeforePhysicsTick(playerControls, RLRC_FIXED_DT)` -> `prepareInterveneStepBeforePhysics()` 调用 `applySlotControls(sim, povSlotIndex, playerControls)`，**把 `pollInputs()` 刚写入物理引擎的玩家输入直接覆盖清零**。

因此，倒计时结束进入 `intervening` 阶段（倒计时音效正常）后，物理引擎每帧收到的 POV 控制量永远是空输入，车子完全动不了。

### 2. 为什么把 6 个 slots 全调成 `player` 也不起作用？
1. POV 槽位本身就被上述 `readPlayerControls()` 的空输入覆盖了。
2. 在 `DynamicTrainingMode.ts` 的 `mapTakeoverModeToRLRC()` 中：
   ```ts
   case 'as-previous':
   case 'player':
   default:
     return 'follow-state';
   ```
   非 POV 槽位选 `'player'` 在当前实现里直接映射为 `'follow-state'`（跟随原回放轨迹），并不会读取任何本地玩家输入。

### 3. 修复方向（供后续新建 `z/agy/rc/*` 分支时参考）
将 `DynamicTrainingMode.readPlayerControls()` 改为直接读取 `this.runtime.playerControls`（即 `RuntimeInputPoller.pollInputs()` 写入的字段），即可恢复 POV 槽位的玩家输入。
