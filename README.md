# Wave 1 (`W1-02` / `feat/timeline-frame-buffer`) 交付报告

- **分支**：`feat/timeline-frame-buffer`（基于 `origin/z/agy/rc/multiplayer-recheck-base` `7c80b445`）
- **已验证 Bot Commit**：[`54f17b463224e262436e1bda0c17cfb6a456a658`](https://github.com/Antigravi-ty/RocketCompanion/commit/54f17b463224e262436e1bda0c17cfb6a456a658)（`signature.isValid = true`, `state = VALID`）

---

## 一、核心实现内容

1. **`src/network/sync/TimelineFrameBuffer.ts`**
   - 实现 `TimelineFrameBuffer`（零稳态 GC 预分配环形缓冲，默认容量 32 帧）：
     - **60Hz+ 直通模式（Pass-through）**：通过到达间隔 EMA（`avgIntervalMs <= 22ms`，即 `>= ~45Hz`）自动识别高帧率流，直接返回最新帧（`mode: 'passthrough'`，`interpolationDelayMs = 0`）。
     - **15Hz / 30Hz 接收时间轴双帧插值（Arrival-Timeline Interpolation）**：以客户端接收时间 `arrivalMs` 为唯一时间轴（`renderTimeMs = nowMs - clamp(avgIntervalMs + jitterMarginMs, 25, 120)`），在相邻到达帧 `[slotA, slotB]` 间插值；**零 Lookahead 外推、零单调 `tick` 假设**，正放、慢放（0.25x/0.5x）、暂停（停在最新帧不漂移）、倒放（Rewind，`tick` 递减）行为 100% 对称一致。
     - **`txSeq` 乱序/重复包丢弃**：单调递增 `txSeq > 0` 过滤乱序或重复包，统计 `droppedOutOfOrder`。
     - **断层与瞬移护栏**：支持 `discontinuity` 标记（如进球重置/开球）整帧瞬切，以及单实体超距（`maxTeleportDistanceUu = 450`，如 Demo 重生）单车瞬切而其余实体继续平滑插值。
     - **动态切频自适应**：60Hz <-> 30Hz <-> 15Hz 切换时自动在直通与插值之间平滑切换。

2. **`src/network/sync/TimelineFrameMath.ts`**
   - 实现 `basisToQuat`（与 `RLRCQuaternionTrajectoryInterpolator.quatToBasisVectors` 精确互逆）和 `interpolateSnapshotPair`：
     - 球与 1~8 辆车位置、线速度、角速度、Boost、轮速/地面法线做线性插值；
     - 球与车辆 3x3 正交基旋转矩阵通过 `basisToQuat` -> `slerpShortestArcQuat` -> `quatToBasisVectors` 做最短弧四元数 SLERP；
     - 离散标志位（`GOAL`、`ON_GROUND`、`SUPERSONIC`、`DEMOED`、`IS_BOOSTING`、`IS_FLIPPING`、`BOOST_PAD` 激活状态）按 `alpha >= 0.5` 切换。

3. **`src/network/sync/ReplicationStateInterpolator.ts` & `scripts/runMultiplayerSuiteTests.ts`**
   - 将 `SpectateInterpolationBuffer` 接入 `TimelineFrameBuffer`，保持既有公开接口兼容；更新 `testSpectateInterpolationBufferReverseDelay` 中的 `bufferedFrames` 断言（环形保留 3 帧）。

4. **单元测试 `scripts/testTimelineFrameBuffer.ts`**
   - 覆盖 `basisToQuat` 往返精度、60Hz 直通、30Hz/15Hz 正放插值与四元数 SLERP、倒放（非单调递减 tick）、慢放与暂停零外推、`txSeq` 乱序丢弃与 `discontinuity` 瞬切、单实体 Teleport 瞬切、以及 60Hz <-> 30Hz 动态切频。

---

## 二、Wave 2 协议汇总：通道 1（`WorldFrame`）字段需求清单

供 Wave 2 `feat/protocol-schema-alignment` 设计二进制包头与 `WorldFrame` 时对齐：

| 字段 | 类型 / 位宽建议 | 用途说明 |
|---|---|---|
| `txSeq` | `uint32` | 通道 1 严格单调发送序号，用于乱序/重复包丢弃与滑窗裁剪（不可用 `tick` 替代） |
| `tick` | `uint32` | 物理/录像帧号（允许倒放递减、暂停不变或跳帧） |
| `playbackState` | `uint8` enum (`playing \| paused \| rewinding \| fast_forward \| seeking`) | 驱动客户端渲染缓冲与音频静音策略 |
| `flags / discontinuity` | `uint8` bitmask (含 `discontinuity` 位) | 标识开球重置、Seek 跳帧、模式切换等不可跨帧插值的边界 |
| `phase` | `uint8` enum | 房间/比赛阶段 |
| `clockSeconds` | `float32` | 比赛/回放时钟 |
| `focusedCarIndex` | `int8` / `uint8` | 当前观战焦点车号 |
| `cameraPose` (可选) | `vec3` pos + `quat` rot + `float32` fov | Goal Replay / Director 电影机位（当本帧含相机位姿标志时存在） |
| `ball` | `pos(vec3)` + `rot(quat/basis)` + `vel(vec3)` + `angVel(vec3)` | 球刚体状态 |
| `cars[0..N-1]` | `pos(vec3)` + `rot(quat/basis)` + `vel(vec3)` + `angVel(vec3)` + `boost(float32)` + `stateFlags(uint8)` + `wheels` | 车辆状态与离散标志（`onGround`/`supersonic`/`demoed`/`isBoosting`/`isFlipping`） |
| `boostPadMask` | `uint64` / bitmask + cooldown | 34 个 Boost Pad 状态 |

---

## 三、验证结果

- `env -u NODE_OPTIONS node --experimental-strip-types scripts/testTimelineFrameBuffer.ts`：**9/9 单元测试全部通过**
- `pnpm run check`（`radar` + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 边界/体积违规，0 类型错误）
- `RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite`：**28/28 套件测试全部通过**
