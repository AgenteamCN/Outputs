# `?fastest=1` 超低功耗渲染模式实现报告

- **分支**: `z/agy/rc/fastest-low-power` (基于最新 `alpha` 分支 `ff3076a` 切出)
- **提交**: `ef4ad1c` (`verified: true`, bot GPG 签名, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)
- **校验结果**: `pnpm run check` (Radar + `tsc --noEmit`) **GREEN**，`pnpm run build` **PASS**

---

## 一、已实现的 `?fastest=1` 核心改动

1. **启动参数与默认配置 (`src/utils/fastestMode.ts`, `src/ui-core/config/trainingDefaults.ts`)**
   - 新增 `isFastestMode()`（识别 `?fastest=1` 或 `?fastest=true`）与 `getFastestRenderScaleOverride()`（支持 `?renderScale=<1..100>`）。
   - `?fastest=1` 启动时默认锁定：**`maxFps = 90`**（120Hz 物理时钟保持不变）、**`renderScale = 10%`**。
   - 解锁渲染比例下限至 **1%**：`RuntimeDOMEventManager.ts` 下限从 `0.1` 降至 `0.01`（且 `?fastest=1` 下强制 `basePixelRatio = 1`，避免高分屏 2x/3x DPR 放大基数）；`DisplayTab.tsx` 滑块 `min={25}` 放开至 `min={1}`。

2. **渲染管线直出：关闭 Bloom / 后处理 / 阴影 / MSAA / PMREM (`RuntimeSceneBootstrap.ts`, `GameRenderSubsystem.ts`, `RuntimeDOMEventManager.ts`, `GameRuntime.ts`)**
   - `WebGLRenderer({ antialias: false, powerPreference: 'low-power' })`，关闭 MSAA 并提示低功耗 GPU。
   - 关闭 `renderer.shadowMap.enabled`，设置 `toneMapping = THREE.NoToneMapping`。
   - 跳过 `PMREMGenerator` + `RoomEnvironment` 环境贴图生成。
   - 跳过 `EffectComposer`、`BoostBloom`、`SupersonicSpeedLinesPass` 的创建与每帧调用，渲染阶段直接执行 `r.renderer.render(r.arena.scene, activeCamera)`。

3. **零阴影 & 零天空盒/雾效/灯光 (`ArenaLighting.ts`, `ArenaWorld.ts`)**
   - `ArenaLightingManager` 在 `fastest` 模式下强制 `shadowConfig.mode = 'off'`、`shouldRenderShadowMap()` 返回 `false`，`updateShadows()` 直接跳过（同时禁用实时阴影与轻量化 Blob Shadow），且不向场景添加 `DirectionalLight` / `HemisphereLight`。
   - `ArenaWorld` 移除 `sky` 天空盒与 `scene.fog`，背景设为纯深色 `0x06080d`。

4. **CMF 碰撞网格场地替代 GLB (`ArenaEnvironmentManager.ts`, `ArenaAssetLoader.ts`)**
   - 跳过 `driftwoods.glb` 与 `2048×3000` 地面贴图及所有场地 Shader。
   - 直接复用物理引擎已缓存的 `fetchArenaCollisionMeshes()` 数据，通过 `buildCmfCollisionMeshes(chunks, false)` 构建场地几何体：使用共享无光照 `MeshBasicMaterial({ color: 0x101620, side: THREE.FrontSide })` + `EdgesGeometry(geom, 25)` 折痕线条（`LineBasicMaterial({ color: 0x3b6ea5 })`），全程开启 `frustumCulled = true`。

5. **极简车辆碰撞箱线框 + 线框轮子 (`VehicleFactory.ts`, `ArenaAssetLoader.ts`, `vehicleBoostEmitter.ts`)**
   - 跳过所有车辆 `.glb` 下载与预热；无论选择何种车型预设，均按其对应的 `HITBOX_PRESETS` 尺寸与偏移生成单次 Draw Call 的 `THREE.LineSegments`（`vertexColors: true`）：
     - 车头（`+X`）4 条边：**亮绿色 `#00ff00`**；
     - 车尾（`-X`）及纵向连接边：**亮红色 `#ff0000`**；
     - 无实体面片、无反射、无 PBR。
   - 车轮改为 8 边形线框 + 十字轮辐（单 `LineSegments`，无模型/无阴影），保留转向与旋转可见性。
   - `VehicleBoostEmitter` 跳过 `PointLight` 创建，保留轻量 Boost 效果。

6. **极简球体与二态 Boost Pad (`BallVisual.ts`, `BoostPadTemplateLoader.ts`, `BoostPadSystem.ts`)**
   - 球体使用低面数 `IcosahedronGeometry(91.25, 1)` + 无光照 `MeshBasicMaterial({ color: 0xe6ecf5 })`，跳过 `BallGoalRimShader` 与阴影。
   - 跳过 Boost Pad `.glb` 与自定义 `ShaderMaterial`：使用共享 `1×2` RGBA `DataTexture`（底部亮橙黄、顶部透明）+ `MeshBasicMaterial` 4 边开放柱体表示向上渐变光柱，底部小方块表示冷却状态；`BoostPadSystem.update()` 改为纯二态瞬时切换（`pad.full.visible = isActive; pad.base.visible = !isActive`），完全移除收集动画与即将刷新动画。

---

## 二、针对你疑问的评估与进一步极限优化建议

1. **车辆六面体实体 vs 纯边缘线条**：
   - 纯边缘线条（`LineSegments`）在低端 GPU 上开销更低：几乎零片元填充（Fill-Rate），无深度/反射/着色器分支，整车仅 1 个 Draw Call（12 条线段 / 24 顶点）。
2. **CMF 场地“完整三角面” vs “精简折痕线条”**：
   - 当前实现结合了 **暗色 `MeshBasicMaterial`（提供遮挡关系，避免看穿墙后/地板下） + `EdgesGeometry(25°)` 折痕线**。`MeshBasicMaterial` 片元着色器只有单条常量输出指令，在 10% 分辨率下 Fill-Rate 开销极小。
   - 如果想继续榨干最后一丝性能，**直接隐藏 CMF 实体三角面、只保留 `EdgesGeometry(25°)` 折痕线**，可进一步把场地片元着色开销降到接近 0（代价是能透过地板/墙壁看到背面线条）。
3. **如果还要继续往下压开销（在“完全看不见画面”之前的最后几步）**：
   - **关闭拖尾与地面指示圈**：禁用 `SupersonicWheelTrailSystem`、`BallSpeedTrail` 以及球体地面双圆环（`indicatorRing` / `indicatorHeightRing`），可再省去动态顶点缓冲更新与透明混合 Pass。
   - **关闭或极简化 Boost 粒子**：`ArcadeBoostParticleMesh` 仍使用 `InstancedBufferGeometry + ShaderMaterial`；若低端机在连续 Boost 时掉帧，可直接跳过 `VehicleBoostEmitter` 更新或仅保留一个简单线框锥体。
   - **渲染帧率从 90 降至 60**：120Hz 物理不变的前提下，90 FPS → 60 FPS 可直接再减少 33% 的 GPU 提交与功耗。
   - **关闭 CMF 实体面片**：仅渲染 `EdgesGeometry` 折痕线框。
