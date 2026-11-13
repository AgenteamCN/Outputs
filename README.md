# RocketCompanion 分支清理、后缀统一与下一阶段开发规划报告

## 一、分支与废弃 Commit 清理结果

### 1. 13 个指定 Commit 核查
已逐一核查全部 13 个 Commit 与 `origin/alpha` 的代码状态，确认对应功能均已通过 PR 合并或被 `alpha` 中的更新实现替代：
- `96dcecb7` (`fix-replay-slot-and-timeout`) → 已由 PR #18 (`01dcdc6`) 覆盖
- `4e9c5271` (`fix-transparency-glow-post`) → 已由 PR #17 (`5094954`) 覆盖
- `d3fb50c8` (`notification-collapse-toggle`) → 已由 PR #16 (`a77934f`) 覆盖
- `c61ffe12` / `1db147f4` / `848a0684` (`broked-alpha` / `track-a-telestrator-defaults`) → 已由 Telestrator CMF v4 (`5b1ee5c`) 及后续修复覆盖
- `0401e7a2` (`track-c-audit`) → 已由 PR #14 (`db84803`) 覆盖
- `3d289807` (`telestrator-overlay-v3`) → 已由 `5b1ee5c` 覆盖
- `4591563b` (`fix-bootstrap-stall`) → 已由 PR #13 (`a6c593` / `d4492d9`) 覆盖
- `1051b102` → 对象在当前仓库已不存在
- `e32ab9da` (`z/user/tauri/cargowhat`) → 已由 PR #8 (`760a8fc`) 覆盖
- `e2fcd32a` (`repro-online-match-and-trajectory`) → `multiplayerRoomReproPhenomenaPipeline.ts` 等已存在且在 `alpha` 中持续演进
- `02cd265d` (`replay-ver-investigate`) → UE3 NetStream 解析已由 `e492dab` 重构对齐 boxcars

### 2. 远程分支清理汇总
已通过 GitHub API 删除 **31** 个远程分支：
- **16 个已合并到 `alpha` 的分支**：`z/agy/rc/brush-diag`、`z/agy/rc/fix-bootstrap-hangs`、`z/agy/rc/fix-replay-slot-takeover`、`z/agy/rc/fix-telestrator-dashed-freeze`、`z/agy/rc/pr-dynamic-training-takeover`、`z/agy/rc/pr-notification-collapse-toggle`、`z/agy/rc/pr-transparency-glow-post`、`z/agy/rc/tauri-dev-watch`、`z/agy/rc/telestrator-cmf-v4`、`z/agy/rc/track-c-dynamic-training`、`z/agy/rc/track-c-fix-and-audit`、`z/agy/rc/track-d-replication-bus`、`z/agy/ui/fix-floating-window-schema`、`z/agy/ui/fix-sf-mono-font-stack`、`z/user/default_branch`、`z/user/tauri/cargowhatagain1`。
- **15 个废弃端头及中间过程分支**：`broked-alpha`、`z/agy/rc/fix-bootstrap-stall`、`z/agy/rc/fix-replay-slot-and-timeout`、`z/agy/rc/fix-transparency-glow-post`、`z/agy/rc/notification-collapse-toggle`、`z/agy/rc/replay-ver-investigate`、`z/agy/rc/repro-online-match-and-trajectory`、`z/agy/rc/telestrator-overlay-v2`、`z/agy/rc/telestrator-overlay-v3`、`z/agy/rc/track-a-telestrator-defaults`、`z/agy/rc/track-a-telestrator-fix`、`z/agy/rc/track-a-telestrator-sandbox`、`z/agy/rc/track-a-telestrator-v3`、`z/agy/rc/track-c-audit`、`z/user/tauri/cargowhat`。
- **保留的未合并分支**：
  - `a2`（旧版 `envinit-worker`）
  - `z/agy/rc/track-b-voice-configurator` & `z/agy/rc/track-b-voice-fix`（未合入的 Audio / WebRTC 语音分支）
  - `z/agy/rc/rust/*`（未合入 `alpha` 的 Rust/WASM 迁移波次分支，因未在清理清单中，暂予保留）

---

## 二、文件后缀统一（PR #19 已合入 `alpha`）

- **基准分支**：`alpha` -> 工作分支 `z/agy/rc/unify-extensions` -> **PR #19**（Verified Commit `bc1e9a2f`，已合入 `alpha` `8d19408a`，合入后已删除临时分支）。
- **统一标准**：`.RCSPack`、`.RCDPack`、`.RCReplay`、`.RLRCReplay`、`.RCGR`、`.zip`、`.RCClip`，全面移除 `.gz` 后缀（保留内部 GZIP 压缩与魔数自动解压）。
- **本次修改范围（19 个文件）**：
  - 导出器与解析器默认名：`src/replay/ReplayFileExporter.ts`（`.RCReplay`）、`src/replay/rlrc/runtime/RLRCReplaySessionBridgePipeline.ts`（`.RLRCReplay`）、`src/replay/ReplayDirectorExporter.ts`（`.RCGR`）、`src/replay/ReplayDirectorPatternParser.ts`、`src/replay/ReplayDiagnosticPatternController.ts`。
  - UI、Store 与 File Picker：`ReplayDirectorOrientationCard.tsx`、`ReplayDirectorTunerFloatingWindow.tsx`、`StateReplayBenchPanel.tsx`、`StaticTrainingPackToolbar.tsx`、`DiagnosticDumpFloatingWindow.tsx`、`ReplayToRcsHeaderAndDeltaSection.tsx`、`DynamicTrainingPackFloatingWindow.tsx`、`devFloatingPresets.ts`、`devHubExperimentalCatalogEntries.ts`、`Layer3TrainingConfigRecipe.tsx`、`OnlineWarmupTelemetryAndReadySection.tsx`、`MultiTrackReplayTimeline.tsx`、`dynamicTrainingPackCodec.ts`、`replayDirectorStore.ts`。
- **验证结果**：`pnpm check`（1387/1387 TS，0 错误，Radar GREEN）与 `pnpm build` 均通过。
- **提醒可能遗漏的导出/格式**：
  1. `.RCSPackE`（可编辑静态包）与 `.RLRCSPack`（开发者自 `.Tem` 转换的静态包）目前仍在 `staticTrainingPackCodec.ts` 与 `StaticTrainingPackToolbar.tsx` 中使用，需确认保留还是归并为 `.RCSPack`。
  2. `.RCClip` 目前在 `src/` 中尚无导出/导入实现，属于后续新增格式。
  3. 各类 Tuner 预设、诊断 Dump 与自检报告仍导出 `.json` / `.csv`（如 `audioConfig.json`、`rc-selftest-report-*.json`），预计保持不变。

---

## 三、架构与文件体积观测（Radar）

全仓无 Hard Error（800 行 / 18 KB），但以下 10 个文件超出 Soft Threshold（400 行 / 12 KB），后续迭代新增功能时建议按职责拆分：
- `src/ui-core/stores/replayDirectorStore.ts`（446L / 14.8 KB）
- `src/ui-core/stores/devFloatingPresets.ts`（405L / 15.1 KB）
- `src/replay/ReplayDirectorPatternParser.ts`（405L / 14.9 KB）
- `src/ui-core/recipes/Layer3TrainingConfigRecipe.tsx`（379L / 15.0 KB）
- `src/ui-core/panels/dev/DiagnosticDumpFloatingWindow.tsx`（363L / 14.3 KB）
- `src/replay/ReplayDirectorExporter.ts`（351L / 13.2 KB）
- `src/ui-core/charts/MultiTrackReplayTimeline.tsx`（333L / 13.9 KB）
- `src/ui-core/panels/dev/subsystem-isolation-lab/StateReplayBenchPanel.tsx`（341L / 12.8 KB）
- `src/ui-core/recipes/online/OnlineWarmupTelemetryAndReadySection.tsx`（332L / 12.8 KB）
- `src/replay/ReplayDiagnosticPatternController.ts`（336L / 12.3 KB）

此外，`z/agy/rc/track-b-voice-fix` 中的 `WebRtcNativeVoiceBridge.ts` 为 483 行（超过 400 行软限制），合入或重构时建议拆分为「Peer 协商/信令子模块」与「本地/远端音频管线子模块」。

---

## 四、下一阶段开发规划与技术诊断

### 1. Audio / WebRTC 语音问题诊断与整合方案
- **现状与问题根因**：
  1. `z/agy/rc/track-b-voice-fix` 尚未合入 `alpha`。
  2. 在该分支的 `src/audio/voice/voiceAudioGraph.ts` 中，当增益 `<= 100%`（默认）时，WebAudio `gainNode` 增益被设为 `0`，完全依赖未挂载到 DOM 的 `new Audio()`（`hiddenAudioEl`）播放，且 `hiddenAudioEl.play().catch(() => {})` **静默吞掉了所有异常**。在浏览器/WKWebView Lobby 阶段若 `AudioContext` 或媒体播放未获最近手势解锁，会**无声失败且控制台无任何输出**；而本地电平条直接采集自 `localMicAnalyser`，因此本地能看到电平条但两端互相听不见。
- **规划方案**：
  - **统一整合到新网络同步结构**：建议将语音通道生命周期绑定到 `EstablishedSessionManager` / `RoomReplicationBus`（而不是零散挂在旧回调中），确保跨 Activity 热切换不掉线。
  - **修复远端播放与可观测性**：优先统一走 WebAudio `VoiceMasterOutputBus` 播放远端流（不再用 `hiddenAudioEl` 静默兜底），并在 `play()` / `AudioContext.resume()` / `renegotiatePeer()` 失败时输出结构化诊断日志与 UI 状态提示。
  - **Voice Chat Mode 房间配置**：在创建房间（Host）与房间同步状态中增加 `voiceChatMode: 'open' | 'team_only' | 'off'`：
    - `open`：全房间互通；
    - `team_only`：根据各 Peer 的 `team`（Blue / Orange）动态控制 `attachPeer` 发送/接收静音路由（或服务端/Host 广播队伍归属后本地实时 Gate）；
    - `off`：禁用麦克风采集与远端语音播放。
  - **键盘打字音降噪**：在 `localMicSource` 与 WebRTC 发送轨（`MediaStreamDestination`）之间接入 `AudioWorkletNode`，集成 MIT 开源降噪库（如 RNNoise WASM 方案）。

### 2. 新网络同步结构 & Multiplayer Training（Online Match 简化版）
- **核心架构**：基于已有的 `RoomActivityOrchestrator` + `ActivityKernel`（`src/network/server/RoomActivityOrchestrator.ts` & `activities/ActivityKernel.ts`），新增 `MULTIPLAYER_TRAINING` ActivityKernel，支持在房间内无感热切换。
- **Multiplayer Training 默认规则**：
  - 无比分概念（隐藏或禁用记分板结算），进球正常进入 Cinematic Goal Replay。
  - Boost 策略：默认「滞空消耗（`limit_when_aerial`）」——地面保持 100% 持续、允许拾取场内 Boost Pad；一旦离地（`!car.onGround`），Boost 从 100% 开始真实消耗，防止空中无限飞行。
- **UI 控制**：
  - 基础配置页提供 Segmented Control：`Limited` | `Limit When Aerial` | `Unlimited` | `Custom`。
  - 选中 `Custom` 时引导跳转至 Advanced 页面，支持更细粒度参数及 **Per-Player（按特定玩家指定）** Boost/规则覆盖。

### 3. 用户移除（Kick）与主动退出（Leave）修复
- **已定位 Bug**：
  - `src/game/runtime/OnlineRemotePeerVehicleSynchronizer.ts` 的 `removeRemotePlayer(carIndex)` 仅清理了 Host 本地车辆与 `remotePeerCarMap` 并更新 Host UI，**未向被移除客户端发送踢出控制信令，也未关闭对应的 `P2PWebRTCChannel`**，导致被踢端收不到任何通知。
  - 客户端主动退出时需确保先通过可靠通道发送 `leave` 控制帧，再关闭连接，保证 Host 与其余客户端立即感知并更新 Roster 与语音状态。

### 4. 自动化测试规划
- **多人虚拟测试（扩展 `scripts/runMultiplayerRoomRepro.mjs` & `MultiClientLoopbackSimulator.ts`）**：
  - 覆盖真人接管（Bot take over user input）与 Auto-fill Bot 的多端组合矩阵。
  - 新增测试用例：
    1. Multiplayer Training 规则验证（无比分、进球回放、地面 100% Boost + 离地真实消耗、4 种 Segmented Control 模式与 Custom Per-Player 覆盖）。
    2. Kick 与 Leave 双向信令测试（被踢端收到通知并退回大厅、退出端立即从其余端 Roster 与语音列表移除）。
    3. Lobby 与对局中 Voice 信令/Track 绑定及 `Open / Team Only / Off` 权限路由测试。
- **画笔（Telestrator）自动化测试**：
  - 利用 `src/network/protocol/TelestratorWireCodec.ts` 既有的 `stroke.hash` 与 `TelestratorHashAck` 编解码，构建自动化压测场景：在场地内随机/程序化绘制多段笔画，通过 DataChannel 同步至远端并校验实时渲染与 `Hash ACK` 一致性。

---

## 五、待讨论与不明确事项

1. **`track-b-voice-fix` 合入顺序**：是先单独修复 `hiddenAudioEl` 静默播放问题并合入 `alpha` 做一次快速验证，还是直接基于 `alpha` 的新 `RoomReplicationBus` 架构重构后再合入？
2. **静态包衍生后缀**：`.RCSPackE` 与 `.RLRCSPack` 是否保留，还是统一对外只用 `.RCSPack`？
3. **`.RCClip` 格式定位**：`.RCClip` 具体对应哪类导出（如高光片段/短回放），结构上是否复用 `RCREPLAY` V2 容器？
4. **键盘降噪 MIT 仓库**：团队是否已选定具体库（如 `@sapphi-red/web-noise-suppressor` / `rnnoise-wasm` 等）？
5. **Multiplayer Training Advanced 页面范围**：除 Per-Player Boost 选项外，Custom 还需要开放哪些参数（如 Goal Replay 开关、球速/重力、自动复位延迟等）？
6. **Lobby Voice 行为确认**：确认 Lobby 期间预期行为即为 `Open`（或按房间设置的 `voiceChatMode`）可正常互通，无「Lobby 静默」设计。
