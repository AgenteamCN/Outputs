# 下一轮开局模板与基础设施速查（脱敏版）

## 1. 通用 Initial Input 模板（直接复制开下一轮）
```yaml
# 1. Auth Configuration
auth:
  curl -s -H "x-agy-request-id: <YOUR_AUTH_UUID>" "https://getgithubapptoken.maboco-mark.workers.dev/"

# 2. Commit Preference
commit_preference:
  author_type: "bot"
  verified_signature: true
  Co-authored-by: Mark <29165094+MaBoCoMark@users.noreply.github.com>
  required_instruction: |
    1. Token: Run auth curl before pushing/fetching private repos.
    2. Verified Bot Commit ONLY (never `git push` local commits):
       Use GraphQL `createCommitOnBranch` or REST `git/trees` -> `git/commits` -> `git/refs/heads/<branch>`,
       and `POST /repos/<owner>/<repo>/merges` for Verified Bot Merge.
```
- 主仓库：`https://github.com/Antigravi-ty/agyDevKit`（目标分支 `a2`）
- Worker 仓库：`https://github.com/Antigravi-ty/agyWorker`（目标分支 `main`）
- 环境恢复：`node scripts/env/doctor.mjs --auto-restore` + `pnpm run env:check`（<= 100 MB）
- Live Chat：使用 `curl` 访问 `https://agentlivechatchannel.maboco-mark.workers.dev`（禁 `urllib`），开启 Partner 分包 + 交叉评审 + Secretary `[challenge]` 质疑模式。
- 本轮主要目标：`<在此填写下一轮具体 Workflow 或产品优化目标>`

## 2. 基础设施与踩坑速查（已验证规则）
1. **Live Chat 接口规范**：
   - 发送：`POST /chat/send`，正文用 `"text"`（不要用 `"message"`）。
   - 拉取：`POST /chat/poll`，多频道用 `"channelIds"`（不要用已弃用的 `"channels"`）。
   - 备份：`POST /channel/backup` 必须传 `"payload"`，随后用返回的 `sha256` 调用 `POST /channel/backup/verify`。
   - 私有指令包下载：`GET /channel/backup?channelId=<CHANNEL_ID>&kind=instructions_bundle`（需带 `x-agy-request-id`，内容在 `.checkpoint.payload`）。
2. **Git 与分支权限**：
   - 主分支 `a2` / `main` 要求 Verified Bot Commit/Merge（带 GPG 签名），普通 `git push` 会被拒或无签名。
   - Partner 各自在独立功能分支开发、跑通 2x 门禁（`tsc`、`vitest`、`cargo test --locked`、`check:gates`、`env:check`），由独立 Reviewer 交叉评审通过后，再由 Chief 执行 Verified Bot Merge。
3. **沙盒能力边界（能做 vs 不能做）**：
   - **云端能做**：全量 TypeScript / Vitest / Rust `cargo test --locked` 编译与单测、静态门禁（`check:gates`）、OverlayFS 检查、脚本契约测试、HMOS 离线签名/ArkTS 逻辑校验与包打包。
   - **云端不能做（需回本地实测）**：macOS 真机 `osascript` Touch ID 弹窗与 `utun` 创建、Windows 实机 `bootstrap.ps1` / UAC Admin 提权 / UIA 操作、HMOS 真机 `hdc` 安装与推送验证。
