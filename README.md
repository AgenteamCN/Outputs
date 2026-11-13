# Floating Window Schema 修复报告

## 1. 问题排查与答疑

### `PerformanceBoostTunerFloatingWindow` 是谁？是左上角的 Monitor 吗？
**不是左上角的 Monitor。** 它们是三个完全不同的组件：
1. **左上角 HUD Monitor (`CompactTelemetryBarHUD.tsx`)**：位于 `src/ui-core/hud/CompactTelemetryBarHUD.tsx`，是独立挂载在 HUD 上的 4 项精简遥测栏（FPS/刷新率、渲染缩放、物理模拟频率、网络延迟），**底层不是 Floating Window**。
2. **User 性能监控浮窗 (`PerformanceMonitorFloatingWindow.tsx`)**：位于 `src/ui-core/panels/user/PerformanceMonitorFloatingWindow.tsx`（`id: 'performance-monitor'`），属于用户级详细性能监控浮窗（CPU 6 阶段耗时、WASM 物理与 WebGL2 管线），其 `SURFACE_META` 已正确声明 `hideFooter: false`。
3. **Dev 性能加速尾焰调试浮窗 (`PerformanceBoostTunerFloatingWindow.tsx`)**：位于 `src/ui-core/panels/dev/PerformanceBoostTunerFloatingWindow.tsx`（`id: 'performance-boost-tuner'`），是 **Developer Active Tuner**，用于调试零泛光极速模式下的 Boost 尾焰效果（Light Beam / Light Wave）。

### 全量 Floating Window 排查结果
- 扫描 `src/ui-core/panels/{user,dev}/*FloatingWindow.tsx` 共 **36 个** 浮窗模块：
  - **35 个** 均已在 `SURFACE_META` 中显式声明 `hideFooter`（其中 `KeyboardInputOverlayFloatingWindow.tsx` 为 `hideFooter: true`，其余 34 个为 `hideFooter: false`）。
  - **仅 1 个缺失**：`PerformanceBoostTunerFloatingWindow.tsx` 的 `SURFACE_META` 遗漏了 `hideFooter` 字段。
- 同时核查了 `floatingPresets.ts`、`devFloatingPresets.ts`（`makeDevPreset` 默认 `hideFooter: spec.hideFooter ?? false`）以及全仓库所有 `floatingStore.spawnWindow(...)` 调用点，均已显式携带或展开预设中的 `hideFooter`，无其他遗漏。

## 2. 修复内容
- **分支**：基于 `ui` 最新提交 `83c6c4ee` 创建并推送分支 `z/agy/ui/fix-floating-window-schema`。
- **修改文件**：`PerformanceBoostTunerFloatingWindow.tsx`
  - 在 `SURFACE_META` 中显式补充 `hideFooter: false`（与 `devFloatingPresets.ts` 中 `PERFORMANCE_BOOST_TUNER_PRESET` 保持一致）。
- **Verified Bot Commit**：`d69e9471bfdbcd854c778e09d708b676a6c7fa76`（`signature.isValid: true`, `state: "VALID"`）。

## 3. 验证结果
- `pnpm check`（`radar` + `tsc --noEmit`）：通过（`STATUS: GREEN`，1354/1354 TS 文件，0 循环依赖，0 越界导入）。
- `pnpm build`（`vite build`）：通过。
