# RocketCompanion `z/agy/rc/multiplayer-recheck` 分叉核查、合并验证与 E2E 现状报告

## 1. 结论摘要

1. **不能直接对这 7 个 commit 的端头做无脑 Octopus Merge**：
   - 其中 **2 条功能分叉**（`origin/z/agy/rc/multiplayer-recheck` 与 `origin/z/agy/rc/wave2-lookahead-buffer`）**可以干净自动合并**，但运行时尚未真正联动。
   - 另外 **2 条基础设施分叉**（`origin/feat/e2e-env-bootstrap` 与 `origin/z/agy/rc/env-doctor-repro`）存在功能重叠与文件冲突（`package.json`、`scripts/runDevBroadcastReproV2.mjs`、`scripts/runDevBroadcastReproV3.mjs`、`scripts/env/*`），需要权衡整合方案后再合并。
2. **合并后 Bug 并没有全部解决**：
   - 在本地合并 `4ab7525` (W2-A) + `4ec70dc` (W2-B) 后，`pnpm run check` 与 `pnpm run test:multiplayer-suite`（26 项单测）**全部通过**；
   - 但实跑 Headless Chrome + WebGL2 + WASM 端到端套件 `pnpm run repro:dev-broadcast-v3` 时，**5 个 Bug 中仍有 4 个被复现（仅 Bug 4 修复通过）**。
3. **关于你记忆中的 `tmpfs` 软链接 / `overlayfs` 自动检测与 `restore` 建议**：
   - 经逐一核查全部 7 个 commit 的实际代码变更，**没有任何 commit 包含 `tmpfs` 软链接、`overlayfs` 替换或 `tmpfs` 重置检测/restore 提示**。两条基础设施分叉实际上解决的是：`pnpm` / `public/` submodule / `node_modules` / Headless Chrome WebGL2（含代理绕过）/ Rust & Tauri C 工具链 / `.rcreplay` fixture 自动生成。

---

## 2. 7 个 Commit 的真实谱系与分类（架构 vs Bug 修复 vs 基础设施）

共同基座是 `2565774`（`merge(wave0): octopus merge Arch-0A, Arch-0B, Arch-0C, Arch-0D into z/agy/rc/multiplayer-recheck`）。**注意**：Bug 1（`476b783`）、Bug 2（`371043e` / `5fbc87d`）、Bug 5（`0bc5c69`）的早期修复以及 Wave 0 四条架构重构（`SimulationDrivePolicy`、`MatchRosterProjection`、`RuntimeResourceRegistry`、BoostPad 离散字段隔离）**已经包含在 `2565774` 中**。

在 `2565774` 之上分出了 **4 条独立分支**：

| 分支 / 端头 | Commit | 类型 | 实际内容 |
| :--- | :--- | :--- | :--- |
| `origin/z/agy/rc/multiplayer-recheck` | `1114df1` | **Bug 4 修复** (Wave 3 C-Track) | `BoostPadSystem.ts` 跳过重复帧 + 冷却微抖动防抖（已合入基线） |
| `origin/z/agy/rc/multiplayer-recheck` | `4ec70dc` | **Bug 3 管线侧** (Wave 2 B-Track) | `simulationFrameClassifierPipeline.ts` / `simulationNetworkReconcilePipeline.ts` / `spectatePhysicsLookaheadPipeline.ts`：在 `client-csr-spectate` 下启用 `shouldRunLookahead`，在重复帧时步进本地 WASM 物理并平滑收敛 |
| `origin/z/agy/rc/wave2-lookahead-buffer` | `4ab7525` | **Bug 3 缓冲侧** (Wave 2 A-Track) | `ReplicationStateInterpolator.ts` / `spectateLookaheadPipeline.ts`：为 `SpectateInterpolationBuffer` 增加 `>=5` 帧 lookahead 窗口与 `empty/buffering/tracking/lookahead/stalled` 游标状态机 |
| `origin/feat/e2e-env-bootstrap` | `7269d90` -> `bf1f4a6` | **基础设施 A（自动安装 + 代理绕过）** | `scripts/env/bootstrap.sh`（自动装 `pnpm`、`public/` submodule、`node_modules`、`chromium`、可选 `--with-rust` 装 `gcc`/Tauri C 依赖 + Rust）、`launchHeadlessChrome.mjs`（统一 Chrome 启动并绕过代理修复 WebGL2）、`generateSyntheticReplayFixture.mjs`（合成 `.rcreplay`）、`checkEnv.mjs` |
| `origin/z/agy/rc/env-doctor-repro` | `e302483` -> `69d052b` | **基础设施 B（诊断建议 + WASM 真实录像生成）** | `scripts/env/doctor.mjs`（诊断 Node/pnpm/Chrome/SwiftShader WebGL/可选 `--require-rust` 并输出修复命令）、`scripts/env/setup.sh`、`reproPreflight.mjs`、`generateReplayFixture.ts`（用 WASM 实际模拟 6-Ducky 3v3 对局，269ms 生成真实 `.rcreplay`） |

> 注：你提供的 `e3024833c834520ed6993d3b23842eb9eb9a7156` 即是 `origin/z/agy/rc/env-doctor-repro` 上 `69d052b` 的父提交 `e302483`。

---

## 3. 工具链补齐与合并实测结果

### 3.1 沙箱环境补齐
- 已安装 `pnpm@10.28.2`，初始化 `public/` submodule，完成 `pnpm install --no-frozen-lockfile --ignore-scripts`。
- `pixeldrain` 下载链接返回 HTTP 403，因此直接提取 `69d052b:scripts/env/generateReplayFixture.ts` 模拟 6-Ducky 3v3 WASM 对局生成了 `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`（231KB）。
- 注：沙箱当前未安装 Rust/Tauri C 编译工具链（`cargo`/`gcc`/`pkg-config`），本轮重点验证 JS/TS/WASM/Headless Chrome WebGL2 合并与 E2E 复现。

### 3.2 本地合并与测试结果（分支 `work/recheck-integration`，未 push 到远端）
在 `origin/z/agy/rc/multiplayer-recheck` (`4ec70dc`) 上合并 `origin/z/agy/rc/wave2-lookahead-buffer` (`4ab7525`)：
- **合并冲突**：零冲突自动合并（`scripts/runMultiplayerSuiteTests.ts`、`ReplicationStateInterpolator.ts`、`spectateLookaheadPipeline.ts`）。
- **`pnpm run check`**（架构雷达 + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 hard errors）。
- **`pnpm run test:multiplayer-suite`**：**26/26 全部 PASS**。
- **`pnpm run repro:dev-broadcast-v3`**（Headless Chrome E2E）：**`reproduced=4/5`**，明细如下：

| 用例 | 状态 | 实测数据与根因分析 |
| :--- | :--- | :--- |
| **Bug 4** `bug-4-boostpad-model-flicker` | **PASS_FIXED** | `totalPadFlips: 5`, `maxSinglePadFlips: 1`（阈值 `< 2` 且总数 `< 8`），`1114df1` 修复生效。 |
| **Bug 1** `bug-1-goal-replay-client-static-no-dynamics` | **REPRODUCED** | Client 进入 `phase: "replay"`、`hudPhase: "replay"`，但 `bufFrames: 1`，球停在 `(0, 0, 93.2)`，`distinctBallPositions: 1`。Wave 0 重构后 Client Goal Replay 帧流驱动仍未跑通。 |
| **Bug 2** `bug-2-replay-nametag-and-team-desync` | **REPRODUCED** | `teamMismatch: false`（双方 `arenaCarTeams` 均为 `[0,0,0,1,1,1]`）；但 Host 渲染出 4 个 NameTag（`car2, car3, car5, car4`），Client 仅返回 1 个（`car1: Ducky_Blue_2`），存在标签缺失。 |
| **Bug 3** `bug-3-replay-choppy-motion-no-physics-lookahead` | **REPRODUCED** | `distinctTicks: 1`（固定在 `14482`）、`frozenBallFrames: 27`、`maxBallStepUu: 453.1`。**根因**：W2-A (`4ab7525`) 与 W2-B (`4ec70dc`) 并行开发未对齐——W2-B `applySpectateSampleOrLookahead` 仅在 `!sampled` 或 `sampled.isDuplicateFrame === true` 时才步进 WASM 物理，而 W2-A 在 `lookahead` 状态下把 `isDuplicateFrame` 设为 `false`，导致物理步进被跳过；此外 Replay Watch Together 时 Client `replayPlaybackMode` 为 `true` 会直接绕过 `useSpectateBuffer`。 |
| **Bug 5** `bug-5-high-memory-and-unreleased-on-exit` | **REPRODUCED（仅残留项）** | `bootMem.wasmMB: 64`（已降到 64MB）、`retainedWasmMB: 0`、`tauriCloseHidesWindow: false` 均已修复；但退出回放后仍残留 `retainedGeometries: 11`（`40 -> 111 -> 51`）与 `retainedJsMB: 65`，触发了 `retainedGeometries > 0` 阈值。 |

---

## 4. 需要你决策的事项（因此暂未向远端 push 提交）

1. **两条基础设施分支（`feat/e2e-env-bootstrap` vs `z/agy/rc/env-doctor-repro`）如何整合？**
   - 推荐方案（取长补短合并为一套 `scripts/env/`）：
     - 保留 `feat/e2e-env-bootstrap` 的 `launchHeadlessChrome.mjs`（统一 Chrome 启动 + 代理绕过）和 `bootstrap.sh` 的 C/Tauri + Rust 自动安装逻辑；
     - 保留 `z/agy/rc/env-doctor-repro` 的 `generateReplayFixture.ts`（真实 WASM 6-Ducky 3v3 录像生成，比合成 fixture 更贴近真实物理）与 `doctor.mjs` / `reproPreflight.mjs` 的诊断建议模式。
2. **是否继续修完剩余的 E2E 阻断项再合入 `z/agy/rc/multiplayer-recheck`？**
   - 优先级 1：打通 W2-A 与 W2-B 的运行时联动（修正 `applySpectateSampleOrLookahead` 对 `sampled.isLookahead` 的处理，以及 Replay Watch Together 下 `useSpectateBuffer` 的准入条件），解决 **Bug 3**。
   - 优先级 2：排查 Wave 0 之后 **Bug 1**（Client Goal Replay 仅收到 1 帧）与 **Bug 2**（Client NameTag 仅渲染 1 个）的残留原因。
   - 优先级 3：清理 Replay Viewer 退出时残留的 11 个 Three.js `geometry`（**Bug 5**）。

请确认你希望采用哪套整合方案，以及是否继续修复上述 4 个残留 E2E 问题并推送 Verified Bot Commit 到指定分支。
