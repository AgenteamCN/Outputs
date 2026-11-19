# Wave 1 Track A-1 Report: Fix Bug 1 (Goal Replay Client Static Without Dynamics)

## Branch & Commit
- **Base branch**: `origin/z/agy/rc/multiplayer-recheck`
- **Created branch**: `fix/bug-1-goal-replay-client-dynamics`
- **Commit**: `05c528e` — `fix(multiplayer): stream Goal Replay frames and drive client replay timeline (Bug 1)`
- **Push status**: Local commit created; `git push -u origin fix/bug-1-goal-replay-client-dynamics` could not complete in this sandbox because no GitHub credentials/token are configured (`fatal: could not read Username for 'https://github.com': terminal prompts disabled`).

## Root Cause & Changes
1. **`src/game/pipeline/stages/SimulationStage.ts`**
   - During `flags.isGoalReplay`, `flags.shouldHoldAccumulator` is `true`, which previously caused `isPaused` to be passed as `true` to `HostBroadcastPublisher.publishIfReady(...)`.
   - Updated `isHostBroadcastPaused` to exclude `flags.isGoalReplay` (`(flags.shouldHoldAccumulator && !flags.isGoalReplay) || flags.isReplayPaused || isSpectatePlaybackPaused()`) and passed `isGoalReplay: flags.isGoalReplay` to `publishIfReady(...)`.

2. **`src/game/pipeline/stages/HostBroadcastPublisher.ts`**
   - Added `isGoalReplay?: boolean` to `publishIfReady(...)`.
   - Because `applyReplayFrameToEngine(...)` restores historical frames with `{ preserveEngineTick: true }`, `physics.getHeaderView().tickCount` and `snap[0]` remain constant during Goal Replay.
   - When `isGoalReplay && !isPaused`, `publishIfReady` now advances `publishTick` monotonically (`Math.max(currentTick, this.lastPublishedTick + tickStep)`) and writes `snap[0] = publishTick` before calling `server.pushBroadcastSnapshot(snap, publishTick, acknowledgedControls)`, allowing both Host broadcast and Client `SpectateInterpolationBuffer` / `ReplayTimelineController` to ingest consecutive Goal Replay frames.

3. **`src/game/pipeline/stages/simulationNetworkReconcilePipeline.ts`**
   - In the `isPureCsrSpectate && !reconciler.replayPlaybackMode && spectateInterpolationBuffer` branch, when `flags.isGoalReplay && replayTimelineController` is active, drained packets now also initialize `replayTimelineController.beginReplaySession(...)` (if not active) and call `replayTimelineController.appendStreamedReplayFrame(...)` alongside `spectateInterpolationBuffer.pushFrame(...)`.

## Verification
- **`pnpm exec prettier --check`**: Passed on all 3 modified files.
- **`pnpm run check`** (`radar` + `tsc --noEmit`): Passed (`STATUS: GREEN -- 100% TS, tsc --noEmit clean, hard boundaries & volume caps verified`).
- **`pnpm run repro:dev-broadcast-v3`**:
  - Client `spectateInterpolationBuffer` `bufferedFrames` now grows during Goal Replay (`12 -> 22 -> 31 -> 40 -> 46 -> 52`, and `64` in extended sampling) instead of staying stuck at `1`.
  - Client `replayTimelineController.sessionState.isActive` is `true` with `activeFrames` growing (`24 -> 37 -> 43 -> 45 -> 81`), and on Goal Replay exit (`skipGoalReplay()`), `override` returns to `false` and `camMode` returns to `"pov"`.
  - **Note on `distinctBallPositions` in `runDevBroadcastReproV3.mjs`**: In `runDevBroadcastReproV3.mjs`, the Host sits idle at `[0, 0, 93.2]` while the Client boots/joins, so `extractGoalReplayFrames` (`lookbackSeconds: 10.0`) captures ~1,150 idle frames at `[0, 0, 93.2]` before the `setBallState` goal shot (`totalFrames: 1351, firstMoved: 1151`). Because `runDevBroadcastReproV3.mjs` samples 6 × 200ms (`1.2s`) immediately at the start of Goal Replay, both Host and Client are still in the `[0, 0, 93.2]` kickoff-idle prefix of the replay (`distinctBallPositions: 1` in that 1.2s window).
