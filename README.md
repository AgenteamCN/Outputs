# RocketCompanion Wave 0 前置 Merge、架构问题确定性检测与防退化方案

## 1. 两个 Commit (`476b783` / `5fbc87d`) 是否需要先 Merge 再修架构？

**结论：必须先 Merge，且已完成合入并验证通过。**

- **合入前核查**：
  - `476b78307f295e0345567d1a94a282852716a5b0`（`origin/fix/bug-1-goal-replay-client-dynamics`）修改 `HostBroadcastPublisher.ts`、`SimulationStage.ts`、`simulationNetworkReconcilePipeline.ts`。
  - `5fbc87d2ae21f8fee3699ae5ea11be2fba398d8c`（`origin/fix/bug-2-replay-nametag-team-sync`，含父提交 `371043e`）修改 `ArenaVehicleFrameStepper.ts`、`devSyncChannelHandlers.ts`、`onlineSessionEnvironmentPipeline.ts`。
  - 二者此前均**未合入** `origin/z/agy/rc/multiplayer-recheck`，且均直接从 `943b749` 分出、修改文件完全正交。
- **为什么必须先 Merge 再做 Wave 0**：
  1. Wave 0 的 Arch-0A 与 Arch-0B 会直接重构上述 5 个核心文件；若不先合入，后续再合并这两个分支必然发生大面积语义冲突，甚至丢失 Bug 1 / Bug 2 的已验证修复。
  2. 先合入后，Bug 1 / Bug 2 成为 Wave 0 的**行为基线**，重构后可直接用 `repro:dev-broadcast-v3` 验证零回归。
- **已执行操作（Verified Bot Commits）**：
  - Fast-forward `origin/z/agy/rc/multiplayer-recheck` 至 `476b78307f295e0345567d1a94a282852716a5b0`。
  - 通过 GitHub Merges API 将 `5fbc87d2ae21f8fee3699ae5ea11be2fba398d8c` 合入 `z/agy/rc/multiplayer-recheck`（Merge Commit: `124172bc9a39fa3b52efa80a74c058eace1dbe0d`，`verified: true`）。

---

## 2. 本次架构问题能否用 `radar` 确定性检测？（“死的” vs “活的”）

总体判断：**架构退化一半是“死的”（边界与调用点可确定性卡死），一半是“活的”（语义拆分与运行时状态必须靠审计/E2E）。**

| 维度 | 能否确定性检测 | 原因与边界 |
| :--- | :--- | :--- |
| **Arch-0B：SSOT 散落调用**（`setNameTagRuntimeState`） | **✅ 100% 可确定性检测** | 哪个文件调用了特定端口函数是纯语法/引用事实。可通过 **棘轮上限（Ratchet）** 锁死当前数量（15 个引擎文件），Wave 0 完成后降为 `1`（仅允许投影器调用）。 |
| **Arch-0A：多布尔标志组合爆炸** | **🟡 部分确定性（度量+棘轮）** | 静态扫描可确定性统计条件分支中的 `is*/should*/has*` 旗标数量与复合 `if` 次数（精准命中 `simulationNetworkReconcilePipeline.ts` 12 flags / 10 compoundIfs 与 `SimulationStage.ts` 10 flags / 3 compoundIfs）。但“哪些布尔属于互斥模式”在重构前属于语义判断；**重构后**可确定性禁止外部直接分支这些旧布尔。 |
| **Arch-0C：资源生命周期与缓存** | **🟡 部分可审计，泄漏需运行时** | 模块级 `Map`/`Set` 缺少 `dispose`/`clear`/`reset` 可静态审计（已命中 `SafeTensorsLoader.ts` 等 3 个文件）；但是否真正释放（如 `visible = false` 未 `removeCar`、WASM 内存回落）必须靠 CDP 内存断言（`repro:dev-broadcast-v3`）。 |
| **Arch-0D：扁平 `Float32Array` 离散/连续耦合** | **❌ 静态不可检测** | 在类型系统里 `Float32Array[i]` 全部是 `number`，无法静态区分第 `i` 个槽位是物理坐标还是离散 `isActive` 布尔，必须靠重构后的显式离散偏移表契约或运行时视觉/状态校验。 |
| **假拆分 / 硬拆（绕过 800L 上限）** | **🟡 已知模式可抓，语义拆分是猫抓耗子** | 仓库已有 `radarStructuralCoupling.ts` 拦截 `HostPen`、`PassThru`、`new Sub(this)`、`onCb` 乒乓与任意编号后缀。但如 `5fbc87d` 为避开行数上限把函数挪到 `devSyncChannelHandlers.ts`（靠传参而非 `this.host`），正则无法判定其属于“合理职责拆分”还是“凑行数拆分”。 |

---

## 3. 已落地的工具链与 `radar` 确定性增强

1. **工具链补齐**：已安装 `pnpm@12.21.0` 并完成依赖安装（`pnpm install --frozen-lockfile`）。
2. **新增 `scripts/radar/radarArchDebtGuards.ts` 并接入 `scripts/radar.ts`**（Verified Bot Commit: `746126d7931c4ab29c50876e9efbd2c440046361`，`signature.isValid: true`）：
   - **SSOT 散落棘轮（Hard Error）**：`HARD_ENGINE_NAMETAG_SCATTER_LIMIT = 15`，任何新增引擎文件直接调用 `setNameTagRuntimeState(...)` 将直接阻断 `pnpm run radar`（Wave 0 Arch-0B 完成后下调至 `1`）。
   - **布尔组合热点门禁与审计**：`HARD_BOOLEAN_FLAG_CLUSTER_LIMIT = 15`、`HARD_COMPOUND_FLAG_IF_LIMIT = 12`，并在 `pnpm run radar:audit` 输出 `[B]` 热点清单（精准标出 `simulationNetworkReconcilePipeline.ts` 与 `SimulationStage.ts`）。
   - **无卸载钩子单例缓存审计**：在 `pnpm run radar:audit` 输出 `[C]` 清单，扫描 `src/ai/`、`src/entities/`、`src/replay/` 下缺少 `dispose`/`clear`/`reset` 的模块级 `Map`/`Set`。
3. **验证结果**：
   - `pnpm run check`：**GREEN**（1425/1425 TS，`tsc --noEmit` 0 错误，0 循环依赖）。
   - `pnpm run test:multiplayer-suite`：**24/24 PASS**。

---

## 4. 无法确定性检测的部分，何时扫描、如何防治？

建议采用**“平时靠棘轮锁死增量，节点靠审计清理存量”**的四层防线：

1. **每次 PR / 合入前（自动化硬门禁）**：
   - 必跑 `pnpm run check`（`radar` + `tsc --noEmit`）+ `pnpm run test:multiplayer-suite`。
   - 对已知边界使用**只降不升的棘轮（Ratchet）**：每完成一个重构（如 Arch-0B），立刻把 `radarArchDebtGuards.ts` 中的上限从 `15` 拧紧到 `1`，把语义成果固化为确定性规则。
2. **每个 Wave / 里程碑合入前 & 每周定期（半自动语义扫描）**：
   - 运行 `pnpm run radar:audit`，人工/AI 重点复核两类高危变更：
     - **行数接近 700–800L 时新增的跨文件拆分**（检查是否为“传参搬运式假拆分”）；
     - **新增 Mode / Activity 入口**（检查是否绕过 `SimulationDrivePolicy`、`MatchRosterProjection`、`RuntimeResourceRegistry`）。
3. **跨模式与性能/内存回归门禁（动态验证）**：
   - 涉及模拟、回放、多人同步或资源生命周期的 PR，合入前跑 `pnpm run repro:dev-broadcast-v3`（CDP 内存/闪烁/同步断言），弥补静态分析无法覆盖的 Arch-0C / Arch-0D 运行时问题。
4. **Code Review 两条红线（低成本人工/AI 核对）**：
   - 凡是 commit message 出现“keep under size cap / split helper”但功能无新域抽象的，要求说明领域边界；
   - 凡是新增跨状态布尔开关的，优先归入 `SimulationDriveMode` 策略表而非在 `if` 中叠加条件。
