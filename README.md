# Track A Telestrator Overlay — Fixes & Stroke Thickness Strategy Report

- **Branch**: `z/agy/rc/track-a-telestrator-v3` (branched from `origin/z/agy/rc/telestrator-overlay-v2`)
- **Verified Bot Commit**: `541087fb581d72a15af3bb14794e0bf7511a3351` (`signature: VALID`, `isValid: True`)

---

## 1. Answering Your Question: How Previous Stroke Thickness Worked & How Broadcast Consistency Is Solved

### What the previous commit (`ddbd338`) did
`ddbd338` used **Perspective-Compensated Screen-Space Sizing** (`computeScreenRelativeWorldWidth`):
```ts
worldHeightAtDist = 2 * Math.tan(vFovRad * 0.5) * safeDist;
worldPerPx = worldHeightAtDist / viewportHeightPx;
worldWidth = widthPx * worldPerPx;
```
At a given camera distance $d$, it projected the desired screen pixel width into world units (`UU`) using the camera's vertical FOV (`camera.fov`) and `viewportHeightPx`. That is why strokes looked consistent on a single screen regardless of whether you drew near or far from the camera.

### Why `ddbd338` alone breaks across different resolutions in Broadcast (and why `a03403a` broke screen-relative sizing)
1. **`a03403a` regression**: Replaced `widthPx` with fixed world units (`widthUnits` UU), which caused strokes drawn far away to appear tiny and strokes drawn close to appear huge.
2. **`ddbd338` multi-resolution issue**: Dividing by `viewportHeightPx` (actual window height) means `10 px` on a 720p screen is `10 / 720 = 1.39%` of screen height, whereas on a 4K (2160p) screen it is `10 / 2160 = 0.46%` (3× thinner relative to the car and ball, which always scale with `vFov` / viewport height).

### New Strategy Implemented: **Normalized 1080p Reference Screen-Space Width (`widthRefPx`) + Per-Point Depth Compensation**
1. **Resolution-independent virtual reference height (`TELESTRATOR_REF_VIEWPORT_HEIGHT = 1080`)**:
   - Instead of `viewportHeightPx`, we normalize against a fixed virtual `1080p` reference height (`widthRefPx / 1080` of viewport height).
   - Because cars, the ball, and the arena all occupy the same fraction of viewport height across 720p, 1080p, 1440p, and 4K monitors (for a given `camera.fov`), **every viewer sees the exact same stroke thickness relative to the cars and ball regardless of their monitor resolution**.
2. **Per-point distance calculation along the stroke**:
   - `buildContinuousRibbonPositions`, `buildDashedRibbonPositions`, and `buildCameraFacingArrowheadMesh` compute `halfWidth` at **each point** using its distance to `camPos` (`computeNormalizedScreenWorldWidth(widthRefPx, dist, camFovDeg)`), so even a long stroke stretching from foreground to background maintains constant apparent thickness.
3. **Deterministic wire sync (`TelestratorWireCodec.ts`)**:
   - `TelestratorStroke` now carries `widthRefPx`, `arrowLengthRefPx`, `arrowWidthRefPx`, `camPos`, `camForward`, and `camFovDeg` (header expanded from `52` to `68` bytes). Both Host and Client compute identical `computeStrokeHash` values and identical geometry.

---

## 2. Track A Fixes Implemented

### A. Floating Window Top Padding, Left-Pinned Toolbar & Reachable Settings (`TelestratorToolbarFloatingWindow.tsx`, `TelestratorOverlaySubsystem.ts`, `devFloatingPresets.ts`)
- **Removed top blank padding**: Configured `contentPadding: { x: 4, y: 2 }` on `spawnWindow` and removed `py-1` inside `TelestratorToolbarFloatingWindow.tsx` so the toolbar sits flush under the header.
- **Left-pinned layout with right spacer**: Wrapped the layout in `flex flex-row items-start w-full`, keeping the vertical toolbar column (`shrink-0`) flush left and adding `<div className="flex-1 min-w-0" />` on the right so resizing the window wider never shifts the toolbar.
- **Inline Settings panel + auto-expanding window width**: Replaced the clipped `absolute left-full` popover with an inline Settings panel (`ml-2 w-[236px] shrink-0`) next to the toolbar, and `handleToggleSettings` automatically resizes `telestrator-toolbar` between `56px` (collapsed) and `310px` (expanded).

### B. Full 3D Raycasting Across Curved Fillets, Hex Walls, Cars & Ball (`arenaSurfaceRaycaster.ts`, `TelestratorOverlaySubsystem.ts`)
- `TelestratorOverlaySubsystem.getRaycastTargets()` now passes:
  - `dynamicTargets`: visible `arena.cars` and `arena.ball`
  - `stadiumTargets`: active visible stadium mesh (`driftwoodsStadium` or `throwbackStadium`)
- `raycastScreenToArena` checks `dynamicTargets` (cars/ball) and `stadiumTargets` (curved floor-wall fillets, goals, hex-grid walls) with arena-bounds filtering and exterior near-wall backface skipping when the camera is outside the arena, falling back cleanly to analytical `ARENA_PLANES`.

### C. Topmost Camera-Relative Layering (`telestratorStrokeGeometry.ts`)
- Set `depthTest: false` and `depthWrite: false` on `ribbonMesh` (matching `arrowMesh`), with `ribbonMesh.renderOrder = 100` and `arrowMesh.renderOrder = 105`.
- Strokes drawn over cars, the ball, curved fillets, or hex-grid walls always render on top without z-fighting or occlusion.

---

## 3. Verification

- `pnpm run check` (`radar.ts` + `tsc --noEmit`): **PASS** (`STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified.`)
- `pnpm run build`: **PASS** (`built in 28.68s`)
- `npx prettier --write` on all modified files: **PASS**
