# RocketCompanion 全仓代码审查、Tauri P2P Tunnel 评估与下一阶段精准开发规划

> 基准分支：`origin/alpha` (`8d19408`) | 工作分支：`z/agy/rc/next-phase-prep`
> 审查范围：`src/` (1,387 TS files)、`src-tauri/`、`scripts/`、关联未合入分支 (`z/agy/rc/track-b-voice-fix`、`z/agy/rc/tauri-tunnel-settings`)

---

## 一、核心结论：是否“直接开工”？

**结论：建议先确认下方第 4 节的关键细节后再开工（暂不直接盲干）。**
原因如下：
1. **Tauri P2P Tunnel (`z/agy/rc/tauri-tunnel-settings`) 仅完成了抽象层与 Rust UDP/UPnP 原型，尚未接入主网络链路**，且存在可靠传输语义缺失、单 Peer 绑定、IPC JSON 序列化开销及 WebRTC 语音轨兼容 4 个核心架构问题需要拍板。
2. **Spectating（全局状态同步 vs 仅画面同步）与 Multiplayer Training（多球 Ghost Relay、滞空消耗回充细节）** 仍有若干行为边界需要您确认，避免实现后返工。
3. **代码库本身非常健康，无需大规模推倒重构**。

---

## 二、全仓深度审查报告（“大扫除”评估）

### 1. 架构与指标健康度（`pnpm check` / `pnpm radar:audit` 实测）
- **TypeScript 覆盖率**：`1387 / 1387`（100% TS，0 引擎 JS 残留），`tsc --noEmit` 零报错。
- **边界与耦合**：`0` 循环依赖、`0` 值重导出、`0` 向上越层导入（`ui-core` 未直接依赖 `game`/`network`/`physics`/`audio` 运行时，仅 2 处 `import type` 引用 `telestratorTypes`）。
- **文件体积**：全仓 `.ts`/`.tsx` 最大单文件仅 ~557 行（远低于 800 行硬红线），**不存在对 Agent 不友好的超大文件**。
- **混淆检查**：全仓 **0 残留混淆**（`atob(` 仅出现在 4 个合法 Base64 编解码器中）。

### 2. UI 架构验证（确认您的理解完全正确）
您的判断与代码实现 **100% 一致**：
- `src/ui-core/primitives/` 与 `src/ui-core/compounds/` 已完整封装 `SettingSegmentedRow`、`SettingSliderRow`、`SettingToggleRow`、`SettingActionRow`、`SettingPresetCustomRow`、`ConfiguratorAdvancedView` 等标准组件。
- 业务面板只需将 Zustand Store（`src/ui-core/stores/*`）的数据字段绑定到上述标准 Row，并通过 `src/ui-system/core/runtimeEngineUiPort.ts` 桥接到引擎。**后续 UI 团队美化视觉样式时，无需改动数据绑定与业务逻辑，完全符合“指哪打哪”的低耦合要求。**

### 3. 发现的命名不一致、历史包袱与可清理项
1. **组件命名不统一**：
   - `src/ui-core/panels/dev/TelestratorSettingsPanel.tsx`：同目录下 35 个浮窗均以 `*FloatingWindow.tsx` 命名，唯独它是 `*Panel.tsx`。
   - `Layer3TrainingConfigRecipe.tsx`（通用 Training 配置）与 `Layer3HostTrainingRecipe.tsx`（联机 Host Training）语义易混淆；`Layer3CustomPlayRecipe.tsx` 目前仍为 `In Development` 占位。
2. **分支合并风险（关键）**：
   - `z/agy/rc/track-b-voice-fix`（`3e7cf81`）基于较早的 `alpha`（PR #11 时期）切出，**严禁直接 `git merge`**，否则会回退 PR #14（`ReplicationBus`）、PR #18（`DynamicTraining`）、PR #19（`unify-extensions`）。必须按文件精准移植语音模块并接入 `EstablishedSessionManager` / `RoomReplicationBus`。
   - 早期废弃分叉（如 `1051b10` 及已合入/过期实验分支）可直接清理。
3. **轻微耦合关注点**：
   - `pnpm radar --audit` 仅标记 5 个目标，其中 `src/game/subsystems/TelestratorOverlaySubsystem.ts`（481L，6 个回调）存在轻微回调乒乓；其余 4 个（`vehicleBoostEmitter`、`PredictionReconciler`、`SignalEventBuffer`、`exhibitionMultiBotsLifecycleScenario`）均属可控范围，无需过度拆分。
4. **格式后缀清理状态**：
   - `alpha` 最新提交（`bc1e9a2`）已完成 `.RCReplay`、`.RLRCReplay`、`.RCGR`、`.RCDPack`、`.RCSPack`（含 `.RCSPackE` / `.RLRCSPack`）去 `.gz` 后缀统一；`RCClip` 作为 `RCDPack` 内部切片结构保持内聚。

---

## 三、Tauri P2P Tunnel (`z/agy/rc/tauri-tunnel-settings`) 专项检查

**做得好的部分**：
- 抽象清晰：`TunnelTransport` 接口、`TunnelFactory`（`isTauriRuntime()` 自动切换 `RustTunnelTransport` / `WebTunnelTransport`）、`TunnelProtocol.ts`（8-byte `0x5243` 魔数信封）与 `src-tauri/src/tunnel_manager.rs`（`igd-next` UPnP IGD + UDP Socket）职责单一、解耦良好。

**尚未完成 / 存在隐患的部分（需注意）**：
1. **未接入主链路**：`MultiplayerSessionManager.ts` 仅新增了 `createSessionTunnel` 等辅助方法，**没有任何调用方**；`P2PWebRTCChannel` / `RoomReplicationBus` 仍完全走 WebRTC DataChannel。
2. **缺少可靠传输语义**：WebRTC DataChannel 提供可靠有序（`rc-ctrl`）与不可靠（`rc-state`）双通道；`RustTunnelTransport` 目前是裸 UDP，直接承载 `RELIABLE_CONTROL`、`TELESTRATOR_SYNC`、`STATE_PAUSED_ACK` 或大于 MTU（~1200B）的快照会丢包/乱序。
3. **单 Peer 绑定限制**：`tunnel_manager.rs` 中每个 `session_id` 仅存单个 `peer_addr: Option<SocketAddr>`，且 `tauri_tunnel_connect` 发送的 4 字节 `"RCPH"` 打洞包未在 `rx_thread` 过滤，会直接透传给 JS 层。
4. **IPC 开销**：`rx_thread` 通过 `app.emit("rc-tunnel-packet", { data: Vec<u8> })` 逐包发送事件（在 Tauri v2 中 `Vec<u8>` 默认序列化为 JSON `number[]`），60–120Hz 多端同步时建议改用 Tauri `Channel` 二进制流或批量合并。
5. **语音轨兼容**：UDP Tunnel 仅替代 DataChannel，WebRTC 音频 `MediaStreamTrack` 仍需保留 `RTCPeerConnection`（混合模式）或单独封装传输。

---

## 四、精炼版下一阶段开发规划

### 模块 A：Audio / WebRTC 语音整合
1. 从 `z/agy/rc/track-b-voice-fix` 移植 `src/audio/voice/*`、`voiceChatConfig.ts`、`voiceChatStore.ts`、`runtimeVoiceBridge.ts`、`VoiceChatSettingsSection.tsx`。
2. 修复 `voiceAudioGraph.ts`：统一走 `VoiceMasterOutputBus` 播放远端流（不再当增益 `<= 100%` 时将 `gainNode` 设为 `0` 并依赖 `hiddenAudioEl`），移除静默 `.catch(() => {})`，输出结构化诊断日志与 UI 状态提示。
3. 将语音生命周期绑定到 `EstablishedSessionManager` / `RoomReplicationBus`，确保跨 Activity 热切换不掉线。
4. 房间与 Host 配置新增 `voiceChatMode: 'open' | 'team_only' | 'off'`：
   - `open`：全员互通；
   - `team_only`：按 Peer `team`（Blue / Orange）动态 Gate 发送/接收；
   - `off`：禁用麦克风采集与远端播放。
5. **暂不引入** RNNoise WASM 降噪库，先沿用浏览器原生 `noiseSuppression`。

### 模块 B：Multiplayer Training（`MULTIPLAYER_TRAINING` ActivityKernel）
1. 基于 `RoomActivityOrchestrator` + `ActivityKernel` 新增 `MULTIPLAYER_TRAINING` Kernel，支持房间内热切换。
2. 默认规则：无比分结算（隐藏计分板），进球正常进入 Cinematic Goal Replay。
3. Boost 策略：基础配置页提供 Segmented Control（`Limited | Limit When Aerial | Unlimited | Custom`），默认 `limit_when_aerial`（地面保持 100% 且可拾取 Boost Pad，离地 `!car.onGround` 后从当前/100% 真实消耗）。选中 `Custom` 跳转 Advanced 页，**目前仅实现 Per-Player Boost 覆盖**。
4. Ghost 模式：Client 端默认 **OFF**（与 Lobby 默认 ON 相反）；开启 Ghost 时通过 Host Relay 架构实现多球状态分发（Host 接收 1 球并按需广播多球状态）。

### 模块 C：Spectating 升级与全局状态广播包
1. 拆分“对局实时数据包”与“全局状态广播包”（含当前设置页路由、打开/最小化的浮动窗体清单，**不含**窗体屏幕坐标、Notification 与 Telemetry）。
2. 客户端提供二选一观战模式：`spectate_state_only`（仅同步主画面 State / Input / Boost Meter）vs `spectate_all`（同步对方打开的设置菜单与浮动窗体，本地只读锁定；退出该模式后恢复本地控制）。
3. 基于 Replay 同步模型：Play / Pause ACK、暂停期降频心跳（1–10Hz）、Telestrator 双向绘图同步与绘图权限管理。

### 模块 D：Kick & Leave 双向信令修复
1. 修复 `OnlineRemotePeerVehicleSynchronizer.removeRemotePlayer(carIndex)`：清理本地状态前向被踢端发送可靠 `kick` 控制帧并关闭对应 `P2PWebRTCChannel`，被踢端收到后提示并退回大厅。
2. 客户端主动 Leave 时先通过可靠通道发送 `leave` 控制帧，确保 Host 与其余端立即更新 Roster 与语音列表后再销毁连接。

### 模块 E：自动化测试矩阵
1. 扩展 `scripts/runMultiplayerRoomRepro.mjs` 并新建 `MultiClientLoopbackSimulator.ts`（当前仓库尚无此文件），覆盖“真人接管 Bot”与“Auto-fill Bot”组合矩阵。
2. 覆盖用例：Multiplayer Training 4 档 Boost 与 Ghost Relay、Kick/Leave 双向信令、Voice `open/team_only/off` 路由、Telestrator 压测（复用 `TelestratorWireCodec.ts` `stroke.hash` 与 `TelestratorHashAck` 校验多端渲染一致性）。

---

## 五、待确认细节问题（请逐条确认，确保实现零偏差）

1. **Tauri P2P Tunnel 定位**：
   - Tauri 端是否采用 **“混合传输模式”**（数据包走 Rust UDP Tunnel，语音仍走 WebRTC MediaTrack）？还是先将 `tauri-tunnel-settings` 合入后作为可选实验开关，本阶段仍以 WebRTC DataChannel 为主通路？
   - 在 Host-Client 星型拓扑中，Host 端需要同时连接最多 5 个 Client：`tunnel_manager.rs` 需要升级为“单 UDP Socket 按 `from_addr` 路由多 Peer”还是“每个 Peer 独立 `session_id`”？
2. **Multiplayer Training 细节**：
   - `limit_when_aerial` 模式下，车辆在空中消耗 Boost 后**重新落地（`onGround === true`）的瞬间**，Boost 是**立刻恢复 100%**，还是保持剩余量直到自然回充/吃包？
   - 多人开启 Ghost Mode 各自拥有独立球时，Host 端本地渲染哪一个球（只渲染自己的球，还是半透明渲染其他人的球）？Goal Replay 触发时是否只针对各自球或全员同步？
3. **Spectating 细节**：
   - `spectate_all` 模式下，Host 打开浮动窗体但**不共享窗口坐标**时，Client 端弹出的对应浮动窗体应使用 Client 本地预设默认位置还是自动居中？
   - Telestrator 绘图权限默认是 **Host Only（仅 Host 可画，可授权给指定 Client）** 还是 **All Allowed（全员可画）**？
4. **Voice Chat 细节**：
   - `team_only` 模式下，在 **Lobby 阶段（尚未分队或均为 Spectator）** 时，语音路由是全员静音还是允许同组/全员互通？
5. **分支与交付方式**：
   - 确认是否可以清理掉 `z/agy/rc/track-b-voice-fix` 与 `z/agy/rc/tauri-tunnel-settings` 之外的过期历史分支？
   - 确认上述问题后，您希望我们**按模块拆分成多个独立 PR/分支**，还是在当前 `z/agy/rc/next-phase-prep` 分支分步提交？
