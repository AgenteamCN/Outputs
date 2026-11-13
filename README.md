# RocketCompanion PR 整理与分支清理报告

## 1. 已创建的 3 个 Verified Bot PR（目标分支：`alpha`）

所有提交均通过 GitHub GraphQL `createCommitOnBranch` / REST `git/trees` + `git/commits` 生成，均带有 **Verified GPG 签名**（`verified: true`），且已通过 `pnpm check`（`radar` 架构护栏 + `tsc --noEmit`）校验：

1. **PR #16 — Notification Collapse Toggle**
   - 分支：`z/agy/rc/pr-notification-collapse-toggle` -> `alpha`
   - 链接：https://github.com/Antigravi-ty/RocketCompanion/pull/16
   - 说明：基于当前 `origin/alpha`（`ebc6e7b`）干净 cherry-pick `d3fb50c85b5d26697875c92e8fbe5601351294ef`，彻底剥离了原父提交 `c61ffe1`（`broked-alpha`）的错误合并历史。

2. **PR #17 — Transparency Glow Post + Driftwoods Hex Grid Opacity 0.03**
   - 分支：`z/agy/rc/pr-transparency-glow-post` -> `alpha`
   - 链接：https://github.com/Antigravi-ty/RocketCompanion/pull/17
   - 说明：基于当前 `origin/alpha`（`ebc6e7b`）干净 cherry-pick `4e9c52717c1027c35966b48a745e0f90ed94dac2`，并将 Driftwood Stadium Tunnel Shield Hex Grid 的默认 Alpha Opacity 从 `0.22` 调整为 `0.03`：
     - `src/ui-core/config/driftwoods.default.json` (`hexes.opacity: 0.03`)
     - `src/ui-core/panels/dev/driftwoods-tuner/DriftwoodsTransparentTab.tsx` (`config.hexes.opacity ?? 0.03`)
   - 已额外通过 `pnpm build` 生产构建验证。

3. **PR #18 — Dynamic Training Takeover (`3e8ce00` 方案)**
   - 分支：`z/agy/rc/pr-dynamic-training-takeover` -> `alpha`
   - 链接：https://github.com/Antigravi-ty/RocketCompanion/pull/18
   - 说明：基于 `3e8ce00a6cf2e51970e165c0998e34888e3a54e9` 合并 `origin/alpha`（`ebc6e7b`），解决了 `src/ui-core/stores/devFloatingPresets.ts` 中 `DYNAMIC_TRAINING_PACK_PRESET` 与 `TELESTRATOR_TOOLBAR_PRESET` 的合并冲突。

---

## 2. `3e8ce00` 与 `96dcecb` 的对比结论

**`3e8ce00` 明显更优**，原因如下：
- **稀疏 Roster 与 Slot 错乱修复**：`3e8ce00` 在 `hydrateRLRCFramesFromDynamicSession` 中使用 `createInactiveHydratedCar` 补全空缺 slot 并按 `slotIndex` 排序，同时在 `RLRCInterveneFrameStateBuilder.ts` 中将查找改为 `findCarBySlot`（`cars[slot]` O(1) 快速路径 + `.find()` 回退）。
- **Takeover HUD 接管倒计时修复**：`3e8ce00` 在 `GoalReplayHudOverlay.tsx` 中新增 `useDynamicTrainingPrerollHud()`，使动态训练在 `preroll-countdown` 阶段能够正确显示倒计时与队伍色彩 HUD，而 `96dcecb` 未包含此修复。
- **玩家输入与 POV 解析修复**：`3e8ce00` 新增 `resolveEffectivePovSlot` 并在 `enterInterveneAtFrame` 之前设置 `r.playerCarIndex = resolvedPovSlot`，同时 `readPlayerControls` 兼容 `r?.playerControls ?? r?.input?.getPlayerControls?.()`，且传入 `onPhaseChanged: () => {}` 避免污染 Replay Importer 的状态。
- *(注：`96dcecb` 唯一额外做的逻辑是在 `dynamicTrainingPackStore.ts` 中切换某个 slot 为 `player` 时自动把其他 slot 的 `player` 重置为 `as-previous`。这属于小细节，不影响 `3e8ce00` 的核心正确性，已可直接合并 PR #18。)*

---

## 3. 遗留分支端头清理建议

在 PR #16、#17、#18 合并进 `alpha` 后，**建议保留的未合并端头**：
1. **Rust 端头**：`z/agy/rc/rust/audio-flip-reset-fix`（`52b8173`）—— 包含 Wave 1-4 + 音频/物理修复链路的最新节点。
2. **P2P Voice 端头**：`z/agy/rc/track-b-voice-fix`（`3e7cf81`）—— 包含 Track B WebRTC Voice Configurator 及 P2P 信令竞争/静默修复。
3. **可选确认**：
   - `a2`（`de978ea`，envinit-worker 相关，如仍在使用请保留）。
   - `z/agy/rc/track-a-telestrator-defaults`（`1db147f`）：当前 `origin/alpha` 中的笔刷默认值仍为 `width=10, arrowLength=36, arrowWidth=28, presets 100%/70%/50%`，如果你仍想要 `1db147f` 的 `5 / 60 / 30 / 100%/50%/10%`，后续可单独基于 `alpha` 再提一个小 PR；否则可连同 `broked-alpha` 一起删除。

**其余端头均可安全删除**（`broked-alpha`、旧的 `notification-collapse-toggle`、`fix-transparency-glow-post`、`fix-replay-slot-takeover`、`fix-replay-slot-and-timeout`、`track-c-*`、`track-a-telestrator-*`、`telestrator-overlay-*`、已合并的 `brush-diag`/`fix-bootstrap-hangs`/`track-d-replication-bus`、以及 `rust/wave-*` 等中间过程分支）。

---

## 4. 为什么 Agent 总会莫名其妙一口气提交两个 Commit？

常见原因有三个：
1. **先 Merge 再 Commit（如 `848a068` + `1db147f`）**：Agent 在旧分支上工作时先执行了一次 `chore: merge alpha into <branch>` 同步上游代码（第 1 个 Commit），随后再提交自己的功能改动（第 2 个 Commit）。
2. **先提功能 Commit 再提修复 Commit**：Agent 在第一次提交后跑 `pnpm check` / `tsc` 发现漏改或 lint 问题，习惯性直接再 `createCommitOnBranch` 补一个 `fix` Commit，而不是在本地合并成单次提交后再通过 API 一次性上树。
3. **基于已含历史提交的旧分支继续操作**：如果分支本身已经存在未合并的父提交，Agent 在其基础上再加一次提交，对外表现就是至少两个 Commit。
