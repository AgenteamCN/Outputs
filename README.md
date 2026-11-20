# 路 1（Bug 1：Freeplay Goal Replay 时长与动态性）定位与修复报告

## 1. 核心结论（Deterministic）

**产品逻辑完全正常且在 `phase === 'replay'` 期间具备完整动态性，不存在“Client 进入 Goal Replay 后球/车永久静止”的引擎 Bug。**

原 `scripts/runDevBroadcastReproV3.mjs` 报告 `bug-1-goal-replay-client-static-no-dynamics`（`distinctBallPositions: 1`）是由**测试夹具的三个叠加问题**导致的误报：
1. **造球距离过近（0.09s 进门）**：原测试在 `y=4850, vel_y=2950` 造球，距球门线（`y=5120`）仅 `270 uu`，约 `0.09s`（~11 ticks）即进门；叠加 `trimLeadingIdleGoalReplayFrames(extractedFrames, 12)` 裁剪前置空闲帧后，进门前有效动态窗口极短。
2. **`?skipShaderPrewarm=1` 触发 SwiftShader 首次 Goal Replay 着色器编译卡顿（6.6s–13.3s）**：在无头 Chrome SwiftShader 环境下，由于测试 URL 带 `?skipShaderPrewarm=1`，Host 在**第一次**进入 Goal Replay 首帧时会触发一次性的 WebGL 着色器编译卡顿（单 Tab 实测 `dt=6657ms–6839ms`，双 Tab 实测 `dt=8000ms–13292ms`；同一会话第二次 Goal Replay 卡顿完全消失，30 帧丝滑推进）。在此卡顿窗口内 Host 主线程阻塞、暂停广播新快照，Client 停留在已收到的最后一帧（`bufFrames: 3`）。
3. **固定 `6 × 200ms`（1.2s）轮询窗口全部落入卡顿期且未过滤 `phase === 'replay'`**：原测试在 `Client enters Goal Replay` 后仅采样 `1.2s`，6 个采样点全部落在 Host 首次着色器编译卡顿期间，导致 `bx=0, by=4972.8, bz=96` 连续 6 次完全相同（`distinctBallPositions === 1`）。

---

## 2. 实测数据与根因证据链

### 2.1 基线复现数据（修改前）
运行 `pnpm run repro:dev-broadcast-v3`：
```json
{
  "enteredReplay": true,
  "distinctBallPositions": 1,
  "replaySamples": [
    {"phase":"replay","hudPhase":"replay","camMode":"pov","override":true,"bx":0,"by":4972.8,"bz":96,"bufFrames":3},
    {"phase":"replay","hudPhase":"replay","camMode":"pov","override":true,"bx":0,"by":4972.8,"bz":96,"bufFrames":3},
    {"phase":"replay","hudPhase":"replay","camMode":"pov","override":true,"bx":0,"by":4972.8,"bz":96,"bufFrames":3},
    {"phase":"replay","hudPhase":"replay","camMode":"pov","override":true,"bx":0,"by":4972.8,"bz":96,"bufFrames":3},
    {"phase":"replay","hudPhase":"replay","camMode":"pov","override":true,"bx":0,"by":4972.8,"bz":96,"bufFrames":3},
    {"phase":"replay","hudPhase":"replay","camMode":"pov","override":true,"bx":0,"by":4972.8,"bz":96,"bufFrames":3}
  ]
}
```

### 2.2 页面内打点（100ms `setInterval` + `rAF` 帧耗时）
- **双 Tab 原参数（`y=4850, vel_y=2950`）打点**：
  - Host 在 `t=1598ms` 进入 `phase: "replay"`，`t=1801ms` 推进到 `by=5095.5`；随后主线程卡顿 **~7.95s**（`t=1801ms -> t=9749ms`）。
  - 在此期间 Client 处于 `phase: "replay"`，`lpTick: 12393`、`bufFrames: 2`、`by: 4997.4` 保持不变等待 Host 新帧。
- **单 Tab 第一轮 vs 第二轮 Goal Replay 对比（`y=1500, vel_y=2000`）**：
  - **Round 1**：进入 `replay` 首帧后出现 `dt: 6657ms` 卡顿（SwiftShader 首次编译 Goal Replay / Director / Goal Explosion 着色器）。
  - **Round 2**：**零卡顿**，共记录 30 个 `replay` 帧，球坐标连续推进：`by: 2113.7 -> 2377.3 -> 2967.6 -> 3258.3 -> 4004 -> 4993.2 -> 5043.6 -> 5575.2`。
- **双 Tab 中场射门（`y=1500, vel_y=2000`）页面内 `rAF` 采样**：
  - Client 在 `phase === 'replay'` 期间共采集 11 帧（越过一次性着色器编译卡顿后 `bufFrames` 从 10 增长到 51–64），球坐标连续变化：
    `Distinct positions: ["0,2574.3,106.1", "0,4295.5,93.2", "0,5315.8,93.1", "0,5575.2,101.7"]`（`distinctBallPositions = 4`）。

### 2.3 代码链路核查
- **Host 侧**：`src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts`（L90-104）使用 `lookbackSeconds: 10.0` 提取缓冲并裁剪前 12 个静止帧（`trimLeadingIdleGoalReplayFrames`），由 `FramePipeline.ts`（L235/L264 `Math.min(dt, 0.1)` 钳制步长）驱动 `directorPreviewFrameStepPipeline.ts` 播放并广播快照。
- **Client 侧**：
  - `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`（L243-251）将 `client-goal-replay` 分类为 `shouldRunLookahead: false, shouldFeedReplayTimeline: true`。
  - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`（L213-312）启用 `spectateInterpolationBuffer`，将 Host 广播帧压入缓冲并在 `applySpectateSampleOrLookahead` 中采样应用至 `physics` 与 `interpolator`。
  - `src/game/modes/shared/networkStreamedGoalReplayCoordinator.ts`（L122）仅在 `isCustomScrubbing` 时由本地 `applyToEngine`，正常播放时完全由网络插值缓冲平滑驱动，无冲突覆写。

---

## 3. 修复与验证结果

### 3.1 修复内容（已提交新分支）
- **分支**：`z/agy/rc/track1-bug1-goal-replay-test-fix`（基于 `origin/z/agy/rc/multiplayer-recheck-base`）
- **Commit**：`0ff3e161b3cb700d636044aa63641181067f8ee7`（Verified Bot Commit，`verified: true, reason: "valid"`）
- **改动文件**：`scripts/runDevBroadcastReproV3.mjs`（REPRO 1 区块）
  1. 在 `setBallState` 之前于 Client 注入页面内 `rAF` 采样器，仅收集 `phase === 'replay' || hudPhase === 'replay'` 的帧，避免 `waitFor` 轮询错过早期帧或混入退出后的 `playing` 帧。
  2. 将造球参数改为中场射门 `pos: [0, 1500, 95], vel: [0, 2000, 200]`（距球门 ~3620 uu，飞行 ~1.8s+，形成 >= 2s 自然缓冲）。
  3. 等待 Host 完整退出 Goal Replay（超时 30s，覆盖 `?skipShaderPrewarm=1` 下 SwiftShader 首次编译耗时）后再读取 Client 采样结果并统计 `distinctBallPositions`。

### 3.2 回归验证
1. `pnpm run repro:dev-broadcast-v3`：
   - `bug-1-goal-replay-client-static-no-dynamics` -> **`PASS_FIXED`**
   - 输出：`{"enteredReplay":true,"distinctBallPositions":4,"sampleCount":22,...}`
2. `pnpm run check`（`radar` + `tsc --noEmit`）：
   - **STATUS: GREEN**（1430/1430 TS files，0 errors）。
