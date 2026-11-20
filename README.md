# RocketCompanion Octopus Merge 与全量回归验收报告

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **基线分支**: `origin/z/agy/rc/multiplayer-recheck-base`
- **Octopus Merge Commit**: `2fc5fc0a39aee3da59a3d00eee79a058eeb70eb2`（Verified Bot Commit，`verified: true`，3-Parent Octopus Merge）
  - Parent 1: `e8aae572a729b535bae90e74adaba069a4b41524`（`z/agy/rc/multiplayer-recheck-base`，已含 Team B `86998ce` 与 Team C `43e1fc4`）
  - Parent 2: `22d0160524d9c0609fded6c34265435d8b215762`（`z/agy/rc/fix-bug3-client-lookahead`）
  - Parent 3: `ba1b416309386dba348ef7b983f1f5b329b21d9c`（`z/agy/rc/fix-bug4-bug5-cleanup`）

---

## 一、Octopus Merge 与过程分支清理

1. **合并冲突情况**：**0 冲突**。两个并行修复分支修改文件完全正交（`22d0160` 修改 4 个调度/插值/Lookahead 文件，`ba1b416` 修改 2 个 Freeplay 清理与 E2E 脚本文件），通过 GitHub REST API `git/trees` -> `git/commits` -> `git/refs/heads/...` 完成 Verified Bot Octopus Merge。
2. **过程分支清理（已全部从 `origin` 删除，均返回 HTTP 204）**：
   - `z/agy/rc/fix-bug3-client-lookahead`
   - `z/agy/rc/fix-bug4-bug5-cleanup`
   - `z/agy/rc/team-b-bgworker-replay-len`
   - `z/agy/rc/track-c-bug1-bug2-stability`

---

## 二、全量回归验证结果（基于 `2fc5fc0`）

| 检查项 | 结果 | 关键数据 |
|---|---|---|
| `pnpm run check` | **GREEN** | `1430/1430` TS（100% TS），`tsc --noEmit` 0 errors |
| `pnpm run test:multiplayer-suite` | **28/28 PASS** | 全部联机/插值/Lookahead/Replay/BoostPad 单测通过 |
| `pnpm run repro:dev-broadcast-v3`（Run 1） | `reproduced=2/5` | Bug 1/2/4 **PASS_FIXED**；Bug 3、Bug 5 **REPRODUCED_BUG** |
| `pnpm run repro:dev-broadcast-v3`（Run 2） | `reproduced=3/5` | Bug 1/4 **PASS_FIXED**；Bug 2、Bug 3、Bug 5 **REPRODUCED_BUG** |

### `repro:dev-broadcast-v3` 逐项实测明细

1. **Bug 1（Goal Replay 动力学）**：**2/2 PASS_FIXED**
   - Run 1: `enteredReplay: true, distinctBallPositions: 6, sampleCount: 10`
   - Run 2: `enteredReplay: true, distinctBallPositions: 6, sampleCount: 8`
2. **Bug 2（NameTag 与 Edge-Triggered Camera Sync）**：**1/2 PASS_FIXED（偶发 15s 超时）**
   - Run 1: `PASS_FIXED`（`teamMismatch: false, carEntryMismatches: [], edgeSyncOk: true`）
   - Run 2: `REPRODUCED_BUG`（`teamMismatch: false`，`edge1` 与 `localSwitch` 成功，但 `clientAfterHostEdge2: null` 在 15s 超时）
3. **Bug 3（Client Spectate Lookahead 冻结与跳变）**：**2/2 REPRODUCED_BUG**
   - Run 1: `distinctTicks: 20, frozenBallFrames: 10, maxBallStepUu: 4413.3`（采样区间 `tick: 357 -> 1183`，单帧跨越 `+91 ~ +171 ticks`）
   - Run 2: `distinctTicks: 1, frozenBallFrames: 29, maxBallStepUu: 0`（全部 30 帧停在 `tick: 1819, pos: (0,0,93.2)`）
4. **Bug 4（BoostPad 闪烁）**：**2/2 PASS_FIXED**
   - Run 1: `totalPadFlips: 16, maxSinglePadFlips: 2, maxRapidFlipsIn400ms: 1`
   - Run 2: `totalPadFlips: 0, maxSinglePadFlips: 0, maxRapidFlipsIn400ms: 0`
5. **Bug 5（退出后内存与几何体残留）**：**2/2 REPRODUCED_BUG**
   - Run 1: `retainedWasmMB: 0, retainedJsMB: 57, retainedGeometries: 2`（`boot: 37 -> loaded: 65 -> afterExit: 39`）
   - Run 2: `retainedWasmMB: 0, retainedJsMB: 61, retainedGeometries: 10`（`boot: 37 -> loaded: 91 -> afterExit: 47`）

---

## 三、未过项根因分析（为什么合并后依然复现）

1. **Bug 3（Run 1 `maxBallStepUu: 4413.3` / `frozenBallFrames: 10`，Run 2 `tick: 1819`）**：
   - **Run 2 的 `tick: 1819` 是测试耦合导致的伪复现**：Run 2 中 Bug 2 等待 `clientAfterHostEdge2` 消耗了完整的 15s 超时，导致 1800 帧（15s@120Hz）合成录像在进入 Bug 3 采样前就已播到终点并回到开球点 `(0, 0, 93.2)`。
   - **Run 1 的 `maxBallStepUu: 4413.3` 是 SwiftShader 1–3 FPS 采样尺度放大**：在 SwiftShader 下 `rAF` 间隔可达 0.8–1.4s（单次 `rAF` 推进 100–171 ticks），高速球在 1.4s 内移动 4413 uu 属于真实物理位移，并非瞬移；而 `frozenBallFrames: 10` 说明当 Host 因 CPU 抢占短暂断流时，Client 触发 `cursorState === 'stalled'` 并按 `22d0160` 主动暂停了 Lookahead 外推。
2. **Bug 2（Run 2 `clientAfterHostEdge2: null`）**：
   - Bug 2 采样前 `Client` 在前台，随后测试让 Host 切换相机，但双窗口 SwiftShader 并发下 Host `SimulationStage` 或 Client `Runtime.evaluate` 轮询仍可能被单核 CPU 抢占超过 15s。
3. **Bug 5（`retainedGeometries: 2 / 10` 与 `retainedJsMB: 57 / 61`）**：
   - `retainedWasmMB` 已稳定为 `0`，但 `bootMem` 是在冷启动未渲染特效前采样（`37`），而 `ballSpeedTrail`、`carTrails[0]` 及部分延迟加载/未在 `afterExit` 同步等待释放的几何体（Run 2 仍有 `10` 个）导致差值不为 0；同时 `retainedJsMB`（57–61 MB）超过 40 MB 判定阈值。

---

## 四、下一步建议：直接 Real Device Test 还是额外规划？

**结论：建议「立即安排一轮真实设备验证（Real Device Test）+ 同步推进 2 项轻量收尾规划」，双线并行而非盲目继续在 SwiftShader 上死磕。**

### 1. 为什么可以进入 Real Device Test？
- **Bug 1、Bug 4** 在引擎逻辑与 E2E 上均已稳定通过；**Bug 2** 协议与状态同步已通过全部 28 项单元测试，偶发超时仅由 SwiftShader CPU 抢占引发。
- **Bug 3** 在真实 GPU（60–144 FPS）下不存在「单帧 `rAF` 耗时 1.4s = 4413 uu」的测量失真，真机实测能立刻区分：究竟是真机上已经丝滑（仅 SwiftShader 伪复现），还是 Host 切后台/网络抖动时仍有 `stalled` 卡顿。

### 2. 建议同步执行的额外规划（3 项具体动作）
1. **真机对比验证（Real Device Test）**：
   - 在真实桌面浏览器 / Tauri 双端（Host 切后台 Tab 或最小化 + Client 前台看 Replay Watch Together）实测 Bug 2 切车同步、Bug 3 球/车平滑度（是否仍有卡顿感）、Bug 5 退出到 Freeplay 后的实际内存与 DevTools Heap Snapshot。
2. **修复 `runDevBroadcastReproV3.mjs` 的测试耦合**：
   - 在进入 Bug 3 采样前显式调用 `rv.seekFrame?.(120)`（或开启录像循环），彻底隔离 Bug 2 超时对 Bug 3 采样窗口的污染，避免再出现 `tick: 1819` 播完边界。
3. **Bug 5 彻底清零收尾**：
   - 在 `ArenaWorld` 预热阶段对持久化特效池（`SupersonicWheelTrailSystem`、`ballSpeedTrail`）做统一基线上传，或并在退出时显式 dispose 非本地车几何体 + 等待 `runtimeResourceRegistry.disposeScopes()` 完成与触发可测的内存基线对齐（将 `retainedGeometries` 稳定压至 `0`，排查 `57–61 MB` JS Heap 主要持有者是否为 Replay 缓存/Three.js 缓存未释放）。
