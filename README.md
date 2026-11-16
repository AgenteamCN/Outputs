# UI Tuner (`dev:tuner`, Port 3002) 启动性能优化报告

- **分支**: `z/agy/ui/tuner-startup-optimization` (基于 `alpha` 最新提交 `8b6b0a2`)
- **Verified Bot Commit**: [`68c3ae37504259092e102a67b14e028afedd4f44`](https://github.com/Antigravi-ty/RocketCompanion/commit/68c3ae37504259092e102a67b14e028afedd4f44) (`signature.state: VALID`)

---

## 一、核心优化内容

### 1. 彻底消除启动期 549+ 次 `?raw` 请求，并保障审批页实时 Hash 准确性
- **`scripts/vite/tunerWsSyncPlugin.ts`** (`rcUiTunerSyncPlugin`):
  - 新增 Node 服务端预热缓存接口 `/__rc_ui_core_hashes`：在服务监听后异步扫描 `src/ui-core/**/*.{ts,tsx}`，复用 `src/ui-core/manifest/visualFileHash.ts` 的 `computeVisualFileHash` 在服务端直接计算并缓存全量 Hash（550 个文件实测仅 **~67–120ms**），以单次 JSON 响应返回给浏览器。
  - 通过 `server.watcher` 监听 `src/ui-core/` 的 `change` / `add` / `unlink` 事件，毫秒级增量更新服务端缓存，并通过 `server.ws.send('rc-ui-tuner:ui-core-hashes-updated', ...)` 实时推送给前端。
  - 增加清晰的控制台启动日志：`[rc-tuner] Initializing UI Tuner dev server & pre-warming hash cache...` 与 `[rc-tuner] UI Core hash cache ready: 550 files hashed in ...ms`。
- **`src/ui-tuner/components/review/liveSubtreeHashScanner.ts`**:
  - 移除顶层同步 `import.meta.glob('../../../ui-core/**/*.{ts,tsx}', { query: '?raw', eager: true })` 及主线程全量 Hash 计算。
  - 改为按需（首次调用 `detectChangedSubtreeFiles` / `computeLiveSubtreeCombinedHash` / `getLiveFileVisualHash` 时）通过 `fetch('/__rc_ui_core_hashes')` 单次拉取并缓存，同时订阅 `rc-ui-tuner:ui-core-hashes-updated` HMR 事件。
- **`src/ui-tuner/components/review/reviewLedgerStore.ts`**:
  - 新增 `hashesRevision` 状态并订阅 `subscribeLiveSubtreeHashes`，确保审批/审阅页面在异步拉取完成或 `src/ui-core/` 源码修改后自动触发 React 重渲染，保持审阅状态与过期检测实时准确。

### 2. 关闭 `tuner` 的全量 esbuild 依赖扫描 + 消除 Vite 5 二次重载 + 补齐预热
- **`vite.config.ts`**:
  - 为 `role === 'tuner'` 开启 `optimizeDeps.noDiscovery: true`（复用已完整声明的 `uiInclude` 白名单），免除冷启动 7~8.8 秒的全量文件爬取。
  - 同时设置 `optimizeDeps.holdUntilCrawlEnd: role === 'tuner' ? false : undefined`，解决 Vite 5.4.21 在 `noDiscovery: true` 下仍会在首次请求后触发二次 `runOptimizer` 与 `optimized dependencies changed. reloading` 页面重载的问题。
  - 为 `tuner` 补齐 `server.warmup: { clientFiles: ['./src/ui-tuner/main.tsx', './src/ui-tuner/UITunerApp.tsx'] }`。
- **`scripts/devLauncher.ts`**:
  - 在统一启动器中，除 `entry` 外同样为 `tuner` 调用 `server.warmupRequest('/src/ui-tuner/main.tsx')`。

### 3. 非阻塞外链字体加载
- **`tuner.html`**:
  - 将 Google Fonts (`JetBrains Mono`) `<link rel="stylesheet">` 改为 `media="print" onload="this.media='all'"`，避免离线或受限网络阻塞首屏渲染。

---

## 二、沙盒实测对比与验证

### 工具链补充
- 沙盒已预装 Node v22.23.5 / npm 10.9.9 / Google Chrome 157；补充全局安装了 `pnpm` (`12.21.0`) 并完成 `pnpm install --frozen-lockfile`。

### Headless Chrome 首屏实测 (`RC_VITE_ROLE=tuner` Port 3002)

| 测试场景 | 优化前 | 优化后 | 改善 |
| :--- | :--- | :--- | :--- |
| **冷启动**（清空 `.vite/tuner` 缓存） | 16.602s | **9.973s** | 消除 esbuild 扫描与二次 reload，提速 ~40% |
| **热启动**（已有 `.vite/tuner` 缓存，默认首页 `/`） | 7.134s（报告原测 15~17s） | **3.191s** | 提速 **2.2x+（对比原环境 ~5x）** |
| **审批页** (`/?workspace=approval`) `?raw` 请求数 | 549 次 | **0 次**（由 1 次 `/__rc_ui_core_hashes` 替代，550 文件服务端缓存命中） | `?raw` 瀑布流完全消除 |

### 架构与类型检查
- `pnpm run check`（`radar` + `tsc --noEmit`）：**STATUS: GREEN**（1389/1389 TS 文件，`tsc --noEmit` 0 错误，0 循环依赖/越界导入）。
