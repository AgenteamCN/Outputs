# 执行路线研判：沿用 Wave 0（A/B/C/D）主干，按当前分支基线做 4 项瘦身与衔接更新

## 一、直接结论

**整体架构方向（Wave 0 A/B/C/D 前置 -> Wave 1–3）完全正确，可以直接开工；但计划需要结合刚刚合入 `origin/z/agy/rc/multiplayer-recheck`（`746126d`）的 3 个最新进展做精确微调，避免重复劳动或覆盖已有修复。**

---

## 二、新旧计划对比：4 项关键更新

| 阶段 | 原计划 | 更新后计划（基于 `origin/z/agy/rc/multiplayer-recheck` @ `746126d`） | 原因 |
| :--- | :--- | :--- | :--- |
| **前置基线** | 未合入 Bug 1 / Bug 2 / 债务雷达 | **已完成**：`476b783`（Bug 1）、`5fbc87d`（Bug 2）、`746126d`（`radarArchDebtGuards.ts`）均已合入 | 避免 Wave 0 改写核心文件后与两支 fix 分支冲突 |
| **Wave 0 轨道与门禁** | Arch-0A / 0B / 0C / 0D 4 路并行 | **保留 4 路并行**，但增加两点衔接：<br>1. **Arch-0A** 必须把 `476b783` 已修复的 Goal Replay 流式帧游标与 `appendStreamedReplayFrame` 逻辑无损纳入 `'host-goal-replay'` / `'client-goal-replay'` 策略。<br>2. **Arch-0B** 直接基于 `5fbc87d` 抽出的 `syncReplayRosterTeamsAndNameTags` 收敛为 `MatchRosterProjection.ts`，并在完成后将 `radarArchDebtGuards.ts` 的 `HARD_ENGINE_NAMETAG_SCATTER_LIMIT` 从 `15` 下调（最终降至 `1`）。 | 保留 Bug 1/2 已验证行为，同时用雷达棘轮锁死债务只减不增 |
| **Wave 1** | Track A-1（Bug 1）+ Track B（Bug 2）+ Track C（Bug 5 C1–C4） | **大幅瘦身**：<br>- Track A-1（Bug 1）与 Track B（Bug 2）转为 **Arch-0A/0B 合入后的回归验收**（`repro:dev-broadcast-v3`）。<br>- **主力只剩 Track C（Bug 5）**：C1（WASM 延迟分配 + `reclaimWasmMemoryIfGrown`）、C2（`ArenaWorld.removeCar` + 几何体回收）、C3（AI/Cache 接入 `RuntimeResourceRegistry`）、C4（Tauri `src-tauri/src/lib.rs` 独立 Rust 轨道）。 | Bug 1 / Bug 2 代码已合入，无需重复实现 |
| **Wave 2 & Wave 3** | Bug 3（物理前瞻）-> Bug 4（BoostPad 闪烁） | **保持不变**：<br>- Wave 2：基于 Arch-0A `'client-csr-spectate'` 开启 `shouldRunLookahead` + `SpectateInterpolationBuffer` >=5 帧前瞻窗口。<br>- Wave 3：基于 Arch-0D 离散字段隔离与 `isDuplicateFrame` 收尾 `BoostPadSystem.ts`，最终跑通 `repro:dev-broadcast-v3`（0/5）。 | 与原计划完全一致 |

---

## 三、推荐开工顺序（立即执行清单）

1. **Wave 0（4 路并行，直接开干）**：
   - **Arch-0A**：`simulationFrameClassifierPipeline.ts` 定义 `SimulationDriveMode` + `SimulationDrivePolicy`，收口 `SimulationStage.ts` 与 `simulationNetworkReconcilePipeline.ts`（同步消除 `radar:audit` 中的 `[B]` 布尔热点）。
   - **Arch-0B**：新建 `src/game/runtime/MatchRosterProjection.ts`，先接管 `devSyncChannelHandlers.ts`（`5fbc87d`）与 `onlineSessionEnvironmentPipeline.ts`，再逐步迁移剩余 `[S]` 调用点并下调 `HARD_ENGINE_NAMETAG_SCATTER_LIMIT`。
   - **Arch-0C**：新建 `src/game/runtime/RuntimeResourceRegistry.ts`，把 `RocketSimArenaBarrierController.ts` 的 `initReplayBuffer` 改为懒加载，并为 `[C]` 类无 `dispose` 单例（`SafeTensorsLoader` 等）提供注册接口。
   - **Arch-0D**：在 `ReplicationStateInterpolator.ts` / `ReplayFrameInterpolator.ts` 隔离 BoostPad 离散字段与连续物理量，输出 `frameCursor` / `isDuplicateFrame`。
   - **Wave 0 门禁**：`pnpm run check && pnpm run radar:audit && pnpm run test:multiplayer-suite && pnpm run repro:dev-broadcast-v2`。
2. **Wave 1**：验证 Bug 1 & Bug 2 在新底座上保持通过 + 并行完成 Bug 5（C1/C2/C3 + 独立 Rust C4）。
3. **Wave 2 -> Wave 3**：依次推进 Bug 3（前瞻）与 Bug 4（BoostPad 防抖），以 `pnpm run check && pnpm run repro:dev-broadcast-v3`（`reproducedCount === 0/5`）收官。
