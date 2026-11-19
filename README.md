# Wave 0 Arch-0A 完成报告：模拟管线显式状态机

## 1. 分支与提交信息
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck` (`d69e01c`)
- **新建分支**：`z/agy/rc/wave0-arch-0a`
- **Bot 签名提交**：`3deaedf7fed88e62293ec0f7497cacca089cd02b`（GitHub GraphQL `createCommitOnBranch`，`signature.state = VALID`）
- **Commit URL**：https://github.com/Antigravi-ty/RocketCompanion/commit/3deaedf7fed88e62293ec0f7497cacca089cd02b

---

## 2. 核心改动（Arch-0A）

### 2.1 显式状态机与策略表：`src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`
- 新增互斥运行模式枚举 `SimulationDriveMode`（6 种模式）：
  - `'local-freeplay'`
  - `'host-authoritative'`
  - `'host-goal-replay'`
  - `'client-csr-spectate'`
  - `'client-goal-replay'`
  - `'local-replay-playback'`
- 新增显式策略接口 `SimulationDrivePolicy`：
  - `mode: SimulationDriveMode`
  - `shouldStepLocalPhysics: boolean`
  - `shouldHoldAccumulator: boolean`
  - `shouldBroadcastSnapshot: boolean`
  - `shouldRunLookahead: boolean`
  - `shouldFeedReplayTimeline: boolean`
- 新增 `classifySimulationDrivePolicy(input)` 与 `resolveSimulationDriveMode(input)`，将原先分散的 `isBroadcastingHost`、`isPureCsrClient`、`isClientInputAuthority`、`reconciler.replayPlaybackMode`、`flags.isGoalReplay`、`flags.shouldHoldAccumulator`、`remoteHasAuthority` 等布尔组合统一收口为单一互斥模式与策略表。

### 2.2 模拟阶段收口：`src/game/pipeline/stages/SimulationStage.ts`
- `SimulationStage.execute()` 改为先计算 `flags` 与 `policy = classifySimulationDrivePolicy(...)`。
- 物理步进 / 累加器冻结改为基于 `policy.shouldHoldAccumulator` 与 `policy.shouldStepLocalPhysics` 分发（保留 `client-csr-spectate` / `client-goal-replay` 的 `interpolator.sync(timestamp)` 契约及本地 Goal Replay 的插值保护）。
- 主机广播收口为 `if (policy.shouldBroadcastSnapshot && server)`，显式由 `'host-authoritative'` 与 `'host-goal-replay'` 策略控制，并透传 `isGoalReplay: policy.mode === 'host-goal-replay'`。
- 将 `policy` 传入 `stepNetworkReconciliationAndReplayStream(...)`。

### 2.3 网络对账与回放流收口：`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
- `NetworkReconcileStepInput` 新增 `policy: SimulationDrivePolicy`。
- 拆分 `shouldFlushReconcilerControls(...)`、`canRecordLiveDiagnosticTick(...)`、`feedStreamedReplayPacket(...)` 等纯职责辅助函数。
- 消除原有的 10 处复合布尔 `if` 分支，改为基于 `policy.mode` 与 `policy.shouldFeedReplayTimeline` 扁平分发，无损保留 `476b783`（Bug 1）的 Goal Replay 流式帧游标与 `appendStreamedReplayFrame` 逻辑。

---

## 3. 门禁与验证结果

| 检查项 | 结果 | 详情 |
| --- | --- | --- |
| `pnpm run check` (`radar` + `tsc --noEmit`) | **PASS (GREEN)** | 1425/1425 TS 文件 100%，`tsc --noEmit` 0 错误，0 循环依赖 |
| `pnpm run radar:audit` | **PASS** | **`BoolHotspots` 从 9 降至 7**，`SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 均已彻底移出 `BoolHotspots` |
| `pnpm run test:multiplayer-suite` | **PASS (24/24)** | 全部 24 项多人联机矩阵测试通过 |
| 6 模式策略单元测试 | **PASS** | 验证 6 种 `SimulationDriveMode` 策略字段输出符合预期 |
| `pnpm run repro:dev-broadcast-v2` | **环境受限** | 无 GPU/WebGL 容器中于 `Timeout waiting for Host boot after 60000ms` 超时（符合已知环境基线，需在带 WebGL 渲染能力的 runner 跑端到端浏览器回归） |
