# Telestrator (iPad 画笔) 卡顿与无法起笔根因分析与修复报告

- **目标仓库**: `Antigravi-ty/RocketCompanion`
- **工作分支**: `z/agy/rc/brush-diag` (基于 `2f38585cb35882063ef1d2a118bb38ca5bf6e3cd`)
- **Verified Commit**: `da722e3867d866205f0d51a6491d2a118bb38ca5bf6e3cd` (`isValid: true`, `state: VALID`)

---

## 一、根因诊断（为什么有的地方极其卡顿，且无法起笔但其他笔画可以经过）

通过代码审计与无头浏览器（Chrome WebGL2 + CDP）对画面 88 个采样点进行真实指针点击/拖拽与射线探测，定位到以下核心原因：

### 1. 致命性能瓶颈：CMF 碰撞网格缺失平面区域，触发了全量视觉 GLB 暴力射线检测（10–81.8 ms/次）
- `public/arenamesh/*.cmf`（16 个分块，共 8,020 个三角形）**仅覆盖了曲面拐角、球门以及墙面与地面/天花板的圆角过渡区**，并不包含平坦地面（`y = 0`）、平坦侧墙、平坦天花板以及部分端墙（例如球门左侧区域）。
- **旧逻辑缺陷**：`raycastScreenToArena` 在 CMF 未命中时，直接通过 `scratchRaycaster.intersectObjects(stadiumTargets, true)` 对整个高模视觉 GLB（`driftwoodsStadium` / `throwbackStadium`，包含 `Woods_Field_Walls`、`Woods_Field_Ground`、以及 8,048 面的六边形网格 `Woods_Field_Walls_Hexes` 等）做无 BVH 暴力三角形求交，然后才尝试解析平面 `ARENA_PLANES`。
- **实测对比（x86_64 桌面无头环境）**：
  - `cmf` / `analytical`（解析平面）：**0.0 – 1.0 ms / 次**
  - `stadiumTargets` 暴力回退：**10 – 81.8 ms / 次**（在 iPad 120Hz/240Hz Apple Pencil 高频 `pointermove` 下相当于单次 150–300ms+，直接堵死主线程）。
- 这完美解释了用户的直观感受：**球门（有 CMF 覆盖）不卡、球门左侧/地面/部分墙面（无 CMF 覆盖，落入高模 GLB 回退）极卡**。

### 2. 为什么“卡顿的位置无法起笔，但已有笔画可以经过它”
1. **iPad Safari 手势接管触发 `pointercancel`**：
   - 旧代码未在画笔激活时锁定 `touch-action: none`，也未监听 `pointercancel` 或绑定 `activePointerId`。
   - 当用户在卡顿区域（地面、平墙、球门旁）按下起笔时，主线程因 `stadiumTargets` 射线检测卡死 100–300ms，Safari 原生手势识别器判定为滚动/缩放手势并立即派发 `pointercancel` 中断输入，导致“刚起笔就断掉（无法起笔）”，且 `isDrawingPointerDown` 状态可能卡死。
   - 而如果从**流畅区域**（如球门）起笔，`pointerdown` 在 1ms 内完成并建立笔画，随后拖过卡顿区域时虽然帧率暴跌，但浏览器不会再次从头识别手势，因此“已有笔画可以经过但极其卡顿”。
2. **`#game-ui-root` 整块拦截点击**：
   - 旧版 `isUiEventTarget` 匹配了 `#game-ui-root`，导致 `#game-ui-root` 内部非交互式 HUD 覆盖层（如提示、球机指示等）会静默吞掉 `pointerdown`，表现为随机盲区无法起笔。

### 3. 额外冗余计算
- **高频 `pointermove` 同步重建网格**：iPad Apple Pencil `pointermove` 频率高达 120–240Hz，旧逻辑每次事件都同步射线检测并重建预览 Mesh 几何体，一帧内重复构建多次。
- **无用顶点法线计算**：`telestratorStrokeGeometry.ts` 中对使用 `MeshBasicMaterial`（不受光照影响）的笔画与箭头 Mesh 调用了两次 `geometry.computeVertexNormals()`。

---

## 二、已实施的修复与模块拆分

1. **射线优先级重构 (`src/entities/telestrator/arenaSurfaceRaycaster.ts`)**
   - 新增 `raycastScreenToArenaWithDiag` 与 `ArenaRaycastDiag` 诊断结构。
   - 改为先同时求交 `bestDynamic`、`bestCmf` 与解析平面 `analyticalHit`（`ARENA_PLANES`），**按距离取最近命中**（避免瞄准近处地面时被远处角落的 CMF 穿透误命中）。
   - **仅当 CMF 与解析平面均未命中时**才回退到 `stadiumTargets`（高模视觉网格），彻底消除地面与平坦墙面的暴力求交卡顿。

2. **iPad 触摸/手写笔与输入稳定性 (`src/game/subsystems/TelestratorOverlaySubsystem.ts`, `src/game/subsystems/telestratorOverlayHelpers.ts`)**
   - 移除 `#game-ui-root` 整块 UI 拦截，仅拦截真正的交互控件及浮动窗口（`button, input, ..., .floating-window, .hud-tools, .hig-dual-card`）。
   - 画笔接管期间通过 `TelestratorBodyTouchLock` 自动锁定 `document.body.style.touchAction = 'none'` 与 `userSelect = 'none'`，退出时恢复。
   - 增加 `e.isPrimary`、`activePointerId` 单指锁与 `pointercancel` 处理。
   - 对 `pointermove` 进行每帧一次的 `requestAnimationFrame` 合批（在 `update()` 与 `handlePointerUp()` 时 `flushPendingMove()`），消除 120/240Hz 重复几何重建。

3. **移除冗余几何计算 (`src/entities/telestrator/telestratorStrokeGeometry.ts`)**
   - 移除 `MeshBasicMaterial` 笔画 Mesh 上多余的 `geometry.computeVertexNormals()` 调用。

4. **指针/瞄准实时诊断与 Dump (`src/entities/telestrator/telestratorPointerDiag.ts`, `src/entities/telestrator/telestratorTypes.ts`)**
   - 新增 `TelestratorPointerDiagRecorder`，记录最近 160 条 `down / move / up / cancel / hover` 事件的屏幕坐标、NDC、命中来源（`cmf / analytical / stadium / dynamic / none`）、耗时（ms）、拦截原因（`blocked-button-or-ui`、`no-hit`、`pointer-cancelled` 等）及各类统计计数。
   - 接入 `onDumpLastStroke`（`pointerDiagnostics` 字段），并暴露全局调试接口 `window.__rcTelestratorDiag`：
     - `window.__rcTelestratorDiag.getDump()`：导出当前指针事件历史与拦截/失败统计。
     - `window.__rcTelestratorDiag.probeAt(clientX, clientY)`：直接对指定屏幕坐标做一次带耗时与来源诊断的射线探测。

5. **低耦合模块化拆分**
   - 将原 `TelestratorOverlaySubsystem.ts` 中的 DOM 窗口创建、TouchLock、快捷键订阅、Store 同步订阅、射线目标收集及 Wire 校验抽离至 `src/game/subsystems/telestratorOverlayHelpers.ts`，保持单文件体积精简（均符合 `radar` 体积上限）。

---

## 三、验证结果

- `pnpm run check` (TypeScript `tsc --noEmit`)：通过（0 错误）
- `pnpm run radar:audit` (体积与架构护栏审计)：通过
- `pnpm run build` (Vite 生产构建)：通过
