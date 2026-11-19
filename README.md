# RocketCompanion 多人/回放 5 项缺陷修复排期与分支协作方案

> **基准分支**: `origin/z/agy/rc/multiplayer-recheck`（最新复现提交：`943b749` / `366a686`）
> **验证套件**: `pnpm run check` + `pnpm run repro:dev-broadcast-v3`（`scripts/runDevBroadcastReproV3.mjs`）

---

## 一、核心结论：不能做 5 路并行 + Octopus Merge

**不建议将 5 个 Bug 拆成 5 条平行分支最后做 Octopus Merge**，原因如下：

1. **Bug 1 与 Bug 3 存在强文件冲突与状态机耦合（必冲突）**
   - 两者同时修改以下核心文件：
     - `src/game/pipeline/stages/SimulationStage.ts`
     - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
     - `src/game/pipeline/stages/HostBroadcastPublisher.ts` / `src/network/sync/ReplicationStateInterpolator.ts`
   - Git `octopus` merge 策略**不支持需要人工解决内容冲突的合并**，只要两条分支改到同一文件的相邻区块就会直接中止。
2. **Bug 4 在行为与数据链路上强依赖 Bug 3**
   - `Bug 4`（BoostPad 模型逐帧闪烁）的直接诱因在 `src/entities/BoostPadSystem.ts`（`respawnedAndCollected` / `padCooldownTimers` 在冷却跳变或冻结帧时反复触发 `startCollectAnimation`），而上游输入来自 `ReplicationStateInterpolator.ts`（`SpectateInterpolationBuffer`）与 `ReplayFrameInterpolator.ts` 的快照插值/应用方式。
   - `Bug 3` 改造 `SpectateInterpolationBuffer` 为“缓冲 >= 5 帧驱动确定性 WASM 物理前瞻”后，客户端每帧送入 `BoostPadSystem.update()` 的状态流会发生根本变化。先做 Bug 4 再合 Bug 3 极易返工。
3. **Bug 2 与 Bug 5 与核心模拟管线完全正交（可真并行）**
   - **Bug 2** 集中在 `devSyncChannelHandlers.ts`、`onlineSessionEnvironmentPipeline.ts` 与 `setNameTagRuntimeState` / `arena.carTeams`，与 `SimulationStage` 无交集。
   - **Bug 5** 集中在回放退出时的 JS/Three.js/WASM 资源释放（`FreeplayMode` / `StandaloneReplayViewer`）与 `src-tauri/src/lib.rs`（`WindowEvent::CloseRequested`），与网络/模拟/NameTag/BoostPad 均无交集。

---

## 二、文件重叠与依赖矩阵

| 缺陷 | 主要涉及文件 | 耦合/依赖关系 | 是否可并行 |
| :--- | :--- | :--- | :--- |
| **Bug 1**<br>Goal Replay 客户端静止无动态 | `simulationFrameClassifierPipeline.ts`<br>`HostBroadcastPublisher.ts`<br>`SimulationStage.ts`<br>`simulationNetworkReconcilePipeline.ts`<br>`ReplayFrameInterpolator.ts` | 核心链路起点；修改 `shouldHoldAccumulator`、`isPaused` 广播条件及客户端 `isPureCsrSpectate` 分流 | Wave 1（Track A 第一棒） |
| **Bug 3**<br>无确定性物理前瞻导致卡顿瞬移 | `ReplicationStateInterpolator.ts`<br>`SimulationStage.ts`<br>`simulationNetworkReconcilePipeline.ts` | **强依赖 Bug 1**（共用 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts`） | **必须串行在 Bug 1 之后**（Wave 2） |
| **Bug 4**<br>BoostPad 模型剧烈闪烁 | `BoostPadSystem.ts`<br>`ReplicationStateInterpolator.ts` / `ReplayFrameInterpolator.ts` | **行为强依赖 Bug 3**（快照插值与物理前瞻决定 boostpad 状态跳变特征） | **建议串行在 Bug 3 之后**（Wave 3） |
| **Bug 2**<br>3D NameTag 与 `carTeams` 未同步 | `devSyncChannelHandlers.ts`<br>`onlineSessionEnvironmentPipeline.ts` | 独立（调用 `setNameTagRuntimeState` 并同步 `arena.carTeams`） | **Wave 1 完全并行**（Track B） |
| **Bug 5**<br>高内存占用及退出/关闭未释放 | `FreeplayMode.ts` / `StandaloneReplayViewer` 相关清理链路<br>`src-tauri/src/lib.rs` | 独立（Three.js geometry/texture `dispose()`、WASM/回放帧缓存清理、Tauri 关闭策略） | **Wave 1 完全并行**（Track C） |

---

## 三、最优排期方案：3 线分阶段并行（Wave 1 -> Wave 2 -> Wave 3）

采用 **“独立模块并行 + 核心模拟链路串行推进”**，兼顾最高吞吐与零合并冲突：

### Wave 1（第 1 阶段：3 路同时开工，0 冲突）

* **Track A-1（主干链路负责人）：修复 Bug 1（Goal Replay 客户端静态无动态）**
  - 分支：`fix/bug-1-goal-replay-client-dynamics`（基于 `z/agy/rc/multiplayer-recheck`）
  - 改动点：
    1. Host 侧（`HostBroadcastPublisher.ts` / `SimulationStage.ts`）：在 `isGoalReplay` 期间，即使 `shouldHoldAccumulator = true` 且 `preserveEngineTick: true` 导致 `tickCount` 不变，只要处于 Goal Replay 回放推进状态（非暂停）或帧内容变化，仍需允许 `publishIfReady` 下发快照（或携带 replay 帧序号/时间游标）。
    2. Client 侧（`simulationNetworkReconcilePipeline.ts`）：在 `isPureCsrSpectate && !reconciler.replayPlaybackMode` 分支下，当 `flags.isGoalReplay` 为真时，不再跳过 `replayTimelineController.beginReplaySession` / `appendStreamedReplayFrame(...)`，确保客户端进入 Goal Replay 后能驱动回放时间轴与视角恢复。
  - 验收：`runDevBroadcastReproV3.mjs` 中 `bug-1-goal-replay-client-static-no-dynamics` 的 `reproduced === false`（`distinctBallPositions > 1`、`bufferedFrames` 正常增长、退出后视角正常）。

* **Track B（独立并行）：修复 Bug 2（3D NameTag 与 `arena.carTeams` 不同步）**
  - 分支：`fix/bug-2-replay-nametag-team-sync`（基于 `z/agy/rc/multiplayer-recheck`）
  - 改动点：
    1. 在 `src/game/runtime/devSyncChannelHandlers.ts` 与 `src/game/runtime/onlineSessionEnvironmentPipeline.ts` 更新 `replayViewerStore` / `hudStore` 的同时，同步调用 `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode: true, carEntries })`。
    2. 确保 Client 侧 `arena.setCarTeam(carIndex, team)` / `arena.carTeams` 与 `physics.setCarTeam`、Roster `team` 保持一致。
  - 验收：`runDevBroadcastReproV3.mjs` 中 `bug-2-replay-nametag-and-team-desync` 的 `reproduced === false`（`tags.length === 6`、`teamMismatch === 0`、`tagTextMismatches === 0`）。

* **Track C（独立并行）：修复 Bug 5（内存释放与 Tauri 关闭行为）**
  - 分支：`fix/bug-5-memory-release-and-tauri-close`（基于 `z/agy/rc/multiplayer-recheck`）
  - 改动点：
    1. 回放退出链路（`onReturnToFreeplay()` -> `FreeplayMode` / `StandaloneReplayViewer`）：释放已加载 `.rcreplay` 的帧数组/TypedArray 引用，遍历销毁回放期间创建的 Three.js `BufferGeometry`、`Material`、`Texture`（`dispose()`），并在可回收边界清理/收缩临时 WASM 缓冲。
    2. Tauri 侧（`src-tauri/src/lib.rs`）：调整 `WindowEvent::CloseRequested` 策略（区分“明确退出/关闭销毁资源”与“最小化到托盘”，避免无条件 `api.prevent_close(); window.hide();` 导致 WebView/Three.js/WASM 永久驻留）。
    3. 基线内存评估：排查启动即占 `wasmMB: 259` 的预分配来源（如线性内存初始页/多实例残留）。
  - 验收：`runDevBroadcastReproV3.mjs` 中 `bug-5-high-memory-and-unreleased-on-exit` 的 `retainedGeometries <= 0`、`retainedJsMB` 显著回落、`tauriCloseHidesWindow` 符合新关闭语义。

---

### Wave 2（第 2 阶段：Track A 串行推进 Bug 3；Track B/C 合入主干）

* **合入动作**：
  - 将 **Bug 1**、**Bug 2**、**Bug 5** 按序通过测试后合入集成分支（此时三者文件几乎完全正交，**只有 `[Bug 1, Bug 2, Bug 5]` 这三个已完成且正交的分支之间才适合用 Octopus Merge 或依次 fast-forward/rebase**）。
* **Track A-2（基于 Bug 1 合入后的最新代码）：修复 Bug 3（确定性 WASM 物理前瞻）**
  - 分支：`fix/bug-3-deterministic-physics-lookahead`（基于合入 Bug 1 后的集成分支）
  - 改动点：
    1. `src/network/sync/ReplicationStateInterpolator.ts`（`SpectateInterpolationBuffer`）：引入缓冲前瞻阈值（如 `>= 5` 帧），在有足够未来帧缓冲时，不再仅靠 `interpolator.sync(timestamp)` 静止挂起或单靠线性/SLERP 插值硬拉。
    2. `src/game/pipeline/stages/SimulationStage.ts` & `simulationNetworkReconcilePipeline.ts`：在 `isPureCsrClient` 下结合缓冲帧与控制输入/物理步进驱动本地确定性 WASM 物理前瞻（使 `tickCount` 连续推进而非 `distinctTicks === 1`），消除 `frozenBallFrames >= 3 && maxBallStepUu > 200` 的卡顿瞬移。
  - 验收：`runDevBroadcastReproV3.mjs` 中 `bug-3-replay-choppy-motion-no-physics-lookahead` 的 `reproduced === false`。

---

### Wave 3（第 3 阶段：Track A 收尾 Bug 4 + 全量回归）

* **Track A-3（基于 Bug 3 合入后的代码）：修复 Bug 4（BoostPad 模型闪烁）**
  - 分支：`fix/bug-4-boostpad-flicker`（基于合入 Bug 3 后的集成分支）
  - 改动点：
    1. `src/entities/BoostPadSystem.ts`：加固状态跳变判定（`respawnedAndCollected` 与 `isBufferFrozen` 逻辑），避免因网络快照微抖、插值帧冷却倒计时轻微回弹或重复应用同帧状态导致 `startCollectAnimation` 被反复触发、`pad.full.visible` 逐帧翻转。
    2. 检查 `ReplayFrameInterpolator.ts` / `ReplicationStateInterpolator.ts`：确保 BoostPad 离散状态（`isActive`）与连续冷却（`cooldown`）在插值/前瞻切换时不发生虚假跳变。
  - 验收：`runDevBroadcastReproV3.mjs` 中 `bug-4-boostpad-model-flicker` 的 `reproduced === false`（`maxSinglePadFlips < 2 && totalPadFlips < 8`）。

---

## 四、工具链与环境检查清单（下发执行前必看）

在当前 Linux 验证环境中实测结果如下，下发给开发/CI 环境时请同步对齐：

1. **包管理器与前端工具链**：
   - 仓库 `package.json` 强制要求 `pnpm`（`preinstall: npx only-allow pnpm`）。若环境未安装，需执行 `npm install -g pnpm` 并运行 `pnpm install`。
   - Node.js（v22.x，支持 `--experimental-strip-types`）与 `google-chrome`（供 CDP E2E 测试 `runDevBroadcastReproV3.mjs` 使用）为必需依赖。
2. **Rust / Tauri 工具链（Track C 必需）**：
   - 修改 `src-tauri/src/lib.rs` 后，纯 Web E2E 无法编译验证 Rust 侧改动；负责 **Bug 5** 的开发者本机或 CI 必须具备 `cargo` / `rustc` 及 Tauri 系统依赖，执行 `cargo check` / `pnpm run tauri:build` 验证。
3. **测试回放文件放置**：
   - 运行 `pnpm run repro:dev-broadcast-v3` 前，需将 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` 置于 `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`、`/workspace/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`，或通过环境变量 `RC_REPLAY_FILE` 指定路径。
4. **统一门禁**：
   - 每个分支合入前必须通过：
     ```bash
     pnpm run check
     pnpm run repro:dev-broadcast-v3
     ```
   - 最终全部合入后，跑一次全量 `repro:dev-broadcast-v3`，确认 `reproducedCount === 0/5`。
