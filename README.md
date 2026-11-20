# 棘轮锁紧提交记录与架构检测策略深度评估报告

## 一、已完成操作：棘轮锁紧与 Verified Bot Commit

已修改 `scripts/radar/radarArchDebtGuards.ts`，将 Batch C 已清零的 3 项架构债务棘轮上限全部锁紧至 `0`：
- `HARD_VOLUME_CEILING_FILES_LIMIT`: `12 -> 0`
- `HARD_EMPTY_CATCH_LIMIT`: `31 -> 0`
- `HARD_SILENT_PROMISE_CATCH_LIMIT`: `23 -> 0`

**验证结果**：
- `pnpm radar`：PASS（`Hard=800L/18KB (0/0 ratchet)`，`SilentCatches=0/0+0/0`）
- `pnpm check`：PASS（`tsc --noEmit`、`eslint src`、`vitest run` 全部通过）

**GitHub Verified Bot Commit**：
- Branch: `a2`
- Commit SHA: `b99fd53a42fafe7f39304b0b1640a460e3f2419d`
- Message: `chore(radar): lock cleared arch-debt ratchets to 0`
- Signature: `isValid: true`, `state: VALID`（`antigravity-fine-grained-token[bot]`，含 `Co-authored-by: Mark`）
- 本地已执行 `git fetch origin && git reset --hard origin/a2` 同步。

---

## 二、`HostPen` 与 `ReactHooks` 策略评估：保留（宁可误报，绝不漏报）

对于 `HostPen <= 55` 与 `ReactHooks <= 28` 的取舍，**明确建议：保持现状，不删除指标、不放宽阈值（采用“允许少量误报、绝不漏报”的保守策略）**。原因如下：

1. **为什么不能删掉 `HostPen`？**
   - `HostPen` 监控的是 `host` / `owner` / `parent` / `ctrl` / `runtime` / `engine` / `store` / `state` 的跨对象属性读写。在普通业务/服务模块中，它是防止“子模块反向穿透修改宿主内部状态”的唯一静态防线。
   - `taskMutationSlice.ts` 报出 `52(52R/0W)`，是因为 Zustand 的 `set((state) => ...)` 惯例使用了 `state` 作为形参名；它目前仍在 `<= 55` 的安全区内，**并未阻断构建**。如果为了消除这个 52 的数字而删掉 `HostPen`，后续一旦有人在 `services/` 或 `utils/` 里直接写 `ctrl.xxx = ...` 破坏封装，雷达就会完全失明（漏报）。
   - **未来若要优化**：只应升级为 AST 精确识别“排除 Zustand `set((state) => ...)` 的只读形参”，而绝不能删除 `HostPen` 检测本身。

2. **为什么不能删掉或放宽 `ReactHooks`？**
   - `useEmqxBridge.ts`（27 hooks）与 `useSafariWindowManager.ts`（26 hooks）**不是误报，而是 100% 真实的复杂度**。保留 `ReactHooks <= 28` 能够倒逼开发者在触碰红线前主动拆解副作用与状态，防止出现“60 个 Hook 纠缠在一个文件里导致闭包陈旧、重渲染失控”的经典 React 灾难。

---

## 三、深度解析：“核心编排型 Custom Hook 属于真实隐患”是什么意思？

### 1. 它该不该有？
**编排层（Orchestrator）本身应该存在，但不应该把所有子域的 `useState`、`useEffect`、`useRef` 全塞进同一个 Custom Hook 里。**
以 `useEmqxBridge.ts`（400L / 27 hooks）和 `useSafariWindowManager.ts`（523L / 26 hooks）为例，它们同时承担了：
- 底层连接/窗口轮询生命周期（定时器、重连、Ref 同步）
- 业务协议路由与状态机（待审批队列、网络录制状态、目标窗口过滤）
- 持久化读写与 UI 错误提示拼接

当一个 Custom Hook 承担超过 2~3 个正交职责时，任何微小的状态更新都会导致依赖该 Hook 的整个组件树重新求值，且极易出现 `useEffect` 依赖循环或 Ref 同步遗漏。

### 2. 更好的实现方式是什么？
两条标准演进路径（可按需选用）：
1. **逻辑下沉为纯 TS 控制器 / Store（0 React Hooks）**：
   - 把 EMQX 连接状态机、重连退避、消息分发、聊天记录缓存下沉到纯 TS 类/模块（或 Zustand store），用 `useSyncExternalStore` 或轻量订阅桥接给 React。这样 Hook 本身只需 3~5 个 Hooks，核心逻辑还可脱离 React 直接做单元测试。
2. **按职责拆分为小粒度子 Hook**：
   - 例如将 `useEmqxBridge` 拆为 `useEmqxConnection`（只管 WS 连接与心跳）+ `useEmqxApprovalsAndChat`（只管审批队列与消息持久化），每个 Hook 控制在 8~12 个 Hooks 以内。

---

## 四、贴线大文件与“改前先扫、先拆再写”工作流评估

**完全赞同你提出的工程流程，这是一套非常标准的“Clean-Before-Extend（Boy Scout Rule / 改前先清债）”门禁机制：**

1. **为什么不要现在对所有贴线大文件做“提前一刀切拆分”？**
   - 像 `taskSimplifiedExport.ts`（17.9KB）或 `telemetryStateResolver.ts`（17.6KB）这类稳定模块，当前所有测试与门禁全绿，提前为了拆而拆不仅收益低，还可能引入不必要的回归风险。
2. **为什么“改前先扫、先拆掉报警再写新功能”是最优解？**
   - **流程闭环**：在开发任何新功能前，先运行 `pnpm radar -- --audit`（或检查目标文件是否已处于 Soft Warning：`>400L` / `>12KB` / ESLint `max-lines` / `ReactHooks >= 24`）。
   - **如果目标文件已经报警**：**禁止直接往里塞新代码**，必须先做一次独立的预备拆分 PR/Commit，把该文件降回绿区（消除报警），然后再基于干净的结构实现新功能。
   - **效果**：既避免了“前人留债、后人擦屁股/越写越烂”的恶性循环，又把重构成本精确分摊到真正需要演进的模块上，不会浪费精力去重构无人触碰的静态代码。
