# 小队 C 排查报告：Bug 1 & Bug 2 在 Headless SwiftShader 下的测试稳定性与超时阈值

**基线 Commit**：`7af0003476afc0b275e34e7abebfd08df018d990`（`origin/z/agy/rc/multiplayer-recheck-base`）  
**修复分支**：`z/agy/rc/track-c-bug1-bug2-stability`（Verified Bot Commit: `43e1fc42c9a437ea1708c2901e9be8bb8258fec7`, `signature.isValid: true`）  
**类型检查**：`pnpm run check` GREEN（`1430/1430 TS`, `0 errors`）

---

## 一、核心结论摘要

1. **Bug 1（`Timeout waiting for Host enters Freeplay Goal Replay after 20000ms` 与 `sampleCount: 3`）**：
   - 根因是 **双窗口 SwiftShader CPU 抢占 + `dt <= 0.1s` 物理步长钳位 + `?skipShaderPrewarm=1` 首次进球着色器懒编译** 共同导致的时序瓶颈，而非 Goal Replay 物理/同步逻辑回退。
   - 球从 `y=1500` 以 `vel_y=2000` 射入球门（`y=5120`）需 `217 ticks`（`1.81s`），加上 `POST_GOAL_CELEBRATION_TICKS = 150`（`1.25s`，`src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts:45`），共需 **367 ticks** 才能进入 `isGoalReplayActive()`。由于 `src/game/pipeline/FramePipeline.ts:235,264` 将每帧 `dt` 钳位在 `0.1s`（最多 `12 ticks/frame`），Host 至少需要 **31 个完整渲染帧**。在双窗口 SwiftShader 仅 1.5–4 FPS 且首个进球触发 Goal Explosion 着色器编译时，很容易超过 `20_000ms`。
   - `sampleCount: 3`（偶发 `distinctBallPositions <= 2`）则是因为 `hostCdp.send('Page.bringToFront')` 让 Host 持续占满前台 SwiftShader 渲染，而 Client 仅在 `requestAnimationFrame` 中采样，~3s Goal Replay 期间 Client 仅获 3–8 次 rAF 回调。
2. **Bug 2（`clientAfterHostEdge2: null`，5s 超时）**：
   - 检查 `src/game/pipeline/stages/SimulationStage.ts:402-421` 与 `src/game/pipeline/stages/AuthorityCameraSyncTracker.ts:68-163`：**不存在低 FPS 漏发边沿同步的逻辑缺陷**。`pollLocalCameraChange()` 基于状态键 `isFree:ballCam:focusedCarIndex` 做 diff，只要 `camera` 停留在新状态，无论 FPS 多低，下一次 `SimulationStage.execute()` 必发；接收端 `shouldApplyIncomingCameraSync()` 亦基于单调 `(authorityEpoch, cameraSyncSeq)` 严格幂等。
   - 偶发 `5_000ms` 超时是因为 `Target.createTarget({ newWindow: true })` 使 Host 与 Client 两扇窗口的 `document.hidden` 均为 `false`（实测 `hostBeforeEdge2.docHidden === false`），6 车 + 6 NameTags 并发 SwiftShader 渲染导致 Host `SimulationStage.execute()` 与 Client `Runtime.evaluate` 轮询延迟突破 `5_000ms`。
3. **额外发现（全局耗时瓶颈）**：
   - 在 `7af0003` 上实测 `Client boot` 耗时高达 **95.8s – 100.7s**（其中单次 `clientCdp.eval` 阻塞达 85.5s–92s），因为 Host 已在 Window 1 全速渲染 3D WebGL 时，Client 在 Window 2 并发启动 WASM/Three.js 并编译着色器。这直接消耗了 270s watchdog 超过 1/3 的预算，建议后续可考虑在 Client 启动期间暂停/降频 Host 渲染或分阶段超时。

---

## 二、实测打点数据（基线 `7af0003` vs 修复后 `43e1fc4`）

### 1. 基线 `7af0003` 实测（含细粒度打点）
- **Run A（默认配置）**：
  - `Host boot`: `23.1s`，`Client boot`: **`100.7s`**（`poll 41` 后单次 `eval` 阻塞 `92s`）
  - `Host enters Freeplay Goal Replay`: **在 `20,000ms` 超时失败**（`poll 1..81` 每 20 次轮询耗时 `3.6s–8.1s`，约 400ms/poll）
- **Run B（临时放宽 Bug 1 超时至 45s 复测 Bug 1 + Bug 2）**：
  - `Host boot`: `26.1s`，`Client boot`: **`95.8s`**
  - `Host enters Freeplay Goal Replay`: **`8,192ms`**
  - `Client enters Goal Replay`: **`10,540ms`**（已超过原 `10_000ms` `.catch(()=>{})` 上限！）
  - `Host exits Freeplay Goal Replay`: `2,909ms`
  - Bug 1 结果：`PASS_FIXED`，`sampleCount: 8, distinctBallPositions: 3`（位置 `3513.3` x3, `4762.3` x2, `5575.2` x3，处于失败边缘）
  - Bug 2 结果：`REPRODUCED_BUG`：
    - `clientAfterHostEdge1`（car 2, `ballCam=false`）：**`112ms`** 成功
    - `clientAfterLocalSwitch`：`focusedCarIndex: 4` 成功
    - **`clientAfterHostEdge2`（car 5, `ballCam=true`）：`null`（`5,000ms` 超时）**
- **Run C（隔离跳过 Bug 1 单独测 Bug 2）**：
  - `hostBeforeEdge2`: `{"tick":77,"camFocused":5,"camBall":true,"docHidden":false}`
  - `edgeSyncOk: true`，`clientAfterHostEdge2`: `{"focusedCarIndex":5,"ballCam":true,"isFreeCam":false}`
  - 证实协议与 `SimulationStage` 逻辑完全正常，`5_000ms` 在前面跑完 Bug 1 后处于 SwiftShader 抖动临界点。

### 2. 代码与机制根因定位

#### (1) `?skipShaderPrewarm=1` 与 Bug 1 首次 Goal Replay
- `src/game/runtime/RuntimeBloomOcclusion.ts:87-95`：当 URL 含 `?skipShaderPrewarm=1` 时，直接跳过 `prewarmSceneShaders()`、boost/exhaust emitter 预加载与 `renderBloom()`。因此测试中**第一次进球**（Goal Explosion 粒子/光效 + Goal Replay 切换）必然触发首次着色器懒编译峰值。
- `src/game/modes/freeplay/FreeplayGoalAndTouchSubController.ts:45,326`：`POST_GOAL_CELEBRATION_TICKS = 150`（1.25s @ 120Hz）。结合 `y=1500, vel=2000` 射门（`~217 ticks`），进入 `isGoalReplayActive()` 至少需要 `367 ticks`。
- `src/game/pipeline/FramePipeline.ts:235,264`：`const dt = Math.min(..., 0.1)`，每帧最多推进 `12 ticks`，即至少需要 **31 个完整渲染帧**。在 SwiftShader 2 FPS 下即需 `15.5s+`，叠加首次着色器编译即突破 `20s`。
- **为何不建议缩短射门距离**：`src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts:102-104` 会对无触球进球执行 `trimLeadingIdleGoalReplayFrames(extractedFrames, 12)`。若改为近门射门（如 `y=4200`），回放总帧数会从 ~367 帧腰斩至 ~194 帧（~1.6s），反而加剧 Client `sampleCount` 不足。

#### (2) `SimulationStage.ts` 与 Bug 2 边沿同步
- `src/game/pipeline/stages/SimulationStage.ts:402-421`：每次 `SimulationStage.execute()` 都会调用 `sharedAuthorityCameraSyncTracker.pollLocalCameraChange(camera, authEpoch, localPeerId)`。
- `src/game/pipeline/stages/AuthorityCameraSyncTracker.ts:68-108`：以 `computeCameraKey`（`${isFree}:${ballCam}:${focused}`）做状态对比，仅当状态变化时递增 `localSeq` 并广播 `camera_state_sync`。低 FPS 不会导致状态变更被“跳过”或“漏发”。
- `src/game/runtime/RenderClockScheduler.ts:201-210` & `scripts/runDevBroadcastReproV3.mjs:229-232`：因 `Target.createTarget({ newWindow: true })` 创建独立窗口，Host 与 Client 的 `document.hidden` 均为 `false`，双端同时以 `frameInterval=0` 抢占 SwiftShader CPU，导致 `5_000ms` 轮询窗口在 6 车渲染压力下偶发超时。

---

## 三、已提交的确定性修复（Branch: `z/agy/rc/track-c-bug1-bug2-stability`）

已在 `scripts/runDevBroadcastReproV3.mjs` 提交 Verified Bot Commit `43e1fc4`（仅改测试脚本，不改引擎逻辑）：
1. **Bug 1 超时与前台采样优化**：
   - `Host enters Freeplay Goal Replay` 超时由 `20_000ms` 放宽至 `40_000ms`。
   - `Client enters Goal Replay` 超时由 `10_000ms` 放宽至 `20_000ms`。
   - 在 Host 确认进入 Goal Replay 后立即执行 `await clientCdp.send('Page.bringToFront')`，避免 Host 独占前台 SwiftShader 导致 Client `rAF` 采样被饿死（消除 `sampleCount: 3` 误报）。
2. **Bug 2 边沿同步超时放宽**：
   - `clientAfterHostEdge1` 与 `clientAfterHostEdge2` 的 `waitFor` 超时由 `5_000ms` 放宽至 `15_000ms`（保持 `100ms` 轮询间隔，正常情况仍 ~112ms 返回）。

### 修复后实测验证（`pnpm run repro:dev-broadcast-v3`）
- **`bug-1-goal-replay-client-static-no-dynamics`**: **`PASS_FIXED`** (`enteredReplay: true, distinctBallPositions: 3, sampleCount: 5`)
- **`bug-2-replay-nametag-and-team-desync`**: **`PASS_FIXED`** (`edgeSyncOk: true`, `clientAfterHostEdge1: {2, false}`, `clientAfterLocalSwitch: {4, false}`, `clientAfterHostEdge2: {5, true}`)
- **`bug-4-boostpad-model-flicker`**: **`PASS_FIXED`** (`totalPadFlips: 6, maxSinglePadFlips: 1`)
- （注：`bug-3` 与 `bug-5` 仍按预期处于 `REPRODUCED_BUG`，分别由小队 A/B 与小队 D 负责。）

---

## 四、给汇总合并的后续建议
1. 若需进一步压降 `repro:dev-broadcast-v3` 总耗时（避免接近 270s watchdog），建议在 `Client boot`（95–100s）期间临时节流 Host 渲染或分步初始化，而非仅靠 `?skipShaderPrewarm=1`。
2. `z/agy/rc/track-c-bug1-bug2-stability`（`43e1fc4`）可直接合入最终修复分支，与小队 A/B/D 的修复无文件冲突。
