# RocketCompanion `d69e01c` 报告解读与测试真实性核查

> 核查分支：`z/agy/rc/multiplayer-recheck`（最新提交 `d69e01c`）
> 环境验证：已在沙箱中安装 `pnpm`（`12.21.0`）并执行 `pnpm install --frozen-lockfile`，复现运行 `pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite`，全部通过且退出码为 `0`。

---

## 一、为什么有这么多 `[!]`、`[B]`、`[S]`、`[C]` 和 `PingPong`，最终还是 `STATUS: GREEN`？

`pnpm run radar`（`scripts/radar.ts`）是一个**静态架构边界与代码规范检查器**（非运行时单测），它把检查项分为**硬阻断（Hard Errors，触发即 `STATUS: FAILED` 并 `exit 1`）**和**审计债务提示（Audit / Ratchet Notices，用于追踪待重构热点，不阻断构建）**两类：

1. **为什么是 `STATUS: GREEN`？**
   - `scripts/radar.ts` 在 `errors.length === 0 && hardCeilingViolations.length === 0` 时输出 `STATUS: GREEN`。
   - 当前所有硬性指标均通过：`1425/1425` 全量 TypeScript、`tsc --noEmit` 零报错、`0` 循环依赖、`0` 值重导出（Value Re-Exports）、`0` 引擎向上引用 UI、无文件超过硬体积上限（`> 800L` 或 `> 18KB`），且各项棘轮（Ratchet）均未超上限。

2. **各个符号与指标的真实含义（仅在加 `--audit` 即 `pnpm run radar:audit` 时打印）：**
   - **`[!]` — `STRUCTURAL COUPLING & PING-PONG AUDIT`（9 个目标）**
     - 源码：`scripts/radar/radarStructuralCoupling.ts`。
     - 触发条件：单文件达到软审计阈值（如 `HostPen >= 12`、`PassThru >= 5`、`newThis >= 2`、`onCb >= 6` 或耦合目标 `>= 2`），仅作重构提示（并给出 `Rx:` 修复处方）。
     - 不报错原因：硬性阻断上限是 `HostPen <= 105`、`PassThru <= 40`，这 9 个文件远低于硬上限。
   - **关于“Pingpong Audio”（实际是 `PingPong` 耦合审计，与音频无关）**
     - 报告中的 `PingPong=(newThis:X,onCb:Y)` 指的是**双向回调/引用乒乓耦合**：即 `new Sub(this)` 传 `this` 指针的数量（`newThis`）以及 `onXxx:` 回调属性包的数量（`onCb`），用来发现宿主与子控制器之间相互回调的问题，**并不是 Audio 音频模块报错**。
   - **`[B]` — `BoolHotspots`（9 个文件）**
     - 源码：`scripts/radar/radarArchDebtGuards.ts`。
     - 触发条件：`src/game/`、`src/network/`、`src/replay/` 下，条件判断中出现 `>= 8` 个不同的 `is/should/has...` 布尔标志且有 `>= 3` 个复合 `if` 的文件（建议重构为显式状态机策略 `EXPLICIT_STATE_MACHINE_POLICY`）。
     - 不报错原因：硬性棘轮上限为 `DistinctFlags <= 15` 且 `CompoundIfs <= 12`，当前 9 个文件均未越线。
   - **`[S]` — `NameTagScatter=15/15`（15 个文件）**
     - 源码：`scripts/radar/radarArchDebtGuards.ts`。
     - 含义：有 15 个引擎文件直接调用了 `setNameTagRuntimeState(...)`（违反单一数据源 SSOT 理想状态）。
     - 关键点：`HARD_ENGINE_NAMETAG_SCATTER_LIMIT = 15` 是**棘轮上限**。当前正好卡在 `15/15` 通过；只要再多出第 16 个文件直接调用，就会触发 `[SSOT SCATTER VIOLATION]` 并导致构建失败。
   - **`[C]` — `UnscopedCaches`（3 个文件）**
     - 源码：`scripts/radar/radarArchDebtGuards.ts`。
     - 含义：`src/ai/`、`src/entities/`、`src/replay/` 中存在模块级 `new Map()` / `new Set()` 但缺少 `dispose`/`clear*`/`reset*`/`reclaim*` 清理钩子的文件。目前仅作审计记录，不阻断。
   - **`INFO (Non-blocking): 481 existing file(s) in 12KB..18KB / 400..800L`**
     - 超过软阈值（`400L / 12KB`）但未超硬上限（`800L / 18KB`）的存量文件数量，非阻断。

---

## 二、这些测试是“假的”吗？无论有没有问题都会输出 Green 吗？

**不是假的，也没有“无论怎样都输出 Green”的问题。** 经逐一核查源码：

1. **`scripts/radar.ts`（架构雷达）是真检查：**
   - 一旦出现 `tsc --noEmit` 编译错误、循环依赖、孤儿模块、越界引用、`any` 超标、超 `800L/18KB` 硬上限或棘轮超标，就会写入 `errors` / `hardCeilingViolations`，打印 `STATUS: FAILED` 并执行 `process.exit(1)`。
2. **`scripts/runMultiplayerSuiteTests.ts`（24 项多人联机套件测试）是真测试：**
   - 该文件共 3,281 行，覆盖 24 个联机场景（Tunnel 传输、丢包重传、语音路由、GhostBallRelay、Telestrator 50 笔压测与 HashAck、6 车回环模拟、Pure CSR 同步、Replay 同步、权威移交等）。
   - 全文件包含 **382 处 `assert.*`（`node:assert/strict`）严格断言**。
   - `runSection(...)` 没有任何吞掉异常的 `try/catch`，最终通过 `main().catch((err) => { console.error('[FAIL]', err); process.exit(1); })` 处理失败——只要任意一条断言失败就会直接报错并以退出码 `1` 中止。

---

## 三、结论

- 工具链（`pnpm` + 依赖）已在沙箱中完整安装并实测通过。
- `radar:audit` 中的 `[!]`、`[B]`、`[S]`、`[C]` 属于**已知技术债的审计提醒**（未突破硬上限/棘轮），`PingPong` 是指对象双向回调耦合而非音频问题。
- 24 项多人联机测试均为真实断言测试，**不存在“假测试/永远输出 Green”的缺陷**，因此无需针对测试本身做修复或推送。
