# 合并验证、Bug 5 验收与 tmpfs 自愈底座优化报告

## 1. 两个 Commit 能否合并？

**结论：可以无冲突直接合并（Zero Conflicts）。**

经核查，两个 commit 均直接基于 `origin/z/agy/rc/multiplayer-recheck` (`7252e42`)，且修改文件完全正交：
- `9b0401dd7d6c5a03d57ee6509b64ddf0b047c856`（`feat(env): add tmpfs offloading, OverlayFS budget guard, and /tmp reset auto-restore`）：仅修改 `scripts/env/*`。
- `559fbafc705534bdc7c43dd55753867f3498444f`（`fix(replay): dispose per-car FlipResetVisuals and replay-scoped resources on clearReplay`）：仅修改 `src/game/runtime/RuntimeResourceRegistry.ts`、`src/replay/ReplayViewerCausalSyncController.ts`、`src/replay/ReplayViewerRosterSubController.ts`。

## 2. Bug 5（Track C）是否真正解决？

**结论：已彻底解决，E2E 验证为 `PASS_FIXED`。**

在合并后执行 `bash scripts/env/bootstrap.sh` 完成环境初始化并运行全部验证套件：
- `pnpm run check`（Radar + `tsc --noEmit`）：**GREEN**（1430/1430 TS，0 循环依赖，0 hard errors）
- `pnpm run test:multiplayer-suite`：**26/26 PASS**
- `pnpm run repro:dev-broadcast-v3`（Headless Chrome + WebGL2 + WASM）：**`reproduced=3/5`**
  - **Bug 5 (`bug-5-high-memory-and-unreleased-on-exit`)：`PASS_FIXED`**
    - `bootMem`: `geometries: 40`, `wasmMB: 64`
    - `loadedMem`: `geometries: 96`, `wasmMB: 64`
    - `afterExitMem`: `geometries: 40`, `wasmMB: 64`
    - **`retainedGeometries: 0`**, **`retainedWasmMB: 0`**
  - **Bug 4 (`bug-4-boostpad-model-flicker`)：`PASS_FIXED`**
  - **Bug 1 / Bug 2 / Bug 3**：仍为 `REPRODUCED_BUG`（注：Bug 2 本次出现偶发 Client 仅渲染部分 NameTag 的 `tagTextMismatches`，与计划中 Track E 提到的时序抖动一致，需在 Track A/B/E 中继续跟进）。

## 3. tmpfs 超低占用 & 自愈底座评价与优化

### 实测表现
- 完整初始化后，仓库 OverlayFS 占用仅为 **29 MB**（预算上限 `<= 100 MB`），`node_modules`、`public/`、`.vite`、`dist`、`.rcreplay` 全部软链接分流至 `/tmp/rc-cache/`，`git status` 保持完全干净。

### 我们对底座做的增强（已提交至新基线分支）
原有的 `scripts/env/README.md` 仍写着不存在的 `setup.sh`，且缺少统一的 `help` 入口；同时在 `/tmp` 重置后若只跑 `pnpm run check` 等业务命令，缺少“一键恢复”提示。本次已在合并基线上补充了以下改进（commit `35dbc97`）：
1. **新增 `pnpm run env:help`（`scripts/env/help.mjs`）**：用一屏精简输出列出所有 `env:*` 命令、静默自愈 vs `--auto-restore` 边界、以及 `git reset --hard` 后的 `--sync-public` 恢复命令。
2. **为 `doctor.mjs` 与 `bootstrap.sh` 增加 `--help` / `-h` 参数**：随时可查看用法。
3. **增强 `pnpm` 包装器（`scripts/env/tmpfsGuard.mjs`）**：在非 `env:*`/`install` 命令下，若检测到 `/tmp` 重置导致 `node_modules/.modules.yaml` 或 `public/` 缺失，立即在 stderr 打印 1 行提示引导执行 `node scripts/env/doctor.mjs --auto-restore`。
4. **更新 `scripts/env/README.md`**：修正 `setup.sh` 为 `bootstrap.sh`，补齐 `env:help`、`env:check`、`--auto-restore` 文档。

### 已推送的 Verified Bot 基线分支
已通过 GitHub API（GPG `verified: true`）创建并推送集成基线分支：
- **分支**：`origin/z/agy/rc/multiplayer-recheck-base`（`35dbc97`）
- **包含内容**：`origin/z/agy/rc/multiplayer-recheck` (`7252e42`) + `9b0401d` (tmpfs/自愈) + `559fbaf` (Bug 5 修复，merge commit `2cd5301`) + `35dbc97` (`env:help` 与包装器提示增强)。

## 4. 后续团队（Track A / B / E）的精简 User Input 模板

不需要把上一份长篇报告全贴给后续团队。**因为底座已经自带 `pnpm run env:help` 和自愈能力**，后续给各 Track 的 Prompt 只需附带以下 **4 行精简约定** 即可：

```markdown
### 基线与环境约定
- 本次任务基于 `origin/z/agy/rc/multiplayer-recheck-base`（已含 Bug 5 修复与 tmpfs 低占用底座，OverlayFS 预算 <= 100 MB）。
- 若环境未初始化或 `/tmp` 快照重置：无 `pnpm` 时跑 `bash scripts/env/bootstrap.sh`，有异常时跑 `node scripts/env/doctor.mjs --auto-restore`（含 Rust 需求加 `--require-rust`）。
- 执行 `git reset --hard` / `git checkout -f` 后若 `public/` 异常，跑 `node scripts/env/tmpfsGuard.mjs --sync-public`。
- 随时可运行 `pnpm run env:help` 查看环境命令，或 `pnpm run env:check` 检查状态。
```
