# RocketCompanion UI Variation & Approval Playlist 对齐报告（Phase 1 核查 + Phase 2 & 3 实施）

**分支**：`origin/z/agy/ui/phase1-tier-orthogonal-variants`
**最新 Verified Bot Commit**：`dcbc99563dd28094d11243fa5447e4c39d88bfee`（在 `a14ce83` 基础上推进）
**校验结果**：`pnpm run check`（`radar` + `tsc --noEmit`）全量 **GREEN**（1390/1390 TS 文件，0 边界/契约/体积错误）。

---

## 一、Phase 1 核查与前序基础补齐

经核查，前序 commit `a14ce83` 已完成 Phase 1 主体（`ProfileTierVariantKind` 扩展 `'pack-imported'`、`GarageSyncTab` 补全 `sync-syncing`/`sync-conflict`/`sync-error`、`AdvancedTab` 5 态正交矩阵、`SignalingProfileSection.snapshotExpanded`、以及 `MenuLayerStageRecipeRouter` 内层 `StoreSnapshotScope` `developerMode` 覆写修复）。

本次进一步排查并补齐了以下前序遗留细节：
1. **`Layer3DevSelfTestRecipe.tsx` 非法 `source` 值修正**：`public-default` 变体原先声明 `signalingProfileStore: { source: 'public-default' }`，与 `SignalingProfileSource`（`'public' | 'custom' | 'imported-bundle' | 'dev-private'`）不一致，已修正为 `source: 'public'`。
2. **折叠面板快照控制补齐**：
   - `DeveloperProfileExportSection.tsx`：接入 `signalingProfileStore.snapshotExpanded` 并同步至 `isExpanded`。
   - `RealisticBgmSection.tsx`：新增可选 `snapshotExpanded?: boolean` prop 并同步至 `isExpanded`。
3. **`Layer4ControllerSettingsRecipe.tsx` Store 作用域兜底**：原先直接读取 `useAppStore` 且忽略 `appStore.developerMode`，改为使用 `useScopedStore('appStore', useAppStore, ...)`，并在未显式传入 `developerMode` prop 时回退读取 `storeDeveloperMode`，确保 `<StoreSnapshotScope>` 注入生效。

---

## 二、Phase 2 评估与实施：Playlist 自动生成与路由层对齐

为避免 `menuLayerLinearPlaylists.ts` 突破 Radar 18KB 硬上限，新增独立构建模块 `src/ui-tuner/pages/menu-layer/menuLayerVariantPlaylistBuilder.ts`：

1. **自动消费 `SETTINGS_TAB_VARIANTS`（10 变体）与全部 `SURFACE_META.variants`**：
   - `Layer2SettingsRecipe`：`USER_LINEAR_PLAYLIST` 自动展开 8 个 User 变体（含 `display-tauri-fullscreen`、`advanced`、`advanced-pack-imported`、`advanced-signed-in`、`advanced-custom-broker` 等）；`DEV_LINEAR_PLAYLIST` 自动展开 2 个 Developer 变体（`advanced-developer`、`developer`）。
   - 自动展开此前被折叠为单步的 L3/L4 Recipe：`Layer3AcousticsEqualizerRecipe`、`Layer3BallTrajectoryRecipe`、`Layer3CameraRecipe`、`Layer3DevSelfTestRecipe`、`Layer3ExhibitionRecipe`、`Layer3KeybindingsRecipe`（注入 `subtab: 'All' | 'Driving' | 'Aerial'`）、`Layer3KickoffTrainingRecipe`、`Layer3TrainingConfigRecipe`、`Layer4StaticTrainingPackBrowseRecipe`（User 映射 `static-training-pack`，Dev 映射 `dev-tem-pack-converter`）。
2. **补全缺席 Recipe 与子组件 Store 依赖变体**：
   - `Layer1TauriConfiguratorRecipe`：在 `menuLayerRouteConfig.ts` 注册 `tauri-configurator` 路由、`MENU_SANDBOX_JUMP_ITEMS` 与 `MenuLayerSecondaryRecipeRoutes.tsx` 挂载，并纳入 `USER_LINEAR_PLAYLIST`。
   - `Layer3DevSandboxesRecipe`：补充 `overview` + 7 个 Isolation Lab Bench（`probes`、`multi-ghost`、`state-replay`、`90s-match`、`goal-replay`、`kickoff-duel`、`demolish-respawn`）共 8 个变体，并更新 `profileTierPolicy` 为 `'user-dev-split'`。
   - `Layer2PlayRecipe`：升级 `profileTierPolicy` 为 `'auth-triad'`，补充 `public-fallback`、`pack-imported`、`dev-private` 三个 `signalingProfileStore` 变体。
   - `Layer3HostMatchRecipe` & `Layer3HostTrainingRecipe`（`menuLayerOnlineRoomSteps.ts`）：补充 `idle-pack-imported` 与 `idle-dev-private` 变体，覆盖 `OnlineRoomIdentityFields.tsx` 在不同 signaling source 下的 UI 渲染。
   - `Layer3DevQuickActionsRecipe`、`Layer4AdvancedAudioRecipe`、`Layer4ControllerSettingsRecipe`：补齐 `SURFACE_META.variants` 并纳入自动展开。
3. **`MenuLayerSecondaryRecipeRoutes.tsx` 路由对齐**：
   - 统一计算 `activeStepStoreOverrides`、`resolvedDeveloperMode` 与 `activeStepSubtab`。
   - 修复剩余 4 处（`acoustics-equalizer`、`camera`、`controller-settings`、`trajectory`）硬编码 `visibilityTier === 'developer'` 覆写 `StoreSnapshotScope` 的盲区。
   - 为 `Layer3KeybindingsRecipe` 透传 `activeTab={activeStepSubtab}`。
4. **`LinearPlaylistStep.variantId` 对齐**：
   - 在 `LinearPlaylistStep` 新增 `variantId?: string`，使 `resolveRouteAuditInfo` 优先使用规范 `variantId`，`findUnifiedStepByLedgerKey` 同时按 `route + (variantId || id)` 精确反查。

---

## 三、跨子组件契约修正与 Radar 自动化守卫（Phase 3 提前落地）

在 `scripts/radar/radarUiAndRepoGuards.ts` 中新增两项阻断级静态守卫，并同步修正了全仓 6 个此前被漏检的 Recipe 声明：
1. **跨子组件 Store 依赖审计（`scanRecipeSubtreeForTierStores`）**：递归扫描 `tier-invariant` Recipe 所引用的 `src/ui-core/recipes/**` 子组件是否读取 `developerMode`、`useCloudProfileSyncStore` 或 `useSignalingProfileStore`。同步修正了 `Layer3DevSelfTestRecipe`（`auth-triad`）、`Layer3JoinRoomRecipe`（`auth-triad`）、`Layer3DevActiveTunersRecipe`、`Layer3DevUnderDevelopmentRecipe`、`Layer4DevSignalingLabRecipe`、`Layer4EventTriggerRulesRecipe`（`user-dev-split`）。
2. **Playlist 覆盖率守卫（`checkMenuPlaylistVariantCoverage`）**：校验所有声明了 `SURFACE_META.variants` 的 `Layer*Recipe.tsx` 必须被 `menuLayerVariantPlaylistBuilder.ts` / `menuLayerLinearPlaylists.ts` / `menuLayerOnlineRoomSteps.ts` 消费，否则 `pnpm run radar` 直接报错阻断。

---

## 四、后续建议（剩余可选项）

- **`uiApprovalLedger.json` 历史分裂键清理**：`reviewLedgerAdapter.ts` 已统一使用 `${surfaceId}::${variantId}`，后续可在实际跑通 UI 审批流并刷新快照哈希时，顺带清理 `uiApprovalLedger.json` 中的历史重复键（如 `recipe:Layer2GarageRecipe` 下的 `car` vs `user-l2-garage-car`）。
