# Track C: Dynamic Training Pack (.rcd) Implementation & Architecture Report

- **Branch**: `z/agy/rc/track-c-dynamic-training`
- **Verified Commit**: `5aab06257b65d6a39128dcb498e23eb62e794efa` (`verified: true`, `state: VALID`)

---

## 1. Implemented Architecture & Deliverables

### 1.1 Independent StateSession + Linked-List `.rcd` Pack Schema
- `dynamicTrainingPackTypes.ts`:
  - Separates recorded physics slices (`DynamicTrainingStateSession`, 4s @ 120Hz = 480 frames) from pack definitions (`DynamicTrainingPackRecord` / `RCDPackArchive`).
  - Uses `prevShotId` linked-list ordering (`sortDynamicShotsByPrevLink`, `relinkDynamicShotChain`) so shots reference independent `sessionId` entries without duplicating heavy frame data during editing.
  - Defines per-car takeover rules (`DynamicCarTakeoverRule`: `'player' | 'disappear' | 'as-previous' | 'model' | 'no-input'`).
- `dynamicTrainingPackCodec.ts`:
  - Self-contained `.rcd` (`RCDPack`) JSON encoder/decoder that bundles pack metadata, ordered shot references, and all referenced `DynamicTrainingStateSession` payloads on export, and hydrates them back into the session pool on import.
- `dynamicTrainingPackStorage.ts` & `dynamicTrainingPackStore.ts`:
  - IndexedDB async storage (`rc-dynamic-training-db`) with `state_sessions`, `dynamic_packs`, and `shot_stats` object stores + in-memory fallback.
  - Supports `captured_from_replay` default folder, `moveShotBetweenPacks`, `duplicateDynamicShotToPack`, reference-counted `deleteShotWithSessionCheck`, and persistent attempt/goal/timeout stats.

### 1.2 4s Replay Capture & Exact Countdown-to-Intervene Takeover
- `DynamicTrainingSliceExtractor.ts`:
  - Extracts the 4-second (480 frames @ 120Hz) pre-roll `DynamicTrainingStateSession` before the cursor/intervene point from `RLRCConvertedReplaySession`, and hydrates sessions back to `RLRCFrame120Hz[]` + `RLRCPlayerSlotMeta[]`.
- `RLRCReplayTypes.ts`, `RLRCInterveneTargetFrameApplier.ts`, `RLRCInterveneNonPovSyncPipeline.ts`, `RLRCInterveneTakeoverSubController.ts`:
  - **Countdown vs. Intervene Fix**: During `preroll-countdown` (`countdownRemainingTicks > 0`), all active cars remain visible and follow their original recorded state. At the exact moment `launchShotNow()` begins accepting user input (`countdownRemainingTicks === 0`), cars configured as `'disappear'` are deactivated and `'bot-takeover'` / `'model'` slots begin AI control.
  - **Per-Slot Takeover Routing**: Added `perSlotModes` and `perSlotModelIds` to `RLRCInterveneLaunchConfig` and `decideControlsForSlot(..., modelId)`.

### 1.3 Dynamic Training Mode Runner & Simplified Judge
- `DynamicTrainingJudge.ts`:
  - Simplified outcome evaluator strictly checking **Goal scored** (team 0 / team 1) or **Timeout** (`1..60s`).
  - Optional controllable-car-only spawn variation (`applyOptionalCarSpawnVariationToFrame`), disabled by default to preserve scenario determinism.
- `DynamicTrainingMode.ts` & `GameModeManager.ts`:
  - Registered under `'dynamic-training-pack'`. Drives 4s pre-roll countdown, exact intervene takeover, live 120Hz stepping, and Goal/Timeout persistence.

### 1.4 Developer Sandbox UI & Replay Importer Integration
- `DynamicTrainingPackFloatingWindow.tsx`, `DynamicTrainingPackManagerSection.tsx`, `DynamicTrainingShotEditorCard.tsx` (following `src/ui-core/AGENTS.md` layer rules):
  - Pack selector, `.rcd` import/export, create pack, shot list with Move / Duplicate / Delete (with double-check confirmation).
  - Shot editor: Timeout, Countdown (`0..3s`), Intervene Offset (frames), Per-Car Takeover Matrix (`Player` / `As Previous` / `Disappear` / `AI Model` / `No Input`), optional POV spawn variation toggle, and Play/Restart/Stop controls.
- `ReplayToRcsImporterFloatingWindow.tsx`, `replayToRcsImporterStore.ts`, `RLRCReplaySessionController.ts`, `runtimeDynamicTrainingBridge.ts`:
  - Added **"Put 4s Pre-Roll into Dynamic Training Pack (.rcd)"** button to capture directly from any loaded `.replay` cursor frame into `captured_from_replay`.

---

## 2. Verification Results
- `pnpm run format:check`: **PASS**
- `pnpm run check` (`radar` + `tsc --noEmit`): **PASS** (1364/1364 TS files, 0 circular deps, 0 boundary violations, 0 tsc errors)
- `pnpm run build`: **PASS** (`vite build` succeeded cleanly)

---

## 3. Open Items for Follow-Up Discussion
1. **Scratch Authoring Workflow**: Recording sequential car trajectories starting from a static ball (spawning car 1 -> recording -> spawning car 2 at a chosen timestamp/pose while keeping car 1's recorded actions) requires a multi-pass recording state machine on top of `DynamicTrainingStateSession`.
2. **Per-Slot Multi-Model Inference**: `perSlotModelIds` is now plumbed through `RLRCInterveneNonPovSyncPipeline` to `decideControlsForSlot(..., modelId)`, but `RLRCInterveneEnvPortBuilder` / `BotCoordinator` currently routes a single active bot policy; multi-model simultaneous inference per slot needs `BotCoordinator` support if different cars use different models (e.g., Nexto vs. Necto) in the same shot.
3. **Online Multiplayer Takeover**: Assigning or swapping which connected client controls which `slotIndex` in a `.rcd` session depends on Track D/F `RoomReplicationBus` authority routing.
4. **Spawn Variation Policy**: Currently off by default for dynamic packs; decide whether controllable-car-only jitter should ever be enabled per-pack or kept strictly opt-in per-shot.
