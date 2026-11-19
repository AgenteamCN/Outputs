# Wave 1 Track B — Bug 2 修复报告（`fix/bug-2-replay-nametag-team-sync`）

## 1. 分支与提交状态
- **基准分支**：`origin/z/agy/rc/multiplayer-recheck`
- **新建分支**：`fix/bug-2-replay-nametag-team-sync`
- **本地提交**：`ba6527e` — `fix(replay-sync): sync 3D nameTags and arena.carTeams on client during Replay Watch Together`
- **推送状态**：`git push -u origin fix/bug-2-replay-nametag-team-sync` 失败（沙箱环境未配置 GitHub 凭据：`fatal: could not read Username for 'https://github.com': terminal prompts disabled`）。本地分支与提交已就绪，需在具备推送权限的环境中执行 `git push -u origin fix/bug-2-replay-nametag-team-sync`。

## 2. 根因分析
1. **3D NameTag 未同步**：`src/game/runtime/devSyncChannelHandlers.ts`（`channel.onGlobalWorkspaceBroadcast`）与 `src/game/runtime/onlineSessionEnvironmentPipeline.ts`（`applyClientRoomActivitySwitch`）在收到 Host 下发的 `replaySync` 时，仅更新了 `useReplayViewerStore` 与 `hudStore`，未调用 `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode, carEntries })`，导致客户端 `VehicleNameTagSystem` 仍使用残留的 Freeplay 机器人名牌状态。
2. **`arena.carTeams` 阵营错位**：
   - `src/game/runtime/onlineSessionEnvironmentPipeline.ts` 在 `activityId === 'replay'` 切换分支中未同步 `rs.roster` 到 `physics.setCarTeam` 与 `arena.setCarTeam`。
   - `src/entities/arena/ArenaVehicleFrameStepper.ts` 在客户端首次惰性创建车辆网格（`!arena.cars[carIndex]`）时，直接使用 `carIndex % 2` 作为 `defaultTeam` 调用 `vr.ensureCar(carIndex, defaultTeam, ...)`，覆盖了此前已通过 `arena.setCarTeam(carIndex, team)` 写入的 `vr.carTeams[carIndex]`（例如 `carIndex = 4` 被 `4 % 2 = 0` 覆盖为蓝队）。

## 3. 修改内容
- [src/game/runtime/devSyncChannelHandlers.ts](file:///workspace/RocketCompanion/src/game/runtime/devSyncChannelHandlers.ts)
  - 引入 `setNameTagRuntimeState`，在收到 `replaySync.roster` 时同步更新 `setNameTagRuntimeState({ isMultiplayerOrMultiCarMode: replaySync.roster.length > 1, carEntries })`。
- [src/game/runtime/onlineSessionEnvironmentPipeline.ts](file:///workspace/RocketCompanion/src/game/runtime/onlineSessionEnvironmentPipeline.ts)
  - 引入 `setNameTagRuntimeState`，在 `applyClientRoomActivitySwitch` 的 `activityId === 'replay'` 分支中，将 `rs.roster` 同步到 `ctx.physics?.setCarTeam`、`ctx.arena?.setCarTeam` 与 `setNameTagRuntimeState`。
- [src/entities/arena/ArenaVehicleFrameStepper.ts](file:///workspace/RocketCompanion/src/entities/arena/ArenaVehicleFrameStepper.ts)
  - 惰性创建车辆网格时优先读取 `vr.carTeams[carIndex]`，仅在未设置时回退到 `carIndex === 6 ? 0 : carIndex === 7 ? 1 : carIndex % 2`。

## 4. 验证结果
- **静态检查（`pnpm run check`）**：通过（`radar` 1425/1425 TS 文件 GREEN，`tsc --noEmit` 0 错误）。
- **Prettier 格式检查**：修改文件已通过 `prettier --write` 格式化。
- **E2E 复现套件（`pnpm run repro:dev-broadcast-v3`）**：
  - `bug-2-replay-nametag-and-team-desync`：**PASS_FIXED (`reproduced === false`)**
    - `host.arenaCarTeams`: `[0, 0, 0, 1, 1, 1]`
    - `client.arenaCarTeams`: `[0, 0, 0, 1, 1, 1]`
    - `teamMismatch`: `false`
    - `tagTextMismatches`: `[]`
    - `client.tags`: `[{"carIndex":1,"displayText":"Nexto #5"},{"carIndex":0,"displayText":"Nexto #4"},{"carIndex":2,"displayText":"Nexto #6"}]`
