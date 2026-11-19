# 架构前置研判、Commit 合并与 Radar 确定性检测报告

## 1. 结论先行：`476b783` 与 `5fbc87d` 必须先 Merge，且已完成合入

### 1.1 核查结果
在本次操作前，两个目标 commit 均**未合入** `origin/z/agy/rc/multiplayer-recheck`：
- `476b78307f295e0345567d1a94a282852716a5b0`（分支 `origin/fix/bug-1-goal-replay-client-dynamics`）：修复 Bug 1（Goal Replay 客户端动态），修改 `HostBroadcastPublisher.ts`、`SimulationStage.ts`、`simulationNetworkReconcilePipeline.ts`。
- `5fbc87d2ae21f8fee3699ae5ea11be2fba398d8c`（分支 `origin/fix/bug-2-replay-nametag-team-sync`，含前置提交 `371043e`）：修复 Bug 2（Replay Watch Together NameTag 与队伍同步），修改 `devSyncChannelHandlers.ts`、`onlineSessionEnvironmentPipeline.ts`。

### 1.2 为什么必须在 Wave 0 架构修复前先 Merge
1. **零冲突且正交**：两分支均直接从 `943b749`（原 `multiplayer-recheck` HEAD）分叉，修改文件完全不重叠。
2. **保留已验证修复与初步收敛成果**：`5fbc87d` 已经把 `onlineSessionEnvironmentPipeline.ts` 中的重复逻辑抽成 `syncReplayRosterTeamsAndNameTags` 与 `applyClientReplayActivitySync`。若不先合并而直接启动 Wave 0（`Arch-0A` 改写 `SimulationStage.ts`，`Arch-0B` 改写 `devSyncChannelHandlers.ts`），这两个分支会立刻产生严重冲突，导致 Bug 1 / Bug 2 修复需要人工二次搬运。

### 1.3 已执行动作与验证
- 已通过 GitHub API 将 `476b783` 与 `5fbc87d` 合入 `origin/z/agy/rc/multiplayer-recheck`（Verified Bot Merge Commit: `124172bc9a39fa3b52efa80a74c058eace1dbe0d`），当前远程分支最新提交为 `746126d7931c4ab29c50876e9efbd2c440046361`。
- 已在本地安装 `pnpm@12.21.0` 并完成 `pnpm install --frozen-lockfile`，验证通过：
  - `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1425/1425 TS，0 循环依赖，0 越级引用）
  - `pnpm run test:multiplayer-suite`：**24/24 PASS**

---

## 2. 架构问题是“死的”还是“活的”？Radar 能否 100% 确定性检测？

结论：**一半是“死的”（可确定性拦截或用棘轮锁死增量），一半是“活的”（纯语义问题，静态规则必然陷入“猫抓耗子”）。**

### 2.1 当前仓库 Radar 的真实能力边界
当前 `scripts/radar/`（`radarAstScanners.ts`、`radarStructuralCoupling.ts`、`radarArchDebtGuards.ts`）以及 `package.json` 中的 `"typescript": "7.0.2"`（native `tsgo`）：
- **确定性极强的部分**（不依赖变量命名猜测）：
  - 模块依赖图（0 循环依赖、0 向上越级导入、0 孤儿模块）
  - 文件体积与单行长度（`400L/12KB` 软上限、`800L/18KB` 硬上限）
  - 类型系统约束（`tsc --noEmit`、`any <= 5`）
  - **SSOT 唯一出口棘轮**（如 `radarArchDebtGuards.ts` 中的 `NameTagScatter=15/15`，锁死白名单，任何**新增文件**直接调用 `setNameTagRuntimeState(...)` 都会立即失败）
- **容易被绕过（“猫抓耗子”）的正则启发式部分**：
  - `radarStructuralCoupling.ts` 中的 `HOST_ACCESS_RE` 只匹配 `host|owner|parent|ctrl|runtime|engine|mode`；只要改名为 `ctx` / `world` / `deps` 就能绕过。
  - `PASSTHROUGH_METHOD_RE` 只匹配单行 `{ return this.a.b(...); }`；拆成两行 `const res = ...; return res;` 即可绕过。

### 2.2 本次 Wave 0 四个架构病灶的可检测性矩阵

| 架构任务 | 问题本质 | 能否确定性检测？ | 当前 Radar 现状与应对策略 |
| :--- | :--- | :--- | :--- |
| **Arch-0A**<br>模拟管线状态机 | 6+ 布尔标志位组合爆炸 | **部分可测**<br>（分支复杂度可测，业务互斥语义不可测） | `radarArchDebtGuards.ts` 已用 `BoolHotspots=9` 标出 `simulationNetworkReconcilePipeline.ts`（12 flags / 10 compound ifs）与 `SimulationStage.ts`（10 flags / 3 compound ifs）。落地 `SimulationDriveMode` 后，配合 TS `switch` + `assertNever` 即可由编译器确定性保证分支穷尽。 |
| **Arch-0B**<br>NameTag/Team SSOT | `setNameTagRuntimeState` 散落 15+ 文件 | **✅ 可 100% 确定性检测** | 谁允许调用特定导出函数是闭集问题。`radarArchDebtGuards.ts` 已建立 `NameTagScatter=15/15` 棘轮，阻断任何新增散落调用；Wave 0 每迁移一个就从白名单删一个，最终归零并锁定到 `MatchRosterProjection.ts`。 |
| **Arch-0C**<br>资源生命周期与延迟分配 | 启动期 eager 分配 WASM + 无 `dispose` 单例 | **半确定性**<br>（静态入口/钩子可测，真实泄漏需运行时测） | `radarArchDebtGuards.ts` 已标出 `UnscopedCaches=3`（`SafeTensorsLoader.ts`、`vehiclePaintMaterial.ts`、`ReplayDirectorCsvParser.ts` 无 `dispose/clear/reset` 钩子）；但 WASM 线性内存和 Three.js 闭包泄漏必须靠运行时 E2E（`repro:dev-broadcast-v3`）实测 `retainedWasmMB` 与 `retainedGeometries`。 |
| **Arch-0D**<br>离散态与连续插值耦合 | `Float32Array` 混装布尔态与物理量 | **❌ 静态不可测** | 在类型系统里 `Float32Array` 只是数字数组，静态扫描无法分辨哪个下标是离散布尔 `isActive`。必须先重构为独立类型视图（`DiscretePadState` vs 连续物理状态），再用 `tsc --noEmit` 确定性防止把离散态传入 `lerp()`。 |

---

## 3. 防治机制：无法 100% 静态拦截时，何时扫描与如何治理？

既然“假拆分（传参来回倒）”与“语义级耦合”属于**活的问题**，单靠正则加码只会变成猫抓耗子，建议采用**三层防线**：

1. **每次提交 / PR 自动门禁（阻断增量债务）**：
   - 运行 `pnpm run check`（`radar` + `tsc --noEmit`）。
   - 依靠 `radarArchDebtGuards.ts` 的**棘轮（Ratchet）机制**确保 `NameTagScatter`、`BoolHotspots`、`UnscopedCaches` 只减不增；利用文件体积硬上限和 0 循环依赖防止结构继续恶化。
2. **周期性深度审计（每周或每个 Milestone 合入前）**：
   - 运行 `pnpm run radar:audit`，集中审查三大清单：
     - `STRUCTURAL COUPLING & PING-PONG AUDIT`（当前 9 个热点，检查是否有通过重命名绕过 `HostPen` 的“假拆分”）
     - `BoolHotspots`（当前 9 个热点，检查是否有人新增布尔标志位而非扩展状态机枚举）
     - `UnscopedCaches`（当前 3 个模块级缓存，检查是否接入 `RuntimeResourceRegistry`）
3. **跨模块大改动后的运行时 E2E 兜底**：
   - 运行 `pnpm run test:multiplayer-suite`（纯 Node 24 项协议与状态矩阵）与 `pnpm run repro:dev-broadcast-v3`（Headless Chrome 真实渲染、插值、内存回收门禁）。
   - 注：Wave 1 的 C4（`src-tauri/src/lib.rs` 窗口关闭语义）需在具备 `cargo` / `rustc` 的本地或 CI 环境中验证（当前沙箱无 Rust 工具链）。
