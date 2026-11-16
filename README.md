# RocketCompanion `?fastest=1` 低功耗模式对比、优化交付与音频杂音诊断报告

## 1. 分支对比与清理结果

对两个目标提交进行了逐文件 diff 对比：
- `ef4ad1caf514ab7388ed26060444659629d39242`（`z/agy/rc/fastest-low-power`）
- `01f5153344b841fde3cbd2f6c84af8f721f71653`（`z/agy/rc/fastest-ultra-low-power`）

### 功耗对比结论：**保留 `z/agy/rc/fastest-ultra-low-power`（`01f5153`），删除 `z/agy/rc/fastest-low-power`（`ef4ad1c`）**

| 维度 | `z/agy/rc/fastest-ultra-low-power` (`01f5153`) | `z/agy/rc/fastest-low-power` (`ef4ad1c`) | 胜出 |
| :--- | :--- | :--- | :--- |
| **球场渲染** | 仅渲染 CMF `EdgesGeometry` 线框并释放表面三角面（0 表面像素填充） | 同时渲染 `MeshBasicMaterial` 实体表面 + `EdgesGeometry` 线框（双倍 Draw Call + 全屏填充率） | `01f5153`（功耗显著更低） |
| **球体渲染** | Wireframe 二十面体（`wireframe: true`） | 实心 `MeshBasicMaterial` 二十面体 | `01f5153` |
| **车轮材质** | 全局共享 `sharedFastestWheelMaterial` | 每个车轮新建 `LineBasicMaterial` | `01f5153` |
| **灯光与阴影** | 未关闭 `ArenaLighting` 与 `PointLight` | 显式关闭 `carSun`、`ballSun`、`hemiLight`、`keyLight` 与 `boost` 点光源 | `ef4ad1c`（已将此项优化合入保留分支） |

- **已删除分支**：`origin/z/agy/rc/fastest-low-power`（GitHub API `DELETE` 返回 `204`）
- **已保留并更新分支**：`z/agy/rc/fastest-ultra-low-power`
- **已签名 Bot 提交**：[`1ce8ee6cf560af7063c1785684a2b14a82ea1cf2`](https://github.com/Antigravi-ty/RocketCompanion/commit/1ce8ee6cf560af7063c1785684a2b14a82ea1cf2)（`signature.isValid: true`, `state: VALID`）

---

## 2. 本次代码修改摘要（`?fastest=1` 模式）

1. **默认渲染倍率与帧率上限**
   - 默认 `renderScale` 从 `10%` 调整为 `30%`（`0.3`）。
   - 默认 `maxFps` 从 `90` 调整为 `75`。
   - 涉及文件：`src/ui-core/config/trainingDefaults.ts`、`src/game/runtime/RuntimeSceneBootstrap.ts`、`src/game/runtime/RuntimeDOMEventManager.ts`。

2. **关闭轨迹预测器（Trajectory Predictor）**
   - 在 `src/game/runtime/RuntimePostprocessingAndSessionBootstrap.ts` 中，当 `isFastestModeEnabled()` 为 `true` 时不再实例化 `BallTrajectoryPredictor`（保持为 `null`），彻底消除每帧物理预测采样、几何体构建与异常桥接开销。

3. **简化轮胎几何与运动更新**
   - `src/entities/arena/VehicleFactory.ts`：移除 `createFastestWheelWireframe` 中的十字辐条，仅保留 12 段外圈轮廓。
   - `src/entities/VehicleSuspensionSystem.ts`：在 `isFastestModeEnabled()` 下跳过 `getWheelSusLength`、`getWheelSteerAngle`、左右转向角（`steer.rotation.y`）及悬挂臂/弹簧计算，轮胎直接固定在碰撞盒相对位置，仅保留正常的滚动自转（`spin.rotation.z`）。

4. **音频精简与低功耗防爆音**
   - 仅保留 6 类音效：**jump**、**double jump**、**dodge**、**boost**（含 start / loop / release）、**boost collect**、**empty boost**（`sfx_error_no_boost`）。
   - 关闭并跳过加载/更新：引擎颗粒合成器（`turtle_modem`）、球体滚动/飞行循环音、超音速进入与循环音、翻转重置音效、碰撞/撞墙/门柱/爆破/进球/倒计时与提示音。
   - `src/audio/audioAssetLoader.ts`：在 `?fastest=1` 下仅拉取并解码上述 6 类所需音频资源，大幅减少网络/磁盘 I/O 与内存占用。
   - `src/audio/audioContext.ts`：在 `?fastest=1` 下将 `AudioContext` 的 `latencyHint` 从 `0` 改为 `'playback'`。
   - 涉及文件：`src/game/subsystems/GameAudioSubsystem.ts`、`src/audio/IncidentAudioEngine.ts`、`src/audio/VehicleAudioManager.ts`、`src/audio/SignalAudioEngine.ts`、`src/audio/audioAssetLoader.ts`、`src/audio/audioContext.ts`。

5. **移植灯光与点光源关闭优化**
   - `src/entities/arena/ArenaLighting.ts`：在 `?fastest=1` 下不向场景添加太阳光/半球光/方向光，并将阴影模式设为 `'off'`，短路 `shouldRenderShadowMap()` 与 `updateShadows()`。
   - `src/entities/vehicle/boost/vehicleBoostEmitter.ts`：在 `?fastest=1` 下不创建动态 `PointLight`。

### 验证结果
- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（1390/1390 TS 文件通过，`0` 错误，架构边界 `GREEN`）。
- `pnpm run build`（`vite build`）：**PASS**。

---

## 3. 低性能设备音频“滋啦滋啦响”根因分析报告

### 现象回顾
高配/桌面设备音频正常，仅低功耗、低性能移动设备出现持续“滋啦滋啦”的破音/杂音，且由于移动端无控制台难以直接排查。

### 代码层面定位到的核心根因

1. **最主要根因：`AudioContext({ latencyHint: 0 })` 强制极小硬件缓冲区导致欠载（Buffer Underrun / XRun）**
   - 在 `src/audio/audioContext.ts` 第 202 行中，初始化 `AudioContext` 时默认传入 `{ latencyHint: 0 }`。
   - 在 Web Audio 规范中，`latencyHint: 0` 会向底层系统音频驱动（Android AAudio/OpenSL ES 或 iOS CoreAudio）请求**最小可能缓冲区**（通常仅 128 或 256 采样帧，48kHz 下仅约 **2.6ms – 5.3ms** 的渲染时间窗口）。
   - 在低性能移动设备上（尤其是小核调度、温控降频或与 120Hz WASM 物理步进、WebGL 绘制、JS GC 并发时），音频渲染线程或 IPC 只要偶尔超过 3–5ms，就会发生**周期性音频缓冲区欠载（XRun）**，人耳听感正是连续的“滋啦滋啦”电子杂音/爆音。
   - **修复**：本次已在 `?fastest=1` 模式下将 `latencyHint` 切换为 `'playback'`（由系统分配约 20–50ms 的安全缓冲余量），可从根源消除低性能设备的 XRun 杂音。

2. **次要叠加因素：高并发 Web Audio 图节点与主线程参数调度负载**
   - **引擎颗粒合成器（`RemoteEngineAudioLodController` / `turtle_modem`）**：每帧持续调制多路 `AudioBufferSourceNode` + `BiquadFilterNode` + `GainNode` + 空间总线。
   - **多路空间音频节点**：`AudioSlotPool` 预创建了 16 路 `PannerNode` + `StereoPannerNode`，每辆车与球体各有独立的 `PannerNode`，叠加 `MasterAcousticProcessor`（3 段 EQ + `DynamicsCompressorNode`）以及持续连接的 `AnalyserNode`（`fftSize = 256`）。
   - 在低端移动 CPU 上，上述节点与每帧大量的 `AudioParam` 调度会加剧音频线程抖动，进一步放大 `latencyHint: 0` 的欠载问题。本次关闭引擎音效与多余声道后，音频线程负载已大幅下降。

---

## 4. 进一步降低功耗、冲击稳定 90 FPS 的优化建议

当前低性能设备在 Free Play 已能稳定 75 FPS；若希望进一步压榨功耗以冲击 **90 FPS**，建议按优先级推进以下措施：

1. **绕过 `MasterAcousticProcessor` 与 `AnalyserNode`（音频零后处理）**
   - 在 `?fastest=1` 下，`AudioMixer`（`src/audio/AudioMixer.ts`）仍默认串联了 `MasterAcousticProcessor`（多段 BiquadFilter + DynamicsCompressor）和 `AnalyserNode`。在低功耗模式下可直接 `input.connect(output)` 旁路压缩器与分析器，进一步节省音频线程 DSP 开销。

2. **WebGL 上下文参数极简化**
   - 在 `RuntimeSceneBootstrap.ts` 创建 `WebGLRenderer` 时，可在 `?fastest=1` 下显式传入 `{ antialias: false, alpha: false, stencil: false, powerPreference: 'low-power', precision: 'mediump' }`。关闭 `alpha` 合成与 `stencil` 缓冲可减少移动端 Tile-based GPU（Mali / Adreno）的内存带宽与片上显存开销。

3. **裁剪辅助视觉器与拖尾更新**
   - 检查 `?fastest=1` 下是否仍需要 `ReachabilityVisualizer`、`BallLocatorArrow`、`BallSpeedTrail`、`SupersonicWheelTrailSystem`。若训练场景不需要，可在启动时跳过创建或停止其每帧几何/矩阵更新。

4. **降低 React HUD / Zustand 高频状态同步频率**
   - 75–90 FPS 下，若每帧向 React HUD（如 boost 数值、速度表、计时器）派发状态更新，会触发 DOM 重排与合成器（Compositor）额外功耗。可对 `?fastest=1` 的 HUD 刷新做节流（例如限制在 15–20 Hz 更新 UI，或仅在数值变化超过阈值时更新）。

5. **主循环帧调度与物理步进对齐**
   - 在 90 FPS 目标下，120Hz 物理步进会出现每帧 1~2 次 WASM 步进的交替抖动（1-2-1-2 帧耗时不均），在低端设备容易造成掉帧体感。可结合 `clockScheduler` 优化时间步长平滑，并在不可见/菜单冻结时完全暂停物理与渲染循环（`menuPowerSave: 'freeze'`）。
