# RocketCompanion UI Variation 对齐与 Phase 1 交付报告（更新版）

- **基准分支**: `origin/alpha` @ `ff3076a`（在原报告 `8b6b0a2` 之后已合入 PR #22 `git-hook-auto-reload` 与 PR #23 `tuner-startup-optimization`）
- **新分叉 UI 分支**: `z/agy/ui/phase1-tier-orthogonal-variants`
- **Verified Bot Commit**: [`a14ce8350c655078713f90b26b6de1db988163bc`](https://github.com/Antigravi-ty/RocketCompanion/commit/a14ce8350c655078713f90b26b6de1db988163bc)（`signature.state: VALID`，`antigravity-fine-grained-token[bot]`）
- **基线门禁**: `pnpm run check`（`radar` 1389/1389 TS + `tsc --noEmit`）全绿通过

---

## 一、前序基础状态核查结论（`alpha @ ff3076a`）

1. **原报告中的未修复项在 `ff3076a` 仍然存在**：
   - `ProfileTierVariantKind` 仍只有 3 态（`'anonymous' | 'signed-in' | 'developer'`），无法表达「已导入 `.rctbnspib` Pack 但未登录」状态。
   - `menuLayerLinearPlaylists.ts` 第 7 行导入 `SETTINGS_TAB_VARIANTS` 但未使用，`USER_SETTINGS_QUICK_TABS` 仅展开 4 个扁平步骤。
   - `MenuLayerStageRecipeRouter.tsx`（第 186、216、253 行）在 `settings`、`manage-room`、`join-room` 内层 `<StoreSnapshotScope>` 硬编码 `developerMode: visibilityTier === 'developer'`，会覆写外层 `activePlaylistStep.storeOverrides.appStore.developerMode`。
2. **新增扫描细节（原报告未覆盖）**：
   - `MenuLayerSecondaryRecipeRoutes.tsx`（第 51、65、242、259 行）同样存在内层 `<StoreSnapshotScope>` / prop 直接传入 `visibilityTier === 'developer'`，需在 Phase 2 一并改为继承 `activeStepStoreOverrides`。
   - `menuLayerRouteConfig.ts` 第 232 行 `LinearPlaylistStep.profileTier` 硬编码了 `'anonymous' | 'signed-in' | 'developer'`，未复用 `surfaceMetaTypes.ts` 的 `ProfileTierVariantKind`。
   - `SignalingProfileSection.tsx` 之外，`RealisticBgmSection.tsx` 与 `DeveloperProfileExportSection.tsx` 也使用局部 `useState(false)` 控制折叠面板，可在后续按需引入 `snapshotExpanded`。

---

## 二、Phase 1 已完成交付内容（`z/agy/ui/phase1-tier-orthogonal-variants`）

1. **统一 4 态 Tier 与正交轴元数据模型**
   - `src/ui-core/manifest/surfaceMetaTypes.ts`：
     - 将 `ProfileTierVariantKind` 扩展为 `'anonymous' | 'pack-imported' | 'signed-in' | 'developer'`。
     - 在 `SurfaceVariantMeta` 新增可选正交元数据字段：`authTier`（`'anonymous' | 'signed-in' | 'developer'`）、`bundleSource`（`'public' | 'custom' | 'imported-bundle' | 'dev-private'`）、`runtimePlatform`（`'web' | 'tauri'`）。
   - `src/ui-tuner/pages/menu-layer/menuLayerRouteConfig.ts`：
     - `LinearPlaylistStep.profileTier` 改为引用 `ProfileTierVariantKind`，消除类型分裂。
   - `src/ui-core/recipes/settings/tabs/DisplayTab.tsx`：
     - 为 `settings:display` 与 `settings:display-tauri-fullscreen` 补充 `runtimePlatform: 'web' | 'tauri'`。

2. **Garage Sync 变体矩阵补全（`src/ui-core/recipes/garage/garageSyncVariants.ts`）**
   - 将 `sync-sign-in` 修正为 `profileTier: 'pack-imported'`（`authTier: 'anonymous'`，`bundleSource: 'imported-bundle'`）。
   - 为全部分支补齐 `signalingProfileStore` 快照覆写，消除全局残留状态干扰。
   - 新增 3 个关键状态变体：`sync-syncing`（同步中）、`sync-conflict`（含 2 条 3-Way Merge 冲突项，触发 `GarageCloudProfileSlotCard` 内联冲突解决面板）、`sync-error`（中继超时错误横幅）。

3. **Settings AdvancedTab 5 态正交变体与折叠面板快照控制**
   - `src/ui-core/stores/signalingProfileStore.ts` 与 `src/ui-core/recipes/settings/components/SignalingProfileSection.tsx`：
     - 新增 `snapshotExpanded?: boolean` 状态字段，`SignalingProfileSection` 通过 `useScopedStore` 初始化并同步 `isExpanded`，支持静态快照直接展开导入/PIN/探测面板。
   - `src/ui-core/recipes/settings/tabs/AdvancedTab.tsx` 与 `src/ui-core/recipes/Layer2SettingsRecipe.tsx`：
     - 扩展为 5 个正交变体：`settings:advanced-anonymous`、`settings:advanced-pack-imported`（已导入未登录 + 展开面板）、`settings:advanced-signed-in`（已导入已登录 + 远程部署开启）、`settings:advanced-custom-broker`（自建中继 + 展开面板）、`settings:advanced-developer`，并全部注册进 `SETTINGS_TAB_VARIANTS`（Settings 现共 10 个变体）。

4. **Router 层 `developerMode` 覆写修复（前置解除 Phase 2 阻塞）**
   - `src/ui-tuner/pages/menu-layer/MenuLayerStageRecipeRouter.tsx`：
     - 提取 `resolvedDeveloperMode`（优先读取 `activeStepStoreOverrides?.appStore?.developerMode`，未指定时再回退 `visibilityTier === 'developer'`），修复 `settings`、`manage-room`、`join-room` 内层 `<StoreSnapshotScope>` 锁死 `developerMode` 的 Bug。

---

## 三、更新后的后续推进计划（Phase 2 & Phase 3）

### Phase 2：Playlist 自动生成与路由层彻底对齐
1. **重构 `menuLayerLinearPlaylists.ts`**：
   - 自动消费 `Layer2SettingsRecipe` 的 `SETTINGS_TAB_VARIANTS`（10 变体）及另外 9 个已声明 `SURFACE_META.variants` 却被折叠的 L3/L4 Recipe。
   - 补全 `Layer1TauriConfiguratorRecipe` 与 `Layer3DevSandboxesRecipe`（6 Tabs）遍历条目。
   - 按子组件实际 Store 依赖补全 `Layer2PlayRecipe`、`Layer3HostMatchRecipe` / `Layer3HostTrainingRecipe`（signaling source 差异）、`Layer3DevQuickActionsRecipe`、`Layer4AdvancedAudioRecipe`、`Layer4ControllerSettingsRecipe` 的变体。
2. **对齐 `MenuLayerSecondaryRecipeRoutes.tsx`**：
   - 传入 `activeStepStoreOverrides` 并使用相同的 `resolvedDeveloperMode` 逻辑，消除剩余 4 处 `visibilityTier === 'developer'` 覆写盲区。
3. **按需补齐 Settings 其余折叠面板快照控制**：
   - 为 `RealisticBgmSection` / `DeveloperProfileExportSection` 增加类似 `snapshotExpanded` 的快照支持（如需要审阅展开态）。

### Phase 3：Ledger 键统一与 Radar 自动化守卫
1. **统一 `reviewLedgerAdapter.ts` unitKey**：
   - 严格使用 `${surfaceId}::${variant.variantId}`，清理 `uiApprovalLedger.json` 历史分裂键（如 `car` vs `user-l2-garage-car`）。
2. **增强 `scripts/radar/radarUiAndRepoGuards.ts`**：
   - 新增「Playlist 覆盖率守卫」：校验所有 `Layer*Recipe.tsx` 的 `SURFACE_META.variants` 100% 出现在 `ALL_UNIFIED_APPROVAL_PLAYLIST` 中。
   - 新增「跨子组件 Store 依赖审计」：递归扫描 Recipe 引用的子组件是否读取了 `developerMode`、`useCloudProfileSyncStore`、`useSignalingProfileStore` 或 `isTauriRuntime`，防止顶层 `tier-invariant` 声明与子组件脱节。
