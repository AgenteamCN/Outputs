已完成“基础设施收口”任务：统一了全局 4.5 分钟（270,000ms）软超时 Watchdog 与逐阶段 Streaming Output，并通过 Verified Bot Commit 推送到 `origin/z/agy/rc/multiplayer-recheck-base`。

### 一、核心改动

1. **新增统一 Watchdog 与流式日志模块**：[softTimeoutWatchdog.mjs](file:///workspace/RocketCompanion/scripts/lib/softTimeoutWatchdog.mjs)
   - 默认 `RC_SOFT_LIMIT_MS=270000`（兼容 `RC_SUITE_WATCHDOG_MS` 与显式 CLI `--max-duration-ms=`），相对容器 300s 硬上限保留 30s 安全裕度。
   - 定时器默认调用 `.unref()`，不会阻止正常完成的进程退出；`dispose()` 自动清理定时器与信号监听。
   - 注册 `SIGINT` / `SIGTERM` 与软超时处理，超时或中断时输出当前 `activeStage` / 已完成阶段统计，并执行 `onTimeout` / `onSignal`（带 3000ms grace timer 防止清理挂起）落盘部分报告与清理 Chrome/Vite 子进程。
   - 提供 `runStage` / `stageStart` / `stagePass` / `stageFail` / `log` / `warn` / `streamEvent`，直接写入 `process.stdout`/`process.stderr` 实现逐用例/逐阶段实时 flush。

2. **接入所有 `test:*` / `repro:*` 与 `bootstrap.sh`**：
   - [bootstrap.sh](file:///workspace/RocketCompanion/scripts/env/bootstrap.sh)：新增 `RC_SOFT_LIMIT_MS=270000` 后台 watchdog 与 `EXIT INT TERM` trap 清理。
   - [runMultiplayerSuiteTests.ts](file:///workspace/RocketCompanion/scripts/runMultiplayerSuiteTests.ts)：接入 270s watchdog，`runSection` 改为实时流式输出 `[START]` 与 `[PASS]/[FAIL] (ms)`。
   - [runMultiplayerSyncIssuesRepro.ts](file:///workspace/RocketCompanion/scripts/runMultiplayerSyncIssuesRepro.ts)：接入 270s watchdog，从结束汇总打印改为逐个 issue 实时流式输出，超时时输出已完成计数。
   - [runDevBroadcastReproV2.mjs](file:///workspace/RocketCompanion/scripts/runDevBroadcastReproV2.mjs) & [runDevBroadcastReproV3.mjs](file:///workspace/RocketCompanion/scripts/runDevBroadcastReproV3.mjs)：移除硬编码 `260_000`，接入统一 270s watchdog（超时自动落盘部分 JSON 并清理 Chrome/Vite）。
   - [runDevSyncReproSuite.mjs](file:///workspace/RocketCompanion/scripts/runDevSyncReproSuite.mjs) & [runDevSyncExtraRoutesE2E.mjs](file:///workspace/RocketCompanion/scripts/runDevSyncExtraRoutesE2E.mjs)：移除硬编码 `260_000`，接入统一 270s watchdog 与超时部分报告落盘。
   - [runMultiplayerRoomRepro.mjs](file:///workspace/RocketCompanion/scripts/runMultiplayerRoomRepro.mjs) & [runMultiplayerChaosRepro.ts](file:///workspace/RocketCompanion/scripts/runMultiplayerChaosRepro.ts)：统一使用 `resolveSoftLimitMs()` 与 `createSoftTimeoutWatchdog`（保留 `MultiplayerRoomRepro` 的 `autoExitOnTimeout: false` 优雅收尾与遥测汇总）。
   - [runScenarioBaseline.mjs](file:///workspace/RocketCompanion/scripts/runScenarioBaseline.mjs)、[runDevRoomSyncE2E.mjs](file:///workspace/RocketCompanion/scripts/runDevRoomSyncE2E.mjs)、[runSoakWatchdog.mjs](file:///workspace/RocketCompanion/scripts/runSoakWatchdog.mjs)：移除各自硬编码的 `270_000` / `180_000`，统一接入 270s watchdog。

### 二、验证结果

- `pnpm run check`：GREEN（`radar` 1430/1430 TS + `tsc --noEmit` 通过）。
- `pnpm run test:multiplayer-suite`：28 个测试全部 PASS，实时流式输出 `[START]` / `[PASS]`。
- `pnpm run repro:sync-issues`：5 个 case 逐个实时输出，全部通过（`0/5 issues reproduced`）。
- `pnpm run repro:multiplayer-chaos`：3 个阶段实时 JSONL 流式输出，全部 PASS。
- `node --check`（全部修改的 `.mjs` 脚本）与 `bash -n scripts/env/bootstrap.sh`：全部通过。
- Watchdog 专项验证：`.unref()` 在未调用 `dispose()` 时进程正常退出（exit 0）；`RC_SOFT_LIMIT_MS=1000` 触发 `onTimeout` 回调并按配置退出码（exit 2）退出。
- `pnpm run env:check`：全项 `[OK]`，OverlayFS 维持 23 MB（<= 100 MB）。

### 三、Git 提交与推送

- 通过 GitHub GraphQL `createCommitOnBranch` 生成 Verified Bot Commit：
  - Commit: [`aef499e2549606e58514836f1373972b6891fa66`](https://github.com/Antigravi-ty/RocketCompanion/commit/aef499e2549606e58514836f1373972b6891fa66)（`signature.isValid: true`, `state: VALID`）
  - Branch: `z/agy/rc/multiplayer-recheck-base`
  - 本地已执行 `git fetch && git reset --hard origin/z/agy/rc/multiplayer-recheck-base` 同步。
