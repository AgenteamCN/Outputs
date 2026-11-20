# 小队 B 排查报告：Bug 3 —— Host 后台 Blob Worker 实际心跳与 Replay 到头回绕/暂停行为

- **排查基线**：`origin/z/agy/rc/multiplayer-recheck-base`（HEAD: `7af0003476afc0b275e34e7abebfd08df018d990`）
- **已提交修复分支**：`origin/z/agy/rc/team-b-bgworker-replay-len`（Verified Bot Commit: `86998ce3f0244ad04192f67f0941b4f6815ab501`，`signature.isValid: true`）
- **聚焦文件**：
  - `src/game/RenderClockScheduler.ts`
  - `src/replay/StandaloneReplayViewerController.ts`
  - `src/replay/ReplayTakeoverSubController.ts`
  - `src/replay/ReplayViewerCausalSyncController.ts`
  - `scripts/env/generateReplayFixture.ts`
  - `scripts/runDevBroadcastReproV3.mjs`

---

## 一、核心结论（带实测打点证据）

### 结论 1：`RenderClockScheduler.startBackgroundWorker()` 在 `7af0003` 上从未被调用（死代码）

1. **代码审查**：Track 3（`0d68494`）在 `src/game/RenderClockScheduler.ts` 中新增了 `bgWorker`、`bgWorkerUrl`、`startBackgroundWorker()`、`stopBackgroundWorker()`，并在 `dispose()` 中调用了 `stopBackgroundWorker()`，但 **`wake()` 中遗漏了调用**（`rg -n "startBackgroundWorker" src/` 仅命中 `RenderClockScheduler.ts` 定义处）。
   - 当 `document.hidden === true` 时，`wake()` 只取消了 `displayRequest` 并 `clearPending()`，未调用 `this.startBackgroundWorker()`。
   - 当 `document.hidden === false` 或 `!this.active()` 时，`wake()` 也未调用 `this.stopBackgroundWorker()`。
   - 此外，`queueTick()` 仅将 `this.frameTimer = null; this.frameTask = null;` 置空，未调用 `this.clearFrameTimer()`，会导致 `bgWorker` 唤醒 `queueTick()` 后残留旧的 `setTimeout` / `scheduler.postTask`。

2. **Headless Chrome 实测打点（`7af0003`）**：
   - 在 `runDevBroadcastReproV3` 模式下，由于 `launchHeadlessChrome.mjs` 使用 `Target.createTarget({ newWindow: true })` 创建独立窗口并带有 `--disable-backgrounding-occluded-windows --disable-renderer-backgrounding --disable-background-timer-throttling`，执行 `clientCdp.send('Page.bringToFront')` 后：
     - Host `document.hidden` 仍为 `false`（`visLog: []`），`cs.bgWorker` 为 `false`。
     - Host `renderFrame` 在 2000ms 窗口内运行 **0 次**（受 SwiftShader 单核 WebGL2 `fenceSync` 抢占影响，`pending.length >= 2` 持续阻塞渲染分支）。
     - Host `executeSimulationOnly()` 走入 `else if (this.simulate && now >= this.nextSimTime - 0.1)` 分支，在 2000ms 内执行了 **121 次**（约 60.5 Hz，`minMs: 2`, `maxMs: 75.5`, `avgMs: 16.7`），并未完全卡死，Host `cursor` 从 `312` 正常推进到 `553`。
   - **但在真实浏览器 / 同窗口切 Tab 场景下**（`document.hidden === true`），因 `startBackgroundWorker()` 未被 `wake()` 触发，Host 主线程仍会遭受浏览器 1Hz 后台定时器节流。

---

### 结论 2：720 帧夹具在 Replay Viewer 中实际仅播放 6 秒，且 `seekToTime` 为空调用，导致采样第 7 帧即播完暂停

1. **为什么 720 帧只播 6 秒？**
   - `scripts/env/generateReplayFixture.ts` 原默认生成 `frameCount = 720`、`ticksPerFrame = 2`（容器头 `tickRate = 60`）。
   - 然而 `StandaloneReplayViewerController` / `ReplayTakeoverSubController.stepPlaying()`（`src/replay/ReplayTakeoverSubController.ts:445`）按 **120Hz 物理 tick 每 tick 推进 1 帧**（`this.cursor += (effectiveFlowPct || 100) / 100`）。
   - 因此 720 帧在 Replay Viewer 中实际播放时长仅为 `720 / 120 = 6.0 秒`！

2. **为什么采样起点更早耗尽？**
   - `scripts/runDevBroadcastReproV3.mjs:356` 调用了 `rv.seekToTime?.(1.0)`，但 `StandaloneReplayViewerController` 只有 `seekFrame(frameIndex)`，**不存在 `seekToTime` 方法**（实测 `typeof rv.seekToTime === 'function'` 为 `false`），该调用静默空转，播放始终从 `cursor = 0` 开始。

3. **实测复现（`7af0003`）**：
   - 载入夹具后：`framesLength: 720, hasSeekToTime: false, cursorAfterStart: 0, viewerState: "playing", firstFrameTick: 2, lastFrameTick: 1440`。
   - 经过测试脚本的 `2500ms` 初始等待 + NameTags 等待 + 两次 `edgeSync` 校验后，进入 Client 30 帧 `rAF` 采样前：
     - `hostBeforeSample`: `{"cursor": 554, "viewerState": "playing"}`（仅剩 165 帧，约 1.4~2.7s 余量！）。
   - 在 SwiftShader 下 Client 采集 30 个 `rAF` 耗时约 28 秒（约 1s/帧）：
     - 第 0~6 帧：Client tick 从 `13713` 推进到 `13894`。
     - **从第 7 帧起直到第 29 帧（连续 23 帧）**：Host 已到达 `cursor: 719` 并自动切换为 `viewerState: "paused"`（`hostAfterSample: {"cursor": 719, "viewerState": "paused"}`），导致 Client 画面与 tick 完全冻结在 `tick: 13894, pos: (1184.8, 107, 115.5)`！

---

## 二、已实施修复与验证结果（分支 `z/agy/rc/team-b-bgworker-replay-len`）

### 修改清单
1. **`src/game/RenderClockScheduler.ts`**：
   - 在 `wake()` 中：`!this.active()` 时调用 `this.stopBackgroundWorker()`；`document.hidden` 时调用 `this.startBackgroundWorker()`；恢复可见时调用 `this.stopBackgroundWorker()`。
   - 在 `queueTick()` 中改用 `this.clearFrameTimer()`，避免后台 Worker 唤醒时遗留未清理的 `setTimeout` / `postTask`。
2. **`scripts/env/generateReplayFixture.ts`**：
   - 将默认 `frameCount` 从 `720`（120Hz 下仅 6 秒）提升至 `1800`（120Hz 下 15 秒 / 60fps 下 30 秒），生成耗时仅 ~389ms，大小 ~594KB。
3. **`scripts/runDevBroadcastReproV3.mjs`**：
   - 将无效的 `rv.seekToTime?.(1.0)` 替换为 `rv.seekFrame?.(120)`。

### 验证数据（`86998ce`）
- `pnpm run check`：**GREEN**（0 TS errors，`tsc --noEmit` 通过）。
- `RenderClockScheduler` 后台 Worker 生命周期实测：`{"whenHidden": true, "whenVisible": false}`（隐身即启，可见即停）。
- 1800 帧夹具 + `seekFrame(120)` 30 帧采样实测：
  - `hostBefore`: `{"cursor": 551, "viewerState": "playing"}`
  - `hostAfter`: `{"cursor": 1591, "viewerState": "playing"}`（全程保持 `playing`，未再撞上播完暂停边界）。
  - Client 30 帧指标：`{"distinctTicks": 11, "frozenBallFrames": 16, "maxBallStepUu": 1923.4}` —— 说明在彻底排除 Host 暂停因素后，Client 侧仍存在 `frozenBallFrames: 16` 与 `maxBallStepUu: 1923.4`（超出 1800 uu 阈值触发硬切），该部分属于 **小队 A** 负责的 Client Lookahead / `ReplicationStateInterpolator` 范畴。
