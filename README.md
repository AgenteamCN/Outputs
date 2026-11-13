# RocketCompanion 多人联机版本深度审查报告（`49ecac7` vs `9a5d063`）

## 一、 核心结论（直接回答）

1. **这两个版本（`49ecac7` 和 `9a5d063`）均未做完。**
   它们是同一天基于 `24fc4a4`（`z/agy/rc/tauri/webtransport-tunnel`）并行尝试的两个半成品分支：
   - `49ecac7360da47af7e09ec82d953c202086aadef`（`origin/z/agy/rc/phase2-implementation`）
   - `9a5d06376e121a8296ef4b12ed5a42cdfbd1706a`（`origin/z/agy/rc/multiplayer-suite`）
   总体情况是：**协议/Store/孤立辅助类写了不少，但关键的运行时主链路（Session 生命周期、Voice Peer 绑定、UI 全局状态广播、Ghost Ball 同步、真实多端 E2E）普遍处于“只写了定义、0 处调用”的断连状态**。如果现在合入，联机语音、观战窗口镜像、Multiplayer Training 房间切换与多球 Ghost 均无法在真实对局中工作。

2. **自动化测试“表面全绿”，但存在严重的“自欺欺人式 Mock”，完全没有遍历 `1v2 / 2v1 / 2v2 / 2v3 / 3v3` 等真实组合。**
   - 实测结果：
     - `49ecac7`：`pnpm check`（1410/1410 TS，0 报错）、`pnpm build`、`pnpm run test:matrix`（8/8 PASS）均返回退出码 `0`。
     - `9a5d063`：`pnpm check`（1404/1404 TS，0 报错）、`pnpm run test:multiplayer-suite`（6/6 PASS）均返回退出码 `0`。（注：同环境 `pnpm build` 因先前构建残留的 `dist/.git` 只读 pack 权限报错，代码本身无 TS/打包语法错误。）
     - `cargo check`：因沙盒未安装 `cc` 链接器未完成本地编译，但两个提交均**未修改** `src-tauri/` 下任何 Rust 代码。
   - **测试矩阵的真实水分**：
     - **未扩展真实 E2E 脚本**：仓库原有的多标签页无头 Chrome E2E 脚本 `scripts/runMultiplayerRoomRepro.mjs` 在两个提交中**只字未改**，仍然只支持 `2p`（1v1）、`2p2ai`（2v2）、`3p3ai`（3v3）。
     - **`49ecac7` 的 `testBotRosterMatrix()` 是纯本地假循环**：`scripts/MultiClientLoopbackSimulator.ts`（与原有 `src/game/scenario/MultiClientLoopbackSimulator.ts` 重名）里的 `testBotRosterMatrix()` 完全没有调用任何游戏/网络/物理代码，只是在测试函数里 `for` 循环往本地数组 `slots` 里 `push` 对象并断言数组长度，且只写了 `1v1`、`2v2`、`3v3`，没有 `1v2 / 2v1 / 2v3`。
     - **`9a5d063` 的 `testMultiClientLoopbackBotMatrix()` 只跑了写死的 6 车 Mock**：`scripts/runMultiplayerSuiteTests.ts` 调用了原有的 `src/game/scenario/MultiClientLoopbackSimulator.ts`（`startSixCarLoopbackSession`），但传入的是完全写死的 `mockRuntime`（`physics: { numCars: 6, isCarSlotActive: () => true }`, `arena: { cars: 6 }`），根本没有运行真实物理引擎、房间状态机或 `1v2 / 2v1 / 2v2 / 2v3 / 3v3` 组合矩阵。

---

## 二、 逐模块差距对照（原计划 vs `49ecac7` vs `9a5d063`）

| 模块 | 原计划核心要求 | `49ecac7` (`phase2-implementation`) 状态 | `9a5d063` (`multiplayer-suite`) 状态 | 关键差距 / 致命问题 |
| :--- | :--- | :--- | :--- | :--- |
| **1. P2P Tunnel / WebTransport** | 数据包走 WebTransport/UDP 隧道，具备可靠分片/重传语义；语音保留 WebRTC 混合模式 | **未接通**：写了 `ensureSessionTunnelOpen` / `connectPeerTunnel` 但**全仓 0 调用**；写了 `ReliableTunnelFramer.ts`（分片+重传）但未真正运行 | **半接通**：在 `hostRoom`/`joinRoom` 调用 `openSessionTunnelIfTauri` 并通过 `tunnel_endpoint_advertise` 绑定到 `DataChannelTransport` | 1. 两者均仅在 `isRustTunnelAvailable()`（Tauri 桌面端）启用，浏览器端 `WebTunnelTransport` 仍是 WebRTC DataChannel，无纯浏览器 WebTransport。<br>2. **`9a5d063` 缺少 `ReliableTunnelFramer`**：直接将 `reliable_control` 和大包走裸 UDP `RustTunnelTransport`，超 MTU（~1200B）或丢包时必丢帧/乱序。 |
| **2. Audio / WebRTC 语音** | 移植语音模块；修复 `VoiceMasterOutputBus` 路由与静默 `.catch`；绑定会话生命周期；支持 `open` / `team_only`（Lobby 全员互通）/ `off` | **主链路断连**：`voiceAudioGraph.ts` 已修复；写了 `runtimeVoiceBridge.ts`，但 **`attachVoicePeer` 全仓 0 调用**；甚至未挂入 `AudioTab.tsx` | **主链路断连**：`voiceAudioGraph.ts` 已修复；已挂入 `AudioTab.tsx` 且 `runtimeOnlineBridgeHandlers.ts` 同步了策略，但 **`attachVoiceToPeerChannel` 全仓 0 调用** | **两版均无法在真实联机中通话**：Peer 连接时未调用 `attachVoice*` 注册 `channel.onVoiceSignal` 与 `bridge.attachPeer`；`setVoicePeerTeam` 也 0 调用，换队/进对局后 `team_only` 无法动态更新 Peer 队伍。 |
| **3. Multiplayer Training** | 新增 `MULTIPLAYER_TRAINING` Kernel；隐藏计分板、保留 Goal Replay；默认 `limit_when_aerial`（地面不耗气且可补气）；`Custom` 跳 Advanced 页；Ghost Client 默认 OFF + Host Relay | **未接通**：写了 `MultiplayerTrainingKernel.ts` 和 `TrainingBoostPolicy.ts`，但 `PhysicsEngineManager` 未调用；`Layer3HostTrainingRecipe` 未触发 `switchActivity` | **部分接通**：`PhysicsEngineManager.stepAuthoritativeTick()` 已按 `sim.getCarStateView(i).isOnGround` 调用 `applyTrainingBoostPolicyToSim` | 1. **UI 未接线**：`Layer3HostTrainingRecipe.tsx` 未触发 `switchActivity(MULTIPLAYER_TRAINING)`，也未实现选中 `Custom` 跳转 `Advanced` 页。<br>2. `hideScoreboard` 无任何 HUD 读取。<br>3. `GhostBallRelayCoordinator` 无客户端上报与 Arena 多球渲染。 |
| **4. Spectating 与全局状态广播** | 区分仅画面观战 vs 镜像观战（`spectate_mirror`）；同步菜单路由与浮窗开关状态（**不带坐标，坐标跟 Client 走**），本地只读锁定，退出恢复；Pause ACK + 1–10Hz 心跳；Telestrator 默认 `host_only` | **主链路断连**：写了 `SpectatorGlobalStateCoordinator` + `spectatorSyncStore`，但 **`buildGlobalUiBroadcast` / `applyHostGlobalUiState` 在网络同步中 0 调用** | **主链路断连**：写了 `GlobalWorkspaceProtocol` + `spectateWorkspaceStore` + `DuringMatchSpectatorsSection`，但 **`buildGlobalWorkspaceBroadcast` / `applyHostWorkspaceBroadcast` 0 调用** | 1. Host 不会采集并发送全局 UI 状态，Client 也不会接收并应用。<br>2. 真实 `FloatingWindow` 与菜单系统未接入 `isReadOnlyLocked` 只读锁及开关同步。<br>3. `STATE_PAUSED_ACK` 与 1–10Hz 心跳未接入 `AuthoritativeServerHost` / `RoomReplicationBus`。 |
| **5. Kick & Leave 双向信令** | Kick 前发送可靠控制帧并退回大厅；Leave 先发控制帧更新 Roster/语音再销毁连接 | **存在时序 Bug**：`removeRemotePlayer` 调用 `kickPlayerByCarIndex` 发送 `session_kick` 后立即销毁连接 | **存在时序 Bug**：`removeRemotePlayer` 发送 `peer_kick` 后**同步立即 `ch.destroy()`** | WebRTC DataChannel 发送控制帧后立即同步 `destroy()`（`dc.close()`）会导致控制帧在离开缓冲前被丢弃，需改为短暂 flush/ACK 延迟销毁；两版信令命名也不统一（`session_kick` vs `peer_kick`）。 |
| **6. 命名与架构卫生** | `TelestratorSettingsPanel.tsx` 改名为 `*FloatingWindow.tsx` | 仅移动到 `telestrator/` 子目录，未改名；新增了与 `src/game/scenario/MultiClientLoopbackSimulator.ts` 重名的 `scripts/MultiClientLoopbackSimulator.ts` | 已正确重命名为 `TelestratorSettingsFloatingWindow.tsx` | `49ecac7` 留下了重名文件与未统一命名。 |

---

## 三、 是否推荐从 WebTransport 开始重做？

**结论：不需要推倒重做 WebTransport 底座，但必须停止在现有两个分支上直接“打补丁式合并”，推荐以 `9a5d063` 为骨架基线、吸收 `49ecac7` 的 `ReliableTunnelFramer`，按闭环链路重构收口。**

原因如下：
1. **WebTransport 底座已经可用**：基准分支 `24fc4a4` 已经完成了 Rust 侧 `src-tauri/src/tunnel_bridge.rs`（本地 QUIC WebTransport + WebSocket 降级）与前端 `LocalBridgeClient.ts` / `RustTunnelTransport.ts`，`9a5d063` 也已打通 `tunnel_endpoint_advertise` 交换。这部分不需要重写，只需：
   - 把 `49ecac7` 的 `ReliableTunnelFramer.ts` 接入 `9a5d063` 的 `DataChannelTransport` 隧道发送路径，解决大包分片与可靠控制帧丢包问题；
   - 明确浏览器端无原生 P2P WebTransport（浏览器之间只能走 WebRTC DataChannel 或通过中继服务器），`WebTunnelTransport` 作为浏览器降级通道保持现状即可，无需在浏览器强行造不存在的 P2P WebTransport。
2. **当前最大的问题不是底层传输，而是“模块之间没接线”**：
   Voice、Spectating Global Broadcast、Multiplayer Training UI/Ghost Relay 全都是“写好了类/Store，但没人调用”。

---

## 四、 下一阶段收口与重构建议路线

建议按以下 4 步逐一打通并做真机/无头浏览器闭环验收：

1. **Step 1：收口基线与传输可靠性**
   - 基于 `9a5d063` 切出新集成分支，合入 `49ecac7` 的 `ReliableTunnelFramer.ts` 并接入 `DataChannelTransport` 的隧道收发路径。
   - 修复 `OnlineRemotePeerVehicleSynchronizer.removeRemotePlayer` 与 `leaveSession` 的销毁时序：发送 `peer_kick` / `peer_leave` 后等待 DataChannel bufferedAmount 清空或 100–150ms 延迟后再 `ch.destroy()`。
2. **Step 2：打通 Voice 真实会话生命周期**
   - 在 `HostSessionController` / `ClientSessionController`（或 `EstablishedSessionManager`）建立 Peer 连接时调用 `attachVoiceToPeerChannel(channel, peerId, peerName, team)`，断开时调用 `detachVoiceFromPeer(peerId)`。
   - 在队伍变更、Lobby <-> Match 切换时同步调用 `setVoicePeerTeam()` 与 `syncVoiceRoomPolicy()`，确保 `team_only` 在 Lobby 全员互通、对局内按队伍隔离。
3. **Step 3：打通 Multiplayer Training 与 Spectating 真实闭环**
   - 将 `Layer3HostTrainingRecipe.tsx` 的启动动作与 `BoostPolicy` / `Custom` Advanced 页（Per-Player Boost）接到 `RoomActivityOrchestrator.switchActivity(MULTIPLAYER_TRAINING)`；在 HUD 计分板组件中消费 `hideScoreboard`；补全 `ghost_ball_state` 上报与 Arena Ghost 球渲染。
   - 将 `RoomActivityOrchestrator.buildGlobalWorkspaceBroadcast` 接入 Host 定时/事件广播，Client 收到后写入 `useSpectateWorkspaceStore.applyHostWorkspaceBroadcast`，并把 `isReadOnlyLocked` 与浮窗 `isOpen/isMinimized`（保留 Client 本地 `(x, y)` 坐标）接到真实 `FloatingWindow` 组件；将 `STATE_PAUSED_ACK` 与 1–10Hz 心跳接入 `AuthoritativeServerHost`。
4. **Step 4：建立“真·遍历”自动化测试矩阵（拒绝假 Mock）**
   - 扩展 `scripts/runMultiplayerRoomRepro.mjs` 与 `src/game/scenario/MultiClientLoopbackSimulator.ts`，**真正参数化遍历**：
     - **人数与人机组合矩阵**：`1v1`（1H+1B / 2H）、`1v2`（1H vs 2B / 1H+2B）、`2v1`、`2v2`（2H+2B / 4H）、`2v3`、`3v3`（1H+5B / 3H+3B / 6H），覆盖**真人接管 Bot** 与 **Auto-fill Bot** 场景。
     - **功能闭环断言**：在真实 `PhysicsEngineManager` + `RoomReplicationBus` + 无头 Chrome WebRTC 下验证 Boost 策略（`limit_when_aerial` 地面不耗气/空中消耗/地面吃包回满）、Ghost Relay、Kick/Leave 双方状态清理、Voice 门控状态、`spectate_mirror` 窗口同步与坐标保留、Telestrator 50-Stroke HashAck。
