# 主分支误合修复与 `z/agy/rc/new-alpha` 重建报告

不用慌，**代码和干净的替换分支我已经直接帮你搞定了**，现在只需要你在 GitHub 仓库后台做一次“切换默认分支 -> 删除旧 `alpha` -> 重命名为 `alpha`”的管理员操作即可。

---

## 一、已完成的处理

1. **创建干净分支 `z/agy/rc/new-alpha`**
   - 远程分支：`origin/z/agy/rc/new-alpha`
   - 最新提交：`ebc6e7b55b25c9c8aef43709a4a35a53040a7173`（已通过 GitHub API 创建，**GPG Verified (`verified: true`)**，Author 为 `antigravity-fine-grained-token[bot]`，带 `Co-authored-by: Mark`）
   - 起点基线：`db8480300db46e6ffd54cacdd0d82583366f12cf`（即 PR #14 `track-d-replication-bus` 刚合入时的干净 `alpha`，完全在误合 PR #15 之前）
   - 合入内容：`da722e3867d866205f0d51a6491d24dfdd66174c`（`origin/z/agy/rc/brush-diag`，包含 `5b1ee5c`、`2f38585`、`da722e3` 三个提交，零冲突合入）

2. **污染提交剔除校验**
   - `c61ffe1`（误合 PR #15）：**不在** `z/agy/rc/new-alpha` 历史中
   - `1db147f4debff915603210c73d141e8e2827b46d`：**不在** `z/agy/rc/new-alpha` 历史中
   - `848a06849a457315574ebee4bd04c6cec98bea20`：**不在** `z/agy/rc/new-alpha` 历史中
   - `db8480300db46e6ffd54cacdd0d82583366f12cf`（PR #14）：**在** `z/agy/rc/new-alpha` 中
   - `da722e3867d866205f0d51a6491d24dfdd66174c`（`brush-diag`）：**在** `z/agy/rc/new-alpha` 中

3. **构建与检查验证**
   - `pnpm check`：通过
   - `pnpm radar`：通过
   - `pnpm build`：通过

---

## 二、后续两个分叉分支的处理建议（非常重要）

当前从被污染的 `c61ffe1` 上新拉出了两个分支：
- `origin/z/agy/rc/notification-collapse-toggle`（`d3fb50c`）
- `origin/z/agy/rc/fix-transparency-glow-post`（`4e9c527`）

经核查，这两个分支各自只有 **1 个独立功能/修复提交**，且**完全不涉及 `telestrator` 相关文件**。

> **注意：千万不要直接把这两个分支 merge 到新的 `alpha` 上！**
> 直接 merge 会把它们祖先里的 `c61ffe1` / `1db147f` / `848a068` 重新带回主分支。

正确做法：等新的 `alpha` 替换完成后，从新 `alpha` 拉分支并只把这两个提交 **`cherry-pick`** 过来（或把它们 rebase 到新 `alpha` 上）：
- `d3fb50c` (`feat(notifications): add force-collapse toggle with yellow bell and auto-expand on menu exit`)
- `4e9c527` (`fix(render): prevent semi-transparent perimeter overlays from occluding hex glow and bloom`)

---

## 三、需要你这边在 GitHub 网页端做的操作（3 步）

由于 `alpha` 是默认主分支，切换默认分支与删除/重命名默认分支需要你在 GitHub 仓库管理后台操作：

1. 打开仓库 **Settings -> General -> Default branch**，点击切换图标，把默认分支切换为 **`z/agy/rc/new-alpha`**。
2. 打开仓库 **Branches** 页面，删除旧的 **`alpha`** 分支（如想保留旧现场也可先重命名为 `archive/alpha-bad-pr15`，不删直接改名也行）。
3. 在 **Branches** 页面找到 **`z/agy/rc/new-alpha`**，点击 **Rename**，将其重命名为 **`alpha`**（GitHub 会自动保持它为默认分支）。

完成后，新的 `alpha` 就是完全干净且自带可用 `brush-diag` 功能的主分支了。
