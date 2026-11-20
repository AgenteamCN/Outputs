# RocketCompanion 分支统一整合报告与后续并行执行计划

## 1. 本轮已完成操作与验证结果

### 1.1 统一合并与远端清理（Verified Bot Commit）
- 已将 4 条分叉通过 **4-parent Octopus Merge** 统一整合至 **`origin/z/agy/rc/multiplayer-recheck`**：
  - **统一端头 Commit**：`7252e425069b52988cab014ddb8c85bfd237f879`（GitHub Verified Bot Signature: `verified: True`，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）
  - **合并父节点**：`4ec70dc` (`multiplayer-recheck`) + `4ab7525` (`wave2-lookahead-buffer`) + `bf1f4a6` (`feat/e2e-env-bootstrap`) + `69d052b` (`env-doctor-repro`)
- **基础设施取长补短整合**：
  - 保留 `launchHeadlessChrome.mjs`（统一 Chrome 启动参数与代理环境变量 `--no-proxy-server` 绕过，修复沙箱 WebGL2）与 `bootstrap.sh`（支持 `--with-rust` 自动安装 `gcc`/Tauri C 依赖及 Rust 工具链）。
  - 保留 `generateReplayFixture.ts`（用 WASM 实际模拟 6-Ducky 3v3 对局生成 `.rcreplay`）与 `doctor.mjs` / `reproPreflight.mjs` 诊断能力；`bootstrap.sh` 已优先调用 `generateReplayFixture.ts`（当 `public/arenamesh/manifest.json` 就绪时），否则回退到 `generateSyntheticReplayFixture.mjs`。
  - `package.json` 统一暴露：`env:check`、`env:doctor`、`env:setup`、`env:setup:rust`、`replay:generate-ducky`。
- **已清理的冗余远端分支**：
  - 已删除 `origin/z/agy/rc/wave2-lookahead-buffer`、`origin/feat/e2e-env-bootstrap`、`origin/z/agy/rc/env-doctor-repro`。

### 1.2 工具链安装与功能验证
- **工具链实装**：已执行 `bash scripts/env/bootstrap.sh --with-rust`，在沙箱完成 `pnpm@10.28.2`、`public/` submodule、`node_modules`、`gcc`/Tauri C 依赖、`cargo 1.100.0` 及 Headless Chrome WebGL2 探针验证（全部 `[OK]`）。
- **`pnpm run check`**（架构雷达 + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 hard errors）。
- **`pnpm run test:multiplayer-suite`**：**26/26 全部 PASS**。
- **`pnpm run repro:dev-broadcast-v3`**（Headless Chrome E2E）：**`reproduced=3/5`**（Bug 2 与 Bug 4 已稳定通过；Bug 1、Bug 3、Bug 5 仍需后续修复）。

---

## 2. 后续并行推进计划（可直接下发执行）

所有后续分支均从最新 **`origin/z/agy/rc/multiplayer-recheck` (`7252e42`)** 切出。以下 **4 条业务修复线 + 1 条环境工具线** 文件边界高度隔离，可 **5 路完全并行**，最后再做一次 Octopus Merge 收口。

### Track A（优先级 P0）：打通 Bug 3 — W2-A / W2-B 运行时联动与 Replay Watch Together 平滑
- **建议分支**：`fix/bug3-lookahead-wiring`
- **负责文件（独占）**：
  - `src/game/pipeline/stages/spectatePhysicsLookaheadPipeline.ts`
  - `src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`
  - `src/network/sync/ReplicationStateInterpolator.ts`
  - `scripts/runMultiplayerSuiteTests.ts`（最后合入时仅在末尾追加用例，冲突极小）
- **具体改造点**：
  1. 在 `applySpectateSampleOrLookahead` 中，当 `sampled.isLookahead === true`（W2-A 游标进入 lookahead 状态）或 `sampled.isDuplicateFrame === true` 时，均允许触发 `shouldRunLookahead` 的 WASM `physics.step(1)` + `acknowledgedControls` 预测与平滑收敛，而不是被 `!sampled.isDuplicateFrame` 拦住。
  2. 检查 Replay Watch Together 下 Client `reconciler.replayPlaybackMode` 为 `true` 时直接跳过 `useSpectateBuffer` 的分支，允许其走 `SpectateInterpolationBuffer` + lookahead 窗口（或确保 Host 广播节拍与 `tickCount` 递增）。
- **验收标准**：`repro:dev-broadcast-v3` 中 `bug-3-replay-choppy-motion-no-physics-lookahead` 变为 `PASS_FIXED`（`distinctTicks > 1` 且无 20+ 帧冻结后大跳变）。

### Track B（优先级 P0）：修复 Bug 1 — Client Goal Replay 静态无动态
- **建议分支**：`fix/bug1-goal-replay-client-dynamics`
- **负责文件（独占）**：
  - `src/game/pipeline/stages/HostBroadcastPublisher.ts`
  - `src/game/pipeline/stages/SimulationStage.ts`（注意：与 Track A 无重叠或仅在 `SimulationStage.ts` 局部，需在最终 merge 时核对）
  - `src/replay/ReplayTimelineController.ts` / `src/modes/FreeplayMode.ts`（Goal Replay 触发与客户端同步段）
- **排查与改造点**：
  1. 定位为何 Client 进入 `phase: "replay"` 后 `spectateInterpolationBuffer` 始终只有 `bufFrames: 1` 且球停在 `(0, 0, 93.2)`。
  2. 重点核查：`HostBroadcastPublisher.publishIfReady` 在 Freeplay Goal Replay 期间是否持续发包、Client 侧 `hasResetSignal`（`BALL_RESET` / `ROUND_STARTED`）是否因 `lastParsedPacket` 未消费而每帧重置 buffer，以及 `reconciler.drainedReplayPackets` 是否正确驱动 `ReplayTimelineController`。
- **验收标准**：`repro:dev-broadcast-v3` 中 `bug-1-goal-replay-client-static-no-dynamics` 变为 `PASS_FIXED`（`enteredReplay: true` 且 `distinctBallPositions > 2`）。

### Track C（优先级 P1）：修复 Bug 5 — Replay 退出后残留 Geometry 与 JS 堆
- **建议分支**：`fix/bug5-replay-geometry-leak`
- **负责文件（独占）**：
  - `src/replay/StandaloneReplayViewer.ts`（及其 sub-controllers：`ReplayViewerRosterSubController.ts` / `ReplayViewerCausalSyncController.ts`）
  - `src/game/runtime/RuntimeResourceRegistry.ts`
- **具体改造点**：
  1. 追踪 `StandaloneReplayViewer` 加载 3v3 `.rcreplay` 到退出回 Freeplay 过程中 `renderer.info.memory.geometries` 从 `40 -> 61 -> 41` 残留的 `1~11` 个 Geometry 来源（例如车体/尾焰/指示器/NameTag/轨迹线等临时 Mesh 未在 `dispose` 时调用 `geometry.dispose()`）。
  2. 将缺失的 `dispose()` 纳入 `RuntimeResourceRegistry` 作用域清理，使退出后 `retainedGeometries === 0`。
- **验收标准**：`repro:dev-broadcast-v3` 中 `bug-5-high-memory-and-unreleased-on-exit` 变为 `PASS_FIXED`（`retainedWasmMB <= 0` 且 `retainedGeometries <= 0`）。

### Track D（优先级 P2）：基础设施增强 — `tmpfs` 软链接 / OverlayFS 降耗与重置恢复提示
- **建议分支**：`feat/env-tmpfs-restore-guard`
- **负责文件（独占）**：
  - `scripts/env/bootstrap.sh`
  - `scripts/env/doctor.mjs` / `scripts/env/checkEnv.mjs`
  - `scripts/env/tmpfsGuard.mjs`（新建）
- **建议实现要点**：
  1. **缓存与构建产物软链接到 `tmpfs`（如 `/tmp/rc-cache`）**：将 `node_modules/.vite`、`target/`（Rust/Cargo 构建产物，体积最大、对 overlayfs I/O 压力最高）、Headless Chrome `user-data-dir` 以及 `.rcreplay` fixture 软链接到 `/tmp/`，显著降低容器 overlayfs 写入放大与磁盘占用。
  2. **`tmpfs` 重置自动检测与 `restore` 建议**：在 `/tmp/.rc-env-sentinel` 写入环境指纹（含 `CARGO_TARGET_DIR`、`.rcreplay`、Vite cache 状态）；当 `doctor.mjs` / `checkEnv.mjs` 检测到宿主机快照恢复导致 `/tmp` 清空（软链接成为悬空链接或 sentinel 丢失）时，**不直接静默失败**，而是明确打印可复制执行的恢复命令（例如 `pnpm run env:setup` / `pnpm run env:setup:rust` / `pnpm run replay:generate-ducky`），并在 `--auto-restore` 参数下自动重建软链接与 fixture。

### Track E（可选核查）：Bug 2 稳定性确认
- 本轮实测 `bug-2-replay-nametag-and-team-desync` 在合并后已转为 `PASS_FIXED`（`teamMismatch: false`，`tagTextMismatches: []`），但此前一次运行曾出现 Client 仅渲染 1 个 NameTag 的情况，建议 Track A/B 合入后连跑 3 次 `repro:dev-broadcast-v3` 确认无偶发时序抖动。

---

## 3. 最终合流方式（Octopus Merge）

当 Track A、B、C、D 各自完成并在独立分支通过 `pnpm run check` 后：
1. 以 `origin/z/agy/rc/multiplayer-recheck` 为基线执行一次 Octopus Merge（仅 `scripts/runMultiplayerSuiteTests.ts` 与 `SimulationStage.ts` 可能存在极小范围相邻行冲突，当场手工合并即可）。
2. 统一跑 `pnpm run check`、`pnpm run test:multiplayer-suite`、`pnpm run repro:dev-broadcast-v3`（目标：`reproduced=0/5`），确认全绿后清理临时分支。
