# Track A Delivery Report: 3D Spatial Telestrator, Camera Direction Adjust & Drawing Sandbox

- **Branch**: `z/agy/rc/track-a-telestrator-sandbox`
- **Verified Bot Commit**: [`c348169433bf9b6bbae5d8bbca1f0b2ccf61c7f0`](https://github.com/Antigravi-ty/RocketCompanion/commit/c348169433bf9b6bbae5d8bbca1f0b2ccf61c7f0) (`signature.isValid: true`)
- **Validation**: `pnpm run format:check` (PASS), `pnpm run check` (`radar.ts` + `tsc --noEmit`: GREEN, 0 errors, 0 circular deps, 0 upward imports), `pnpm run build` (PASS).

## Implemented Subsystems

### 1. 3D Telestrator Core (`src/entities/telestrator/`)
Strictly isolated to `three` (zero `src/game/` or `src/network/` imports):
- `telestratorTypes.ts`: Brush tools (`solid`, `dashed`, `arrow`), 6-color palette (`dark-grey`, `white`, `red`, `yellow`, `bright-green`, `dark-green`), 3 opacity/width presets (`100%`, `70%`, `50%`), and stroke/point interfaces.
- `arenaSurfaceRaycaster.ts`: Analytical CarBall arena surface raycaster (`raycastArenaSurface`, `raycastScreenToArena`) covering floor (`y=0`), ceiling (`y=2044`), side/end walls, 45-degree corner bevels, and goal boxes, with optional custom mesh targets.
- `telestratorStrokeGeometry.ts`: Fixed-distance resampling + Ramer-Douglas-Peucker polyline simplification (`simplifyAndResampleStroke`), FNV-1a 32-bit stroke hash (`computeStrokeHash`), surface-hugging ribbon & arrowhead mesh builder (`buildStrokeRenderGroup`), and aerial drop stems + ground shadow rings.
- `TelestratorManager.ts`: Stroke lifecycle, screen-relative world width scaling (`computeScreenRelativeWorldWidth`), undo/clear, drop-stem toggling, and replay tick binding (`updateReplayTick`).

### 2. Wire Protocol Codec (`src/network/protocol/TelestratorWireCodec.ts`)
- Standalone binary codec (`encodeTelestratorStroke`, `decodeTelestratorStroke`, `encodeTelestratorHashAck`, `decodeTelestratorHashAck`) packing Float32 world coordinates & surface normals with FNV-1a hash ACK verification, without modifying `HostWorkerProtocol.ts` or `DataChannelTransport.ts`.

### 3. Camera Extensions (`src/camera/CameraManager.ts`, `src/camera/FreeCameraController.ts`)
- **Persistent Direction Adjust**: `setDirectionAdjustActive`, `applyDirectionOffsetDelta`, `resetDirectionOffset`, and `getDirectionOffsetDeg` rotate camera heading/pitch while keeping XYZ position fixed and persisting across frames.
- **Coach POV Follow & Cycling**: `setCoachFollowCarIndex` and `cycleCameraView(carCount)` for `C`-key cycling across cars `1-6` and Free Camera (`7`), automatically resetting angular offset on view switch.
- **Drag Suppression**: `FreeCameraController.shouldSuppressDrag` prevents free-cam rotation while drawing or Direction Adjust is active.

### 4. UI Toolbar, Store & Developer Settings (`src/ui-core/`, `src/ui-system/`)
Following `src/ui-core/AGENTS.md` unidirectional layer rules:
- `telestratorStore.ts`: Zustand store for active tool, color, 3 configurable brush presets, Direction Adjust mutual exclusion, drop stems, and runtime handlers.
- `TelestratorToolbarFloatingWindow.tsx`: Narrow vertical icon toolbar (`width: 72`, `hideFooter: true`, `icon: 'Pen'`) with hover instruction bubbles, Draw vs Direction Adjust toggle, Restore POV button, 3 opacity presets, 3 brush types, 6 color swatches, Drop Stems toggle, Undo (`Ctrl/Cmd+Z`), and Clear All.
- `TelestratorDeveloperSection.tsx` mounted in `DeveloperTab.tsx`: Configures the 3 brush presets (opacity % and screen-relative width px) and launches Drawing Sandbox.
- `devSandboxBenchesCatalog.ts` & `Layer3DevSandboxesRecipe.tsx`: Added `drawing-sandbox` entry under Developer -> Sandboxes.

### 5. Drawing Sandbox Mode (`src/game/modes/DrawingSandboxMode.ts`)
- Registered as `'drawing-sandbox'` in `GameModeManager.ts` and wired through `runtimeUiPort.ts` / `runtimeEngineUiPort.ts`.
- Spawns 6 cars (3v3 kickoff formation), populates two-sided HUD player indicators, opens `telestrator-toolbar`, binds `1-6` (car POVs), `7` (Free Cam), `C` (cycle), `Ctrl/Cmd+Z` (undo), and exercises `TelestratorWireCodec` roundtrip verification on stroke commit.

## Remaining / Follow-up Notes for Integration
- Network transport hookup (Track D): `TelestratorWireCodec` is ready to be routed over the Reliable Control DataChannel once `RoomReplicationBus` is merged.
- UI polish: Hover bubble positioning and narrow vertical shell styling can be refined by the UI team in `TelestratorToolbarFloatingWindow.tsx`.
