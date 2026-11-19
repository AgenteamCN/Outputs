# Wave 0 Arch-0B 完成报告：Roster / Team / NameTag SSOT (`MatchRosterProjection`)

## 1. 分支与提交信息
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck` (`d69e01c`)
- **新建并推送分支**：`z/agy/rc/wave0-arch-0b`
- **Verified Bot Commit**：`479f7426a1bad40cf45e3fc3413c0007519fdd3b` (`verified: true`, `reason: valid`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)

## 2. 核心改造内容

### 2.1 新建单一数据源投影器 `src/game/runtime/MatchRosterProjection.ts`
- **`projectMatchRoster(params)`**：原子同步车辆队伍与 NameTag 运行时状态（`physics.setCarTeam` + `arena.setCarTeam` + `setNameTagRuntimeState` + 可选 `hudStore.setMatchState({ localTeam })`）。
- **`projectRosterCarEntries(carEntries, options)`**：为已构建标准化 `NameTagCarEntry` 的调用方提供统一投影入口。
- **`clearMatchRosterProjection(options)`** / **`setMatchRosterMultiCarMode(flag)`**：统一清理/切换多车 NameTag 状态。
- **`patchProjectedCarEntry(carIndex, patch)`** / **`mergeProjectedCarEntries(mergeFn, isMulti)`**：统一处理单车队伍/装备局部更新与诊断车队合并。
- **`projectNameTagCameraFocus(mode, carIndex)`**：统一相机焦点与模式变更投影（含脏检查）。
- **`bindNameTagTunerCallbacks(callbacks)`** / **`projectNameTagTunerState(state)`**：统一 NameTag Tuner 回调注册与预览状态投影。

### 2.2 全量迁移引擎侧 15 处 `[S]` 散落调用点
已将全部 15 个直接调用 `setNameTagRuntimeState(...)` 的引擎文件迁移至 `MatchRosterProjection.ts`：
1. `src/game/runtime/devSyncChannelHandlers.ts`（`syncReplayRosterTeamsAndNameTags` 委托至 `projectMatchRoster`）
2. `src/game/runtime/OnlineRosterSyncCoordinator.ts`
3. `src/game/modes/FreeplayMode.ts`
4. `src/game/modes/NameTagTunerArenaController.ts`
5. `src/game/modes/experimental-multi-bots/ExperimentalMultiBotsRosterController.ts`
6. `src/game/modes/freeplay/directorPreviewPipeline.ts`
7. `src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts`
8. `src/game/runtime/DiagnosticsFleetAndGhostController.ts`
9. `src/game/runtime/PlayerLoadoutRuntimeBridge.ts`
10. `src/game/runtime/RuntimeLoadoutCoordinator.ts`
11. `src/game/runtime/signalWorldTransitionPipeline.ts`
12. `src/game/subsystems/GameCameraSubsystem.ts`
13. `src/replay/ReplayViewerCausalSyncController.ts`
14. `src/replay/ReplayViewerRosterSubController.ts`
15. `src/replay/rlrc/runtime/RLRCReplaySessionBridgePipeline.ts`

### 2.3 收紧架构棘轮上限
- `scripts/radar/radarArchDebtGuards.ts`：`HARD_ENGINE_NAMETAG_SCATTER_LIMIT` 从 `15` 下调至 `1`。

## 3. 门禁与验证结果
- `pnpm run check` (`radar` + `tsc --noEmit`)：**PASS**（1426/1426 TS 文件，0 Circular Deps，0 Value Re-Exports，0 Upward Imports，0 Multi-Stmt Lines）
- `pnpm run radar:audit`：**PASS**（`Arch-Debt Ratchets: NameTagScatter=1/1`，全引擎仅剩 `src/game/runtime/MatchRosterProjection.ts` 1 处 `[S]`）
- `pnpm run test:multiplayer-suite`：**PASS**（`ALL MULTIPLAYER SUITE TESTS PASSED`）
- `pnpm run repro:dev-broadcast-v2`：当前无 GPU/WebGL 容器中 Chrome 无法正常完成渲染启动（退出码 1，`ENOTEMPTY` 发生在 Chrome 异常退出后的临时目录清理阶段），需在具备 WebGL 渲染能力的目标 runner 执行完整端到端回归。
