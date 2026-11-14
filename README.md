# RocketCompanion 联机与 Real Device Test 差距评估报告

- **基准 Commit**: `3f4bd93557110c2a8d678de75f56f32fe06b3b29`
- **工作分支**: `z/agy/rc/multiplayer-diagnostics-audit`
- **本次 Verified Bot Commit**: `dfbcf57340873c68cfe6e36f74e1efe79aacb486` (`signature.isValid: true, state: VALID`)

---

## 一、核心结论：距离 Real Device Test 还差多少？

**结论：`3f4bd93` 尚未达到可以直接上 Real Device Test 的状态。**
虽然代码库构建与架构门禁全绿、Node 层的 Mock 单元测试全过，但在**真实多标签页 Headless Chrome 实测**和**跨节点信令/活动同步链路**上仍存在 3 个 P0 级断层：

1. **真实浏览器联机 Repro（`repro:multiplayer-room`）实测 Client 未连上**：Host 建房后拉起 Client 标签页，但 Client 全程停留在 `mode=freeplay`、`pings=0/0`、`bw=0kbps`、`stuckInGoal=true`，脚本未报错退出而是一直空跑到 Watchdog 超时。
2. **`tunnelEndpoint` 尚未写入实际信令 Token 交换流程**：在对称 NAT / 防火墙导致 WebRTC ICE 失败的场景下，Rust UDP Tunnel 仍然无法提前拿到对端端点建立直连。
3. **动态房间活动切换（`ROOM_ACTIVITY_SWITCH`）只接通了半条链**：Host 端 Worker 支持 `match / freeplay / multiplayer_training / replay / sandbox` 切换，但 **Host 未向远端 Client 广播，Client 也未消费该事件**。

---

## 二、问题逐项解答（概念澄清与现状核查）

### 1. 关于 `cargo check` 报错与 Tauri 测试可行性
- **为什么沙盒里 `cargo check` 退出？**
  当前云端沙盒（Ubuntu 24.04 x86_64）未预装 `gcc`/`cc` 链接器以及 GTK3/WebKitGTK 桌面系统库，因此 Rust 在编译 `proc-macro` / `build.rs` 阶段因找不到 `cc` 退出。**这不代表 Tauri 不能在 Linux CLI 下测试，也不代表 Rust 源码有语法错误**；只要在有完整工具链的本机或带 `gcc` + WebKitGTK 的 Linux CI 容器中即可正常执行 `cargo check` / `pnpm tauri build`。
- **Tauri UDP Tunnel 的定位（为什么应当是 WebRTC-First 自动降级）**：
  `src-tauri/src/tunnel_manager.rs` 与 `tunnel_bridge.rs` 的职责是通过 `igd-next`（UPnP 端口映射）+ 原生 UDP Socket（`RCPH` 魔数打洞）+ 本地 WebSocket/WebTransport 桥接给前端提供传输通道。
  - 当 **WebRTC DataChannel 正常连通时**，直接走 WebRTC 性能最高（无需经过 Rust ↔ TS 本地桥接与额外帧封装）。
  - 当 **WebRTC 因对称 NAT / 防火墙打洞失败时**，才应自动降级到 Rust UDP Tunnel。
  - **本次已在 `DataChannelTransport.ts` 与 `tunnelFramerBridge.ts` 中将其修正为 `webrtc-first` 默认策略**（WebRTC DataChannel `open` 时优先走 WebRTC，未 `open` 时自动降级到 Rust UDP Tunnel）。
- **Tunnel 能不能本机测？怎么 Debug？**
  - **本机能测的部分**：本机 `127.0.0.1` 回环可以验证本地 WebSocket/WebTransport 桥接、`PeerRoutingTable` 和 `ReliableTunnelFramer` 的分片/ACK/丢包重传逻辑。
  - **本机测不出来的部分**：真实路由器 UPnP IGD 端口映射（`igd-next`）以及跨公网/对称 NAT 的 UDP 打洞成功率，**必须上真实设备（Real Device）才能验证**。
  - **Debug 与诊断导出机制现状**：
    - `src/ui-system/core/diagnosticDumpExportPipeline.ts` 的 `exportUnifiedDiagnosticDumpArchive()` 已经实现了打包导出 `rc-diagnostic-dump-*.zip`，包含：
      - `diagnostic-manifest.json`（会话元数据与缓冲统计）
      - `client-presentation-*.rcreplay`（所有 Client 及 Host 主线程的客户端表现层回放）
      - `reconcile-pacing-metadata.json` + `reconcile-pacing-telemetry.csv`（网络对账与 Pacing 遥测）
      - `server-authoritative-*.rcreplay`（**仅 Host 端额外包含**权威服务端回放）
    - **现状缺口**：`diagnostic-manifest.json` **目前尚未记录传输层诊断信息**（当前走的是 `webrtc` 还是 `rust-upnp-udp`、UPnP 是否映射成功、`ReliableTunnelFramer` 的重传/丢包计数），也没有在 Tunnel 降级或断连时自动触发类似 Replay 下载的通知提示。

### 2. Developer 能否创建任意房间、动态切换活动（Training / Replay Watch Together）？
- **底层基础设施已具备**：
  - `src/network/server/RoomActivityOrchestrator.ts` 与 `src/network/server/activities/ActivityKernel.ts` 已经实现了 5 种活动内核：`match`、`freeplay`、`multiplayer_training`、`replay`、`sandbox`。
  - `HostWorkerMessageRouter.ts`（第 401-478 行）已处理 `MSG.ROOM_ACTIVITY_SWITCH`、`MSG.REPLICATION_MODE_SET`、`MSG.STATE_PAUSED_ACK`、`MSG.ROOM_CSR_STATE_BROADCAST`。
- **尚未打通的断层（无法真正用于多端联动）**：
  1. `OnlineMatchControlBridge.ts`（第 130 行）仅在 `hostOnlineWarmup` 时向本地 Host Worker 发了 `switchActivity({ activityId: 'multiplayer_training' })`。
  2. 全仓搜索 `drainTransportEvents`、`sendRoomActivitySwitch`、`room_activity_switch`：**`DataChannelTransport.drainTransportEvents()` 在全仓没有任何调用方**；`HostSessionController` 没有把 Worker 返回的 `ROOM_ACTIVITY_SWITCH` 广播给 Client；Client 也没有消费 `room_activity_switch` 事件去切换本地状态机/模式。
  3. UI 层（`Layer2ManageRoomRecipe.tsx`、`Layer3HostTrainingRecipe.tsx` 等）尚未提供 Developer 任意切换房内活动（如 Match ↔ Training Pack ↔ Replay Watch Together）的控制面板。

### 3. 测试脚本覆盖面与“静默兜底”问题排查
- **`test:multiplayer-suite` 与 `repro:multiplayer-chaos` 的局限性**：
  `scripts/runMultiplayerChaosRepro.ts`（~100ms 跑完）虽然遍历了“3 轮取消/重开房”、“6 种 Activity 路径”、“18 种 1v1~3v3 人机组合”，但它**完全运行在 Node 进程内，使用 `createPairedLoopbackTunnels()` 内存 Mock 对象**，不经过真实浏览器、Web Worker、`MultiplayerSessionManager`、`BroadcastChannel` 或 WebRTC。因此它无法暴露真实浏览器中的建链失败、状态残留或画面 Glitch。
- **真实 Headless Chrome `repro:multiplayer-room` 实测发现**：
  执行 `pnpm run repro:multiplayer-room -- --mode=2p --max-duration-ms=90000 --chaos --glitch-check`：
  - Host 输出：`visibleCars=[0,1] clientCar1Vis=true nameTags=[#0:HostRepro, #1:Player2727]`，Host 侧认为 Client 已加入并开始 Match。
  - **Client 实测输出**：
    - `mode=freeplay phase=goal`（**未跟随进入 `match` 模式**）
    - `visibleCars=[0] clientCar1Vis=false`
    - `dualWorld=false stuckInGoal=true`（**球卡在球门内未重置**）
    - `pings=0/0 bw=0kbps`（**Client 数据通道未真正收到 Host 包**）
    - 最终 90s Watchdog 超时退出，且报告中 `hostSummary: null, clientSummary: null`。
  - **静默吞错根因之一**：`src/game/scenario/multiplayerRoomReproClientFlowPipeline.ts` 第 84 行对 `joinOnlineWarmup` 使用了 `.catch(() => {})`，导致建链异常被完全吞掉，控制台无任何报错。

---

## 三、本次已完成的补齐与验证

在分支 `z/agy/rc/multiplayer-diagnostics-audit` 提交了 Verified Commit `dfbcf57340873c68cfe6e36f74e1efe79aacb486`：

1. **传输优先级改为 `webrtc-first` 自动降级（`DataChannelTransport.ts` & `tunnelFramerBridge.ts`）**：
   - 当 WebRTC `dataChannel` / `controlDataChannel` 处于 `open` 状态时，直接走 WebRTC 零 IPC 开销传输；仅当 WebRTC 未打开时自动降级到 `tunnelBridge`（Rust UDP Tunnel）。
   - 在 `tunnelFramerBridge.ts` 新增 `resolveActiveTransportPath()` 辅助函数，返回 `'webrtc' | 'rust-upnp-udp' | 'crosstab' | 'disconnected'`，同时保持 `DataChannelTransport.ts` 在 Radar 18KB 硬限制内。
2. **信令 Token 类型预留隧道端点（`SignalTokenCodec.ts`）**：
   - 在 `DecodedSignalTokenPayload` 中补充 `tunnelEndpoint?: Record<string, unknown> | null`。
3. **移除 Client Repro 静默吞错（`multiplayerRoomReproClientFlowPipeline.ts`）**：
   - 移除 `joinOnlineWarmup(...).catch(() => {})`，改为在返回 `false` 或 reject 时通过 `console.error` 与 `[ReproStream]` 明确输出错误。

### 验证结果
- `pnpm run check`（Radar 1407/1407 TS + `tsc --noEmit`）：**PASS**
- `pnpm run test:multiplayer-suite`（6 项矩阵）：**PASS**
- `pnpm run repro:multiplayer-chaos`（3 项混沌流）：**PASS**
- `pnpm run build`（Vite 生产构建）：**PASS**

---

## 四、后续推进路线图（建议按优先级执行）

### P0：进入 Real Device Test 前必须打通
1. **修复 `repro:multiplayer-room` 的 Client 建链与 Fail-Fast 断言**：
   - 排查为何 Host `applyHostCrossTabSignalingBridge` 单向注入了 `cross_tab_join`，而 Client 端 `pings=0/0`、`mode=freeplay`。
   - 在 `runMultiplayerRoomRepro.mjs` 中增加硬断言：Client 必须在 15s 内收到 Host 包且进入 `match` 模式，否则立即抛出失败并打印双向握手状态，避免空跑到 Watchdog 超时。
2. **完成 `tunnelEndpoint` 在 `MultiplayerSessionManager` 的信令打包与自动建连**：
   - 在生成 Offer/Answer Token 时写入 `localTunnelDescriptor`，接收端解码后若发现 `tunnelEndpoint` 即提前调用 `sessionTunnel.connectPeer()`，不依赖 WebRTC DataChannel 先连通。
3. **打通 `ROOM_ACTIVITY_SWITCH` 端到端广播与消费**：
   - Host Worker 切换活动后，通过 `HostSessionController` 广播 `sendRoomActivitySwitch` 给所有 Peer；Client 消费 `DataChannelTransport.drainTransportEvents()` 中的 `room_activity_switch` 事件并同步切换本地模式/状态。

### P1：实机 Alpha 排障与真实混沌覆盖
4. **增强 `diagnosticDumpExportPipeline.ts` 诊断清单与通知**：
   - 将 `resolveActiveTransportPath()`、UPnP 映射状态、ICE 状态及 `ReliableTunnelFramer` 重传/丢包统计写入 `diagnostic-manifest.json`。
   - 当发生 Tunnel 降级、高重传或异常断连时，通过 `useQuickNotificationStore` 弹出带一键导出诊断包入口的通知。
5. **将混沌路径遍历接入真实 Headless Chrome（`runMultiplayerRoomRepro.mjs`）**：
   - 补充真实浏览器内的“Cancel → Re-host → Re-join”、“Freeplay → Training → Lobby → Replay → Match”多轮重入脚本，并开启 `--glitch-check` 采集 WebGL/JS 异常。

### P2：模块拆分与命名健康度提醒
- `src/network/transport/webrtc/DataChannelTransport.ts` 当前体积为 `18.0 KB / 535 行`，已紧贴 Radar `18.0 KB` 硬上限。后续若继续扩展传输监控或通道策略，建议将“模拟注入/Ping-Pong 探测”进一步拆出到独立 pipeline，避免触发架构门禁拒绝构建。
