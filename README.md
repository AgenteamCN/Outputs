# UI Audit Fix Report · `z/agy/ui/tier-variation-and-floating-icon`

- **Base Branch / Commit**: `ui` (`8d19408`)
- **Working Branch**: `z/agy/ui/tier-variation-and-floating-icon`
- **Verified Bot Commit**: `4fbfebd57facd313cee3f1daffb9f9f8df1f9a9e` (`Signature: VALID`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`)
- **Verification**: `pnpm check` (1389/1389 TS files, `tsc --noEmit`, Radar hard boundaries) GREEN; `pnpm build` PASS.

---

## 1. 关于“审批结论在云端还是本地”的解答

当前 UI Tuner 的审批状态采用 **“Git 基线账本 + 浏览器本地覆盖”** 的双层机制：
1. **Git 仓库基线账本**：`src/ui-core/manifest/uiApprovalLedger.json` 随代码库提交（不依赖额外云端存储），记录每个 surface / variant 的 `status`、`visualAstHash`、`fileVisualHashes` 与 `note`。
2. **浏览器本地运行态**：`src/ui-tuner/components/review/reviewLedgerStore.ts` 将评审过程中的 `decisionsMap`、`activeBucketFilter` 持久化在浏览器 `window.localStorage`（key: `rc_ui_tuner_review_ledger_v1`）。
3. **为什么会出现“改了 fix 好像没同步过来”**：如果本地 `localStorage` 里保留着旧的 `rejected` 结论，或者分支切换后没有通过 Review Exporter 导出/更新 `uiApprovalLedger.json`（或清空本地 stale decision），UI Tuner 仍会优先读取 `localStorage` 中的历史决策。后续只需在修完问题后同步清理/导出到 `uiApprovalLedger.json` 即可保持一致。

---

## 2. 问题一：Auth / Profile Tier (`anonymous` / `signed-in` / `developer`) 自动遍历与破坏性规范约束

### 根因与架构调整
- **现状问题**：
  - `Layer2GarageRecipe` 的 `sync` 标签页在 `USER_LINEAR_PLAYLIST` 中只有单一的 `user-l2-garage-sync` 节点，没有将 `anonymous`、`signed-in`、`developer` 三种状态自动展开为独立可遍历页面，导致在默认公开沙盒环境下只能看到 `UnderConstructionPlaceholder` 或单一状态。
  - `Layer3DevRemoteProfileRecipe` 同样在 `DEV_LINEAR_PLAYLIST` 中只有单一节点，且未读取 `StoreSnapshotScope` 的 `activeTab` 覆盖。
  - `Panel.tsx` 与 `PanelFooter.tsx` 直接调用了 `useCloudProfileSyncStore(...)` 而非 `useScopedStore('cloudProfileSyncStore', ...)`，导致 `<StoreSnapshotScope>` 无法穿透到 Header/Footer 的远程会话态。
  - `MenuLayerStageRecipeRouter.tsx` 没有自动将当前 `activePlaylistStep.storeOverrides` 注入 `<StoreSnapshotScope>`。
- **破坏性契约与 Radar / 运行时双端强校验**：
  1. **Schema 破坏性更新**（`src/ui-core/manifest/surfaceMetaTypes.ts`）：
     - 新增 `ProfileTierVariantKind = 'anonymous' | 'signed-in' | 'developer'` 与 `SurfaceProfileTierPolicy = 'tier-invariant' | 'user-dev-split' | 'auth-triad'`。
     - 所有 33 个 `Layer*Recipe.tsx`（及 `StaticTrainingShotInspectorDrawer.tsx`）的 `SURFACE_META` 均已更新，强制声明 `profileTierPolicy`。
  2. **Radar 静态守卫**（`scripts/radar/radarUiAndRepoGuards.ts`）：
     - 任何 `Layer*Recipe.tsx` 若缺失 `profileTierPolicy` 直接报 `[SURFACE SCHEMA VIOLATION]`。
     - 凡引用 `useCloudProfileSyncStore` 或 `developerMode` 的 Recipe，若声明为 `'tier-invariant'` 直接报 `[SURFACE TIER CONTRACT VIOLATION]`；声明为 `'auth-triad'` 时必须提供带 `profileTier` 与 `storeOverrides` 的 `variants`。
  3. **运行时守卫**（`src/ui-core/navigation/floatingWindowSchemaValidator.ts` + `floatingSurfaceRegistry.ts`）：
     - 新增 `validateRecipeProfileTierPolicy`，在发现未按规范声明 `profileTierPolicy` 或 `auth-triad` 缺失 `variants` 时立即发出显著 Schema 错误告警。
  4. **自动遍历与自动作用域注入**：
     - 拆分出 `src/ui-core/recipes/garage/garageSyncVariants.ts`，定义 `sync-anonymous`、`sync-signed-in`、`sync-developer` 三个独立变体及其 `storeOverrides`。
     - `menuLayerLinearPlaylists.ts` 自动将 Garage Sync 展开为 `user-l2-garage-sync-anonymous`、`user-l2-garage-sync-signed-in`（User 队列）与 `dev-l2-garage-sync-developer`（Dev 队列），同时将 `Layer3DevRemoteProfileRecipe` 的 3 个变体展开进 `DEV_LINEAR_PLAYLIST`。
     - `MenuLayerStageRecipeRouter.tsx` 顶层自动包裹 `<StoreSnapshotScope overrides={activeStepStoreOverrides ?? {}}>`，后续新增任何带 `storeOverrides` 的页面均可零样板代码自动生效。

---

## 3. 问题二：Floating Window 与 Active Tuners 图标/颜色不一致

### 为什么直接基于当前分支修复，而不是从 `alpha` 另开分支？
- 经 `git diff --stat alpha..HEAD -- src/ui-core/navigation/ src/ui-core/panels/ src/ui-core/recipes/dev/ src/ui-core/layout/UniversalThreeStageShell.tsx` 核对，当前 `ui` 分支与 `alpha` 分支在 Floating Window 壳层、导航、目录与面板实现上 **完全一致（0 差异）**。
- 因此无需额外从 `alpha` 拆出独立分支造成合并成本，直接在本次 `z/agy/ui/tier-variation-and-floating-icon` 中一并彻底修复，合并后即可同步回 `ui` / `alpha`。

### 根因与修复内容
- **根因**：
  1. 图标与颜色分散在三处：`devHubSurfaceCatalog.ts` / `devHubExperimentalCatalogEntries.ts`（`ACTIVE_TUNER_SPECS` / `CALIBRATED_PRESET_SPECS`）、各处调用的 `<FloatingWindowCard icon={...} iconColor={...} />`，以及 `FloatingWindowIcon.tsx` 中基于字符串 `includes` 的旧规则。例如 `ball-trajectory-tuner`、`supersonic-trail-tuner`、`demolish-shockwave-tuner` 缺失规则回退成了蓝色 `Activity`；`engine-audio-tuner`、`vehicle-global-fx-tuner`、`network-stress-injection` 颜色与列表不一致。
  2. `FloatingWindowIcon.tsx` 依赖 Tailwind `dark:` 类名而非组件的 `isLight` prop，导致在 UI Tuner 局部切换 Light/Dark 预览、或展开（`FloatingWindowInstance`）与收纳（`FloatingStackIcon` / `FlyingGhostElement`）之间可能出现主题色不同步。
- **修复**：
  1. 新建单一数据源 `src/ui-core/navigation/floatingWindowIconRegistry.ts`（`FLOATING_WINDOW_ICON_REGISTRY` + `resolveFloatingWindowIconSpec` + `resolveFloatingWindowIconColorClass`），覆盖全部 37 个 User/Dev Floating Window Preset ID。
  2. `FloatingWindowIcon.tsx` 改由 `floatingWindowIconRegistry.ts` 驱动，并透传 `isLight` 参数。
  3. `FloatingWindowInstance.tsx`、`FloatingStackIcon.tsx`、`FlyingGhostElement.tsx`、`FloatingWindowsFlatMatrixWall.tsx` 统一透传 `isLight` 给 `getFloatingWindowIcon`。
  4. `FloatingWindowCard.tsx` 在传入 `preset` 时自动从 `resolveFloatingWindowIconSpec` 解析规范化 `icon` 与 `iconColor`，保证 **Active Tuners / Calibrated Presets / Settings 列表图标、飞行动画幽灵图标、展开后 Floating Window 标题图标、收纳后 Stack Tray 图标** 100% 一致。
  5. 在 `scripts/radar/radarUiAndRepoGuards.ts` 新增 `[FLOATING ICON REGISTRY VIOLATION]` 守卫，强制所有 `floatingPresets.ts` / `devFloatingPresets.ts` 中的 Preset ID 必须在 `FLOATING_WINDOW_ICON_REGISTRY` 中显式注册。
