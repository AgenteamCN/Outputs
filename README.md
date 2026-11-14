# RocketCompanion 多人联机与 Tauri UDP Tunnel 二次核查报告

**基准 Commit**: `af528a850a84466d6a263ba92f8bab974cc052a1`  
**工作分支**: `z/agy/rc/multiplayer-recheck`  
**Verified Commit**: `e6bb63788fbf7b2752c740bfc7dce9ffcdcc941a` (`signature.isValid: true`, `state: "VALID"`)

---

## 一、核心结论：是否已达到 Real Device Test 状态？

**结论：核心传输、信令与活动同步链路已修补完成，可进入「受控 Real Device Test（真机 Alpha 联调）」阶段**；但在全量自动化验收上仍有 2 项明确边界需要注意：

1. **已打通并实测通过的部分**：
   - `pnpm run check`（Radar 1409/1409 TS + `tsc --noEmit`）：**PASS**
   - `pnpm run test:multiplayer-suite`（从 6 项扩展至 **10 项**）：**PASS**
   - `pnpm run repro:multiplayer-chaos`（3 项混沌流）：**PASS**
   - `pnpm run build`（Vite 生产构建）：**PASS**
   - `cargo check` & `cargo test`（`src-tauri` 补齐工具链后）：**PASS（2/2 Rust 单元测试通过）**
   - `pnpm run repro:multiplayer-room -- --mode=2p --max-duration-ms=180000 --chaos --glitch-check`（真实多标签页 Headless Chrome）：**EXIT=0**（Host/Client 成功建链、`visibleCars=[0,1]`、`bw=487~504kbps`、`stuckInGoal=false`、`poseAnomalies=0`、`runtime_error=0`）。
2. **尚未闭环的边界（不影响真机传输验证，但需在后续迭代处理）**：
   - **UI 控制面板尚未提供 Developer 房内活动切换入口**：`src/ui-*` 中仍无调用 `matchPort.switchRoomActivity` 的界面控件（需通过调试端口或后续 UI 接入）。
   - **Chrome Repro 中 `Edge-Case 3: 2-Goal Replay->Kickoff` 仍显示 `completed=0/2 | clientFrozen=true`**：原因是 `forceStartOnlineMatch(90)` 后无车辆物理触球，比赛停留在 `phase=kickoff` 等待首次触球阶段，射门探针未触发正式进球回放。

---

## 二、二次核查发现的隐藏逻辑漏洞与本次修正

在逐行审计 `af528a8` 改动及上下游调用链后，发现了 **5 个原报告未提及的实质性漏洞**：

### 1. [P0] `HostSessionController._createPeerChannel` 预热通道 `peer_1` 闭包捕获过期 `peerId`
- **根因**：`HostSessionController.initializeChannel()` 会先创建一个 `peerId = 'peer_1'` 的预热通道。当首位真实 Client 连入时（`HostSessionController.ts` 第 272-288 行），代码把 `prewarmed.peerId` 改写为真实 `peerId`（如 `peer_abc123`），并从 `peerChannels` 删除 `'peer_1'`、写入 `'peer_abc123'`。但 `_createPeerChannel` 内部的 `onConnected`、`onDisconnected`、`onIceCandidate`、`onRemoteTunnelEndpoint` 四个回调仍通过闭包捕获了旧的形参 `peerId`（`'peer_1'`）。
- **后果**：
  - `MultiplayerSessionManager` 收到 `onRemoteTunnelEndpoint(endpoint, 'peer_1')` 时，`hostController.peerChannels.get('peer_1')` 返回 `undefined`，**首位真实 Client 的 Rust UDP Tunnel 预连接被静默跳过！**
  - `onIceCandidate`、`onConnected`、`onDisconnected` 均传入错误 `peerId`，且导致 `peerReadyStates` / `peerPairingCodes` Map 泄漏。
- **修正**：在 `src/network/transport/HostSessionController.ts` 的上述 4 个回调中统一读取 `const activePeerId = ch.peerId || peerId;`。

### 2. [P0] `ROOM_ACTIVITY_SWITCH` 缺少 Epoch 防乱序、迟到 Client 补发、以及 Host 本地状态同步
- **根因**：
  - `HostSessionController.broadcastRoomActivitySwitch` 将 `epoch` 硬编码为 `0`，无法防止网络乱序/重复包覆盖最新活动。
  - 若 Host 已切换到 `multiplayer_training` 或 `replay`，**后加入的 Client（Late Joiner）** 不会收到当前房内活动，仍停留在 `freeplay`。
  - `OnlineMatchControlBridge.switchRoomActivity` 只通知了 Worker 与远端 Client，**未同步更新 Host 自身主线程**的 `networkReconciler.setReplayPlaybackMode` 与 `useAppStore.onlineSession.activeRoomActivity`。
- **修正**：
  - `HostSessionController.ts` 增加单调递增 `_activityEpoch` 与 `activeRoomActivity` 状态，并在 `ch.onConnected` 时对非 `freeplay` 活动自动向新连入 Peer 补发一次 `sendRoomActivitySwitch`。
  - `PredictionReconciler.ts` 增加 `lastRoomActivityEpoch`，丢弃 `epoch > 0 && epoch < lastRoomActivityEpoch` 的过期切换包。
  - `OnlineMatchControlBridge.ts` 在 `switchRoomActivity` 时同步更新 Host 本地的 `setReplayPlaybackMode` 与 `useAppStore.getState().setOnlineSession({ activeRoomActivity })`。

### 3. [P1] `DiagnosticReplayBufferBridge.getTransportDiagnostics()` 在 Host 端读取了本地回环通道
- **根因**：Host 端的 `sessionCoordinator.networkChannel` 是连接本机 Worker 的 Loopback 通道，而非连接远端 Client 的 `P2PWebRTCChannel`，导致 Host 导出的 `diagnostic-manifest.json` 中 `transportPath` 恒为 `'disconnected'`、`iceConnectionState` 为 `null`。
- **此外发现**：`af528a8` 在 `src/ui-system/core/diagnosticDumpExportPipeline.ts` 新增的 `notifyTransportDiagnosticDumpReady` 在全仓没有任何调用方（实际断连通知已由 `RuntimePostprocessingAndSessionBootstrap.ts` 的 `notifyNetworkTelemetryExportAvailable('disconnected')` 承担）。
- **修正**：更新 `src/game/runtime/DiagnosticReplayBufferBridge.ts`，当 `sessionCoordinator.networkChannel` 无 `transport` 时自动回退读取 `domain.multiplayerSession.hostController.peerChannels` 的首个远端通道诊断信息。

### 4. [P1] `pacingControllerEvaluator.ts` 无扩展名导入破坏 Node ESM 测试
- **根因**：`src/ui-core/config/pacingControllerEvaluator.ts` 第 15 行使用了 `from "./pacingControllerStepMath"`（缺少 `.ts` 后缀），导致 Node v22 `--experimental-strip-types` 在测试中导入 `PredictionReconciler.ts` 时抛出 `ERR_MODULE_NOT_FOUND`。
- **修正**：改为 `from './pacingControllerStepMath.ts'`。

### 5. [P1] 真实浏览器 Repro 同步屏障过短与硬断言缺失
- **根因**：
  - `multiplayerRoomReproClientFlowPipeline.ts` 等待 `hostStartingGoalFreezeProbe` 仅 2 秒、等待 `hostProbesComplete` 仅 4.5 秒，而 Host 端 `PhysicsRateSweepDiag` 比 Client 慢约 25 秒，导致 Client 提前结束并断开连接（`client_final_state` 出现 `peers=0`）。
  - `scripts/runMultiplayerRoomRepro.mjs` 未对 `stuckInGoal`、`poseAnomalies`、`trajectoryCollapsedBelow30Ticks` 做失败断言。
- **修正**：将 Client 屏障等待上限提升至 30 秒，并在 `runMultiplayerRoomRepro.mjs` 补齐上述硬断言。

---

## 三、测试用例规范性审查与补齐

### 1. 原测试套件存在的问题
- **Mock 接口失真**：`runMultiplayerSuiteTests.ts` 与 `runMultiplayerChaosRepro.ts` 中的 `loopbackTunnel` 使用了已废弃字段 `mode: 'rust_udp'`、`localEndpoint`，未完整实现 `TunnelTransport` 契约（`kind: 'rust-upnp-udp'`）。
- **覆盖盲区**：缺少对 `webrtc-first` 降级策略、`HostSessionController` 预热通道重命名、`ROOM_ACTIVITY_SWITCH` 真实线缆编码与 Epoch 乱序、`ReliableTunnelFramer` 乱序分片+多帧丢包重传的验证；`src-tauri` 下 Rust 单元测试数量为 `0`。

### 2. 本次补齐的自动化测试
- **`scripts/runMultiplayerSuiteTests.ts` 新增 4 项核心测试（总计 10 项全绿）**：
  - `7. WebRTC-First & Rust UDP Tunnel Automatic Fallback`：验证 DataChannel `open` 时优先走 WebRTC（Tunnel 发送计数为 0），关闭后自动降级至 `rust-upnp-udp` 继续投递。
  - `8. Host Prewarmed Peer Rename & Tunnel Endpoint Routing`：复现 `peer_1 -> peer_real_42` 重命名场景，验证 `onRemoteTunnelEndpoint` 与 `onDisconnected` 精确路由至 `peer_real_42` 且无 Map 泄漏。
  - `9. ROOM_ACTIVITY_SWITCH Wire Delivery & Epoch Ordering`：打通 `HostSessionController -> DataChannelTransport -> PredictionReconciler.reconcile()` 全链路，并验证过期 `epoch: 1` 在 `epoch: 2` 之后被正确丢弃。
  - `10. ReliableTunnelFramer Multi-Chunk Out-of-Order & Loss Recovery`：对 `>3000B` 载荷分片进行首帧丢弃 + 其余帧逆序投递，验证重传定时器恢复后完整重组。
- **`src-tauri/src/tunnel_manager.rs` 新增 2 项 Rust 单元测试（`cargo test` 2/2 PASS）**：
  - `peer_routing_table_multi_peer_upsert_and_remove`：验证多 Peer 路由表插入、端口变更旧地址驱逐与移除。
  - `bridge_frame_roundtrip_and_udp_loopback`：验证本地桥接帧编解码与 `127.0.0.1` UDP `RCPH` 魔数回环收发。

---

## 四、用户可能触发但原报告未覆盖的风险场景（Real Device Test 重点核查）

1. **Client 在 Host 已切换活动（如 `multiplayer_training` / `replay`）后才加入房间**：本次已在服务端与信令层补齐 `activeRoomActivity` 自动补发，真机需验证 Client 场景加载完成后不会因模式切换时序竞争被重置回 `freeplay`。
2. **对称 NAT 下 WebRTC ICE 从 `checking` 超时变为 `failed` 的瞬时切换**：需真机跨公网验证 `DataChannelTransport` 降级到 `rust-upnp-udp` 期间，控制帧与状态帧不发生双端状态撕裂。
3. **Host 踢人 / Client 主动退出后同一 Peer 或新 Peer 秒级重连**：需真机验证 `PeerRoutingTable` 与 `SlotAllocator` 槽位复用无残留视角/车辆控制权错乱。
4. **Developer UI 切换活动面板**：目前 `src/ui-*` 尚未暴露 `switchRoomActivity` 按钮，真机测试需先通过控制台/调试桥触发或补齐 UI 控件。

---

## 五、关于「云端沙盒未预装 gcc / GTK 系统库」的澄清与实测验证

原报告将沙盒内 `cargo check` 退出归因于“云端沙盒未预装 `gcc/cc` 及 GTK3/WebKitGTK 桌面系统库，只能回本机或完整 CI 容器执行”。

**实际核查结论：该限制并非沙盒不可突破的环境缺陷，而是未执行系统包安装。**
本沙盒为 Ubuntu 24.04 x86_64 环境且网络策略允许访问 Ubuntu apt 镜像源。本次实测通过以下命令直接完成安装：
```bash
DEBIAN_FRONTEND=noninteractive apt-get update -qq
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  gcc libc6-dev pkg-config libgtk-3-dev libwebkit2gtk-4.1-dev \
  libayatana-appindicator3-dev librsvg2-dev
```
安装完成后，在 `/work/RocketCompanion/src-tauri` 内实测：
- `cargo check`：**EXIT=0**（完整编译通过 `wtransport`、`igd-next`、`tauri`、`webkit2gtk` 等全部依赖）
- `cargo test`：**EXIT=0**（`2 passed; 0 failed`）

> 唯一不能在无桌面路由器环境的云端沙盒内验证的，是**真实家用路由器 UPnP IGD 映射（`igd-next`）**与**跨公网对称 NAT UDP 打洞成功率**，这两项仍必须上 Real Device Test。
