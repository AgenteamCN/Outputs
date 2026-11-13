# RocketCompanion UI SF Mono 字体栈修复与分支清理报告

## 1. 已合入 `alpha` 的历史分支清理

在确认无未合并提交（`git rev-list --count origin/alpha..<branch> == 0`）且无关联 Open PR 后，已安全清理 **26 个** 确认合并到 `origin/alpha` 的 `z/agy/*` 历史分支（保留了 `alpha`、`a2`、`z/user/*` 及未合入分支）：

- `z/agy/fix-profile-sync` (`80e36ae`)
- `z/agy/rc/fix-hud-telemetry-and-transport-rtt` (`daf3361`)
- `z/agy/rc/fix-input-starvation-and-host-loopback` (`00a5442`)
- `z/agy/rc/fix-join-room-state-and-ui` (`4db613a`)
- `z/agy/rc/fix-kickoff-replay-freeze` (`39cdc82`)
- `z/agy/rc/fix-lobby-goal-reset-and-memory-footprint` (`01b2644`)
- `z/agy/rc/fix-multiplayer-reconcile-and-lobby-diag` (`2634cf2`)
- `z/agy/rc/fix-online-host-probe-trajectory` (`69b9c50`)
- `z/agy/rc/fix-room-ui-and-setstate-in-render` (`7c0f450`)
- `z/agy/rc/hex-thickness-tuning` (`7e776ea`)
- `z/agy/rc/light-trail-effects` (`77f1996`)
- `z/agy/rc/multiplayer-fixes-octopus` (`6dd92f1`)
- `z/agy/rc/multiplayer-octopus-merge` (`b10419f`)
- `z/agy/rc/performance-boost-tuner` (`391df02`)
- `z/agy/rc/radar-output-compact` (`1730391`)
- `z/agy/rc/replay-boxcars-prober` (`e492dab`)
- `z/agy/rc/replay-smooth-and-viewer-unify` (`5453721`)
- `z/agy/rc/repro-multiplayer-room` (`c891cdc`)
- `z/agy/rc/rlrc-replay-fixes` (`677b64b`)
- `z/agy/rc/tarui/configurator-refinements` (`ca3b841`)
- `z/agy/rc/tauri/dock-tray-lifecycle-fix` (`9de427e`)
- `z/agy/rc/tauri/icon-and-build-id` (`3fa2386`)
- `z/agy/rc/tauri/tray-and-promotion` (`1cfe096`)
- `z/agy/rc/unify-multiplayer-repro-harness` (`2d4a607`)
- `z/agy/ui/rework-fixes` (`af69934`)
- `z/agy/ui/triage-infra` (`cb7ed4d`)

---

## 2. SF Mono 字体栈修复详情

基于 `origin/alpha` 创建了新分支 **`z/agy/ui/fix-sf-mono-font-stack`**，完成以下解耦与修复：

### 2.1 全局 CSS Token 修正
- `ui-system.css` & `ui-tuner.css`：
  - 将 `--font-mono` 调整为 `ui-monospace, SFMono-Regular, "SF Mono", "JetBrains Mono", Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace`，确保 Apple 设备优先匹配系统 SF Mono，非 Apple 设备自动 fallback 到 JetBrains Mono。
  - 移除已废弃的 `--font-jetbrains` 变量（零历史包袱）。

### 2.2 `ui-core/tokens` 集中化字体 Token（消除多处重复定义）
- 新增 `typography.ts`（位于 `tokens` 层，严格遵守 `config <- tokens <- stores <- ...` 单向分层架构），导出 `MONO_FONT_STACK` 与 `APPLE_SANS_FONT_STACK`。
- 移除 `QuickNotification.tsx` 与 `dialogueStyles.tsx` 中重复定义的 `MONO_FONT_STACK`，并将 `QuickNotification.tsx`、`QuickNotificationToastCard.tsx`、`QuickNotificationPage.tsx`、`Dialogue.tsx` 统一改为直接从 `tokens/typography` 导入。
- 更新 `CircularBoostGauge.tsx`、`EventImpulseScatterChart.tsx`、`telemetrySparklinePainter.ts` 的 Canvas `ctx.font` 使用集中化的 `MONO_FONT_STACK`。
- 更新 `mainMenuStyleStore.ts`，移除本地 `"JetBrains Mono"` 前置栈，直接复用 `APPLE_SANS_FONT_STACK` 与 `MONO_FONT_STACK`。

### 2.3 NameTag 默认字体与 Renderer 兜底修正
- `nameTagConfig.ts`：将 `fontFamily` 与 `singleCharFontFamily` 默认值从 `'jetbrains-mono'` 切换为 `'sf-mono'`，并将 `resolveNameTagFontFamilyCss` 中 `sf-mono` 分支的 `ui-monospace, SFMono-Regular` 移至首位。
- `nameTagOffscreenRenderer.ts`：兜底字体栈同步调整为 `ui-monospace, SFMono-Regular, "SF Mono", "JetBrains Mono", Menlo, Monaco, Consolas, monospace`。

### 2.4 统一替换 `var(--font-jetbrains)` 为 `var(--font-mono)`
- `ui-core`：`PlayModeCard.tsx`、`playModeCardHelpers.tsx`、`EdgePlayerHudStrip.tsx`、`StepperScoreboardHUD.tsx`、`InGameHUD.tsx`、`TemporalReplayFooterHUD.tsx`、`CenterUpperInfoHUD.tsx`、`TakeoverCountdownHUD.tsx`。
- `ui-tuner`：`MainMenuTypographyAndOutlineSection.tsx`、`TakeoverArchetypesSection.tsx`、`TakeoverStageCanvas.tsx`、`PlayCardWatermarkAndColorTuner.tsx`、`CenterUpperStageCanvas.tsx`、`CenterUpperArchetypesSection.tsx`、`HudSpatialChoreographyStageCanvas.tsx`、`DialogueSlotControlPanel.tsx`、`DialogueButtonManagerColumn.tsx`。

---

## 3. 验证与提交结果

- **架构与类型检查 (`pnpm run check`)**：`radar` + `tsc --noEmit` 全部通过（1354/1354 TS 文件，0 循环依赖，0 越层引用，0 值重导出，状态 `GREEN`）。
- **生产构建 (`pnpm run build`)**：`vite build` 成功（3732 modules transformed）。
- **Verified Commit**：
  - 分支：`z/agy/ui/fix-sf-mono-font-stack`
  - Commit SHA：`2dfd5c9e98410d4635de9754761b0b1fee1a5d49`
  - 签名状态：`isValid: true` (`VALID`)
  - URL：https://github.com/Antigravi-ty/RocketCompanion/commit/2dfd5c9e98410d4635de9754761b0b1fee1a5d49
