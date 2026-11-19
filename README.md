# Batch B Branch 6 (`refactor/agy-stores-slim`) 完成报告

## 1. 任务与分支信息
- **目标分支**：`refactor/agy-stores-slim`（基于 `a2` 最新提交 `e95e292`）
- **Verified Bot Commit**：[`4c72644661e9690c175c66126e8d6b4a9887b6c0`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/4c72644661e9690c175c66126e8d6b4a9887b6c0)（GPG 签名 `VALID`，`isValid: true`）
- **工具链补齐**：安装 `pnpm@12.21.0`，执行 `pnpm install --frozen-lockfile` 与 `pnpm sync:profile`。

## 2. 拆分与重构明细

### 2.1 `telemetryStore.ts`（1,103L / 32.4KB → 398L / 15.5KB）
- **纯计算与快照合并管道** -> `src/utils/telemetrySnapshotMerge.ts`（396L）：
  - 提取 `normalizeProbeState`、`parseRemoteImportPayload`（消除空 `catch`）、`promotePrefix6Entry`、`mergePrefix6EntriesIntoFullEnvIds`、`buildDispatchedActiveEnvEntry`、`buildGuiRoundStartedUpdate`、`mergeTelemetrySnapshotPatch`、`mergeAllShardsTraces`、`extractDirtyTracesPayload`。
- **同步/归档/分片/心跳计时子控制器** -> `src/stores/telemetrySyncCoordinator.ts`（300L）：
  - 封装 `dirtyTraceEnvIds`、`inFlightEnvTelemetryLoads`、`inFlightAllShardsLoad`、`scheduledFlushTimer`、`heartbeatExpirationTimer` 及其 `dispose()` 生命周期，维持 `UnscopedCaches=0`。
- **Broker 连接与订阅子控制器** -> `src/stores/telemetryBrokerCoordinator.ts`（181L）：
  - 封装 EMQX Global / EMQX CN / Bemfa SZ 连接生命周期、`lastEnvEmqxPullAtMs` 节流与 `publishRetained` / `dispose()`。

### 2.2 `presetStore.ts`（622L / 20.3KB → 343L / 12.6KB）
- **持久化、通用 CRUD 与默认值解析管道** -> `src/utils/presetPersistence.ts`（275L）：
  - 提取存储键常量、`loadPresetStorageSnapshot`、`appendAndPersistPreset`、`updateAndPersistPreset`、`removeAndPersistPreset`、`reorderAndPersistPreset` 与各领域 `resolveDefault*` 纯函数，并消除 `readJsonList` 空 `catch`。

### 2.3 `themeStore.ts`（625L / 18.7KB → 338L / <12KB）
- **本地存储与 DOM 样式应用管道** -> `src/utils/themeStorageAndDom.ts`（283L）：
  - 统一 `readStorageItem` / `writeThemeStorageItem` 错误日志告警，**彻底清零 `themeStore.ts` 原有的 18 处空 `catch (_) {}`**；
  - 抽取 `readAllAppearanceSettings`、`persistAppearanceDefaults` 与 `apply*ToDocument` 函数，消除初始状态与 `initTheme` 之间的重复读取逻辑。

## 3. 验收与检查结果

| 检查项 | 重构前基线 | 重构后结果 |
| :--- | :--- | :--- |
| `telemetryStore.ts` 行数 | 1,103 行 | **398 行**（`<= 400`） |
| `presetStore.ts` 行数 | 622 行 | **343 行**（`<= 400`） |
| `themeStore.ts` 行数 | 625 行 | **338 行**（`<= 400`） |
| `HardVolume` 硬超标文件数 | 19 / 19 | **16 / 19**（3 个 Store 全部退出硬超标清单） |
| `SilentCatches` 棘轮 | 56 + 50 | **36 + 50**（净减少 20 处空 `catch`） |
| `Cycles` / `LayerViolations` | 0 / 0 | **0 / 0** |
| `BoolHotspots` / `UnscopedCaches` | 0 / 0 | **0 / 0** |
| `pnpm check`（tsc + eslint + vitest） | 通过 | **通过（0 errors，2 个测试文件 16/16 用例通过）** |
| `pnpm p2p:check` | 通过 | **通过（0 errors）** |
| `pnpm radar` | 通过 | **通过（All hygiene & architecture checks passed）** |
