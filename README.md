# BB-Track A-2 完成报告 (`fix/bug3-replay-buffer-cadence`)

## 1. 分支与 Verified Bot Commit
- **基线分支**：`origin/z/agy/rc/multiplayer-recheck-base` (`7435735`)
- **目标分支**：`origin/fix/bug3-replay-buffer-cadence`
- **Verified Bot Commit**：`0ecf9e56d8ae03e89978e9b4e63c91efa5532d68`（`verified: true`，含 `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）
- **改动文件**（严格限定在 BB-Track A-2 独占边界内）：
  - `src/network/sync/ReplicationStateInterpolator.ts`

## 2. 根因分析与修复内容
在 Replay Watch Together 模式下，通过 Headless Chrome WebGL2 (`repro:dev-broadcast-v3`) 采样定位到 `SpectateInterpolationBuffer` 存在以下 4 个节拍与缓冲同步缺陷：
1. **重复 tick 广播误递增 `frameRevision`**：Host 播放 30Hz `.rcreplay` 或帧率波动时，60Hz 广播会发送相同 `tick` 与相同球/车位姿的快照；`pushFrame()` 无条件覆写并递增 `frameRevision++`，导致 `sample()` 返回 `isDuplicateFrame: false`，阻断客户端 WASM lookahead 接管。
   - **修复**：在 `pushFrame()` 中增加 `isSamePose(last.snapshot, snapshot)` 判定；同 `tick`、同车数且位姿未变化时直接忽略冗余广播，保持 `frameRevision` 与 `lookaheadFrames` 稳定。
2. **同帧批量到达拉高 `jitterMs`**：同一 `reconcile()` 批次推入多包时共用 `nowMs`，`actualDeltaMs === 0` 会被计入抖动采样，虚高 `interpDelayTicks`。
   - **修复**：仅在 `actualDeltaMs > 0 && actualDeltaMs < 500` 时更新 `jitterMs`。
3. **跨帧间距误判为 Teleport 重置缓冲**：当 Host 后台节流或网络卡顿产生较大 `tickDelta`（如 `55 -> 160`、`166 -> 568`）时，固定 `900 uu` 阈值的 `isSpectateTeleportJump` 会清空整个缓冲并触发单帧大跳变。
   - **修复**：新增 `isTrueTeleportJump(a, b, tickDelta)`，当 `tickDelta > 12` 时按合理最大球速（`max(900, tickDelta * 65) uu`）判定是否为真实传送，同时在 `pushFrame()` 与 `sample()` 的插值区间检查中统一使用。
4. **`120ms` 窗口回退夹紧与中等漂移硬跳**：`sinceAdvanceMs` 超过 `120ms`（但仍在 `stallTimeoutMs = 250ms` 内）时，`maxRenderTick` 突然回落至 `newestTick` 将 `renderTick` 向后拉回；此外 `|drift| > 24` 直接硬跳 `renderTick = targetTick`。
   - **修复**：在 `isWithinStallWindow` 内保留 `lookaheadMaxTick` 上界（且超窗后不向后夹紧 `renderTick`）；对 `24 < |drift| <= 600` 改为每帧最多平滑追赶 `6 ticks`，仅在 `|drift| > 600`（如大跨度 Seek）时直接对齐。

## 3. 验证结果
- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（1430/1430 TS，0 hard errors，`ReplicationStateInterpolator.ts` 保持在 18.0 KB 硬上限内）
- `pnpm run test:multiplayer-suite`：**PASS**（全量 1–27 项单测通过，含 Test 16 / 26 / 27）
- `repro:dev-broadcast-v3` 观测：
  - `SpectateInterpolationBuffer` 的 `renderTick` 全程单调推进，不再出现向后回拉与 `1479–1533 uu` 缓冲重置跳变；
  - 剩余 `frozenBallFrames` 与 tick 回拉振荡来自 AA-Track A-1 负责的 `spectatePhysicsLookaheadPipeline.ts`（`SPECTATE_LOOKAHEAD_MAX_TICKS = 30` 与无条件覆写 `physics.state[TICK]`），需与 AA-Track A-1 合流后共同转绿 `bug-3`。
