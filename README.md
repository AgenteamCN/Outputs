# RocketCompanion 报告解读与测试真实性核查（分支 `z/agy/rc/multiplayer-recheck` @ `d69e01c`）

我已在沙盒中安装 `pnpm`（`12.21.0`）、完成依赖安装并完整复现了 `pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite`，同时逐行核查了 `scripts/radar.ts`、`scripts/radar/radarStructuralCoupling.ts`、`scripts/radar/radarArchDebtGuards.ts` 以及 `scripts/runMultiplayerSuiteTests.ts`。

结论先行：**这些检查和测试都不是假的，不是“无论有没有问题都会输出 green”；它们确实会真实拦截错误并退出非零状态码，因此无需针对“假测试”进行修复和推送。**

---

## 一、为什么这么多 `[!]`、`[B]`、`[S]`、`[C]`，最后却还是 `STATUS: GREEN`？

`scripts/radar.ts`（架构雷达）把检查分成**两层**：

1. **硬性阻断红线（Hard Ceilings / Ratchets）**：只要触发任意一条，就会打印 `STATUS: FAILED` 并以 `process.exit(1)` 失败退出。包括：
   - 0 个引擎 JS 文件（1425/1425 全 TS）
   - `tsc --noEmit` 零报错
   - 单文件硬上限：`<= 800 行` 且 `<= 18 KB`，单行 `<= 400 chars`
   - 0 循环依赖、0 值重导出（Value Re-Exports）、0 引擎层反向导入 UI、0 孤儿模块、每文件 `any <= 5`
   - 硬性耦合与债务棘轮（Ratchet）上限：`HostPen <= 105`、`PassThru <= 40`、`NameTagScatter <= 15`、`DistinctFlags <= 15`、`CompoundIfs <= 12`
2. **非阻塞技术债审计（`--audit` 模式才打印的软阈值提醒）**：这些 `[!]`、`[B]`、`[S]`、`[C]` 都是**低于硬性阻断上限**、但超过了“重构关注软阈值”的历史技术债清单，用来提示后续重构方向，因此**不会导致失败**：

### 1. `[!]` 结构耦合与 Ping-Pong 审计（`STRUCTURAL COUPLING & PING-PONG AUDIT`）
你提到的“Pingpong Audio”实际是语音识别把 **`PingPong`** / **`Audit`** 识别成了 Audio。它出自 `scripts/radar/radarStructuralCoupling.ts`，用来检测“伪拆分（Anti-Fake-Split）”和双向耦合，当引擎文件触发以下任一软阈值时就会标 `[!]`：
- `HostPen=10(10R/0W)`（软阈值 `>= 12`，硬上限 `<= 105`）：访问注入的 `host`/`owner`/`parent`/`ctrl`/`runtime`/`engine`/`mode` 属性的次数（读次数 R / 写次数 W）。
- `PassThru=4`（软阈值 `>= 5`，硬上限 `<= 40`）：单行透传代理方法数量。
- `PingPong=(newThis:0,onCb:6)`（软阈值 `newThis >= 2` 或 `onCb >= 6`）：双向乒乓耦合指标——`new Sub(this)` 的次数，以及 `onXxx: ...` 回调包的个数。
- `Rx: ...`：雷达自动给出的重构处方（Prescription），例如 `CALLBACK_TO_READONLY_SNAPSHOT`（把回调改成只读快照）、`OWN_STATE_SUBCONTROLLER`、`PURE_PIPELINE`。

### 2. `[B]` / `[S]` / `[C]` 架构债务审计（`ARCH DEBT AUDIT`）
出自 `scripts/radar/radarArchDebtGuards.ts`：
- **`[B]`（Boolean Flag Hotspots，9 个）**：单个文件在 `if (...)` 中组合了过多布尔模式标志（软阈值 `DistinctFlags >= 8 && CompoundIfs >= 3`，硬上限 `15` / `12`）。处方 `Rx: EXPLICIT_STATE_MACHINE_POLICY`（改用显式状态机策略）。
- **`[S]`（NameTag Scatter，15/15）**：有 15 个引擎文件直接调用了 `setNameTagRuntimeState(...)`。**注意：`15/15` 已经顶到硬棘轮上限**，只要再多 1 个文件直接调用，`radar` 就会立刻变成 `STATUS: FAILED`。处方 `Rx: ROSTER_PROJECTION_SSOT`。
- **`[C]`（Unscoped Caches，3 个）**：`src/ai/`、`src/entities/`、`src/replay/` 中存在模块级 `new Map()` / `new Set()`，但同一文件里没有 `dispose` / `clear*` / `reset*` / `reclaim*` 清理钩子。处方 `Rx: DISPOSABLE_RESOURCE_REGISTRY`。

### 3. `INFO (Non-blocking) : 481 existing file(s) in 12KB..18KB / 400..800L range`
有 481 个既有文件处于软阈值（`12KB..18KB` 或 `400..800L`）区间，但没有超过硬上限（`18KB / 800L`），因此雷达判定 `ACCEPTABLE`。

---

## 二、`test:multiplayer-suite`（24 项全 PASS）是不是假测试？

**不是假测试，但也需要看清它的测试边界：**

1. **真实性核查（`scripts/runMultiplayerSuiteTests.ts`，3,281 行）**：
   - 全文件共有 **382 处 `assert.*` 严格断言**（`node:assert/strict`）。
   - **没有任何吞掉异常的 `try { ... } catch` 块**；`main()` 末尾挂载了 `.catch((err) => { console.error('[FAIL]', err); process.exit(1); })`。
   - 24 个测试全部直接导入并执行真实业务 TS 模块（如 `DataChannelTransport`、`ReliableTunnelFramer`、`RoomActivityOrchestrator`、`PredictionReconciler`、`TelestratorWireCodec`、`MultiClientLoopbackSimulator`、`CoachAuthorityHandoverController`、`SpectateInterpolationBuffer`、`bindDevSyncClientChannelHandlers` 等），真实校验了丢包重传、分片乱序重组、Epoch 栅栏、权限控制、状态机切换与编码解码一致性。
2. **客观局限性（Node.js 环境边界）**：
   - 由于该套件通过 `node --experimental-strip-types` 在 Node.js 中运行，没有真实浏览器、WebRTC 原生连接、Tauri UDP 隧道和 WASM 物理引擎，因此测试中对底层传输/运行时边界使用了内存桩（例如 `fakeDc`、`loopbackTunnel`、`mockRuntime`、`getAudioContext: () => null`）。
   - 也就是说，它**真实可靠地验证了协议编解码、状态机、控制面与同步逻辑**；而涉及真实浏览器标签页与渲染/物理的端到端链路，仓库里另配有 `pnpm run test:dev-room-sync-e2e`、`pnpm run test:baseline`、`pnpm run test:soak` 等脚本负责。

---

## 三、本地复现结果

在 `/workspace/RocketCompanion`（分支 `z/agy/rc/multiplayer-recheck`，commit `d69e01c`）实测：
- `pnpm run check`：通过（1425/1425 TS，`tsc --noEmit` 通过，硬性架构边界与体积上限通过）
- `pnpm run radar:audit`：通过（输出 9 个 `[!]`、9 个 `[B]`、15 个 `[S]`、3 个 `[C]` 审计项，最终 `STATUS: GREEN`）
- `pnpm run test:multiplayer-suite`：通过（24/24 项全部 `[PASS]`，`=== ALL MULTIPLAYER SUITE TESTS PASSED ===`）
