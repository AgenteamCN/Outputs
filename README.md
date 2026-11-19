# Wave 0 Octopus Merge 验证报告与后续最高效并行执行计划

## 一、Wave 0 Octopus Merge 执行与验证结果

### 1. Octopus Merge 结果
- **目标分支**：`origin/z/agy/rc/multiplayer-recheck`
- **合并前基线**：`d69e01c`
- **合并的 4 条 Wave 0 分支**：
  - `origin/z/agy/rc/wave0-arch-0a` (`3deaedf`) — `SimulationDrivePolicy` 显式状态机
  - `origin/z/agy/rc/wave0-arch-0b` (`479f742`) — `MatchRosterProjection` SSOT
  - `origin/z/agy/rc/wave0-arch-0c` (`fa35923`) — `RuntimeResourceRegistry` 生命周期总线与延迟/作用域回收
  - `origin/z/agy/rc/wave0-arch-0d` (`ed7d5df`) — `BoostPad` 离散字段隔离与 `frameCursor` / `isDuplicateFrame` 输出
- **合并方式**：4 路 Octopus Merge（GitHub REST API `git/trees` + `git/commits` + `git/refs`，省略 author/committer 自动签名）
- **合并结果**：**无冲突自动合并成功**（`src/game/modes/FreeplayMode.ts` 为 `0b` 与 `0c` 唯一共同修改文件，Git 自动三方合并通过）。
- **已推送 Verified Bot Commit**：
  - Commit SHA：`256577470a3188242e72362d9db2dbceaa5f548b`
  - Verification：`verified: true` (`reason: "valid"`, author: `antigravity-fine-grained-token[bot]`)
  - Parents：`[d69e01c, 3deaedf, 479f742, fa35923, ed7d5df]`

### 2. 工具链与门禁验证（在 `2565774` 上实测）
- **工具链**：容器内已安装 `pnpm@12.21.0` 并完成 `pnpm install --no-frozen-lockfile --ignore-scripts`（2.8s）。
- **`pnpm run check`**：**PASS (GREEN)**
  - `1428/1428` TS 文件，0 Engine JS
  - `tsc --noEmit`  clean
  - `NameTagScatter=1/1`（已从 15 收敛至仅 `MatchRosterProjection.ts`）
  - `BoolHotspots=7`（`SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts` 已移出热点）
  - `UnscopedCaches=0`（已从 3 降至 0）
  - `0 Circular Deps | 0 Value Re-Exports | 0 Upward Imports | Any<=5`
- **`pnpm run radar:audit`**：**PASS (GREEN)**
- **`pnpm run test:multiplayer-suite`**：**PASS (24/24 全部通过)**
- **环境限制项（需目标 Runner）**：
  - `pnpm run repro:dev-broadcast-v2`：当前容器无 GPU/WebGL 渲染环境，Headless Chrome 无法完成渲染启动（退出清理报 `ENOTEMPTY`）。需在具备 WebGL 渲染能力的 Runner 上执行 E2E。
  - `cargo` / `rustc`：当前容器未安装 Rust 工具链，C4（`src-tauri/src/lib.rs`）需在具备 Rust 的 CI/本地环境执行 `cargo check` / `pnpm run tauri:build`。

---

## 二、后续计划优化：Wave 1 / Wave 2 / Wave 3 合并与最大并行化方案

### 1. 关键发现：Wave 1 已无代码阻塞，不必单线卡住 Wave 2 / Wave 3
根据当前代码库状态（`2565774`）：
- **Bug 1（Goal Replay 客户端动态）** 已由 `476b783` 实现并已保留在 Arch-0A 中；
- **Bug 2（NameTag 与队伍同步）** 已由 `5fbc87d` + Arch-0B（`NameTagScatter=1/1`）彻底收敛；
- **Bug 5 C1/C2/C3（内存释放）** 已由 `0bc5c69` + Arch-0C（`UnscopedCaches=0`）接入总线；
- **Bug 5 C4（Tauri 关闭语义）** 已由 `0bc5c69` 修改 `src-tauri/src/lib.rs`。

因此，**Wave 1 剩余工作全部是“验收测试”，而不是“代码开发”**，不需要作为前置阻塞阶段单线排队。可以将 Wave 1 的 WebGL E2E 验收直接合并到最终的 `repro:dev-broadcast-v3` 全量回归中，Rust `cargo check` 独立异步执行。

### 2. Wave 2 与 Wave 3 可拆分为 3 路完全无冲突并行开发
检查 `2565774` 代码边界后，Wave 2（Bug 3）与 Wave 3（Bug 4）的核心文件完全正交，且 Arch-0D 已经在 `ReplicationStateInterpolator.ts` 输出了 `frameCursor` 与 `isDuplicateFrame` 契约，**不需要等 Wave 2 写完再写 Wave 3 代码**：

| 并行轨道 | 目标 | 核心修改文件（彼此无交集） | 依赖 |
| :--- | :--- | :--- | :--- |
| **Track W2-A** | Bug 3（2-A）：`SpectateInterpolationBuffer` >= 5 帧前瞻窗口与游标状态机 | `src/network/sync/ReplicationStateInterpolator.ts` | 无（基于 `2565774`） |
| **Track W2-B** | Bug 3（2-B）：`client-csr-spectate` 开启 `shouldRunLookahead = true` 与本地 WASM 物理前瞻收敛 | `src/game/pipeline/stages/simulationFrameClassifierPipeline.ts`<br>`src/game/pipeline/stages/SimulationStage.ts`<br>`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` | 仅依赖 `SpectateInterpolationBuffer` 现有接口 |
| **Track W3** | Bug 4（3-A/3-B）：利用 Arch-0D 的 `isDuplicateFrame` 与离散 `isActive` 阶跃输出，增加重复帧跳过与冷却微抖阈值保护 | `src/entities/BoostPadSystem.ts`（及 BoostPad 快照同步调用点） | 无（Arch-0D 契约已合入 `2565774`） |
| **Track V1-Rust**（异步） | Bug 5（C4）：Rust 窗口关闭逻辑编译验证 | 无代码改动（`src-tauri/src/lib.rs`） | 仅需含 `cargo`/`rustc` 环境 |

### 3. 最高效执行拓扑（2 步收官）
1. **第 1 步（3 路并行开发 + 异步 Rust 检查）**：
   - 同时拉出 `wave2-lookahead-buffer`（W2-A）、`wave2-lookahead-driver`（W2-B）、`wave3-boostpad-debounce`（W3）三条分支开发，每条分支各自过 `pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite`。
2. **第 2 步（Octopus Merge + 一次全量收官验收）**：
   - 将 W2-A、W2-B、W3 **一次 Octopus Merge** 合入 `z/agy/rc/multiplayer-recheck`。
   - 在具备 **WebGL 渲染能力** 的 Runner 上一次性跑完最终收官门禁：
     ```bash
     pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite && pnpm run repro:dev-broadcast-v3
     ```
   - 一次性同时验收 **Bug 1、Bug 2、Bug 3、Bug 4、Bug 5 (C1–C3)**（目标 `reproducedCount === 0/5`），无需把 Wave 1 和 Wave 2/3 分成两次串行 E2E。
