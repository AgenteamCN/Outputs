# RocketCompanion Wave 1–4 Octopus Merge 验证与真机测试报告

## 1. Octopus Merge 执行结论

**结论：可以执行 Octopus Merge，且已完成无损合并与 GPG Verified Bot 提交。**

- **目标分支**：`z/agy/rc/rust/wave-1-4-octopus`
- **Verified Commit SHA**：`84f20d2177d12df7a7593ef8fc2c78fc86294f50`（`verified: true`，4 个 Parent 均基于共同基线 `b88102a60527e50892564b15b7ff2e1acee401c0`）：
  1. Wave 1：`8a6e00a408d2fd39ffa10601c1b14ed3a886f023` (`origin/z/agy/rc/rust/wave-1`)
  2. Wave 2：`1409ec743742b234f3b2843a6fd195a987818802` (`origin/z/agy/rc/rust/t5-wave2-ue3-replay`)
  3. Wave 3：`514be256901d72d568dfe5f272dc147432bbef92` (`origin/z/agy/rc/rust/wave3-multiplayer-bus-reconciler`)
  4. Wave 4：`50b9b01d92d2a1c62aa1f7c5129bb16e5aab4157` (`origin/z/agy/rc/rust/wave4-room-kernels-csr-soak`)
- **冲突与处理**：
  - `src/physics/core.d.ts` 与 `src/physics/core.js`（Wave 2 与 Wave 3 重叠）：Git 自动无冲突合并成功。
  - `src/ui-system/GameUIRoot.tsx`（Wave 1 与 Wave 4 重叠）：仅在顶层 `useEffect` 生命周期挂载处存在 1 处简单文本冲突（Wave 1 挂载 `ensureVoiceChannelBridge` / `DrawingSandboxController`，Wave 4 挂载 `syncCsrFollowHostNotification`），已合并保留双方初始化与清理逻辑。

---

## 2. Linux Headless 沙盒自动化验证结果

| 检查项 | 结果 | 详情 |
| :--- | :--- | :--- |
| `pnpm run check` (Radar 2.0 + `tsc --noEmit`) | **PASS** | 1331/1331 TS 文件 + 83 自定义 Rust 文件，0 循环依赖、0 桶文件、0 类型错误 |
| `pnpm run build:wasm:check` | **PASS** | `cargo check --target wasm32-unknown-unknown --workspace` 9.09s 通过 |
| `pnpm run build:wasm:dev` | **PASS** | 生成 `src/physics/core.wasm`（1160.2 KB，198 exports） |
| `npx vite build` | **PASS** | 生产构建 23.16s 通过 |
| `node scripts/benchmarkTrajectory120Hz.mjs` | **PASS** | 3.01 us/frame，120Hz 预算余量 99.96% |
| `cargo test --manifest-path crates/Cargo.toml` | **BLOCKED (环境限制)** | 沙盒未安装 `cc`/`gcc` 链接器，WASM 可编译但 native test 需在真机运行 |
| `pnpm run test:baseline` (Headless Chrome) | **RED (2 项断言失败)** | 0 运行时/控制台错误，24/24 静态描述符匹配；失败项：<br>1. `multiplayer_6car_loadout_stress`: `multiplayer_kickoff_transitioned_to_playing` (`isJumping=true, posZ=74.59`) 与 `multiplayer_no_phantom_double_jump_on_flip_e` (`phantomDoubleJumps=1`)<br>2. `training_and_kickoff`: `kickoff_ppo_ghost_trajectory_and_episode_ste` (`epTick=254, pts=0, aiTouch=380`) |
| `pnpm run repro:multiplayer-room` | **EXIT 0 (有基线异常)** | Host 端正常；Client 端存在 `missing_match_participant_car` / `client_lobby_connected` 等异常（Wave 3/4 新总线尚未完全替换旧 `MatchDomainState` 主循环） |
| `scripts/verifyWave4Multiplayer.ts` | **FAIL (Node ESM 解析)** | `node --experimental-strip-types` 无法解析 `quickNotificationStore.ts` 中无后缀导入 `'./syncBridge'` |

---

## 3. 各 Branch 成果审查与未闭合的集成缺口

虽然 4 个分支可以无损 Octopus Merge 并通过编译与架构雷达检查，但代码审计发现以下**尚未完全闭合的跨模块接线缺口**（可在真机测试观察后通过单线分支补齐）：

1. **Wave 1 (`8a6e00a4`)**：
   - **Tier 3 音频 & Tier 4 轨迹**：`director_impact_events.rs` 与 `incident_classifier.rs` 双路径已修复（移除 `ev.speed` fallback、接入 `dir_dot < 0.15` 浅角抑制、冷却提升至 `>= 60ms / 8 ticks`）；`WasmAudioDirectorBridge.ts` 已接入 `AUDIO_CUE_CARS_WORD_OFFSET = 22`，`VehicleAudioManager` / `TurtleModemEngineSynth` 已瘦身切流。
   - **3D 战术画笔 (Telestrator)**：`DrawingSandboxController`、`arenaSurfaceRaycaster`、`TelestratorRibbonRenderer`、`TelestratorToolbarFloatingWindow` 与 `CameraManager` 的 `povAngularOffset` 已实现并挂载于 `GameUIRoot`，但**尚未接入 Wave 3 的 `TelestratorSyncChannel`**，联机同步仍待连线。
   - **WebRTC 原生语音**：`WebRtcNativeVoiceBridge` 与 `runtimeVoiceChannelBridge.ts` 已实现本地麦克风权限显式请求、降噪开关与 0–250% 增益控制，但 **`bindVoicePeerConnection(playerId, pc)` 尚未在 `WebRTCConnection` / `MultiplayerSessionManager` 建连处调用**，远端 Peer 音频流尚未注入。
   - **RCDPack 动态训练**：`DynamicTrainingMode`、IndexedDB 存储与 `DynamicTrainingEditorFloatingWindow` 已实现，但 **`DynamicTrainingMode` 尚未注册接入 `GameModeManager` 游戏模式运行主循环**（当前仅编辑器调用了 `captureSessionFromReplay`）。
2. **Wave 2 (`1409ec74`)**：
   - `crates/rc_replay_codec` 已实现 UE3 BitReader、Header、Actor/NetStream 解码与 `touch_analyzer.rs`，`RLRCWasmReplayBridge.ts` 已接入 `RLReplayToRLRCConverter.ts` 与 `ReplayTouchTimelineAnalyzer.ts`，并删除了 21+ 个旧 TS 解析文件。
3. **Wave 3 (`514be256`)**：
   - `clock_sync.rs`、`reconciler_exports.rs`、`wasmReconcilerBridge.ts` 已接入 `RocketSimPhysicsEngine.ts` 与客户端回滚管线；`RoomReplicationBus`、双 DataChannel 与 `TelestratorSyncChannel` 已实现，但尚未完全切离旧 `MatchDomainState` 房间同步循环。
4. **Wave 4 (`50b9b01d`)**：
   - `RoomActivityOrchestrator`、4 类 `ActivityKernel`（Freeplay/Match/ReplayIntervene/Sandbox）、`SpectatorRoomTimelineController`、`InterveneHandoffCoordinator` 与 `csrFollowHostBridge` 已落地并接入 `HostWorkerMessageRouter.ts` 与 `GameUIRoot.tsx`。

---

## 4. Real Device（真机）重点测试清单

由于 Linux Headless 环境缺乏真实声卡输出、麦克风硬件、桌面 GPU/高刷渲染交互与多端真实网络，**真机测试（macOS / Windows）请重点验证以下 6 项**：

1. **音频引擎听感与碰撞防抖（Tier 3）**
   - 在真机扬声器/耳机下实测：高速贴地平滚、球以极浅角度擦碰侧墙/弧角、地面低速带球，确认已无高频连击“机关枪抖动音”。
   - 对比旧版 Golden Sample，确认引擎转速音、Boost 启停、超音速音爆、球车/门柱撞击音高与空间衰减听感一致，无爆音或静音丢失。
2. **WebRTC 语音权限与降噪增益（Voice）**
   - 在 Developer Settings 确认语音开关**默认关闭**且启动时不弹出麦克风权限请求；手动开启时确认每次均正确触发系统麦克风权限弹窗。
   - 测试输入降噪（AEC/NS）开关与 Peer 音量滑块（0% → 100% → 250% 放大）在真机音频设备下的清晰度与防削波表现。
3. **3D 战术画笔与视角锁定交互（Telestrator）**
   - 在 Developer Settings 进入 Drawing Sandbox，验证 6 辆车生成、无 Title/Footer 竖栏工具条交互（实线/虚线/箭头、6 色、100%/70%/50% 透明度）。
   - 验证鼠标在草皮、侧墙、弧角、天花板上的 Raycast 贴合精度、`Cmd+Z`/`Ctrl+Z` 撤销、1–6 号车视角与 7 号自由视角切换。
   - 验证 **Direction Adjust** 视角偏转锁定（转动视角时禁止绘制、绘制时禁止转视角）及一键 Restore / 切回 1–6 自动归零。
4. **真实 `.replay` 大文件流式解析（Wave 2）**
   - 在真机 UI 中导入真实 Rocket League `.replay` 文件，验证 Rust WASM 解码耗时、内存占用、触球时间线标注准确性及回放画面同步性。
5. **跨端联机回滚与 `test:baseline` 暴露的问题定位（Wave 3 + Wave 4）**
   - 在两台真机（或同机双窗口真实 WebRTC）下测试 Freeplay / Match / Replay 房间热切换与 CSR `Completely Follow Host`（尤其验证不同屏幕分辨率/宽高比客户端的渲染一致性）。
   - **重点复现排查**：`test:baseline` 中出现的 `multiplayer_no_phantom_double_jump_on_flip_e`（`phantomDoubleJumps=1`）与 `kickoff_ppo_ghost` 失败问题，确认是 Wave 3 WASM Reconciler 边沿状态同步引入还是原有基线漂移。
6. **Rust 原生单元测试补跑**
   - 在真机（带 Xcode `cc` / MSVC 环境）执行 `cargo test --manifest-path crates/Cargo.toml`，确认 `rc_audio_director::director_impact_tests` 等全部 Rust 单元测试通过。
