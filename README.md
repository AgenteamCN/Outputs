# RocketCompanion 环境工具链 + 6-Ducky 超高速 WASM 对局 Replay 生成与 E2E 跑通报告

- **基准分支与提交**：`z/agy/rc/multiplayer-recheck` (`2565774`)
- **新分支**：`z/agy/rc/env-doctor-repro`
- **两次 Verified Bot Push（GitHub GraphQL `createCommitOnBranch`，`verified: true`）**：
  1. **Push 1 (`e302483`)**：`feat(env): add scripts/env one-click setup, WebGL doctor, replay fixture generator & E2E preflight`
  2. **Push 2 (`69d052b`)**：`feat(env): simulate 6-Ducky 3v3 WASM match to auto-generate .rcreplay fixture`

---

## 一、核心成果

### 1. 解决 `repro:dev-broadcast-v2` / `v3` 跑不起来的根因
- 容器内 SwiftShader WebGL2 可正常渲染（`WebGL 2.0 (OpenGL ES 3.0 Chromium) | WebKit WebGL`）。
- 此前失败的根因是 `public/` git submodule 未初始化（导致 `Verify Camera Microkernel Integrity` SHA-256 校验失败并卡死在 `Timeout waiting for Host boot`）以及 `finally` 同步 `fs.rmSync` 抛出 `ENOTEMPTY` 掩盖了真实错误。

### 2. 新增 `scripts/env/` 独立工具目录与内置指令
- `pnpm run env:setup`（`scripts/env/setup.sh`）：一键安装/修复 `pnpm`、`public/` submodule、`node_modules`、`.rcreplay` 夹具（支持 `--with-rust` / `--with-chrome`）。
- `pnpm run env:doctor`（`scripts/env/doctor.mjs`）：环境诊断 + Headless Chrome WebGL2 SwiftShader 实测探测。
- `scripts/env/reproPreflight.mjs`：接入 `runDevBroadcastReproV2.mjs` 与 `runDevBroadcastReproV3.mjs`，启动前预检（缺失直接提示 `pnpm run env:setup`）、按需自动生成 `.rcreplay`、捕获 CDP 浏览器错误并安全清理 Chrome 临时目录。

### 3. 6 只 Ducky 超高速 WASM 物理模拟生成 `.rcreplay`（不再依赖外部文件）
- `pnpm run replay:generate-ducky`（`scripts/env/generateReplayFixture.ts`）：
  - 直接在 Node 中加载 `RocketSim` WASM（`core.wasm` + `public/arenamesh/`）与 `multiCarHeuristicDecide`（6 只 Ducky 战术 AI：3 Blue vs 3 Orange）。
  - 以超高速无头物理步进模拟 720 帧（1440 ticks = 12 秒 3v3 对局），仅耗时 **~272ms**，直接打包并 GZIP 输出为 V2 `.rcreplay` 文件（~234 KB），既可供 E2E 自动加载，也可供现场测试使用。

---

## 二、最终实测门禁（在 `69d052b` 上全量实跑通过）

- `pnpm run replay:generate-ducky`：PASS（`272ms` 生成 `234,618` bytes 3v3 6-Ducky WASM 对局 `.rcreplay`）
- `pnpm run check`（`radar` + `tsc --noEmit`）：PASS (GREEN，`1428/1428` TS)
- `pnpm run radar:audit`：PASS (GREEN)
- `pnpm run test:multiplayer-suite`：PASS（24/24）
- `pnpm run repro:dev-broadcast-v2`（直接加载 6-Ducky WASM `.rcreplay`）：**PASS（`reproduced=0/4`，4 项全部 `PASS_FIXED`）**
