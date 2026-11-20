# EE-Track E 完成报告 (`fix/bug2-nametag-render-stability`)

## 1. 交付信息
- **分支**: `origin/fix/bug2-nametag-render-stability`（基于 `origin/z/agy/rc/multiplayer-recheck-base` @ `7435735`）
- **Verified Bot Commit**: [`b669bf17a0493fbd467dcf8588dd603ad39c86f3`](https://github.com/Antigravi-ty/RocketCompanion/commit/b669bf17a0493fbd467dcf8588dd603ad39c86f3) (`signature.isValid: true`, `state: VALID`)
- **修改文件（严格限定在 Track E 独占边界内）**:
  - `src/entities/arena/VehicleNameTagSystem.ts`
  - `scripts/runDevBroadcastReproV3.mjs`

## 2. 根因分析与改动说明
1. **后台标签页 rAF 节流导致 `client.tags: []` (`scripts/runDevBroadcastReproV3.mjs`)**:
   - 原脚本在 REPRO 1 期间调用了 `hostCdp.send('Page.bringToFront')`，导致 `clientCdp` 处于后台标签页、`requestAnimationFrame` 被 Chrome 节流，REPRO 2 采样前 `client` 未完成 6 车状态下的 NameTag 渲染帧更新。
   - **修复**：在 Bug 2 采样前调用 `await clientCdp.send('Page.bringToFront')`，等待 `rt.physics?.getHeaderView?.()?.numCars === 6 && (nts?._visibleCards2D?.length ?? 0) > 0`，并再等待 2 个 `requestAnimationFrame` 完成渲染帧更新后再采样（未改动 Bug 2 断言逻辑）。
2. **模式切换时 NameTag 缓存清理 (`src/entities/arena/VehicleNameTagSystem.ts`)**:
   - 当 `shouldShowVehicleNameTags(...)` 返回 `false`（如 Goal Replay -> Freeplay -> Replay Viewer 切换过程）时，原有逻辑仅隐藏 `this.root` 与 overlay canvas，未清空 `_visibleSprites`、`_visibleCards2D`、`_offscreenIndicators2D`。
   - **修复**：在 `shouldShowVehicleNameTags` 返回 `false` 分支同步清空 `_visibleSprites.length = 0`、`_visibleCards2D.length = 0`、`_offscreenIndicators2D.length = 0`，避免模式切换残留旧帧卡片。

## 3. 门禁与实测结果
- `pnpm run check`：**PASS**（1430/1430 TS，`tsc --noEmit` 通过，`NameTagScatter=1/1`，0 循环依赖，0 hard errors）
- `pnpm run test:multiplayer-suite`：**PASS**（全部单测通过）
- `pnpm run repro:dev-broadcast-v3`（在独立分支 `b669bf1` 合入 Track A-1/A-2 前）：
  - `client.tags: []` 问题已消除：Client 已稳定渲染出 6 车同步后的 NameTag（`numCars: 6`, `arenaCarTeams: [0,0,0,1,1,1]`, `client.tags: [{"carIndex":1,"displayText":"Ducky_Blue_2"}]`）。
  - 注：当前独立分支尚未合入 Track A-1 / Track A-2（Bug 3 未修复，Client `tick` 仍滞后 Host，处于 `tick: 74` 而 Host 处于 `tick: 719`），因此双方视锥内可见车辆集合不同，导致 `tagTextMismatches`（`[2,3,4,5]` 在 Client 当前视角处于屏幕外且默认 `offscreenIndicatorEnabled: false`）。待与 Track A-1 / Track A-2 合入使 Host 与 Client 保持同步帧后，可连续 3 次执行 `repro:dev-broadcast-v3` 验证稳定 `PASS_FIXED`。
