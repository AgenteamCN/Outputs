# RocketCompanion · Auth/Tier & Tauri Variation 修复报告

## 问题根因

1. **Garage Sync 多状态被统一覆盖为 Anonymous（带小锤子占位符）**
   - 在 `src/ui-tuner/pages/menu-layer/useMenuLayerNavigationHandlers.ts` 中，`activePlaylistStep` 无论当前是否处于 Free Roam，都会无条件调用 `findPlaylistStepIndex(...)` 并用 `activePlaylist[matchedStepIdx]` 覆盖 `anchorPlaylistStep`。
   - 由于 `findPlaylistStepIndex` 仅按 `(route, subtab)` 匹配，而 `sync-anonymous`、`sync-sign-in`、`sync-signed-in`、`sync-developer` 共用 `route: 'garage'` 与 `subtab: 'sync'`，因此始终命中列表第一个条目 `user-l2-garage-sync-anonymous`（`cloudProfileSyncStore.available = false`，渲染 `UnderConstructionPlaceholder` 小锤子）。
   - 该问题同时影响了其他同路由多变体页面（如 `Layer3DevRemoteProfileRecipe` 的 3 个变体及新增的 Settings 变体）。

2. **Settings 页面与 Online 组件未接入 `StoreSnapshotScope` / 缺少多状态与 Tauri 变体**
   - `SignalingProfileSection`、`SignalingRelayNodeListSection`、`DeveloperProfileExportSection`、`OnlineRoomIdentityFields`、`Layer3HostMatchRecipe`、`Layer3HostTrainingRecipe` 直接调用 `useSignalingProfileStore(...)`，未通过 `useScopedStore('signalingProfileStore', ...)` 读取快照覆盖。
   - `DisplayTab` 与 `DisplayRefreshRateCard` 直接调用 `isTauriRuntime()`，在浏览器预览环境下无法展示 Tauri 桌面端 `Window Mode (Fullscreen / Borderless / Windowed)`。
   - `Layer2SettingsRecipe`、`menuLayerLinearPlaylists.ts`、`recipeStateMatrixSpecs.ts` 缺少 Settings 的 `Anonymous` / `Pack Imported` / `Developer` 与 `Tauri Fullscreen` 变体，且 `layer2RecipeRegistrySpecs.tsx` 中 Garage 与 Settings 矩阵卡片未透传 `st.storeOverrides`。

---

## 修改内容

### 1. 修复多变体播放列表匹配覆盖
- `src/ui-tuner/pages/menu-layer/useMenuLayerNavigationHandlers.ts`：当 `!isFreeRoaming`（即 `anchorPlaylistStep` 已匹配当前视图）时直接保留 `anchorPlaylistStep`，仅在 `isFreeRoaming` 时回退到 `findPlaylistStepIndex` 匹配结果。

### 2. Signaling Store 全面接入 `useScopedStore`
- `src/ui-core/recipes/settings/components/SignalingProfileSection.tsx`
- `src/ui-core/recipes/settings/components/SignalingRelayNodeListSection.tsx`
- `src/ui-core/recipes/settings/components/DeveloperProfileExportSection.tsx`
- `src/ui-core/recipes/online/OnlineRoomIdentityFields.tsx`
- `src/ui-core/recipes/Layer3HostMatchRecipe.tsx`
- `src/ui-core/recipes/Layer3HostTrainingRecipe.tsx`

### 3. Tauri Display Fullscreen 变体与 `tauriRuntime` 覆盖支持
- `src/ui-core/recipes/settings/tabs/DisplayTab.tsx`：新增 `useStoreSnapshotOverride('tauriRuntime')` 支持，并导出 `DISPLAY_TAURI_FULLSCREEN_VARIANT_META`（`settings:display-tauri-fullscreen`，模拟 Windows Tauri 运行时并默认选中 `Fullscreen`）。
- `src/ui-core/compounds/DisplayRefreshRateCard.tsx`：接入 `tauriRuntime` 快照覆盖。

### 4. Settings `Anonymous` / `Pack Imported` / `Developer` 变体与矩阵同步
- `src/ui-core/recipes/settings/tabs/AdvancedTab.tsx`：拆分导出 `SURFACE_VARIANT_META`（`settings:advanced-anonymous`）、`ADVANCED_PACK_IMPORTED_VARIANT_META`（`settings:advanced-pack-imported`）、`ADVANCED_DEVELOPER_VARIANT_META`（`settings:advanced-developer`），分别配置 `appStore.developerMode`、`signalingProfileStore`（`public` / `imported-bundle` / `dev-private`）与 `cloudProfileSyncStore`。
- `src/ui-core/recipes/settings/tabs/DeveloperTab.tsx`：为 `SURFACE_VARIANT_META` 补充 `dev-private` 的 `signalingProfileStore` 覆盖，确保 `DeveloperProfileExportSection` 在 tuner 中正常渲染。
- `src/ui-core/recipes/Layer2SettingsRecipe.tsx`：将上述新变体注册到 `SETTINGS_TAB_VARIANTS`。
- `src/ui-tuner/pages/menu-layer/menuLayerLinearPlaylists.ts`：基于 `SETTINGS_TAB_VARIANTS` 生成 `SETTINGS_USER_PLAYLIST_STEPS` 与 `SETTINGS_DEV_PLAYLIST_STEPS`。
- `src/ui-tuner/pages/recipes/recipeStateMatrixSpecs.ts` & `src/ui-tuner/pages/recipes/layer2RecipeRegistrySpecs.tsx`：为 `RecipeStateSnapshotSpec` 增加 `storeOverrides`，扩展 `LAYER2_GARAGE_STATES` 与 `LAYER2_SETTINGS_STATES` 并在 `renderStateCard` 中通过 `<StoreSnapshotScope>` 注入。

---

## 验证结果

- 工具链：安装 `pnpm` 并执行 `pnpm install --frozen-lockfile`。
- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS**（1389/1389 TS 文件，0 边界违规，0 类型错误）。
- `pnpm run build`（主应用 Vite 生产构建）：**PASS**。
- `NODE_OPTIONS=--max-old-space-size=4096 RC_VITE_ROLE=tuner npx vite build --outDir /tmp/tuner-dist`（Tuner Vite 生产构建）：**PASS**。

## 提交信息

- 分支：`z/agy/ui/tier-variation-rework`（基于 `1e76c182614a0e3530819bb68d33776956fad541`）
- Commit：`6b6ace3064738b5eeba484253cbd2b517f061a8d`（GitHub GraphQL `createCommitOnBranch`，`signature.isValid: true`, `state: VALID`）
