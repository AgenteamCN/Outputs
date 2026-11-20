# RocketCompanion 审查、环境实测与三连修未决 Bug 根因及后续协同计划

**目标分支**：`origin/z/agy/rc/multiplayer-recheck-base`（HEAD: `02adc3d1a404858c8186fee283658b69c2b3b3fa`）  
**执行内容**：双 Commit 审查、已合并分支清理、冷/温启动环境脚本实测、全局 270s 软超时与流式输出审查、`bug-1/2/3` 屡修未过根因诊断与下一步计划。

---

## 1. 两个新增 Commit 的审查结论

| Commit SHA | 所在分支 | 审查结论 | 原因说明 |
| :--- | :--- | :--- | :--- |
| `02adc3d1a404858c8186fee283658b69c2b3b3fa` | 已在 `z/agy/rc/multiplayer-recheck-base` (HEAD) | **保留（Keep）** | 修复了跨模式切换（Goal Replay `numCars=1` -> 3v3 Replay `numCars=6`）或向后 seek > 12 ticks 时未重置 `SpectateLookaheadState` 导致拒绝新快照的问题，且首帧不再被旧 `monotonicFloor` 卡住。逻辑干净、必要。 |
| `6889b8627d69c5f240334f6db6533ba7d2b9db83` | `origin/fix/bug5-replay-geometry-leak`（基于旧基线 `7252e42`） | **不建议合并 / 丢弃（Discard）** | 1. 当前基线 `02adc3d` 上 `bug-5` 已 `PASS_FIXED`（`retainedGeometries: -3`），无需此补丁。<br>2. **存在副作用风险**：其中 `disposeArenaHiddenEffectGeometries` 会遍历 `arena.cars[0]`、`carDemolitions[0]`、`carBoosts[0]`、`SpeedTrail_PS-ribbon`、`GoalBarrierSurface`、`ball-locator-arrow` 等对象，**只要节点当前 `visible === false` 就直接 `geometry.dispose()`**。这些属于对象池/复用特效，退回 Freeplay 后再次触发进球或加速时会因几何体已被销毁而出现 WebGL 报错或特效消失。 |

---

## 2. 已合并远端分支清理结果

已通过 GitHub REST API 删除 `origin` 上 **10 个已合入 `z/agy/rc/multiplayer-recheck-base` 的 `fix/bug*` 分支**（全部返回 `HTTP 204`）：
- `fix/bug-1-goal-replay-client-dynamics`
- `fix/bug-2-replay-nametag-team-sync`
- `fix/bug-5-memory-release-and-tauri-close`
- `fix/bug1-goal-replay-client-dynamics`
- `fix/bug1-goal-replay-snapshot-stream`
- `fix/bug2-nametag-render-stability`
- `fix/bug3-lookahead-monotonic-tick`
- `fix/bug3-lookahead-wiring`
- `fix/bug3-replay-buffer-cadence`
- `fix/bug5-remaining-geometry-leak`

> 注：`origin/fix/bug5-replay-geometry-leak`（`6889b86`）暂未删除，待你确认丢弃后可一并清理。

---

## 3. 环境脚本实测结果（当前容器 `/tmp` 为空场景）

1. **冷启动 `bash scripts/env/bootstrap.sh`**：**通过**
   - 自动完成 `pnpm@10.28.2` 安装、`public/` 单次直拉（`e66c0523`）、`pnpm install`（9.9s）、3v3 `.rcreplay` 生成（281ms）及探针校验。
   - OverlayFS 占用：**29 MB**（预算 <= 100 MB），Headless WebGL2（`WebKit WebGL`）正常。
2. **温启动 `node scripts/env/doctor.mjs --auto-restore`**：**通过（耗时 1.52s）**。
3. **`pnpm run env:help`**：**正常输出**，自愈说明与命令对齐。

---

## 4. 全局 4.5 分钟（270s）软限制与 Streaming Output 审查

**现状结论：尚未统一！** 目前各脚本各自硬编码超时，缺少统一的全局软限制工具与一致的流式输出：

| 类别 | 当前实现情况 | 风险 / 缺口 |
| :--- | :--- | :--- |
| **270s（4.5m）** | `runScenarioBaseline.mjs`、`runMultiplayerRoomRepro.mjs`、`runMultiplayerChaosRepro.ts` | 符合“5 分钟上限 - 30s 安全裕度 = 270s”要求 |
| **260s** | `runDevBroadcastReproV3.mjs`、`runDevBroadcastReproV2.mjs`、`runDevSyncReproSuite.mjs`、`runDevSyncExtraRoutesE2E.mjs` | 略保守但可用，未统一 |
| **180s** | `runDevRoomSyncE2E.mjs`、`runSoakWatchdog.mjs` | 短任务阈值 |
| **280s（裕度不足）** | `runMultiplayerSuiteTests.ts`、`scripts/env/bootstrap.sh` | 仅留 20s 裕度（低于要求的 30s 安全裕度） |
| **缺失 Watchdog** | `scripts/runMultiplayerSyncIssuesRepro.ts`（`repro:sync-issues`） | 无软超时保护，可能触及 300s 硬截断 |
| **Streaming Output** | 仅部分脚本（如 `runMultiplayerChaosRepro.ts`）有实时流式事件输出 | 多数脚本在末尾汇总打印，一旦超时容易丢失中间遥测 |

**建议**：抽取统一的 `scripts/lib/softTimeoutWatchdog.mjs`（默认 `RC_SOFT_LIMIT_MS=270000`，支持环境变量覆盖、`.unref()` 定时器、逐阶段/逐用例立即 `stdout` flush），覆盖全部 `test:*`、`repro:*` 与 `bootstrap.sh`。

---

## 5. 为什么 `bug-1 / bug-2 / bug-3` 修了三轮仍未全过？根因拆解

基于对 `scripts/runDevBroadcastReproV3.mjs` 与 `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts` 的代码走查，这 3 个问题**并非同一个原因**，而是**需求不明确 + 测试夹具副作用 + 局部状态机边界**交织所致：

1. **`bug-1-goal-replay-client-static-no-dynamics`（修复引入的时长压缩 + 测试窗口错位）**
   - `7147838` 裁剪了 Freeplay Goal Replay 前置空闲帧，导致从发球到进球的 Replay 总时长被压缩到仅 ~1 秒。
   - 测试中 `waitFor('Client enters Goal Replay')` 完成后再间隔 200ms 采样 6 次（共 1.2s），前 2 帧在 `phase: "replay"`，第 3 帧已经结束并切回 `phase: "playing"`（`0, 0, 93.2`），导致 `distinctBallPositions` 只有 `2`（要求 `> 2`）。
   - **本质**：需先明确 Goal Replay 最小有效播放时长（或测试采样逻辑是否应只统计 `phase === 'replay'` 期间，还是需要确认在 `replay` 期间球确实在连续运动）。

2. **`bug-2-replay-nametag-and-team-desync`（需求不明确 + 测试断言与视角耦合）**
   - 数据层面 `arenaCarTeams` 已经完全一致（`[0,0,0,1,1,1]`）。
   - 失败原因是 `runDevBroadcastReproV3.mjs` 比较的是 `_visibleCards2D + _offscreenIndicators2D`，但 Host 与 Client 此时处于**不同摄像机视角**（`host.tags: [1,4,5,3]`，而 `client.tags: [2]`）。视锥体外的车辆本就不会出现在 `_visibleCards2D` 中。
   - **本质**：**需求未定义清楚**——Replay Watch Together 到底要求“NameTag **数据/阵营文本**同步（测试应对齐双方视角或直接校验全车 NameTag 数据）”，还是要求“Client **相机视角/POV** 也强制同步 Host”？

3. **`bug-3-replay-choppy-motion-no-physics-lookahead`（Chrome 后台 Tab 限频 + Lookahead 未接管冻结帧）**
   - 测试在采样前执行了 `await clientCdp.send('Page.bringToFront')`，导致 **Host Tab 退居后台**，Chrome 自动对后台 Tab 的 `requestAnimationFrame` 降频（Host 变成 1Hz~4Hz 突发性发送，单次跳变 41~54 ticks）。
   - 更关键的是：在 Host 突发的间隙，Client 连续 `23/29` 个 rAF 帧球位置完全冻结（`dist < 0.01`），说明 **Client 在 Replay Watch Together 模式下并没有通过 `spectatePhysicsLookaheadPipeline.ts` 持续步进补帧**（可能是 `simulationFrameClassifierPipeline` 中该模式下 `shouldRunLookahead` 为 `false`，或 `SpectateInterpolationBuffer` 处于等待/阻塞状态）。

---

## 6. 建议的下一步执行计划（支持边扫描边 Real-Time Chatting）

1. **第一阶段：并行只读扫描（3 组并行，快速出精准证据，不盲目改代码）**
   - **Scan Team A（Bug 1）**：追踪 Freeplay Goal Replay 裁剪后的实际帧数/播放时长，以及 Client 在 `phase === 'replay'` 期间每帧球坐标是否真正在变。
   - **Scan Team B（Bug 2）**：盘点 Replay Watch Together 的相机控制权与 `VehicleNameTagSystem` 全车数据来源，明确是改测试视角对齐还是改相机同步。
   - **Scan Team C（Bug 3 + Watchdog）**：追踪 Replay Watch Together 下 `shouldRunLookahead` 实际取值、Chrome 后台 Tab 降频影响，以及全仓库脚本统一接入 `270s` 软超时 + 流式输出清单。
2. **第二阶段：Real-Time Chatting 需求对齐（等你拍板以下 4 个问题）**
   - **Q1（Bug 2 验收标准）**：Replay Watch Together 时，Client 是否允许自由切视角（只同步各车 NameTag 数据与队伍色），还是必须跟随 Host 视角？
   - **Q2（Bug 1 验收标准）**：Freeplay 极短进球（如开球 1 秒内进球）时，Goal Replay 是否应有保底播放时长（例如 >= 2.5s），还是只要求“只要处于 replay 阶段就必须动态播放”？
   - **Q3（Bug 3 预期范围）**：Replay Watch Together（非实时对战）在 Host 掉帧/后台限频时，Client 应该开启 WASM 物理 Lookahead 外推，还是走平滑插值缓冲？
   - **Q4（分支清理）**：是否确认直接删除 `origin/fix/bug5-replay-geometry-leak`（`6889b86`）？
3. **第三阶段：一次性收敛修复 + 统一 270s Watchdog 落地**
   - 在规格对齐后，再按模块拆分并行修复 + 统一接入 `270s` 全局软超时与流式输出，避免继续“猜需求”式返工。
