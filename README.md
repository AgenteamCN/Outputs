# Track D (`cf5cfeee`) 合入评估与 Telemetry 数据分析报告

> **评估对象**：分支 `z/agy/rc/track-d-replication-bus`（Commit `cf5cfeee60fc0cd54eb2dcdc13dbc2aa845d80a1`）  
> **数据包**：`rc-net-telemetry-csts-match-complete-2026-11-13T07-21-06.zip`  
> **结论摘要**：**Track D 可以合并到主干以解锁 Track E / Track F 并行开工**；但需明确它目前交付的是**“协议契约 + 旁路基础设施 + `game_ctrl` 通道建链”**，核心调度器尚未接入现有联机热路径。Client 侧 Telemetry 数据**整体完全符合预期**。

---

## 一、 Track D 合入前门禁与实测结果

在沙盒环境对 `cf5cfeee` 执行了完整静态检查、格式化检查、生产构建与端到端基线脚本：

1. **`pnpm run check`（Radar 2.0 + `tsc --noEmit`）**：**GREEN（全部通过）**
   - TypeScript 覆盖率：`1359/1359`（100% TS）
   - 硬边界：`0` 循环依赖、`0` 值重导出（`index.ts`）、`0` 越界向上引用
2. **Prettier 格式化检查（针对 Track D 全部 14 个变更文件）**：**PASS**
   - 注：仓库全局 `pnpm run format:check` 存在 205 个历史遗留未格式化文件，但 **Track D 自身新增/修改的 14 个文件 100% 符合 Prettier 规范**。
3. **`pnpm run build`（Vite 生产构建）**：**PASS**
4. **端到端脚本现状（供对比基线）**：
   - `pnpm run test:baseline`：除既有的 `multiplayer_6car_loadout_stress` 2 项断言（`multiplayer_kickoff_transitioned_to_playing`、`multiplayer_no_phantom_double_jump_on_flip_e`）外，其余 1v0~3v3 矩阵、场景图几何、零运行时报错均 **PASS**。
   - `pnpm run repro:multiplayer-room`：无新增崩溃，`CSTS` 物理频率扫频 `[60.7..141.99]Hz` 保持 `snaps=0 | starvations=0`；存在的 `3p3ai` 占位车与轨迹诊断告警为既有基线问题，非 Track D 引入。

---

## 二、 为什么“正常打完一局 Multiplayer Match 没受影响”——Track D 真实接线状态

通过全仓引用扫描发现，Track D 的 14 个变更文件中，**绝大多数新模块目前处于“旁路就绪（Side-by-Side Ready）”状态，尚未挂载到现有 Match 热路径上**：

| 模块 / 能力 | 现状 | 是否影响现有 Match |
| :--- | :--- | :--- |
| `HandshakeSession.ts` (`game_ctrl` ordered DataChannel) | **已接入建链**：Host 创建 `game_net` + `game_ctrl`，Client 通过 `ondatachannel` 绑定到 `DataChannelTransport.controlDataChannel` | **唯一在真实对局中跑通的改动**，证明双通道握手与 `game_net` 无回归 |
| `ReplicationBusProtocol.ts` | 定义了 `REPLICATION_MODES`、`ACTIVITY_IDS`、`REPLICATION_PACKET_FLAGS` 与 16B `SparsePingPong16B` 编解码 | 无影响（纯协议） |
| `RoomActivityOrchestrator.ts` / `ActivityKernel.ts` | 仅在 `HostWorkerMessageRouter.ts` 的 4 个新 `MSG` 分支中被实例化调用；`buildWorkerRoomStateSummary()` 尚未填充 `activeActivityId` / `activityEpoch` / `replicationMode` | 无影响（未被触发） |
| `AuthoritativeServerHost.ts` 新增方法 (`switchActivity`, `setReplicationMode`, `sendStatePausedAck`, `broadcastCsrSummary`) | 仅向 Worker 发送命令，**尚未将 Worker 回复转发至 `P2PWebRTCChannel`**；客户端也尚未消费 `DataChannelTransport.transportEvents` 中的这 4 类事件 | 无影响（未被调用） |
| `SyncAuthorityRouter.ts` / `ReplicationStateInterpolator.ts` / `AdaptiveReplicationPacer.ts` | 在上述 14 个文件之外 **0 个外部调用方**；`shouldSendAdaptiveSnapshot()`、`shouldSendSparsePing()` 尚未接入主循环发包器 | 无影响（未被接入） |

**架构评估**：
- 这正是推荐的渐进式合入策略——**先锁定协议与内核契约，不破坏现有 Match**。
- Track E（拆解 `FreeplayKernel` / `MatchKernel` / `SandboxKernel`）与 Track F（`Spectator` 观战室与接管交接）依赖的 `RoomActivityKernel`、`RoomActivityOrchestrator`、`SyncAuthorityRouter`、`ReplicationBusProtocol` 均已就位，**可以立即合并 Track D 并让 E/F 开工**。

---

## 三、 PR 合并前针对 Track D 建议补充的测试

虽然主流程无回归，但为了确保 `game_ctrl` 与新协议不是“未经实测的死代码”，建议在合入 PR 前（或作为 Track D 收尾）补齐以下 **4 项轻量验证**：

1. **双 DataChannel（`game_net` + `game_ctrl`）实机连通性验证**
   - 在真实双端 WebRTC 连接下，确认 `transport.isControlChannelOpen === true`（`control_channel_open` 事件已触发），而非静默回落到 `game_net` 或 `BroadcastChannel`。
2. **`game_ctrl` 可靠控制帧收发往返测试**
   - 手动调用 `channel.sendRoomActivitySwitch(...)` / `channel.sendStatePausedAck(...)`，验证对端 `DataChannelTransport.drainTransportEvents()` 能正确收到对应 `room_activity_switch` / `state_paused_ack` 事件。
3. **16B `SparsePingPong16B` 二进制帧编解码 + 通道往返**
   - 验证 `sendSparsePingPong16B(PING_PONG_KIND_PING, ...)` 经 WebRTC 发送 `Uint8Array(16)` 后，对端 `isSparsePingPong16B` 能准确识别并自动回传 `PONG` 更新 `simulator.handlePong()`。
4. **`HostWorkerMessageRouter` 热切换状态机单元测试 / 烟雾测试**
   - 依次向 Worker 发送 `ROOM_ACTIVITY_SWITCH('freeplay')` -> `('match')` -> `('replay')` -> `('freeplay')`，验证 `epoch` 递增、`MatchDomainState.stateOnlyMode` 随 `ReplayActivityKernel` 正确开启与恢复。
   - *顺手修复建议*：在 `HostWorkerMessageRouter.ts` 的 `buildWorkerRoomStateSummary()` 中补齐 `activeActivityId`、`activityEpoch`、`replicationMode` 三个字段的赋值。

---

## 四、 Client Telemetry 数据包分析（`rc-net-telemetry-csts-match-complete`）

对 `telemetry.csv`（9,144 ticks，时长 `159.5s`）与 `metadata.json`（`clockPacingMode: "csts"`, `clockPacingPriority: "balanced"`）进行了逐行统计分析：

### 1. 核心指标结论：**完全符合预期，表现优秀**
- **链路与抖动**：`meanRttMs = 19.1ms`（max `50ms`），`jitterMs` 均值 `1.20ms`（max `9.2ms`）。
- **时钟平滑度**：`physicsRateHz` 均值 `119.96Hz`，标准差仅 `0.287Hz`；`meanBufferTicks = 3.68`（target `3.09`），`p95BufferAbsError = 1.95 ticks`。
- **预测与回滚精度**：
  - **`tier3HardSnapCount = 0`**（全程零硬切闪现）。
  - **`localPosErrorUu`**：99%+ 的 tick 为 `0.00 uu`，仅 21 个 tick > `1 uu`（max `34.73 uu`），碰撞后 2~4 tick 内收敛回 `0`。
  - **`ballPosErrorUu`**：99%+ 的 tick 为 `0.00 uu`，仅 58 个 tick > `1 uu`（max `83.01 uu`），快速收敛。
- **边界事件解释（均为正常现象）**：
  - `remotePosErrorMaxUu = 4608.01`（`index=7178`）与开局 `index=0`（`2672.76`）：均为单 tick 远程车瞬移（开球/重置），下一 tick（`index=7179`）立即归 `0.00`。
  - 3 次 `clientTickDiff > 1300` 跳跃（`index=2000, 3050, 7892`）：均伴随 `eventFlags=129`，对应进球回放 / 开球阶段切换。
  - `starvationCount = 57`（`0.62%`）、`hardResyncCount = 1`：在 159.5s 对局中处于健康阈值内。

### 2. 一个可顺手记录的调优观察（非阻塞，与 Track D 无关）
- 在 `5,010` 次 Tier-2 回滚（占总 tick 54.8%）中，有 **95.4%（4,781 次）是在 `localPosErrorUu == 0 && ballPosErrorUu == 0` 的情况下，仅因 `remotePosErrorUu` 微小漂移（中位数 `1.59 uu`，P25 `1.12 uu`）触发的**。
- 后续在调优 CSTS Reconciler 时，可考虑适当放宽远程车辆的 Tier-2 触发死区，以进一步降低不必要的全量回滚开销。
