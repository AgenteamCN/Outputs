# RocketCompanion 交接与深度定位报告（用于新对话 + Live Chatting 协同排障）

**当前集成基线**：`origin/z/agy/rc/multiplayer-recheck-base`（HEAD: `02adc3d1a404858c8186fee283658b69c2b3b3fa`）  
**分支清理状态**：所有 `origin/fix/bug*` 历史分支（含已确认丢弃的 `fix/bug5-replay-geometry-leak`）已全部清理完毕（`HTTP 204`）。  
**环境基线状态**：`bootstrap.sh`（冷启动 ~20s）、`doctor.mjs --auto-restore`（温启动 1.52s）、OverlayFS **29 MB**（预算 <= 100 MB）、Headless WebGL2 均正常可用。  
**已通过用例**：`pnpm run check`（1430/1430）、`pnpm run test:multiplayer-suite`（28/28）、`bug-4`（BoostPad 闪烁）、`bug-5`（几何体泄漏）。

---

## 一、核心结论：你的需求没有矛盾，问题出在“测试夹具耦合 + 架构耦合”

经过对用户澄清的规格与代码实现的逐一核对，**你的产品逻辑本身完全自洽**：

1. **Bug 2（Replay Watch Together NameTag & 视角）**：
   - **你的真实规格**：Host 加载 Replay 后全量广播 6 辆车的 roster / loadout / NameTag 数据（`carIndex 0..5`）；Camera / BallCam / Replay Watch Together 采用**边沿触发（Edge-Triggered）**同步——Host 切换时广播给所有 Client 同步切换，长期不切换时 Client 可本地自由切视角。
   - **为什么测试还报 FAIL**：`scripts/runDevBroadcastReproV3.mjs`（L349-352, L377-387）**没有校验全车 NameTag 数据映射**，而是去比较 `_visibleCards2D + _offscreenIndicators2D`（受屏幕视锥体裁剪）。由于 Host 与 Client 默认镜头角度不同（或未触发一次边沿切换），`host.tags` 只看见 `[1,4,5,3]`、`client.tags` 只看见 `[2]`，导致误报失败。
   - **待确认/定位方向**：
     1. 确认 Client 的 `nameTagSystem` 底层是否确实已收到全部 6 辆车的 NameTag 数据与队伍色（`arenaCarTeams` 已一致：`[0,0,0,1,1,1]`）；
     2. 确认 Replay 刚加载进入 Watch Together 时是否需要补发一次初始视角边沿事件，并修正 `runDevBroadcastReproV3.mjs` 的断言（直接校验 6 车 NameTag 数据，或在测试中触发一次 Host 视角切换来验证边沿同步）。

2. **Bug 1（Freeplay Goal Replay 时长与动态性）**：
   - **你的真实规格**：Goal Replay 环形缓冲上限为 **10 秒**；如果进球过程只有 2 秒就放 2 秒，超过 10 秒就放最后 10 秒，有多少放多少。
   - **为什么测试还报 FAIL**：`runDevBroadcastReproV3.mjs`（L245-248）为了快速造进球，把球直接瞬移到门前 `y=4850, vel_y=2950`，**0.1 秒就进门了**；叠加 `7147838` 裁剪前置空闲帧后，整段 Goal Replay 只剩 **~0.4 秒**。测试在 `waitFor('Host/Client enters Goal Replay')` 之后才开始每 200ms 采样一次，前 2 帧在 `replay`，第 3 帧就已切回 `playing`（`0, 0, 93.2`），从而触发 `distinctBallPositions <= 2` 报错。
   - **待确认/定位方向**：
     1. 检查在 `phase === 'replay'` 的有效帧内，Client 端的球和车是否**真正在连续运动**（排除“Replay 里依然静止”的真 Bug）；
     2. 如果 Replay 内本身是动态的，则修正测试造球距离（例如从半场 `y=1500` 射门，产生 >= 2 秒自然缓冲）且只统计 `phase === 'replay'` 期间的采样点。

3. **Bug 3（120Hz 物理 vs 变帧率渲染，以及后台 Tab 冻结）**：
   - **业界标准架构（120Hz 物理 + 144/165/180Hz 渲染）是怎么做的**：
     - **物理线程/逻辑步进**按固定时间步长 `dt = 1/120s (8.333ms)` 通过 **时间累加器（Accumulator）** 独立推进，**不与渲染帧率 1:1 绑定**。
     - 当显示器是 144Hz/165Hz/180Hz（甚至未来物理降到 90Hz 或升到 150Hz）时，每次渲染帧根据 `alpha = accumulator / fixedDt ∈ [0, 1)`，对相邻两个物理帧 `prevState` 与 `currState` 做**状态插值（`PhysicsStateInterpolator`）**后送去渲染。这样无论渲染跑 30 FPS 还是 180 FPS，物理始终精确按 120Hz 演算，画面丝滑无跳变。
   - **为什么当前测试会出现“23 帧冻结 + 单步跳变 1341.8 uu”**：
     1. **测试环境触发了 Chrome 后台 Tab 降频**：`runDevBroadcastReproV3.mjs`（L326, L400）调用了 `clientCdp.send('Page.bringToFront')`，导致 **Host Tab 退居后台**。因为当前 Host 的 Replay 播放与广播仍挂载在 `requestAnimationFrame` 主循环上（未完全与 `rAF` 解耦），Chrome 对后台 Tab 的 `rAF` 强行降频成 1~4Hz 突发，Host 每次突发直接跨 41~54 ticks。
     2. **Client 端 Lookahead / 插值在突发间隙没有平滑接管**：在 Host 两次突发之间的 23 个 Client `rAF` 帧里，Client 球坐标完全不动（`dist < 0.01`）。需要排查在 `client-csr-spectate` / Replay Watch Together 模式下：`shouldRunLookahead`（`src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`）与 `SpectateInterpolationBuffer` 为什么没有在等待新快照时平滑推进或插值。

---

## 二、全局 4.5 分钟（270s）软限制与 Streaming Output 缺口清单

目前仓库**尚未统一**全局软限制（容器硬上限 300s - 30s 安全裕度 = **270s / 270,000ms**）：
- **仍为 280s（裕度仅 20s）**：`scripts/runMultiplayerSuiteTests.ts`、`scripts/env/bootstrap.sh`
- **无 Watchdog**：`scripts/runMultiplayerSyncIssuesRepro.ts`（`repro:sync-issues`）
- **各自硬编码 260s / 270s / 180s**：`runDevBroadcastReproV2/V3`、`runDevSyncReproSuite`、`runDevSyncExtraRoutesE2E`、`runMultiplayerRoomRepro`、`runMultiplayerChaosRepro`、`runScenarioBaseline`、`runDevRoomSyncE2E`、`runSoakWatchdog`
- **Streaming Output 不一致**：仅部分脚本实时逐行输出，多数脚本在结束时才汇总打印，超时易丢遥测。

**改造目标**：新建统一的 `scripts/lib/softTimeoutWatchdog.mjs`（默认 `RC_SOFT_LIMIT_MS=270000`，支持 `.unref()` + `SIGINT/SIGTERM` 清理 + 逐用例/逐阶段实时 flush），并接入所有 `test:*` / `repro:*` / `bootstrap.sh`。

---

## 三、新对话 + Live Chatting 建议推进顺序

1. **并行只读定位（3 路齐发）**：
   - **路 1（Bug 1）**：实测打点 `phase === 'replay'` 期间每一帧的球/车坐标，确认是“真的静态不动”还是“测试 0.1s 进球太短”。
   - **路 2（Bug 2）**：核查 Replay Watch Together 下 Client `nameTagSystem` 的 6 车全量数据状态，以及 Host 加载 Replay 时初始视角边沿广播是否缺失。
   - **路 3（Bug 3）**：核查 Host 后台循环与 `rAF` 解耦方案（如 `MessageChannel`/Worker/`setTimeout` 驱动），以及 Client `SpectateInterpolationBuffer` + `spectatePhysicsLookaheadPipeline.ts` 为何出现 23 帧冻结。
2. **基础设施收口**：落地 `scripts/lib/softTimeoutWatchdog.mjs`（统一 270s 软限制 + Streaming Output）。
3. **一次性精准修复 + 全量回归**：修完后跑通 `pnpm run check`、`pnpm run test:multiplayer-suite`、`pnpm run repro:dev-broadcast-v3`（目标 `reproduced=0/5`）。
