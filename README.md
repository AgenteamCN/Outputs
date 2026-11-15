# RocketCompanion 深度扫描报告：Vite Git Checkout 自动刷新机制 & UI Tuner 启动性能排查

> **扫描基线**：`https://github.com/Antigravi-ty/RocketCompanion` (`alpha` 分支，commit `8d19408a1a7a375e23a1e69cf5edd8e111e1bda0`)
> **验证环境**：Node `v22.23.5` / pnpm `12.21.0` / Vite `5.4.21` / Headless Chrome（沙盒本地实测）

---

## 一、Vite Server 在 `git checkout` 时的自动刷新机制深度调查与优化方案

### 1.1 现状盘点：仓库里“已经做了什么”与“差的那一点点意思”

仓库中其实已经搭建了一套非常用心的 Git HEAD 监听骨架，涉及以下核心文件：
- `scripts/git/gitHeadState.ts`：读取主仓库与 `public` 子模块的 `.git/HEAD`、`logs/HEAD`、`packed-refs` 组成复合签名（`CompositeGitHeadState`），并检测 `.git/index.lock` 与 `.git/HEAD.lock`。
- `scripts/vite/gitHeadAutoRestartPlugin.ts`：Vite `serve` 阶段插件，挂载 `/__rc_git_head` 中间件、注入客户端轮询脚本，并在服务端通过 `setInterval(..., 150)` 检测签名变化，触发 `server.restart(depsChanged)`。
- `scripts/tauriDevWatch.ts`：Tauri 开发态外层守护进程（通过 `RC_DISABLE_VITE_GIT_RESTART=1` 接管重启）。
- `scripts/devLauncher.ts`：单进程内编排 4 个 Vite Server（`:3000 entry`、`:3001 rc-https`、`:3002 tuner`、`:3003 rc-http`）。

**但实际运行中，正是以下 5 个关键竞态与设计缺口，导致了“checkout 一旦激活页面就抢先刷新，但此时系统还没准备好”的问题：**

1. **致命竞态 1：客户端轮询 `/__rc_git_head` 在服务端 `settleTimer` 期间抢先触发 `window.location.reload()`**
   - 在 `scripts/vite/gitHeadAutoRestartPlugin.ts` 中，服务端检测到 HEAD 变化后，会进入一个 `settleTimer`（当前仅 `120ms`）才把 `isRestarting` 设为 `true`。
   - 然而，`/__rc_git_head` 中间件每次收到请求都会**实时读取** `readCompositeGitHeadState()`。
   - 在 `git checkout` 刚改完 `.git/HEAD`、释放 `index.lock` 但工作区文件尚未稳定（或 `settleTimer` 尚未到期）的窗口期内，`isRestarting` 依然是 `false`。一旦浏览器端 `setInterval(checkGitHead, 1200)` 刚好在这个瞬间请求 `/__rc_git_head`，就会拿到**新的 `signature`** 且 `restarting: false`，于是客户端直接执行 `window.location.reload()`——**此时 Vite 甚至还没开始 `server.restart()`！**
   - *注意*：如果单纯把 `settleTimer` 从 `120ms` 拉长到 `1000ms` 而不改 `/__rc_git_head` 的逻辑，反而会让这个“抢跑刷新窗口”从 120ms 扩大到 1000ms，触发率更高！

2. **致命竞态 2：Vite 内置 Chokidar (`server.watcher`) 在 checkout 期间未被抑制**
   - `git checkout` 会连续修改/增删大量工作区文件。在服务端等待 settle 的这段时间里，Vite 自身的 `chokidar` 仍在实时监听文件变更，并立刻向浏览器推送 HMR `update` 或 `full-reload` 事件。
   - 结果就是：浏览器在半切换状态的文件系统上触发了 Vite 原生刷新，紧接着服务端又执行 `server.restart()` 杀掉连接，页面再次报错或白屏。

3. **生命周期 Bug：`cleanup()` 提前杀死了 `pollTimer` + `server.ws.send()` 时机失效**
   - 在 `gitHeadAutoRestartPlugin.ts` 的 `settleTimer` 回调中，代码在 `await server.restart(depsChanged)` **之前**就调用了 `cleanup()`（清除了 `pollTimer`）。如果 `server.restart()` 抛错，`catch` 块只重置了 `isRestarting = false`，但 `pollTimer` 已经永远停止工作。
   - 另外，`await server.restart()` 会关闭旧的 WebSocket Server 并重建实例；在其之后调用 `server.ws?.send({ event: 'rc:git-head-changed' })` 时，旧实例已关闭而新实例尚未有客户端连入，该事件大概率丢失。

4. **`devLauncher.ts` 多端口编排与 `vite.config.ts` 变更盲区**
   - `pnpm dev` (`scripts/devLauncher.ts`) 在同一个 Node 进程内用 `configFile: false` 启动了 4 个 `ViteDevServer`。
   - 当 `git checkout` 发生时，4 个服务器实例内的插件会**同时各自触发 `server.restart()`**，并发争抢资源与控制台输出。
   - 更关键的是：因为传了 `configFile: false`，`vite.config.ts` 和 `scripts/vite/*.ts` 已经缓存在 Node 的 ESM 模块缓存中。如果两次 commit 之间修改了 `vite.config.ts` 或插件代码，`server.restart()` **不会**加载新配置，必须重启外层 Node 进程才能生效。

5. **macOS `.DS_Store` 及其他非源码目录未从 Watcher 中排除**
   - 在 `vite.config.ts` 中，`server.watch.ignored` 目前仅配置了 `['**/src-tauri/**']`。
   - macOS Finder 会在任意打开过的目录（包括 `src/`、`public/`、根目录）生成或更新 `.DS_Store` 文件；此外 `.cache/`（存放 `resolvedProfile.json`）、`rc-vite-cache/`、`dist/` 等也未显式忽略。虽然 `.DS_Store` 不在 JS 模块图中，但 Chokidar 与 `@tailwindcss/vite` v4 的文件扫描器仍会感知到这些变更事件，造成不必要的抖动或样式重扫。

---

### 1.2 针对“等待 1 秒再重启 + 两步极简刷新”的优化方案与业内最佳实践

你的思路（**checkout 后等待 ~1 秒让系统与磁盘稳定 -> 重启服务器 -> 触发网页刷新**，实现用户侧只需 **`git checkout` + 网页刷新** 两步甚至全自动一步到位）方向完全正确。结合业内实践，推荐按以下方式落地：

#### 方案 A：直接升级现有 `rcGitHeadAutoRestartPlugin`（零外部依赖，立即可用）

1. **引入“Checkout 静默期 + 1000ms Trailing Debounce”与“Ready Gate（就绪门控）”**：
   - 在服务端维护 `bootId = randomUUID()`（每次 `configureServer` 初始化时生成新值）以及 `pendingCheckout` 状态。
   - 一旦检测到 `current.locked`（`index.lock` / `HEAD.lock` 存在）或 `current.signature !== baselineSignature`：
     - **立即将服务器标记为 `restarting / settling` 状态**：`/__rc_git_head` 必须返回 `{ restarting: true, signature: null, bootId }`，**严禁在重启完成前返回新的 `signature`**，彻底堵死客户端抢跑刷新的路径。
     - **启动 1000ms Trailing Debounce**：不仅看 `.git/HEAD`，同时在 `server.watcher.on('all', ...)` 上挂一个轻量监听器——在 `pendingCheckout` 期间，只要工作区还有任何非忽略文件变更，就重置 `1000ms` 定时器，直到文件变动完全静默满 1000ms（上限可设比如 3000ms 兜底）才执行重启。
   - **在静默期内抑制 Vite 原生 HMR 推送**：
     - 在 `pendingCheckout` 为 `true` 时，可通过拦截 `server.ws.send`（暂时丢弃或缓冲 `update` / `full-reload` 消息），防止 Vite 在 1000ms 等待期间把浏览器刷新掉。

2. **客户端“等待新 `bootId` 就绪后再刷新”**：
   - 客户端在 `transformIndexHtml` 中记录初始 `bootId`。
   - 当检测到 `data.restarting === true` 或 `vite:ws:disconnect` 时，进入“等待新服务就绪”状态（可在页面顶部挂载一个极轻量的非侵入提示条，如 `Git HEAD switched — waiting for Vite server...`）。
   - 客户端以 300–500ms 间隔探测 `/__rc_git_head`：只有当接口返回 HTTP 200、且 `data.restarting === false`、且 `data.bootId !== initialBootId` 时，才调用 `window.location.reload()`。
   - 这样既不需要手动停/启服务器，也不会在半成品状态下刷新。

3. **完善 `vite.config.ts` 的 `server.watch.ignored`（屏蔽 macOS `.DS_Store`）**：
   ```ts
   watch: {
     ignored: [
       '**/src-tauri/**',
       '**/.git/**',
       '**/.cache/**',
       '**/dist/**',
       '**/rc-vite-cache/**',
       '**/.DS_Store',
       '**/._*',
       '**/.AppleDouble',
     ],
   }
   ```
   *(注：`gitHeadAutoRestartPlugin` 是通过 Node `fs` 轮询 `.git/HEAD`，不依赖 Chokidar，因此忽略 `**/.git/**` 不会影响 HEAD 检测，反而能大幅降低 Vite 自身的文件监听开销。)*

#### 方案 B：业内针对“多 Commit 高频 Checkout”的进阶推荐方案

1. **Git `post-checkout` Hook 确定性通知（替代纯轮询）**：
   - 业内成熟工程（如大型 Monorepo / Tauri+Vite 项目）常配合 `.git/hooks/post-checkout`：Git 保证该 Hook **只在工作区所有文件更新完毕之后**才执行。
   - 在 Hook 中执行一次非阻塞 `curl -s -X POST http://127.0.0.1:3001/__rc_git_checkout_done`（或在项目根目录写入 `.cache/.git-checkout-stamp`），Vite 收到确定性信号后再等待固定 500–1000ms 让 Chokidar 消化事件并重启，可靠性远高于纯猜测性的锁文件轮询。

2. **外层 Supervisor 统一接管（解决 `devLauncher.ts` 多端口与 `vite.config.ts` 热加载）**：
   - 参考 `scripts/tauriDevWatch.ts` 的模式，让 `scripts/devLauncher.ts` 变成纯外层守护进程（或由外层脚本 `spawn` 子进程运行 4 个 Vite Server）。
   - 当检测到 `git checkout`（特别是涉及 `vite.config.ts`、`scripts/vite/**`、`package.json`、`pnpm-lock.yaml`）时，由外层进程统一先等待 1000ms 稳定，再优雅重启整个子进程，彻底避免 4 个 Vite 实例各自并发 `server.restart()` 以及 ESM 配置缓存不更新的问题。

3. **`git worktree` 并行工作区（跨 Commit 频繁对比的终极实践）**：
   - 如果场景是在 2–3 个特定 commit 之间反复横跳对比效果，频繁 `git checkout` 会反复触发大量磁盘 IO 与 Vite 依赖/模块缓存失效。业内更推荐直接用 `git worktree add ../RocketCompanion-compare <commit>` 开出第二个工作目录，各自独立缓存、互不干扰，切窗口即可对比。

---

## 二、UI Tuner (`dev:tuner`, Port 3002) 启动极慢且控制台无输出调查

### 2.1 本地沙盒实测数据

我们在本地沙盒中清理缓存后，实际运行 `RC_VITE_ROLE=tuner npx vite --port 3002`，并使用 Headless Chrome 与 `DEBUG=vite:deps` / `DEBUG=vite:time` 进行了完整复现与计时：

| 测试场景 | 实测指标 | 耗时 / 数量 | 控制台输出情况 |
| :--- | :--- | :--- | :--- |
| **Vite CLI 启动** | `VITE v5.4.21 ready` | **~463 - 516 ms** | 打印 Local/Network URL 后**完全静默** |
| **冷启动（无 `.vite/tuner` 缓存）** | 首次请求 `/src/ui-tuner/main.tsx` | **7.136s ~ 7.276s** | **无任何输出**（实际卡在 `vite:deps`） |
| **冷启动 `vite:deps` 内部耗时** | esbuild 依赖扫描 + 预打包 | 扫描 **6.99s ~ 8.78s** + 打包 **1.37s** = **~10.16s** | 默认隐藏，仅 `DEBUG=vite:deps` 可见 |
| **冷启动 Chrome 完整首屏** | `google-chrome --headless --dump-dom` | **12.434s** | 无任何 Vite 进度提示 |
| **热启动（已有 `.vite/tuner` 缓存）** | 首请求至最后一个模块完成 (`vite:time`) | **13.03s**（Chrome 总计 **15.09s ~ 17.43s**） | 请求了 **~575 个模块**，控制台静默 |

**结论**：不仅冷启动超过 10 秒，**即使缓存已经预热（热启动），UI Tuner 首屏依然需要 13~17 秒！**

---

### 2.2 四大核心根因分析

#### 根因 1（最致命，冷/热启动都慢的元凶）：`liveSubtreeHashScanner.ts` 同步全量 `?raw` 导入了 549 个文件
通过追踪默认首屏的模块依赖链，我们定位到如下调用路径：
1. `src/ui-tuner/workspaceSpecs.ts` 中，`readWorkspaceFromUrl('overview')` 默认回退到 `'primitives'`，而 `WORKSPACE_SPECS.primitives.defaultTab` 为 **`'panels'`**（即 `PanelsPage`）。
2. `PanelsPage.tsx` -> `PanelShellTuner.tsx` -> `useReviewLedgerStore` (`src/ui-tuner/components/review/reviewLedgerStore.ts`) -> `reviewLedgerAdapter.ts` -> **`src/ui-tuner/components/review/liveSubtreeHashScanner.ts`**。
3. 在 `liveSubtreeHashScanner.ts` 顶层存在如下代码：
   ```ts
   const rawUiCoreSources: Record<string, string> =
     typeof importMetaGlob === 'function'
       ? importMetaGlob('../../../ui-core/**/*.{ts,tsx}', {
           query: '?raw',
           import: 'default',
           eager: true,
         })
       : {};
   ```
   紧接着在模块顶层同步遍历 `Object.entries(rawUiCoreSources)` 计算每个文件的 `computeVisualFileHash(sourceText)`。
4. 沙盒实测 `src/ui-core/**/*.{ts,tsx}` 共有 **549 个文件**！这意味着无论用户打开 UI Tuner 的哪个默认页面（哪怕只是 Workspace 1 的基础组件面板），浏览器在启动瞬间都会被迫向 Vite 发起 **549 个 `?raw` 文件转换与 HTTP 请求**，并在主线程同步完成全量源码 Hash 计算。

#### 根因 2（热启动 13s 瀑布流）：Port 3002 使用 HTTP/1.1，触发 6 并发连接队头阻塞
- 在 `vite.config.ts` 中，主应用 `rc-https` (:3001) 启用了 `basicSsl()` + `http2HighConcurrencyPlugin()`（将 Node HTTP/2 `maxConcurrentStreams` 提升至 1000，专门用于应对几百个 ESM 模块的并发爆发）。
- 而 `tuner` (:3002) 运行在**普通 HTTP/1.1** 上，浏览器对同一源最多只有 **6 个并发 TCP 连接**。
- 当 `575+` 个未打包的 ESM 与 `?raw` 模块通过 HTTP/1.1 串行排队加载时，即便单文件转换只需 `0.2ms`，整体网络瀑布流依然会拉长到 **13 秒以上**。

#### 根因 3（冷启动 +7~9s 且控制台静默）：`optimizeDeps` 全量 esbuild 扫描 + 缺失 `warmup`
- 在 `vite.config.ts` 中，`tuner` 的 `optimizeDeps.include` 已经显式列出了全部 13 个第三方依赖（`uiInclude`），但仍设置了 `entries: ['tuner.html', 'src/ui-tuner/main.tsx']` 且未关闭自动扫描。
- 冷启动时，Vite 会先打印 `ready in 463 ms` 欺骗开发者“已就绪”，随后在第一次访问 `/src/ui-tuner/main.tsx` 时**阻塞 HTTP 响应**，用 esbuild 递归爬取 `UITunerApp.tsx` 中全部 26 个 `React.lazy` 页面以及 `liveSubtreeHashScanner.ts` / `surfaceRegistry.ts` / `floatingSurfaceRegistry.ts` 所覆盖的数百个 TS/TSX 文件，仅扫描就耗时 **7.0 ~ 8.8 秒**，且默认不在终端打印任何提示。
- **本地实测对比**：我们在沙盒中仅对 `tuner` 配置临时开启 `optimizeDeps.noDiscovery: true` 后重新冷启动，`/src/ui-tuner/main.tsx` 的首次响应时间从 **`7,276 ms` 骤降至 `629.7 ms`（提速 11.5 倍）**。
- 此外，`vite.config.ts` 中 `server.warmup` 仅对 `rc-https` / `rc-http` 开启，`tuner` 为 `undefined`；`scripts/devLauncher.ts` 也仅对 `entry` (:3000) 调用了 `server.warmupRequest()`，完全没有预热 `tuner`。

#### 根因 4（次要累加项）：`surfaceRegistry` 急切加载 & Tailwind/外链字体开销
- `src/ui-core/manifest/surfaceRegistry.ts` 与 `floatingSurfaceRegistry.ts`（以及 `floatingWindowRegistry.tsx`）同样使用了 `import.meta.glob(..., { eager: true })` 预加载全部 `Layer*Recipe.tsx` 与 `*FloatingWindow.tsx`。
- `@tailwindcss/vite` v4 编译 `src/styles/ui-tuner.css` 在热缓存下约耗时 `~680ms`；`tuner.html` 中的 Google Fonts 外链 `<link rel="stylesheet">` 在离线或受限网络下也会额外拖慢首屏渲染。

---

### 2.3 UI Tuner 启动优化建议（预计可将首屏从 12~17s 降至 1.5s 以内）

1. **彻底消除 `liveSubtreeHashScanner.ts` 的启动期 549 次 `?raw` 请求（最高优先级）**：
   - 将 `importMetaGlob('../../../ui-core/**/*.{ts,tsx}', { query: '?raw', eager: false })` 改为**懒加载**，仅当用户实际进入需要源码 Hash 对比的审阅/导出面板时再按需触发加载与计算；
   - **更优方案**：由 Vite dev 插件提供一个轻量中间件接口（如 `/__rc_ui_core_hashes`），在 Node 服务端直接读取磁盘文件计算 Hash 并缓存返回 JSON，彻底把 549 次浏览器 HTTP 请求降为 **1 次**。
2. **为 `tuner` 开启 `optimizeDeps.noDiscovery: true`（或精简扫描入口）**：
   - 既然 `vite.config.ts` 已经为 `tuner` 维护了完整的 `uiInclude` 白名单，开启 `noDiscovery: true` 可直接免除冷启动 7~8.8 秒的 esbuild 全量文件爬取。
3. **补齐 `tuner` 的预热与控制台进度提示**：
   - 在 `vite.config.ts` 中为 `tuner` 配置 `server.warmup: { clientFiles: ['./src/ui-tuner/main.tsx', './src/ui-tuner/UITunerApp.tsx'] }`（或在 `devLauncher.ts` 中对 `tuner` 同样执行 `warmupRequest`）。
   - 在自定义 Vite 插件的 `configureServer` 中，针对冷启动依赖预构建与预热输出清晰的终端状态日志（如 `[rc-tuner] Optimizing dependencies & warming up...` -> `ready`），消除“控制台无输出干等 10 秒”的黑盒体验。
4. **评估 `surfaceRegistry` / `floatingSurfaceRegistry` 的按需加载**：
   - 在完成上述第 1 步后，UI Tuner 首屏模块数将从 ~575 个锐减，HTTP/1.1 下的网络瀑布流将基本消失；若仍需进一步提速，可将 `Recipes` / `FloatingWindows` 的 `import.meta.glob` 也改为懒加载或在 `tuner` 角色下按需启用 HTTP/2。
