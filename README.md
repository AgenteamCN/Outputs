# RocketCompanion Bootstrap Investigation & Fix Report

## 1. PR #11 (`d69e9471bfdbcd854c778e09d708b676a6c7fa76`) -> `alpha`
- Verified open PR #11 (`Antigravi-ty/z/agy/ui/fix-floating-window-schema` -> `alpha`) and merged it into `alpha` before starting investigation.
- Merge commit SHA on `alpha`: `e943d507b1c70d84b0b231ef457b4baaa3ab9525`.

## 2. Bootstrap Investigation Findings

### A. Why `0%` stayed stuck for a long time (worse than earlier versions)
1. **DOM not updated in `MinimumProgressLoader.onTaskStart`**: `onTaskStart()` updated `this.overallPercent` in memory, but never updated `#hig-overall-percent` or `#hig-primary-fill` in the DOM.
2. **Sequential heavy module graph loading in `main.ts`**: `src/main.ts` awaited `import("./game/GameBootstrap.ts")` and then sequentially awaited `import("./ui-system/index.tsx")`, with zero intermediate progress updates.
3. **Unwarmed dev endpoints & forced `optimizeDeps` rebuild**: `scripts/devLauncher.ts` only warmed up `entry` (`:3000`), leaving `rc-https` (`:3001`) and `rc-http` (`:3003`) cold (~18-23s in headless Chrome). `gitHeadAutoRestartPlugin.ts` called `server.restart(true)` (`forceOptimize = true`), forcing Vite to discard and rebuild `node_modules/.vite/<role>` on every Git HEAD switch.

### B. Why loading stalled at `~30%`, `~60%`, or `70%` (`Streaming ... (2/3 ready)`)
- **70% (`2/3 ready`)** corresponds to Stage 4 (`geometry.scene.build`, `50% + 30% * 2/3 = 70%`) inside `runParallelArenaAssetSteps` (`ArenaAssetLoader.ts`).
- `GLTFLoader.loadAsync`, `CameraMicrokernel` `fetch(wasmPath)`, `RocketSimMeshLoader.fetchArenaCollisionManifest`, and `realisticConeVisual` `TextureLoader.load` had no timeouts and no retries, so any stalled HTTP/2 stream or image load hung `Promise.all` indefinitely.

### C. Why manual refresh after Git branch switch stalled on Tab A while a New Tab B worked
- `gitHeadAutoRestartPlugin.ts` sent `rc:git-head-changed` and `full-reload` **before** `await server.restart(true)`, causing Tab A to reload into a tearing-down HTTP/2 server and `optimizeDeps` rebuild, whereas a new Tab B opened after restart succeeded cleanly.

## 3. Implemented Fixes (Branch `z/agy/rc/fix-bootstrap-stall`)
- `src/utils/resilientAssetLoad.ts`: Shared `fetchWithTimeoutAndRetry`, `loadGltfWithTimeoutAndRetry`, and `loadTextureWithTimeoutAndRetry` wired into `CameraMicrokernel.ts`, `RocketSimMeshLoader.ts`, `DriftwoodsStadiumLoader.ts`, `customGlbCarModel.ts`, and `BoostPadTemplateLoader.ts`.
- `src/entities/vehicle/boost/realisticConeVisual.ts`: 10s timeout guard on boost texture preload promises.
- `src/loader/StreamDownloadTracker.ts`: 3-attempt retry with 20s timeout per attempt in `fetchBuffer`.
- `src/loader/MinimumProgressLoader.ts` & `src/main.ts`: Render `overallPercent` immediately in `onTaskStart`, clamp monotonically, and parallelize `GameBootstrap` + `ui-system` imports with `2%` -> `5%` -> `8%` updates.
- `scripts/vite/gitHeadAutoRestartPlugin.ts` & `scripts/devLauncher.ts`: Wait for `server.restart(false)` before broadcasting reload, expose `ready` on `/__rc_git_head`, trigger `checkGitHead()` on `vite:ws:connect`, and warm up `rc-https` and `rc-http` in the background.

## 4. Verification & Git Artifacts
- `pnpm run check` (`radar` + `tsc --noEmit`): PASS (1355/1355 TS).
- `pnpm run build`: PASS.
- Headless Chrome bootstrap & reload test (`https://localhost:3001/?touch=2`): PASS.
- Verified Bot Commit: `4591563b84106a932db4514a731b2f449fb24ed6` (`VALID`) on `z/agy/rc/fix-bootstrap-stall`.
- PR #12: https://github.com/Antigravi-ty/RocketCompanion/pull/12
