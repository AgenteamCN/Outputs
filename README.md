# Floating Window Icon, Color & Title Unification Audit & Implementation Report

## 1. Execution Summary
- **Branch**: `z/agy/ui/unify-floating-window-icons` (branched from `8d19408a1a7a375e23a1e69cf5edd8e111e1bda0`)
- **Verified Bot Commit**: `2fb905260b492ba5717cd806069d7dd7e1410d32` (`signature.state: VALID`)
- **Toolchain & Verification**:
  - Installed `pnpm` and workspace dependencies (`pnpm install`).
  - `pnpm run check` (`radar` + `tsc --noEmit`): **PASS (GREEN)**
  - `pnpm run format:check`: **PASS**
  - `pnpm run build`: **PASS**

---

## 2. Root Cause Analysis
Previously, every floating window's visual identity was fragmented across **four disconnected locations**:
1. **Menu Launchers** (`devHubSurfaceCatalog.ts`, `devHubExperimentalCatalogEntries.ts`, `devSandboxBenchesCatalog.ts`, `AdvancedTab.tsx`, `DeveloperTab.tsx`, `Layer3*Recipe.tsx`, `TelestratorDeveloperSection.tsx`): Manually passed `icon` and `iconColor` (or omitted `iconColor`, falling back to neutral gray).
2. **Floating Window Header & Minimized Stack Tray** (`FloatingWindowIcon.tsx` -> `FloatingWindowInstance.tsx`, `FloatingStackIcon.tsx`, `FlyingGhostElement.tsx`): Used a 23-branch `if/else` string-matching heuristic (`normId.includes(...) || normCat.includes(...)`) that fell back to `Activity (sky)` for unmatched IDs.
3. **Co-located `SURFACE_META`** (`src/ui-core/panels/{user,dev}/*FloatingWindow.tsx`): Declared `title` and `icon` independently (without `iconColor`), and was not checked against `devFloatingPresets.ts` because `radarUiAndRepoGuards.ts` only matched `= { ... };` and missed `makeDevPreset({ ... });`.
4. **`PresetConfig`** (`floatingPresets.ts`, `devFloatingPresets.ts`): Declared `id`, `title`, `category`, `summary`, etc., but had no `iconName` or `iconColor`.

---

## 3. Full Scan of All 38 Floating Windows (Before Alignment)

### 3.1 Windows Falling Through to `Activity (sky)` Fallback in `FloatingWindowIcon.tsx` (6 Windows)
1. `ball-trajectory-tuner`: Menu had `Target` (`amber-400`) in Active Tuners and `Sliders` (`amber-400`) in `Layer3BallTrajectoryRecipe.tsx`; `SURFACE_META.icon` was `undefined`; window/stack showed `Activity` (`sky`).
2. `bots-loadout-tuner`: Menu & `SURFACE_META` had `Bot` (`amber-400`); window/stack showed `Activity` (`sky`).
3. `supersonic-trail-tuner`: Menu & `SURFACE_META` had `Zap` (`purple-400`); window/stack showed `Activity` (`sky`).
4. `demolish-shockwave-tuner`: Menu & `SURFACE_META` had `Sparkles` (`amber-400`); window/stack showed `Activity` (`sky`).
5. `dynamic-training-pack`: `SURFACE_META.icon` was invalid `'UsersRoundCollaboration'`; window/stack showed `Activity` (`sky`).
6. `telestrator-toolbar`: Menu & `SURFACE_META` had `Pen` (`emerald-400`); window/stack showed `Activity` (`sky`).

### 3.2 Icon & Color Mismatches Between Menu, `SURFACE_META`, and `FloatingWindowIcon.tsx`
- `engine-audio-tuner`: `Volume2` `emerald-400` in menu vs `Volume2` `amber-500/400` in window/stack.
- `network-stress-injection`: `Router` `rose-400` in menu (`Wifi` in `SURFACE_META`) vs `Router` `violet-500/400` in window/stack.
- `performance-boost-tuner`: `Sparkles` `amber-400` in menu (`Flame` in `SURFACE_META`) vs `Sparkles` `amber-500/400` in window/stack.
- `kickoff-training-tuner`: `PencilRuler` `sky-400` in Active Tuners vs `Crosshair` `amber-500` in `Layer3KickoffTrainingRecipe.tsx` vs `Gauge` in `SURFACE_META` vs `PencilRuler` `sky-500/400` in window/stack.
- `vehicle-global-fx-tuner`: `Car` `emerald-400` in Calibrated Presets (`Palette` in `SURFACE_META`) vs `Car` `amber-500/400` in window/stack.
- `experimental-bot-pack`: `Sparkles` `teal-400` in Sandboxes & `SURFACE_META` vs `Bot` `emerald-500/400` in window/stack.
- `replay-to-rcs-importer`: `FileArchive` `cyan-400` in Sandboxes (`undefined` in `SURFACE_META`) vs `Film` `amber-500/400` in window/stack.
- `network-monitor`, `performance-monitor`, `floating-window-opacity`: `AdvancedTab.tsx` omitted `iconColor` (neutral gray in menu) while window/stack used `Router` (`violet`), `Activity` (`sky`), `Sliders` (`amber`); `SURFACE_META.icon` also differed (`wifi`, `cpu`, `sliders-horizontal`).
- `goal-barrier-configurator`, `diagnostic-dump`, `experimental-multi-bots`, `atmosphere-fx-tuner`, `dual-arena-determinism-harness`, `static-training-ball-cam-tuner`, `reachability-tuner`, `camera-determinism`: `SURFACE_META.icon` either differed from menu/window (`brick-wall-shield`, `FileArchive`, `Users`, `Sun`, `CheckCircle2`) or was `undefined`.

### 3.3 Title Mismatches (`SURFACE_META.title` vs `PresetConfig.title`) (6 Windows)
1. `name-tag-tuner`: `SURFACE_META` = `"3D Nameplate & Off-Screen Indicator Tuner"` vs `PresetConfig` = `"Multiplayer Name Tag Tuner"`
2. `goal-volume-tuner`: `SURFACE_META` = `"Goal Volume & Rim FX"` vs `PresetConfig` = `"Goal Volume Hitbox Active Tuner"`
3. `vehicle-model-offset-tuner`: `SURFACE_META` = `"Vehicle Offsets & Hitbox"` vs `PresetConfig` = `"Vehicle Model & Offset Tuner"`
4. `vehicle-global-fx-tuner`: `SURFACE_META` = `"Vehicle Global FX"` vs `PresetConfig` = `"Vehicle Global Surfaces & FX Tuner"`
5. `boost-pad-tuner`: `SURFACE_META` = `"Boost Pad Tuner"` vs `PresetConfig` = `"Boost Pad Visual Tuner"`
6. `supersonic-trail-tuner`: `SURFACE_META` = `"Supersonic Wheel Trail"` vs `PresetConfig` = `"Supersonic Wheel Trail Tuner"`

---

## 4. Non-Spawn Menu Actions & Legitimate Exceptions

### 4.1 Multi-Bench 7:1 Sandbox (`subsystem-isolation-lab`) — Legitimate Exception
- `SANDBOX_BENCH_SPECS` in `src/ui-core/recipes/dev/devSandboxBenchesCatalog.ts` defines 7 distinct test benches (`probes`, `multi-ghost`, `state-replay`, `90s-match`, `goal-replay`, `kickoff-duel`, `demolish-respawn`) that all map to `SUBSYSTEM_ISOLATION_LAB_PRESET`.
- **Assessment**: **Should remain distinct in the Sandboxes menu.** Each row launches a different isolated physics/match bench and switches the active tab inside the shared multi-tab floating window, so having distinct bench titles, descriptions, and icons in the Sandboxes list is semantically correct.

### 4.2 Sandbox / Mode Launchers with Direct Spawn
- `experimental-multi-bots` & `experimental-bot-pack` (`Layer3DevSandboxesRecipe.tsx`, `ExpBotsMatchOptionsAndPovSection.tsx`, `runtimeModeBridgeHandlers.ts`): Clicking the row navigates to the sub-menu; clicking the launch button or entering the mode spawns the corresponding floating window minimized.
- `replay-to-rcs-importer` & `replay-viewer` (`Layer3SelfTrainingRecipe.tsx`, `runtimeModeBridgeHandlers.ts`): Entering the mode directly spawns the corresponding window.
- `kickoff-training-tuner` (`runtimeModeBridgeHandlers.ts`) & `dynamic-training-pack` (`runtimeDynamicTrainingBridge.ts`): Entering the mode directly spawns the window.
- **Assessment**: Because `FloatingWindowIcon.tsx` now resolves icons/colors directly from `PRESET_WINDOWS` by `id`, any direct `floatingStore.spawnWindow({ ...PRESET })` or `ensureFloatingWindowSpawned(PRESET)` automatically displays the exact canonical icon & color in both the window header and minimized stack tray without changing mode-entry control flow.

### 4.3 Custom Non-`FloatingWindowCard` Launchers
- `TelestratorDeveloperSection.tsx`: Uses `SettingActionRow` with `toggleFloatingPreset(TELESTRATOR_TOOLBAR_PRESET)` and dynamic active state label (`3D Telestrator Overlay (Active)`). Its `iconColor` is now bound to `TELESTRATOR_TOOLBAR_PRESET.iconColor`.
- `StaticTrainingVariationsModal.tsx`: Contains a secondary quick-launch button (`"3D Arrow & Visuals Tuner"`) that opens `STATIC_TRAINING_BALL_CAM_TUNER_PRESET` and closes the modal.
- `ManageRoomRosterAndActionsSection.tsx`: Online room roster action directly spawns `NETWORK_MONITOR_PRESET`.

---

## 5. Architecture & Changes Implemented

1. **Single Source of Truth on `PresetConfig`** (`floatingPresets.ts`, `devFloatingPresetFactory.ts`, `devFloatingPresets.ts`):
   - Added mandatory `iconName: string` and `iconColor: string` to `PresetConfig` and `DevPresetSpec`, populated across all 38 presets (7 user + 31 dev).
   - Added `FileArchive`, `History`, and `Pen` to `src/ui-core/primitives/iconLookup.tsx`.
2. **Deterministic `FloatingWindowIcon` Resolution** (`src/ui-core/navigation/FloatingWindowIcon.tsx`):
   - Replaced the 23-branch `if/else` heuristic with `PRESET_WINDOWS` lookup + `lookupLucideIcon(preset.iconName)` + `preset.iconColor`. Window headers (`FloatingWindowInstance`), minimized stack pills (`FloatingStackIcon`), and ghost flights (`FlyingGhostElement`) now share the exact same icon and color.
3. **Automatic Inheritance in `FloatingWindowCard` & Catalogs**:
   - `FloatingWindowCard.tsx` now defaults `icon` and `iconColor` from `preset.iconName` and `preset.iconColor`.
   - Removed redundant/mismatched overrides in `AdvancedTab.tsx`, `DeveloperTab.tsx`, `Layer3TrainingConfigRecipe.tsx`, `Layer3BallTrajectoryRecipe.tsx`, `Layer3KickoffTrainingRecipe.tsx`, `Layer3DevQuickActionsRecipe.tsx`, and aligned `TelestratorDeveloperSection.tsx`, `devHubSurfaceCatalog.ts`, `devHubExperimentalCatalogEntries.ts`, and `devSandboxBenchesCatalog.ts`.
4. **Aligned `SURFACE_META` & Radar Static Enforcement**:
   - Added `iconColor?: string` to `SurfaceMeta` (`surfaceMetaTypes.ts`) and aligned `title`, `icon`, and `iconColor` across all 38 `*FloatingWindow.tsx` files (and `useFloatingWindowsCalibration.ts`).
   - Updated `scripts/radar/radarUiAndRepoGuards.ts` so `presetBlockRegex` matches both user presets and `makeDevPreset({ ... })` dev presets, enforcing 1:1 parity of `title`, `icon`/`iconName`, and `iconColor` between `PresetConfig` and `SURFACE_META` on every `pnpm run check`.
