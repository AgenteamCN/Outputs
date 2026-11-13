# Track B (`z/agy/rc/track-b-voice-configurator` @ `a44c14d`) 合并评估与 Lobby 语音无声根因诊断报告

> 检验基准提交：`a44c14dd698bcc96a8bfb0aee76347f5effc0d2f`  
> 对比主干：`origin/alpha` (`ae6c593dd9bde4fa92fbd24a529956b0d3a79502`)  
> 对比参考：`origin/z/agy/rc/track-d-replication-bus` (`cf5cfee`)

---

## 一、 核心结论（直接回答你的三个问题）

### 1. Lobby 期间双方能看到自己电平在动，但互相听不见——这符合预期吗？
**不符合预期。**
- 你能在菜单里看到自己的麦克风电平在动，只说明本地采集链路（`navigator.mediaDevices.getUserMedia` -> `localMicAnalyser` -> `voiceChatStore`）是通的，**完全不代表 WebRTC P2P 远端音频链路已经打通**。
- 在 `a44c14d` 中，`src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts` 已经在房间连接建立阶段（Host 的 `onPeerConnected` 与 Client 的 `onJoinServer`）调用了 `attachVoiceToPeerChannel`，**设计上在 Lobby 期间只要双方 P2P 连上就应该能互相听见**。

### 2. 是不是应该等 Track D 合并后，再由 Track E / F 来打通语音链路？
**不是。这是一个关键的架构边界误解。**
- 已核查 `origin/z/agy/rc/track-d-replication-bus`（`cf5cfee`）的全部 61 个变更文件：**Track D 没有任何 `voice` 或 `audio` 相关改动**。Track D 负责的是 `RoomReplicationBus`（游戏状态 60/120Hz 快照与 Sparse 冻结双模同步、`ROOM_ACTIVITY_SWITCH` 热切换、DataChannel 不可靠/可靠双信道分流）。
- **Track B（纯原生 WebRTC 语音）从一开始就是 Stage 1 零阻塞独立子系统**：它直接复用 `P2PWebRTCChannel` 的 `RTCPeerConnection.addTrack` 走原生 Opus 媒体流，并通过 `DataChannelTransport.sendVoiceSignal` 走独立的 `voice_signal` 控制帧做 SDP 重协商。
- Track E（Freeplay/Match/Sandbox 内核）和 Track F（多人观战/接管）运行在 `RoomReplicationBus` 上层，**不会也不应该去接管或修复 Track B 的 WebRTC 音频轨**。语音通话链路必须由 **Track B 自身闭环打通并验收通过后再合并**。

### 3. 当前如何检验 Track B 是否可以开始合并？
- **静态架构与合并冲突层面**：**已达标**。
  - `pnpm run check`（Radar 2.0 架构扫描 + `tsc --noEmit`）全绿通过（`EXIT=0`，1361/1361 TS 文件，0 循环依赖，0 越层引用，0 桶文件）。
  - 与 `origin/alpha`（`ae6c593`）**0 个重叠文件**，合并无冲突。
  - 分层合规：`IVoiceAudioBridge` 窄接口位于 `src/audio/voice/`，`runtimeVoiceBridge.ts` 位于 `src/ui-system/core/` 进行桥接，UI 位于 `src/ui-core/recipes/settings/components/VoiceChatSettingsSection.tsx`，未违反 `src/ui-core/AGENTS.md` 与 `[TRANSPORT ISOLATION VIOLATION]`。
- **运行时功能层面**：**暂不建议直接合并**。需先修复下方定位到的 3 个代码级 Bug，并在 Lobby 中完成双端真机语音互通验收。

---

## 二、 Lobby 期间“能看自己电平、但双方互相听不见”的根因定位

通过逐行审计 `a44c14d` 的信令、连接生命周期与 Web Audio 播放链路，定位到以下 **4 个直接导致无声的问题**：

### 根因 1（致命）：`onVoiceSignal` 注册时序竞态 + Offer Glare 死锁
- **位置**：
  - `src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts`（L259-L269）
  - `src/network/transport/webrtc/DataChannelTransport.ts`（L214-L223）
  - `src/audio/voice/WebRtcNativeVoiceBridge.ts`（L474-L489）
- **原因**：
  1. Client 侧在 `onJoinServer` 中先 `await sessionCoordinator.joinOnlineServer(opts)`（内部会异步加载 3D Lobby 场景），**之后**才调用 `attachVoiceToPeerChannel` 注册 `channel.onVoiceSignal`。Host 侧在 `onPeerConnected` 中也先 `await rosterSyncCoordinator.onRemotePlayerConnected(...)` 再注册。
  2. 但此时 `RTCDataChannel` 已经处于 `open` 状态。先完成 `await` 的一端会立刻发出 `voice_identity` 和（若已开启语音）`voice_offer`。
  3. 另一端此时 `channel.onVoiceSignal` 仍为 `null`，`DataChannelTransport.handleIncomingMessage` 仅执行 `if (this.onVoiceSignal) this.onVoiceSignal(control.signal)`，**不会重放 `transportEvents` 中缓存的 `voice_signal`**，导致 `voice_offer` 被静默丢弃。
  4. 若双方在进房前都已开启语音：Host 发出的 `voice_offer` 被 Client 丢弃；随后 Client 完成加载并发送自己的 `voice_offer`，Host 此时卡在 `signalingState === 'have-local-offer'` 且 `isPolite === false`，触发 `if (offerCollision && !peer.isPolite) return;`，**把 Client 的 Offer 也丢弃**，双方永久死锁，永远不会触发 `ontrack`。

### 根因 2（致命）：`hiddenAudioEl.muted = true` + `AudioContext` 自动播放拦截 / WKWebView 兼容问题
- **位置**：`src/audio/voice/WebRtcNativeVoiceBridge.ts`（L370-L397）
- **原因**：
  1. `bindRemoteStream` 创建了一个 `audioEl.muted = true` 的隐藏 `<audio>` 元素，把实际声音完全交给 `AudioContext.createMediaStreamSource(stream)` -> `GainNode` -> `masterGainNode` -> `ctx.destination`。
  2. 如果接收方**尚未点击开启语音**（或未在用户手势中激活 `getAudioContext()`），`ontrack` 属于异步网络事件而非用户手势，此时 `ensureOutputBus()` 里的 `ctx.resume()` 会被浏览器 Autoplay 策略拦截，导致 `AudioContext.state` 停留在 `'suspended'`，完全无声。
  3. 在 **macOS Tauri (`WKWebView`)** 及部分 Chromium 环境下，对远端 WebRTC `MediaStream` 使用 `createMediaStreamSource` 配合 `muted = true` 的 `<audio>` 常会出现拉流失败或输出全零静音的问题。

### 根因 3（环境依赖）：同机双开若回退到 `CrossTab` (`BroadcastChannel`) 模式，无真实 `RTCPeerConnection`
- **位置**：`src/ui-system/core/runtimeVoiceBridge.ts`（L123）、`src/network/transport/webrtc/HandshakeSession.ts`
- **原因**：如果测试时两端的 P2P 连接走的是同源 `CrossTab` 模拟通道而非真实 WebRTC `RTCPeerConnection`，`channel.connection.pc` 未完成真实 ICE/DTLS 建连，无法承载 `MediaStreamTrack` 音频流（但本地 `getUserMedia` 电平依然会正常跳动，极易造成“以为连上了但听不见”的错觉）。

### 根因 4（边界缺陷）：重协商静默跳过与文件体积预警
- **位置**：`src/audio/voice/WebRtcNativeVoiceBridge.ts`（L331-L345, L423-L448）
- **问题**：
  1. `attachPeer` 监听了 `negotiationneeded`，同时 `syncLocalTrackToPeer` 在 `addTrack` 后又手动调用 `renegotiatePeer`。
  2. 当 `signalingState !== 'stable'`（例如开启/关闭语音切换过快）时，`renegotiatePeer` 直接 `return` 而未排队重试，会导致轨状态与远端不同步。
  3. `WebRtcNativeVoiceBridge.ts` 当前为 **559 行 / 17.8 KB**，已非常接近 Radar 2.0 的 `800 行 / 18 KB` 硬拦截红线（当前触发软提示），后续修复建议适当拆出音频输出总线或信令状态机子模块。

---

## 三、 Track B 合并前的修复与验收清单（Actionable Checklist）

1. **修复信令丢失与死锁**：
   - 在 `DataChannelTransport` 中为未消费的 `voice_signal` 增加小型队列，在 `attachVoiceToPeerChannel` 绑定 `onVoiceSignal` 时立即回放；或将 `attachVoiceToPeerChannel` 提前到 `await joinOnlineServer` / `onRemotePlayerConnected` 之前执行。
2. **修复远端音频播放静默**：
   - 检查接收端 `AudioContext.state`，必要时在 UI 交互时确保 `resume()`，并在 `volumePct <= 100` 且未开启特殊处理时验证 `<audio>` 直出或确认 `WKWebView` 下 `MediaStreamAudioSourceNode` 实际有采样输出（可在 `WebRtcNativeVoiceBridge` 中观察 `peer.audioLevel01` 是否大于 0 来区分“没收到流”还是“收到流但没播出来”）。
3. **Lobby 实机验收标准（无需等待 Track D/E/F）**：
   - [ ] 双方进入同一房间 Lobby 后，在 **Developer -> Under Development -> Voice Chat Configurator** 的 `Per-Player Voice Mix` 中不再显示 `0 Peers`，能看到对方名称与 ID。
   - [ ] 单方开启语音时，另一方界面显示 `Voice active (xx%)` 且对方电平条随说话跳动，能清晰听到声音。
   - [ ] 双方进房前均已开启语音时，进房后不发生 glare 死锁，双向正常通话。
   - [ ] `0% - 250%` 增益调节、本地 Mute、关闭语音后重新开启均能正常恢复。
   - [ ] 运行 `pnpm run check` 保持 0 Error。
