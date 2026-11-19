# RocketCompanion Dev Broadcast / Replay / Memory 问题复现报告

**仓库**: `Antigravi-ty/RocketCompanion`  
**分支**: `z/agy/rc/multiplayer-recheck`  
**提交**: [`366a6869e67b0f2e4d597e661bf9b5da3073e6dd`](https://github.com/Antigravi-ty/RocketCompanion/commit/366a6869e67b0f2e4d597e661bf9b5da3073e6dd)（Verified Bot Commit）  
**新增测试脚本**: `scripts/runDevBroadcastReproV3.mjs`（`pnpm run repro:dev-broadcast-v3`）  
**原则**: 本次仅做**问题复现与测试套件更新**，未修改产品源码修复 bug。

---

## 一、复现汇总（5/5 全部稳定复现）

运行 `node scripts/runDevBroadcastReproV3.mjs`（双 Chrome Headless CDP + Vite Dev Server + BroadcastChannel / Pure CSR），输出 `reproduced=5/5`：

| 编号 | 场景 ID | 状态 | 核心复现指标 |
| :--- | :--- | :--- | :--- |
| 1 | `bug-1-goal-replay-client-no-motion` | `[REPRODUCED_BUG]` | `clientReplayFrames: 0`, `clientReplayPlaybackMode: false`, `clientReplayBallTravel: 0` |
| 2 | `bug-2-replay-watch-together-nametag-unsynced` | `[REPRODUCED_BUG]` | UI `EdgePlayerSlotCard`: `6`，`nameTagIsMultiCar: false`, `nameTagCarEntriesCount: 0`, `activeLabels: []` |
| 3 | `bug-3-broadcast-choppy-state-only-no-physics-lookahead` | `[REPRODUCED_BUG]` | `physicsStepCalls: 0`, `prevEqCurrRatio: 1`, `maxStepRatio: 3.89` (`medianDeltaUu: 566.2`, `maxDeltaUu: 2200.55`) |
| 4 | `bug-4-boost-state-flicker-every-frame` | `[REPRODUCED_BUG]` | `totalCarBoostingRapidToggles: 1`, `sampleCar0Boosting: [0,...,0,1,0,0,...]` |
| 5 | `bug-5-memory-footprint-and-close-retention` | `[REPRODUCED_BUG]` | Boot 后 WASM `271–325 MB` + JS Heap `162–200 MB`；Active `336.2 MB`；`dispose()` 后 `wasmMemoryBytes: 0` 但 JS Heap `337.4 MB`（`heapRetentionRatio: 1`）；Tauri `CloseRequested` 仅 `prevent_close() + hide()` |

---

## 二、逐项根因定位与复现证据

### 1. Client 端 Goal Replay 能正常进入/退出但无动态（Spectating 与 Match Cinematic Goal Replay 未打通）
- **复现表现**:
  - Client 端进入 Goal Replay 时，`hudPhase === 'replay'`、`phase === 'replay'`，但 `clientReplayPlaybackMode: false`、`clientReplayFrames: 0`。
  - 球与车辆位置完全静止在 kickoff 坐标 `(0, 0, 93.15)`（`clientReplayBallTravel: 0`、`clientInterpBallTravel: 0`），退出后重新同步正常。
- **代码根因**:
  1. `src/game/runtime/RuntimeGoalReplaySignalSubController.ts` 中 `handleGoalReplayStart` 在 `isDevSyncClientPureCsrSpectateMode(reconciler)` 为 `true` 时直接跳过 `reconciler.setReplayPlaybackMode(true)`。
  2. `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` 中 `if (isPureCsrSpectate && !reconciler.replayPlaybackMode && spectateInterpolationBuffer)` 分支把 `drainedReplayPackets` 全部送入 `spectateInterpolationBuffer`，导致后续 `else if` 分支中的 `replayTimelineController.appendStreamedReplayFrame(...)` 不可达，Goal Replay 时间轴帧数为 `0`。

### 2. Replay Watch Together 同步了 UI 信息，但 3D NameTag 未同步
- **复现表现**:
  - 双方进入 `activeRoomActivity: 'replay'` 后，Client 端 UI 侧已渲染 6 个 `EdgePlayerSlotCard`（UI Roster 与计分板已同步），但 3D NameTag 状态为 `nameTagIsMultiCar: false`、`nameTagCarEntriesCount: 0`、`activeLabels: []`。
- **代码根因**:
  - `src/game/runtime/devSyncChannelHandlers.ts` 处理 `replaySync` 时仅更新了 `useReplayViewerStore` 与 `hudStore`，没有调用 `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode: true, carEntries: ... })`（`VehicleNameTagSystem` 依赖该运行时状态渲染 3D NameTag）。

### 3. 广播回放卡顿（仅下发/应用 State 快照，未使用确定性物理引擎做缓冲前瞻）
- **复现表现**:
  - Client 端采样 20 帧：`physicsStepCalls: 0`、`prevEqCurrRatio: 1`（每帧 `prevState === currState`），单帧位移最大跳变 `2200.55 uu`（中位数 `566.2 uu`，`maxStepRatio: 3.89`）。
- **代码根因**:
  - `src/game/pipeline/stages/SimulationStage.ts` 在 `isPureCsrClient` 分支下直接执行 `interpolator.sync(timestamp)` 并跳过 `physics.step(...)`，完全依赖 `SpectateInterpolationBuffer` 的纯状态插值，未在缓冲区积累到一定帧数（如 `> 5` frames）时利用确定性物理引擎做多帧连续步进/前瞻。

### 4. Boost 状态逐帧闪烁
- **复现表现**:
  - Client 端 20 帧采样中检测到 `car[0].isBoosting` 出现 `0 -> 1 -> 0` 跳变（`totalCarBoostingRapidToggles: 1`，`carBoostingTogglesByCar: [1, 0, 0, 0, 0, 0]`）。
- **代码根因**:
  - 快照压缩/插值路径中，`globalPhysicsSnapshotSectionCodec.ts` 的 `unpackBoostPadBitmaskIntoSnapshot` 会在 pad inactive 时将冷却时间重置为 `BOOST_PAD_TIMEOUT_BIG`/`SMALL`，且 `interpolateReplaySnapshots` 在 `alpha = 0.5` 对 `car[19..39]` 离散开关做硬切换，加之网络快照与本地插值帧之间的状态切换导致 boost 状态高频闪烁。

### 5. 高内存占用与关闭窗口后不释放（需 Quit 才释放）
- **复现测量（Chrome CDP Performance / Heap / WebGL + Tauri 源码检查）**:
  - **Host 启动后**: `jsHeapUsedMB: 162`、`wasmMemoryBytes: 271056896`（约 `258.5 MB`）、`geometries: 40`、`programs: 33`。
  - **Client 启动后**: `jsHeapUsedMB: 199.8`、`wasmMemoryBytes: 325320704`（约 `310.2 MB`）。
  - **Client 活跃阶段**: `jsHeapUsedMB: 336.2`、`jsHeapTotalMB: 377.7`、`domNodes: 750`、`replayBufferFrames: 279`、`geometries: 62`、`programs: 58`。
  - **Client `runtime.dispose()` 后**: `wasmMemoryBytes` 降为 `0`、`domNodes` 降至 `24`，但 **JS Heap 保持 `337.4 MB`（`heapRetentionRatio: 1`，100% 滞留）**。
  - **Tauri 端 (`src-tauri/src/lib.rs`)**: `WindowEvent::CloseRequested` 调用了 `api.prevent_close(); window.hide();`，关闭窗口时仅隐藏窗口，并未销毁 WebView 或触发 `runtime.dispose()`，导致整个进程（以及 WASM/WebGL/JS Heap）驻留后台直到用户彻底 Quit。

---

## 三、验证与构建检查

- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1425/1425 TS 文件，0 循环依赖，0 错误）
- `pnpm run test:multiplayer-suite`：**24/24 PASS**
- `pnpm run repro:dev-broadcast-v3`：**5/5 REPRODUCED**
