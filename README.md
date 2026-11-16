# RocketCompanion UI Variation & Approval Pipeline Deep Scan 验收报告

> 分支：`origin/z/agy/ui/phase1-tier-orthogonal-variants`
> 最新验证提交：`d309d1d`（Verified Bot Commit on top of `6c1c7a8`）

---

## 一、现在到底需要验收什么？（不仅仅是“系统没坏”）

Phase 1–3 完成后，UI-Tuner 的**审批覆盖面从过去的手工子集扩展为 181 个精确去重单元（User 87 + Dev 94）**。这次真正需要人工视觉验收的是以下 **4 类新增可见状态**：

1. **4 态正交 Auth / Bundle / Platform 矩阵**：
   - `Settings → Advanced`（5 态：`anonymous` / `pack-imported` / `signed-in` / `custom-broker` / `developer`），其中 `SignalingProfileSection` 与 `DeveloperProfileExportSection` 在快照中默认展开（`snapshotExpanded: true`），可直接审阅 `.rctbnspib` 导入/导出与中继探测 UI。
   - `Garage → Sync`（新增 `sync-syncing`、`sync-conflict` 3-way merge 冲突弹窗、`sync-error` 错误态，且 `sync-sign-in` 已修正为 `pack-imported`）。
   - `Settings → Display` 的 **`settings:display-tauri-fullscreen`**（Tauri 桌面端 Window Mode：`Fullscreen` / `Borderless` / `Windowed`，本次已修复其浏览器快照不生效的 Bug）。
2. **此前被折叠的 9 个 L3/L4 Recipe + Layer1 Tauri Configurator + Dev Sandboxes（8 态）**：
   - 包括 `Layer3AcousticsEqualizerRecipe`、`Layer3BallTrajectoryRecipe`、`Layer3CameraRecipe`、`Layer3DevSelfTestRecipe`、`Layer3ExhibitionRecipe`、`Layer3KeybindingsRecipe`、`Layer3KickoffTrainingRecipe`、`Layer3TrainingConfigRecipe`、`Layer4StaticTrainingPackBrowseRecipe`（User/Dev 双路由拆流）、`Layer3DevSandboxesRecipe`（overview + 7 labs）、`Layer1TauriConfiguratorRecipe`。
3. **子组件驱动的 Signaling Source 与 Tier 分流**：
   - `Layer2PlayRecipe`、`Layer3HostMatchRecipe`、`Layer3HostTrainingRecipe`、`Layer3JoinRoomRecipe` 的 `public` vs `imported-bundle` vs `dev-private` 联机入口差异。
   - `Layer3DevQuickActionsRecipe`、`Layer4AdvancedAudioRecipe`、`Layer4ControllerSettingsRecipe` 的 User/Dev 分流与 `resolvedDeveloperMode` 路由对齐。
4. **统一的 Review Queue 账本状态**：
   - 打开 UI-Tuner → Review Queue，确认所有新展开的变体都能按 `${surfaceId}::${variantId}` 精确跳转并渲染，且无旧分裂键残余。

---

## 二、Deep Scan 全量核查结果

通过 Vite SSR 实际加载运行时模块（`menuLayerLinearPlaylists.ts`、`floatingWindowsLinearPlaylists.ts`、`surfaceCalibration.ts`、`reviewLedgerAdapter.ts` 及全部 33 个顶层 `Layer*Recipe.tsx`），核查结果如下：

| 核查维度 | 扫描结果 | 状态 |
| :--- | :--- | :---: |
| **Playlist 总量与去重** | `ALL_UNIFIED_APPROVAL_PLAYLIST` 共 **181** 步（User **87** / Dev **94**）；重复 `step.id` = **0**；重复 `unitKey` = **0** | PASS |
| **路由与审计回退** | `resolveRouteAuditInfo` 意外回退到 `Layer1MainMenuRecipe` = **0** | PASS |
| **Floating Windows 覆盖** | 38 个 `SURFACE_META` 表面 / 64 个单元（User 7 + Dev 57），`SURFACE_META.variants` 覆盖率 **100%**（缺失 = 0） | PASS |
| **Ledger 历史孤立/分裂键** | `uiApprovalLedger.json` 中不存在任何 Playlist 无法命中的孤立 surface 或孤立 variant key（孤立数 = **0**） | PASS |
| **待人工首次审阅的表面** | `recipe:Layer1TauriConfiguratorRecipe` 及 3 个 Floating 窗口（`performance-boost-tuner`、`dynamic-training-pack`、`telestrator-toolbar`）尚未录入 `uiApprovalLedger.json`（队列中正常归为 `deferred` 待审） | INFO |
| **在线/主菜单命名体系差异** | `Layer1MainMenuRecipe`（用 `user-l1-*` / `dev-l1-*`）、`Layer2ManageRoomRecipe`、`Layer3HostMatchRecipe`、`Layer3HostTrainingRecipe`、`Layer3JoinRoomRecipe` 共 29 个 `SURFACE_META` 原始 `variantId` 未按原名进入 Playlist，而是通过 `menuLayerOnlineRoomSteps.ts` 的 18+ 扩展步覆盖（`radarUiAndRepoGuards.ts` 对这 5 个文件做了白名单跳过） | INFO |

---

## 三、本次发现并修复的隐蔽 Bug（Commit `d309d1d`）

在核查所有 `storeOverrides` 顶层键（`appStore`, `cloudProfileSyncStore`, `experimentalMultiBotsStore`, `kickoffTrainingStore`, `signalingProfileStore`, `subsystemIsolationLabStore`, `tauriRuntime`）的消费情况时，发现 `tauriRuntime` 未接入快照作用域：

1. **`DisplayTab.tsx` & `DisplayRefreshRateCard.tsx` 忽略 `storeOverrides.tauriRuntime`**
   - **问题**：`DisplayTab.tsx` 为 `settings:display-tauri-fullscreen` 声明了 `storeOverrides: { tauriRuntime: { isTauri: true, platform: 'windows', windowMode: 'fullscreen' } }`，但组件内部直接调用 `isTauriRuntime()`（检测 `window.__TAURI_INTERNALS__`），导致在浏览器中运行的 UI-Tuner 永远判定为非 Tauri，无法渲染 Window Mode 选择器。
   - **修复**：在 `src/ui-core/recipes/settings/tabs/DisplayTab.tsx` 与 `src/ui-core/compounds/DisplayRefreshRateCard.tsx` 中接入 `useStoreSnapshotOverride('tauriRuntime')`，优先读取快照覆盖的 `isTauri`、`platform` 与 `windowMode`。
2. **`Layer1TauriConfiguratorRecipe.tsx` 挂载时污染 UI-Tuner 全局主题**
   - **问题**：组件挂载后 `refreshFromBackend()` 会无条件调用 `applyDocumentTheme(state.config.themeMode)` 修改 `document.documentElement` 的 `dark`/`light` class。
   - **修复**：在 `src/ui-core/recipes/Layer1TauriConfiguratorRecipe.tsx` 中改为仅当 `propIsLight === undefined`（非 UI-Tuner 托管模式）时才同步全局 `document.documentElement` 主题。

---

## 四、工具链与验证结果

- 环境：`pnpm` v12.21.0 已安装，`pnpm install --frozen-lockfile` 完成。
- `pnpm run check`（Radar + `tsc --noEmit`）：**GREEN**（1390/1390 TS 文件，0 循环依赖，0 边界违规）。
- `pnpm run radar:audit`：**GREEN**。
- `pnpm run build`：**PASS**（Vite 生产构建成功）。
