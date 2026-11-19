# RocketCompanion 多人同步与内存修复：架构诊断与细粒度执行计划

> 基线分支：`origin/z/agy/rc/multiplayer-recheck`（HEAD: `943b749`）
> 目标：在不直接改动业务代码的前提下，评估 Wave 1–3 的可拆分性、Track A/B 先行合并可行性、以及仓库架构对修复难度的影响，给出可落地的细粒度并行与合并计划。

---

## 一、核心结论摘要

1. **Track A-1（Bug 1）与 Track B（Bug 2）可以立即合并，零冲突。**
   两者修改文件完全正交（模拟管线 vs. Runtime Sync Channel / NameTag 状态同步），建议在 Wave 1 率先合入作为 **Integration Checkpoint 1**，使 Bug 1 + Bug 2 先清零，同时为 Wave 2（Bug 3）提供稳定的 Goal Replay 管线基线。
2. **Wave 1 Track C（Bug 5）之所以“太难”，核心原因在于把 4 个完全正交的子系统捆绑成了一个巨石分支，且被 `runDevBroadcastReproV3.mjs` 的四元 `OR` 门禁绑死。**
   可拆分为 **C1（WASM 内存） / C2（Three.js 几何体与车位销毁） / C3（JS Heap / AI 模型与 RingBuffer 清理） / C4（Tauri 窗口关闭语义）** 四个独立子轨道。其中 C1–C3 均为纯 Web/TS/CDP 可验证，C4 是唯一依赖 Rust/Tauri 工具链的部分。
3. **Wave 2（Bug 3）与 Wave 3（Bug 4）均可进一步细拆为“数据/缓冲契约层”与“管线/动画消费层”两步走**，降低在 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 上反复返工的风险。
4. **“难修”的本质确实与仓库架构强相关**（详见第四节）：状态标志位组合爆炸、NameTag/Team 缺乏单一数据源（25+ 处散落调用）、全局单例缺乏生命周期卸载协议、以及 `Float32Array` 快照将连续量与离散布尔量混在同一线性插值通道中。

---

## 二、环境与工具链实测状态

在当前 Linux 验证容器（`/workspace/RocketCompanion`）中已完成以下检查与补齐：

| 工具/依赖 | 状态 | 说明 |
| :--- | :--- | :--- |
| `node` | ✅ `v22.23.5` | 支持 `--experimental-strip-types` |
| `pnpm` | ✅ 已安装 `v12.21.0` | 原环境缺失，已通过 `npm install -g pnpm` 补齐并完成 `pnpm install`（3.5s） |
| `pnpm run check` | ✅ PASS | `radar.ts`（1425/1425 TS 文件、0 循环依赖）与 `tsc --noEmit` 均绿灯通过 |
| `google-chrome` | ✅ `158.0.8127.36` | 可用于 `repro:dev-broadcast-v3` Headless CDP 测试 |
| `rustc` / `cargo` / `cc` | ❌ 缺失 | 当前容器无 C 编译器与 Rust 工具链，**仅影响 Sub-track C4（`src-tauri/src/lib.rs`）的编译验证**；C1–C3 及 Bug 1–4 均可在当前环境直接开发与验证 |

> **测试回放文件**：运行 `pnpm run repro:dev-broadcast-v3` 前需确保 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` 位于 `/tmp/`、`/workspace/` 或通过 `RC_REPLAY_FILE` 指定。

---

## 三、细粒度拆分与合并计划（Wave 1 → Wave 3）

### Wave 1：低耦合快赢 + Track C 四路拆解

#### 1. Track A-1（Bug 1：Goal Replay 客户端静态无动态）— **优先合入**
* **分支**：`fix/bug-1-goal-replay-client-dynamics`（基于 `z/agy/rc/multiplayer-recheck`）
* **核心改动面**：
  * `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`
  * `src/game/pipeline/stages/SimulationStage.ts`
  * `src/game/pipeline/stages/HostBroadcastPublisher.ts`
  * `src/replay/ReplayFrameInterpolator.ts`
  * `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
* **门禁**：`bug-1-goal-replay-client-static-no-dynamics.reproduced === false`。

#### 2. Track B（Bug 2：3D NameTag 与 `arena.carTeams` 不同步）— **与 Track A-1 并行并先行合并**
* **分支**：`fix/bug-2-replay-nametag-team-sync`（基于 `z/agy/rc/multiplayer-recheck`）
* **核心改动面**：
  * `src/game/runtime/devSyncChannelHandlers.ts`
  * `src/game/runtime/onlineSessionEnvironmentPipeline.ts`
* **与 Track A-1 的合并评估**：
  * **文件重叠度 = 0**，**语义耦合 = 0**。
  * **合并策略**：Track A-1 与 Track B 各自通过 `pnpm run check` 后，直接合并生成集成分支（如 `int/wave1-bug1-bug2`），作为 Wave 2 的统一底座。

#### 3. Track C（Bug 5：内存释放与 Tauri 关闭）— **拆分为 4 个独立子轨道（C1–C4）**

在 `scripts/runDevBroadcastReproV3.mjs`（L489–493）中，Bug 5 的判定是一个四元 `OR` 门禁：
```js
const bug5Reproduced =
  (bootMem.wasmMB ?? 0) >= 256 ||
  retainedWasmMB > 0 ||
  retainedGeometries > 0 ||
  tauriCloseHidesWindow;
```
这意味着只要其中任意一项未修，总开关就仍是 `reproduced: true`。但各子项的底层指标在 `/tmp/dev-broadcast-repro-v3.json` 中是**独立输出**的，因此完全可以拆成 4 个可独立验收的子任务：

| 子轨道 | 目标指标 | 涉及文件 | 工具链需求 | 相互依赖 |
| :--- | :--- | :--- | :--- | :--- |
| **C1：WASM 启动预分配与退出回收** | `bootMem.wasmMB < 256`（目标 ~64 MB）且 `retainedWasmMB <= 0` | `RocketSimArenaBarrierController.ts`（移除启动时 eager `initReplayBuffer`）、`PhysicsEngineManager.ts`（按 `isRecordingFullReplay` 守卫）、`RocketSimWasm.ts`、`RocketSimPhysicsEngine.ts`（`reclaimWasmMemoryIfGrown`）、`MatchModeTransitionBridge.ts` | 纯 Web / TS | 无，完全独立 |
| **C2：Three.js 几何体与车辆槽位销毁** | `retainedGeometries <= 0`（从 +65 降至 <= 0） | `ArenaWorld.ts`、`arenaVehicleSlotPipeline.ts`（`removeCar` + 裁切尾部空槽）、`ArenaVehicleLoadoutManager.ts`、`VehicleFactory.ts`（`collectReferencedGeometries` 仅销毁非共享几何体）、`RespawnGhostPreview.ts`、`customGlbCarModel.ts`、`freeplayKickoffAndArenaResetPipeline.ts`、`ReplayViewerCausalSyncController.ts` | 纯 Web / TS | 建议 C2 先于 C3 合入，避免 `freeplayKickoffAndArenaResetPipeline.ts` 冲突 |
| **C3：JS Heap / AI 模型与 RingBuffer 清理** | `retainedJsMB` 显著回落（227 MB → ~91 MB） | `wasmSimdMatVec.ts`（`resetWasmSimdMatVec`）、`SafeTensorsLoader.ts`（`clearSafeTensorsModelCache`）、`freeplayKickoffAndArenaResetPipeline.ts`（`applyCleanFreeplaySubsystemsReset` 清理 `ringBuffer`、`ballTrajectoryPredictor`、`renderer.renderLists`、`THREE.Cache.clear()`） | 纯 Web / TS | 接在 C2 之后改 `freeplayKickoffAndArenaResetPipeline.ts` |
| **C4：Tauri 窗口关闭语义** | `tauriCloseHidesWindow === false` | `src-tauri/src/lib.rs`（移除无条件 `api.prevent_close(); window.hide();`，主窗口关闭时清理子窗口/tunnel，处理 `WindowEvent::Destroyed`） | **需要 `cargo` / `rustc` + Tauri 系统库** | 完全独立，可随时合入 |

> **并行建议**：C1、C2、C4 可与 Wave 1 的 Bug 1、Bug 2 **同时开工**；C3 在 C2 合入后快速跟进。每完成一个子轨道，直接核对 `/tmp/dev-broadcast-repro-v3.json` 对应字段即可验收，无需等待全部 4 项齐备。

---

### Wave 2：确定性 WASM 物理前瞻（Bug 3）— 细分为两步

* **前置基线**：必须基于 **Bug 1 已合入** 的代码（因为 Bug 1 和 Bug 3 共同修改 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts`，且 Goal Replay 的状态机分支必须先稳定）。
* **为什么建议拆成 2 步**：直接把缓冲队列改造和 WASM 物理步进混在一起，极易引入 Replay 快退/Goal Replay 跳变问题。

1. **Wave 2-A：前瞻缓冲契约层（`ReplicationStateInterpolator.ts`）**
   * 为 `SpectateInterpolationBuffer` 引入缓冲前瞻阈值（如 `>= 5` 帧）与游标推进接口，区分“缓冲不足等待”与“可驱动前瞻窗口”，并确保与 Goal Replay / Pause 状态切换时的 `reset()` 语义兼容。
   * 可通过单元测试（`runMultiplayerSuiteTests.ts`）先行验证缓冲状态机。
2. **Wave 2-B：确定性 WASM 前瞻驱动与快照收敛（`SimulationStage.ts` + `simulationNetworkReconcilePipeline.ts`）**
   * 在 `isPureCsrClient` 下，当缓冲满足阈值时，不再只调用静止的 `interpolator.sync(timestamp)`，而是驱动本地 WASM 物理连续步进（保证 `tickCount` 连续递增，消除 `distinctTicks === 1` 与 `frozenBallFrames >= 3 && maxBallStepUu > 200`），并在新权威快照到达时平滑收敛误差。
   * **门禁**：`bug-3-replay-choppy-motion-no-physics-lookahead.reproduced === false`。

---

### Wave 3：BoostPad 模型闪烁（Bug 4）— 细分为两步

* **前置基线**：必须基于 **Wave 2（Bug 3）已合入** 的代码。原因是物理前瞻引入后，客户端本地预测帧与远端快照修正帧交替出现，正是诱发 BoostPad 冷却时间抖动与 `respawnedAndCollected` 误判的主要场景。

1. **Wave 3-A：插值与快照离散字段保护（`ReplicationStateInterpolator.ts` / `ReplayFrameInterpolator.ts`）**
   * 审查 `SpectateInterpolationBuffer.sample()` 与 `applyReplayFrameToEngine()`：确保 BoostPad 的离散激活标志（`isActive`）不被线性插值成中间态，且连续 `cooldown` 在跨帧插值/前瞻校正时不出现虚假回跳。
2. **Wave 3-B：`BoostPadSystem.ts` 状态跳变去抖与重复帧守卫**
   * 加固 `BoostPadSystem.ts`（L294–316）中的 `respawnedAndCollected` 与 `isBufferFrozen` 判定：屏蔽同帧重复应用、微小网络抖动导致的冷却回弹（`cooldown` 轻微增大但不应触发 `startCollectAnimation`），避免 `pad.full.visible` 逐帧翻转。
   * **门禁**：`bug-4-boostpad-model-flicker.reproduced === false`（`maxSinglePadFlips < 2 && totalPadFlips < 8`），并跑通全量 `repro:dev-broadcast-v3`（`reproducedCount === 0/5`）。

---

## 四、架构诊断：为什么这些 Bug 在当前仓库中修起来很吃力？

通过静态检查 `origin/z/agy/rc/multiplayer-recheck`（1,425 个 TS 文件），可以确认**代码仓库的架构特征确实显著放大了修复难度**，主要体现在以下 4 点：

1. **核心模拟管线的“状态标志位组合爆炸”**
   * `SimulationStage.ts`（468 行）与 `simulationNetworkReconcilePipeline.ts`（281 行）同时承载本地 Freeplay、Host 广播、Pure CSR Spectate、本地 Replay 回放、Goal Replay 五种模式。
   * 逻辑高度依赖多重布尔组合（`isPureCsrClient`、`isPureCsrSpectate`、`reconciler.replayPlaybackMode`、`flags.isGoalReplay`、`flags.shouldHoldAccumulator`、`server`）。修改任意一个分支（如 Bug 1 解开 Goal Replay 广播），很容易影响相邻分支（如 Bug 3 的 `interpolator.sync` 时机），这也是 Track A 必须串行推进的根本原因。
2. **NameTag / Team 同步缺乏单一数据源（SSOT）**
   * 全库有 **25+ 个文件** 散落调用 `setNameTagRuntimeState(...)`；与此同时，`hudStore`、`replayViewerStore`、`arena.carTeams`、`physics.setCarTeam` 各自维护一份队伍与车辆列表。
   * 当新入口（如 Dev Sync Replay Watch Together）接入时，只要遗漏其中一环的命令式同步（如只更新了 UI store 没调 `setNameTagRuntimeState`），就会出现 UI 与 3D 世界割裂（Bug 2）。
3. **全局单例与隐式缓存缺乏统一的 `Dispose` 生命周期协议**
   * `RocketSimArenaBarrierController` 在启动期即 eager 预分配 ~194.5 MB replay buffer；`SafeTensorsLoader`（~114 MB）、`wasmSimdMatVec`（~120 MB）、`customVehicleAssets`、`THREE.Cache` 均以模块级单例驻留。
   * `ArenaWorld` 在退出多车模式时仅做 `visible = false` 隐藏而非销毁槽位与非共享几何体。缺乏“模式退出即统一回收”的生命周期契约，导致 Bug 5 需要横跨 15 个文件逐一补洞。
4. **扁平 `Float32Array` 快照导致离散状态与连续插值耦合**
   * 车辆/球/BoostPad 状态打包在同一 `Float32Array` 快照中，`SpectateInterpolationBuffer` 对整块浮点数组做统一插值或覆盖，未对离散布尔态（BoostPad `isActive`）与单调倒计时（`cooldown`）做语义隔离，导致物理前瞻或快照抖动直接表现为模型闪烁（Bug 4）。

---

## 五、推荐合并拓扑图

```text
z/agy/rc/multiplayer-recheck
 ├── [Wave 1] Track A-1 (Bug 1) ──┐
 ├── [Wave 1] Track B   (Bug 2) ──┼──▶ int/wave1 (Bug 1+2 已清零)
 ├── [Wave 1] Track C1  (WASM内存) ──┤         │
 ├── [Wave 1] Track C2  (Three销毁) ─┼─▶ [Wave 2-A/B] Bug 3 (物理前瞻)
 ├── [Wave 1] Track C4  (Tauri关闭) ─┤         │
 └── [Wave 1后] Track C3 (AI/JS堆) ──┘         ▼
                                        [Wave 3-A/B] Bug 4 (BoostPad防抖)
                                               │
                                               ▼
                                      全量门禁: check + repro:dev-broadcast-v3 (0/5)
```
