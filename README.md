# Track A 修复报告与全链路推进讨论

## 一、Track A 修复与推送结果

- **分支**：`z/agy/rc/fix-telestrator-dashed-freeze`（基于 `5b1ee5c3b78bb16bcfc35eb1688b6d930904b97a`，已提交 `2f38585` 并推送至 `origin`）
- **改动文件**：`src/entities/telestrator/telestratorStrokeGeometry.ts`

### 1. 虚线卡死（无报错冻结）根因与修复
- **根因**：在 `buildDashedRibbonPositions` 中，`while (localOffset < segLen)` 循环使用 `(accumPhase + localOffset) % dashPeriod` 计算当前相位与步长 `step`。由于 `dashLen` / `gapLen` / `dashPeriod` 均为浮点数，当 `phase` 极其逼近 `dashLen` 或 `dashPeriod`（差值在 `1e-15` 量级）且 `localOffset` 较大时，`step` 小于 IEEE-754 双精度浮点数的最小表示精度（ULP），导致 `localOffset += step` 实际上没有推进 `localOffset`，从而触发**主线程死循环**（画面彻底卡死、控制台无任何报错）。
- **修复**：
  - 增加 `Number.isFinite(halfWidth) && Number.isFinite(dashPeriod) && dashPeriod > 0` 防护；
  - 引入最小步进下界 `minStep = Math.max(1e-3, segLen * 1e-5)`，确保每次迭代 `localOffset` 必定向前推进；
  - 当 `step <= 1e-3` 时跳过退化小面片生成，避免无效几何体；
  - 增加 `maxIter` 安全上限，从根本上杜绝无限循环。
- **验证**：已通过 `prettier --check`、`pnpm check`（Radar + `tsc --noEmit` 0 Error）及 `pnpm build`。

### 2. 点击 Configurator 只生成 Floating Window 是否为预期行为？
- **是预期行为**。
- 在当前代码（`src/ui-core/recipes/settings/components/TelestratorDeveloperSection.tsx` 与 `src/ui-core/panels/dev/TelestratorToolbarFloatingWindow.tsx`）中，Telestrator 被实现为**与模式无关的功能叠加层（Mode-independent functional override）**，而不是单独的 `Drawing Sandbox` 模式。
- 点击打开悬浮窗后，`TelestratorToolbarFloatingWindow` 挂载时会自动触发 `setOverlayActive(true)`，使画笔直接叠加在当前正在运行的 3D 场景（Freeplay / Replay / Training 等）上；关闭悬浮窗则自动关闭叠加层。

### 3. Track A 当前代码拆分与状态提醒
- 目前 Track A 所有文件均通过 Radar 检查，但有 3 个文件超过了 Radar 的软阈值（400 行 / 12KB，非阻塞）：
  - `src/entities/telestrator/TelestratorManager.ts`（533 行 / 16.0 KB）
  - `src/game/subsystems/TelestratorOverlaySubsystem.ts`（465 行 / 15.5 KB）
  - `src/entities/telestrator/telestratorStrokeGeometry.ts`（430 行 / 13.5 KB）
  - 后续若继续扩展功能（如圆形工具、网络同步发送），建议将 `telestratorStrokeGeometry.ts` 的箭头几何构建拆分出去，并将 `TelestratorOverlaySubsystem.ts` 中的 CMF 网格加载与键鼠事件处理剥离为独立模块。
- 另外，`TelestratorOverlaySubsystem.ts` 目前在松开鼠标后只做了 `encodeTelestratorStroke` / `decodeTelestratorStroke` / `encodeTelestratorHashAck` 的本地编解码自检（统计 `wireBytes` 与校验 hash），**尚未接入 WebRTC 实际传输**，待与网络层信道对接。

---

## 二、Track B / C / D 现状与验收标准

> 注：在当前基准 Commit `5b1ee5c` 上，仓库里**仅合入了 Track A 的代码**；`src/audio/voice/`（Track B）、`src/game/modes/dynamic-training/`（Track C）、`src/network/sync/`（Track D）、`src/network/server/activities/`（Track E）与 `src/network/server/spectator/`（Track F）均**尚未合入主干**。

1. **Track B（WebRTC 语音）：电平条动了算胜利，还是多人互相听到才算？**
   - 仅看到自己本地电平条在动**不算最终胜利**（它只能证明本地 `getUserMedia` + 音频分析器与 UI 绑定正常）。
   - **完整验收标准**：在 Multiplayer 房间中，默认关闭且不弹窗请求麦克风权限；开启后才请求权限并将音轨注入 `RTCPeerConnection`，**双方能在联机中实际听到彼此清晰的原声**，且基于 Player ID 的独立音量调节（0%~250%）与降噪开关正常工作。

2. **Track C（Dynamic Training Pack）：为什么是灰色的？怎么测？**
   - 在当前代码 `src/ui-core/recipes/Layer3SelfTrainingRecipe.tsx`（第 66–73 行）中，`dynamic-training-pack` 菜单项目前**硬编码了 `disabled: true`**，且 Replay 界面也还没有“打包进动态训练包”的入口，因此目前显示为灰色，**需要等 Track C 分支实现并合入后才能测试**。
   - **后续验收链路**：从 Replay 截取介入前状态生成动态切片/Session -> 在 Dynamic Training Pack 列表中解锁并可预览/管理（Move / Duplicate / Delete）-> 进入训练后完成 Lead-in 播放 -> 倒计时 -> 物理接管瞬间模型/角色切换 -> 进球或超时结算并持久化成绩。

3. **Track D（统一房间复制总线与双模同步）：联机没改坏就算成功吗？**
   - “能正常联机没改坏”**只是底线回归（Smoke Test），不算 Track D 的完整验收**。
   - **核心验收标准**：
     - **不断线热切换（`ROOM_ACTIVITY_SWITCH`）**：同一房间内 Freeplay / Match / Replay 模式切换时，不踢人、不断开 WebRTC 连接；
     - **双模状态广播（Continuous vs Sparse）**：Replay 暂停或冻结帧时，总线自动降频进入 Sparse/Frozen 模式（停止无效 60/120Hz 全量快照，仅保留 2Hz 轻量 Ping-Pong 与 `STATE_PAUSED_ACK`）；
     - **双 DataChannel 分流**：Unreliable 信道跑高频快照、Reliable 信道跑控制信令/画笔/模式切换；
     - **自动化压测通过**：`pnpm repro:multiplayer-room` 多端压测通过。

---

## 三、推进顺序与 Pull Request (PR) 标准

### 1. 是否必须等 A、B、C、D 全完成才能推进 E 和 F？
- **不需要等 A、B、C 全部完成**。Track A（画笔）、Track B（语音）、Track C（单机动态训练包）与底层房间架构解耦，**相互独立、谁好谁先合**，不会阻塞 E 和 F。
- **唯一的强依赖是 Track D**：必须先由 Track D 锁定 `RoomReplicationBus` 协议、`ROOM_ACTIVITY_SWITCH` 信令以及与 `HostWorkerProtocol` 的接口契约，**之后再基于该契约并行推进 Track E（Freeplay/Match/Sandbox 内核拆解）与 Track F（多人观战室与接管）**，否则 E/F 会与底层总线产生大量结构性冲突。

### 2. 什么状态可以提 Pull Request (PR)？合并顺序是什么？
- **可提 PR 的准入条件**：
  1. 代码格式化与架构门禁全绿：`pnpm run format:check && pnpm run check`（Radar 0 Error + `tsc --noEmit` 0 Error）及 `pnpm build` 通过；
  2. 遵守仓库红线：无中文注释/日志/Emoji、无 `index.ts` 桶式重导出、无 `soccer` 命名、无超 800 行/18KB 巨型文件；
  3. 职责闭环且无死代码/无兼容垫片，可独立完成自身分支的功能验证（若有耦合旧文件，需在分支内直接清理）。
- **建议合并顺序**：
  1. **第一梯队（并行独立合入）**：Track A（当前修复分支可先提 PR 合入）、Track B、Track C；
  2. **第二梯队（基础设施锁契约）**：Track D 优先合入（或至少先锁定协议接口）；
  3. **第三梯队（上层业务收敛）**：基于 Track D 并行开发并合入 Track E 与 Track F，最后统一执行 `pnpm test:baseline` 与 `pnpm repro:multiplayer-room` 做全量联调验收。
