# `?fastest=1` 进一步优化与检查报告

**仓库 / 分支**：`Antigravi-ty/RocketCompanion` (`z/agy/rc/fastest-ultra-low-power`)  
**已提交 Verified Bot Commit**：[`cd5292d1bca4ee9b862bd39a1ed80e2115fa087b`](https://github.com/Antigravi-ty/RocketCompanion/commit/cd5292d1bca4ee9b862bd39a1ed80e2115fa087b) (`signature.isValid: true`)

---

## 一、已完成的代码改动

### 1. 恢复指定音效（`?fastest=1`）
在保留原本精简原则（跳过引擎合成器 `turtle_modem`、球体循环音、普通碰撞/进球/超音速等）的基础上，加回以下 3 类音效及其资源加载：
- **碰撞门柱 / 横梁音效**：
  - `src/audio/audioAssetLoader.ts`：加载 `goalpost` 与 `crossbar_ping_*`（含 `crossbar_ping_small_*`）。
  - `src/audio/IncidentAudioEngine.ts`：在 `isFastestModeEnabled()` 分支中放行 `BALL_GOALPOST_HIT`，以及 `surfaceTag` 为 `GOALPOST` / `CROSSBAR` 或 `isCrossbar` 的 `BALL_WORLD_HIT`，转发至 `worldImpactHandler.handleBallGoalpostHit`。
- **倒计时音效**：
  - `src/audio/audioAssetLoader.ts`：加载 `match_countdown`、`training_countdown`、`match_countdown_10s`、`match_start_go`。
  - `src/audio/SignalAudioEngine.ts`：`handleSignal` 与 `play` 在 `?fastest=1` 下放行 `match_countdown_321`、`training_countdown`、`match_countdown_10s`、`match_start_go`（其余 60s/30s 播报、加时、结束等仍保持跳过）。
- **Flip Reset 音效**：
  - `src/audio/audioAssetLoader.ts`：加载 `reset`。
  - `src/audio/VehicleAudioManager.ts`：移除 `FLIP_RESET_GAINED` 分支中的 `if (isFastestModeEnabled()) break;` 拦截。

### 2. 旁路 `AudioMixer` 后处理与分析节点
- `src/audio/AudioMixer.ts`：在 `isFastestModeEnabled()` 下跳过 `MasterAcousticProcessor`（3 段 EQ + 压缩器）与 `AnalyserNode` 的创建及连接，直接 `this.input.connect(this.output)`；`getAcousticSnapshot()` 兼容返回 `null`。

### 3. 精简 `WebGLRenderer` 上下文配置
- `src/game/runtime/RuntimeSceneBootstrap.ts`：在 `?fastest=1` 下初始化 `THREE.WebGLRenderer` 时显式传入：
  ```ts
  {
    antialias: false,
    alpha: false,
    stencil: false,
    precision: 'mediump',
    powerPreference: 'low-power',
  }
  ```
  减少移动端 Tile GPU 的帧缓冲带宽与着色器精度开销。

### 4. `ReachabilityVisualizer` 状态核查与彻底跳过
- **核查结论**：`DEFAULT_REACHABILITY_CONFIG.enabled` 默认确实为 `false`（`src/ui-core/stores/reachabilityStore.ts`），且 `ReachabilityVisualizer.update()` 在 `!config || !config.enabled` 时会直接 `return`，**并未**在关闭状态下参与每帧计算。
- **发现的冗余**：`src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts` 在 `?fastest=1` 下仍会实例化 `new ReachabilityVisualizer(...)`，导致创建 Three.js Group/Mesh/Material 并调用 `preloadGroundReachabilityLut()`。
- **改动**：在 `RuntimePostprocessingAndSessionBootstrap.ts` 中将 `ReachabilityVisualizer` 移入 `if (!isFastestModeEnabled())` 块内，彻底零分配、零预加载。

### 5. 裁剪 `BallSpeedTrail`（保留 `BallLocatorArrow` 与 `SupersonicWheelTrail`）
- `src/entities/ArenaWorld.ts`：`?fastest=1` 下不将 `ballSpeedTrail.object` 加入场景，`prepareBallSpeedTrail` 仅调用 `supersonicWheelTrailSystem.prepare(camera, this)`，跳过 `ballSpeedTrail.prepare(camera)`。
- `src/entities/arena/ArenaEnvironmentManager.ts` & `src/entities/vehicle/supersonic-trail/SupersonicWheelTrailReplaySync.ts`：`?fastest=1` 下跳过 `ballSpeedTrail.reset()` 与 `ballSpeedTrail.update()`。
- `BallLocatorArrow` 与 `SupersonicWheelTrailSystem` 均完整保留。

---

## 二、额外检查与新发现的优化点

### 1. Bloom 与单面片特效核查
- **Bloom 确认未开启**：在 `RuntimeSceneBootstrap.ts` 与 `GameRenderSubsystem.ts` 中，`?fastest=1` 下 `r.composer` 与 `r.boostBloom` 均为 `null`，渲染直接走 `r.renderer.render(r.arena.scene, activeCamera)`，因此 **Boost 与 SupersonicWheelTrail 不会触发 Bloom 后处理**。
- **新发现 1 — `SupersonicWheelRibbon` 双 Pass 绘制（已修复）**：
  - `src/entities/vehicle/supersonic-trail/SupersonicWheelRibbon.ts` 原本为每条轮迹创建两个 Mesh：`preOcclusionMesh`（Pass A）和基于模板测试（`stencilRef = 2, mask = 0x02`）的 `foregroundMesh`（Pass B，用于穿透半透明球场边框）。
  - 在 `?fastest=1` 下不仅已关闭 `stencil: false`，而且球场为线框模式、不存在半透明边框遮挡。本次已在 `rebuildGeometry()` 中设置 `this.foregroundMesh.visible = !isFastestModeEnabled()`，**直接将超音速轮迹的 Draw Call 减半**。
- **新发现 2 — `VehicleBoostEmitter.preload()` 预加载（已修复）**：
  - `src/entities/vehicle/boost/vehicleBoostEmitter.ts` 的 `preload()` 原先会无条件调用 `this.goldRush.preload()`。本次在 `isFastestModeEnabled()` 下直接提前返回，避免加载多余贴图资源。

### 2. 其他可参考的底层观察（本次未大改，供后续按需推进）
- **HUD / Zustand 同步**：核查 `src/ui-core/stores/hudStore.ts` 后发现，`setBoost` 已走 `globalHudTransientChannel` 直改 DOM 且做了数值去重，`setHasFlip` 也已做 20Hz 量化节流，因此当前 HUD 并非主要瓶颈。
- **`VehicleBoostEmitter` 构造期内存**：即使当前主题为单面片 `light-wave`（`PerformanceBoost`），`VehicleBoostEmitter` 构造函数仍会预先创建 `ArcadeBoostParticleMesh`（`drive`/`trail`）、`cone`、`nozzleFlash` 与 `RealisticBoostEmitter`（`goldRush`）；若后续想进一步压低内存与场景树节点数，可改为按需懒初始化。
- **`PerformanceBoostShader.ts` 片元循环**：其片元着色器中 `evalCubicBezier01` 含 5 次牛顿迭代，若在极低端移动 GPU 上仍觉像素填充率吃紧，可在 `?fastest=1` 下替换为线性插值渐变。

---

## 三、验证结果
- `pnpm run check`（`radar` + `tsc --noEmit`）：**通过**
- `pnpm run build`：**通过**
