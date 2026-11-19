# RocketCompanion `repro:dev-broadcast-v2` WebGL2 E2E 修复与一键环境引导报告

## 1. 问题根因定位（为什么之前跑不起来）

在当前无 GPU 容器环境中实测排查后，确认 **`google-chrome` + ANGLE/SwiftShader WebGL2 本身完全可用**，此前 `pnpm run repro:dev-broadcast-v2` 报 `Timeout waiting for Host boot after 60000ms` 及 `ENOTEMPTY` 并非缺乏 GPU/WebGL，而是以下 3 个具体工程问题导致：

1. **容器代理劫持回环请求（核心根因）**：
   - 容器环境变量设置了 `HTTP_PROXY` / `HTTPS_PROXY`（如 `http://10.100.37.1:8081`）。
   - E2E 脚本启动 Headless Chrome 时未传 `--no-proxy-server`，导致 Chrome 访问 `http://127.0.0.1:<port>` 时走代理失败并停留在 `chrome-error://chromewebdata/`，`window.__rcE2E` 永远不会挂载，最终 60s 超时。
2. **`public/` Git Submodule 未初始化**：
   - `public/` 是指向 `RocketCompanionStatics.git` 的子模块，未执行 `git submodule update --init --recursive` 时缺少 `camera.wasm` 等静态资源，触发 `Verify Camera Microkernel Integrity` 致命校验失败（SHA-256 mismatch）。
3. **`.rcreplay` 测试夹具缺失与清理竞态**：
   - `runDevBroadcastReproV2.mjs` / `V3.mjs` 依赖 `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay`，缺失时直接报错退出。
   - `finally` 中 `chromeProc.kill('SIGKILL')` 后立即同步 `fs.rmSync(userDataDir)` 触发 `ENOTEMPTY`。

---

## 2. 新建分支与改动内容

基于 `origin/z/agy/rc/multiplayer-recheck` (`2565774`) 新建分支 **`feat/e2e-env-bootstrap`**（commit `ff602c5`）：

### 新增专用环境工具目录 `scripts/env/`
- `scripts/env/launchHeadlessChrome.mjs`：
  - 统一封装 Headless Chrome 启动参数：**强制注入 `--no-proxy-server`** + SwiftShader/ANGLE WebGL2 参数。
  - 启动前执行环境预检（`assertE2EEnvironmentReady`）：检测 Chrome 二进制、`node_modules`、`public/` submodule 是否就绪；若缺失直接报错并提示运行内置指令 `pnpm run env:setup`（或 `pnpm run env:check`）。
  - 提供 `cleanupHeadlessChrome` 安全清理临时目录（带重试与容错，消除 `ENOTEMPTY`）。
- `scripts/env/generateSyntheticReplayFixture.mjs`：
  - 当 `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` 不存在时，自动生成合法的 3v3 `.rcreplay`（gzipped JSON，含 6 车 34 垫与 360 帧物理状态及 3v3 名单）。
- `scripts/env/checkEnv.mjs`（`pnpm run env:check`）：
  - 一键体检 Node.js、pnpm、`node_modules`、`public/` submodule、Chrome 二进制、**真实启动 Headless Chrome 探测 WebGL2/SwiftShader 上下文**、代理绕过状态、`.rcreplay` 夹具及可选 Rust/Cargo 工具链。
- `scripts/env/bootstrap.sh`（`pnpm run env:setup`）：
  - 一键幂等初始化脚本：自动补齐 `pnpm`、`git submodule update --init --recursive`、`pnpm install`、apt Chromium（若缺 Chrome）、生成 3v3 `.rcreplay` 夹具，并支持 `--with-rust`（参考孤立提交 `de978ea22fea805ddfd678318c080b215b35dac8` 的 Rust 工具链引导方式）安装 Rust/Cargo。

### 更新 E2E 与脚本配置
- `package.json`：新增 `"env:check"` 与 `"env:setup"` 内置指令。
- `scripts/runDevBroadcastReproV2.mjs` & `scripts/runDevBroadcastReproV3.mjs`：接入 `launchHeadlessChrome`、`cleanupHeadlessChrome` 与 `.rcreplay` 自动回退生成。
- `scripts/runDevRoomSyncE2E.mjs`、`scripts/runDevSyncReproSuite.mjs`、`scripts/runDevSyncExtraRoutesE2E.mjs`、`scripts/runScenarioBaseline.mjs`、`scripts/runSoakWatchdog.mjs`、`scripts/runMultiplayerRoomRepro.mjs`、`scripts/generateTauriCrossbarIcons.mjs`：统一补充 `--no-proxy-server`。

---

## 3. 实测验证结果（分支 `feat/e2e-env-bootstrap` @ `ff602c5`）

- `pnpm run check`（`radar` + `tsc --noEmit`）：**PASS (GREEN)**
  - `1428/1428` TS 文件，`NameTagScatter=1/1`，`BoolHotspots=7`，`UnscopedCaches=0`，`tsc --noEmit` clean。
- `pnpm run test:multiplayer-suite`：**PASS (24/24)**。
- `pnpm run env:setup` / `pnpm run env:check`：**PASS**（`WebGL2 ready (WebKit WebGL)`）。
- `pnpm run repro:dev-broadcast-v2`：**PASS (`reproduced=0/4`)**
  - `bug-1-freeplay-goal-replay`: `[PASS_FIXED]`
  - `bug-2-replay-ui-and-roster`: `[PASS_FIXED]`
  - `bug-3-camera-player-switch-sync`: `[PASS_FIXED]`
  - `bug-4-exhibition-countdown-321`: `[PASS_FIXED]`
