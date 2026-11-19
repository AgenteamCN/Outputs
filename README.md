# Deep Scan 结论与全新执行计划（基于 `746126d` + `0bc5c69`）

## 一、核心结论（TL;DR）

1. **两个 Commit 能否 Merge？—— ✅ 可以直接无冲突自动合并（Clean Merge）**
   - `746126d7931c4ab29c50876e9efbd2c440046361`（`radarArchDebtGuards.ts`）**已在 `origin/z/agy/rc/multiplayer-recheck` 的 `HEAD`**。
   - `0bc5c69034f4df560636be9b2edccb9be97f99e4`（`origin/fix/bug-5-memory-release-and-tauri-close`）基于 `943b749` 分叉，实测 `git merge --no-commit --no-ff 0bc5c69` **0 冲突自动合入**。
   - 合并后实测门禁：
     - `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1425/1425 TS，0 循环依赖，0 越级引用，0 类型错误）
     - `pnpm run radar:audit`：**GREEN**（`NameTagScatter=15/15 | BoolHotspots=9 | UnscopedCaches=3`）
     - `pnpm run test:multiplayer-suite`：**24/24 PASS**

2. **对 Architecture A / B / C / D 的影响？—— ✅ 无冲突、完全兼容，且大幅压缩了 Wave 1 工作量**
   - `0bc5c69` **未触碰** Arch-0A（`SimulationStage.ts` / `simulationNetworkReconcilePipeline.ts`）、Arch-0B（`devSyncChannelHandlers.ts` / `OnlineRosterSyncCoordinator.ts`）以及 Arch-0D（`ReplicationStateInterpolator.ts` / `BoostPadSystem.ts`）的核心文件，**Architecture A/B/C/D 仍可按原架构设计正常推进**。
   - 同时，`0bc5c69` **提前完成了 Bug 5 的大部分实现（C1 WASM 延迟分配与回收、C2 Three.js 槽位与几何体销毁、C4 Tauri 关闭语义）**，使 Wave 1 进一步瘦身，只需补齐 **C3（AI/全局单例缓存回收）** 并把 `0bc5c69` 的命令式清理收口进 **Arch-0C `RuntimeResourceRegistry`**。

3. **⚠️ 深度扫描发现的高危风险（Volume 18KB 悬崖）**
   - 合入 `0bc5c69` 后，以下两个文件已紧贴 `radar.ts` 的 `18.0 KB (18,432 B)` 硬上限：
     - `src/game/modes/FreeplayMode.ts`：**18,424 B**（仅剩 **8 字节**余量）
     - `src/replay/StandaloneReplayViewerController.ts`：**18,416 B**（仅剩 **16 字节**余量）
   - 若后续在 Wave 0/1 直接往这两个文件加代码，会立刻触发 `radar` 硬报错。因此 **Arch-0C 必须将 `FreeplayMode.ts` 中的命令式回收代码抽离至 `RuntimeResourceRegistry`**，顺便为其腾出体积余量。

---

## 二、深度扫描明细（Deep Scan Findings）

### 1. Commit `0bc5c69` 已覆盖 vs 未覆盖清单（对比 Bug 5 C1–C4 与 Arch-0C）

| 子项 | `0bc5c69` 实装状态 | 代码位置与实证 | 遗留缺口（需 Arch-0C / Wave 1 补齐） |
|---|---|---|---|
| **C1: WASM 内存** | ✅ 已实现 | `RocketSimArenaBarrierController.ts` 移除构造期 `mod._physics_initReplayBuffer?.()`，新增 `reclaimWasmMemoryIfGrown()` + `clearCachedRocketSimWasmModule()` | 退出清理仍由 `FreeplayMode.ts` 命令式调用，未接入 `RuntimeResourceRegistry` |
| **C2: Three.js / 槽位** | ✅ 已实现 | `arenaVehicleSlotPipeline.ts` 新增 `keepGeometryUuids` 活跃槽位引用检查 + `disposeAndRemoveRegistrySlot`；`freeplayKickoffAndArenaResetPipeline.ts` 与 `ReplayViewerCausalSyncController.ts` 改用 `vehicleRegistry.removeCar(i)` 并调用 `renderer.renderLists.dispose()`、`replayRingBuffer.clear()` | 同上，命令式散落调用 |
| **C3: AI / JS 单例缓存** | ❌ 未覆盖 | `radar:audit` 仍报 `UnscopedCaches=3`：`src/ai/experimental/SafeTensorsLoader.ts`、`src/entities/vehicle/vehiclePaintMaterial.ts`、`src/replay/ReplayDirectorCsvParser.ts`；`wasmSimdMatVec` 与 `THREE.Cache` 无清理钩子 | **Wave 1 主力**：接入 `RuntimeResourceRegistry` 并实现 `disposeScope('replay' \| 'ai')` |
| **C4: Tauri 关闭语义** | ✅ 已实现 | `src-tauri/src/lib.rs` 将 `WindowEvent::CloseRequested` 改为 `WindowEvent::Destroyed`，并在 `RunEvent::ExitRequested` 保留 `tray_only` 判断 | 需在具备 `cargo/rustc` 的环境跑 `cargo check` 验证 |

### 2. `746126d` 债务雷达（`radarArchDebtGuards.ts`）基线核对

- **`NameTagScatter = 15 / 15`**（棘轮上限 `15`）：15 个引擎文件仍直接调用 `setNameTagRuntimeState(...)`（含 `FreeplayMode.ts`、`devSyncChannelHandlers.ts`、`OnlineRosterSyncCoordinator.ts` 等），Arch-0B 完成后将下调 `HARD_ENGINE_NAMETAG_SCATTER_LIMIT` 至 `1`。
- **`BoolHotspots = 9`**（`DistinctFlags <= 15`, `CompoundIfs <= 12`）：核心目标 `simulationNetworkReconcilePipeline.ts`（12 flags / 10 compoundIfs）与 `SimulationStage.ts`（10 flags / 3 compoundIfs）保持原样，Arch-0A 可直接重构。
- **`UnscopedCaches = 3`**：对应 Arch-0C + Bug 5 C3，目标清零至 `0`。

### 3. 本地工具链与环境验证

- 已补充安装 `pnpm`（`12.21.0`）并完成 `pnpm install --frozen-lockfile`。
- 当前沙箱已验证可执行：`pnpm run check`、`pnpm run radar:audit`、`pnpm run test:multiplayer-suite`（24/24 PASS）。
- 备注：`src-tauri` Rust 编译链（`cargo`/`rustc`）不在当前 Web 沙箱内，C4 需在 Rust/Tauri CI 环境跑 `cargo check`。

---

## 三、全新执行计划（基于 `z/agy/rc/multiplayer-recheck` + Merge `0bc5c69`）

### Phase 0：立即合入前置分支
1. 将 `origin/fix/bug-5-memory-release-and-tauri-close`（`0bc5c69`）合入 `z/agy/rc/multiplayer-recheck`。
2. 跑门禁确认基线全绿：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite`。

### Wave 0：架构清理前置波（4 路并行，纯重构 + 契约建立）

- **Arch-0A：模拟管线显式状态机（`SimulationDriveMode` + `SimulationDrivePolicy`）**
  - 在 `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts` 定义互斥枚举 `SimulationDriveMode`（`local-freeplay` / `host-authoritative` / `host-goal-replay` / `client-csr-spectate` / `client-goal-replay` / `local-replay-playback`）与策略表 `SimulationDrivePolicy`。
  - 收口 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 的多重布尔分支，**无损保留 `476b783`（Bug 1）已合入的 Goal Replay 流式帧游标与 `appendStreamedReplayFrame` 逻辑**。
- **Arch-0B：Roster / Team / NameTag 单一数据源投影（SSOT）**
  - 新建 `src/game/runtime/MatchRosterProjection.ts`，以 `5fbc87d`（Bug 2）抽出的 `syncReplayRosterTeamsAndNameTags` 为核心，统一接管 `physics.setCarTeam`、`arena.setCarTeam`、`setNameTagRuntimeState`、`hudStore` / `replayViewerStore`。
  - 率先接管 `devSyncChannelHandlers.ts` 与 `onlineSessionEnvironmentPipeline.ts`，并迁移其余 13 处 `[S]` 调用点，将 `HARD_ENGINE_NAMETAG_SCATTER_LIMIT` 从 `15` 下调至 `1`。
- **Arch-0C：资源生命周期总线（`RuntimeResourceRegistry.ts`）+ 18KB 体积解套**
  - 新建 `src/game/runtime/RuntimeResourceRegistry.ts`（`RuntimeDisposableResource` + `scope: 'session' | 'replay' | 'ai'`）。
  - 将 `0bc5c69` 合入后散落在 `FreeplayMode.ts`、`freeplayKickoffAndArenaResetPipeline.ts`、`ReplayViewerCausalSyncController.ts` 的命令式清理（`reclaimWasmMemoryIfGrown`、`clearReplayBuffer`、`renderLists.dispose()`、`replayRingBuffer.clear()`）**收口注册到 `RuntimeResourceRegistry`**，同时将 `FreeplayMode.ts` 从 `18,424 B` 悬崖降回安全体积。
  - 为 `SafeTensorsLoader.ts`、`vehiclePaintMaterial.ts`、`ReplayDirectorCsvParser.ts`、`wasmSimdMatVec`、`THREE.Cache` 预留/注册标准 `dispose` 钩子。
- **Arch-0D：快照离散字段与连续插值语义隔离**
  - 在 `ReplicationStateInterpolator.ts` / `ReplayFrameInterpolator.ts` 显式声明 BoostPad 离散字段偏移表（`isActive` 走阶跃/最近邻，严禁线性插值；`cooldown` 增加单调/跨帧跳变保护）。
  - `SpectateInterpolationBuffer.sample()` 显式输出 `frameCursor` 与 `isDuplicateFrame` 标记。
- **Wave 0 总门禁**：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite && pnpm run repro:dev-broadcast-v2`。

### Wave 1：回归验收 + Bug 5 C3 收尾（大幅瘦身）
- **Bug 1 / Bug 2 / Bug 5 (C1, C2, C4)**：转为基于 Wave 0 底座的回归验收（代码均已合入，不重复开发；C4 在 Rust CI 补跑 `cargo check`）。
- **Bug 5 C3（唯一新开发项）**：将 `SafeTensorsLoader`（~114 MB）、`wasmSimdMatVec`（~120 MB）、`vehiclePaintMaterial`、`ReplayDirectorCsvParser`、`THREE.Cache` 接入 Arch-0C `RuntimeResourceRegistry.disposeScope('replay' | 'ai')`，清零 `radar:audit` 的 `UnscopedCaches=3 -> 0`，并验证 `retainedJsMB` 回落。

### Wave 2：Bug 3（物理前瞻 >= 5 帧）
- **2-A**：基于 Arch-0A `'client-csr-spectate'` 模式，在 `SpectateInterpolationBuffer` 增加 `>= 5` 帧前瞻窗口与游标状态机。
- **2-B**：在 `SimulationDrivePolicy` 为 `'client-csr-spectate'` 开启 `shouldRunLookahead = true`，驱动本地 WASM 物理连续步进与快照收敛。
- **验收**：`bug-3-replay-choppy-motion-no-physics-lookahead.reproduced === false`。

### Wave 3：Bug 4（BoostPad 防抖）+ 全量收官
- **3-A / 3-B**：利用 Arch-0D 输出的 `isDuplicateFrame` 与离散 `isActive` 阶跃值，在 `BoostPadSystem.ts` 增加重复帧跳过与冷却微抖阈值保护。
- **终验门禁**：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite && pnpm run repro:dev-broadcast-v3`（确认 `reproducedCount === 0/5`）。
