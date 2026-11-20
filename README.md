# RocketCompanion 分支梳理、工具链现状与 Bug 复测报告

## 一、结论速览

1. **这 7 个 Commit 不能直接做 Octopus Merge**：
   - 其中 **2 个 Commit（`1114df1`、`4ec70dc`）已经在 `origin/z/agy/rc/multiplayer-recheck` 上**；
   - **1 个 Commit（`4ab7525`，Wave 2 A-Track）可以无冲突合并**到 `multiplayer-recheck`，且合并后 `pnpm run check` 与 `pnpm run test:multiplayer-suite`（26/26）全部通过；
   - 另外 **4 个 Commit 分属两个互相冲突的基础设施分叉**（`origin/z/agy/rc/env-doctor-repro` 与 `origin/feat/e2e-env-bootstrap`），共同修改了 `package.json`、`scripts/runDevBroadcastReproV2.mjs`、`scripts/runDevBroadcastReproV3.mjs`，无法直接 Octopus Merge，需要决策合并策略。
2. **Bug 并未全部解决**：
   - 我们在沙箱中补齐了工具链（`pnpm@10.28.2`、`public/` submodule、`node_modules`），并用 `generateReplayFixture.ts` 生成了真实 6-Ducky 3v3 `.rcreplay`（注：pixeldrain 返回 403，wenshushu 需前端交互，已用 WASM 3v3 仿真生成有效 V2 `.rcreplay`），实际跑了 `scripts/runDevBroadcastReproV3.mjs`（Headless Chrome + SwiftShader WebGL2 + WASM）。
   - **合并 `4ab7525` 前后，5 个 Bug 中仍有 4 个在真实 E2E 中复现（`reproduced=4/5`）**，仅 Bug 4（BoostPad 闪烁）基本修复（偶发 `maxSinglePadFlips: 2`）。

---

## 二、7 个 Commit 拓扑与分类（架构 vs 修 Bug vs 基础设施）

全部 7 个 Commit 均基于 Wave 0 架构重构 Octopus Merge（`2565774`，包含 `Arch-0A/0B/0C/0D`）之后：

| Commit | 所属分支 | 类型 | 实际作用 | 当前状态 |
| :--- | :--- | :--- | :--- | :--- |
| `1114df112b50e09bd93063455e7ba7467419f8e3` | `origin/z/agy/rc/multiplayer-recheck` | **修 Bug（Wave 3 C-Track / Bug 4）** | `BoostPadSystem` 跳过重复帧 + 冷却微抖动防抖 | **已在目标分支** |
| `4ec70dc6efce64790c2f560b20580ffd1c2c695f` | `origin/z/agy/rc/multiplayer-recheck` | **修 Bug（Wave 2 B-Track / Bug 3 半边）** | `SimulationDrivePolicy` 开启 `shouldRunLookahead`，在重复/预热帧用 `acknowledgedControls` 步进本地 WASM 并收敛 | **已在目标分支（HEAD）** |
| `4ab75253a8b1967d761d4460b959d38795df6fd4` | `origin/z/agy/rc/wave2-lookahead-buffer` | **修 Bug（Wave 2 A-Track / Bug 3 另半边）** | 为 `SpectateInterpolationBuffer` 增加 `>=5` 帧前瞻窗口与 `SpectateCursorState` 状态机 | **未合并，可无冲突合入** |
| `e3024833c834520ed6993d3b23842eb9eb3a7156` | `origin/z/agy/rc/env-doctor-repro` | **基础设施（分叉 A-1）** | `scripts/env/setup.sh`、`doctor.mjs`、`reproPreflight.mjs`（修复 Chrome `ENOTEMPTY`）、`generateReplayFixture.ts` | **与分叉 B 冲突** |
| `69d052b0567b333efc204b5a75eecdccc2f5058d` | `origin/z/agy/rc/env-doctor-repro` | **基础设施（分叉 A-2）** | 将 `generateReplayFixture.ts` 升级为真实 6-Ducky 3v3 WASM 比赛仿真生成 `.rcreplay`（~267ms） | **与分叉 B 冲突** |
| `7269d90b579c4799b419a0ac6f2e5d8a920a034a` | `origin/feat/e2e-env-bootstrap` | **基础设施（分叉 B-1）** | `scripts/env/bootstrap.sh`、`checkEnv.mjs`、`launchHeadlessChrome.mjs`（**关键：绕过容器代理 `NO_PROXY=127.0.0.1,localhost` + `--proxy-server=direct://`**） | **与分叉 A 冲突** |
| `bf1f4a68e900a277c8ce6157edfd08cf18c11d70` | `origin/feat/e2e-env-bootstrap` | **基础设施（分叉 B-2）** | 在 `bootstrap.sh --with-rust` 与 `checkEnv.mjs` 中增加 Rust + Tauri Linux C/WebKit 依赖检查与安装 | **与分叉 A 冲突** |

> 注：Bug 1（`476b783`）、Bug 2（`5fbc87d`）、Bug 5（`0bc5c69`）的早期修复其实早在 Wave 0（`2565774`）之前就已经合入 `multiplayer-recheck`，但随后 Wave 0 的架构重构（`SimulationDrivePolicy`、`MatchRosterProjection`、`RuntimeResourceRegistry`）改变了调用链，导致部分修复在 E2E 中再次失效或未完全接通。

---

## 三、两个基础设施分叉回答（自动安装 vs 建议？tmpfs 软链？）

针对你记忆中的几个问题，逐一核对代码后的事实如下：

1. **是“给建议”还是“直接帮你装好”？—— 两者都有（诊断 + 一键修复分离）**：
   - `doctor.mjs`（分叉 A）和 `checkEnv.mjs`（分叉 B）属于 **只读诊断（Problem Detector / Doctor）**：检测 Node、pnpm、`node_modules`、`public/` submodule、Chrome + SwiftShader WebGL2、`.rcreplay` fixture、Rust/cc，缺失时打印修复命令建议（如 `pnpm run env:setup`）。
   - `setup.sh`（分叉 A）和 `bootstrap.sh`（分叉 B）属于 **自动安装/修复（Setup / Bootstrap Helper）**：真正执行 `npm install -g pnpm`、`git submodule update --init --recursive`、`pnpm install`、生成 `.rcreplay`、安装 Chrome/Chromium，以及在传入 `--with-rust` 时自动安装 `rustup`（分叉 B 还会顺带 `apt-get install` Tauri 所需的 `gcc`、`libwebkit2gtk-4.1-dev` 等 C 依赖）。
   - **定位**：它们是 **环境诊断与一键修复脚本（Environment Doctor & Bootstrap Helper）**，不是运行时自愈的 Problem Solver。
2. **关于你提到的 `tmpfs` 软链（替代 overlayfs 降消耗）与 `tmpfs` 重置检测 + `restore` 建议**：
   - **目前两个分叉里都没有实现这一部分**（`node_modules`、`target`、`.cargo` 均未做 `/tmp` tmpfs 软链，也没有 tmpfs 重置检测）。
   - 如果需要该能力，建议在统一后的 `scripts/env/` 中新增可选参数（例如 `--tmpfs-link`），检测 `/tmp` 挂载与符号链接是否因容器快照恢复而失效，若失效则输出一步到位的 `bash scripts/env/setup.sh --restore` 命令。

---

## 四、当前真实 E2E 复测结果（`runDevBroadcastReproV3.mjs`）

我们在 `z/agy/rc/multiplayer-recheck` 上合入 `4ab7525`（W2-A）后，执行结果如下：
- `pnpm run check`（Radar 1429/1429 TS + `tsc --noEmit`）：**PASS**
- `pnpm run test:multiplayer-suite`（26 个单元测试）：**PASS (26/26)**
- `NO_PROXY=127.0.0.1,localhost node scripts/runDevBroadcastReproV3.mjs`（真实浏览器 E2E）：**`reproduced=4/5`**

| Bug 编号 | E2E 状态 | 实测现象与关键指标 | 初步定位 / 待决策点 |
| :--- | :--- | :--- | :--- |
| **Bug 1**：Client 进入 Goal Replay 无动力学 | **REPRODUCED** | `enteredReplay: true`, `distinctBallPositions: 1`, `bx:0, by:0, bz:93.2`, `bufFrames: 1` | Client 仍停留在初始/重置帧（`bufFrames: 1`），`HostBroadcastPublisher` 在 `host-goal-replay` 下向 Client 的流式推送与 `client-goal-replay` 消费链路未打通。 |
| **Bug 2**：Replay Watch Together 3D NameTag 缺失 | **REPRODUCED**（随相机视角偶发 PASS） | Run 1 中 Host 有 `carIndex: 1,3,4,5`，Client 只有 `1,2`（缺失 Orange 队 3,4,5）；Run 2 中 Host 视角内 `tags: []` 导致校验跳过而误报 PASS | 需确认两点：① `VehicleNameTagSystem` 是否在 Client Replay 时只渲染了部分队伍或被视锥裁剪影响；② E2E 断言本身受相机朝向影响存在假阴性。 |
| **Bug 3**：Client Spectate/Replay 卡顿与无物理前瞻 | **REPRODUCED** | 即使合入 W2-A，`distinctTicks: 1`（tick 停在 `12595` 不动），`frozenBallFrames: 23`, `maxBallStepUu: 1924.7` | **关键脱节**：`SimulationStage.ts` 在 `policy.shouldHoldAccumulator` 分支中先调用了 `interpolator.sync(timestamp)`；且在 Replay Watch Together 模式下 `applySpectateSampleOrLookahead` 未能持续推进客户端可见 tick/插值，W2-A 与 W2-B 之间仍缺少端到端衔接验证。 |
| **Bug 4**：BoostPad 模型每帧剧烈闪烁 | **基本修复（偶发边界）** | Run 1：`totalPadFlips: 4, maxSinglePadFlips: 1`（PASS）；Run 2：`totalPadFlips: 8, maxSinglePadFlips: 2`（边界触发） | `1114df1` 已大幅缓解，但阈值 `maxSinglePadFlips >= 2` 在高抖动帧下仍可能偶发触发一次。 |
| **Bug 5**：高内存与退出未释放 | **部分修复，几何体仍有泄漏** | `wasmMB: 64`（已从 259MB 大降），`retainedWasmMB: 0`，`tauriCloseHidesWindow: false`（已修复）；但 `retainedGeometries: 11~12`（`39 -> 70 -> 50`），`retainedJsMB: 62` | WASM 与 Tauri 关闭行为已达标，仅剩退出 Replay 返回 Freeplay 时仍有 **11~12 个 Three.js Geometry 未被 `RuntimeResourceRegistry` / `dispose` 释放**。 |

---

## 五、需要你拍板的下一步方案

目前我尚未向远端推送任何修改（仅在沙箱本地验证）。建议按以下顺序推进，请确认你的决策：

1. **是否先合入 `4ab7525`（W2-A）到 `z/agy/rc/multiplayer-recheck`？**
   - 推荐：**是**。它与 `4ec70dc`（W2-B）是同一波次的上下游，合并零冲突，单测全过。
2. **两个基础设施分叉如何统一？**
   - 推荐 **以 `env-doctor-repro`（`69d052b`）为主体，吸收 `feat/e2e-env-bootstrap`（`bf1f4a6`）的两个核心优点**：
     - 保留 `69d052b` 的真实 6-Ducky 3v3 WASM `.rcreplay` 生成器 + `ENOTEMPTY` 安全清理；
     - 合并 `bf1f4a6` 的 **Headless Chrome 代理绕过（`NO_PROXY` + `--proxy-server=direct://`）** 与 **Tauri C 依赖检查**；
     - （可选）是否要我顺便加上 **`/tmp` tmpfs 软链 + 重置检测与 `restore` 提示**？
3. **剩余 4 个 E2E 问题的修复优先级**：
   - 建议先集中修复 **Bug 1（Goal Replay 客户端无动力学）**、**Bug 3（W2-A + W2-B 真正跑通 E2E，消除 `distinctTicks: 1` 与 20+ 帧冻结）** 和 **Bug 5（释放退出 Replay 后残留的 11~12 个 Three.js Geometry）**，最后校验 **Bug 2（NameTag 同步与 E2E 稳定断言）** 与 **Bug 4 边界**，直到 `runDevBroadcastReproV3.mjs` 达到 `reproduced=0/5`。
