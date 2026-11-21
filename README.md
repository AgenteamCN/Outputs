# RocketCompanion vNext 联机与观战架构：实时对齐、端到端接线与真机排障交付报告

> **目标分支**：`origin/integration/vnext-core`
> **最新 Verified Bot Commit**：`1b99959` (`1b999594c106460c12f3d39a6646f7f48a683d57`)
> **实时协作频道**：`ch_73eb3a4e`（`agentlivechatchannel.maboco-mark.workers.dev`）

---

## 一、Git 历史追溯留痕与分支清理

1. **10 父节点 Verified 追溯合并提交**：`8b081ec`（`verified: true`）
   - 保持 `e3488ff` 文件树 100% 不变（`-s ours` 风格），将 `e3488ff` 与 9 条 Wave 1 并行特性分支（`W1-01..W1-09`）的 HEAD 全部关联为父节点，在 Git DAG 图谱中完整保留了 9 条分支的独立开发历史与可回溯性。
2. **清理冗余临时分支**：
   - 已从远端删除 9 个临时特性分支引用（HTTP 204）：`feat/worker-authority-bus`、`feat/timeline-frame-buffer`、`feat/mode-match-exhibition`、`feat/mode-freeplay-spectate`、`feat/mode-replay-watch-together`、`feat/mode-training-fpv-dual`、`feat/consumer-audio-arena`、`feat/consumer-hud-camera`、`feat/bulk-asset-channel`。

---

## 二、Wave 4 端到端接线与真机联调修复

| Commit | 修改文件 | 核心改动与解决问题 |
|---|---|---|
| `2bfe660` | `src/network/client/reconcilerInboundPacketParser.ts`<br>`src/replay/ReplayFrameInterpolator.ts`<br>`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts` | 1. 客户端接入 `decodeVNextChannelPacket`（识别 Channel 1 `WorldFrame` 与 Channel 2 `RosterManifest`）；<br>2. 流式网络观战/回放帧取消强制 `preserveEngineTick: true`，使客户端 `tickCount` 正常推进；<br>3. 批量到达帧按 16.67ms 分布 `arrivalMs`，修复 `TimelineFrameBuffer` 同帧时间戳冲突。 |
| `e5a0fc4` | `src/network/server/AuthoritativeServerWorker.ts`<br>`src/network/client/reconcilerInboundPacketParser.ts` | 1. 在所有状态广播模式（Freeplay / Replay Watch Together / Exhibition / Spectate）下，`AuthoritativeServerWorker` 直接编码并发出 `VNextWorldFrameCodec` 通道 1 二进制包；<br>2. 客户端 `decodeVNextChannelPacket` 分支补齐 `COMMAND_ACK` 信号确认。 |
| `2e39376` | `src/network/client/reconcilerInboundPacketParser.ts` | 修复真机“Client 进房一动不动但菜单同步”的根因：`VNextWorldFrameCodec` 写入 4B `schemaHash` 为大端序（Big-Endian），接收端读取改为 `getUint32(0, false)`，端到端物理帧恢复同步。 |
| `1b99959` | `src-tauri/src/lib.rs`<br>`src/game/modes/ExperimentalMultiBotsMode.ts` | 1. Tauri 主窗口触发 `WindowEvent::Destroyed` 时立即调用 `cleanup_all_tunnels`，解决 macOS 关窗重开后无法建房的问题；<br>2. Exhibition 观战模式下 Boost HUD 改读 `r.camera.focusedCarIndex` 对应车辆，解决镜头在 1 号车但 Boost 显示他车的问题。 |

---

## 三、门禁与 E2E 验证结果

- `pnpm run check`（Radar + `tsc --noEmit`）：**GREEN**（1457/1457 TS，0 错误，0 循环依赖，0 越权导入）
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**PASS（30/30 全部通过）**
- `pnpm run repro:dev-broadcast-v3`（在 `2e39376` 上实测）：
  - `bug-1`（Goal Replay）：**PASS_FIXED**
  - `bug-2`（Replay 6 车 NameTag & Team & Camera Edge Sync）：**PASS_FIXED**
  - `bug-4`（BoostPad 防闪烁）：**PASS_FIXED**
  - `bug-3`：`distinctTicks` 已恢复为 `15`（正常推进）

---

## 四、下一步规划

1. **Tauri 与 Exhibition 验证**：拉取 `origin/integration/vnext-core`（`1b99959`），重启 `pnpm tauri:dev:tray` 验证关窗重开建房与 Exhibition Boost HUD。
2. **Exhibition 通道 2 广播**：在 `AuthoritativeServerWorker` 中补齐通道 2 `VNextRosterManifestCodec` 广播，完成 Exhibition 6 人用户名单同步。
3. **双端数值一致性自动化测试**：基于 Host 与 Client 运行时核心数值快照（`focusedCarIndex` 对应 Boost、`RosterManifest` 名字/阵营、球与 6 车位姿/tick）建立自动化一致性断言。
