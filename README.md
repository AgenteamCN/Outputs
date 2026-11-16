# RocketCompanion 任务交付报告

## 1. Ultra Low Power (`?fastest=1`) PR 与合并

- **源分支**: `z/agy/rc/fastest-ultra-low-power` (HEAD: `cd5292d1bca4ee9b862bd39a1ed80e2115fa087b`)
- **目标分支**: `alpha`
- **Pull Request**: [PR #24](https://github.com/Antigravi-ty/RocketCompanion/pull/24)
- **状态**: 已创建并合并 (`merge` 方法)，合并提交 SHA: `42c32b809eae4a61aeeca0dfbcb9c7f6bd7a3d73`

---

## 2. `z/agy/ui/phase1-tier-orthogonal-variants` Settings 切换失效复现与修复

### 问题复现与根因
在 Unified Approval Stage (`?tab=layer-preview`) 中，路径：
`src/ui-core/recipes/Layer2SettingsRecipe.tsx :: Layer2SettingsRecipe [user-l2-settings-settings:audio]`

1. **复现结果**：使用无头 Chrome (Playwright) 访问 `http://localhost:3002/?tab=layer-preview&route=settings&subtab=settings:audio` 或通过 `Next Page` 从 `settings:gameplay` 切换到 `settings:audio` 时，页面始终停留在 **Gameplay** Tab（`aria-selected="true"` 在 `Gameplay` 上，未渲染 Audio 音量滑块）。
2. **根因定位**：
   - `src/ui-core/recipes/settings/tabs/*.tsx` 中所有 Settings 变体的 `variantId` 均带有 `settings:` 前缀（如 `settings:gameplay`、`settings:audio`、`settings:display`、`settings:display-tauri-fullscreen`、`settings:advanced-*`、`settings:developer`），并在 `storeOverrides.appStore.settingsTargetTab` 中声明了真实目标 Tab。
   - `src/ui-tuner/pages/menu-layer/menuLayerVariantPlaylistBuilder.ts` 中的 `resolveSettingsSubtab(variantId)` 仅判断了 `variantId.startsWith('display')` 和 `variantId.startsWith('advanced')`，未剥离 `settings:` 前缀，也未读取 `storeOverrides.appStore.settingsTargetTab`，导致生成的 `step.subtab` 为 `'settings:audio'` 等非法 Tab ID。
   - 当 `settingsTab` 传入 `Layer2SettingsRecipe` 的 `normalizeTab()` 时，由于 `'settings:audio'` 不在 `'gameplay' | 'audio' | 'display' | 'advanced' | 'developer'` 中，全部静默回退到 `'gameplay'`。

### 修复内容
1. **`src/ui-tuner/pages/menu-layer/menuLayerVariantPlaylistBuilder.ts`**：
   - 将 `resolveSettingsSubtab` 改为优先读取 `variant.storeOverrides?.appStore?.settingsTargetTab`，并兜底剥离 `^settings:` 前缀后再映射 `display*` / `advanced*`。
2. **`scripts/radar/radarVariantSyncGuards.ts` & `scripts/radar.ts`**：
   - 新增静态架构守卫 `checkPlaylistVariantSyncAlignment`，在 `pnpm run radar` / `pnpm run check` 中强制校验：
     - 所有 `src/ui-core/recipes/settings/tabs/*.tsx` 的 `*_VARIANT_META` 必须定义合法的 `storeOverrides.appStore.settingsTargetTab`；
     - `menuLayerVariantPlaylistBuilder.ts` 必须通过 `settingsTargetTab` 解析 Settings 子 Tab。
3. **Verified Bot Commit**：
   - 分支: `z/agy/ui/phase1-tier-orthogonal-variants`
   - Commit: [`081c5b802b49417be33961cc281e5c9501b3d291`](https://github.com/Antigravi-ty/RocketCompanion/commit/081c5b802b49417be33961cc281e5c9501b3d291) (`signature.isValid: true, state: "VALID"`)

---

## 3. 全量扫描结果与“无需 LLM 的可靠测试方法”

### 全量 Playlist 扫描结果
修复后通过 Playwright 对 `tier=user`（67 步）与 `tier=developer`（67 步）进行了全量 `Next Page` 步进扫描：
- **Settings 全部 10 个变体**（`gameplay`、`audio`、`display`、`display-tauri-fullscreen`、`advanced-anonymous`、`advanced-pack-imported`、`advanced-signed-in`、`advanced-custom-broker`、`advanced-developer`、`developer`）均已正确切换到对应 Tab 并渲染各自独立状态，**0 个碰撞**。
- **其他同路由多变体页面**：
  - `garage`（5 个主 Tab + 6 个 Sync 状态）、`manage-room`（6 个会话上下文 × 成员状态）、`keybindings`（3 个分类）、`dev-remote-profile`（3 个状态）、`Floating Windows`（带子 Tab 的浮窗）均能正确切换。
  - 注：`play`（`public-fallback` vs `pack-imported`）和 `dev-quick-actions`（`user-locked` vs `developer-active`）属于同一路由下仅改变隐藏/悬停元数据或权限标记的变体，主区域可见文本一致属于预期。

### 如何不依赖 LLM 可靠地测试这类问题？
不需要内置 LLM 去“理解”页面是不是 Audio，可以通过以下 **两层完全确定性的工程手段** 覆盖：
1. **静态契约守卫（已集成到 `pnpm run radar`）**：
   - 检查所有变体定义（`SURFACE_VARIANT_META.storeOverrides`）与 Playlist 生成器（`step.subtab`）之间的一致性，确保 `subtab` 永远属于目标 Recipe `normalizeTab()` 的合法枚举域，从编译/扫描阶段阻断前缀不匹配或静默回退。
2. **运行时 DOM 结构不变量测试（Headless Chrome / E2E 回归）**：
   - **激活态对齐断言**：遍历 Playlist 每一步时，读取 DOM 中激活 Tab 的语义标记（`[aria-selected="true"]`、`[data-tab-active="true"]`、`[data-state="active"]`），断言其与当前 `step.subtab` 一致（例如切换到 `settings:audio` 后，激活 Tab 必须是 `Audio`，而不是 `Gameplay`）。
   - **同路由变体去重（Non-Collapsing DOM）断言**：在同一 `route` 下连续切换不同 `variantId` 时，抓取可见 DOM 签名（可见文本 + `aria-selected`/`aria-pressed`/`aria-checked`/高亮样式类）。如果两个不同的 Tab 变体渲染出了完全相同的 DOM 签名，即可 100% 确定发生了“切换未生效/静默回退”。

---

## 4. 验证结果
- `pnpm run check` (`pnpm run radar && pnpm run typecheck`): **PASS**（1390/1390 TS 文件，`tsc --noEmit` 0 错误，Radar 全绿）。
- Headless Chrome (`google-chrome` + `playwright-core`) 全量步进验证：Settings 10 个变体全部正常切换且无页面报错。
