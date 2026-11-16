# Tauri 音频丢失与首次加载卡顿排查及修复报告

## 1. 分支与提交信息
- **基线提交**: `df4a3c80c569c21c1670d6027fed2fe3b2a3f570` (`z/agy/rc/multiplayer-recheck`)
- **新建分支**: `z/agy/tauri`
- **已验证签名提交**: [`78d0e0239792da9c518a62b54bba3581434adbe9`](https://github.com/Antigravi-ty/RocketCompanion/commit/78d0e0239792da9c518a62b54bba3581434adbe9) (`signature.isValid = true`, `state = VALID`)

---

## 2. 根因分析

### (1) 为什么「打开第二遍会丢失音频」？
在 Tauri 托盘模式（`RC_TRAY_ONLY=1` / `src-tauri/src/lib.rs`）中，关闭主窗口不会销毁进程，而是 `api.prevent_close()` + `window.hide()`，同时切换 macOS `ActivationPolicy::Accessory`。这会导致两层问题：
1. **隐藏时静音、二次 `show()` 时未完整恢复**：
   - `window.hide()` 触发 `handleFocus(false)`，将 `master.setActive(false)`、`slotPool.isMuted = true`、`audioEngine.silenceAll()`、并使 `stadiumSpatialEngine` 进入失焦静音状态；同时 macOS WKWebView 可能会将 `AudioContext` 切到 `suspended` / `interrupted`。
   - 二次打开窗口（`launch_main_window_with_config` 调用 `main_win.show(); main_win.set_focus();`）时，若 URL 参数未变则不会刷新页面。WKWebView 中 `window.focus` 可能早于 `document.hidden === false` 触发，导致 `handleFocus(true)` 误判为仍处于隐藏状态；此外非用户手势下的 `ctx.resume()` 在 WebKit 中可能挂起。
   - **关键漏洞**：`src/audio/audioContext.ts` 中的 `onAudioReady()` 仅恢复了 `master.setActive(true)` 和 `ballEngine._startLoops()`，**未解除 `slotPool.isMuted = false`，也未通知 `stadiumSpatialEngine.handleWindowFocusChange(true, ...)`**；且 `unlock()` 在 `ctx.state === 'running'` 时只检查 `!master.active || !master.connected`，**忽略了 `slotPool.isMuted === true`**，导致后续按键/点击也无法恢复音效。

### (2) 为什么「第一次加载会卡在音频 notification 上很久」以及频繁 `checkout` 后的现象？
1. **频繁 `git checkout` 导致 Vite/esbuild 崩溃**：
   - 从日志可见 `[vite] Pre-transform error: The service was stopped: write EPIPE` 与 `esbuild` `TypeError: callback is not a function`。在频繁 `checkout` 切换分支时，`devLauncher.ts` 重启 `devWorker.ts` 期间若旧 `esbuild` 服务中断，会导致前端静态/音频请求挂起或失败；且 `devLauncher.ts` 在 `devWorker` 非零退出后仅打印提示并停止自动拉起，必须手动重启 server 才能恢复（这正是「重启一下 server 就好了」的直接原因）。
2. **音频加载无超时保护与进度空窗**：
   - `src/audio/audioAssetLoader.ts` (`fetchAudioBuffer`) 与 `src/audio/EngineAudioLoopConditioner.ts` (`fetchAndDecodeBuffer`) 均未对 `fetch` 和 `ctx.decodeAudioData` 设置超时；一旦单个请求在 Vite 重启或 WKWebView 连接波动时挂起，后台 `Background Assets` 通知就会无限卡住。
   - `src/game/subsystems/GameAudioSubsystem.ts` (`setupAudio`) 中，`loadEngineAudioPackAssets` 在 `gameAudio.preload` 之前执行且未上报进度，`arena.prepareAssets` 在之后执行也未上报进度，导致通知长时间停留在初始百分比或 `65/65`。

---

## 3. 修复内容

1. **`src/audio/audioContext.ts`**：
   - `onAudioReady()` 在页面可见或允许后台音频时，同步恢复 `slotPool.isMuted = false` 与 `stadiumSpatialEngine.handleWindowFocusChange(true, ...)`。
   - `unlock()` 在 `ctx.state === 'running'` 时增加 `subsystemState.slotPool?.isMuted` 检查。
   - `handleFocus(true)` 遇到 `document.hidden === true`（WKWebView 时序竞态）时增加 80ms 延迟复检。
   - 监听 `rc-tauri-window-shown` 自定义事件，主动触发 `handleFocus(true, 'tauri.window_shown')`。
2. **`src-tauri/src/window_manager.rs`**：
   - 在 `launch_main_window_with_config` 执行 `main_win.show()` 与 `main_win.set_focus()` 后，向主窗口派发 `rc-tauri-window-shown` 事件。
3. **`src/audio/audioAssetLoader.ts` & `src/audio/EngineAudioLoopConditioner.ts`**：
   - 为 `fetch`（15s `AbortController`）与 `ctx.decodeAudioData`（10s）增加超时保护。
   - 为 `loadEngineAudioPackAssets` 增加 `onProgress` 回调，并在失败时清理 `packAssetCache` 避免缓存坏 Promise。
4. **`src/game/subsystems/GameAudioSubsystem.ts`**：
   - 将 `onProgress` 透传给 `loadEngineAudioPackAssets` 和 `arena.prepareAssets`，消除加载通知进度条长时间静止的假卡死现象。
5. **`scripts/devLauncher.ts`**：
   - 当 4 端口 `devWorker` 因意外错误（如频繁 `checkout` 触发 `esbuild` `EPIPE`）非正常退出时，增加 1000ms 自动重启恢复，避免端口假死需要手动重启。

---

## 4. 验证情况
- `pnpm run check`（`radar` 1415/1415 TS + `tsc --noEmit`）：**PASS (GREEN)**
- `npx prettier --check`（修改文件）：**PASS**
- Headless Chrome (`http://localhost:3003/?touch=2`) 启服与 `blur` / `rc-tauri-window-shown` 事件调度验证：**PASS**
- 说明：当前 Linux 沙盒无 macOS / Rust `cargo` 环境，无法直接编译运行 macOS Tauri 原生壳，Rust 侧改动（`src-tauri/src/window_manager.rs` 为单行 `main_win.eval` 调用）请在本地 macOS 上执行 `pnpm tauri:dev:tray` 验证二次打开音频与编译通过。
