# 下一步行动与分支策略评估报告

## 一、三个可选 Commit 的真实状态核查

通过 `git ls-remote` 与 GitHub Compare API 对当前远程仓库 `Antigravi-ty/RocketCompanion` 核实，三个可选 commit 的关系与改动范围如下：

| Commit / 分支 | 与 `7af0003` 关系 | 修改文件与内容 | 当前状态与价值 |
| :--- | :--- | :--- | :--- |
| **`7af0003`**<br>`z/agy/rc/multiplayer-recheck-base` | 基线（4-Parent Octopus Merge） | — | Bug 1/2/4 在 SwiftShader 下有偶发超时抖动；Bug 3（含 Host 6s 播完干扰）与 Bug 5 必现。 |
| **`86998ce`**<br>`z/agy/rc/team-b-bgworker-replay-len` | `ahead_by: 1, behind_by: 0`（基于 `7af0003`） | • `src/game/RenderClockScheduler.ts`（+8/-4）<br>• `scripts/env/generateReplayFixture.ts`（+1/-1）<br>• `scripts/runDevBroadcastReproV3.mjs`（+1/-1，第 356 行） | **已修复** `wake()` 未接入 `startBackgroundWorker()` 的死代码、将合成录像从 720 帧（6s）扩至 1800 帧（15s）、并将无效 `seekToTime(1.0)` 改为 `seekFrame(120)`。已验证 `pnpm run check` GREEN。 |
| **`43e1fc4`**<br>`z/agy/rc/track-c-bug1-bug2-stability` | `ahead_by: 1, behind_by: 0`（基于 `7af0003`） | • `scripts/runDevBroadcastReproV3.mjs`（+5/-4，第 295–310、444–469 行） | **已修复** Bug 1/Bug 2 在 SwiftShader 下的超时抖动与 Client 前台采样饿死问题。实测 Bug 1/2/4 均 `PASS_FIXED`，`pnpm run check` GREEN。 |

**冲突评估**：`86998ce`（改 `runDevBroadcastReproV3.mjs` 第 356 行）与 `43e1fc4`（改第 295–310、444–469 行）**修改行段完全不重叠，零冲突，可直接合并**。

---

## 二、4 路排查报告的价值评估

**结论：这份报告价值极高，已经完成了整个排障周期中最耗时的“根因拆解与去伪存真”工作**，可以直接指导精准修复：

1. **Bug 1 & Bug 2（小队 C）**：已证实引擎侧无逻辑回退，纯属 SwiftShader 双窗口并发渲染耗时与前台焦点调度问题，`43e1fc4` 已直接解决。
2. **Bug 4（小队 D）**：已证实 `BoostPadSystem.ts` 防抖与去重机制完备，`maxSinglePadFlips: 2` 是长采样窗口 + 大 tick 跳变下单个小 pad（4s 冷却）的正常物理生命周期，只需微调测试脚本判定逻辑，无需动引擎。
3. **Bug 5（小队 D）**：已通过探针排除 Track 2 `focusedCarIndex` 切换的嫌疑，精确定位为 **持久化特效池（`SupersonicWheelTrailSystem` 等）冷启动未上传 GPU 导致的基线统计不可比 + `onReturnToFreeplay()` 异步清理竞态**，并非真内存泄漏。
4. **Bug 3（小队 A + 小队 B）**：将混合现象精准剥离为两部分：
   - **Host 侧干扰**（6s 播完暂停 + `bgWorker` 死代码 + `seekToTime` 无效）已被 `86998ce` 解决；
   - **Client 侧核心根因**已被小队 A 锁定在：`ReplicationStateInterpolator.ts`（`SpectateInterpolationBuffer` 默认 `maxFrames = 64` 仅 ~0.53s，SwiftShader 卡顿时被顶满淘汰旧帧引发单帧 +178 tick 硬切）、`spectatePhysicsLookaheadPipeline.ts`（Lookahead 步长限制与 Host 暂停/播完状态未同步导致外推 180 ticks 后死锁），以及 `RenderClockScheduler` 在 SwiftShader 下对 `SimulationStage` 的调度饥饿。

---

## 三、推荐的下一步执行方案（基于哪个 Commit & 是否并行）

### 1. 基线选择：先合并 `86998ce` + `43e1fc4` 作为新集成基线
不要直接停留在 `7af0003` 重新改，也不要只挑其中某一个：
- **立即基于 `7af0003` 将 `86998ce`（小队 B）和 `43e1fc4`（小队 C）合并**（零冲突的 Verified Bot Merge），得到一个 **Bug 1/2 稳定通过、且 Host 15s 录像全程 `playing` 不再播完暂停** 的新基线。
- 这样后续验证 Bug 3 时不再受“第 7 帧录像播完”和“Goal Replay 偶发超时”的噪音干扰。

### 2. 并行 vs 串行效率评估：建议 **2 路小规模并行**（不再开 4 路）
- **为什么不开 4 路并行**：Bug 1/2 已完成，Bug 4 仅需改 2 行测试脚本，开过多分支反而增加合并协调和重复跑 `repro:dev-broadcast-v3`（单次 ~200s+）的时间成本。
- **为什么 2 路并行最快、最稳**：剩余工作天然分为**完全无文件冲突**的两块：
  1. **分支 1（Bug 3 引擎核心修复，对应小队 A 结论）**：
     - `src/network/sync/ReplicationStateInterpolator.ts`：将 `SpectateInterpolationBuffer` `maxFrames` 从 `64` 提升至 `240–360`（2–3s 缓冲），避免 SwiftShader 卡顿期间淘汰旧帧导致单帧 +100~+180 tick 硬跳。
     - `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`：放宽/分段处理 Lookahead 恢复时的 `dtMs` 上限，并同步 Host 录像暂停/播完状态以阻止播完后盲目外推至 180 tick 死锁。
     - `src/game/RenderClockScheduler.ts`（如有必要）：确保 SwiftShader 长渲染帧间隙 `SimulationStage` 仍能按时推进，消除连续多个 `rAF` 不执行 `applySpectateSampleOrLookahead` 的饥饿现象。
  2. **分支 2（Bug 4 & Bug 5 收尾对齐，对应小队 D 结论）**：
     - `scripts/runDevBroadcastReproV3.mjs`：修正 Bug 4 判定（如仅当同一 pad 在 `< 400ms` 内发生 `>= 2` 次翻转，或放宽长窗口阈值至 `>= 3` 时判为闪烁）；在 `onReturnToFreeplay()` 后等待 `runtimeResourceRegistry` 清理与本地车 loadout 恢复完成再采样 `afterExitMem`。
     - `src/entities/vehicle/supersonic-trail/SupersonicWheelTrailSystem.ts` / `ArenaWorld`：在预热阶段对持久化特效几何体执行一次预上传（或退出时销毁非本地车槽位 `carIndex >= 1` 的 ribbon），使 `bootMem` 与 `afterExitMem` 的 `geometries` 基准严格一致（`retainedGeometries === 0`）。

### 3. 最终汇合与验收流水线
两路完成后合入同一集成分支（零文件冲突），一次性执行全量验收：
1. `pnpm run check`（`1430/1430 TS`，0 errors）
2. `pnpm run test:multiplayer-suite`（`28/28 PASS`）
3. `pnpm run repro:dev-broadcast-v3` 连续运行 3 次均达到 `reproduced=0/5`
