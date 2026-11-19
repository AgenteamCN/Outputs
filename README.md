# Wave 0 Arch-0D 完成报告（快照离散字段与连续插值语义隔离）

- **仓库**：`Antigravi-ty/RocketCompanion`
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck` (`d69e01c`)
- **新分支**：`z/agy/rc/wave0-arch-0d`
- **Verified Bot Commit**：[`ed7d5df4a8fd9fd39bfa0b03458f0e49feec7932`](https://github.com/Antigravi-ty/RocketCompanion/commit/ed7d5df4a8fd9fd39bfa0b03458f0e49feec7932)（`signature.isValid: true`）

## 1. 变更内容

1. **显式声明 BoostPad 离散/连续字段偏移表与隔离插值管线**
   - 新增 [boostPadSnapshotInterpolationPipeline.ts](file:///workspace/RocketCompanion/src/replay/boostPadSnapshotInterpolationPipeline.ts)：
     - 显式声明 `BOOST_PAD_DISCRETE_FIELD_OFFSETS`（`isActive` 为 `'discrete-step'`，`cooldown` 为 `'monotone-cooldown'`，跨帧向上跳变阈值 `cooldownJumpThresholdSec = 0.05`）。
     - 实现 `interpolateBoostPadSnapshots(prevSnap, currSnap, out, t)`：
       - `isActive` 严格使用最近邻/阶跃取值（`t >= 0.5 ? currActive : prevActive`），绝不线性插值，并保持输出为 `0` 或 `1`。
       - 当 `outActive === 1` 时强制 `cooldown = 0`。
       - 当 `prevActive === 0 && currActive === 0` 且处于单调倒计时（`currCd <= prevCd + 0.05`）时对 `cooldown` 线性插值；若发生跨帧向上跳变（如重生后立即被拾取或状态重置），则禁止跨跳变插值，改用阶跃取值。
   - 更新 [ReplayFrameInterpolator.ts](file:///workspace/RocketCompanion/src/replay/ReplayFrameInterpolator.ts)：
     - 在 `interpolateReplaySnapshots()` 末尾调用 `interpolateBoostPadSnapshots(prevSnap, currSnap, out, t)`，消除 BoostPad 状态随 `currSnap` 无条件拷贝/跳变的问题（同时满足 Radar 18.0 KB 体积硬上限）。

2. **明确 `SpectateInterpolationBuffer.sample()` 输出契约**
   - 更新 [ReplicationStateInterpolator.ts](file:///workspace/RocketCompanion/src/network/sync/ReplicationStateInterpolator.ts)：
     - 显式声明 `SPECTATE_BOOST_PAD_DISCRETE_OFFSETS`。
     - 在 `SpectateInterpolationSample` 新增 `frameCursor: number` 与 `isDuplicateFrame: boolean` 字段；在 `SpectateInterpolationStats` 新增 `duplicateSamples: number`。
     - 在 `SpectateInterpolationBuffer` 中追踪 `frameRevision` 与上一次采样的 `lastSampledFrameCursor` / `lastSampledRevision`，所有 `sample()` 返回路径统一经由 `buildSample(...)` 输出 `frameCursor` 与 `isDuplicateFrame`（`reset()` 时清除游标历史）。

3. **回归与契约单元测试**
   - 更新 [runMultiplayerSuiteTests.ts](file:///workspace/RocketCompanion/scripts/runMultiplayerSuiteTests.ts) 的第 16 项测试 `testSpectateInterpolationBufferReverseDelay`：
     - 覆盖 `frameCursor`、`isDuplicateFrame`（重复采样判定与 `reset()` 后复位）。
     - 覆盖 BoostPad `isActive` 阶跃取值、`isActive === 1` 时 `cooldown === 0`、单调倒计时插值以及跨帧向上跳变保护。

## 2. 工具链与门禁验证

- 工具链补充：全局安装 `pnpm` (`12.21.0`)，完成 `pnpm install`。
- 门禁结果（全部通过）：
  - `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1426/1426 TS 文件，0 循环依赖，0 值重导出，所有文件均在 18.0 KB / 800 行硬上限内）
  - `pnpm run radar:audit`：**GREEN**
  - `pnpm run test:multiplayer-suite`：**24/24 PASS**
