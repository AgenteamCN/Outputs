# RocketCompanion：先清架构还是直接修 Bug？—— Wave 0 架构重构前置计划 + 瘦身版 Wave 1–3 下发方案

> 基线分支：`origin/z/agy/rc/multiplayer-recheck`（HEAD: `943b749`）
> 核心原则：**拒绝在“布尔地狱 + 散落命令式同步 + 无生命周期单例”上继续打补丁（雕花）**；但也不搞脱离测试基线的“全仓大重写”。采用 **Wave 0（精准架构止血前置波）→ 瘦身版 Wave 1–3（基于清晰契约修 Bug）** 的两段式工程方案，可直接下发给研发/Agent 团队执行。

---

## 一、战略决策：为什么选择「Wave 0 精准重构前置」而非「全仓重写」或「直接硬修」？

| 方案 | 优点 | 致命问题 | 结论 |
| :--- | :--- | :--- | :--- |
| **A. 直接按原 Wave 1–3 硬修** | 看似少一步 | 在 `SimulationStage` 的布尔泥潭和 25 处散落 `setNameTagRuntimeState` 上打补丁，修 Bug 1 容易炸 Bug 3，修 Bug 3 必炸 Bug 4，后续新模式继续复发 | ❌ 不可持续（雕花） |
| **B. 全仓推倒重来** | 理论上最干净 | 仓库有 1,425 个 TS 文件且已通过 `radar.ts`（0 循环依赖、0 越级引用、严格体积上限），问题**不在全仓**，只集中在 4 个局部架构坏味道；全仓重写会直接摧毁现有 E2E 基线 | ❌ 成本与风险失控 |
| **C. Wave 0 精准架构前置 + 瘦身版 Wave 1–3** | **先清理 4 个病灶的抽象契约，再填 Bug 修复逻辑**；每一步都有 `pnpm run check` 与现有测试套件兜底 | 多 1 个前置 Wave，但大幅压缩 Wave 1–3 的联调与返工时间 | ✅ **最优解** |

> **关键认知**：`RocketCompanion` 并非整体烂掉（`radar.ts` 门禁已经保证了模块边界与单文件体积），真正导致 Bug 1–5 难修的是 **4 个高耦合局部病灶**。Wave 0 只针对这 4 个病灶做“手术刀式”重构。

---

## 二、Wave 0：架构清理前置波（4 路并行，纯重构 + 契约建立）

> **Wave 0 总门禁**：每条轨道合入前必须保持 **零行为退化**——通过 `pnpm run check`（`radar` + `typecheck`）、`pnpm run test:multiplayer-suite`、`pnpm run repro:dev-broadcast-v2`。

### Arch-0A：模拟管线显式状态机（消除布尔组合地狱）
* **解决的架构病灶**：`SimulationStage.ts`（468L）与 `simulationNetworkReconcilePipeline.ts`（281L）同时用 `isPureCsrClient`、`isPureCsrSpectate`、`reconciler.replayPlaybackMode`、`flags.isGoalReplay`、`flags.shouldHoldAccumulator`、`server` 等 6+ 个布尔值隐式组合出 5 种运行模式，导致分支极其脆弱。
* **重构动作**：
  1. 在 `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts` 中定义**显式互斥枚举** `SimulationDriveMode`：
     - `'local-freeplay'`
     - `'host-authoritative'`
     - `'host-goal-replay'`
     - `'client-csr-spectate'`
     - `'client-goal-replay'`
     - `'local-replay-playback'`
  2. 为每种模式派生显式策略对象 `SimulationDrivePolicy`：
     ```ts
     interface SimulationDrivePolicy {
       mode: SimulationDriveMode;
       shouldStepLocalPhysics: boolean;
       shouldHoldAccumulator: boolean;
       shouldBroadcastSnapshot: boolean; // 显式声明，不再用 !isPaused && tickChanged 隐式推断
       shouldRunLookahead: boolean;
       shouldFeedReplayTimeline: boolean;
     }
     ```
  3. 将 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 中的多重布尔 `if/else` 全部收口为基于 `policy.mode` / `policy.*` 的扁平分发。
* **直接收益**：Wave 1（Bug 1）和 Wave 2（Bug 3）不再互相踩踏，各自只需修改对应 `mode` 的策略表与独立 handler。

### Arch-0B：Roster / Team / NameTag 单一数据源投影（SSOT）
* **解决的架构病灶**：`setNameTagRuntimeState` 散落在全库 **25+ 个文件** 中命令式调用，`hudStore`、`replayViewerStore`、`arena.carTeams`、`physics.setCarTeam` 各自为政。
* **重构动作**：
  1. 新建/收敛统一的协调投影器 `src/game/runtime/MatchRosterProjection.ts`（或收敛至 `OnlineRosterSyncCoordinator.ts`），作为**唯一允许批量同步车辆队伍与 NameTag 的出口**：
     - 输入：`RosterSnapshot`（carIndex, name, team, isLocal, activeCount）
     - 原子输出：同步更新 `physics.setCarTeam(...)`、`arena.setCarTeam(...)`、`setNameTagRuntimeState(...)`、`hudStore` / `replayViewerStore`。
  2. 率先将 `devSyncChannelHandlers.ts`、`onlineSessionEnvironmentPipeline.ts` 接入该投影器（后续渐进迁移其余 20+ 处）。
* **直接收益**：Wave 1（Bug 2）在接入投影器后即天然解决，且彻底杜绝后续新模式漏调 `setNameTagRuntimeState`。

### Arch-0C：资源生命周期总线与延迟分配契约（Disposable Registry）
* **解决的架构病灶**：WASM 194.5 MB replay buffer 启动即 eager 分配；AI `SafeTensorsLoader`（~114 MB）、`wasmSimdMatVec`（~120 MB）、`customVehicleAssets`、`THREE.Cache` 均为无卸载协议的模块级单例；`ArenaWorld` 退出仅隐藏车辆不销毁槽位。
* **重构动作**：
  1. 定义轻量级生命周期接口 `RuntimeDisposableResource`（`id`, `scope: 'session' | 'replay' | 'ai'`, `dispose(): void`）及全局注册表 `src/game/runtime/RuntimeResourceRegistry.ts`。
  2. 确立两项硬约束：
     - **延迟分配（Lazy Allocation）**：重型缓冲区（如 `initReplayBuffer`）严禁在 Boot/Constructor 阶段无条件初始化，必须由 `isRecordingFullReplay` 或首次录制请求按需触发。
     - **统一退出回收**：`MatchModeTransitionBridge.returnToFreeplay()` 与 `applyCleanFreeplaySubsystemsReset()` 统一调用 `RuntimeResourceRegistry.disposeScope('replay' | 'ai')`。
  3. 为 WASM 回收（`reclaimWasmMemoryIfGrown`）、Arena 车辆槽位销毁（`removeCar` + 非共享几何体回收）、AI 缓存清理（`resetWasmSimdMatVec` / `clearSafeTensorsModelCache`）预留标准注册入口。
* **直接收益**：Bug 5 的 C1/C2/C3 不再是“到处打补丁”，而是各自向 `RuntimeResourceRegistry` 注册一个干净的 `Disposable` 实现。

### Arch-0D：快照离散字段与连续插值语义隔离
* **解决的架构病灶**：`SpectateInterpolationBuffer`（`ReplicationStateInterpolator.ts`）把包含离散布尔态（BoostPad `isActive`）和连续物理量的 `Float32Array` 混在一起做线性插值/覆盖，导致前瞻切换或抖动时触发虚假边缘跳变。
* **重构动作**：
  1. 在 `ReplicationStateInterpolator.ts` / `ReplayFrameInterpolator.ts` 中显式声明 **BoostPad 离散字段偏移表**（`isActive` 走最近邻/阶跃取值，严禁线性插值；`cooldown` 增加单调性/跨帧跳变保护）。
  2. 明确 `SpectateInterpolationBuffer` 的输出契约：`sample()` 必须同时给出 `frameCursor / isDuplicateFrame` 标记，供下游渲染系统识别“重复帧/静止帧”。
* **直接收益**：Wave 3（Bug 4）的闪烁根因在插值层即被切断，`BoostPadSystem.ts` 只需做极简防抖。

---

## 三、Wave 0 之后：瘦身版 Wave 1–3 执行计划

完成 Wave 0 并合入主集成分支后，原 5 个 Bug 的修复复杂度大幅下降，按以下顺序快速推进：

### Wave 1（3 路并行，基于 Wave 0 底座）
1. **Track A-1（Bug 1：Goal Replay 客户端动态）**
   * 在 `Arch-0A` 的 `SimulationDrivePolicy` 中为 `'host-goal-replay'` 开启 `shouldBroadcastSnapshot = true`（携带 replay 帧游标），为 `'client-goal-replay'` 开启 `shouldFeedReplayTimeline = true`。
   * 验收：`bug-1-goal-replay-client-static-no-dynamics.reproduced === false`。
2. **Track B（Bug 2：NameTag 与队伍同步）**
   * 确认 `devSyncChannelHandlers.ts` 与 `onlineSessionEnvironmentPipeline.ts` 全量走 `Arch-0B` 的 `MatchRosterProjection` 即可。
   * 验收：`bug-2-replay-nametag-and-team-desync.reproduced === false`。
3. **Track C（Bug 5：拆为 4 个 Disposable 子任务并行）**
   * **C1（WASM 内存）**：接入 `Arch-0C` 延迟分配 + `reclaimWasmMemoryIfGrown`（`bootMem.wasmMB < 256`, `retainedWasmMB <= 0`）。
   * **C2（Three.js 槽位与几何体）**：`ArenaWorld.removeCar` + `collectReferencedGeometries` 注册到 `Arch-0C`（`retainedGeometries <= 0`）。
   * **C3（AI / JS Heap 清理）**：`SafeTensorsLoader` / `wasmSimdMatVec` / `ringBuffer` 注册到 `Arch-0C`（`retainedJsMB` 回落）。
   * **C4（Tauri 关闭语义，独立 Rust 轨道）**：修改 `src-tauri/src/lib.rs` `WindowEvent::CloseRequested`，在具备 `cargo`/`rustc` 的 CI 或本地环境验证（`tauriCloseHidesWindow === false`）。

### Wave 2（基于 Wave 1 合入后，2 步推进 Bug 3）
* **2-A**：基于 `Arch-0A` 的 `'client-csr-spectate'` 模式，在 `SpectateInterpolationBuffer` 增加 `>= 5` 帧前瞻窗口与游标状态机。
* **2-B**：在 `SimulationStage` 中为 `'client-csr-spectate'` 开启 `shouldRunLookahead = true`，驱动本地 WASM 物理连续步进与快照收敛。
* 验收：`bug-3-replay-choppy-motion-no-physics-lookahead.reproduced === false`。

### Wave 3（基于 Wave 2 合入后，收尾 Bug 4 + 全量门禁）
* **3-A**：利用 `Arch-0D` 的 `isDuplicateFrame` 标记与离散字段阶跃输出，校验前瞻切换帧的 BoostPad 状态一致性。
* **3-B**：在 `BoostPadSystem.ts` 增加重复帧跳过与冷却微抖阈值保护。
* 验收：`bug-4-boostpad-model-flicker.reproduced === false`，最终全量运行 `pnpm run check && pnpm run repro:dev-broadcast-v3` 确认 `reproducedCount === 0/5`。

---

## 四、工具链与下发检查清单

* **Web / TS / CDP 环境（Wave 0 + Wave 1 C1/C2/C3 + Wave 2 + Wave 3）**：
  - 必需：`node v22.x`、`pnpm`、`google-chrome`、测试回放文件 `rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`。
  - 每 PR 门禁：`pnpm run check` + `pnpm run test:multiplayer-suite`（Wave 0 必跑）+ `pnpm run repro:dev-broadcast-v3`。
* **Rust / Tauri 环境（仅 Wave 1 C4）**：
  - 必需：`rustc` / `cargo` + C 链接器 + Tauri 系统依赖，执行 `cargo check` / `pnpm run tauri:build`。
