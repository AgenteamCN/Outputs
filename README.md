# Wave 1 (`W1-06`: `feat/mode-training-fpv-dual`) 交付报告

- **分支**：`feat/mode-training-fpv-dual`
- **基线**：`origin/z/agy/rc/multiplayer-recheck-base` (`7c80b445f2953076e5356320c6e91bc04da4bba8`)
- **已签名 Bot 提交**：`df5d2c0bb3193698b8ffc0e739cad4c311fa258f`（`signature.isValid: true`, `state: VALID`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）

---

## 一、核心实现内容

### 1. FPV Dual Controller（Coach–Student 绝对主从移交与抢占护栏）
- **300ms 迟滞 + Student 非零输入护栏**（`src/game/runtime/CoachAuthorityHandoverController.ts`）：
  - 新增 `studentControls` 与 `studentHasActiveInput` 参数及 `getIdleElapsedMs(nowMs)`。
  - Coach 有输入或停手 `< 300ms` 时，Host 绝对持权并忽略 Student 输入。
  - Coach 停手 `>= 300ms` **且 Student 确实有非零输入**时，才触发 `grant_to_client` 移交广播权；若 Student 同样无输入，Host 保持广播权不空转移交。
- **Student 输入探针追踪**（`src/network/transport/hostCrossTabAndAuthority.ts`）：
  - `HostInputAuthorityTracker` 新增 `recordStudentInputProbe(peerId, hasActiveInput, nowMs)` 与 `hasRecentStudentInput(nowMs, maxAgeMs = 250)`，在 Host 持权阶段持续感知 Student 是否存在有效操作输入。
- **抢回覆盖护栏 & Client 侧单调 Epoch 围栏**：
  - 新增 `src/game/runtime/ClientFpvAuthorityGuard.ts`：Student 收到更高 `authorityEpoch`（如 Coach 重新输入抢回控制权）时，立即返回 `preempted: true`、`shouldStopLocalPhysics: true`、`shouldSnapToAuthoritative: true`，瞬间停算并屏蔽本地输入与残余帧发送；过期 `authorityEpoch` 直接丢弃。
  - `src/network/client/PredictionReconciler.ts` 的 `setAuthorityState()` 同步增加单调递增 Epoch 围栏（拒绝 `< this.authorityEpoch` 的陈旧状态）。
  - 新增 `src/game/pipeline/stages/fpvDualAuthorityPipeline.ts` 并在 `src/game/pipeline/stages/SimulationStage.ts` 中接入 `stepHostFpvCoachHandover` 与 `syncClientFpvAuthorityGuard`（同时使 `SimulationStage.ts` 体积降至 17.2 KB，低于 18 KB 硬上限）。

### 2. Training Pack 接入与 Ghost Car 前瞻预留
- **Wave 1 临时 TS 接口与字段清单**（`src/network/protocol/trainingFpvDualVNextTypes.ts`）：
  - 定义 `RosterCarRoleFlag`（`'player' | 'bot' | 'ghost'`）、`TrainingGhostCarReservation`、`GhostTrajectoryManifest`、`TrainingPackVNextMetadata`、`TrainingPackVNextFrameHeader`、`FpvDualAuthorityState`、`StudentInputProbePayload` 与 `TRAINING_FPV_DUAL_FIELD_MANIFEST`。
- **Training Pack 统一广播适配器**（`src/game/modes/TrainingPackBroadcastAdapter.ts`）：
  - 统一输出 Static / Dynamic / Kickoff Training 的频道 1 帧头与频道 2 元数据清单，并计算确定性 32-bit FNV-1a `metadataHash`（HEX）。
  - 提供 `buildGhostCarReservation()`（`roleFlag: 'ghost'`、`interactWithBall: false`、`interactWithCars: false`、默认半透明 `opacity: 0.45`）与 `lookupGhostSampleAtTick()`（静态录制幽灵车轨迹二分查找与线性插值）。
  - 已接入 `src/game/modes/static-training/StaticTrainingPackMode.ts`（`reloadCurrentShot` 触发 `beginShotReset`，`onLeave` 重置）。

---

## 二、Wave 2 协议对齐字段清单（供 `feat/protocol-schema-alignment` 汇总）

1. **Channel 1（高频 State 流）**：
   - `authorityEpoch: uint32`（FPV Coach–Student 抢占围栏版本号）
   - `activeInputPeerIdIndex: uint8`（`0 = host/coach`, `1..N = student slot`）
   - `authorityMode: uint8`（`0=host_only`, `1=client_only`, `2=fpv_master_slave`）
   - `trainingPhase: uint8`（`idle | preroll_countdown | ready | playing | goal_scored | result | replay`）
   - `shotIndex: uint16`、`shotResetSeq: uint32`
   - `countdownSeconds: float32`、`remainingTimeoutSeconds: float32`、`isShotLaunched: bitflag`
2. **Channel 2（低频 Metadata 清单）**：
   - `RosterManifestEntry.roleFlag: uint8`（`0=player`, `1=bot`, `2=ghost`）
   - `RosterManifestEntry.ghostOpacity: float32`（`roleFlag=ghost` 时生效）
   - `trainingKind: uint8`（`static | dynamic | kickoff`）、`packId`、`packName`、`shotName`、`totalShots: uint16`
   - `boostPolicy: uint8` + `perCarBoostPolicies`
   - `ghostModeEnabled: boolean`、`ghostTrajectoryHash: uint32 HEX`
3. **Channel 3（瞬时 Signal）**：
   - `TRAINING_SHOT_RESET (param1=shotIndex, param2=shotResetSeq)`
   - `TRAINING_COUNTDOWN_TICK (param1=remainingSeconds)`
   - `TRAINING_OUTCOME_SETTLED (param1=outcomeCode, param2=shotIndex)`
   - `FPV_AUTHORITY_PREEMPTED (param1=authorityEpoch, param2=activePeerSlot)`
4. **Channel 4（Bulk / Asset）**：
   - `GhostTrajectoryManifest`（静态录制幽灵车按 tick 索引帧切片）
   - `DynamicTrainingStateSession`（RLRC 动态训练切片）

---

## 三、验证结果

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（1432/1432 TS 文件，0 循环依赖，0 越层引用，0 孤儿模块，0 硬上限超限）。
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**PASS**（29/29 全部通过，含新增第 29 项 `FPV Dual Arbitration (300ms + Student Input Guardrail, Preemption) & Training Ghost Reservation (W1-06)`）。
