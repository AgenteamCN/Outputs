# RocketCompanion `z/agy/rc/multiplayer-recheck` 全面复核与验收报告

## 一、总览结论

**结论：当前分支（`origin/z/agy/rc/multiplayer-recheck` @ `9e889cb`）已具备进入真实设备（Roadwash Test / Real Device Test）的条件。**

本次在无 GUI Linux 沙盒中完成了工具链补齐（安装 `pnpm@12.21.0`、初始化 `public` 静态资源子模块 `RocketCompanionStatics.git` 以提供 `camera.wasm`）、全量代码静态/架构检查、14 项多人单元测试矩阵以及 9 阶段 Headless Chrome 双标签页真实 E2E 验收，全部通过，且 `runtime_error=0`、`poseAnomalies=0`、`stuckInGoal=false`。

---

## 二、环境与工具链补齐情况

1. **包管理器**：沙盒初始缺少 `pnpm`，已全局安装 `pnpm@12.21.0` 并完成 `pnpm install --frozen-lockfile`。
2. **静态资源子模块（关键）**：首次运行 Headless Chrome E2E 时因 `public/assets/camera/camera.wasm` 缺失触发 `ERR_POV_MICROKERNEL_TAMPERED`；通过 GitHub Installation Token 初始化 `public` 子模块（`e66c052`）后，安全预检与完整 3D/WASM 运行时正常启动。

---

## 三、分步实现与细节复核（Double Check）

### Step 1：Pure CSR "Sync All" 房间闭环 & `DevRoomSyncFloatingWindow`
- **入口与窗口规范**：`Layer3DevUnderDevelopmentRecipe.tsx` 已提供 `Create Developer Sync Room` 入口，唤起 `DevRoomSyncFloatingWindow`（符合 `PresetConfig + SURFACE_META + floatingSurfaceRegistry` 规范）。
- **协议与双端门控**：`GlobalWorkspaceProtocol.ts` 与 `spectateWorkspaceStore.ts` 已扩展 `clientInputEnabled`、`followHostWorkspace`、`arenaPreset`、`cameraSync`；`PredictionReconciler.sampleAndPredictInput` 与服务端输入路由实现双端门控。
- **重要修复验证（`9e889cb`）**：
  - Client 进入 Pure CSR 房间后自动退出 `lobbyDualWorldMode`（实测 `lobbyDualWorldMode: false`），避免本地双世界物理覆盖 Host 快照（`ballSyncDist: 0`）。
  - `arenaPreset` 同步已同步更新 Host Worker 物理场地预设。
  - `P2PWebRTCChannel` 对早到 `GlobalWorkspaceBroadcast` 进行缓冲直到回调挂载，防止跨标签页/慢设备首帧竞态丢失；`runtimeSpectateWorkspaceBridge` 保留 `hideFooter` 与预设元数据。

### Step 2：Telestrator (`0x05`) 全链路实时同步
- **传输层与中继**：`DataChannelTransport.ts`、`dataChannelFramingPipeline.ts`、`P2PWebRTCChannel.ts` 已打通 `TUNNEL_PACKET_TYPES.TELESTRATOR_SYNC (0x05)` 收发（同时支持真实 `RTCDataChannel`、UDP Tunnel 与同源 `BroadcastChannel`）。
- **权限与双 Store 联动**：`hostTelestratorRelay.ts` 校验 `host_only` / `everyone` 权限并回传 `STROKE_HASH_ACK`；`useTelestratorStore` 与 `useSpectateWorkspaceStore` 联动，`isCameraSynced` 在 CSR 房间中自动置为 `true`。
- **实测结果**：Host/Client 双向落笔与两段式全清（`triggerClear`）均在 Headless Chrome 3D 画布真实鼠标事件下通过。

### Step 3：Replay Watch Together（Pure CSR + 相机同步）
- **帧注入与同步**：`replayOnlineSyncHelper.ts` 将 `StandaloneReplayViewerController` / `ReplayViewerCausalSyncController` 接入 `authoritativeServer.enterStateOnlyMode(snap, tick)` 与 `GlobalWorkspaceBroadcast`（`playbackPaused`、`pausedAtTick`、`cameraSync`）。
- **相机同步**：`captureCameraSyncFromPort` / `applyCameraSyncToPort` 已接通 `ballCam`、`isFreeCam`、`focusedCarIndex` 与自由视角位姿同步。
- **实测结果**：注入 `Float32Array` 状态快照后，Client 球体坐标精确同步至 `[120, -350, 240]`（`dist: 0`），`pausedAtTick: 420` 同步一致。

### Step 4：Custom Training Pack & Room Match 自动拉入
- **训练包模式**：`MultiplayerTrainingPolicy.ts` 与 `OnlineMatchControlBridge.ts` 支持 `training_shot_reset` 与题目元信息同步，Client 无需本地 `.RCDPack` 文件即可同步重置与训练。
- **Match 自动拉入**：切入 `activityId: 'match'` 时自动将已连接 P2P Peer 分配至蓝/橙队并标记 `ready` / `arenaLoaded`，自动开启 `clientInputEnabled = true`。

### Step 5：复杂逆向工况与 E2E 覆盖面
- **已验证工况（`runDevRoomSyncE2E.mjs` Stage 1–9）**：
  1. Host Dev UI 建房 -> `DevRoomSyncFloatingWindow`
  2. 普通无后缀 Client UI（`join-room`）输入房间号加入
  3. Pure CSR Sync All（`spectate_mirror`、`lobbyDualWorldMode: false`、`arenaPreset` `throwback <-> standard`）
  4. 双端 Input 门控开关（`false` 清零 / `true` 放行 / 复关）
  5. Telestrator `0x05` 双向画画与清屏
  6. Replay Watch Together（真实紧凑快照 `dist: 0`、暂停 tick、相机同步）
  7. Custom Training Pack 与 `Reset Shot`
  8. Room Match 自动拉入（`dist: 0`）
  9. **逆向切换与退房恢复（Match -> Replay -> Freeplay -> Leave Room）**：验证了切回 `Replay`/`Freeplay` 后状态清理与退房后 `localSnapshotBeforeMirror` 无损恢复。

---

## 四、自动化验证结果（基于 `9e889cb`）

| 检查项 | 命令 | 结果 | 关键指标 |
| :--- | :--- | :--- | :--- |
| 架构雷达与 TypeScript | `pnpm run check` | PASS (GREEN) | 1415/1415 TS 文件，0 循环依赖，0 类型错误 |
| 耦合与乒乓审计 | `pnpm run radar:audit` | PASS (GREEN) | 所有指标在硬性阈值内 |
| 多人单元测试矩阵 | `pnpm run test:multiplayer-suite` | PASS (14/14) | Test 1–14 全绿（含 CSR/Telestrator/Replay/Training/Match） |
| Headless Chrome 双端 E2E | `pnpm run test:dev-room-sync-e2e` | PASS (9/9 Stages) | `durationMs=95357`, `runtime_error=0`, `poseAnomalies=0`, `stuckInGoal=false` |

---

## 五、进入 Real Device Test 前的注意事项（供 Roadwash Test 参考）

1. **传输通道差异**：Linux Headless Chrome 同源双标签页主要走 `BroadcastChannel` 中继；真实跨设备测试时将实际走 **Signaling（WebSocket / Supabase Realtime）+ WebRTC `RTCDataChannel`（或 Rust UDP Tunnel 回退）**。代码层 `DataChannelTransport` 与 `dataChannelFramingPipeline` 已对 `0x05` 二进制封包、可靠控制帧与快照分片做好双通道接线，但真实 NAT/ICE 穿透与弱网抖动仍需在真机验证。
2. **后续高频切换工况建议重点观察**：
   - **Replay -> Match -> Lobby/Freeplay -> Replay** 多轮往返切换时，关注 Client 端输入开关（`clientInputEnabled`）是否按预期在进入 Match 时放行、回到 Replay/Freeplay 时自动回落为只读 CSR，以及 Match HUD/计时器是否完全退场。
   - **中途加入（Late Join）**：Client 在 Host 已切换到 `throwback`、已开启 `everyone` 画画或正在播放 Replay 后才加入时，确认 `HostSessionController` 下发的 `activeRoomActivity` 与 `activeGlobalWorkspace` 首包完整对齐。
