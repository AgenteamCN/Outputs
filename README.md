# RocketCompanion PR #21 合并结果 & UI Variation 全量扫描报告与重构计划

## 1. PR 提交、合并与工具链验证结果

- **PR 创建与合并**：已将 `z/agy/ui/tier-variation-rework` (`6b6ace3064738b5eeba484253cbd2b517f061a8d`) 提交为 [PR #21](https://github.com/Antigravi-ty/RocketCompanion/pull/21) 并无冲突合并至 `alpha` (`8b6b0a2b91036d03c05886c0ddbe0e5f1b403f87`)。
- **工具链安装与基线校验**：
  - 环境缺少 `pnpm`，已安装 `pnpm@12.21.0` 并执行 `pnpm install --frozen-lockfile`。
  - 在合并后的 `alpha` (`8b6b0a2`) 上执行 `pnpm run check` 与 `pnpm run radar:audit` 全部通过（`STATUS: GREEN` — `1389/1389` 文件 100% TS，`tsc --noEmit` 零错误，边界与体积守卫全绿）。

---

## 2. 对「先 Merge 再分叉再改」工作流的评估

**完全同意先合并 `6b6ace3` 到 `alpha` 再开新分支重构**，原因有三点：
1. **PR #20 是跨 57 个文件的横向破坏性重构**（统一 `FloatingWindowIcon`、`SURFACE_META`、`iconLookup`、`radarUiAndRepoGuards.ts`、`devFloatingPresets`、`AdvancedTab`、`DeveloperTab`）。如果不在 `alpha` 基线上汇合，后续任何 variation 扩展都会与 `radarUiAndRepoGuards.ts` 和 `surfaceMetaTypes.ts` 产生二次冲突。
2. **当前最大问题不是「代码跑不起来」，而是「声明层（`SURFACE_META.variants`）、遍历层（`menuLayerLinearPlaylists.ts`）、路由层（`MenuLayerStageRecipeRouter.tsx`）与审批账本（`uiApprovalLedger.json`）四者脱节」**。在合并后的 `alpha` 上统一抽象，可以一次性用破坏性重构对齐这四层契约。
3. **每个 commit 靠 `pnpm run check`（Radar + `tsc --noEmit`）兜底**，适合以「契约先行 + 自动生成遍历队列」的方式推进。

---

## 3. 针对已知疑点的精确核查结论

### 3.1 `Layer2SettingsRecipe [user-l2-settings-advanced]` 是否区分「已导入 / 未导入」？

**结论：在 Unified Approval Playlist（审批遍历队列）中完全没有区分，存在明确漏洞。**

1. **导入了却未使用**：`src/ui-tuner/pages/menu-layer/menuLayerLinearPlaylists.ts` 第 7 行导入了 `SETTINGS_TAB_VARIANTS`，但全文未使用：
   - `USER_LINEAR_PLAYLIST`（第 118-121 行）仅展开了 `USER_SETTINGS_QUICK_TABS`，只生成 4 个扁平步骤（`user-l2-settings-gameplay` / `audio` / `display` / `advanced`），**没有任何 `storeOverrides` 与 `profileTier`**。
   - 结果：`AdvancedTab.tsx` 中定义的 `settings:advanced-anonymous`、`settings:advanced-pack-imported`、`settings:advanced-developer` 以及 `DisplayTab.tsx` 中的 `settings:display-tauri-fullscreen` **全部没有进入审批队列**。在审批流里点击 `user-l2-settings-advanced` 时，它只会读取当前全局残留的 store 状态。
   - 反观 `src/ui-tuner/pages/recipes/recipeStateMatrixSpecs.ts`（第 109-130 行）却正确映射了全部 8 个 `SETTINGS_TAB_VARIANTS`，导致「Matrix 视图」和「Unified Approval Playlist」严重不一致。
2. **Router 层的 `StoreSnapshotScope` 覆盖 Bug**：
   - `src/ui-tuner/pages/menu-layer/MenuLayerStageRecipeRouter.tsx` 第 181-201 行在 `displayedRoute === 'settings'` 内部又包了一层 `<StoreSnapshotScope overrides={{ appStore: { settingsTargetTab: settingsTab, developerMode: visibilityTier === 'developer', ... } }}>`。
   - 这会**强制覆写**外层 `activePlaylistStep.storeOverrides.appStore.developerMode`。即使未来在 User 队列里加入 `settings:advanced-developer` 或在 Dev 队列测试不同 tier，内层 Scope 也会把 `developerMode` 锁死为 `visibilityTier === 'developer'`。
3. **`AdvancedTab.tsx` 本身的状态盲区**：
   - `SignalingProfileSection` 有 4 种 `source`（`'public' | 'custom' | 'imported-bundle' | 'dev-private'`），目前缺少 `'custom'`（自建中继）变体。
   - `SignalingProfileSection`、`AudioDebugSection`、`RealisticBgmSection` 均使用组件局部 `useState(false)` 控制 `isExpanded`，在静态快照预览中默认全部折叠，导致 `.rctbnspib` 导入/PIN 解锁/中继探测面板在审批中不可见。

### 3.2 Garage Sync 三模式与 `pack imported`（未登录 vs 已登录）

**结论：分类枚举模型本身只有 3 态，无法表达 4 态正交现实。**

- `src/ui-core/manifest/surfaceMetaTypes.ts` 第 43 行定义：
  `export type ProfileTierVariantKind = 'anonymous' | 'signed-in' | 'developer';`
- 实际业务存在 **4 个独立状态**（当引入 `.rctbnspib` pack 导入后）：
  1. **Anonymous（未导入 Pack）**：`signalingProfileStore.source = 'public'`，`cloudProfileSyncStore.available = false`，`loggedInUser = null`
  2. **Pack Imported · Signed Out（已导入 Pack，未登录）**：`source = 'imported-bundle'`，`available = true`，`loggedInUser = null`
  3. **Pack Imported · Signed In（已导入 Pack，已登录）**：`available = true`，`loggedInUser != null`，`developerMode = false`
  4. **Developer（开发者模式）**：`developerMode = true`，`canManageAccounts = true`，`source = 'dev-private'`
- 因为 `ProfileTierVariantKind` 缺少 `'pack-imported'`，`garageSyncVariants.ts` 的 `sync-sign-in` 和 `AdvancedTab.tsx` 的 `settings:advanced-pack-imported` 被迫全部标成 `profileTier: 'anonymous'`。
- 此外，`GarageCloudAccountSyncSection` / `GarageCloudProfileSlotCard` 还缺少 `conflicts.length > 0`（`CloudProfileConflictDialogue` 3-way merge 冲突弹窗）、`syncStatus === 'syncing'`、`errorMessage != null` 三个关键 UI 状态变体。

---

## 4. 全仓 UI Variation 覆盖面扫描报告

### 4.1 整体资产盘点（`alpha` @ `8b6b0a2`）
- **Menu Recipes 顶层文件**：33 个（`src/ui-core/recipes/*.tsx`），含子组件共 107 个 `.tsx`。
- **Floating Windows**：43 个（8 User + 35 Dev）。
- **审批账本 (`uiApprovalLedger.json`)**：仅记录 67 个 surface / 57 个 variant（且存在 key 命名分裂，如 `recipe:Layer2GarageRecipe` 同时存在 `'car'` 与 `'user-l2-garage-car'`）。

### 4.2 声明了 `SURFACE_META.variants` 却未在 `menuLayerLinearPlaylists.ts` 展开的 Recipe（共 9 个 + Settings）

对比 `floatingWindowsLinearPlaylists.ts`（第 126-134 行已实现自动读取 `SURFACE_META.variants`），Menu Recipe 的 `menuLayerLinearPlaylists.ts` 仍是手工拼接，导致以下 **9 个已声明多变体的 Recipe 在审批队列中被压缩成单一步骤**：

| Recipe | 声明的 `variants` 数量 | `menuLayerLinearPlaylists.ts` 现状 |
| --- | --- | --- |
| `Layer2SettingsRecipe` | 8 (`SETTINGS_TAB_VARIANTS`) | 仅 4 user + 1 dev（无 `storeOverrides`） |
| `Layer3AcousticsEqualizerRecipe` | 2 (`dynamic-default`, `night-compressed`) | 仅 1 步 (`dev-l3-audio-eq`) |
| `Layer3BallTrajectoryRecipe` | 2 (`user-tier`, `developer-tier`) | 仅 1 步 (`user-l3-trajectory`) |
| `Layer3CameraRecipe` | 2 (`default`, `developer`) | 仅 1 步 (`user-l3-camera`) |
| `Layer3DevSelfTestRecipe` | 2 (`dev-private`, `public-default`) | 仅 1 步 (`dev-l3-self-test`) |
| `Layer3ExhibitionRecipe` | 2 (`standard-necto`, `duel-nexto`) | 仅 1 步 (`user-l3-exhibition`) |
| `Layer3KeybindingsRecipe` | 3 (`all`, `Driving`, `Aerial`) | 仅 1 步 (`user-l3-keybindings`) |
| `Layer3KickoffTrainingRecipe` | 3 (`instant`, `easy-learning`, `realistic`) | 仅 1 步 (`user-l3-kickoff`) |
| `Layer3TrainingConfigRecipe` | 2 (`default`, `standard-no-reset`) | 仅 1 步 (`user-l3-training`) |
| `Layer4StaticTrainingPackBrowseRecipe` | 2 (`user`, `dev`) | 仅 1 步 (`user-l4-rcspack`) |

### 4.3 跨子组件状态依赖与未覆盖分支（按轴扫描）

通过全量扫描 `src/ui-core/recipes/**/*.tsx` 的 Store 依赖与条件渲染分支，发现以下高风险遗漏：

1. **完全缺席审批队列的 Recipe**：
   - `Layer1TauriConfiguratorRecipe.tsx`：未出现在 `USER_LINEAR_PLAYLIST` 或 `DEV_LINEAR_PLAYLIST` 中。
2. **多 Tab 但只遍历 1 步的 Recipe**：
   - `Layer3DevSandboxesRecipe.tsx`（含 6 个 Tab，队列仅 `dev-l3-isolation` 1 步）。
3. **子组件依赖 `signalingProfileStore` / `cloudProfileSyncStore` / `isTauri`，但顶层声明为 `tier-invariant` 或未建模变体**：
   - `Layer2PlayRecipe.tsx`：声明 `tier-invariant` / 无 variants，但实际读取 `signalingProfile`（影响联机入口提示/不可用状态）。
   - `Layer3HostMatchRecipe.tsx` & `Layer3HostTrainingRecipe.tsx`：子组件 `OnlineRoomIdentityFields.tsx` 依赖 `signalingProfile`（`imported-bundle` vs `public` vs `dev-private`）与错误状态，但当前变体只遍历了创建进度（`idle` / `running-step-*` / `completed` / `error-step-*`），未覆盖 signaling source 差异。
   - `Layer3DevQuickActionsRecipe.tsx`、`Layer4AdvancedAudioRecipe.tsx`、`Layer4ControllerSettingsRecipe.tsx`：均声明了 `profileTierPolicy: 'user-dev-split'` 且读取 `developerMode`，但全部没有 `variants` 定义。
   - `DisplayTab.tsx`：存在 8 处 `isTauri` 分支（桌面端全屏/窗口模式切换），依赖 `SETTINGS_TAB_VARIANTS` 展开后才能被审到。
4. **Radar 守卫 (`scripts/radar/radarUiAndRepoGuards.ts`) 的盲区**：
   - 第 190-204 行仅正则检查顶层 `Layer*Recipe.tsx` 单文件是否出现 `developerMode` 或 `useCloudProfileSyncStore`，**不会递归检查子组件**（如 `GarageSyncTab`、`AdvancedTab`、`SignalingProfileSection`、`OnlineRoomIdentityFields`），导致顶层与子组件契约不一致时 Radar 仍然报 GREEN。
   - Radar 目前没有检查「`SURFACE_META.variants` 是否全部被 `menuLayerLinearPlaylists.ts` 消费」，导致出现类似 `SETTINGS_TAB_VARIANTS` 导入却未使用的静默漏检。

---

## 5. 重构与全遍历计划（建议按 Phase 推进）

### Phase 1：统一 Tier 与正交轴模型（解决 3 态 vs 4 态错配）
1. 扩展 `src/ui-core/manifest/surfaceMetaTypes.ts`：
   - 将 `ProfileTierVariantKind` 扩展为 `'anonymous' | 'pack-imported' | 'signed-in' | 'developer'`（或拆分 `authTier` + `bundleSource` + `runtimePlatform` 三个正交元数据字段）。
   - 更新 `GARAGE_SYNC_VARIANT_METAS` 中的 `sync-sign-in` 为 `profileTier: 'pack-imported'`，并在 `GarageSyncTab` 补充 `sync-conflict` / `sync-error` 变体。
   - 更新 `AdvancedTab.tsx` 变体矩阵：区分 `advanced-anonymous`（Public Broker）、`advanced-pack-imported`（已导入未登录）、`advanced-signed-in`（已导入已登录 + 远程部署开关）、`advanced-custom-broker`（自建中继）、`advanced-developer`，并支持通过 store/prop 控制 `SignalingProfileSection` 默认展开快照。

### Phase 2：消除手工拼接，Playlist 自动从 `SURFACE_META.variants` 生成
1. 像 `floatingWindowsLinearPlaylists.ts` 一样，重构 `menuLayerLinearPlaylists.ts`：
   - 对所有声明了 `SURFACE_META.variants` 的 Recipe（含 `Layer2SettingsRecipe` 及上述 9 个被折叠的 L3/L4 Recipe）**自动展开所有 `variants`**，自动注入 `storeOverrides`、`profileTier`、`subtab` 与标准化的 `variantId`。
   - 补全 `Layer1TauriConfiguratorRecipe` 与 `Layer3DevSandboxesRecipe`（6 Tabs）的遍历条目。
2. 修复 `MenuLayerStageRecipeRouter.tsx` 第 182-201 行的 `StoreSnapshotScope` 覆写问题：
   - 将 `activeStepStoreOverrides` 置于最内层或与路由默认 overrides 做深度合并，确保 `activePlaylistStep.storeOverrides.appStore.developerMode` 不会被 `visibilityTier === 'developer'` 强行覆盖。

### Phase 3：对齐 Ledger Key 与 Radar 自动化守卫
1. **统一 `unitKey`**：在 `reviewLedgerAdapter.ts` 中严格使用 `${surfaceId}::${variant.variantId}`，清理 `uiApprovalLedger.json` 中类似 `car` vs `user-l2-garage-car` 的重复历史键。
2. **增强 `scripts/radar/radarUiAndRepoGuards.ts`**：
   - 增加「Playlist 覆盖率守卫」：校验每个 `Layer*Recipe.tsx` 的 `SURFACE_META.variants`（以及 `Layer1`、`Garage`、`Settings`）必须 100% 出现在 `ALL_UNIFIED_APPROVAL_PLAYLIST` 中，否则 `pnpm run radar` 直接报错阻断。
   - 增加「跨子组件 Store 依赖审计」：提示顶层 `profileTierPolicy: 'tier-invariant'` 但子组件读取了 `cloudProfileSyncStore` / `signalingProfileStore` / `developerMode` 的表面。
