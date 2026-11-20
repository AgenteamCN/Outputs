# RocketCompanion OverlayFS Storage 性能修复、280s Watchdog 审查与五分支 Octopus Merge 验证报告

**目标仓库**：`Antigravi-ty/RocketCompanion`  
**集成基线分支**：`origin/z/agy/rc/multiplayer-recheck-base`（HEAD: `02adc3d1a404858c8186fee283658b69c2b3b3fa`）  
**执行日期**：2026-11-20  

---

## 1. 核心结论摘要

1. **OverlayFS Storage（`scripts/env/`）性能与可用性问题已定位并修复**：
   - 在完全清空 `/tmp` 的冷启动场景下，原 `bash scripts/env/bootstrap.sh` **超过 300s 触发容器硬超时且无任何输出**。
   - 经排查存在三大根因：
     1. `NODE_OPTIONS` 注入 `global-agent/bootstrap` 导致 `pnpm install` 完成后保持代理 keep-alive socket 挂起不退出（`8718d96` 已修复）；
     2. `syncPublicSubmoduleToTmpfs` 先 `git clone --depth 1` 拉取 `main` HEAD，再二次 `git fetch --depth 1` 拉取 `HEAD:public`（`e66c0523`），造成双倍下载耗时；
     3. `bootstrap.sh` 与 `runMultiplayerSuiteTests.ts` 缺少 280s 软超时（Watchdog）机制。
   - 修复并推送 `b8dd156` 后，冷启动 `bash scripts/env/bootstrap.sh` 与 `node scripts/env/doctor.mjs --auto-restore` 均稳定在 **~22s–26s** 内完成，OverlayFS 占用维持 **29 MB**（预算 <= 100 MB），温启动仅 **2.7s**。
2. **五分支 Octopus Merge 可行性**：
   - 5 个分支（`AA-Track A-1`、`BB-Track A-2`、`CC-Track B-2`、`DD-Track C-2`、`EE-Track E`）改动文件**完全正交无冲突**，`git merge -s octopus` **0 冲突一次性成功**。
   - 已通过 GitHub REST API 生成 6-parent **Verified Bot Octopus Merge Commit**：`071fc4b7e55100ac420fa9bedb3ea61c367b8595`（`verified: true`）。
3. **合并后全量验证与问题排查结果**：
   - `pnpm run check`：**PASS**（1430/1430 TS，0 循环依赖，0 hard errors）。
   - `pnpm run test:multiplayer-suite`：**PASS**（28/28 全部单测通过，含新增 Test 28）。
   - `pnpm run repro:dev-broadcast-v3`：
     - 初次合并实测发现 `AA-Track A-1` 在跨模式切换（Goal Replay `tick=12937, numCars=1` -> 3v3 Replay `tick~960, numCars=6`）时未重置 `lastSampledTick`，导致 Client 拒绝所有 3v3 Replay 快照（`numCars` 卡在 `1`）。
     - 已追加修复提交 `02adc3d1a404858c8186fee283658b69c2b3b3fa`（`verified: true`）：当 `numCars` 变化或 `renderTick` 回退 > 12 ticks 时自动 `resetSpectateLookaheadState(buffer)`。
     - 修复后复测 `repro:dev-broadcast-v3`：**`reproduced=3/5`**（`bug-4`、`bug-5` 为 `PASS_FIXED`；`bug-1`、`bug-2`、`bug-3` 仍存在各自场景/时序或步进缺陷，详见第 5 节）。

---

## 2. OverlayFS Storage 底座与用户约定体验评估

### 2.1 对“基线与环境约定”User Input 的可用性检验
- **优点**：
  - `pnpm run env:help` / `pnpm run env:check` / `pnpm run env:doctor` 语义清晰，`tmpfsGuard.mjs` 对 `/tmp/rc-cache`（`node_modules`、`public/`、`.vite`、`dist`、`chrome-userdata`）的 symlink 卸载非常有效，OverlayFS 实测稳定在 **29 MB**（基线 29 MB，预算 <= 100 MB）。
  - `pnpm` wrapper 自愈提示在 `/tmp` 重置时能准确输出 `node scripts/env/doctor.mjs --auto-restore`。
- **实际使用中的痛点与门槛**：
  1. **跨 Turn `/tmp` 重置高频发生**：在沙盒快照恢复机制下，每次 Turn 间 `/tmp` 可能被清空。此时直接跑 `pnpm run ...` 会因 `node_modules` 悬空报错 `ERR_MODULE_NOT_FOUND`，必须先跑 `node scripts/env/doctor.mjs --auto-restore`（或与测试命令用 `&&` 串联在同一条命令中）。
  2. **`bootstrap.sh` 此前会在冷启动下静默超时（>300s）**：见下节性能修复。
  3. **`public/` 判定条件偏差**：`bootstrap.sh` 原先仅检查 `public/camera.wasm` 与 `public/wasm`，而当前 `RocketCompanionStatics` 仓库包含的是 `public/arenamesh/manifest.json` 与 `public/assets/`，导致每次执行 `bootstrap.sh` 都会多余触发 `--sync-public`。

### 2.2 性能瓶颈定位与修复清单
| 问题点 | 根因 | 修复方案 | 提交 SHA |
| :--- | :--- | :--- | :--- |
| **`pnpm install` 完成后挂起不退出** | `NODE_OPTIONS` 注入 `global-agent/bootstrap` 导致代理 keep-alive socket 阻塞 Node 22 事件循环退出 | 在 `bootstrap.sh`、`tmpfsGuard.mjs`、`pnpm` wrapper 中剥离 `global-agent/bootstrap` 并禁用 update-notifier | `8718d96` |
| **`public/` 子模块双倍拉取** | `syncPublicSubmoduleToTmpfs` 先 `git clone --depth 1` 拉取 `main` HEAD，再二次 `git fetch --depth 1` 拉取 `HEAD:public` | 改用 `git init` + `git fetch --depth 1 origin <pinnedSha>` + `checkout FETCH_HEAD` 单次直拉，并增加 `RC_GIT_TIMEOUT_MS`（180s）防挂死 | `b8dd156` |
| **`bootstrap.sh` 温启动重复判定** | 仅检测 `camera.wasm`/`wasm`，未检测 `public/arenamesh/manifest.json` 与 `public/assets` | 补全文件存在性检测，温启动直接跳过 `--sync-public` | `b8dd156` |

- **优化前后冷启动实测对比**（`rm -rf /tmp/rc-cache /tmp/.rc-env-sentinel /tmp/rc-replay*` 后运行）：
  - 优化前：`bash scripts/env/bootstrap.sh` **> 300s 超时失败**（无 stream output）。
  - 优化后：`bash scripts/env/bootstrap.sh` **25.7s** 完成；`node scripts/env/doctor.mjs --auto-restore` **22.4s** 完成；温启动 **2.7s**。

---

## 3. 测试套件 280s Soft Limit / Time Watchdog 审查

| 脚本 | Watchdog 状态 | 阈值 | 审查与加固结果 |
| :--- | :--- | :--- | :--- |
| `scripts/runDevBroadcastReproV3.mjs` | 已有，已增强 | `260_000ms` (260s) | 新增超时时打印已完成用例列表 `Completed=X/5`，便于定位卡死阶段（`b8dd156`） |
| `scripts/runDevBroadcastReproV2.mjs` | 已有 | `260_000ms` (260s) | 低于 280s 硬阈值，合格 |
| `scripts/runDevSyncReproSuite.mjs` | 已有 | `260_000ms` (260s) | 低于 280s 硬阈值，合格 |
| `scripts/runDevSyncExtraRoutesE2E.mjs` | 已有 | `260_000ms` (260s) | 低于 280s 硬阈值，合格 |
| `scripts/runDevRoomSyncE2E.mjs` | 已有 | `180_000ms` (180s) | 低于 280s 硬阈值，合格 |
| `scripts/runMultiplayerChaosRepro.ts` | 已有 | `270_000ms` (270s) | 低于 280s 硬阈值，合格 |
| `scripts/runMultiplayerRoomRepro.mjs` | 已有 | `270_000ms` (270s) | 低于 280s 硬阈值，合格 |
| `scripts/runScenarioBaseline.mjs` | 已有 | `270_000ms` (270s) | 低于 280s 硬阈值，合格 |
| `scripts/runSoakWatchdog.mjs` | 已有 | `180_000ms` (180s) | 低于 280s 硬阈值，合格 |
| `scripts/runMultiplayerSuiteTests.ts` | **原缺失 -> 已补齐** | `280_000ms` (280s) | 新增 `.unref()` 280s Watchdog 安全网（`b8dd156`） |
| `scripts/env/bootstrap.sh` | **原缺失 -> 已补齐** | `280s` | 新增 280s 后台 Watchdog + `CURRENT_STEP` 阶段追踪与耗时打印（`b8dd156`） |

---

## 4. 五分支 Octopus Merge 结果与 Verified Bot Commits

### 4.1 合并分支清单（全部 0 冲突）
1. `origin/fix/bug3-lookahead-monotonic-tick` (`a657c49`) — `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
2. `origin/fix/bug3-replay-buffer-cadence` (`0ecf9e5`) — `src/network/sync/ReplicationStateInterpolator.ts`
3. `origin/fix/bug1-goal-replay-snapshot-stream` (`7147838`) — `scripts/runMultiplayerSuiteTests.ts`, `src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts`, `src/replay/ReplayPlaybackState.ts`
4. `origin/fix/bug5-remaining-geometry-leak` (`0d9d4ac`) — `src/entities/arena/BoostPadShadersAndMeshes.ts`, `src/entities/trajectory/BallTrajectoryTubeRenderer.ts`
5. `origin/fix/bug2-nametag-render-stability` (`b669bf1`) — `scripts/runDevBroadcastReproV3.mjs`, `src/entities/arena/VehicleNameTagSystem.ts`

### 4.2 远端 Verified Bot 推送记录（`origin/z/agy/rc/multiplayer-recheck-base`）
- `b8dd156197aa59b40cb4af909929be5b1f65085d` (`verified: true`): `perf(env): single-round-trip public/ fetch & 280s soft watchdogs`
- `071fc4b7e55100ac420fa9bedb3ea61c367b8595` (`verified: true`): `merge: octopus merge 5 fix branches (A-1, A-2, B-2, C-2, E) into z/agy/rc/multiplayer-recheck-base`（6-parent Octopus Merge）
- `02adc3d1a404858c8186fee283658b69c2b3b3fa` (`verified: true`): `fix(sim): reset spectate lookahead state on numCars/seek discontinuity`

---

## 5. 合并后验证结果（基于 `02adc3d`）

- **`pnpm run check`**：**PASS**（1430/1430 TS 文件 100%，`tsc --noEmit` 0 错误，0 循环依赖，0 hard errors）
- **`pnpm run test:multiplayer-suite`**：**PASS**（28/28 全部单测通过）
- **`pnpm run repro:dev-broadcast-v3`**：**`reproduced=3/5`**（2 个 `PASS_FIXED`，3 个仍 `REPRODUCED_BUG`）

| 用例 ID | 状态 | 实测关键指标与根因诊断 |
| :--- | :--- | :--- |
| **`bug-4-boostpad-model-flicker`** | **PASS_FIXED** | `totalPadFlips: 6`, `maxSinglePadFlips: 1` |
| **`bug-5-high-memory-and-unreleased-on-exit`** | **PASS_FIXED** | `geometries: 37 -> 105 -> 34`（`retainedGeometries: -3`），退出后完全释放，无几何体泄漏 |
| **`bug-1-goal-replay-client-static-no-dynamics`** | **REPRODUCED_BUG** | `enteredReplay: true`, `distinctBallPositions: 2`。前 2 帧采样在 `phase: "replay"`（`by: 5144.5`），但随后 Goal Replay 已结束切回 `phase: "playing"`（`0, 0, 93.2`），导致 replay 期间仅采样到 2 个位置（阈值要求 `> 2`）。说明裁剪前置空闲帧后 Goal Replay 有效时长偏短或采样窗口启动偏晚 |
| **`bug-2-replay-nametag-and-team-desync`** | **REPRODUCED_BUG** | `teamMismatch: false`（`[0,0,0,1,1,1]`），`client.numCars: 6`，但 `host.tags` 为 `[1,4,5,3]` 而 `client.tags` 仅 `[2]`。Client 与 Host 处于不同视角/朝向，未对齐到同一观察视角或未开启全车 offscreen indicator，导致双方可见 NameTag 集合不一致 |
| **`bug-3-replay-choppy-motion-no-physics-lookahead`** | **REPRODUCED_BUG** | `distinctTicks: 8`, `frozenBallFrames: 23`, `maxBallStepUu: 1341.8`。在 Headless SwiftShader 低帧率（rAF 间隔较大）下，`tick` 单帧跳变达 `41~54 ticks`（`18527 -> 18568 -> 18622`），导致球单步位移达 `1341.8 uu` 且多帧冻结 |
