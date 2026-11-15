# UI 分支回退排查、Auth-Triad 自动遍历修复与 Floating Icon 拆分评估

## 1. 为什么 `4fbfebd` 看起来在“开倒车”？

经对 Git 提交图谱与差异（`git diff --stat`）核对：

- `origin/alpha` 位于 `8d19408`。
- `1e44c85`（`origin/z/agy/ui/triage-rework-and-tuner`）是从 `8d19408` 拉出的分支，完成了**评审面板左移（`SidebarReviewTriagePanel.tsx` / `Sidebar.tsx`）、新增 `fix` 状态标签（`RapidReviewTriageBar.tsx`）**，以及 Garage Sync 的 3 态手动切换原型。
- `4fbfebd`（`origin/z/agy/ui/tier-variation-and-floating-icon`）**错误地直接从 `origin/alpha`（`8d19408`）重新开分支**，没有基于 `1e44c85` 继续推进，导致 `1e44c85` 中已经改好的左侧 Triage 面板与 `fix` 标签全部丢失。

## 2. Floating Window 图标/颜色不一致修复：应该合在一起还是单独从 `alpha` 开分支？

**结论：两者是完全独立、零冲突的两类修复，应当拆分开，不要混在同一个分支里。**

- 对比 `8d19408..1e44c85`（7 个文件）与 `8d19408..4fbfebd` 中 Floating Window 图标注册表相关改动（`floatingWindowIconRegistry.ts`、`FloatingWindowIcon.tsx`、`FloatingStackIcon.tsx`、`FloatingWindowInstance.tsx`、`FlyingGhostElement.tsx`、`FloatingWindowCard.tsx`、`FloatingWindowsFlatMatrixWall.tsx`、`radarUiAndRepoGuards.ts` 中的 `[FLOATING ICON REGISTRY VIOLATION]`）：
  - **文件层面 0 重叠**；
  - **逻辑层面 0 耦合**（一个是 Menu Layer / Garage Sync 的 Auth-Triad 自动遍历与评审工具链，一个是 Floating Window 展开/收纳/幽灵动画/卡片图标的单一数据源统一）。
- **建议做法**：
  - 本分支 `z/agy/ui/tier-variation-rework`（基于 `1e44c85`，最新 Verified Commit：`1e76c18`）**只专注**保留左侧 Triage 重构 + 完成 Auth-Triad 自动遍历与 Garage Sync 权限状态修复；
  - Floating Window 图标/颜色统一修复可以单独从 `alpha`（或直接 cherry-pick `4fbfebd` 的图标部分）开一条独立 `z/agy/ui/*` 分支合并，互不干扰，真正做到“指哪儿打哪儿”。

## 3. 本次基于 `1e44c85` 的修复内容（分支 `z/agy/ui/tier-variation-rework`，Commit `1e76c18`）

1. **完整保留 `1e44c85` 的左侧 Triage 面板与 `fix` 标签**
   - 保留 `SidebarReviewTriagePanel.tsx`、`RapidReviewTriageBar.tsx`、`Sidebar.tsx` 的全部重构。

2. **修复 `Layer2GarageRecipe [user-l2-garage-sync-*]` 的状态定义与自动遍历**
   - 原 `4fbfebd` 将 `sync-anonymous` 错配为 `available: true`（导致 Anonymous 看到了登录表单），且缺少独立的 Sign-In（已导入 bundle 但未登录）页面。
   - 现已在 `src/ui-core/recipes/garage/garageSyncVariants.ts` 修正为 4 个清晰变体，并自动展开至 `USER_LINEAR_PLAYLIST` 与 `DEV_LINEAR_PLAYLIST`：
     - `user-l2-garage-sync-anonymous`（Anonymous · No Bundle）：`available: false, loggedInUser: null` -> 渲染 `UnderConstructionPlaceholder`（`BUNDLE REQUIRED`），**看不到登录表单**。
     - `user-l2-garage-sync-sign-in`（Pack Imported · Sign In）：`available: true, loggedInUser: null` -> **看得到登录表单**（Username / Passphrase / Sign In）。
     - `user-l2-garage-sync-signed-in`（Signed In）：`available: true, loggedInUser: {...}` -> 渲染已登录账号状态（`#ace-pilot`、`Pull & Merge`、`Push Local` 等）。
     - `dev-l2-garage-sync-developer`（Developer）：`tier: 'developer', canManageAccounts: true, loggedInUser: {...}, activeRemoteSession: {...}` -> 自动进入 `DEV_LINEAR_PLAYLIST` 队列。

3. **修复顶部与底部（`Panel` / `PanelFooter`）不跟随 `StoreSnapshotScope` 变化的问题**
   - `src/ui-core/layout/Panel.tsx` 与 `src/ui-core/layout/PanelFooter.tsx` 均已改为通过 `useScopedStore('cloudProfileSyncStore', useCloudProfileSyncStore, ...)` 读取 `activeRemoteSession` 与 `exitRemoteTuningSession`。
   - `src/ui-tuner/pages/menu-layer/MenuLayerStageRecipeRouter.tsx` 顶层统一使用 `<StoreSnapshotScope overrides={activeStepStoreOverrides ?? {}}>` 自动注入当前 Playlist 步骤的 `storeOverrides`（并移除此前临时加的顶部手动 `Sync Variation` 切换条）。
   - `src/ui-core/recipes/Layer3DevRemoteProfileRecipe.tsx` 同步接入 `profileTierPolicy: 'auth-triad'`、`useStoreSnapshotOverride`（支持 `activeTab` 覆盖）与 `useScopedStore('cloudProfileSyncStore', ...)`，并展开 3 个变体到 `DEV_LINEAR_PLAYLIST`。

4. **Schema 与 Radar 静态/运行时守卫**
   - 同步 `surfaceMetaTypes.ts`、`floatingWindowSchemaValidator.ts`、`floatingSurfaceRegistry.ts`、所有 `Layer*Recipe.tsx` / `StaticTrainingShotInspectorDrawer.tsx` 的 `profileTierPolicy` 声明，以及 `scripts/radar/radarUiAndRepoGuards.ts` 中的 `[SURFACE SCHEMA VIOLATION]` / `[SURFACE TIER CONTRACT VIOLATION]` 守卫（未混入 Floating Icon 部分，保持低耦合）。

## 4. 验证结果

- `pnpm check`（`pnpm run radar && pnpm run typecheck`）：通过（0 errors）。
- `pnpm build`（Vite 生产构建）：通过（0 errors）。
- GitHub Verified Bot Commit：`1e76c182614a0e3530819bb68d33776956fad541`（`signature: { isValid: true, state: 'VALID' }`），已同步至 `origin/z/agy/ui/tier-variation-rework`。
