# Track A（3D Telestrator & Drawing Sandbox）问题定位、修复与使用说明

## 一、为什么之前“画不出 Stroke、看不到 1-6、永远像 Direction Adjust”？

经过对 `origin/z/agy/rc/track-a-telestrator-sandbox` 的排查，之前确实不是单纯“没做完”，而是**模式状态机与 HUD/相机管线之间有 4 处关键断链**，导致功能在运行时直接失效：

1. **最大元凶：`DrawingSandboxMode` 在第 1 帧就自动退出了（导致画不出 Stroke + 永远只能拖动旋转视角）**
   - 此前点击 Sandboxes 菜单里的 `Drawing Sandbox` 只调用了 `modeManager.setMode('drawing-sandbox')`，**没有同步写入 `runtime.match.state.mode = 'drawing-sandbox'` 和 `appStore.matchMode`**。
   - 结果下一帧 `GameModeManager.getActiveMode()` 读到 `match.state.mode` 还是 `'freeplay'`，立刻把模式切回 `FreeplayMode`，触发了 `DrawingSandboxMode.onLeave()`：
     - 指针事件被解绑（`handlePointerDown/Move/Up` 全部失效，所以画不出任何 Stroke）；
     - `freeController.shouldSuppressDrag` 被清空，于是 `FreeCameraController` 接管了鼠标左键拖拽，表现出来就是**无论点 Pen 还是 Compass，拖动鼠标都在旋转视角**，看起来就像“永远处于 Direction Adjust”。
2. **两侧 1-6 玩家条（`EdgePlayerHudStrip`）被 `InGameHUD` 过滤掉了**
   - `DrawingSandboxMode` 虽然推送了 6 辆车的 Roster 到 `globalHudTransientChannel`，但：
     - `InGameHUD.tsx` 的 `showEdgeRosterStrips` 只对 `match` / `exp-multi-bots` / `replay-viewer` 开放，没包含 `drawing-sandbox`；
     - `TelemetryStage.ts` 与 `hudRosterResolverPipeline.ts` 每帧都会重算并覆盖 Roster，因为缺少 `drawing-sandbox` 分支，每帧都把 6 辆车冲刷成空数组；
     - `useReplayPovSlotHotkeys.ts` 点击卡片时也没路由给 `useTelestratorStore.triggerSelectCameraSlot`。
3. **相机视角切不到 2-6 号车**
   - `GameCameraSubsystem.resolveFrameCameraMetadata()` 此前只读取 `replayPovCarIndex` 或 `activeCar (0)`，没有读取 `drawing-sandbox` 下的 `camera.focusedCarIndex`，导致即使切到 2-6 号车，镜头也依然锁死在 1 号车（index 0）。
4. **关于 `[FLOATING WINDOW SCHEMA ERROR]` 报错**
   - 报错来自 `PerformanceBoostTunerFloatingWindow.tsx`（`alpha` 分支上遗漏了 `hideFooter` 字段），与 `TelestratorToolbarFloatingWindow.tsx` 无关（后者已显式声明 `hideFooter: true`）。如您所说，这部分交给 UI 团队清理即可。

---

## 二、本次修复内容（分支 `z/agy/rc/track-a-telestrator-fix`）

已基于 `origin/z/agy/rc/track-a-telestrator-sandbox` 创建新分支 `z/agy/rc/track-a-telestrator-fix` 并提交 Verified Bot Commit `ddbd338`：

- `DrawingSandboxMode.ts`：
  - 进入模式时同步设置 `runtime.match.state.mode = 'drawing-sandbox'` 与 `useAppStore.getState().setMatchMode('drawing-sandbox')`；
  - 进入时强制初始化为 **Draw Mode（`drawingEnabled = true`, `directionAdjustActive = false`）**；
  - 将工具栏浮窗初始位置下移至 `{ x: 16, y: 280 }`，避免与左上角 `EdgePlayerHudStrip` 重叠。
- `SimulationStage.ts` & `appStoreTypes.ts`：
  - 将 `'drawing-sandbox'` 加入 `CUSTOM_STEP_MODES` 与 `AppState['matchMode']` 联合类型，防止被 `SimulationStage` 误判或覆写。
- `hudRosterResolverPipeline.ts` & `TelemetryStage.ts`：
  - 为 `drawing-sandbox` 补全 6 辆车（Blue 1-3 / Orange 1-3）Roster 解析，并根据 `camera.isFreeCam` / `camera.focusedCarIndex` 高亮当前聚焦车辆。
- `GameCameraSubsystem.ts`：
  - 在 `drawing-sandbox` 模式下将 `obsTargetIdx` 绑定到 `camera.focusedCarIndex`，确保 1-6 切人真实跟随对应车辆。
- `InGameHUD.tsx`、`useReplayPovSlotHotkeys.ts`、`Layer3DevSandboxesRecipe.tsx`：
  - 在 `drawing-sandbox` 模式下显示左右两侧 `EdgePlayerHudStrip`，支持直接点击卡片切人；
  - 从 Sandboxes 列表无论是点击整行还是点击右侧启动按钮，都会完整进入 `DrawingSandboxMode`。

**验证结果**：
- `pnpm run check`（Radar 2.0 架构检查 + TypeScript 严格检查）：**0 Error 通过**；
- `pnpm run build`（Vite 生产构建）：**通过**；
- 修改文件 `prettier --check`：**通过**。

---

## 三、Drawing Sandbox 操作指南

1. **进入方式**：打开菜单 -> `Developer` -> `Sandboxes` -> 点击 `Drawing Sandbox`（会在场地生成 6 辆车，默认进入 **7 号自由视角 + Draw Mode**）。
2. **画笔绘制**：
   - 确认工具栏最上方 **Pen（画笔图标）** 为绿色高亮（进入时默认激活）；
   - 直接在 3D 场地（地面、侧墙、斜角墙、天花板、球门内壁）按住鼠标左键拖动即可绘制；
   - 工具栏依次支持：
     - **模式切换**：Draw Mode（画笔）/ Direction Adjust（指南针图标，旋转视角朝向）/ Restore POV（偏移归零）；
     - **笔型**：实线（Solid）/ 虚线（Dashed）/ 战术箭头（Arrow）；
     - **3 档 Preset**：`100` / `70` / `50`（对应 100% / 70% / 50% 透明度，可在 `Developer Settings -> Telestrator` 调整粗细与透明度）；
     - **6 种颜色**：深灰、白、红、黄、亮绿、暗绿；
     - **辅助操作**：Aerial Drop Stems（空中下垂虚线开关）、Undo（或 `Ctrl+Z` / `Cmd+Z`）、Clear All。
3. **视角与切人**：
   - 键盘 `1`~`6` 或直接点击屏幕左右两侧的 `Blue 1-3` / `Orange 1-3` 卡片：切换到对应车辆 POV（同时自动把 Direction Adjust 偏移归零）；
   - 键盘 `7`：切回自由视角（可用 `WASD` + `Space/I` 上升 + `V/M` 下降移动位置）；
   - 键盘 `C`：在 1~6 号车 POV 与 7 号自由视角之间循环切换；
   - **Direction Adjust**：点击工具栏指南针图标激活后，在画面左键拖拽可固定当前 XYZ 位置并偏移朝向（切换到车视角后偏移会持久保留）；再次点击指南针或点击 Pen 可切回画图；点击 Restore POV（或按 `1`-`7` / `C`）可一键归零。

---

## 四、当前仍待后续对齐/优化的部分

1. **UI 团队接管部分**：
   - 浮窗当前为默认样式，后续由 UI 团队调整为更紧凑的居中悬浮条/外置指示器，以及修复 `alpha` 上 `PerformanceBoostTunerFloatingWindow.tsx` 缺少 `hideFooter` 的报错。
2. **网络同步联调（待 Track D 收敛后）**：
   - `TelestratorWireCodec.ts` 与 Hash Ack 校验已完成单机闭环，后续接入 `RoomReplicationBus` 的 Reliable 控制信道即可在视角同步前提下实现多端绘制同步。
