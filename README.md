# RocketCompanion 环境诊断与 Headless WebGL E2E 跑通报告

- **基准分支与提交**：`z/agy/rc/multiplayer-recheck` (`2565774`)
- **新分支与提交**：`z/agy/rc/env-doctor-repro` (`aa82f27`)
- **参考隔离提交**：`de978ea22fea805ddfd678318c080b215b35dac8`（`envinit-worker`，参考了其 `submoduleSync.sh` 的 `public` 子模块初始化思路）

---

## 一、根因定位：为什么此前误判 `repro:dev-broadcast-v2` 跑不起来？

经实测排查，**容器内并非缺少 GPU/WebGL 渲染能力**。Google Chrome 配合 SwiftShader 参数（`--use-gl=angle --use-angle=swiftshader --enable-unsafe-swiftshader`）可直接创建正常的 `WebGL 2.0 (OpenGL ES 3.0 Chromium) | WebKit WebGL` 上下文。

此前 `pnpm run repro:dev-broadcast-v2` 报错退出的真实原因有 3 个：

1. **`public/` Git Submodule 未初始化**（最核心根因）
   - `public/` 是一个独立的 Git Submodule（`RocketCompanionStatics.git`），克隆后默认未初始化为空目录。
   - Vite 启动后请求 Microkernel 资源时回退返回了 SPA HTML，导致 `Verify Camera Microkernel Integrity`（`preflight.camera_microkernel`）校验 SHA-256 失败（期望 `b8adc96e...`，实际 `81c41995...`），引擎启动中止，测试端卡死在 `Timeout waiting for Host boot`。
2. **`finally` 中的同步 `fs.rmSync` 掩盖真实错误（`ENOTEMPTY`）**
   - `scripts/runDevBroadcastReproV2.mjs` 在 `finally` 中 `chromeProc.kill('SIGKILL')` 后立即同步执行 `fs.rmSync(userDataDir, { recursive: true, force: true })`，因 Chrome 子进程尚未完全释放 `Default` 目录锁，抛出 `ENOTEMPTY`，把前面真正的 `Timeout waiting for Host boot` 异常吞掉了。
3. **缺少 3v3 `.rcreplay` 测试夹具**
   - 初始化 `public/` 后，Host/Client 可正常启动并通过 Bug 1，但执行到 REPRO 2 时因 `/tmp/rc-replay-3v3-1-0-2026-11-13T07-26-58.rcreplay` 不存在而失败。

---

## 二、新建 `scripts/env/` 工具目录与改进内容

在仓库内新增独立目录 `scripts/env/`，并在 `package.json` 暴露内置指令：

| 文件 / 指令 | 作用 |
| --- | --- |
| `scripts/env/setup.sh` (`pnpm run env:setup`) | **一键环境初始化/修复脚本**（幂等、非交互）：自动安装缺失的 `pnpm`、执行 `git submodule update --init --recursive` 初始化 `public/`、安装 `node_modules`、生成 3v3 `.rcreplay` 夹具、可选 `--with-rust` / `--with-chrome`，并自动执行 `doctor.mjs` 校验。 |
| `scripts/env/doctor.mjs` (`pnpm run env:doctor`) | **内置环境诊断指令**：检查 Node、pnpm、`node_modules`、`public/` 子模块、Chrome 二进制、**实际启动 Headless Chrome 进行 WebGL2 SwiftShader 渲染探测**、3v3 `.rcreplay` 夹具及可选 Rust 工具链；缺失时明确提示运行 `pnpm run env:setup`（或 `bash scripts/env/setup.sh`）。 |
| `scripts/env/generateReplayFixture.ts` | 使用仓库自身的 `RCReplayBinaryEncoder`（`serializeRCReplayBinaryContainer`）生成合法的 V2 3v3 6 车 `.rcreplay` 二进制夹具（固定 6 车不同名与稳定 Boost 值，避免外部文件缺失）。 |
| `scripts/env/reproPreflight.mjs` | 被 `runDevBroadcastReproV2.mjs` 与 `runDevBroadcastReproV3.mjs` 复用的前置门禁与安全清理模块：启动前检查 `public/` 与 Chrome（缺失直接提示 `pnpm run env:setup`，不再盲等 60 秒超时）；自动按需生成 `.rcreplay` 夹具；捕获 CDP `console.error` / `exceptionThrown`；`safeCleanupChrome` 等待进程退出并带重试清理 `userDataDir`，彻底消除 `ENOTEMPTY` 掩盖真因。 |
| `scripts/env/README.md` | 使用说明与故障排查文档。 |

---

## 三、实测验证结果（在当前容器内全部实跑通过）

1. **`pnpm run env:doctor`**：PASS
   - `Headless WebGL (SwiftShader): WebGL 2.0 (OpenGL ES 3.0 Chromium) | WebKit WebGL`
   - `public/ submodule: initialized`
2. **`pnpm run check`（radar + typecheck）**：PASS (GREEN)
   - `1428/1428 files (100.0% TS)`, `tsc --noEmit` clean, `NameTagScatter=1/1`, `BoolHotspots=7`, `UnscopedCaches=0`
3. **`pnpm run radar:audit`**：PASS (GREEN)
4. **`pnpm run test:multiplayer-suite`**：PASS（24/24 全部通过）
5. **`pnpm run repro:dev-broadcast-v2`（删除 `/tmp/*.rcreplay` 后实测自动生成 + 全流程 E2E）**：**PASS（`reproduced=0/4`）**
   - `bug-1-freeplay-goal-replay`: `PASS_FIXED`
   - `bug-2-replay-ui-and-roster`: `PASS_FIXED`
   - `bug-3-camera-player-switch-sync`: `PASS_FIXED`
   - `bug-4-exhibition-countdown-321`: `PASS_FIXED`
6. **`pnpm run repro:dev-broadcast-v3`（在 `2565774` 底座实测运行）**：**已可在当前容器完整运行至结束**（输出完整 5 项遥测，`reproduced=4/5`，其中 `bug-5` 已为 `PASS_FIXED`，其余为 Wave 2/3 待修项）。
