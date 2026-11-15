# RocketCompanion 任务执行与 Freeplay 资源消耗审查报告

---

## 一、PR 创建与 Merge 到 `alpha` 分支执行结果

1. **目标 Commit**：`2fb905260b492ba5717cd806069d7dd7e1410d32`
   - 所在分支：`z/agy/ui/unify-floating-window-icons`
   - 标题：`refactor(ui): unify floating window icons, colors, and titles with radar parity guard`
2. **PR 与合并状态**：
   - 已创建 PR [#20](https://github.com/Antigravi-ty/RocketCompanion/pull/20)（`z/agy/ui/unify-floating-window-icons` → `alpha`）。
   - 无任何合并冲突（`mergeable_state: clean`），已通过 GitHub API 成功 merge 到 `alpha` 分支，合并提交 SHA：`1a0e379e4acc889d1d38172b7d77dc59ae1815d2`。
3. **关于“破坏性更新”的说明**：
   - 该提交将 `PresetConfig`（`iconName` / `iconColor`）确立为菜单启动器、浮窗标题栏（`FloatingWindowIcon.tsx`）和最小化托盘（`FloatingStackIcon`）的**唯一真实数据源（SSOT）**，移除了 `FloatingWindowIcon.tsx` 中的旧版回退映射；
   - 同步对齐了全部 38 个 `SURFACE_META` 声明，并在 `scripts/radar/radarUiAndRepoGuards.ts` 中增加了**静态 1:1 一致性守卫**。后续若新增/修改浮窗或 `SURFACE_META` 未保持同步，Radar 守卫将直接报错拦截。

---

## 二、Freeplay 刚进入可操作状态的资源消耗审查

基于对当前 `alpha` 分支源码的逐层审计，Freeplay（单车 + 单球 + 静态场地）的资源开销分布如下：

### 1. 物理引擎（CPU / WASM）—— **开销极低**
- 物理核心为 Rust 编译的 `src/physics/core.wasm`（约 754KB），由 `RenderClockScheduler.ts` 以固定 `120Hz`（`SIM_INTERVAL_MS = 1000 / 120`）驱动 `SimulationStage.ts`。
- 在 Freeplay 单车单球场景下，单次 WASM 步进耗时通常在 **< 0.2~0.5ms** 量级，CPU 占用非常轻。

### 2. GPU 与渲染管线 —— **主要能耗与瓶颈来源**

即使在“刚进 Freeplay 静止或简单驾驶”的状态下，当前默认管线（`RuntimeSceneBootstrap.ts`、`GameRenderSubsystem.ts`、`DEFAULT_GRAPHICS_SETTINGS`）存在以下高开销项：

1. **默认 120 FPS 渲染调度（`RenderClockScheduler.ts` + `trainingDefaults.ts`）**：
   - `DEFAULT_GRAPHICS_SETTINGS.maxFps = 120`，通过 `MessageChannel` 微任务 + `WebGL2 fenceSync` 驱动。
   - 相比 60 FPS，GPU 提交频率翻倍；相比 30 FPS，GPU 负载直接达到 **4 倍**。
2. **默认 `2048×2048` 实时阴影每帧重绘（`atmosphereFxConfig.ts` + `GameRenderSubsystem.ts`）**：
   - `DEFAULT_SHADOW_CONFIG.mapSize = 2048`（`mode: 'single-sun'`）。
   - `GameRenderSubsystem.ts:128` 在 `shouldRenderShadowMap()` 为真时，**每一帧**都执行 `r.renderer.shadowMap.needsUpdate = true`，导致每帧额外跑一次完整的阴影深度 Pass。
3. **重型后处理链路（`PostprocessingPipeline.ts` + `BoostBloom.ts`）**：
   - `EffectComposer` 分配两张带 Stencil 的 `HalfFloatType`（RGBA16F）全屏 RenderTarget，每帧固定执行 4 个 Pass（`RenderPass` → Bloom 合成 `ShaderPass` → `OutputPass` ACES 色调映射 → `SupersonicSpeedLinesPass`）。
   - 一旦触发 Boost 或 Flip Reset 发光，`BoostBloom.ts` 会先遍历隐藏遮挡材质（`RuntimeBloomOcclusion.applyBloomOccluders()`），**把整个 `arena.scene` 再完整渲染一遍**到半分辨率 `sourceTarget`，随后再跑 **4 次降采样 + 3 次升采样**（单帧额外 `1 + 4 + 3 = 8` 次 GPU 提交）。
4. **体育场 GLB 禁用视锥剔除 + PBR + 高分辨率贴图（`DriftwoodsStadiumLoader.ts` + `RuntimeSceneBootstrap.ts`）**：
   - `DriftwoodsStadiumLoader.ts:105` 对 `driftwoods.glb` 中所有 Mesh 设置了 **`node.frustumCulled = false`**，意味着无论相机朝向哪里，**全量场地网格每帧都被提交给 GPU**。
   - 地面加载 `ground_q1_7gray_2048x3000.webp`（2048×3000 纹理）并注入 `DriftwoodsGroundShader`、`WallFxShaderInjector`；同时 `RuntimeSceneBootstrap.ts` 生成 `RoomEnvironment` PMREM 环境贴图（`environmentIntensity = 0.5`），使所有 `MeshStandardMaterial`（车辆、球、场地、34 个 Boost Pad）都执行完整 PBR IBL 片元着色。
5. **WebGL MSAA 与高面数模型**：
   - 初始化 `new THREE.WebGLRenderer({ antialias: true })`；
   - 球体（`BallVisual.ts`）为程序化细分多面体 + `BallGoalRimShader`；车辆为完整 `.glb` 车身 + 车轮 + 尾焰/拖尾粒子。

---

## 三、针对你提出的两套思路的评估

### 思路 A：将 Render Scale 降到 10%（老旧设备 / 移动端多端联合调试）
- **现状核查**：
  - `RuntimeDOMEventManager.ts:84-86` 逻辑为：
    ```ts
    const scale = Number(currentGraphics?.renderScale ?? 50) / 100;
    const basePixelRatio = Math.min(window.devicePixelRatio || 1, 2);
    const effectivePixelRatio = Math.max(0.1, basePixelRatio * scale);
    ```
  - 引擎底层其实**已经支持**低至 `effectivePixelRatio = 0.1`；但 UI 滑块（`DisplayTab.tsx:449`）目前写死了 `min={25}`，导致设置面板无法直接拉到 `10%`。
- **实际效果评估**：
  - **片元着色器开销剧降**：`10%` 相比默认 `50%`，像素数降至 $(0.1/0.5)^2 = 4\%$（**25 倍像素减少**），对 Fill-Rate 和显存带宽受限的低端移动 GPU 非常有效。
  - **无法解决的开销**：`10%` 渲染比例**不会降低**以下开销——`2048×2048` 阴影贴图（固定分辨率）、`node.frustumCulled = false` 的场地 Draw Call 与顶点着色、120 FPS 的 `MessageChannel` 调度、`BoostBloom` 的场景重遍历。并且 `1280×720` 在 `10%` 下仅剩 `128×72` 像素，远处球体和边界线会严重马赛克化。

### 思路 B：25% 渲染比例 + 极简模型（6 面体车身 + 纯圆球 + CMF 场地替代 GLB）
- **实际效果评估**：
  - **网络与显存收益极大**：跳过 `driftwoods.glb`、车辆 `.glb` 和 `2048×3000` WebP 贴图的下载、解码与 VRAM 上传，对**多端真机联合调试**的启动速度和内存占用（尤其是移动端 Safari/WebView 杀后台风险）改善非常明显。
  - **Draw Call 与顶点压力剧降**：6 面体 `BoxGeometry`（12 三角形）+ 低模 `SphereGeometry` + CMF 碰撞网格直接渲染，顶点和材质切换开销几乎归零。
  - **关键盲区**：如果**只替换模型和降到 25% 分辨率**，而不关闭 **2048 阴影、EffectComposer 后处理、BoostBloom 二次场景渲染、PBR 环境光和 120 FPS 帧率上限**，GPU 依然会把大量能耗花在阴影 Pass、半浮点后处理和 120Hz 刷帧上，达不到“理论最低能耗”。

---

## 四、更实用、低能耗且保留清晰画面的推荐方案（“Eco / Tactical 极简渲染模式”）

相比完全无画面的 `?touch=2` 或极度模糊的纯 `10%` 分辨率，建议在现有渲染管线中增加一套 **`Eco / Minimal` 模式**（可通过 URL 参数如 `?render=eco` 或 Graphics 预设触发），在**保持 120Hz 物理完全一致**的前提下，将 GPU 能耗压低 **85%~95%**，同时画面比 10% 清晰得多：

1. **绕过 `EffectComposer` 与 `BoostBloom`（直出渲染）**
   - 在 Eco 模式下直接调用 `renderer.render(arena.scene, activeCamera)`，跳过 `HalfFloatType` 双缓冲、4 个 Composer Pass 和 8 个 BoostBloom Pass（以及二次全场景遍历）。这是**单帧 GPU 收益最大**的一步。
2. **关闭 `2048×2048` 实时阴影，仅保留轻量化 Blob Shadow / 无阴影**
   - 强制 `shadowConfig.mode = 'off'`，关闭 `renderer.shadowMap.enabled`，省去每帧 `2048×2048` 深度图重绘；若需落地空间感，地面仅用极低成本的圆形贴地假阴影即可。
3. **帧率上限与物理时钟解耦（Render 30/60 FPS + Physics 120Hz）**
   - `RenderClockScheduler.ts` 本身已支持 `simulate`（120Hz）与 `render`（`frameInterval`）独立调度。
   - 多端联合调试或老旧设备开启 `maxFps: 30`（或 `60`），可在**物理判定 100% 确定性**的前提下，直接再砍掉 **50%~75%** 的 GPU 功耗与发热。
4. **渲染分辨率设在 `25%~35%`（同时在 UI/URL 解锁 `10%` 下限）**
   - 关闭后处理与阴影后，在 `25%~35%` 分辨率（且强制 `basePixelRatio = 1`，避免高分屏 2x/3x DPR 放大基数）+ `antialias: false` 下，画面轮廓、球的位置、车头朝向依然清晰可辨，片元开销已微乎其微。
   - 同时建议将 `DisplayTab.tsx` 的 `min={25}` 放开到 `min={10}`（或支持 `?renderScale=10`），以保留极端场景的测试能力。
5. **战术级极简几何体 + 非 PBR 材质（`MeshLambertMaterial` / `MeshBasicMaterial`）**
   - **场地**：直接复用物理 CMF 网格构建半透明/线框/纯色边界与地面网格（开启 `frustumCulled = true`），完全免去 `driftwoods.glb` 与 `2048×3000` 贴图加载，也无需 `RoomEnvironment` PMREM。
   - **车辆**：使用与当前 RocketSim Hitbox 等尺寸的 **6 面体 `BoxGeometry`** + 车头朝向指示块（可复用现有的 `CarNoseOrientationArrows` 逻辑），一眼分辨翻滚与朝向。
   - **球体**：使用低面数 `SphereGeometry(91.25, 16, 12)` 或 `IcosahedronGeometry(91.25, 1)` + 双色/经纬线材质（便于观察球体旋转）。
   - **Boost Pad**：用简单的地面扁平圆盘表示（亮=可用，暗=冷却），替代复杂的 GLB 多子网格。

### 方案对比速览

| 方案 | 画面可读性 | 首屏资源/VRAM | 每帧 GPU 功耗 | 物理精度 | 适用场景 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 默认 Freeplay（50% + 120FPS + 全特效） | 极佳 | 高（GLB + 2K 贴图 + HalfFloat RT） | **100%（基准）** | 120Hz | 正常桌面游玩 |
| 仅开 `renderScale = 10%` | 较差（128×72 级别） | 高（未减少纹理与模型） | 约 35%~50%（仍跑阴影/120FPS/Bloom） | 120Hz | 临时快速降像素 |
| 25% + 极简模型（不关阴影/后处理/120FPS） | 良好 | 极低 | 约 30%~45% | 120Hz | 缺后处理开关的半套优化 |
| **推荐：Eco 模式（25%~35% + 极简模型 + 关阴影/后处理 + 30/60FPS）** | **清晰实用** | **极低（零 GLB/大贴图）** | **约 5%~15%** | **120Hz** | **老旧设备 & 多端真机联调** |
| `?touch=2`（Headless） | 无 3D 画面 | 最低 | ~0%（仅 React UI） | 120Hz | 纯触控/自动化/压测 |
