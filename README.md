# RocketCompanion 深度扫描与最新执行计划（基于 `origin/z/agy/rc/multiplayer-recheck` + `0bc5c69`）

## 一、核心结论

1. **两个 Commit 能否 Merge？—— ✅ 可以直接无冲突合并**
   - `746126d7931c4ab29c50876e9efbd2c440046361`（`chore(radar): add deterministic arch-debt ratchets and audit guards for Wave 0`）**已经是 `origin/z/agy/rc/multiplayer-recheck` 的当前 `HEAD`**（位于 Bug 1 `476b783` 与 Bug 2 `5fbc87d` 合入之后）。
   - `0bc5c69034f4df560636be9b2edccb9be97f99e4`（`fix(memory,tauri): release replay WASM/JS/Three.js resources on exit and allow window close`，所在分支 `origin/fix/bug-5-memory-release-and-tauri-close`）与 `746126d` 共享同一祖先 `943b749`。
   - **实测合并验证结果**：
     - `git merge-tree --write-tree 746126d 0bc5c69`：**退出码 `0`（20 个文件零文本冲突）**。
     - 实际执行 `git merge --no-edit 0bc5c69` 后运行门禁：
       - `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1425/1425 TS 文件，0 循环依赖，0 越级引用，0 值重导出，体积门禁通过）。
       - `pnpm run radar:audit`：**GREEN**（`NameTagScatter=15/15`、`BoolHotspots=9`、`UnscopedCaches=3`，均未恶化）。
       - `pnpm run test:multiplayer-suite`：**24/24 PASS**。

2. **对后续 Architecture A / B / C / D 计划是否有影响？—— ✅ 架构底座无冲突，Wave 0 仍按 A/B/C/D 4 路并行；Wave 1 大幅瘦身**
   - **Arch-0A / 0B / 0D**：与 `0bc5c69` **零交集**，完全按原计划推进。
   - **Arch-0C**：`0bc5c69` 已经落地了 Bug 5 中 **C1（WASM 懒加载 + 堆回收）**、**C2（Arena 槽位与几何体销毁）** 与 **C4（Tauri `CloseRequested` 修复）** 的具体实现，但目前以命令式分散在 `FreeplayMode.ts`、`freeplayKickoffAndArenaResetPipeline.ts`、`ReplayViewerCausalSyncController.ts` 三处，且 `radar:audit` 命中的 3 个 `[C]` 无卸载缓存单例尚未接入。**Arch-0C 正好将这些已验证的回收原语收口到 `RuntimeResourceRegistry.ts`，并补齐 C3（AI / JS Cache 清理）。**
   - **风险提示**：`src/replay/StandaloneReplayViewerController.ts` 在合入 `0bc5c69` 后体积已达 **`18KB / 514L`**（贴近 18KB 硬上限），后续 Wave 0–3 严禁继续向该文件直接堆逻辑。

---

## 二、深度扫描明细（Merged State: `746126d` + `0bc5c69`）

### 1. 工具链与门禁基线
- **已安装工具链**：Node `v22.23.5`、`pnpm v12.21.0`、`google-chrome`（`/usr/bin/google-chrome`）、`pnpm install` 依赖已就绪。
- **缺失环境**：容器内无 `cargo` / `rustc`（仅影响 C4 Rust 侧 `src-tauri/src/lib.rs` 编译验证，需在本地/CI Rust 环境验证）；`pnpm run repro:dev-broadcast-v3` 在当前无 GPU 容器中于 `Timeout waiting for Host boot` 超时，需在具备 WebGL 渲染能力的目标 runner 执行完整 V3 回归。

### 2. `0bc5c69`（Bug 5）已覆盖范围 vs 剩余架构缺口
| 子任务 | `0bc5c69` 已完成内容 | 剩余缺口（交由 Arch-0C / Wave 1 收口） |
|---|---|---|
| **C1（WASM 内存）** | `RocketSimArenaBarrierController.ts` 移除启动期 `_physics_initReplayBuffer()`，新增 `reclaimWasmMemoryIfGrown()` + `clearCachedRocketSimWasmModule()`；`RocketSimWasmStateAndReplayBase.ts` 在 buffer 未初始化时跳过录制 | 退出回收调用分散在 `FreeplayMode.ts` 等多处，需统一注册为 `RuntimeResourceRegistry` 的 `'replay'` scope 资源 |
| **C2（Three.js 槽位与几何体）** | `arenaVehicleSlotPipeline.ts` 新增 `keepGeometryUuids` 引用计数保护；`freeplayKickoffAndArenaResetPipeline.ts` 与 `ReplayViewerCausalSyncController.ts` 改为调用 `vehicleRegistry.removeCar(i)` + `renderer.renderLists.dispose()`；FlipReset/SpeedTrail/BoostEmitter 补齐 dispose | 需封装为统一 Disposable 注册入口，避免新模式退出时再次退化为 `visible = false` |
| **C3（AI / JS Heap 缓存）** | `StandaloneReplayViewerController.ts` 清空 `resolvedRoster` | `radar:audit` 仍有 3 处 `[C]` 无 dispose 模块级 Map/Set：`src/ai/experimental/SafeTensorsLoader.ts`（297L）、`src/entities/vehicle/vehiclePaintMaterial.ts`（337L）、`src/replay/ReplayDirectorCsvParser.ts`（508L），以及 `wasmSimdMatVec` / `THREE.Cache` 需接入 Arch-0C |
| **C4（Tauri 关闭语义）** | `src-tauri/src/lib.rs` 将 `WindowEvent::CloseRequested` 改为销毁窗口，仅在 `tray_only` 时通过 `RunEvent::ExitRequested` 保留托盘 | Rust 代码已到位，仅需在含 `cargo/rustc` 的 CI/本地环境执行 `cargo check` / `pnpm run tauri:build` 验收 |

### 3. 四大架构病灶当前精确基线（`radar:audit`）
- **Arch-0A（多布尔组合爆炸，`BoolHotspots=9`）**：
  - 核心目标：`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`（308L，`DistinctFlags=12, CompoundIfs=10`）、`src/game/pipeline/stages/SimulationStage.ts`（473L，`DistinctFlags=10, CompoundIfs=3`）、`src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`（145L）。
- **Arch-0B（NameTag / Team SSOT 散落，`NameTagScatter=15/15`）**：
  - 15 个引擎文件直接调用 `setNameTagRuntimeState(...)`（含 `devSyncChannelHandlers.ts`、`FreeplayMode.ts`、`OnlineRosterSyncCoordinator.ts`、`ReplayViewerCausalSyncController.ts` 等）。
- **Arch-0C（无生命周期协议缓存，`UnscopedCaches=3`）**：
  - `SafeTensorsLoader.ts`、`vehiclePaintMaterial.ts`、`ReplayDirectorCsvParser.ts`。
- **Arch-0D（离散/连续快照耦合）**：
  - `src/network/sync/ReplicationStateInterpolator.ts`（418L）、`src/replay/ReplayFrameInterpolator.ts`（445L）、`src/entities/BoostPadSystem.ts`（475L）。

---

## 三、全新更新后执行计划（立即执行版）

### Pre-Wave 0：合入 `0bc5c69`
1. 将 `0bc5c69034f4df560636be9b2edccb9be97f99e4`（`origin/fix/bug-5-memory-release-and-tauri-close`）合入 `origin/z/agy/rc/multiplayer-recheck`。
2. 门禁：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite`。

### Wave 0：架构清理前置波（Arch-0A / 0B / 0C / 0D 4 路并行，零行为退化）
- **Arch-0A（模拟管线显式状态机）**：
  - 在 `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts` 定义 `SimulationDriveMode`（6 种互斥模式）与 `SimulationDrivePolicy`。
  - 收口 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 的布尔分支，**无损保留 `476b783`（Bug 1）的 Goal Replay 流式帧游标与 `appendStreamedReplayFrame` 逻辑**至 `'host-goal-replay'` / `'client-goal-replay'` 策略。
- **Arch-0B（Roster / Team / NameTag SSOT）**：
  - 新建 `src/game/runtime/MatchRosterProjection.ts`，以 `5fbc87d` 抽出的 `syncReplayRosterTeamsAndNameTags` 为核心收敛原子投影（`physics.setCarTeam` + `arena.setCarTeam` + `setNameTagRuntimeState` + store）。
  - 优先接管 `devSyncChannelHandlers.ts` 与 `onlineSessionEnvironmentPipeline.ts`，逐步迁移其余 13 处 `[S]` 调用点，并将 `scripts/radar/radarArchDebtGuards.ts` 中的 `HARD_ENGINE_NAMETAG_SCATTER_LIMIT` 从 `15` 下调至 `1`。
- **Arch-0C（资源生命周期总线）**：
  - 新建 `src/game/runtime/RuntimeResourceRegistry.ts`（`register` / `disposeScope('session' | 'replay' | 'ai')`）。
  - 将 `0bc5c69` 已实现的 `clearReplayBuffer` / `reclaimWasmMemoryIfGrown` / `vehicleRegistry.removeCar` / `renderLists.dispose` 注册进总线，在 `MatchModeTransitionBridge.returnToFreeplay()` 与 `applyCleanFreeplaySubsystemsReset()` 统一触发。
  - 为 `SafeTensorsLoader.ts`、`vehiclePaintMaterial.ts`、`ReplayDirectorCsvParser.ts`（以及 `wasmSimdMatVec` / `THREE.Cache`）补齐 `dispose/clear/reset` 注册（将 `UnscopedCaches` 降至 `0`）。
- **Arch-0D（离散字段与连续插值隔离）**：
  - 在 `ReplicationStateInterpolator.ts` 与 `ReplayFrameInterpolator.ts` 显式声明 BoostPad 离散字段偏移表（`isActive` 阶跃/最近邻取值，`cooldown` 单调跳变保护），并在 `SpectateInterpolationBuffer.sample()` 输出 `frameCursor` 与 `isDuplicateFrame`。
- **Wave 0 总门禁**：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite && pnpm run repro:dev-broadcast-v2`。

### Wave 1：回归验收 + 补齐 C3/C4（大幅瘦身）
- **Bug 1（Goal Replay 客户端动态）**：已由 `476b783` 实现 → 在 Arch-0A 新底座上执行回归验收。
- **Bug 2（NameTag 与队伍同步）**：已由 `5fbc87d` 实现 → 在 Arch-0B 新底座上执行回归验收。
- **Bug 5（内存释放与 Tauri 关闭）**：
  - C1 / C2 已由 `0bc5c69` 实现并经 Arch-0C 纳入总线；
  - C3（AI / JS Cache 清理）在 Arch-0C 完成注册后验证 `retainedJsMB` 回落；
  - C4（`src-tauri/src/lib.rs`）已由 `0bc5c69` 实现 → 在含 `cargo/rustc` 环境跑 `cargo check` / `pnpm run tauri:build`。

### Wave 2：Bug 3（Pure CSR Spectate 物理前瞻）
- **2-A**：在 `SpectateInterpolationBuffer` 增加 `>= 5` 帧前瞻窗口与游标状态机。
- **2-B**：在 Arch-0A 的 `'client-csr-spectate'` 策略开启 `shouldRunLookahead = true`，驱动本地 WASM 物理连续步进与快照收敛。
- **验收**：`bug-3-replay-choppy-motion-no-physics-lookahead.reproduced === false`。

### Wave 3：Bug 4（BoostPad 模型闪烁收尾）+ 全量验收
- **3-A / 3-B**：利用 Arch-0D 输出的 `isDuplicateFrame` 与离散 `isActive` 阶跃值，在 `BoostPadSystem.ts` 增加重复帧跳过与冷却微抖阈值保护。
- **最终收官门禁**（在具备 WebGL 渲染能力的 Chrome 环境运行）：
  `pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite && pnpm run repro:dev-broadcast-v3` 确认 `reproducedCount === 0/5`。
