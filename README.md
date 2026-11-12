# Radar 输出格式重构与体积警报分级设计报告

## 1. 任务背景与核心痛点
在原有 `scripts/radar.ts` 架构守卫检查中：
1. **冗余噪音过大**：每个超过软阈值（12KB / 400L）的文件都会打印一遍长达数百字符的拆分规范说明。当多个文件触发时，长达上千行的重复文案导致 CI 日志刷屏，且淹没了关键的文件路径与体积数据。
2. **文案歧义与缺乏分级**：提示文案统一标记为 `modified file notice(s)`，未明确区分增量检测与全量扫描；同时缺少将 **拦截级硬报错（Hard Limit Exceeded）** 与 **非拦截级软提示（Soft Info Notices）** 清晰分层聚合的展示机制。

---

## 2. 方案实现与架构调整

### 2.1 数据结构解耦 (`scripts/radar/radarConstantsAndFs.ts`)
引入专用的 `FileVolumeMeta` 接口，并在 `RadarScanResult` 中显式分离硬性体积违规与软性通知：
```typescript
export interface FileVolumeMeta {
  rel: string;
  byteSize: number;
  kbFormatted: string;
  lineCount: number;
  exceedsHardLine: boolean;
  exceedsHardByte: boolean;
}

export interface RadarScanResult {
  errors: string[];
  hardCeilingViolations: FileVolumeMeta[];
  softLimitNotices: FileVolumeMeta[];
  tsCount: number;
  jsCount: number;
  subsystemStats: Map<string, SubsystemCount>;
  tsResult: TypeScriptCheckResult;
}
```

### 2.2 聚合扫描与降序排序 (`scripts/radar.ts`)
1. **聚合分类**：在文件遍历阶段，体积超标的逻辑分支将违规对象分别注入 `hardCeilingViolations` 与 `softLimitNotices`。
2. **体积降序**：对两个列表均按 `byteSize` 降序（相等时按 `lineCount` 降序）排序，优先暴露出最接近或超出上限的临界模块。
3. **两列对齐输出**：实现 `formatVolumeList` 辅助方法，统一计算路径最大宽度（`padEnd`），输出整洁的 `- <path>  (<size> KB, <lines> lines)`。
4. **统一指引提炼**：拆分指导建议仅在各自 Section 顶部打印一次，彻底根除 90+ 次冗余长句复制。

---

## 3. 多场景实机验证

### 场景 1：全仓纯净态扫描 (`pnpm check`)
```text
======================================================================
 ROCKET COMPANION GEN-3 SEMANTIC ARCHITECTURE RADAR (8226ms)
======================================================================
 TypeScript Coverage : 1350/1350 files (100.0% TS | 0 Engine JS remaining)
 TypeScript Strict   : PASS ("/work/RocketCompanion/node_modules/.bin/tsc" --noEmit)
 Volume Guardrails   : SoftInfo=400L/12KB | HardError=800L/18KB | MaxLine<=400chars
 Hard Boundaries     : 0 Circular Deps | 0 Value Re-Exports | 0 Upward Imports | Any<=5
 Anti-Fake-Split     : 0 Multi-Stmt Lines | HostPen<=105 | PassThru<=40 (4 audit targets; pass --audit)
 Engine Subsystems   : src(1ts), ai(20ts), audio(38ts), camera(11ts), effects(8ts), entities(103ts), game(160ts), input(3ts), loader(10ts), network(96ts), physics(23ts), replay(106ts), utils(3ts)
 UI Subsystems       : entry(2ts), ui-core(533ts), ui-system(20ts), ui-tuner(213ts)
----------------------------------------------------------------------
 INFO (Non-blocking) : 442 existing file(s) in 12KB..18KB / 400..800L range — Status: ACCEPTABLE (pass --all-info to list).
----------------------------------------------------------------------
 STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified.
======================================================================
```

### 场景 2：触发软阈值提示（`pnpm run radar --all-info` 或修改文件）
说明指引单次输出，列表按体积降序规整对齐，退出状态码为 `0`：
```text
[-] INFO: Volume Soft Threshold Notices (Status: ACCEPTABLE — 442 files):
    Files exceeding soft bounds (> 12.0 KB or > 400 lines). Non-blocking.
    Refactoring guidance: No immediate action required. Do NOT artificially compress
    code or shorten names. If adding features later, split by domain/pure pipelines.

    - src/game/modes/FreeplayMode.ts                                                          (17.9 KB, 433 lines)
    - src/ui-core/stores/quickNotificationStore.ts                                            (17.4 KB, 500 lines)
    - src/ui-tuner/components/review/RapidReviewTriageBar.tsx                                 (17.3 KB, 452 lines)
    - src/ui-tuner/components/review/reviewLedgerStore.ts                                     (17.1 KB, 519 lines)
...
----------------------------------------------------------------------
 STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified.
======================================================================
```

### 场景 3：硬阈值违规拦截注入测试（模拟超过 18KB / 800L）
雷达精准拦截，报告置顶标红，退出状态码为 `1`：
```text
[!] ERROR: Hard Limit Exceeded (Status: NOT ACCEPTABLE — 1 file):
    Violations exceed hard ceilings (> 18.0 KB or > 800 lines). Build rejected.
    Action required: Must split strictly by functional domain, pure logic pipeline,
    or independent UI Slot before commit.

    - src/game/modes/FreeplayMode.ts (27.5 KB, 934 lines)
----------------------------------------------------------------------
 STATUS: FAILED -- Build rejected due to architecture or volume violations.
======================================================================
```

### 场景 4：硬报错与软提示共存测试
硬性错误置顶拦截，软性提示紧随其后统揽，状态码严格为 `1`。

---

## 4. Git 提交、PR 与自动化合并信息
- **工作分支**：`z/agy/rc/radar-output-compact`
- **提交方式**：GitHub Verified Bot Commit (`verified: true`)
- **Commit SHA**：`17303919aa9c8acf61d13568412dfdac29d4b35a`
- **Pull Request**：[#7](https://github.com/Antigravi-ty/RocketCompanion/pull/7)
- **合并状态**：已合并至主基准分支 `alpha`（Merge commit: `f5e8c9f8de5e901c3278fd88e2d872eef9784ceb`）
- **CI 验证**：`pnpm check`（`radar` + `typecheck`）100% 通过。
