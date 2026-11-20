# RocketCompanion 5 分支 Octopus Merge 验证与基线环境评估报告

## 1. 总结结论

- **Octopus Merge 可行性**：基于 `origin/z/agy/rc/multiplayer-recheck-base` (`8718d96`) 对 5 条修复分支执行 `git merge --no-ff` Octopus Merge **0 冲突一次性合入成功**。
- **静态检查与单测门禁**：`pnpm run check`（1430/1430 TS，0 循环依赖）与 `pnpm run test:multiplayer-suite`（全部 28+ 项单测）**全部 PASS**。
- **端到端 E2E（`pnpm run repro:dev-broadcast-v3`）**：**尚未全部解决，暂不建议合入并推送 `origin/z/agy/rc/multiplayer-recheck-base`**。
  - 原样 5 分支 Octopus Merge 实测：**`reproduced=2/5`**（Bug 1/4/5 `PASS_FIXED`，**Bug 2 与 Bug 3 `REPRODUCED_BUG`**）。
  - 定位并本地验证修复了 Bug 2/3 之间的 `resetEpoch` 状态不同步后：Bug 2 转为 `PASS_FIXED`，但暴露出 **Bug 3 / Bug 4 / Bug 5 仍存在真实残留缺陷（`reproduced=3/5`）**。
- **Storage / 环境底座（`8718d96`）验收**：**合格**。`bootstrap.sh` 与 `/tmp` 重置后的 `doctor.mjs --auto-restore` 均一次通过，OverlayFS 稳定在 **29 MB**（远低于 100 MB 预算）。

---

## 2. 5 分支 Octopus Merge 与门禁实测明细

### 2.1 合入分支清单（基于 `8718d96`）

1. `origin/fix/bug3-lookahead-monotonic-tick` (`a657c49`) — `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
2. `origin/fix/bug3-replay-buffer-cadence` (`0ecf9e5`) — `src/network/sync/ReplicationStateInterpolator.ts`
3. `origin/fix/bug1-goal-replay-snapshot-stream` (`7147838`) — `scripts/runMultiplayerSuiteTests.ts`, `src/game/modes/freeplay/freeplayGoalReplaySessionPipeline.ts`, `src/replay/ReplayPlaybackState.ts`
4. `origin/fix/bug5-remaining-geometry-leak` (`0d9d4ac`) — `src/entities/arena/BoostPadShadersAndMeshes.ts`, `src/entities/trajectory/BallTrajectoryTubeRenderer.ts`
5. `origin/fix/bug2-nametag-render-stability` (`b669bf1`) — `scripts/runDevBroadcastReproV3.mjs`, `src/entities/arena/VehicleNameTagSystem.ts`

执行结果：`Merge made by the 'octopus' strategy.`（0 冲突，文件修改完全正交）。

### 2.2 静态检查与单测

- `pnpm run check`：**PASS**（1430/1430 TS，0 Circular Deps，`tsc --noEmit` 0 errors）。
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**PASS**（全部用例通过，含新增 Test 28）。

### 2.3 `pnpm run repro:dev-broadcast-v3` E2E 实测

#### 轮次 A：原样 5 分支 Octopus Merge（`reproduced=2/5`）

| 用例 | 状态 | 关键观测数据 |
| :--- | :--- | :--- |
| `bug-1-goal-replay-client-static-no-dynamics` | **PASS_FIXED** | `enteredReplay: true`, `distinctBallPositions: 3`, `bufFrames: 5 -> 8 -> 12` |
| `bug-2-replay-nametag-and-team-desync` | **REPRODUCED_BUG** | `arenaCarTeams: [0,0,0,1,1,1]`, 但 `client.numCars: 1`, `client.tags: []` |
| `bug-3-replay-choppy-motion-no-physics-lookahead` | **REPRODUCED_BUG** | `distinctTicks: 1`, `frozenBallFrames: 29`, `maxBallStepUu: 0`, 全部采样停在 `tick: 14043, (0, 0, 93.2)` |
| `bug-4-boostpad-model-flicker` | **PASS_FIXED** | `totalPadFlips: 0`, `maxSinglePadFlips: 0` |
| `bug-5-high-memory-and-unreleased-on-exit` | **PASS_FIXED** | `geometries: 37 -> 67 -> 26` (`retainedGeometries: -11`) |

**根因定位（Bug 2 & Bug 3 联动失效）**：
- `0ecf9e5` (`ReplicationStateInterpolator.ts`) 在 `SpectateInterpolationBuffer.pushFrame()` 检测到 `numCars !== lastNumCars` 或 `tick < last.tick - 12` 时会内部调用 `this.reset()`。
- `a657c49` (`spectatePhysicsLookaheadPipeline.ts`) 在 `SpectateLookaheadState` 中记录了 `lastSampledTick`，并丢弃所有 `sampled.renderTick + 0.5 < state.lastSampledTick` 的样本。
- 当测试从 Bug 1（1 车 Freeplay Goal Replay，`tick ~ 14043`）切换至 Bug 2/3/4（6 车 Replay Viewer，`tick ~ 719`）时，`SpectateInterpolationBuffer` 内部触发了 `this.reset()`，但 `SpectateLookaheadState` 未被同步重置，导致 `lastSampledTick = 14043` 拦截了所有 Replay Viewer 新帧，Client 永远停留在 1 车 `tick 14043`（直接造成 Bug 2 `numCars: 1, tags: []` 与 Bug 3 `frozenBallFrames: 29`）。

#### 轮次 B：本地验证 `resetEpoch` 同步补丁后（`reproduced=3/5`）

在 `SpectateInterpolationBuffer` 增加 `resetEpoch` 并在 `applySpectateSampleOrLookahead` 同步重置后重测：

| 用例 | 状态 | 关键观测数据 |
| :--- | :--- | :--- |
| `bug-1-goal-replay-client-static-no-dynamics` | **PASS_FIXED** | `enteredReplay: true`, `distinctBallPositions: 4`, `bufFrames: 2 -> 17 -> 37` |
| `bug-2-replay-nametag-and-team-desync` | **PASS_FIXED** | `host.numCars: 6`, `client.numCars: 6`, `teamMismatch: false`, `tagTextMismatches: []` |
| `bug-3-replay-choppy-motion-no-physics-lookahead` | **REPRODUCED_BUG** | `distinctTicks: 11`, `frozenBallFrames: 23`, `maxBallStepUu: 3936.6`（`tick: 255 -> 281 -> 290 -> 367 -> 547 -> 745` 大跨度跳变） |
| `bug-4-boostpad-model-flicker` | **REPRODUCED_BUG** | `totalPadFlips: 8`, `maxSinglePadFlips: 1`（跨 198 ticks 大跳变导致同一采样窗口内 8 个 BoostPad 状态翻转） |
| `bug-5-high-memory-and-unreleased-on-exit` | **REPRODUCED_BUG** | `geometries: 37 -> 105 -> 45` (`retainedGeometries: 8`，Replay 真正推进到 `tick 745` 后新分配的 Geometry 在退出时仍有 8 个未释放） |

### 2.4 后续处理建议（未合入原因与修复方向）

1. **修复 `SpectateInterpolationBuffer` 与 `SpectateLookaheadState` 的 `resetEpoch` 同步**（解决 Bug 2/3 跨模式切换被旧 `lastSampledTick` 拦截的问题）。
2. **排查 Bug 3 大跨度跳帧（`367 -> 547 -> 745`）**：当 Client 标签页被 `Page.bringToFront` 置前、Host 与 Client 双开 SwiftShader WebGL2 渲染时，Host 的 Replay Viewer 出现单帧推进 180+ ticks 的卡顿，Client lookahead 触及上限后产生 `3936.6 uu` 瞬移，并连带触发 Bug 4 `totalPadFlips >= 8`。需确认 Host Replay 推进节拍（避免单帧大步长跳跃）及 Client 平滑插值/lookahead 策略。
3. **排查 Bug 5 剩余 8 个 Geometry 泄漏**：在 Replay 真正播放至 `tick 745` 时，`loadedMem.geometries` 从之前的 `67` 升至 `105`，退出后残留 `45 - 37 = 8` 个，需针对高帧数播放阶段产生的实体/特效几何体补齐 `dispose()`。

---

## 3. Storage / 环境底座（`8718d96`）验收结果

- **实测表现**：
  - 全新容器执行 `git clone` + `bash scripts/env/bootstrap.sh`：自动完成 `pnpm` tmpfs 安装、`public/` 子模块克隆、`node_modules` tmpfs 软链安装、Chrome WebGL2 检测与 3v3 `.rcreplay` WASM fixture 生成，OverlayFS 仅 **29 MB**（预算 `<= 100 MB`）。
  - 执行过程中容器触发一次 `/tmp` 快照重置，`pnpm` 自愈包装器立即准确提示 `node scripts/env/doctor.mjs --auto-restore`，运行后 **10 秒内自动恢复全部 tmpfs 依赖与 fixture**。
- **结论**：当前 Storage 底座与 `基线与环境约定` 模板**合格可用**。

---

## 4. 专项设计建议

### 4.1 统一 5 分钟硬超时 Watchdog 与流式输出机制

为避免在每个测试脚本中分散硬编码 `260s` / `RC_SUITE_WATCHDOG_MS=280000`，建议引入统一配置文件与通用包装器：

1. **单一超时配置源（如 `scripts/env/watchdog.config.json`）**：
   ```json
   {
     "containerHardLimitSec": 300,
     "safeMarginSec": 30
   }
   ```
   所有脚本统一计算 `safeLimitMs = (containerHardLimitSec - safeMarginSec) * 1000`（默认 `270000ms`，预留 30s 安全余量）。
2. **通用 Watchdog 执行器（`scripts/env/withWatchdog.mjs`）**：
   - `package.json` 中的长耗时脚本统一通过 `node scripts/env/withWatchdog.mjs -- <command>` 启动，测试脚本自身保持纯粹通用、无需各自维护定时器。
   - **流式输出（Stream Output）**：子进程 `stdio` 实时透传并按行解析/记录最近完成的阶段标签（例如 `[PHASE] ...` 或 `[PASS] ...`），确保不因缓冲丢失已完成的输出。
   - **超时优雅退出与诊断**：到达 `safeLimitMs` 时，立即向子进程树（含 Chrome/Vite）发送 `SIGTERM`/`SIGKILL`，并输出结构化诊断信息（已耗时、超时阈值、最后完成的测试阶段、当前卡住的阶段），随后以非零退出码提前退出，避免触及 300s 平台硬截断。

### 4.2 Cloudflare R2 静态资源（`public/`）无鉴权加速与 Agent 自动发现设计

1. **为什么适合 Cloudflare R2**：
   - Cloudflare R2 开启 **Public Bucket + 自定义域名** 后，**Egress 出口流量费为 $0**，纯下载不消耗流量额度，且无需配置鉴权 Token（若后续有私有资产需求，才需前置极简 Worker 校验 `x-agy-request-id`）。
   - 相比 GitHub `git-upload-pack`（含 `.git` pack 约 182 MB，代理链路下握手慢），Cloudflare 边缘 CDN 对机房 IP 更友好，单包下载解压可将 `public/` 恢复时间从 12–60s 压缩到 **2–4s**，同时省去 `.git` 目录约 90 MB 的 tmpfs 占用。
2. **核心难点与解决思路（让 Agent 与脚本自动走 R2 加速）**：
   - 难点不在上传，而在**版本一致性**与**零心智负担的自动发现**。
   - **按 Gitlink SHA 不可变寻址**：CI / 发布脚本在 `RocketCompanionStatics` 更新时，将静态资源打包为 `public-<commitSha>.tar.gz`（不含 `.git`）上传至 R2。
   - **仓库内声明镜像源（如 `scripts/env/staticsMirror.json`）**：在主仓库提交公开的 R2 基础域名配置（例如 `{ "r2BaseUrl": "https://rc-statics.example.com" }`），Agent 与 `bootstrap.sh` 无需从外部 Prompt 记忆 URL。
   - **`tmpfsGuard.mjs --sync-public` 优先走 R2 + 自动回退**：
     1. 通过 `git ls-tree HEAD public` 读取当前主仓库锁定的 `public` submodule SHA。
     2. 优先请求 `${r2BaseUrl}/public-${sha}.tar.gz`，流式解压至 `/tmp/rc-cache/public`。
     3. 若 R2 不可达或该 SHA 尚未发布归档，自动无缝回退到 `git clone --depth 1 --single-branch`。
