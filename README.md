# RocketCompanion `?fastest=1` 超低功耗测试分支交付报告

## 1. 分支与提交信息

- **源分支**：`alpha` (`ff3076a4381b548d3b2585cac430af4e1513d68b`)
- **新分支**：`z/agy/rc/fastest-ultra-low-power`
- **Verified Bot Commit**：[`01f5153344b841fde3cbd2f6c84af8f721f71653`](https://github.com/Antigravi-ty/RocketCompanion/commit/01f5153344b841fde3cbd2f6c84af8f721f71653) (`signature.isValid: true`, `state: VALID`)
- **校验结果**：
  - `pnpm run check` (`radar` + `tsc --noEmit`)：**PASS (GREEN)**，1390/1390 TS 文件，0 循环依赖，0 越界引用
  - `pnpm run build` (`vite build`)：**PASS (exit 0)**

---

## 2. `?fastest=1` 已实现优化清单

通过启动 URL 参数 `?fastest=1` 触发（工具函数 `isFastestModeEnabled()` 位于 `src/utils/fastestMode.ts`），在保持 **120Hz WASM 物理** 完全不变的前提下，对渲染与资源加载链路做了以下改造：

1. **后处理、Bloom、阴影、环境贴图与抗锯齿全关**
   - `RuntimeSceneBootstrap.ts`：初始化 `WebGLRenderer({ antialias: false, powerPreference: 'low-power' })`，关闭 `renderer.shadowMap.enabled = false`（无实时阴影、无轻量化阴影）、关闭 `ACESFilmicToneMapping`、跳过 `PMREMGenerator` / `RoomEnvironment`、跳过 `BoostBloom`、`EffectComposer` 与 `SupersonicSpeedLinesPass`。
   - `GameRenderSubsystem.ts` & `GameRuntime.ts`：当 `composer` 为 `null` 时，直接调用 `renderer.render(arena.scene, activeCamera)` 直出渲染，跳过全屏半浮点 RenderTarget 与所有后处理 Pass。
   - `RuntimeDOMEventManager.ts`：`updateViewport()` 与画布事件监听兼容 `composer === null` 与 `boostBloom === null` 的直出路径。

2. **帧率上限 90 FPS + 渲染分辨率默认 10%（最低解锁至 1%）**
   - `trainingDefaults.ts` & `RuntimeSceneBootstrap.ts`：`?fastest=1` 下默认 `maxFps: 90`、`limitFps: true`、`renderScale: 10`。
   - `cameraAndGraphicsSlice.ts`：`?fastest=1` 会话内的临时图形调整不会覆盖用户在 `localStorage` 中保存的正常模式配置。
   - `RuntimeDOMEventManager.ts`：`?fastest=1` 下强制 `basePixelRatio = 1`（避免高分屏 2x/3x DPR 放大像素基数），并将 `effectivePixelRatio` 下限放开到 `0.01`（支持 1% 渲染比例）。
   - `DisplayTab.tsx`：将 Render Scale 滑块 `min` 从 `25` 解锁至 `1`。

3. **场地：零 GLB / 零大贴图，直接复用 CMF 物理网格生成结构边缘线框**
   - `ArenaWorld.ts`：跳过 `createStadiumDomeSky()` 天空穹顶与雾效，使用纯深色背景 `#0b0f17`。
   - `ArenaEnvironmentManager.ts`：`loadArena()` 直接读取物理初始化阶段已缓存的 CMF chunks（`fetchArenaCollisionMeshes()`），通过 `EdgesGeometry(mesh.geometry, 15)` 提取结构边缘（滤除平面内部对角线），以单个共享 `LineBasicMaterial` 渲染 `LineSegments` 并开启 `frustumCulled = true`。

4. **车辆：纯碰撞箱边缘线框 + 线框车轮（零车辆 GLB / 零 PBR / 零反射）**
   - `ArenaAssetLoader.ts` & `ArenaVehicleRegistry.ts`：跳过所有车辆 `.glb` 下载、延迟加载与后台预热（`prepareAllArenaWarmupAssets` 直接返回）。
   - `VehicleFactory.ts`：
     - 无论选择何种车型，均根据其对应 `HITBOX_PRESETS` 尺寸绘制单个 `LineSegments`（24 顶点 / 1 Draw Call，使用 `vertexColors: true`）：
       - 车头 4 条边：亮绿色 `RGB(0, 255, 0)`
       - 车尾 4 条边：亮红色 `RGB(255, 0, 0)`
       - 4 条纵向连接边：由车尾红向车头绿渐变，便于任意视角瞬间识别朝向与翻滚姿态。
     - 车轮：使用 12 段线框圆环 + 十字辐条（`LineSegments`），保留转向与滚动反馈，完全取消轮胎/轮毂 3D 模型与 PBR 材质。
   - `arenaVehicleSlotPipeline.ts`：跳过 `applyVehiclePaint` 遍历。

5. **球体：极简线框二十面体**
   - `BallVisual.ts`：跳过高细分程序化足球几何体与 `BallGoalRimShader`，改用 `IcosahedronGeometry(CLASSIC_BALL_RADIUS, 1)` + `MeshBasicMaterial({ wireframe: true })`（仅 80 个三角面，1 Draw Call）。

6. **Boost Pad：地面方块 + 向上渐变光柱，瞬时开关（无蓄力/收集动画）**
   - `BoostPadTemplateLoader.ts` & `ArenaAssetLoader.ts`：跳过 4 个 Boost Pad `.glb` 与 `SphereGeometry(32, 24)` 浮球加载。所有 34 个 Pad 共享少量几何体与 `MeshBasicMaterial`：未激活时仅显示暗色地面方块（2 三角形）；激活时显示亮色地面方块 + 十字交叉向上渐变光片（4 三角形，顶点色由地面金黄向上渐变至透明，`AdditiveBlending`），任意角度清晰可见。
   - `BoostPadSystem.ts`：`update()` 在 `?fastest=1` 下走极简快路径，直接根据 `isActive` 瞬时切换 `pad.full.visible` 与 `pad.base.visible`，跳过所有贝塞尔动画与距离阴影计算。

---

## 3. 针对提问的技术评估与进一步优化建议

### Q1：六面体 Box vs 纯边缘线条，哪个性能更优？
- **纯边缘线条（`LineSegments`）更优**。六面体 `BoxGeometry` 需要光栅化 12 个三角形的内部像素（产生 Fill-Rate 消耗与深度/遮挡计算），而 `LineSegments` 仅绘制 12 条 1px 宽的线段（24 顶点、1 Draw Call），几乎没有片元着色开销，完全不存在反射与面片遮挡，且在 10% 低分辨率下绿头/红尾的辨识度比纯色方块更高。

### Q2：CMF 完整三角面 vs 精简线条（`EdgesGeometry`）差距大吗？
- **内存/网络层面差距不大，但 GPU 顶点与画面清晰度差距明显**：
  - 因为 RocketSim WASM 物理本就必须下载并解析 CMF，复用已缓存的 CMF 不会增加任何网络下载开销。
  - 如果直接用 `wireframe: true` 渲染 CMF 全部三角面，平坦地面和直墙上的大量三角剖分对角线会在 10% 分辨率下糊成一片且增加顶点数；使用 `EdgesGeometry(geom, 15)` 后，共面三角面的内部对角线会被剔除（约减少 70%~80% 线段），只保留球场弧形拐角、斜坡、球门框与边界转折线，既省顶点又清晰。

### Q3：Boost Pad 地面画圆 vs 画方？
- **画方（`PlaneGeometry` 2 三角形）最省事且开销最低**，搭配两片交叉垂直的 `BufferGeometry` 渐变光片（4 三角形，顶点色从底部金黄渐变到顶部透明），单个 Pad 仅 6 个三角形且全场地 34 个 Pad 共享材质与几何体，无需任何自定义 GLSL Shader。

### Q4：在这个基础上，还有哪些可以进一步压榨功耗的优化？
如果后续测试发现仍有功耗/发热瓶颈，可在不完全丢失画面的前提下继续优化以下 4 项：
1. **Boost 尾焰与超音速拖尾（`VehicleBoostEmitter` / `SupersonicWheelTrailSystem`）**：目前仍保留了默认粒子/拖尾逻辑；若要更极限，可在 `?fastest=1` 下跳过 Boost 粒子材质或替换为单根线段/简模圆锥，并关闭 `ballSpeedTrail`。
2. **React HUD 刷新降频**：物理 120Hz + 渲染 90FPS 时，React 层的速度表/遥测 HUD 刷新在低端移动端 CPU 上也有一定开销，可将 HUD 订阅节流到 15~30Hz。
3. **关闭后台音频预加载**：`GameAudioSubsystem` 仍会在后台预加载 41 通道空间音频与合成器资源；若联调不需要声音，可配合跳过音频解码以进一步降低内存占用与初始化 CPU 峰值。
4. **帧率上限进一步下调**：当前按要求设为 `90 FPS`；若低端设备仍发热，可在界面或参数中降至 `60 FPS` 或 `45 FPS`（直接再降 33%~50% GPU 负载）。
