# RocketCompanion 推进报告：Track A/D 合入、Track B 诊断、Track C 盘点与新路线图建议

## 1. 已执行操作（Track D & Track A 已合并至 `alpha`）

1. **Track D（PR #14）已验收并合并**
   - 分支：`z/agy/rc/track-d-replication-bus` -> `alpha`
   - 合并前实测 `pnpm run check`（Radar 2.0 + `tsc --noEmit`）全绿通过。
   - 已通过 GitHub API 合并至 `alpha`（Merge SHA: `db8480300db46e6ffd54cacdd0d82583366f12cf`）。

2. **Track A 默认参数更新（PR #15）已提交 Verified Commit 并合并**
   - 新建分支：`z/agy/rc/track-a-telestrator-defaults`（基于 `origin/z/agy/rc/track-a-telestrator-v3` 并合入最新 `alpha`）。
   - 修改文件：
     - `src/entities/telestrator/telestratorTypes.ts`
     - `src/game/runtimeUiStoreProxies.ts`
   - 更新后的默认参数：
     - `DEFAULT_TELESTRATOR_LINE_WIDTH_REF_PX = 5`
     - `DEFAULT_TELESTRATOR_ARROW_LENGTH_REF_PX = 60`
     - `DEFAULT_TELESTRATOR_ARROW_WIDTH_REF_PX = 30`
     - `DEFAULT_TELESTRATOR_PRESETS`:
       - `preset-100`: `opacity: 1`, `widthRefPx: 5`
       - `preset-70`: `opacity: 0.5`, `widthRefPx: 5`
       - `preset-50`: `opacity: 0.1`, `widthRefPx: 5`
   - 已通过 GraphQL `createCommitOnBranch` 生成 Verified Bot Commit（`1db147f4debff915603210c73d141e8e2827b46d`，`VALID` 签名），创建 PR #15 并直接合并至 `alpha`（Merge SHA: `c61ffe12b741ff87af741b242ab70c65d29e8823`）。
   - 合并后在最新 `alpha` 上运行 `pnpm run check`：**0 Radar Error，0 tsc Error，100% TypeScript**。

---

## 2. Track B（WebRTC 语音）深度诊断与建议

### 2.1 为什么「电平条在动、控制台无报错，但听不见自己说话」？
- **直接原因**：在 `origin/z/agy/rc/track-b-voice-fix` 的 `src/audio/voice/WebRtcNativeVoiceBridge.ts`（第 105–109 行）中，`localMicSource` 仅连接到了 `localMicAnalyser`（用于计算 `localMicLevel01` 驱动 UI 电平条），**没有连接到 `ctx.destination`**。
- 这是正常 VoIP 语音通话的防回声/防啸叫设计（本地不播放自己的 sidetone，只把音轨通过 `RTCPeerConnection.addTrack` 发给远端 Peer）。如果单机测试时没有连第二个 Peer，就会表现为「电平条正常随人声跳动、无报错、但听不见自己」。
- 若需要单机自测，只需在 UI 中增加一个可选的 **「Mic Test / 本地回听」** 开关，临时将 `localMicSource` 接到一个受控 `GainNode` -> `ctx.destination` 即可。

### 2.2 当前实现是否符合要求？能否直接提 PR 合并？
- **架构与需求匹配度**：
  - 已通过 `IVoiceAudioBridge` 防腐层 + `runtimeVoiceBridge.ts` 注入，完全不违反 Transport 隔离红线。
  - 已按最新要求移除了车顶说话指示器与 3D 空间衰减，入口放在 Developer Under Development Configurator，且**默认关闭**（开启时才调用 `getUserMedia` 请求麦克风权限）。
  - 已支持按 `playerId` 区分注册用户（持久化）与游客（仅当前会话），并支持 0%–250% 独立音量调节（100% 以下走 `<audio>`，101%–250% 自动切换 WebAudio `GainNode` + `DynamicsCompressorNode` 限幅）。
  - 语音音轨走原生 RTP 媒体流，与 DataChannel 状态同步完全解耦，不会干扰后续总线重构。
- **合并前唯一阻塞项（3 文件冲突）**：
  - 因 Track D（PR #14）刚刚修改了 DataChannel 分流管道，`z/agy/rc/track-b-voice-fix` 与最新 `alpha` 在以下 3 个文件存在合并冲突：
    1. `src/network/transport/P2PWebRTCChannel.ts`
    2. `src/network/transport/webrtc/DataChannelTransport.ts`
    3. `src/network/transport/webrtc/dataChannelFramingPipeline.ts`
  - **建议**：基于最新 `alpha` 新建 `z/agy/rc/track-b-voice-rebased` 解掉这 3 处冲突（把 `sendVoiceSignal` 走 Track D 的 Reliable 控制信道），顺便加上「Mic Test 本地监听」开关后**即可直接提 PR 合并**，无需等后续网络层全部完成。

### 2.3 输入 / 输出降噪方案评估
- **输入降噪**：
  - 当前已启用浏览器/WebRTC 原生 `echoCancellation` / `noiseSuppression` / `autoGainControl`。
  - 如需更强的键盘声/环境底噪抑制，建议直接引入成熟的 MIT 开源库 **`@sapphi-red/web-noise-suppressor`**（基于 `AudioWorklet` + RNNoise / Speex WASM，体积轻量，可在 `MediaStream` 进 `addTrack` 前插一层 Worklet 处理），**不要自己手写降噪算法**。
- **输出降噪**：
  - **不建议做**。语音质量主要取决于发送端输入降噪；对经过 Opus 有损编码传输后的远端音频再跑 AI 降噪，不仅浪费 CPU，还容易出现声音发闷/抽吸效应。保留当前 `VoiceMasterOutputBus` 的 `DynamicsCompressorNode` 输出防爆音限幅器即可。

---

## 3. Track A 剩余待办（UI 协同 & 网络联调）

`alpha` 当前已具备 3D 贴地/贴墙 Raycasting、屏幕比例自适应线宽、实线/虚线/箭头、6 色、3 档透明度、`Ctrl/Cmd+Z` 撤销、一键清空、Direction Adjust 视角偏转与一键 Restore，以及 `TelestratorWireCodec`（FNV-1a Hash + ACK 帧）。
后续需要跨团队协同的部分：
1. **UI 团队**：为窄竖栏 `TelestratorToolbarFloatingWindow` 增加 Hover 自适应左右弹出 Bubble Dialogue 提示气泡。
2. **网络联调（Track D/F）**：目前 `TelestratorOverlaySubsystem` 仅做了本地编解码自环校验，需将 `TelestratorWireCodec` 接入 `RoomReplicationBus` 的 Reliable 控制信道，实现多端同视角下的笔画同步与 Hash ACK 校验重传。

---

## 4. Track C（Dynamic Training Pack）现状与待定事项

`origin/z/agy/rc/track-c-audit` 目前与最新 `alpha` 仅在 `src/ui-core/stores/devFloatingPresets.ts` 存在 1 处小冲突，已实现 `.RCDPack` 编解码、Replay 4 秒预卷截取、120Hz 防双步进、双标记截取与单车独立接管配置。
对照最新需求，以下设计点建议放入后续迭代或专项讨论：
1. **State Session 独立存储与链表结构（`prevSessionId`）**：将 4 秒录制的 State Session 独立落盘，Pack 仅通过 ID 引用并支持跨 Pack 预览、`Move To` / `Duplicate To` / 二次确认 `Delete`。
2. **Intervene 精确切换时机**：倒计时阶段严格保持原录制状态，直到真正接收用户输入的瞬间（或提前半秒预热悬挂）才执行模型替换、他车消失或 AI 接管。
3. **多人在线接管权限**：后续在 Track F 中定义「申请接管 / 交换控制权」的 Reliable 信令。

---

## 5. 新路线图评估（替代旧 6-Track / 22 分支计划）

鉴于 **Track A** 与 **Track D** 已正式合入 `alpha`，**Track B** 与 **Track C** 也已有完整单分支原型，旧计划中「22 个细粒度分支」已无必要继续机械拆分，建议收敛为 **4 个高内聚收敛步骤**：
1. **收敛 Track B & Track C 至 `alpha`**：各自基于最新 `alpha` 解冲突（Track B 3 处、Track C 1 处）并做轻量验证后依次合入，形成完整基线。
2. **落地 Track E（Activity Kernels）**：基于 Track D 已合入的 `RoomActivityOrchestrator` 与 `ActivityKernel` 契约，拆出 `FreeplayKernel`、`MatchKernel` 与 `DevSandboxKernel`（含 Drawing Sandbox / CSR 广播占位）。
3. **落地 Track F（Replay 观战室 + 远程接管 + Telestrator 联机）**：将 `TelestratorWireCodec` 接入 `RoomReplicationBus` Reliable 信道，复用 Track D 的 Sparse 冻结帧 + 2Hz Ping-Pong 模式实现多人观战与 `4-3-2-1-0` 接管。
4. **端到端压测**：在 `scripts/runMultiplayerRoomRepro.mjs` 中补齐 `Freeplay <-> Match <-> Replay` 热切换与动态训练压测。
