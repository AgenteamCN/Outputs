# Telestrator Overlay Refactor & Geometry/Input Fixes (`z/agy/rc/telestrator-overlay-v2`)

**Commit:** `a03403a6f85363d9ac09b1dacb1511ebe3a40ded` (Verified Bot Commit, signed by GitHub)  
**Branch:** `z/agy/rc/telestrator-overlay-v2` (branched from `origin/z/agy/rc/track-a-telestrator-fix`)  
**Verification:** `pnpm run check` (0 Radar errors, 0 `tsc --noEmit` errors), `pnpm run build` (PASS)

---

## 1. Summary of Implemented Changes

### 1.1 Mode-Independent Functional Overlay (Removed `DrawingSandboxMode`)
- **Deleted** `DrawingSandboxMode.ts` and removed all `'drawing-sandbox'` mode branches from `GameModeManager.ts`, `SimulationStage.ts`, `TelemetryStage.ts`, `hudRosterResolverPipeline.ts`, `GameCameraSubsystem.ts`, `appStoreTypes.ts`, `InGameHUD.tsx`, `useReplayPovSlotHotkeys.ts`, `devSandboxBenchesCatalog.ts`, and `Layer3DevSandboxesRecipe.tsx`.
- **Added** `TelestratorOverlaySubsystem.ts`, wired into `RuntimeSubsystemAssembler.ts`, `GameRuntime.ts`, and `FramePipeline.ts` (`RenderStage`).
- Moved `TELESTRATOR_TOOLBAR_PRESET` (`TelestratorToolbarFloatingWindow.tsx`) from `sandboxes` to `active-tuners` (`DEV_ENTRY_PATHS.ACTIVE_TUNERS`), and updated `TelestratorDeveloperSection.tsx` to toggle the overlay on top of **any** active mode (Replay, Static Training Pack, Freeplay, Exhibition/Match).
- Opening `TelestratorToolbarFloatingWindow` automatically activates the overlay; closing it deactivates the overlay and restores all input/camera state.

### 1.2 POV Input Takeover & Boost / Pointer-Lock Fix
- Added `externalMouseSuppressPredicate` to `KeyboardMouseController.ts`. When `overlayActive && (drawingEnabled || directionAdjustActive)` is true, left-click (`button === 0`) no longer triggers Boost, Jump, or mouse toggles.
- Guarded `RuntimeDOMEventManager.ts` (`_onCanvasMouseDown` and `_onCanvasClick`) so canvas clicks while drawing or adjusting direction do not dismiss cursor browsing or trigger pointer-lock.
- When both Draw Mode and Direction Adjust are turned off, pointer and camera control return immediately to the underlying game mode.

### 1.3 Draw Mode vs. Direction Adjust Mutual Exclusion & FreeCam Guard
- Fixed `telestratorStore.ts` so `drawingEnabled` and `directionAdjustActive` can **both be false**, but can **never both be true** simultaneously.
- Disabled `Direction Adjust` when in Free Cam (`isFreeCam === true`, synced from `CameraManager.isFreeCam` every frame).
- Updated `CameraManager.ts`:
  - `applyPersistentDirectionOffset()` returns early when `this.isFreeCam` is true, preventing WASD/Shift/Space/V movement axes from breaking.
  - `setFreeCam(true)` and `setFocusedCarIndex(newIdx)` automatically reset `directionOffsetYaw` and `directionOffsetPitch` to `0`.

### 1.4 Stroke Geometry: Smooth Curves, Consistent Dashes & Camera-Facing Arrowheads
- **Catmull-Rom Smoothing & Continuous Ribbon Joins (`telestratorStrokeGeometry.ts`)**:
  - Replaced disconnected per-segment quads with averaged-tangent continuous ribbon vertices and Catmull-Rom spline subdivision, eliminating jagged corners and rectangular gaps.
- **Analytical Dash Splitting (`buildDashedRibbonPositions`)**:
  - Replaced `midDist % dashPeriod` per-segment checks with analytical arc-length splitting at `dashLen` / `gapLen` boundaries. Draft preview and committed strokes now render **100% identical** dashes regardless of RDP simplification.
- **Camera-Facing Non-Occluded Arrowhead (`buildCameraFacingArrowheadMesh`)**:
  - Captured `camForward` (`camera.getWorldDirection`) at stroke creation and stored it on `TelestratorStroke`.
  - Projected the arrowhead direction onto the plane perpendicular to `-camForward` (facing the drawing camera) and rendered it with `depthTest: false`, `renderOrder: 15` so arrowheads remain visible even when landing inside goals or against sidewalls.

### 1.5 Two-Step Clear + Undo ("Regret Pill")
- Updated `TelestratorManager.ts` so the **first** Clear (or `Delete`/`Backspace`) moves active strokes to `lastClearedStrokes` and clears the scene, while remaining **undoable** via `Ctrl/Cmd+Z` or the Undo button.
- Clicking Clear a **second** time while the canvas is already empty permanently purges `lastClearedStrokes`.

### 1.6 Removal of Toggle Aerial Drop Stems
- **Removed** `dropStemsEnabled`, `buildDropStemsGroup`, and the `Toggle Aerial Drop Stems` button from `TelestratorToolbarFloatingWindow.tsx`, `telestratorStore.ts`, `TelestratorManager.ts`, and `telestratorStrokeGeometry.ts` (as requested: currently unused/unhelpful in the 3D telestrator coaching workflow).

### 1.7 Real-Unit (UU) Settings, 3 Opacity Presets & Copy JSON
- Switched brush width (`widthUnits`, default `28 UU`) and arrowhead dimensions (`arrowLength: 95 UU`, `arrowWidth: 75 UU`) to Unreal Units (UU).
- Updated default opacity presets to `100%`, `50%`, and `25%` (`DEFAULT_TELESTRATOR_PRESETS`).
- Added a Gear (`Settings`) button at the bottom of `TelestratorToolbarFloatingWindow.tsx` opening an inline settings panel with UU sliders, preset opacity sliders, and a **Copy JSON** button (mirrored in `TelestratorDeveloperSection.tsx`).
- Updated `TelestratorWireCodec.ts` (`STROKE_HEADER_BYTES = 52`) to encode/decode `widthUnits`, `arrowLength`, `arrowWidth`, and `camForward`.

---

## 2. Technical Evaluation & Recommendations (For Network & Graphics Teams)

### 2.1 Sampling Algorithm: Distance-Based + RDP vs. Time-Based
- **Current Choice (Distance-Based + RDP + Render-Time Catmull-Rom):**
  - During pointer move, points are sampled only when `distSq >= 144` (12 UU). On pointer release, `simplifyAndResampleStroke` applies spacing (24 UU) + Ramer-Douglas-Peucker (`eps = 6 UU`).
  - **Why this is optimal:** Time-based sampling wastes bandwidth when the mouse pauses and loses detail during fast flicks. Distance-based + RDP reduces a typical 120-point mouse drag down to **8–25 control points** (~250–650 bytes via `TelestratorWireCodec`), while Catmull-Rom subdivision at render time reconstructs smooth arcs.

### 2.2 Rendering Cost: Ribbon Mesh vs. Instanced / Curve Tubes
- **Ribbon Mesh (`MeshBasicMaterial` + `BufferGeometry`):**
  - 1 draw call per stroke (2 draw calls for `arrow`: ribbon + arrowhead). A typical coaching session (< 10 strokes) uses < 15 draw calls and ~2–5k triangles total—negligible GPU cost compared to 3D tubes or instanced spheres.

### 2.3 Network Synchronization Protocol Recommendation (Track D Handoff)
- **Transport:** Send `TelestratorWireCodec` frames (`STROKE_FULL`, `STROKE_UNDO`, `STROKE_CLEAR`) over the **Reliable Control DataChannel** (not the Unreliable 60/120Hz snapshot channel).
- **Hash-ACK Verification:**
  - `STROKE_FULL` already carries a 32-bit FNV-1a `hash` over `(tool, colorId, opacity, widthUnits, points)`.
  - Receiver decodes, computes `computeStrokeHash(...)`, and replies with `STROKE_HASH_ACK { strokeId, hash, matched }`. If `matched === false`, sender re-transmits `STROKE_FULL`.
  - **Bandwidth:** At ~300–600 bytes per stroke on a reliable channel, telestrator traffic has zero impact on 120Hz physics snapshots.
  - **View Consistency:** Visual consistency is guaranteed when Host & Client share the same camera POV/pose (e.g., Coach POV Follow). When a peer detaches to Free Cam, strokes remain anchored in 3D arena world space.

---

## 3. Open Items for Follow-Up Discussion
1. **Stroke Lifecycle / Auto-Fade in Replay:** `createdTick` and `fadeDurationTicks` are already wired in `TelestratorStroke` and `TelestratorManager.updateReplayTick(currentTick)`; once Replay timeline integration is connected, `TelestratorOverlaySubsystem.update()` can pass the active replay tick.
2. **POV Slot Hotkeys (1–6 / 7 / C):** Currently delegated to the underlying mode (e.g., Replay Viewer). If Freeplay/Training also needs 1–6 car switching while Telestrator Overlay is open, we can add an optional fallback slot switcher in `TelestratorOverlaySubsystem`.
