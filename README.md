# RocketCompanion 多人同步收尾并行修复计划（精简执行版）

- **新基线分支**：`origin/z/agy/rc/multiplayer-recheck-base`
- **新基线 HEAD**：`e8aae572a729b535bae90e74adaba069a4b41524`（Verified Bot 3-Parent Merge，`verified: true`）
  - 已合入小队 B（`86998ce`：`RenderClockScheduler` `bgWorker` 接线 + 1800 帧/15s 夹具 + `seekFrame(120)`）
  - 已合入小队 C（`43e1fc4`：`runDevBroadcastReproV3.mjs` Bug 1/2 SwiftShader 超时与 Client 前台采样稳定化）
- **前置事实（无需重复排查）**：
  - **Bug 1 & Bug 2**：已通过 `43e1fc4` 解决，`PASS_FIXED`。
  - **Bug 3 Host 侧干扰**：6s 播完暂停与 `bgWorker` 未启用已通过 `86998ce` 解决；**剩余问题纯属 Client 侧 Lookahead/缓冲与 SwiftShader 调度**。
  - **Bug 4 & Bug 5**：已排除闪烁与真内存泄漏嫌疑，**只需做判定对齐与基线预热/清理时序收尾**。

---

## 并行推进方案（2 路无文件冲突并行）

### 分支 1：Bug 3 Client Lookahead 与缓冲/调度修复（对应小队 A 结论）
- **建议分支名**：`z/agy/rc/fix-bug3-client-lookahead`（基于 `e8aae57`）
- **锁定文件（与分支 2 零重叠）**：
  - `src/network/sync/ReplicationStateInterpolator.ts`
  - `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
  - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
  - `src/game/RenderClockScheduler.ts`
- **已定位根因与修复动作**：
  1. **防旧帧淘汰导致单帧 +100~+180 tick 硬切**：
     - `SpectateInterpolationBuffer` 默认 `maxFrames = 64`（120Hz 下仅 ~0.53s）。SwiftShader 长渲染帧期间数据包堆积顶满队列，淘汰旧帧后 `oldestTick` 跃升，`sample()` 第 458 行 `Math.max(oldestTick, ...)` 强行硬切 `renderTick`。
     - **改法**：将 `maxFrames` 提升至 `240–360`（2–3s 缓冲）。
  2. **Lookahead 恢复步长与 Host 暂停/播完态同步**：
     - `2-lookahead-step` 单步 `dtMs` 硬限 `50ms`（4 ticks），长帧恢复后外推严重滞后；且 Client `flags.isReplayPaused` 不感知 Host 录像暂停/播完，播完后继续外推 180 ticks 触发 `3a-max-la-freeze` 死锁。
     - **改法**：放宽或分段处理 Lookahead 恢复步长；同步 Host 录像暂停/播完状态到 Client（或在到达录像终点时停止外推）。
  3. **消除 SwiftShader 下 `SimulationStage` 调度饥饿**：
     - 实测多个 `rAF` 周期内 `applySpectateSampleOrLookahead` 一次都未被调用（`traceLen` 静止），导致连续冻结帧。
     - **改法**：检查 `RenderClockScheduler` 在 `pending.length >= 2` / 长 `render()` 阻塞期间的 `simulate` 触发条件，确保 `SimulationStage` 稳定推进而不被渲染门控饿死。

---

### 分支 2：Bug 4 & Bug 5 收尾对齐（对应小队 D 结论）
- **建议分支名**：`z/agy/rc/fix-bug4-bug5-cleanup`（基于 `e8aae57`）
- **锁定文件（与分支 1 零重叠）**：
  - `scripts/runDevBroadcastReproV3.mjs`
  - `src/entities/vehicle/supersonic-trail/SupersonicWheelTrailSystem.ts`
  - `src/entities/arena/ArenaWorld.ts`（或相关预热/重置管线 `freeplayKickoffAndArenaResetPipeline.ts`）
- **已定位根因与修复动作**：
  1. **Bug 4 测试判定去误报**：
     - `BoostPadSystem.ts` 已具备完备防抖；`maxSinglePadFlips: 2` 是 SwiftShader 2–4s 长窗口 + 单帧跳 110 ticks 下小 pad（4s 冷却）的正常充能/拾取周期。
     - **改法**：在 `scripts/runDevBroadcastReproV3.mjs` 中调整 Bug 4 判定（如仅当同一 pad 在 `< 400ms` 内发生 `>= 2` 次翻转，或将长窗口阈值放宽至 `maxSinglePadFlips >= 3`）。
  2. **Bug 5 持久化特效池基线对齐 + 退出异步清理等待**：
     - `retainedGeometries: 2 / 10` 源于 `SupersonicWheelTrailSystem` 等持久化池在 `bootMem` 时 `visible = false` 未上传 GPU（0 计数），跑过超音速后被首次上传（每车 2 个 ribbon，5 车 = 10 个），退出仅隐藏未销毁；另 `onReturnToFreeplay()` 后固定等待 500ms 与异步 `restoreLocalPlayerFreeplayLoadout(0)` / `disposeScopes` 存在竞态。
     - **改法**：
       - 引擎侧：在预热阶段对持久化特效几何体执行一次 0-alpha 预上传使 `bootMem` 基线一致，或在 `resetFreeplayExtraCarsAndArenaToSingleCar` 中销毁 `carIndex >= 1` 的非本地车 ribbon。
       - 测试侧：`scripts/runDevBroadcastReproV3.mjs` 中 `onReturnToFreeplay()` 后等待 `runtimeResourceRegistry` 清理与本地车 loadout 恢复完成，再采样 `afterExitMem`。

---

## 最终汇合与验收标准
两路完成后以 Verified Bot Merge 合入同一分支，执行全量验收：
1. `pnpm run check`：GREEN（`1430/1430 TS`，0 errors）
2. `pnpm run test:multiplayer-suite`：`28/28 PASS`
3. `pnpm run repro:dev-broadcast-v3`：连续运行 3 次均达到 `reproduced=0/5`
