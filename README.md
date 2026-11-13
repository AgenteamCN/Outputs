# RocketCompanion 架构推进、Track D/B/A 审查修复与后续分支规划报告

## 一、 Track Delta (`z/agy/rc/track-d-replication-bus` @ `cf5cfee`) 审查与 PR 状态

### 1. 你的理解是否准确？
- **结论：业务层判断准确，协议层可补充但不阻塞合入。**
- **业务端到端测试（Freeplay ↔ Match ↔ Replay 热切换）**确实必须等 **Track E**（拆出 `FreeplayKernel` / `MatchKernel` / `SandboxKernel`）与 **Track F**（多人观战与接管）就位后才能真正跑通；在只有 Track D 的情况下强行测业务热切换，确实只能自己打自己。
- **已执行操作**：已为 Track D 创建 Pull Request **[#14](https://github.com/Antigravi-ty/RocketCompanion/pull/14)**（`z/agy/rc/track-d-replication-bus` → `alpha`）。经 `git merge-tree` 验证与 `origin/alpha`（`ae6c593`）**0 冲突**，可直接合入主干以解锁 Track E / F 并行开工。

### 2. Track D 额外可做的轻量验证 / 顺手修复（非阻塞）
如果你希望在合入前或合入后的第一个 Track E/F 提交中补齐协议层信心，仅需关注以下 2 点（不需要等业务内核）：
1. **`HostWorkerMessageRouter.ts` 状态摘要补全**：`buildWorkerRoomStateSummary()` 目前尚未填充 `activeActivityId`、`activityEpoch`、`replicationMode` 三个字段，合入后应在 Track E 首包顺手补齐。
2. **纯协议环回烟雾验证（可选）**：
   - 验证双 DataChannel（`game_net` + `game_ctrl`）建链后 `transport.isControlChannelOpen === true`。
   - 验证 `sendRoomActivitySwitch` / `sendStatePausedAck` 与 `Uint8Array(16)` 的 `SparsePingPong16B` 编解码往返。

---

## 二、 Track Bravo (`z/agy/rc/track-b-voice-configurator` @ `a44c14d`) 修复报告

针对 Lobby 期间“本地电平正常但双方互相听不见”的问题，已在新分支 **`z/agy/rc/track-b-voice-fix`**（Verified Bot Commit `3e7cf81`）完成根因修复并验证通过。

### 修复内容清单
1. **消除 `onVoiceSignal` 注册时序竞态与信号静默丢弃**
   - `src/network/transport/P2PWebRTCChannel.ts`：为 `onVoiceSignal` 增加 `getter/setter` 与 `pendingVoiceSignals` 缓冲队列（上限 64 条）。在语音桥接注册前到达的 `voice_identity` / `voice_offer` / `voice_answer` 会暂存并在挂载瞬间立即回放，彻底杜绝早期 Offer 被静默丢弃导致的 Perfect Negotiation 死锁。
   - `src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`：将 `onJoinServer` 与 `onPeerConnected` 中的 `attachVoiceToPeerChannel(...)` 提前到 `await joinOnlineServer` / `await onRemotePlayerConnected` 之前执行，避免异步加载 3D Lobby 场景期间漏接信令。
2. **修复非 `stable` 状态下重协商被静默跳过**
   - `src/audio/voice/WebRtcNativeVoiceBridge.ts`：新增 `pendingRenegotiation` 标志位。当 `makingOffer === true` 或 `pc.signalingState !== 'stable'` 时不再直接 `return` 丢弃，而是在 `renegotiatePeer` 的 `finally` 块及 `handleRenegotiationSignal` 回到 `stable` 后自动补发 Offer。
3. **修复 `AudioContext` 挂起与 WKWebView (macOS Tauri) 远端流静音**
   - 新增 `src/audio/voice/voiceAudioGraph.ts`（同时将 `WebRtcNativeVoiceBridge.ts` 从 560 行/17.8KB 精简至 484 行/15.0KB，远离 Radar 18KB 硬红线）：
     - **0% ~ 100% 音量或 `AudioContext` 未 `running` 时**：直接通过 `<audio>` 元素输出（`muted = false`，`volume = effectiveGain`），同时保留 `MediaStreamAudioSourceNode → AnalyserNode` 驱动 UI 电平条（`gainNode = 0` 避免双声回音）。这彻底绕开了 WKWebView 下 `createMediaStreamSource(remoteWebRtcStream)` 输出全零以及浏览器未手势解锁 `AudioContext` 导致的静音问题。
     - **101% ~ 250% 增益且 `AudioContext` 已 `running` 时**：自动静音 `<audio>` 并切换至 Web Audio `GainNode` + `DynamicsCompressorNode` 限幅器输出。
     - 绑定一次性 `pointerdown` / `keydown` 手势监听，确保挂起的 `AudioContext` 在任意交互后立即 `resume()`。

### 本地验证结果
- `npx prettier --write`（针对全部 4 个变更/新增文件）：**PASS**
- `pnpm run check`（Radar 2.0 + `tsc --noEmit`）：**GREEN**（`1362/1362` TS 文件，0 循环依赖，0 桶文件，0 越层引用）
- `pnpm run build`（Vite 生产构建）：**PASS**

---

## 三、 Track Alpha (`da722e3`) 现状评估：是否需要推翻重做？

### 核心结论：**不需要推翻重来，核心架构合理，仅需补齐网络接线与局部微调。**
已审查 `da722e3` 及其前序提交（`5b1ee5c`、`2f38585`，共 27 个文件）：
1. **已做对的部分（保留）**：
   - **CMF 射线检测**：通过 `cmfCollisionMeshBuilder.ts` + `arenaSurfaceRaycaster.ts` 直接对碰撞网格求交，解决了 GLB 网格射线检测卡顿问题。
   - **几何与屏幕自适应粗细**：`telestratorStrokeGeometry.ts` 与 `telestratorStrokeMath.ts` 实现了 1080p 基准 `widthRefPx`、实线/虚线/视向箭头（View-Ray Arrowhead）、笔画简化重采样及确定性 `computeStrokeHash`。
   - **二进制线协议预留**：`src/network/protocol/TelestratorWireCodec.ts`（343 行）已经写好了 `RCTL` 二进制编解码（72B Header + 每点 8B：`3×Int16` 坐标 + `2×Int8` 八面体法线），并定义了 `STROKE_COMMIT`、`STROKE_HASH_ACK`、`STROKE_UNDO`、`STROKE_CLEAR`、`STROKE_DRAFT`、`HOST_PERMISSION_SYNC`。
2. **尚未接通 / 需后续演进的部分**：
   - **网络同步未接线**：`TelestratorWireCodec.ts` 目前仅用于本地自检，尚未接入 Track D 的 `game_ctrl`（Commit/Undo/Clear/HashAck/Permission）与 `game_net`（Draft 预览）。
   - **Draft 增量优化（后续）**：当前 `STROKE_DRAFT` 每次发送完整点集，正常短笔画（<100 点 ≈ <900B）无压力，若后续出现超长笔画可改为增量点或固定间隔节流。

---

## 四、 `alpha` 合入 Track D 后的新分支并行规划

建议在 PR `#14`（Track D）与 `z/agy/rc/track-b-voice-fix`（实机双端语音验收后）合入 `alpha` 后，基于最新 `alpha` 拉出以下解耦分支并行推进：

1. **`z/agy/rc/track-e-freeplay-match-sandbox`（上层模式内核拆解）**
   - 在 `src/network/server/activities/` 下拆分纯粹的 `FreeplayKernel`（无计分/无限气/FIFO 滚动池/一键重置球）、`MatchKernel` 与 `DevSandboxKernel`。
   - 补全 `HostWorkerMessageRouter.ts` 的 `buildWorkerRoomStateSummary()`（`activeActivityId`、`activityEpoch`、`replicationMode`），打通 `ROOM_ACTIVITY_SWITCH` 热切换与 `repro:multiplayer-room` 多端压测。
2. **`z/agy/rc/track-f-spectator-ack-sync`（多人观战、省流冻结与 1s ACK 门控）**
   - 接入 Track D 的 **Sparse 模式（冻结帧 + 2Hz 16B Ping-Pong）**：Host 暂停/播放/Seek 时发送状态切换信令，收集全员 `STATE_PAUSED_ACK` / 恢复 ACK。
   - **1 秒 ACK 超时门控**：仿照 Match 的 `Match started → Kickoff` 确认机制，设置 **1000ms ACK 超时窗口**；超时未收齐全员 ACK 时通过 `Notification System` 弹出提示（列明未应答端，后续再决策踢出或强制开跑）。
   - 桥接 4-3-2-1-0 接管倒计时与 `RLRCInterveneTakeoverSubController`。
3. **`z/agy/rc/track-a-telestrator-net-sync`（画笔网络同步与视角一致性）**
   - 将 `da722e3` 的 `TelestratorWireCodec.ts` 接入 Track D 的 `game_ctrl`（`STROKE_COMMIT` / `STROKE_HASH_ACK` / `STROKE_UNDO` / `STROKE_CLEAR`）与 `game_net`（`STROKE_DRAFT`）。
   - 结合 Replay / Spectator 的视角同步（Host POV Follow + Direction Adjust 偏移恢复），确保视角同步下画笔视觉完全一致。
4. **`z/agy/rc/track-c-dynamic-pack`（Dynamic Training Pack 单机与切片引用架构）**
   - 落地 `.rcd`（Rocket Companion Dynamic Training Pack）与独立 `State Session` 存储模型（支持从 Replay `intervene from here` 前 4 秒捕获、Move/Duplicate/Delete Double-check，以及“仅进球或超时”的极简判定器）。
