# RocketCompanion PR #10 合并与工作流说明

## 1. PR #10 已创建并合并进 `alpha`
- **PR**：[#10 feat(tauri): auto-restart tauri dev on Git HEAD switch and ignore .DS_Store](https://github.com/Antigravi-ty/RocketCompanion/pull/10)
- **来源分支**：`z/agy/rc/tauri-dev-watch` (`c3fb43c`) -> 目标分支：`alpha`
- **合并 Commit**：`83c6c4ee86d3356aedf248f0cce29c07f4522c80`

## 2. 关于 PR 自动审批与验证工作流说明
1. **为什么之前没“审批”也能直接合并？**
   - GitHub API 禁止 PR 创建者审批自己的 PR（会返回 `422`），但因为当前 `alpha` 分支没有开启「必须有外部 Review Approval 才能合并」的分支保护规则，所以 Bot 创建 PR 后可以直接调用 Merge API 合入 `alpha`。
2. **为什么直接合入 `alpha` 是当前最合理的做法？**
   - 后续所有并行分支都从 `alpha` 分叉，合入 `alpha` 后，所有新分支天然自带 `.taurignore` 和 `tauriDevWatch.ts`。
   - **本地验证技巧**：只要你在最新的 `alpha` 上启动一次 `pnpm tauri:dev`（或 `pnpm tauri:dev:tray`），`tauriDevWatch.ts` 就已经常驻在 Node 内存中了——哪怕你随后 `git checkout` 到**还没包含这个改动的旧分支/旧 commit**，它依然能检测 HEAD 变化并帮你杀掉重起 `tauri dev`，切回 `alpha` 时也会再次自动重启；如果遇到任何异常，随时可以用 `pnpm tauri:dev:raw` 退回原始 `tauri dev`。
