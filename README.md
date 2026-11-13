# RocketCompanion Bootstrap 卡顿/刷新时序调查与修复报告

## 1. PR 创建与合并（`alpha` 分支）
- 已将已验证的 Commit `d69e9471bfdbcd854c778e09d708b676a6c7fa76`（分支 `z/agy/ui/fix-floating-window-schema`）创建 Pull Request 并合并进 `alpha`：
  - **PR #11**: https://github.com/Antigravi-ty/RocketCompanion/pull/11
  - **Merge Commit**: `e943d507b1c70d84b0b231ef457b4baaa3ab9525`

---

## 2. Bootstrap 卡顿与分支切换刷新问题调查结论

通过 Headless Chrome + CDP 对 `https://localhost:3001/` 进行全链路抓包与计时，定位到以下 4 个核心根因：

### (1) 为什么经常卡在 `0%` 很久（实测冷启动 27.2s、刷新 10.0s）？
1. **UI 与引擎模块树串行加载（~1,186 个 ESM 请求）**：`src/main.ts` 串行 `await import('./game/GameBootstrap.ts')` 和 `await import('./ui-system/index.tsx')`（后者静态引入了大量开发面板、Lab、Chart、Training Pack 解析器等组件），在 Stage 1~3 还没开始前就阻塞了 ~1,186 个模块请求；且自签名 HTTPS (`basicSsl()`) 下浏览器会绕过磁盘缓存，导致刷新也要重走 1,186 次模块请求。
2. **`MinimumProgressLoader.onTaskStart` 未刷新百分比 DOM**：`src/main.ts` 传入 `overallPercent: 2`，但 `MinimumProgressLoader.onTaskStart` 只更新了内部属性而没有更新 `#hig-overall-percent` 和 `#hig-primary-fill`，导致进度条在加载整个模块图期间一直死锁显示 `0%`。
3. **Vite `rc-https` / `rc-http` 未开启 `server.warmup`**：服务启动后不会后台预编译核心入口，把 27s 的冷变换开销全部压在第一个浏览器请求上。

### (2) 为什么有时卡 `30%`、`60%~70% (streaming 2/3 ready)` 或 `88%`？
1. **`~30%`（Stage 3 `physics.mesh.stream`）**：`StreamDownloadTracker.fetchBuffer` 并发拉取 16 个 `.cmf` 碰撞网格分片时，单个请求超时高达 `30000ms` 且**没有任何重试机制**，一旦单个 HTTP/2 流抖动就会干等 30s；同时每个 chunk 都直接触发 DOM 更新。
2. **`~60% - 70%`（Stage 4 `geometry.scene.build` 显示 `Streaming ... (2/3 ready)`）**：`loadAllArenaWorldAssets` 并行跑 3 个任务（Ball 纯程序生成秒完 -> 场馆 `DriftwoodsStadiumLoader` -> 车辆与 BoostPad `BoostPadTemplateLoader`）。其中 `GLTFLoader.loadAsync` **完全没有超时与重试**，一旦某一个 GLB 请求在 HTTP/2 上卡住，就会卡在 `2/3 ready`（恰好对应权重 `50% + 30% * 2/3 = 70%`）直到 `LoadingTask` 60s 超时报错。
3. **`~88%`（Stage 5 `shaders.prewarm`）**：实测在软件渲染（SwiftShader）或慢速 GPU 下，全主题 Shader 预热超过 30s 会触发 `LoadingTask` 超时并因 `critical: true` 直接弹出 Fatal Error 阻断进入游戏。

### (3) 为什么切换分支后自动刷新 / 手动刷新同标签页容易卡住，而新开标签页没事？
1. **时序倒置（你的直觉完全正确）**：在 `scripts/vite/gitHeadAutoRestartPlugin.ts` 中，旧逻辑在检测到 `HEAD` 变更后**先**向浏览器发送 `rc:git-head-changed`（触发 `window.location.reload()`）和 `full-reload`，**然后才**执行 `await server.restart(true)`。浏览器在旧页面立即 reload 时，正好撞上正在关闭 HTTP/2 服务并强制重打包 `node_modules/.vite` (`forceOptimize=true`) 的 Vite 进程，导致旧标签页的 HTTP/2 连接半开/挂起；而几秒后新开标签页时，Vite 已经重启完成，所以完全正常。
2. **强制 `server.restart(true)` 浪费 5~10s**：每次切分支无论 `package.json`/`pnpm-lock.yaml` 是否变化，都传 `true` 强制清掉 `node_modules/.vite` 缓存重新用 esbuild 打包 `three`/`react` 等依赖。
3. **同标签页刷新未清理资源**：页面刷新/关闭时没有 `pagehide` 钩子调用 `destroyEngine()`，旧 WebWorker / WebGL / AudioContext 会在刷新过渡期残留。

---

## 3. 已实施的修复（新分支 `z/agy/rc/fix-bootstrap-hangs`）

- **Verified Bot Commit**: `d4492d95c7883b4419d98036f8234a2b63d6d6c6`（基于 `alpha` `e943d50`）
- **改动文件**：
  1. `scripts/vite/gitHeadAutoRestartPlugin.ts`：
     - 检测 `package.json` / `pnpm-lock.yaml` 变化，仅在依赖变更时传 `forceOptimize=true`，常规切分支使用 `server.restart(false)` 保留 `.vite` 预打包缓存。
     - 重启期间 `/__rc_git_head` 返回 `{ restarting: true, signature: null }`；**移除重启前的 `full-reload`**，改为等 `await server.restart(...)` 完成后再通知客户端，客户端结合 `vite:ws:disconnect` / `vite:ws:connect` 与 `/__rc_git_head` 状态确认新服务器就绪后再执行单次 `window.location.reload()`。
  2. `vite.config.ts`：为 `rc-https` / `rc-http` 配置 `server.warmup.clientFiles`（预热 `main.ts`、`GameBootstrap.ts`、`ui-system/index.tsx`）。
  3. `src/loader/MinimumProgressLoader.ts`：修复 `onTaskStart` 未更新 `#hig-overall-percent` 与 `#hig-primary-fill` 的问题。
  4. `src/main.ts`、`src/game/GameBootstrap.ts`、`src/loader/engineLoadingPipeline.ts`：
     - 将 `ui-system/index.tsx` 与引擎 Bootstrap Stages 1~3 并行重叠加载，仅在 Stage 4 需要挂载 UI 前解析 `loadMountUI`。
     - 为模块加载阶段增加 500ms 实时耗时心跳提示；注册 `pagehide` 自动调用 `destroyEngine()` 清理 WebGL/Workers/Audio。
     - 将 Stage 5 `shaders.prewarm` 设为 `critical: false`，慢速/软件 GPU 超时时优雅降级。
  5. `src/loader/StreamDownloadTracker.ts`：`.cmf` 碰撞分片请求改为 `12s` 单次超时 + 最多 3 次自动重试，并对 `_notify` 增加 50ms 节流。
  6. `src/utils/assetLoadRetry.ts`、`src/entities/stadium/driftwoods/DriftwoodsStadiumLoader.ts`、`src/entities/arena/BoostPadTemplateLoader.ts`：新增共享 `loadGltfWithRetry` / `loadTextureWithRetry`，为 Stage 4 场馆与 BoostPad GLB/纹理加载加上超时与自动重试。

---

## 4. 验证结果
- `pnpm check`（`radar` + `tsc --noEmit`）：**GREEN**（1355/1355 TS 文件，0 循环依赖，0 越层引用，0 TS 错误）。
- `pnpm build`：**PASS**。
- Headless Chrome CDP 实测对比（`https://localhost:3001/`）：
  - 冷启动模块图阶段（`Loading Engine Core Modules...`）：从 **27.2s 降至 8.3s**（提速 **3.2x+**），且进度条立即跳出 `0%` 并持续更新耗时。
  - 冷启动与同标签页刷新均完整跑到 **`100% READY`**（此前冷启动在 SwiftShader 下 88% 超时崩溃）。
