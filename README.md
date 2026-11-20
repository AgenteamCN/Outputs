# RocketCompanion 团队环境与 tmpfs 自愈使用指南（Track D 交付说明）

> **远程分支**：`origin/feat/env-tmpfs-restore-guard`（Verified Bot Commit `9b0401dd7d6c5a03d57ee6509b64ddf0b047c856`）

---

## 一、回答核心问题：是不是“任何指令失败都会自动检测并全恢复”？

**不是无条件全自动拦截所有 shell 命令，而是分层级设计（避免每次命令都产生额外耗时或隐藏真实业务报错）：**

### 1. 哪些是“静默自动检测 + 自动自愈”的？
1. **调用 `pnpm ...`（通过 `/usr/local/bin/pnpm` 包装器）**：
   - 每次执行 `pnpm` 时，若检测到 `/tmp` 被容器快照重置（`/tmp/rc-cache/pnpm-global/bin/pnpm` 丢失），包装器会**自动通过 `npm` 在 ~3 秒内将 `pnpm@10.28.2` 重装到 tmpfs**，并自动调用 `node scripts/env/tmpfsGuard.mjs --ensure` 重建 `/tmp/rc-cache/*` 目录与基础软链接（`node_modules`、`dist`、`.vite`、`src-tauri/target`）。
   - **注意**：`pnpm` 包装器为了不拖慢常规命令，**不会**在每次运行 `pnpm` 时自动重新克隆 `public/` 或重新跑 `pnpm install` / 重新生成 `.rcreplay`。

### 2. 哪些是“自动检测 + 明确打印 `try ... first` 恢复指令”的？
1. **`pnpm run env:check`（`scripts/env/checkEnv.mjs`）**
2. **`pnpm run env:doctor`（`scripts/env/doctor.mjs`）**
   - 当检测到以下任意 `/tmp` 重置信号时：
     - 存在悬空软链接（dangling symlinks，如 `public` 或 `node_modules` 指向已清空的 `/tmp/rc-cache/...`）
     - `.git/rc-overlay-baseline.json` 存在但 `/tmp/.rc-env-sentinel` 丢失
     - sentinel 中记录为存在的 `viteCacheDir`、`CARGO_TARGET_DIR` 或 `.rcreplay` fixture 丢失
   - 均会打印醒目的 `[tmpfsGuard] /tmp (tmpfs) RESET DETECTED!` 诊断横幅，并直接给出可复制执行的恢复命令：
     ```bash
     node scripts/env/doctor.mjs --auto-restore
     pnpm run env:setup
     pnpm run env:setup:rust
     pnpm run replay:generate-ducky
     ```
3. **E2E 脚本前置检查（`scripts/env/reproPreflight.mjs` / `launchHeadlessChrome.mjs`）**：
   - 当 `public/` 或 `node_modules` 或 Chrome 缺失时，会抛出带提示的错误，引导执行 `pnpm run env:setup` 或 `pnpm run env:doctor`（不会静默超时卡死在 60s boot timeout）。

### 3. 一键全量恢复指令（`--auto-restore`）
当你看到上述任何提示、或怀疑容器快照恢复清空了 `/tmp`、或刚执行过 `git reset --hard`（注意：`git reset --hard` 会把 `public` 软链接还原为空目录）时，只需执行以下**任意一条**即可全量恢复（`pnpm` + `public/` + `node_modules` + `.vite` + `6-Ducky .rcreplay` + `/tmp/.rc-env-sentinel`）：
```bash
node scripts/env/doctor.mjs --auto-restore
# 或（当 pnpm 还在时）：
pnpm run env:setup
# 若容器内连 pnpm 都还没有（首次初始化）：
bash scripts/env/bootstrap.sh
```
（若需 Rust / Tauri C4 验证，追加 `--with-rust` 或运行 `pnpm run env:setup:rust`。）

---

## 二、后续给仓库各团队 / Agent 的标准 Input 模板

建议在后续所有 Track（A / B / C / E 及集成验证）的 Prompt / Input 中统一附带以下环境约定段落：

```markdown
### 环境与 tmpfs 自愈约定（RocketCompanion 容器规范）
1. **首次初始化或 `/tmp` 快照重置后**：
   - 若 `pnpm` 不存在：先运行 `bash scripts/env/bootstrap.sh`（仅装 pnpm 可用 `bash scripts/env/bootstrap.sh --pnpm-only`）。
   - 若遇到环境异常、悬空软链接、`public/` 缺失、或 E2E 报 preflight 失败：先运行
     `node scripts/env/doctor.mjs --auto-restore`（需 Rust/Tauri 时加 `--require-rust`）。
2. **禁止在 OverlayFS 长期落盘大体积产物**：
   - `node_modules`、`public/`、`.vite`、`dist`、`src-tauri/target`、`.rcreplay` 均已通过 `scripts/env/tmpfsGuard.mjs` 分流到 `/tmp/rc-cache/`。
   - 保持仓库 OverlayFS 占用 `<= 100 MB`（可通过 `pnpm run env:check` 查看基线与当前占用）。
3. **执行 `git reset --hard` / `git checkout -f` 后**：
   - 若 `public/` 软链接被还原为空目录，执行 `node scripts/env/tmpfsGuard.mjs --sync-public` 或 `node scripts/env/doctor.mjs --auto-restore` 即可恢复且保持 `git status` 干净。
```
