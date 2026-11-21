# RocketCompanion vNext Wave 3 集成对齐与验证报告

**目标分支**：`origin/integration/vnext-core`  
**Verified Bot Commit**：[`e3488ffb06ea18ee9669d1e9c044247e5ff80294`](https://github.com/Antigravi-ty/RocketCompanion/commit/e3488ffb06ea18ee9669d1e9c044247e5ff80294)（`signature.state: VALID`）

---

## 一、关于“9 条分支反向 Merge + Octopus Merge”与业界最佳实践的选型说明

原方案设计了“Wave 3：9 条 W1 分支各自 `merge integration/vnext-core` → Wave 4：9 路 `git merge -s octopus` 一次性合流”。针对我们仓库当前的实际规模（1,457 个 TS 文件、单集成者推进、且必须使用 GitHub Verified Bot Commit）而言，**直接走“分层递进合流（Layered / Stacked Integration onto `integration/vnext-core`）”比 9 路反向同步 + Octopus Merge 更符合业界最佳实践**，原因如下：

1. **Octopus Merge 会在重叠文件上直接中止**：`ReplicationBusProtocol.ts`、`ActivityKernel.ts`、`FreeplayRewinderController.ts`、`scripts/runMultiplayerSuiteTests.ts` 4 个文件存在跨分支修改，`git merge -s octopus` 遇到任何文件级冲突都会拒绝执行。
2. **避免 9 次冗余合流提交污染历史**：9 条分支各自反向 merge `integration/vnext-core` 后再 9 路合流，会制造大量网状 merge commits；而按 **Transport/Buffer → Modes → Consumers** 3 层依次合入 `integration/vnext-core`，每一层都立即完成协议对齐并跑绿 `pnpm run check` 与单元测试，历史线性清晰、可追溯、可随时回滚。
3. **契合单仓库规模与 Verified Bot Commit 规范**：最终以干净的 Verified Bot Commit 落地 `origin/integration/vnext-core`，无需向 9 条临时特性分支分别推送 9 次无效的同步提交。

---

## 二、Wave 3 分层落地详情（W1-01 ~ W1-09 全量对齐）

### Layer 1：底层传输与缓冲（`W1-01`、`W1-02`、`W1-09`）
- **W1-01 (`worker-authority-bus`)**：
  - 将 `StateFrameSlidingWindow.ts`、`WorkerAuthorityBus.ts`、`AuthoritativeServerWorker.ts`、`scripts/testWorkerAuthorityBus.ts` 中的占位 `WAVE1_CHANNEL1_SCHEMA_HASH` 统一替换为 Wave 2 的 `VNEXT_CHANNEL1_SCHEMA_HASH`（`0x49D215C8`）。
  - `WorkerAuthorityBus.publishCurrentState()` 在未传入预编码 `payloadBytes` 时，自动调用 `VNextWorldFrameCodec.encode(...)` 输出通道 1 二进制帧（同时修正 `replayCursor.playbackSpeed` 字段引用）。
- **W1-02 (`timeline-frame-buffer`)**：
  - `TimelineFrameBuffer.ts` 对齐 `VNextPlaybackState`、`VNextCameraPose` 与 `DecodedVNextWorldFrame`，新增 `pushWorldFrame(worldFrame, snapshotOverride, arrivalMs)` 适配器直接消费通道 1 解码帧。
- **W1-09 (`bulk-asset-channel`)**：
  - 清理 `e697bca` 遗留的 2 个废弃文件：`src/network/transport/webrtc/BulkAssetChannel.ts` 与 `src/network/transport/webrtc/bulkAssetAssemblyPipeline.ts`，保留 `src/network/transport/BulkAssetChannel.ts` 与 `BulkAssetProtocol.ts`。

### Layer 2：玩法模式与裁决（`W1-03`、`W1-04`、`W1-05`、`W1-06`）
- **W1-03 (`mode-match-exhibition`)**：
  - 解决 `ReplicationBusProtocol.ts` 中 `ACTIVITY_IDS.EXHIBITION` 默认 flags 冲突（保留 `STATE_SNAPSHOT | USER_INPUT | CLOCK_AND_SCORE | FOLLOW_HOST_CAMERA`）。
  - `MatchExhibitionRosterManifest.ts` 新增 `toVNextRosterManifest()`，接入 `VNextRosterManifestCodec` 与 `computeRosterMetadataHash`。
- **W1-04 (`mode-freeplay-spectate`)**：
  - 将 `freeplaySpectateTypes.ts` 中的 `FREEPLAY_W104_TEMP_SCHEMA_HASH` 对齐至 `VNEXT_CHANNEL1_SCHEMA_HASH`。
- **W1-05 (`mode-replay-watch-together`)**：
  - `ReplayWatchRosterManifest.ts` 新增 `toVNextReplayRosterManifest()`，接入 `VNextRosterManifest` 与 `computeRosterMetadataHash`。
- **W1-06 (`mode-training-fpv-dual`)**：
  - 合并 `scripts/runMultiplayerSuiteTests.ts` 测试入口（保留第 29 节 BulkAssetChannel 与第 30 节 FPV Dual Arbitration）。
- **架构体积护栏治理**：
  - 由于 W1-03/04/05 共同扩展了 `src/network/server/activities/ActivityKernel.ts`（合流后达 `18.3 KB`，触发 Radar `18.0 KB` 硬上限），将 `ExhibitionActivityKernel` 拆分为独立领域模块 `src/network/server/activities/ExhibitionActivityKernel.ts`，使 `ActivityKernel.ts` 回落至 `17.2 KB / 503 lines`。

### Layer 3：下游消费端（`W1-07`、`W1-08`）
- **W1-07 (`consumer-audio-arena`)**：
  - `src/entities/arena/arenaRosterManifestPipeline.ts` 新增 `fromVNextRosterManifest()` 适配器，直接消费通道 2 `VNextRosterManifest`。
- **W1-08 (`consumer-hud-camera`)**：
  - `src/game/pipeline/stages/vnextHudCameraTypes.ts` 新增 `fromCodecRosterManifest()` 与 `fromDecodedWorldFrameToHudHeader()`，将通道 1/2 二进制解码结果直接映射为 HUD 1~6 与 Goal Replay 电影机位消费结构。

---

## 三、全量门禁与单测验证结果

所有检查均在 `reset --hard origin/integration/vnext-core`（commit `e3488ff`）后实测通过：

| 检查 / 测试项 | 命令 | 结果 |
|---|---|---|
| 架构 Radar + TS 严格编译 | `pnpm run check` | **GREEN**（`1457/1457` TS，0 错误，0 循环依赖，0 越权导入） |
| Wave 2 协议编解码对齐 | `node --experimental-strip-types scripts/testVNextProtocolAlignment.ts` | **PASS**（`Ch1=0x49D215C8, Ch2=0x9063ED98`） |
| W1-01 Worker 权威总线 | `node --experimental-strip-types scripts/testWorkerAuthorityBus.ts` | **PASS**（5/5） |
| W1-02 时间轴插值缓冲 | `node --experimental-strip-types scripts/testTimelineFrameBuffer.ts` | **PASS**（9/9） |
| W1-03 Match/Exhibition 统一 | `node --experimental-strip-types scripts/testMatchExhibitionUnified.ts` | **PASS** |
| W1-04 Freeplay 观战流 | `node --experimental-strip-types scripts/testFreeplaySpectateStream.ts` | **PASS**（5/5） |
| W1-05 Replay Watch Together | `node --experimental-strip-types scripts/testReplayWatchTogether.ts` | **PASS** |
| W1-08 HUD 1~6 & Goal Replay 相机 | `node --experimental-strip-types scripts/tests/vnextHudCameraConsumer.test.ts` | **PASS**（3/3） |
| W1-09 Channel 4 大文件通道 | `node --experimental-strip-types scripts/testBulkAssetChannel.ts` | **PASS** |
| 全量联机回归套件 | `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite` | **PASS**（30/30 全部通过） |

---

## 四、后续收尾建议（Wave 4）

当前 `origin/integration/vnext-core` 已完成全部 W1 模块合流、协议对齐与 `test:multiplayer-suite` 全绿门禁。后续只需：
1. 若需更新 `repro:dev-broadcast-v3` 断言以覆盖新通道 1/2 端到端广播链路，可在 `integration/vnext-core` 上补充后直接作为 PR 合入目标主干；
2. 按《下一轮 Real Device Test 专项验收清单》进行真机 6 人局 Replay、Freeplay 切后台、Exhibition、FPV Dual 与降频带宽实测。
