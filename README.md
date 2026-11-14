# UI Triage Rework, RCUITuner Sidebar Upgrade & Cross-Team Workflow Report

- **Branch**: `z/agy/ui/triage-rework-and-tuner` (based on `alpha` @ `8d19408`)
- **Verified Bot Commit**: `1e44c85907ae7336ca4ee39626b71a8fc1b2a8df`
- **Verification**:
  - `pnpm check` (`radar` + `tsc --noEmit`): **PASS** (1388/1388 TS files, 0 boundary/volume errors)
  - `pnpm build` (`vite build`): **PASS**

---

## 1. Completed Fixes & Triage Verification

### A. `#20` L2 Garage · Sync Tab (`recipe:Layer2GarageRecipe::user-l2-garage-sync`)
- **Root Cause**: `cloudProfileSyncStore.available` defaults to `false` when no `.rctbnspib` bundle is loaded, so `GarageCloudAccountSyncSection.tsx` only rendered `<UnderConstructionPlaceholder badge="BUNDLE REQUIRED" />` in RCUITuner — hiding the entire Cloud Profile Sync UI.
- **Fix**:
  - Exported `GARAGE_SYNC_PREVIEW_PRESETS` in `src/ui-core/recipes/garage/GarageSyncTab.tsx` with 3 explicit variations:
    1. `anonymous` — Anonymous / No Bundle (`available: false`, `BUNDLE REQUIRED` placeholder)
    2. `pack-imported` — Pack Imported / Signed Out (`available: true`, `loggedInUser: null`, sign-in form)
    3. `dev-signed-in` — Signed In / Dev (`available: true`, `canManageAccounts: true`, active `loggedInUser` + `userRevHash` / `devRevHash` + `Pull & Merge` / `Push Now`)
  - Wired a 3-way `Sync Variation` pill switcher + `<StoreSnapshotScope>` in `src/ui-tuner/pages/menu-layer/MenuLayerStageRecipeRouter.tsx` when `garageTab === 'sync'` (defaulting to `dev-signed-in` so Cloud Profile Sync is immediately visible).
  - Updated `src/ui-core/recipes/garage/GarageCloudAccountSyncSection.tsx` to clear `usernameInput` and `passwordInput` when `loggedInUser` transitions to `null`.

### B. RCUITuner Left Sidebar 4-State Triage + Multiline Note + Spacebar Fix
- **New Component**: `src/ui-tuner/components/review/SidebarReviewTriagePanel.tsx` (mounted in `src/ui-tuner/Sidebar.tsx` on `layer-preview`, `floating-windows`, and `recipes` tabs):
  - **4-Choice Active Selector (4选一激活)**: `Approve [A]`, `Reject [R]`, `Fixed [F]`, `Hold [D]` in a 2×2 grid with clear active fill + ring styling.
  - **Previous Ledger Note Box**: Reads the baseline note & status from `uiApprovalLedger.json` and displays it separately in a read-only `Previous Ledger Note` card so old baseline notes are never confused with new review edits.
  - **Multi-line Comment Textarea (`#rc-sidebar-triage-note`)**:
    - Supports natural `Enter` newlines (no `Enter` hijack) and auto-saves on `onChange`.
    - Calls `e.stopPropagation()` on `onKeyDown` (blurring only on `Escape`) so Spacebar, Enter, Arrow keys, and single-letter shortcuts never get intercepted by global window listeners.
- **Updated `src/ui-tuner/components/review/RapidReviewTriageBar.tsx`**:
  - Added `Fixed [F]` (`pending`) button and `F` keyboard shortcut.
  - Updated `[Space]` shortcut (when not typing in an input/textarea) to focus `#rc-sidebar-triage-note` in the left sidebar.
  - Replaced the cramped bottom `<textarea>` with a compact `"Edit Note in Sidebar [Space]"` button.

### C. Verification of `#3`, `#13`, `#14`, `#17`, `#18` & Store Diffs
- **Confirmed Already Fixed in `72bd281`**:
  - `#3` & `#14`: `MainMenuRow.tsx` already omits `focus-visible:ring-2 focus-visible:ring-amber-500/60` when `isSolidPrimary` is true.
  - `#13`: `EdgeInspectorDrawer.tsx` already auto-collapses and shows blocked tooltip/shake when `isMenuOpen` is true.
  - `#17`: `GaragePaintTab.tsx` already has `FlippedSprayCanIcon` (`scaleX(-1)`), `Team Priority`, `Blue / Orange` options, and `UserStar` icon.
  - `#18`: `GarageLoadoutTab.tsx` already removed Ball Style and Demolition Effect rows.
  - These appeared in Section 2 of the triage export because their old notes were still stored in `src/ui-core/manifest/uiApprovalLedger.json`.
- **Calibrated Store Diff Applied**:
  - `src/ui-core/config/nameTagConfig.ts`: Changed `offscreenIndicatorEnabled` default from `true` to `false`.

---

## 2. Cross-Team Audit: Dynamic Training Pack, Tauri Configurator & Collaboration Workflow

### A. Dynamic Training Pack (`DynamicTrainingPackFloatingWindow.tsx`)
- **Current State**: Implemented as a `floating-dev` window (`src/ui-core/panels/dev/DynamicTrainingPackFloatingWindow.tsx`) with subcomponents in `src/ui-core/panels/dev/dynamic-training/`. It already uses `DropdownMenu`, `SegmentedSwitch`, and `Checkbox` from `ui-core`, though it still has a few raw `<button>` and `<input>` elements.
- **Can UI restyle / migrate it to `EdgeInspectorDrawer` without breaking RC?**
  - **Yes, 100% safe.** All simulation, replay capture, and takeover logic is cleanly isolated behind `useDynamicTrainingPackStore` (`runtimeHandlers`) and `src/ui-system/core/runtimeDynamicTrainingBridge.ts`.
  - UI can restyle the controls with `Button` / `Panel` / `EdgeInspectorDrawer` or move it into a Play/Training entry without touching `DynamicTrainingMode.ts` or replay/physics code.

### B. Tauri Pre-Launch Configurator (`Layer1TauriConfiguratorRecipe.tsx`)
- **Current State**: Surprisingly clean and well-aligned with `ui-core` standards! It already uses `PanelFooter`, `SettingSegmentedRow`, `SettingToggleRow`, `ConfiguratorActionButtons`, `ConfiguratorAdvancedView`, `Checkbox`, and `Button`, and is decoupled via `src/ui-core/config/tauriLaunchConfigBridge.ts`.
- **Gap**: Its `SURFACE_META.id` is `'layer1-tauri-configurator'` (missing `'recipe:'` prefix) and it is not registered in RCUITuner's linear playlist (`menuLayerLinearPlaylists.ts`), so it hasn't been part of the UI approval loop. No urgent rework needed — just optional playlist registration if UI wants visual sign-off coverage.

### C. Recommended Multi-Team Workflow (UI / RC / Tauri)
1. **Clear Boundary Contract**:
   - **RC / Tauri** own `src/ui-core/stores/*` (actions/state) and `src/ui-core/config/*` (bridges/codecs).
   - **UI** owns `src/ui-core/primitives/*`, `src/ui-core/compounds/*`, `src/ui-core/layout/*` (including `EdgeInspectorDrawer` and `FloatingWindow` slots), and `src/ui-core/recipes/*`.
   - When RC prototypes a new feature (like Dynamic Training Pack), they should expose a store + `SURFACE_META` in `panels/dev/*` using `ui-core` primitives (no ad-hoc raw `<div>`/`<button>` styling), and hand off to UI to wrap in `EdgeInspectorDrawer` / `Layer2/3` recipes.
2. **Approval Ledger Hygiene**:
   - Keep **Previous Ledger Note** (read-only baseline) visually separate from the **Reviewer Note** (active session) as implemented in `SidebarReviewTriagePanel.tsx`.
   - When a unit moves from `Fixed` (`pending`) → `Approved`, clear or archive the old rework note in `uiApprovalLedger.json` so resolved notes don't resurface in future triage summaries.
