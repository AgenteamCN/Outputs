# agyPreviewInstant 初始化脚本排查与修复报告

## 1. 问题根因定位

通过对容器环境与 `scripts/env/` 脚本逐段排查，确认了导致此前初始化超时与状态误判的两个核心问题：

1. **`NODE_OPTIONS` 中的 `global-agent/bootstrap` 导致 `pnpm install` 完成后永久挂起（致命）**
   - 沙箱默认注入了环境变量：`NODE_OPTIONS=--max-old-space-size=1024 --require global-agent/bootstrap`。
   - `pnpm` 在下载 274 个依赖包时，通过 `HTTP_PROXY` / `HTTPS_PROXY` (`http://10.100.40.1:8081`) 建立连接；`global-agent/bootstrap` 会劫持 Node.js 全局 `http`/`https` Agent 并保持大量 `ESTABLISHED` 状态的 keep-alive socket。
   - 结果：`pnpm install` 实际在 **~5 秒**就已完成全部下载与链接并打印 `Done in 5.2s`，但 Node 事件循环因未关闭的 keep-alive 句柄无法退出，一直挂起直到触发命令超时（`upstream request timeout`），并导致 `/tmp` 快照重置。
   - `pnpm` 原生支持 `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`，移除 `global-agent/bootstrap` 后 `pnpm install` 只需 **6.1 秒**即可干净退出（`EXIT=0`）。

2. **`apps/p2p-remote-controller/node_modules` 误报 `[OK]`**
   - `ensureTmpfsLayout` 会提前创建 `/tmp/agy-cache/p2p-node_modules` 并挂载 `apps/p2p-remote-controller/node_modules/.vite` 软链。
   - 旧版 `checkEnv.mjs` / `doctor.mjs` / `tmpfsGuard.mjs` / `bootstrap.sh` 仅检查目录是否存在（或未排除 `.vite`/`.cache`），导致 `/tmp` 重置后即使 `p2p` 依赖为空目录也误报 `[OK] installed (tmpfs)`，进而跳过安装。

3. **非 TTY 与更新检查噪音**
   - `pnpm` 默认的 `update-notifier` 会发起额外网络请求并输出升级提示；非 TTY 环境下默认进度条刷新不适合日志观测。

---

## 2. 已完成的修复（Commit: `f399138` on `a2`）

- **`scripts/env/tmpfsLayout.mjs`**：新增 `sanitizeNodeOptions()`、`getPnpmEnv()`（剥离 `global-agent/bootstrap` 并禁用 `update-notifier`）以及 `hasPopulatedDir()`（检查目录非空且排除 `.vite`/`.cache`）。
- **`scripts/env/tmpfsPnpm.mjs`**：在生成的 `/usr/local/bin/pnpm` 自愈包装脚本及 `npm install -g pnpm` 中剥离 `NODE_OPTIONS` 里的 `global-agent/bootstrap`，并默认设置 `NO_UPDATE_NOTIFIER=1`、`pnpm_config_update_notifier=false`。
- **`scripts/env/bootstrap.sh`**：脚本顶部清理 `NODE_OPTIONS` 中的 `global-agent/bootstrap`；同步检查 `/tmp/agy-cache/pnpm-global/bin/pnpm` 与 `p2p` 依赖目录是否真实有包；非 TTY 自动附加 `--reporter=append-only`。
- **`scripts/env/tmpfsGuard.mjs` / `checkEnv.mjs` / `doctor.mjs`**：统一使用 `getPnpmEnv()` 与 `hasPopulatedDir()`，彻底消除假阳性 `[OK]` 与挂起问题。

---

## 3. 验证结果与 OverlayFS 预算

- **全新冷启动**（清空 `/tmp/agy-cache`、`/tmp/.agy-env-sentinel`、`/usr/local/bin/pnpm*`）：
  - `bash scripts/env/bootstrap.sh` 全程耗时 **10.4 秒**，`EXIT=0`。
- **快照重置自愈**（清空 `/tmp/agy-cache` 与 `/tmp/.agy-env-sentinel`）：
  - `checkEnv.mjs` 准确识别 `root` 与 `p2p` 均为 `missing`。
  - `node scripts/env/doctor.mjs --auto-restore` 耗时 **6.7 秒**完成全量恢复。
- **工程检查与构建**：
  - `pnpm run check`（`tsc --noEmit && eslint src && vitest run`）：全部通过（16/16 tests passed）。
  - `pnpm run p2p:check`：通过。
  - `pnpm run build && pnpm run p2p:build`：通过。
- **OverlayFS 占用**：全程稳定在 **8 MB**（预算上限 `<= 100 MB`），所有 `node_modules`、`.vite`、`dist`、`.cache` 均已卸载至 `/tmp/agy-cache`（约 212 MB）。

---

## 4. 建议给到下一轮的新 User Input 模板

后续每次开启新任务时，可直接精简为如下环境指引（无需模型自行摸索 `pnpm` 挂起或缓存路径）：

```text
仓库：https://github.com/Antigravi-ty/agyPreviewInstant（分支 a2）
环境初始化 / 快照恢复（已内置 tmpfs 卸载与 global-agent 隔离，约 6~10 秒完成）：
- 若尚未安装 pnpm 或首次进入：bash scripts/env/bootstrap.sh
- 若 /tmp 快照重置（pnpm 仍在）：node scripts/env/doctor.mjs --auto-restore
- 含 Rust/Tauri 需求时加：--with-rust / --require-rust
- 日常状态与 OverlayFS 预算检查（<= 100 MB）：pnpm run env:check
```
