# RocketCompanion 全链路 Developer 同步房间（Under Development -> Join Room）收拢规划与 Linux Headless 验证方案

**基准分支/Commit**：`e6bb63788fbf7b2752c740bfc7dce9ffcdcc941a`（`origin/z/agy/rc/multiplayer-recheck`）  
**当前环境实测**：已在 `/workspace/RocketCompanion` 检出基准 commit，完成 `pnpm install`，实测 `pnpm run check`（Radar 1409/1409 TS + `tsc --noEmit`）与 `pnpm run test:multiplayer-suite`（10/10 PASS）均通过。

---

## 一、核心判断：路径还有多远？能否一步到位？

**结论：不能（也不应）在 single-agent single-round 内通过一次 commit 强行"一步到位"，而应在单一集成分支上按 5 个原子步骤串行收拢（每步 1 个独立 commit + 自动化验证）。**

### 1. 为什么不适合并行分支，也不适合单轮一把梭？
- **属于最后收拢阶段**：UI（`ui-core` / `ui-system`）、游戏模式（`game/modes`）、运行时桥接（`game/runtime`）、权威物理 Worker（`network/server`）、传输层（`network/transport`）与客户端预测对账（`network/client`）必须在同一套线缆契约上对齐，拆成并行分支必然产生协议与状态机冲突。
- **跨层改动量与 Radar 护栏**：完整打通涉及 ~15–20 个核心文件，且仓库启用了严格的反耦合与体积护栏（`scripts/radar.ts`：`HardError=800L/18KB`、`0 Circular Deps`、`0 Upward Imports`、`HostPen<=105`、`PassThru<=40`）。分步提交可确保每一步都处于 `pnpm run check` 全绿的可回滚状态。

### 2. 距离"全链路绝对同步"到底还有多远？（现状盘点）

| 能力模块 | 当前代码库真实状态（基于 `e6bb6378` 实查） | 距离目标的差距 |
| :--- | :--- | :--- |
| **普通 Client 无后缀加入** | `Layer3JoinRoomRecipe` 已支持输入 Room Code 后调用 `bridge.onJoinOnlineWarmup`，无需任何 URL 后缀 | **已就绪**（0 改动） |
| **Developer 建房入口与浮窗** | `Layer3DevUnderDevelopmentRecipe` 只有 Three-Wheeler / Throwback / Per-Car Boost；`src/ui-*` 无 `switchRoomActivity` 入口，也无房主同步控制浮窗 | **缺 UI 入口与控制浮窗** |
| **Freeplay / Match 同步** | `AuthoritativeServerHost`（Worker）跑物理 + `SNAPSHOT (0x02)` 广播 + `PredictionReconciler` 对账已完全打通 | **已就绪**（仅缺 Client Input 开关门控） |
| **Client Input 激活/取消门控** | `PredictionReconciler.sampleAndPredictInput` 与 `HostWorkerMessageRouter` 始终接收并应用 Client 输入，无全局/单端输入锁 | **缺前后端门控字段与开关** |
| **Floating Window / UI 同步** | `GlobalWorkspaceProtocol.ts`、`runtimeSpectateWorkspaceBridge.ts`（`applyIncomingWorkspaceBroadcast`）及 Client `ch.onGlobalWorkspaceBroadcast` 接收端**已实现**；但 Host 端从未调用 `sendGlobalWorkspaceBroadcast`，`HostSessionController` 无广播方法 | **缺 Host 端广播触发器** |
| **Drawing / Telestrator 同步** | `TelestratorWireCodec.ts` 与 `ReliableTunnelFramer (0x05)` **已实现**，但只在测试/本地自环调用；`DataChannelTransport`、`P2PWebRTCChannel`、`TelestratorOverlaySubsystem` 未接线 | **缺传输层收发与子系统接线** |
| **Replay Viewing CSR 同步** | `AuthoritativeServerHost.enterStateOnlyMode(snap, cursorTick)` 与 `ReplayActivityKernel` **已存在**；但 ReplayViewer 播放时主线程未将逐帧 pose 注入 Worker 驱动 `SNAPSHOT` 广播 | **缺主线程->Worker 帧注入桥** |
| **Custom Training Pack 同步** | `StaticTrainingPackMode` 仅在 Host 主线程跑本地物理；`MultiplayerTrainingActivityKernel` 仅管理 Boost 与 GhostBall，未同步 `.RCDPack/.RCSPack` 题目切换、重置与发球状态 | **缺 Training Pack 状态/发球同步** |

---

## 二、最终用户交互链路设计（完成后如何使用）

1. **Host（Developer 端）进入路径**：
   - 打开 `Settings -> Developer -> Under Development`（`Layer3DevUnderDevelopmentRecipe`）。
   - 点击新增的 **"Create Developer Sync Room"** 按钮：调用 `bridge.onHostOnlineWarmup({ roomType: 'warmup', ... })` 创建房间、生成标准 Room Code（如 `RL-XXXX`），关闭菜单并自动弹出 **`DevRoomSyncFloatingWindow`**（Developer 同步控制浮窗）。
   - 在 `DevRoomSyncFloatingWindow` 中可随时：
     - 一键切换房间活动：`Freeplay` / `Replay Viewing` / `Custom Training` / `Multiplayer Match`（调用 `matchPort.switchRoomActivity`）；
     - 一键激活/取消 **Client Input**（`clientInputEnabled`）；
     - 切换 **Telestrator Drawing 权限**（`host_only` / `everyone`）；
     - 开关 **Follow-Host Workspace Sync**（同步浮窗/菜单/暂停状态）。
2. **Client（普通用户端，不加任何 URL 后缀）加入路径**：
   - 正常打开应用 -> `Play -> Join Room`（`Layer3JoinRoomRecipe`）-> 输入 Host 的 Room Code -> 加入房间。
   - 自动接收 Host 的 `ROOM_ACTIVITY_SWITCH`、`SNAPSHOT (0x02)`、`GLOBAL_WORKSPACE_BROADCAST`、`TELESTRATOR_SYNC (0x05)`，实现与 Host 端一致的渲染与交互权限状态。

---

## 三、分步收拢实施计划（单分支 5 步走）

建议在单一集成分支（例如基于 `e6bb6378` 拉出 `z/agy/rc/dev-room-sync`）顺序推进以下 5 个 Step，每步 1 个 verified commit：

### Step 1：Developer 建房入口 + `DevRoomSyncFloatingWindow` + Client Input 门控 + Workspace 广播
- **目标**：打通"Under Development 一键建房 -> 浮窗切换活动 -> 激活/取消 Client Input -> 同步 Host 浮窗/菜单状态"的闭环。
- **核心改动**：
  1. `src/ui-core/recipes/Layer3DevUnderDevelopmentRecipe.tsx`：新增 `Create Developer Sync Room` 操作行，调用 `bridge.onHostOnlineWarmup` 并打开 `dev-room-sync` 浮窗。
  2. `src/ui-core/panels/dev/DevRoomSyncFloatingWindow.tsx`（及 `devFloatingPresets.ts` 注册）：提供活动切换按钮、Client Input 开关、Telestrator 权限切换（`host_only` / `everyone`）、Follow-Host Workspace 开关、当前房间码与已连入 Peer 状态展示。
  3. `src/network/protocol/GlobalWorkspaceProtocol.ts` & `src/ui-core/stores/spectateWorkspaceStore.ts`：扩展 `clientInputEnabled: boolean`（默认 `true`），并在 `HostSessionController.ts` / `MultiplayerSessionManager.ts` 新增 `broadcastGlobalWorkspace(payload)`，由 Host 在浮窗/门控状态变更时（及新 Peer `onConnected` 时）下发。
  4. **Client Input 门控双保险**：
     - **Client 侧**（`PredictionReconciler.ts`）：当 `clientInputEnabled === false` 时，`sampleAndPredictInput` 停止发送主动控制输入（或发送零输入帧）并跳过本地预测推进，仅跟随权威 `SNAPSHOT` 渲染。
     - **Host 权威侧**（`AuthoritativeServerHost.ts` / `HostWorkerMessageRouter.ts`）：当 `clientInputEnabled === false` 时，Worker 侧强制将所有非 Host 远端 Peer 的输入清零，防止恶意/延迟残余输入扰动物理。
- **Step 1 验证**：
  - `pnpm run check`
  - `pnpm run test:multiplayer-suite` 新增 Test 11：验证 `clientInputEnabled=false` 时 Client 零输入/权威端拒绝移动，恢复 `true` 后恢复控制；验证 `GlobalWorkspaceBroadcast` 同步 `openFloatingWindows` 与 `clientInputEnabled`。

### Step 2：Telestrator（画画）全链路实时网络同步
- **目标**：Host 与获权 Client 在 3D 场地上的实时画线、草稿预览、撤销、清空在全房间内像素级同步，并受 `telestratorPermission`（`host_only` / `everyone`）约束。
- **核心改动**：
  1. `src/network/transport/webrtc/DataChannelTransport.ts` & `dataChannelFramingPipeline.ts` & `P2PWebRTCChannel.ts`：接入 `TELESTRATOR_SYNC (0x05)` 帧的发送与回调（`sendTelestratorBytes` / `onTelestratorBytes`），复用已有的 `ReliableTunnelFramer` 分片/重传通道。
  2. `src/network/transport/HostSessionController.ts`：Host 收到某个 Client 的 `TELESTRATOR_SYNC` 时，校验 `telestratorPermission === 'everyone'` 后向其余 Peer 转发，并回传 `STROKE_HASH_ACK`。
  3. `src/game/subsystems/TelestratorOverlaySubsystem.ts` & `telestratorOverlayHelpers.ts`：将当前本地自环的 `encodeTelestratorStroke` 改为通过 `multiplayerSession` 广播；远端收到 `decodeTelestratorStroke` 后调用 `TelestratorManager` 渲染，并根据 `spectateWorkspaceStore.telestratorPermission` 控制 Client 端工具栏与绘制拦截。
- **Step 2 验证**：
  - `pnpm run test:multiplayer-suite` 新增 Test 12：双端 `P2PWebRTCChannel` 实测 `STROKE_DRAFT` + `STROKE_COMMIT` 传输、`host_only` 权限拦截 Client 绘制、`everyone` 权限放行并校验 `computeStrokeHash` 一致。

### Step 3：Replay Viewing（回放观看）Host->Client CSR 绝对同步
- **目标**：Host 在房间内进入 Replay Viewer 播放/暂停/倍速/拖拽时间轴（`.RCReplay` / `.RLRCReplay`）时，所有 Client 自动进入 `replay` CSR 渲染模式，与 Host 画面严格同步。
- **核心改动**：
  1. `src/game/runtime/OnlineMatchControlBridge.ts` & `ReplayViewer` 播放管线：当 Host 处于房间中且进入 Replay Viewer 时，自动触发 `switchRoomActivity({ activityId: 'replay' })`；每帧（或按播放步进）将 Replay 当前帧的球与所有车辆 Transform 打包为标准 `Float32Array` 状态快照，调用 `authoritativeServer.enterStateOnlyMode(snap, replayTick)`。
  2. Worker 侧（`MatchSystemControlSubController.ts` / `AuthoritativeServerHost`）：`state_only` 模式下继续按 60Hz/暂停心跳向所有 Client 广播 `SNAPSHOT (0x02)`，并附带 `GlobalWorkspaceBroadcast` 的 `playbackPaused` / `pausedAtTick`。
  3. Client 侧（`PredictionReconciler.ts` + `onlineSessionEnvironmentPipeline.ts`）：`replayPlaybackMode = true` 时关闭本地物理预测，纯插值渲染 Host 推流过来的 `SNAPSHOT`，实现零撕裂的 Replay CSR。
- **Step 3 验证**：
  - `pnpm run test:multiplayer-suite` 新增 Test 13：Host Worker 调用 `enterStateOnlyMode(snap, tick)` -> `SNAPSHOT` 经 `DataChannelTransport` 抵达 Client `PredictionReconciler` -> 校验 Client 渲染态球/车坐标与 Replay 帧误差 `< 1e-3`，且暂停/拖拽时间轴后同步更新。

### Step 4：Custom Training Pack（自定义训练包）联机同步
- **目标**：Host 加载并运行 `.RCDPack` / `.RCSPack`（`StaticTrainingPackMode`）时，房间内 Client 同步切换至训练模式、同步当前 Shot 的球/车初始摆位、Reset/下一题重置、以及 Boost/Ghost 模式策略。
- **核心改动**：
  1. `src/game/modes/static-training/StaticTrainingPackMode.ts` & `StaticTrainingSimulationController.ts`：当 Host 处于在线房间中时，每次 Shot 加载/重置（Reset Shot / Next Shot）通过 `switchRoomActivity({ activityId: 'multiplayer_training', trainingPackId, stateSummary })` 及 Worker 控制指令将 Shot 初始球/车位姿同步给 `AuthoritativeServerHost`（由 Worker 权威执行物理并广播 `SNAPSHOT`）。
  2. `src/network/server/activities/ActivityKernel.ts`（`MultiplayerTrainingActivityKernel`）：扩展 `onControlMessage('training_shot_reset', payload)`，在 Worker 物理引擎中瞬间重置球与车位姿并广播 `stateSummary`（当前 `shotIndex`、`totalShots`、`packName`）。
  3. Client 侧：收到 `ROOM_ACTIVITY_SWITCH(multiplayer_training)` 后切换至训练 HUD（隐藏记分牌、显示当前 Shot 序号），物理与球/车渲染完全跟随 Host Worker 快照。
- **Step 4 验证**：
  - `pnpm run test:multiplayer-suite` 新增 Test 14：验证 `MultiplayerTrainingActivityKernel` 接收 `training_shot_reset` 后 Worker 球/车位置重置并同步至 Client `PredictionReconciler`。

### Step 5：全链路 Linux Headless Chrome E2E 自动化验收收口
- **目标**：在无物理路由器、无 GUI 的 Linux 沙盒下，用真实多标签页 Headless Chrome 验证"Developer UI 建房 -> 普通无后缀 Client UI 加房 -> 四大模式切换 -> Input 门控 -> Telestrator 画画"全链路。
- **核心改动**：
  1. 扩展 `scripts/runMultiplayerRoomRepro.mjs`（或新增 `scripts/runDevRoomSyncE2E.mjs`）：
     - Tab 1（Host）：正常加载 `/` -> 自动点击/驱动 `Settings -> Developer -> Under Development -> Create Developer Sync Room`，记录 Room Code。
     - Tab 2（Normal Client，**不带任何 URL 后缀**）：正常加载 `/` -> 自动点击 `Play -> Join Room` -> 输入 Room Code 加入房间。
     - 依次在 Host 端执行：
       1. Freeplay 默认同步与双向移动验证；
       2. `DevRoomSyncFloatingWindow` 关闭 `clientInputEnabled` -> 验证 Client 按键后车辆不位移 -> 重新打开后恢复；
       3. Telestrator 切为 `everyone` -> Client 与 Host 各画一笔 -> 验证双端 `TelestratorManager.getStrokeCount()` 与哈希一致；
       4. Host 切换 `Replay Viewing`（注入测试帧）-> 验证 Client `activeRoomActivity === 'replay'` 且球/车位姿跟随；
       5. Host 切换 `Custom Training`（加载内置 Pack 并触发 Reset Shot）-> 验证 Client `activeRoomActivity === 'multiplayer_training'` 且位姿同步重置。
     - 硬断言：`runtime_error === 0`、`poseAnomalies === 0`、`stuckInGoal === false`、各阶段同步延迟与状态哈希全部 PASS。

---

## 四、Linux Headless 环境下的测试与验证矩阵

当前沙盒环境已具备完整验证能力（Node `v22.23.5`、pnpm `10.28.2`、`/usr/bin/google-chrome`；若需跑 Rust 端检查可按原报告方式通过 `apt-get` 安装 `gcc` 与 GTK/WebKitGTK 依赖）。

每完成一个 Step，按以下顺序执行全量门禁：

```bash
# 1. 代码格式与架构静态门禁（100% TS、0 循环依赖、0 越级导入、体积护栏）
pnpm run format:check
pnpm run check
pnpm run radar:audit

# 2. 生产构建验证
pnpm run build

# 3. Node.js 协议/Worker/Reconciler 联机集成套件（覆盖 Step 1~4 新增用例）
pnpm run test:multiplayer-suite
pnpm run repro:multiplayer-chaos

# 4. Linux Headless Chrome 真实多标签页 E2E（Vite + 内置 Signaling Bridge）
pnpm run repro:multiplayer-room -- --mode=2p --max-duration-ms=180000 --chaos --glitch-check

# 5. Rust / Tauri 隧道层验证（src-tauri 如有改动时执行）
cd src-tauri && cargo check && cargo test
```

### 唯一无法在 Linux Headless 沙盒覆盖、需留待真机 Alpha 的边界
- 家用路由器真实 UPnP IGD 端口映射（`igd-next`）与跨公网对称 NAT UDP 打洞成功率；其余所有信令、WebRTC DataChannel、Loopback/BroadcastChannel 模拟、Worker 物理权威同步、UI 流程与渲染状态对账均可在 Linux Headless 环境下 100% 自动化跑通并卡点拦截。
