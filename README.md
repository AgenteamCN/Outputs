# DD-Track C-2 (`fix/bug5-remaining-geometry-leak`) 完成报告

## 1. 问题定位（`retainedGeometries: 3` 的精确来源）
在基线 `origin/z/agy/rc/multiplayer-recheck-base`（`7435735`）上，通过 Headless Chrome + WebGL2 拦截 `THREE.BufferGeometry` 的 `addEventListener('dispose')` 与 `dispose()` 事件，对比 `bootMem` 与 `onReturnToFreeplay()` 后的 `afterExitMem`，定位到残留 `+3` 个 WebGL Geometry 的精确来源：
1. **`BoostOrbShadowBillboard`（`+2` 个 Geometry）**：`src/entities/arena/BoostPadShadersAndMeshes.ts` 中 `createBigOrbShadowMesh` 为场上 6 个 Big BoostPad 各自新建独立的 `new THREE.PlaneGeometry(1, 1)`。启动时仅 4 个在相机视锥内被上传至 WebGL；进入 Replay Viewer 移动视角后，其余 2 个进入视锥并上传至 `renderer.info.memory.geometries`（且 `BoostPadSystem` 从未单独 dispose 该几何体）。
2. **`BallTrajectoryTubeRenderer`（`+1` 个 Geometry）**：`src/entities/trajectory/BallTrajectoryTubeRenderer.ts` 在启动时创建 `tubeGeometry`、`bounceInstancedMesh.geometry` 和 `groundDropGeometry`，但初始因球静止而 `visible = false` 未上传至 WebGL。进入 Goal Replay / 运动阶段渲染后上传至 WebGL；退出回 Freeplay 调用 `BallTrajectoryPredictor.clear()` -> `hideAll()` 时仅设置 `visible = false`，未调用 `dispose()` 释放 WebGL GPU 缓冲。

## 2. 修复方案与改动文件
- **`src/entities/arena/BoostPadShadersAndMeshes.ts`**：
  - 新增 `getSharedBigOrbShadowGeometry()` 单例，使所有 6 个 Big BoostPad 的 `BoostOrbShadowBillboard` 共享同一个 `THREE.PlaneGeometry(1, 1)`，避免视角变化后产生额外几何体上传并减少冗余 GPU 分配。
- **`src/entities/trajectory/BallTrajectoryTubeRenderer.ts`**：
  - 新增 `_gpuBuffersAllocated` 标记，在 `bounceInstancedMesh`、`groundDropLines`、`preOcclusionMesh`/`foregroundMesh` 变为可见时置为 `true`。
  - 在 `hideAll()` 中检测到 `_gpuBuffersAllocated` 或任意网格可见时，主动调用 `this.tubeGeometry.dispose()`、`this.bounceInstancedMesh.geometry.dispose()`、`this.groundDropGeometry.dispose()` 释放 WebGL GPU 缓冲（后续再次显示时由 Three.js 自动重新上传）。

## 3. 验证结果
- `pnpm run check`：**PASS**（1430/1430 TS，0 循环依赖，`tsc --noEmit` 洁净）。
- `pnpm run test:multiplayer-suite`：**PASS**（全部单测通过）。
- `pnpm run repro:dev-broadcast-v3`：
  - `bug-5-high-memory-and-unreleased-on-exit`: **`PASS_FIXED`**（`bootMem.geometries: 37`, `loadedMem.geometries: 65`, `afterExitMem.geometries: 34`, `retainedGeometries: -3`, `retainedWasmMB: 0`）。

## 4. Verified Bot Commit
- 分支：`fix/bug5-remaining-geometry-leak`（基于 `7435735`）
- Commit：`0d9d4accb5e5088956b1ea4a643d0b4afeb97d5f`（`signature.isValid: true`, `state: "VALID"`, `Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>`）
- 链接：https://github.com/Antigravi-ty/RocketCompanion/commit/0d9d4accb5e5088956b1ea4a643d0b4afeb97d5f
