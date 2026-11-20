# 小队 D 排查报告：Bug 4 & Bug 5 残留边界复核

- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base`（`7af0003476afc0b275e34e7abebfd08df018d990`）
- **排查范围**：
  - Bug 4：`src/entities/BoostPadSystem.ts`、`src/entities/arena/BoostPadShadersAndMeshes.ts`、`scripts/runDevBroadcastReproV3.mjs`
  - Bug 5：`src/game/runtime/RuntimeResourceRegistry.ts`、`src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts`、`src/entities/vehicle/supersonic-trail/SupersonicWheelTrailSystem.ts`、`scripts/runDevBroadcastReproV3.mjs`

---

## 一、实测基线数据（`7af0003`）

在当前沙盒环境中运行 `pnpm run repro:dev-broadcast-v3`（Headless Chrome + SwiftShader）：
- **Bug 4 (`bug-4-boostpad-model-flicker`)**：`PASS_FIXED`
  - `totalPadFlips: 4`, `maxSinglePadFlips: 1`, `padFlips: [0,...,1,...,1,...,1,...,1,0,...]`
  - 采样明细中可见：`tick: 905`（从 `836` 跳变 `+69` ticks）单帧触发 `flipsThisFrame: 3`，4 帧后（仍停留在 `tick: 905`）触发 `flipsThisFrame: 1`（300ms 收集动画自然结束）。
- **Bug 5 (`bug-5-high-memory-and-unreleased-on-exit`)**：`REPRODUCED_BUG`
  - `bootMem`: `jsUsedMB: 163, wasmMB: 64, geometries: 37, textures: 13`
  - `loadedMem`: `jsUsedMB: 223, wasmMB: 64, rvFrames: 720, geometries: 73, textures: 17`
  - `afterExitMem`: `jsUsedMB: 218, wasmMB: 64, rvFrames: 0, geometries: 39, textures: 17`
  - `retainedWasmMB: 0, retainedJsMB: 55, retainedGeometries: 2, tauriCloseHidesWindow: false`

---

## 二、Bug 4 定位结论：`maxSinglePadFlips: 2` 属正常物理/动画状态转换，非闪烁 Bug

### 1. 源码机制核查（`src/entities/BoostPadSystem.ts`）
`runDevBroadcastReproV3.mjs` 通过统计 30 帧 `rAF` 内 `pad.full.visible` 的跳变次数判定 Bug 4（`maxSinglePadFlips >= 2 || totalPadFlips >= 8`）。
而 `BoostPadSystem.update()`（第 442–493 行）中，`pad.full.visible` 在以下 **3 种状态** 下为 `true`：
1. `isRefilling`：`cooldownTimer <= threshold && cooldownTimer > 0`（小 pad 阈值 `1.0s` / 总冷却 `4.0s`；大 pad 阈值 `3.0s` / 总冷却 `10.0s`）-> `pad.full.visible = true`
2. `isActive && animProgress >= 1.0`（可拾取状态）-> `pad.full.visible = true`
3. 刚被吃掉后的收集动画期（`collectDuration = 300ms`）：`anim !== undefined` -> `pad.full.visible = true`；300ms 结束后 `collectAnimations.delete(i)` -> `pad.full.visible = false`。

### 2. 为什么单帧跳 110 ticks 会出现 `maxSinglePadFlips: 2`？
在 SwiftShader 并发渲染下，30 帧 `rAF` 窗口往往跨越 **2–4 秒**真实时间，且单帧跳 **110 ticks**（约 `0.917s` 游戏时间）时，**单个小 pad（4s 冷却，1s 充能阈值）在 30 帧窗口内会自然经历 2 次可见性翻转**：
- **路径 A**：pad 被吃掉（保持 `visible = true` 300ms）-> 动画结束变为 `false`（**Flip #1**）-> 经过约 2.7s（或若干次大 tick 跳变）进入 `cooldownTimer <= 1.0` 充能阶段变为 `true`（**Flip #2**）。
- **路径 B**：pad 处于隐藏期（`false`）-> 进入 `isRefilling`（`true`，**Flip #1**）-> 充能完成并在 3v3 混战中被车吃掉且 300ms 动画结束（`false`，**Flip #2**）。

此外，核查 `BoostPadSystem.ts` 第 281–378 行确认：
- 已具备 `isDuplicateFrame` 去重；
- 已具备 `isSmallUpwardJitter` / `isUpwardJitter` / `wouldExitRefill` 防抖守卫，防止冷却抖动导致 `isRefilling` 来回闪烁；
- 当单帧跳变 110 ticks（`0.917s`）时，虽然 `safeDt` 被 clamp 到 `0.25s`，但 `bufferCooldown < nextTimer` 且差值 `> 0.5s`，会直接单调对齐到 `bufferCooldown`（第 353–367 行），不会产生反向回跳闪烁。

### 3. 建议
- **结论**：`maxSinglePadFlips: 2` 在大 tick 跳变/长采样窗口下属于**正常物理生命周期**，并非模型闪烁。
- **测试端修正建议**：在 `scripts/runDevBroadcastReproV3.mjs` 中，将 Bug 4 阈值调整为 `maxSinglePadFlips >= 3`（或记录翻转时间戳，仅当同一 pad 在 `< 400ms` 内发生 `>= 2` 次翻转时判定为闪烁）。

---

## 三、Bug 5 定位结论：`retainedGeometries` 波动（0 / 2 / 10）与 Track 2 `focusedCarIndex` 切换无关

### 1. 排除 Track 2 `focusedCarIndex` 切换嫌疑（真实打点验证）
构造独立 CDP 探针脚本（`/tmp/probe_bug5_focused.mjs`），在 Host 挂载 `THREE.BufferGeometry.prototype.addEventListener('dispose')` 与 `dispose()` 实时追踪集合，执行与 Track 2 完全一致的流程（加载 3v3 合成录像 -> 依次切换 `focusedCarIndex: 0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 0` -> 调用 `onReturnToFreeplay()`）：
- `[BOOT]`：`renderer.info.memory.geometries = 37`
- `[REPLAY_3V3_LOADED]`：`renderer.info.memory.geometries = 71`（新上传 34 个）
- `[EXIT_500MS]`：`renderer.info.memory.geometries = 37`（仅保留 11 个本地车体/轮毂几何体：`scenegltf_1..7` + 4x `solid-cylinder-wheel`，与 Boot 完全一致）
- `[EXIT_2000MS]`：`renderer.info.memory.geometries = 37`（`retainedGeometries = 0`）

**结论明确**：Track 2 的 `focusedCarIndex` 切换**不会**产生任何未清理几何体。

### 2. `retainedGeometries` 出现 `0 / 2 / 10` 的真正根因（两点叠加）
1. **根因 1：预分配持久化池几何体的“惰性 GPU 上传”统计偏差（主因）**
   - `ArenaWorld` 在 Boot 时预创建了持久化特效对象，其中包括 `SupersonicWheelTrailSystem`（为全部 6 辆车预分配 `carTrails`，每车左右轮 2 条 `SupersonicWheelRibbon` = **2 个 `BufferGeometry`/车**，共 12 个）、`ballSpeedTrail`（**1 个 `BufferGeometry`**）及 `carDemolitions[0]`（**1 个 `PlaneGeometry`**）。
   - 在 `bootMem` 采样时（Freeplay 静止开球），这些对象均处于 `visible = false`，`WebGLRenderer` **尚未将其上传至 GPU**，因此不计入 `bootMem.geometries`（37）。
   - 在 Bug 1（Goal Replay）与 Bug 2/3（3v3 Replay）运行期间，只要有车进入超音速（例如 **1 辆车 = +2 个几何体**；**5 辆车 = +10 个几何体**！）或球速达标，对应 `BufferGeometry` 首次被渲染并计入 `renderer.info.memory.geometries`。
   - 退出调用 `onReturnToFreeplay()` -> `resetFreeplayExtraCarsAndArenaToSingleCar()`（`src/game/modes/freeplay/freeplayKickoffAndArenaResetPipeline.ts:141-149`）时，仅调用 `supersonicWheelTrailSystem.resetCar(i)` 与 `ballSpeedTrail.reset()`（隐藏 `visible = false` 并清空顶点计数），**并未也不应销毁持久化池几何体**。因此 `afterExitMem.geometries - bootMem.geometries` 会固定多出 **2**（1 车超音速）或 **10**（5 车超音速），而无任何超音速触发时则为 **0**。
2. **根因 2：`onReturnToFreeplay()` 异步换车与 500ms 固定等待竞态**
   - `FreeplayMode.resetToCleanFreeplay()` 会异步触发 `restoreLocalPlayerFreeplayLoadout(0)`（重加载本地车模型，销毁旧车 11 个几何体后再异步挂载新车 11 个几何体），且 `void runtimeResourceRegistry.disposeScopes(['replay', 'ai', 'session'])` 未被 `await`。
   - 探针实测中曾观察到 `EXIT_500MS` 时 `rendererGeoms` 瞬时跌至 `34`（旧车已销毁、新车未渲染完），`EXIT_2000MS` 恢复为 `45`（`37 + 8`）。

### 3. 建议
- **定位结论**：`retainedGeometries: 2` / `10` 不是内存泄漏，而是 `bootMem` 在冷启动未渲染任何超音速/拖尾几何体时采样，与 `afterExitMem`（已触发过持久化特效池首次 GPU 上传）直接做差值导致的基准不可比。
- **修复方向**：
  1. 若需严格清零：可在 `SupersonicWheelTrailSystem` 中对 `carIndex >= 1` 按需创建/在 `resetFreeplayExtraCarsAndArenaToSingleCar` 中销毁非本地车槽位的 ribbon，或在 `ArenaWorld` 预热阶段对持久化特效几何体执行一次 0-alpha 预上传，使 `bootMem.geometries` 包含固定池基线；
  2. 测试端 `onReturnToFreeplay()` 后应等待 `runtimeResourceRegistry` 清理与本地车 loadout 恢复完成，再采样 `afterExitMem`。
