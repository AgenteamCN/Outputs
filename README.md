# 状态确认、`.githooks` 权限修复与并发等待机制说明

## 1. 当前注入与控制台状态确认
- **注入与网络录制已完全正常**：`Injection Verified`、`recording=true`，且已成功捕获 Google AI Studio 的 `UpdatePrompt` XHR 请求与响应；输出中也未包含敏感私钥/Token。
- **控制台两处小提示均无害**：
  - `CoreText note: Client requested name ".SFNSMono-Light"...`：macOS WebKit/AppKit 系统字体回退提示，不影响功能。
  - `ax_bridge.rs` `warning: field id is never read`：Rust 结构体反序列化字段未使用警告，不影响运行。
- **下一步**：可以直接点击 **Connect EMQX**（当前状态是 `EMQX: Off`），即可通过 EMQX Bridge 实时联调指令与审批流。

---

## 2. `.githooks/*` “无行差异却提示修改”的根因与修复

### 根因
对比 `b65f0868fdc006b9aad9214fdbd0f417be831c8d`（分支 `what-the-hell-commit`）确认：
```diff
old mode 100644
new mode 100755
```
**文件内容一行未改，只是文件权限位变了**：
- `origin/a2` 树里 `.githooks/post-checkout`、`post-merge`、`post-rewrite` 存的是 `100644`。
- `scripts/git/ensureGitHooks.ts` 每次启动都会执行 `fs.chmodSync(hookPath, 0o755)` 赋予可执行权限，导致 Git 认为这 3 个文件变成了 `100755` 脏状态，但行级 diff 为空。

### 已处理
- **已修复主分支 `a2`**：通过 GitHub REST `git/trees` 将 `.githooks/post-checkout`、`post-merge`、`post-rewrite` 的 mode 正式更新为 `100755`（Verified Bot Commit `b2af7d6df41c63586442033e951de77d415fa73d`）。
- **已删除临时分支**：`origin/what-the-hell-commit` 已删除（HTTP 204）。后续只需继续在 `a2` 分支拉取即可，不会再反复变脏。

---

## 3. 云端 Agent 调用 Worker 等待机制：阻塞 vs 并行

- **单次工具调用内部是同步等待 HTTP 响应**：当 Agent 调用 `execute_bash` 去请求 `POST https://agentplayground.maboco-mark.workers.dev/mqtt/rpc`（例如 `waitMs: 25000`）时，那一次工具调用会像等待一个慢 HTTP 接口一样挂起等待，直到收到你的 EMQX 回复或达到 `waitMs` 超时返回。
- **为什么不会“彻底卡死”且支持并行**：
  1. **同轮并发工具调用（Parallel Tool Calls）**：Agent 可以在同一轮同时发出多个独立工具调用——例如**一边用 `execute_bash` 调 Worker 等待你的 EMQX 消息，一边并行读/改仓库文件或跑 lint**；但需要注意，如果后续步骤强依赖你在 EMQX 里的回复结果（比如等你确认某步操作），就必须等那次 `waitMs` 返回后才能决定下一步。
  2. **分段短轮询（Bounded Polling）**：每次 bash 命令上限 300 秒、Worker `waitMs` 上限 55 秒。实际上每次设 `waitMs = 20~30s` 即可：收到消息就提前返回，超时则由 EMQX 持久会话（`cleanSession: false` + `QoS 1`）继续替我们缓存未读消息，下一轮再取，不会漏消息也不会一直占着 Worker CPU。
