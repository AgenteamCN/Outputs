# RocketCompanion `alpha` 分支架构研判与 6-Track 并行开发计划核查报告

> 核查基准：`/work/RocketCompanion` 当前 `alpha` 分支（Commit `f5e8c9f`，`origin/HEAD -> origin/alpha`），已通过 `pnpm run check`（Radar 2.0 + `tsc --noEmit` 全绿，1350/1350 TS 文件）。

---

## 一、 总体结论：报告方向准确，但你的合并认知与报告细节各有一处关键偏差

1. **架构研判与现状完全吻合**：报告指出的 `MatchDomainState.ts`、`MatchPhaseLifecycleSubController.ts`、`MatchSystemControlSubController.ts`、`HostWorkerMessageRouter.ts`、`AuthoritativeServerHost.ts` 职责倒挂问题在 `alpha` 分支中 **100% 属实**。所有被点名的核心模块和符号均真实存在。
2. **大方向并行关系（你的核心判断正确）**：
   - **第一梯队（可从 `alpha` 同步并行启动）**：**Track A (Alpha)**、**Track B (Bravo)**、**Track C (Charlie)**、**Track D (Delta)** 四个 Track 确实可以 **100% 并行** 从最新 `alpha` 拉出分支开发。
   - **第二梯队（依赖 Track D）**：**Track E (Echo)** 和 **Track F (Falcon)** 必须等待 **Track D (Delta)** 锁定协议与 `RoomActivityOrchestrator` 后才能真正落地。
3. **需要纠正的 3 个认知偏差（避免下发错误计划）**：
   - **偏差 1（关键）：Track C (Charlie) 不依赖 Track D，也不需要等 Delta 之后再“和 Echo/Falcon 一起往下推”**。
     - 代码实证：`StaticTrainingPackMode.ts` 在 `onEnter()` 中直接调用 `teardownRuntimeNetworkForStaticTraining(r)`，Dynamic Training Pack（Track C）与 Static Training Pack 一样是 **纯单机引擎**（依赖 `GameModeManager` + `ReplayRingBuffer` + `ReplayBufferSliceExtractor` + IndexedDB），与多人网络总线（Track D）**零耦合**。
     - 因此，Track A / B / C 各自完成后，**直接向 `alpha` 提 PR 合并即可完结**，不需要在 Delta 合并后再重新基于 Delta 开分支推进 Charlie。
   - **偏差 2：目标主干分支是 `alpha`，不是 `main`**。
     - 仓库当前 `origin/HEAD -> origin/alpha`，不存在 `main` 分支。所有 6 个 Track 的基准分支与 PR 合入目标都应是 `alpha`。
   - **偏差 3：Track A / B / C 并不是完全“零触碰网络”，有一个轻量收口点**。
     - Track A 的 `branch02` 涉及 `src/network/protocol/`（画笔 Float32 编解码），Track B 的 `branch05` 涉及向 `WebRTCConnection` 的 `RTCPeerConnection` 注入音轨。它们本身可以独立开发并合入 `alpha`，但**多人观战室里把画笔包真正塞进 Reliable DataChannel 广播**，是在 **Track F（或 D+A 合入 `alpha` 后的集成阶段）** 才完成串联。

---

## 二、 代码库深度核查结果（逐项对照报告）

### 1. 报告完全属实、可直接沿用的部分
- **“Match vs State”痛点**：`MatchDomainState.ts`、`MatchRosterPolicyState.ts`、`MatchPhaseLifecycleSubController.ts`、`MatchSystemControlSubController.ts` 中确实混杂了 `lobbyMode`、`isTrainingMode`、`trainingDisableGoalReset`、`SYS_CTRL_STATE_ONLY` 等布尔开关与状态机分支。
- **单 DataChannel 现状**：`HandshakeSession.ts` 当前仅创建了一条不可靠通道 `pc.createDataChannel('game_net', { ordered: false, maxRetransmits: 0 })`，`DataChannelTransport.ts` 也只维护单通道，因此 Track D (`branch13`) 落地双 DataChannel 分流非常必要。
- **可复用资产均存在**：
  - `ReplayBufferSliceExtractor`（`src/replay/ReplayBufferSliceExtractor.ts`）
  - `staticTrainingVariationMath`（`src/game/modes/static-training/staticTrainingVariationMath.ts`）
  - `RLRCHermiteSlerpResampler`（`src/replay/rlrc/synthesis/RLRCHermiteSlerpResampler.ts`）
  - `RLRCInterveneTakeoverSubController`（`src/replay/rlrc/runtime/RLRCInterveneTakeoverSubController.ts`）
  - `SignalAudioEngine`（`src/audio/SignalAudioEngine.ts`）与 `useInGameHudChoreography`（`src/ui-core/hud/useInGameHudChoreography.ts`）
  - `scripts/runMultiplayerRoomRepro.mjs`（当前 `--mode` 仅支持 `2p | 2p2ai | 3p3ai | both | all`，确实缺少 `freeplay` 与热切换压测）

### 2. 报告中与当前 `alpha` 规则冲突、下发前必须修正的 4 个“暗坑”

| 编号 | 报告原表述 | 代码库实际约束（`alpha`） | 下发前修正建议 |
| :--- | :--- | :--- | :--- |
| **坑 1** | `branch03`: `src/ui-core/telestrator/`<br>`branch07`: `src/ui-core/audio/`<br>`branch11`: `src/ui-core/training/`<br>`branch21`: `src/ui-core/overlay/` | `src/ui-core/AGENTS.md` 强制规定 `src/ui-core/` 必须按 **UI 分层目录**组织（`config -> tokens -> stores -> primitives -> layout -> compounds -> navigation -> (hud/charts/viewport/loading) -> (panels/recipes)`），引擎桥接放在 `src/ui-system/`。**不存在按业务功能建一级子目录的先例**。 | 将这 4 个分支的 UI 文件分别落入 `src/ui-core/stores/`、`compounds/`、`hud/`、`panels/` 及 `src/ui-system/`，不要新建 `ui-core/telestrator` 等目录。 |
| **坑 2** | `branch12` & `branch14` 引用 `src/network/sync/` | 当前 `alpha` 仓库中 **不存在 `src/network/sync/` 目录**。另外 `scripts/radar.ts` 会严格检查循环依赖与跨层引用。 | 明确 `src/network/sync/` 为 Track D 新建目录；引用 `RLRCHermiteSlerpResampler` 时注意避免 `src/replay/` 与 `src/network/sync/` 产生双向依赖（建议抽纯数学工具函数）。 |
| **坑 3** | `branch02` (`src/network/protocol/`) 与 `branch12` (`src/network/protocol/`) | 两个分支都会修改 `src/network/protocol/`，同时开发可能产生协议文件冲突。 | 约定 `branch02` 只新增独立编解码文件（如 `TelestratorWireCodec.ts`），**不修改 `HostWorkerProtocol.ts` 或 `DataChannelTransport.ts`**；等 Track D 合并后再在 Track F 接线。 |
| **坑 4** | `branch05` (`src/audio/voice/`) 接入 `WebRTCConnection` | `scripts/radar.ts` 强制执行 `[TRANSPORT ISOLATION VIOLATION]`：**`src/network/transport/` 禁止 import `src/network/` 和 `src/ui-core/config/` 之外的任何模块**。 | `src/network/transport/` 绝不能反向 import `src/audio/voice/`；必须由上层（`src/game/` 或 `src/ui-system/`）把 `WebRTCConnection.pc` 或窄接口 Port 注入给 `WebRtcNativeVoiceBridge`。 |

---

## 三、 修正后的正确分支与合并流向（可直接用于下发任务）

```text
alpha (f5e8c9f, 最新主干)
 ├──► Track A (branch01 ~ 04) ──► PR 合并回 alpha（完全独立；branch01->02，03/04 可并行）
 ├──► Track B (branch05 ~ 07) ──► PR 合并回 alpha（完全独立；branch05 锁 Port 后 06/07 并行）
 ├──► Track C (branch08 ~ 11) ──► PR 合并回 alpha（纯单机引擎，与 Track D 零耦合；branch08 锁 Schema 后 09/10/11 并行）
 └──► Track D (branch12 ~ 15) ──► PR 合并回 alpha（先锁 branch12 协议，再并行 13/14/15）
                                        │
                                        ▼ (待 Track D 合入 alpha 后，从最新 alpha 拉出)
                                  ├──► Track E (branch16 ~ 18) ──► PR 合并回 alpha（16/17 并行，18 压测收口）
                                  └──► Track F (branch19 ~ 22) ──► PR 合并回 alpha（19->20，21 并行，22 做全链路最终审计）
```

### 一句话总结给团队的口径：
- **A / B / C / D 四线同时从 `alpha` 开干**；**A / B / C 做完直接 PR 合回 `alpha` 结束**（不需要跟在 D 后面再跑一轮）；
- **D 合入 `alpha` 后，再从最新 `alpha` 拉出 E 和 F 并行推进**；
- 最终由 **F (`branch22`)** 在包含 A–F 全量代码的 `alpha` 上完成全链路集成审计（`pnpm check` + `test:baseline` + `repro`）。
