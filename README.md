# Track A · Telestrator Overlay v3 修复与策略说明

- **工作分支**: `z/agy/rc/telestrator-overlay-v3`（基于 `origin/z/agy/rc/telestrator-overlay-v2`）
- **Verified Commit**: `3d28980796909d0e1e6116943397005d68cd7427` (`signature.isValid: true`)
- **验证结果**: `pnpm run check`（Radar 2.0 + `tsc --noEmit`）与 `pnpm run build` 均通过（0 Error）。

---

## 一、 核心问题解答：上一版粗细控制是什么策略？为什么 `v2` 改偏了？以及本次的终极解法

### 1. 上一版（`ddbd338`）的策略原理
上一版 `TelestratorManager.ts` 中的 `computeScreenRelativeWorldWidth` 使用了**视锥体反投影公式**：
```ts
worldHeightAtDist = 2 * Math.tan(vFovRad * 0.5) * safeDist
worldPerPx = worldHeightAtDist / viewportHeightPx
worldWidth = widthPx * worldPerPx
```
- **为什么上一版手感很好**：它根据笔划起点到摄像机的距离 `safeDist` 和垂直视场角 `vFov`，把屏幕像素宽度动态换算成 3D 世界宽度（UU）。因此无论你在脚下近处画还是隔着全场远处画，屏幕上的粗细都基本一致。
- **它的唯一短板（多端广播时）**：它除以了本地的 `viewportHeightPx`。在 Broadcast 场景下，如果 Host 是 `1080p`（`viewportHeightPx = 1080`），Viewer 是 `4K`（`2160`）或 `720p` 窗口，同样的 `8px` 在 4K 屏幕上相对于车和球会细一半，在 720p 上会粗 1.5 倍。

### 2. 为什么 `v2`（`a03403a`）理解有偏差？
`v2` 把粗细直接改成了固定的 `28 UU`（真实世界单位）。这导致：
- 摄像机离地面很近时，28 UU 像刷墙滚筒一样巨大；
- 摄像机在全场远端时，28 UU 细成 1~2 像素几乎看不清。

### 3. 本次方案：1080p 虚拟参考高度归一化 + 逐点距离自适应（`widthRefPx`）
我们在 `telestratorTypes.ts` 与 `telestratorStrokeGeometry.ts` 中实现了 **1080p Reference Viewport Normalization**：
```ts
worldHeightAtDist = 2 * Math.tan(vFovRad * 0.5) * max(120, dist(point, camPos))
worldWidth = (widthRefPx / 1080) * worldHeightAtDist
```
- **跨分辨率 100% 一致**：公式完全不依赖本地 `window.innerHeight` 或 `devicePixelRatio`，而是以 `1080` 为常量基准，等价于“占屏幕垂直高度的比例”（例如 `10 px @1080p` 永远等于视口高度的 `0.926%`）。因为车和球在屏幕上的占比也是由 `worldHeightAtDist` 决定的，所以无论 720p、1080p、1440p 还是 4K，所有人看到的**线条/箭头相对于车和球的粗细完全一致**。
- **逐点距离自适应**：不再只用起点距离，而是对曲线上的**每个顶点**分别计算到 `camPos` 的距离来生成 Ribbon 宽度，从脚下画到远端球门也不会变细；箭头长度/宽度（`arrowLengthRefPx` / `arrowWidthRefPx`）同样按 `tipPt` 距离归一化。
- **网络协议同步**：`TelestratorWireCodec.ts`（`STROKE_HEADER_BYTES = 68`）已同步序列化 `widthRefPx`、`arrowLengthRefPx`、`arrowWidthRefPx`、`camPos` 和 `camFovDeg`。

---

## 二、 其余 3 项修复清单

### 1. Telestrator Floating Window 顶部 Padding、左侧锁定与 Settings 菜单不可访问
- **顶部留白过大**：在 `TelestratorOverlaySubsystem.ts`（`spawnToolbarWindow`）和 `devFloatingPresets.ts`（`TELESTRATOR_TOOLBAR_PRESET`）中设置 `contentPadding: 4` / `{ x: 6, y: 4 }`，并去掉 `TelestratorToolbarFloatingWindow.tsx` 顶部的多余 `py-1`。
- **工具栏跑偏 & Settings 被裁切**：
  - 原先 `FloatingWindowInstance` 与 `FloatingWindowContent` 带有 `overflow-hidden`，导致 `absolute left-full` 的 Settings 面板和 hover 提示气泡被完全裁切且无法点击。
  - 现改为横向 Flex 容器：左侧工具栏固定 `w-[56px] shrink-0` 永远紧贴最左侧；右侧在 `settingsOpen === true` 时内联渲染 Settings 面板（并通过 `floatingStore.updateSize` 自动将窗口从 `72px` 展开到 `336px`，关闭时恢复 `72px`），在 `settingsOpen === false` 时用 `flex-1 min-w-0` 占位符占满右侧，保证无论窗口拉多宽，左侧工具列始终贴左。
  - Hover 提示气泡改为通过 `createPortal(..., document.body)` 挂载，彻底摆脱 `overflow-hidden` 裁切，并在靠近屏幕右边缘时自动向左翻转。

### 2. 弧线过渡区 / 六边形网格墙面 / 车身无法起笔或画不上去
- **原因**：此前 `TelestratorOverlaySubsystem.ts` 调用 `raycastScreenToArena(ndcX, ndcY, cam)` 时没有传入 `extraTargets`，只命中了 `arenaSurfaceRaycaster.ts` 里 12 个纯平数学平面（`ARENA_PLANES`），导致地面与墙面的弧线过渡区、六边形网格墙面和车身全部被跳过或穿透到后面的平面。
- **修复**：
  - 在 `TelestratorOverlaySubsystem.ts` 新增 `collectRaycastTargets()`，收集当前可见的车辆（`arena.cars`）、球（`arena.ball`）以及当前激活的球场模型（`driftwoodsStadium` / `throwbackStadium`）传入 `raycastScreenToArena`。
  - 在 `arenaSurfaceRaycaster.ts` 中增加祖先可见性校验 `isObjectChainVisible`，并跳过从场外透视看向场内时命中的单向透明外墙背面（`Woods_Field_Walls` / `Woods_Field_Walls_Hexes` / `Woods_Field_Wall_Posts` 且 `worldNormal.dot(rayDir) > 0`），同时保留 `raycastArenaSurface` 作为兜底。

### 3. 遮挡关系（Stroke 永远在最顶层）
- **原因**：此前 `telestratorStrokeGeometry.ts` 中只有箭头设置了 `depthTest: false, depthWrite: false`，而实线/虚线 Ribbon 仍开启了深度测试（`renderOrder = 10`），导致画在地面或车身上的线条容易被车辆、球或弧形网格遮挡。
- **修复**：将 Ribbon 与 Arrowhead 统一设置为 `depthTest: false`、`depthWrite: false`，并将渲染顺序提升至 `renderOrder = 1050`（Ribbon）与 `1055`（Arrowhead），高于场内所有车辆、球体与特效层，确保笔划相对于摄像机始终处于最顶层。
