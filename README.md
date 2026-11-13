# Track A (Telestrator) 两次 Commit 深度对比与技术问题研判报告

> 基于仓库 `Antigravi-ty/RocketCompanion` 中 `541087fb581d72a15af3bb14794e0bf7511a3351` (`z/agy/rc/track-a-telestrator-v3`) 与 `3d28980796909d0e1e6116943397005d68cd7427` (`z/agy/rc/telestrator-overlay-v3`) 的实际代码对比（两者均基于 `a03403a` 分叉）。

---

## 1. 两次 Commit 有什么区别？应该采用哪一个接着往下做？

两个 Commit 都修改了相同的 11 个文件，核心目标一致（1080p 参考分辨率线宽归一化、最上层渲染不遮挡、Mesh Raycast 修复、左侧窄悬浮工具栏），但具体实现存在 **5 处关键差异**：

| 对比维度 | `541087f` (`track-a-telestrator-v3`) | `3d28980` (`telestrator-overlay-v3`) | 优劣评估 |
| :--- | :--- | :--- | :--- |
| **Raycast 目标分组与过滤** | 在 `TelestratorOverlaySubsystem.getRaycastTargets()` 中拆分为 `dynamicTargets`（球+车）和 `stadiumTargets`（当前激活场馆）；在 `arenaSurfaceRaycaster.ts` 中对场馆命中做 `isInsideArenaRayBounds` 边界裁剪 + 相机在场外时点积过滤朝外墙面，并与 `raycastArenaSurface`（12 个解析平面）比距离兜底（`analyticalHit.distance + 120 < best.distance` 时回退）。 | 把球、车、场馆混在一个 `extraTargets[]` 中直接 `intersectObjects`，仅硬编码跳过 3 个 Driftwoods 外墙 Mesh 名（`Woods_Field_Walls` 等），命中即 `return`，不与解析平面比距离。 | **`541087f` 明显更稳**。`3d28980` 在 Throwback 场馆或遇到未列名的外围装饰/碰撞 Mesh 时容易打偏。 |
| **相机 FOV 读取** | `TelestratorManager.extractCamFovDeg` 使用 `'fov' in camera && typeof ... === 'number'` Duck Typing。 | 使用 `camera instanceof THREE.PerspectiveCamera`。 | **`541087f` 更健壮**（跨上下文或包装相机对象不易失效）。 |
| **虚线相位累计** | `buildDashedRibbonPositions` 在 `while` 内同时累加 `localOffset` 和 `accumPhase`。 | `while` 内用 `(accumPhase + localOffset) % dashPeriod` 计算相位，段末再更新 `accumPhase = (accumPhase + segLen) % dashPeriod`。 | **`3d28980` 逻辑更清晰**。 |
| **悬浮窗与 Bubble 提示** | 宽度 `56px`（`minWidth: 52`），通过点击设置按钮时手动 `floatingStore.updateSize` 切宽到 `310px`。Hover 提示用普通 `absolute left-full`（无左右自适应翻转）。 | 宽度 `72px`（`minWidth: 64`），用 `useEffect([active, settingsOpen])` 自动同步宽度到 `336px`。Hover 提示用 `createPortal(..., document.body)` + `getBoundingClientRect()`，右侧空间不足时自动 `flipLeft`。 | **`3d28980` 的 Portal Bubble 更好**（不会被窄窗裁切，且支持左右自适应弹出）。 |
| **渲染层级 (`renderOrder`)** | Ribbon `100`，Arrow `105`（`depthTest: false`）。 | Ribbon `1050`，Arrow `1055`（`depthTest: false`）。 | **`3d28980` 更安全**（更不易被游戏内其他透明特效遮挡）。 |

### 选型建议
**建议采用 `541087f` (`z/agy/rc/track-a-telestrator-v3`) 作为主干继续开发**，因为它的 Raycast 分组过滤与解析平面兜底架构更扎实；同时在后续迭代中**合入 `3d28980` 的 3 个局部优点**：
1. `createPortal` + 自动左右翻转的 Hover Bubble；
2. `useEffect` 驱动的悬浮窗尺寸同步；
3. `1050 / 1055` 的高 `renderOrder` 与更干净的虚线相位步进。

---

## 2. 线条平滑与采样是怎么做的？（时间 vs 距离？采样点机制？）

完全**按 3D 空间距离**驱动，**不按时间**。整条管线分三步（位于 `TelestratorManager.ts` 与 `telestratorStrokeGeometry.ts`）：

1. **实时采集去重（`appendDraftPoint`）**：
   - 每次 `pointermove` 做 Raycast 得到世界坐标后，与上一采集点计算 3D 欧氏距离平方；
   - 若 `distSq < 144`（即 **< 12 uu**）则丢弃，否则入队并全量重建当前草稿网格。
   - *隐患*：固定 3D 距离阈值在近处会过密、在远处（如高空俯视对角线）鼠标移动 1 像素就可能超过 12 uu，导致远近采样密度不均。
2. **松手抽稀简化（`endStroke` -> `simplifyAndResampleStroke(points, 24, 6)`）**：
   - **最小间距重采样**：按 `minSpacing = 24 uu`（`distSq >= 576`）再次过滤中间点，保留首尾。
   - **RDP 算法（Ramer-Douglas-Peucker）**：以 `rdpEpsilon = 6 uu` 为弦高误差阈值剔除近似共线的冗余点，控制最终存储与网络传输点数。
3. **渲染细分平滑（`buildStrokeRenderGroup` -> `smoothStrokeCurve`）**：
   - 对简化后的控制点执行 **Catmull-Rom 样条插值**；
   - 每段按弦长 `steps = clamp(ceil(segDist / 20), 1, 10)` 动态细分（约每 `20 uu` 生成一个渲染点），法线 `nx/ny/nz` 同步线性插值并归一化。

---

## 3. 方块条 vs 圆角（Round Cap / Round Join）资源消耗评估

- **当前实现**：`buildContinuousRibbonPositions` 和 `buildDashedRibbonPositions` 均使用 `appendQuad`（每段 2 个三角形）沿平均切线生成平头带状几何体，**首尾为平头（Butt Cap），拐角无圆角（急转弯时 Miter 侧向向量容易变窄或自交翻转）**。
- **改为圆角的资源消耗评估**：**消耗极低，完全可行，无需担心性能瓶颈**。
  - **几何增量估算**：
    - 首尾 **Round Cap**（半圆扇形，6~8 段）：每条笔画仅增加 **12~16 个三角形**。
    - 拐角 **Round Join**（仅在相邻两段夹角超过阈值如 `> 15°` 时在外侧补扇形）：通常一条手绘曲线只有少数急弯，每条笔画约增加 **20~60 个三角形**。
    - 对比场景内 GLB 场馆与 6 辆车的数万至十万级三角形，以及纯无光照 `MeshBasicMaterial` 着色，单条笔画多几十个三角形对 GPU 几乎零压力。
  - **可选更优方案（Shader 方案）**：若后续统一改为带 UV（`u` 沿长度，`v ∈ [-1, 1]` 横向）的 Ribbon，甚至可以在 Fragment Shader 里用 `smoothstep` 直接画出抗锯齿圆角与圆头端点，**0 额外三角形**即可解决锯齿与平头问题。

---

## 4. 现在画图从哪儿开始画？Mesh 还是 GLB？为什么有时会从球的位置开始画？

1. **当前原理（三路混合 Raycast）**：
   - 当前并非单一 Mesh 或单一 GLB，而是 **`dynamicTargets`（球 + 6 辆车 Mesh） + `stadiumTargets`（场馆 GLB Mesh） + `ARENA_PLANES`（12 个解析场馆平面）** 三者混合求交，取距离相机最近的有效命中点。
2. **为什么有时会“从球的位置开始画”？**
   - 直接原因在于 `TelestratorOverlaySubsystem.getRaycastTargets()` 把 `arena.ball`（以及 `arena.cars`）放进了 `dynamicTargets`。
   - 当你点击起笔的位置刚好在球的屏幕投影范围内时，射线会先打中球表面：
     1. 球面法线是径向朝外的，下一帧鼠标移出球体打到地面/墙面时，会产生**剧烈的 3D 深度跳变和法线突变**，线条直接从半空球面斜插到地面；
     2. 球移动后，画在当时球位置上的点悬在空中，看起来就像“莫名其妙从球的位置起笔”。
   - **建议**：战术画笔面向的是**场馆表面**，应默认**将 `ball` 和 `cars` 从 Raycast 目标中移除**（只对场馆内壁/解析平面求交），避免球和车截胡起笔射线。

---

## 5. 箭头方向、变形问题 & 实时跟随设置

### 5.1 为什么箭头有时“抽风”、变形，且固定 50px 每次画出来也不一样？
在 `buildCameraFacingArrowheadMesh` 中存在 **3 个明确的数学/逻辑 Bug**：

1. **3D 固定距离回溯取向导致方向抽风**：
   - 代码从终点 `tipPt` 往前找第一个 3D 距离 `>= 35 uu` 的点作为 `tailPt`。
   - 当笔触跨越墙角/边缘，或最后有轻微回勾、或起/终点打到球/车/不平整倒角导致深度跳变时，`tipPt - tailPt` 的 3D 向量会突然剧烈偏转；若该向量恰好接近视线方向，投影到屏幕平面后几乎退化为零向量，箭头方向就会乱跳。
   - 同时，近处 `35 uu` 在屏幕上只有几个像素（极易受手抖影响），远处 `35 uu` 又太长（方向滞后）。
2. **Billboard 法线用的是 `-camForward` 而不是视线方向（导致广角透视斜切变形）**：
   - 目前箭头三角形构造在垂直于 `-camForward` 的平面上，并用欧氏距离 `hypot(tipPt - camPos)` 算世界尺寸。
   - 在透视相机（尤其大 FOV）下，屏幕边缘的箭头视线 `normalize(camPos - tipPt)` 与 `-camForward` 存在较大夹角，导致**每次画在屏幕不同位置时，箭头都会出现不同程度的透视拉伸/斜切变形**（即使长宽都设为 50px，视觉形状也不一致）。
3. **箭头与线身衔接错位**：
   - 目前箭头底边直接压在 `tipPt` 附近，当线宽较粗或 `projDir` 与最后一段 Ribbon 切线不一致时，粗线条末端会从箭头两侧或前方露出来。

### 5.2 修复与实时更新建议
- **方向与投影**：将最后一段轨迹投影到**屏幕空间（NDC / 像素空间）**，取最后 `20~35px` 屏幕弧长的切线方向来确定箭头朝向；Billboard 平面法线改用 `normalize(camPos - tipPt)`（或直接在屏幕空间构造三角形后反投影到 `tipDist` 深度平面），可彻底保证无论画在屏幕中心还是边缘，50px × 50px 的箭头形状严格一致。
- **实时跟随设置**：当前 `endStroke()` 把 `widthRefPx`、`arrowLengthRefPx`、`arrowWidthRefPx` 静态烘焙进了 `BufferGeometry`。建议在 `TelestratorManager` 增加 `rebuildStrokeGeometry(strokeId)` 或 `updateActivePresetStyle()`，当在浮动窗口调整滑块时实时重建对应笔画的 Mesh，即可实现所见即所得的实时预览。

---

## 6. 更多设置（3 个 Preset、宽度、箭头长宽）放到 Configurator Floating Window
- **当前现状**：`541087f` 和 `3d28980` 实际上**在两处都放了**：
  1. `TelestratorToolbarFloatingWindow.tsx` 底部齿轮按钮点开后，右侧展开的面板里已经包含了 `Line Width (Active Preset)`、`Arrowhead Length`、`Arrowhead Width`、`Opacity Presets (1/2/3)` 和 `Copy JSON`。
  2. `TelestratorDeveloperSection.tsx`（Developer Settings）里也保留了一份完全重复的滑块。
- **收敛建议**：后续可直接删掉 `TelestratorDeveloperSection.tsx` 中的重复滑块，Developer Settings 仅保留开关入口，所有细粒度参数统一放在 Floating Window（或后续 UI 团队指定的 Configurator 悬浮窗）中。

---

## 7. 网络数据同步权衡

- **当前现状**：`TelestratorWireCodec.ts` 已实现二进制编解码（Magic `RCTL`，68B 固定头 + 每点 24B `Float32` 的 `x,y,z,nx,ny,nz`，外加 12B 的 `STROKE_HASH_ACK`），经 `simplifyAndResampleStroke` 后单条笔画约 **300~800 字节**。但目前 `TelestratorOverlaySubsystem.handlePointerUp()` 只是本地自编码自解码跑通，尚未接入实际网络信道。
- **同步策略与带宽权衡**：
  1. **发送时机（Reliable 信道）**：
     - 若只需“画完同步”：松手时直接走 Reliable DataChannel 发一次 `STROKE_FULL`（< 1KB），简单可靠、无需复杂重传状态机。
     - 若需“边画边看”：绘制中每 50~100ms 发增量点（`STROKE_DELTA`），松手发 `STROKE_FINAL_HASH`；对端校验本地拼装结果 Hash，匹配则回 `ACK(matched=true)`，不匹配则回 `matched=false` 触发全量重传 `STROKE_FULL`。
  2. **载荷压缩（可再降 65%+ 体积）**：
     - 球场坐标范围在 `[-6000, 6000]`，用 `Int16`（精度 `0.5 uu` 或 `1 uu`）只需 **6B/点**（目前 `Float32` 占 12B）；
     - 法线 `nx, ny, nz` 可用 Octahedral 编码压为 **2B/点**（甚至画在场馆平面时由接收端反推，占 **0B**），单点可从 `24B` 降至 `6~8B`。
  3. **Hash 校验一致性修复**：
     - 当前 `computeStrokeHash` 未包含 `arrowLengthRefPx` / `arrowWidthRefPx`，建议补入；
     - 当前编码把 `opacity` 量化为 `uint8`（`/255`），解码后浮点精度微差可能导致 `Math.round(opacity * 100)` 与编码端不一致，Hash 计算应统一使用量化后的整数（如 `Math.round(opacity * 255)`）。
  4. **视角一致性**：在 `TelestratorStroke` 中已携带 `camPos`、`camForward`、`camFovDeg`，只要双方视角同步，两端渲染出的屏幕视觉宽度与箭头朝向即可严格对齐；自由视角飞走时只保证 3D 场馆附着位置正确，不做屏幕一致性承诺（符合设计预期）。

---

## 8. 如何 Debug？（原始轨迹 vs 平滑后轨迹对比）

建议在 `TelestratorManager` 中增加一个轻量级 **Debug 可视化开关**（不修改线上默认表现，仅在开关打开时渲染）：
1. **保留中间数据**：在 `ManagedStrokeEntry` 中额外保存 `rawPoints`（采集到的原始 Raycast 点）和 `simplifiedPoints`（RDP 抽稀后的控制点）。
2. **4 层同屏对比渲染**：
   - **原始采集轨迹**：红色 `THREE.Points` + 红色细 `THREE.Line`（一眼定位是哪一帧 Raycast 打到了球/车/边缘产生跳变）。
   - **RDP 控制点**：青色 `THREE.Points`（验证抽稀是否误删了关键拐点）。
   - **Catmull-Rom 平滑曲线**：绿色细线（对比原始轨迹与平滑曲线的过冲/贴合度）。
   - **箭头方向向量**：在 `tipPt` 处用绿色/黄色线段画出 `rawDir` 与投影后的 `projDir`，直接定位箭头抽风的具体帧。
3. **面板实时统计**：在设置面板底部显示上一笔的统计信息（如 `Raw: 84 pts -> Simplified: 14 pts -> Curve: 62 pts | Wire: 428 B`），方便同时调优采样阈值与网络包大小。
