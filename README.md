# Track C (`abeab8fa`) 检验评估、Bug 根因调查与修复报告

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **基准 Commit**: `abeab8fa0462382d53bac7ac94b2d0360d6013b6` (`z/agy/rc/track-c-dynamic-training`)
- **修复后分支**: `z/agy/rc/track-c-fix-and-audit` @ `f70fbf369eb1c4a3d72013bf2f30925908acb9e7` (GitHub Verified Bot Commit)

---

## 一、如何检验 Track C 是否可以开始合并

在将 Track C 合入 `alpha` 之前，建议按以下 4 步做门禁检验：

1. **静态架构与类型门禁 (`pnpm check`)**:
   - 运行 `pnpm check`（即 `radar.ts` + `tsc --noEmit`），确保 `0` Radar Hard Error、`0` 循环依赖、`0` 桶文件重导出、`0` 中文注释/Emoji、`0` TypeScript 严格模式错误。
2. **生产构建验证 (`pnpm build`)**:
   - 确保 Vite 生产包无打包/Tree-shaking 异常。
3. **基线隔离度检查 (`git diff --stat 760a8fc..HEAD`)**:
   - 确认 Track C 仅改动其职责范围内的 27 个核心文件（`src/game/modes/dynamic-training/`、`src/replay/DynamicTrainingSliceExtractor.ts`、`src/ui-core/config/dynamicTrainingPack*`、`src/ui-core/stores/dynamicTrainingPack*`、`src/ui-core/panels/dev/dynamic-training/`、`src/ui-system/core/runtimeDynamicTrainingBridge.ts` 等），未污染无关模块。
   - **注意**: `abeab8fa` 分叉自 `760a8fc`，早于 `alpha` 上的 PR #10 / #11 / #13，合并时建议先 `rebase` 或 `--no-ff` 解决 `alpha` 最新提交的对齐。
4. **运行时关键路径实测**:
   - 使用真实 `.rcreplay` / `.RCReplay` 截取 `.RCDPack` 后，验证：
     1. 3-2-1 倒计时是否为严格 120Hz 实时秒数（而非随渲染帧率快慢变化）；
     2. 进入 `intervening` 后物理速度是否为严格 1.0x（120Hz）；
     3. 设为 `As Previous` 的非 POV 车辆在接管后是否继续执行原回放动作直至 `Origin End`。

**结论**: 原始 `abeab8fa` **不可直接合并**（存在下面第 2 节所述的 120Hz 双重步进与截断至 `t0` 导致 `As Previous` 失效等阻塞 Bug）；在合入本次 `f70fbf3` 修复后，已具备合入条件。

---

## 二、问题根因调查 (`1.RCDPack` 与 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz`)

### 1. 为什么 3-2-1 倒计时莫名其妙，结束后速度变得飞快？
- **根因**:
  1. `src/game/pipeline/stages/SimulationStage.ts` 中的 `CUSTOM_STEP_MODES` 集合**漏掉了 `'dynamic-training-pack'`**。
  2. `DynamicTrainingMode.ts` 没有实现 `onBeforePhysicsTick()` / `onAfterPhysicsTick()`，而是在每帧渲染回调 `update(context)` 中直接调用 `this.interveneController.stepTick(...)`（内部会调用 `physics.step(1)`）。
- **后果**:
  - **倒计时阶段 (`preroll-countdown`)**: `SimulationStage` 仍在以 120Hz 自动调用 `physics.step(1)`，而 `DynamicTrainingMode.update()` 仅按渲染帧率（如 60fps）每帧把物理状态覆写回 `prerollFrame` 并只扣减 1 个 tick——导致 60fps 下 3 秒倒计时实际耗时 6 秒，且物理引擎在两帧之间空转抖动。
  - **接管阶段 (`intervening`)**: `SimulationStage` 以 120Hz 调用 `physics.step(1)`，同时 `DynamicTrainingMode.update()` 每渲染帧**再次调用 `physics.step(1)`**，导致物理速度变成 **1.5x ~ 3x+（取决于显示器刷新率）**，即“结束后突然速度变得飞快”。

### 2. 为什么 2775 / 2883 导出后“完全没对上”？
我们用仓库原生的 `deserializeRCReplayBinaryContainer` 解析了 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz`，并与 `1.RCDPack` 逐帧比对：
- `1.RCDPack` 的 `Session 0` 记录 `sourceFrame120Hz = 2772`、`ballPos = [-2020.60, 2259.54, 668.63]`，与原回放第 2772 帧**完全一致**；`Session 1` 记录 `sourceFrame120Hz = 5292`、`ballPos = [-2502.53, -326.77, 675.59]`，与原回放第 5292 帧**完全一致**。
- **真正导致“对不上”的原因有两点**:
  1. **旧截取逻辑在 `t0` 处截断，没有保留 `t0` 之后的帧**: `1.RCDPack` 只存了 `[t0 - 479 .. t0]` 共 480 帧。当进入 `intervening` 时，`RLRCInterveneNonPovSyncPipeline` 按 `followIdx = targetFrame120Hz + postTakeoverFrameOffset`（即 `479 + 0, 480, 481...`）读取 `frames[480]`，结果全部是 `undefined`，所有设为 `As Previous` 的 5 辆 Nexto 车在接管第 1 帧就全部失去控制输入（退化为 `no-input`），场面与预期完全脱节。
  2. 叠加上述 **2x–3x 物理倍速 Bug**，导致接管后球和车瞬间飞走。

### 3. 为什么回放暂停时地面 Boost Pad 充能动画在抖？
- **根因**: `src/entities/BoostPadSystem.ts` 第 269 行：
  `const safeDt = Number.isFinite(dt) && dt > 0 ? Math.min(0.25, dt) : 0.016;`
  即使传入 `dt === 0`，也会回退成 `0.016`。在回放暂停期间，每渲染帧都会把 `padCooldownTimers[i]` 减去 `0.016` 秒，充能进度条向前推进约 `0.5s` 后触发第 305 行的纠偏阈值 `Math.abs(padCooldownTimers[i] - bufferCooldown) > 0.5`，瞬间又弹回 `bufferCooldown`，周而复始产生明显抖动。

---

## 三、本次修复内容 (`z/agy/rc/track-c-fix-and-audit`, commit `f70fbf3`)

1. **修复 120Hz 确定性物理步进与倒计时**:
   - `src/game/pipeline/stages/SimulationStage.ts`: 将 `'dynamic-training-pack'` 加入 `CUSTOM_STEP_MODES`。
   - `src/game/modes/dynamic-training/DynamicTrainingMode.ts`: 移除 `update()` 中的 `stepTick`，改为实现 `onBeforePhysicsTick()` / `onAfterPhysicsTick()` 由 `SimulationStage` 的 120Hz 固定步长累加器驱动；同步设置 `runtime.playerCarIndex = shot.povSlotIndex`；使用 `session.interveneLocalFrameIndex ?? (frames.length - 1)` 作为接管锚点。
2. **新增双标签区间截取 (`Mark Intervene` + `Mark Origin End`)**:
   - `src/ui-core/config/dynamicTrainingPackTypes.ts`: 为 `DynamicTrainingStateSession` 新增可选字段 `originEndFrame120Hz` 与 `interveneLocalFrameIndex`（向后兼容旧 `.RCDPack`）。
   - `src/replay/DynamicTrainingSliceExtractor.ts`: 支持传入 `originEndFrameIndex`，截取 `[prerollStart .. originEnd]`（若未显式标记 `Origin End`，默认向后多保留 4 秒 / 480 帧，保证 `As Previous` 车辆不会在接管瞬间断输入）。
   - `src/ui-core/panels/user/ReplayViewerFloatingWindow.tsx`、`replayViewerStore.ts`、`replayToRcsImporterStore.ts`、`StandaloneReplayViewerController.ts`、`RLRCReplaySessionController.ts`、`runtimeDynamicTrainingBridge.ts`: 新增 **`Mark Intervene`** 与 **`Mark Origin End`** 两个打点按钮及 **`Capture Segment into Dynamic Training Pack (.RCDPack)`** 透传。
3. **修复暂停时 Boost Pad 充能动画抖动**:
   - `src/entities/BoostPadSystem.ts`: 允许 `dt === 0` 保留为 `0`；当 `!wasActive && !isActive` 且 `safeDt === 0` 或 `Math.abs(bufferCooldown - prevBufferCd) < 1e-6`（即 `stateBuffer` 冷却值冻结在同一帧）时，锁定 `padCooldownTimers[i] = bufferCooldown`，不再随渲染帧率递减回弹。
4. **RCDPack 悬浮窗改用 `ui-core` 标准组件 & 禁用未定项**:
   - `src/ui-core/panels/dev/dynamic-training/DynamicTrainingPackManagerSection.tsx`: 将原生 `<select>` 替换为标准 `DropdownMenu`。
   - `src/ui-core/panels/dev/dynamic-training/DynamicTrainingShotEditorCard.tsx`: 将倒计时 `<select>` 替换为标准 `SegmentedSwitch`（`0s | 1s | 2s | 3s`），将 per-car takeover 与 AI 模型 `<select>` 替换为标准 `DropdownMenu`，并将 `Optional POV Car Spawn Variation` 替换为禁用态标准 `Checkbox` 并标注 **`(under evaluation)`** 禁止点击。

### 验证结果
- `pnpm check`: **GREEN**（`1364/1364` TS 文件、`tsc --noEmit` 0 错误、0 循环依赖、0 硬边界违规）。
- `pnpm build`: **PASS**（28.31s 构建成功）。
- 实测脚本（`node --experimental-strip-types` 使用 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay.gz` 截取 `intervene=2775, originEnd=2883`）：输出 `framesLen=588`、`interveneLocalFrameIndex=479`、`prerollStartFrame120Hz=2296`、`originEndFrame120Hz=2883`，球位置与原回放第 2775 / 2883 帧精确对齐。

---

## 四、关于 Replay 500% Playback Speed 与网络广播频率的说明

- **归属**: **不归 Track C 管**，属于 **Track D (`RoomReplicationBus`) + Track F (`MatchSystemControlSubController` / Replay 多人观战室)**。
- **架构建议**: `500%` 仅控制 Host 端回放游标的前进倍速（`cursorFloat += 5.0 * dt * 120`），**网络快照广播频率必须与 Playback Speed 解耦并设置硬上限（如 30Hz 或 60Hz）**，绝对不能按 `500% * 120Hz = 600 packets/sec` 发送；Client 端收到状态包后通过本地插值/重采样平滑播放即可。
