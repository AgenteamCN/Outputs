# RocketCompanion `z/agy/rc/multiplayer-recheck` 最新提交真实性核查报告

**核查目标仓库**：`https://github.com/Antigravi-ty/RocketCompanion`  
**核查分支与基线**：`z/agy/rc/multiplayer-recheck`（HEAD `c50af67`，包含前置提交 `a4ab863`、`24c34f6`、`0caec73`、`4a3cc3f`）  
**核查性质**：仅做工具链搭建、真实运行验证与代码/测试逐项对账，未对仓库做任何代码修改或提交。

---

## 1. Linux Headless 实测验证门禁结果

在全新环境中安装 `pnpm@12.21.0`、初始化 `public` 子模块并完成 `pnpm install` 后，四项门禁全部真实运行通过：

| 验证命令 | 结果 | 关键实测指标 |
|---|---|---|
| `pnpm run check` | **PASS** | `radar` 1421/1421 TS 文件（0 循环依赖、0 向上导入、0 any 超标）+ `tsc --noEmit` 0 错误 |
| `pnpm run test:multiplayer-suite` | **PASS** | **19/19** 全部通过（含新增 Tests 16–19） |
| `pnpm run test:dev-room-sync-e2e` | **PASS** | 9/9 阶段全部通过（耗时 74.2s，`runtime_error=0`，`poseAnomalies=0`，`stuckInGoal=false`） |
| `pnpm run repro:dev-sync-bugs` | **PASS** | **6/6** 场景全部通过（`passed=6/6`，`reproducedBugs=0`） |

### `repro:dev-sync-bugs` 6 个核心场景实测输出
1. **Freeplay Live Driving Sync**：`PASS` — `hostMoved=true`, `hostCar0=[-256,2997,17]`, `clientCar0=[-256,3086,17]`, `dist=89`。
2. **Exhibition 3v3 Spectating**：`PASS` — `hostMode="exp-multi-bots"`, `hostCars=6`, `clientMode="exp-multi-bots"`, `clientCars=6`, `clientSnaps=1`。
3. **Real 3v3 RCReplay Load & Playback + Pause ACK**：`PASS`（有测试深度盲区，见第 4 节）— `replaySource="synthetic-6car"`, `hostCars=6`, `clientCars=6`, `hostCursor=0`, `ballDist=0`, `hasPauseAckProtocol=true`。
4. **Static Training Pack Pose Sync**：`PASS` — `hostMode="static-training-pack"`, `clientMode="static-training-pack"`, `carDist=0`。
5. **Multi-Client (2+ Clients) Dev Sync Room**：`PASS` — `hostConnectedPeers=2`, `hostMode="static-training-pack"`, `client2Mode="static-training-pack"`, `client2Car0=[0,2400,17]`。
6. **Client Input Handover (`clientInputEnabled=true`)**：`PASS` — `clientInputEnabledOnClient=true`, `hostCar0Before=[256,3840,17]`, `hostCar0After=[256,3458,17]`, `clientCar0After=[256,3458,17]`。

---

## 2. 原始 Bug（Bug A–D）修复确认

1. **Bug A（Host 运行 `PredictionReconciler` 与 `AuthoritativeServerWorker` 相互覆盖物理世界）— 已修正**
   - `src/game/pipeline/stages/HostBroadcastPublisher.ts` 与 `SimulationStage.ts` 通过 `isDevSyncHostBroadcastingMode` 在 Dev Sync Room 非 Match 模式下禁用 Host 侧 `PredictionReconciler` 调和与 `syncClientCarTopology`，并由 `HostBroadcastPublisher` 以 60Hz 将主线程 `physics.saveCompactState()` 直接推入 `server.enterStateOnlyMode` / `server.pushBroadcastSnapshot`。
   - Exhibition 3v3（6 车）与 6 车 Replay 不再被 Host 的 1 车 Worker 状态抹除。
2. **Bug B（缺少 Pause ACK / 重试与低频心跳）— 已修正**
   - `src/network/transport/HostPauseAckController.ts` 实现了 `playing -> awaitingPauseAck`（每 100ms 向未 ACK 的 `peerId` 重试 `pause_state_sync`，最长 `maxRetryMs=5000`）`-> pausedIdle`（`HostBroadcastPublisher` 停止物理快照推送降至 0Hz，仅维持 1–2Hz `presence_ping` 心跳）。
   - 客户端 `PredictionReconciler.ts` 与 `RuntimePostprocessingAndSessionBootstrap.ts` 已接入 `pause_state_sync` 接收与 `pause_state_ack` 回传。
3. **Bug C（多客户端 2+ Client 加入 Dev Sync Room 失联）— 已修正**
   - `src/network/transport/hostCrossTabAndAuthority.ts`（`createHostCrossTabDiscovery`、`resolveHostCrossTabPeerChannel`）为每个未知 `senderId/clientPeerId` 动态创建通道并分配独立 `subRoomId`（`${roomId}_peer_${rawPeerId}`）；`webrtcHandshakePipeline.ts`（`classifyRoomMessage`）严格按 `targetPeerId` 过滤 Cross-Tab 控制包；`HostSessionController.ts` 在 `onPeerConnected` 与 `scheduleLateJoinWorkspaceSync` 中为迟到客户端补发 `RoomActivitySwitch`、`GlobalWorkspaceBroadcast` 及当前 `pause_state_sync`。
4. **Bug D（`clientInputEnabled=true` 时客户端驾驶无效）— 已修正**
   - 新增 `HostInputAuthorityTracker`（`authorityEpoch` 单调递增、`activeInputPeerId` 路由与旧 epoch 丢包防护）、`ClientAuthorityPublisher`（客户端以 60Hz 发送 `client_authority_snapshot`）与 `isDevSyncClientInputAuthorityMode`。
   - 客户端接管期间本地以 120Hz 独立步进并回传快照，Host 暂停本地 `physics.step(1)` 并将收到的 `clientFrame` 应用到自身渲染同时通过 `server.pushBroadcastSnapshot` 扇出给其他观战客户端。

---

## 3. 计划分阶段完成度对账（Phase 1 – Phase 4D）

| 阶段 | 计划目标 | 真实完成状态 | 核查证据 / 差异说明 |
|---|---|---|---|
| **Phase 1** | 禁用 Host 自调和 + Pure CSR 广播路径 | **已完成** | `a4ab863`、`4a3cc3f`；`SimulationStage.ts` + `HostBroadcastPublisher.ts` |
| **Phase 2** | 客户端 `SpectateInterpolationBuffer` 反向延迟插值 | **已完成** | `24c34f6`；`ReplicationStateInterpolator.ts`（`2..10` ticks 缓冲、瞬移阈值、`interpolateReplaySnapshots`）+ Unit Test 16 |
| **Phase 3** | 可靠 Pause/Resume ACK + 1–2Hz 心跳 | **核心已完成** | `0caec73`、`4a3cc3f`；`HostPauseAckController.ts` + Unit Test 17（E2E 覆盖不足，见第 4 节） |
| **Phase 4A** | 多客户端扇出、Cross-Tab 隔离、迟到客户端补发 | **已完成** | `c50af67`；`hostCrossTabAndAuthority.ts` + `HostSessionController.ts` + Unit Test 18 + Repro 场景 5 |
| **Phase 4B** | 显式客户端输入移交（`authorityEpoch` + 本地模拟回传） | **基本完成（缺相机解耦）** | `c50af67`；`ClientAuthorityPublisher` + `SimulationStage.ts` + Repro 场景 6（相机仍被覆盖，见第 4 节） |
| **Phase 4C** | FPV Master/Slave 教练抢占与 300ms 空闲自动交还 | **部分完成** | **已实现**：Host 有输入（`throttle/steer/jump/boost`）时立即抢占并递增 `authorityEpoch`。<br>**未实现**：无 `CoachAuthorityHandoverController.ts`；**无 300ms 空闲自动交还给学员的计时逻辑**；`handbrake` 未纳入输入检测。 |
| **Phase 4D** | 升级 `runDevSyncReproSuite.mjs`（回放推进/拖拽/ACK、动态训练包、3 端 Telestrator） | **未完成** | `runDevSyncReproSuite.mjs` 仍只有原 6 个场景，未新增场景 7（Dynamic Training Pack）与场景 8（1 Host + 2 Client Telestrator），场景 3 也未验证 cursor 推进与 Pause ACK 全链路。 |

---

## 4. 原始报告未覆盖的盲区与真实性隐患扫描

1. **客户端驾驶接管期间的相机覆盖盲区（Phase 4B 遗漏）**
   - `src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`（第 319–321 行）在收到 `GlobalWorkspaceBroadcast` 时无条件调用 `applyCameraSyncToPort(camera, payload.cameraSync)`；`src/ui-system/core/runtimeSpectateWorkspaceBridge.ts` 中的 `applyCameraSyncToPort` 也未判断 `clientInputEnabled` 或 `isDevSyncClientInputAuthorityMode`。当学员（Client）处于驾驶接管状态时，若 Host 触发工作区广播，学员的 `ballCam` / `isFreeCam` / `focusedCarIndex` 仍会被 Host 强行覆盖。
2. **Phase 4C 300ms 空闲自动移交未实现 + 抢占瞬间最后一帧状态可能被清空**
   - `SimulationStage.ts`（第 159–179 行）仅实现了“Host 按键抢占”，没有任何 300ms 无输入自动把控制权交还给 `targetPeerId` 的状态机。
   - 同时，抢占时调用 `hostInputAuthority.setAuthority(false, 'host')` 会在 `hostCrossTabAndAuthority.ts`（第 35 行）立即将 `this.latestFrame = null` 清空；若当帧恰好有新到达但尚未消费的 `clientFrame`，Host 会跳过该帧并从上一帧本地物理继续步进。
3. **Host 观战学员端画面时未使用 `SpectateInterpolationBuffer`**
   - `SimulationStage.ts`（第 251–257 行）在 `remoteHasAuthority` 时直接调用 `applyReplayFrameToEngine(physics, clientFrame.snapshot, interpolator)`，未经过抖动缓冲插值，在真实公网抖动下 Host 观战学员端画面可能出现微卡顿。
4. **`repro:dev-sync-bugs` 场景 3 存在“假真”验证深度不足**
   - 由于环境内不存在 `/workspace/rc-replay-3v3-0-1-2026-11-17T05-00-24.RCReplay`，场景 3 实际走入了 `synthetic-6car` 注入分支（`hostCursor: 0`）。
   - 通过条件仅检查 `repHost.numCars === 6 && repClient.numCars === 6 && hasPauseAckProtocol`（仅检测 `typeof sendPauseAck === 'function'`），**并未真实触发回放播放推进、Pause -> 客户端 ACK -> 0Hz -> 拖拽单帧同步 -> Resume** 的浏览器端到端链路。
5. **动态训练包与 3 端 Telestrator 尚未纳入 `repro:dev-sync-bugs`**
   - 计划 Phase 4D 要求的“Dynamic Training Pack 切题/重置同步”与“1 Host + 2 Client Telestrator (0x05) 双向绘制与权限门禁”尚未补齐到 `runDevSyncReproSuite.mjs`（仅在单测 `runMultiplayerSuiteTests.ts` 与单 Client `runDevRoomSyncE2E.mjs` 中部分覆盖）。

---

## 5. 总结结论

- **已完成并真实有效**：Dev Sync Room 下的 **Bug A（Host 自调和冲突）、Bug B（Pause ACK/心跳协议）、Bug C（多客户端连接与迟到补发）、Bug D（显式客户端输入移交）** 以及 **Phase 1、Phase 2、Phase 3、Phase 4A、Phase 4B 核心链路** 均已真实落地，且 4 项自动化验证门禁全部 100% 通过。
- **尚未收口部分**：**Phase 4C 的 300ms 空闲自动交还**、**客户端接管时的相机解耦**、以及 **Phase 4D 在 `runDevSyncReproSuite.mjs` 中的回放推进/Pause ACK 实测与场景 7/8 补齐** 仍待后续补全。
