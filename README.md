# RocketCompanion vNext 架构核查、问题修复与重点评估报告

> **基线分支**：`origin/integration/vnext-core`
> **已提交 Verified Bot Commit**：[`af6c9f9`](https://github.com/Antigravi-ty/RocketCompanion/commit/af6c9f9095f34f0ce20d015dbca0af2fec414490)（`signature.state = VALID`）
> **验证结果**：`pnpm run check`（`radar` + `tsc --noEmit`）全绿（1457/1457 TS）；`RC_SUITE_WATCHDOG_MS=280000 pnpm run test:multiplayer-suite` **30/30 全部通过**。

---

## 一、本次已落地的修复与接线补全

### 1. Tauri 关窗后无法重开 & `Main window handle not found`（红叉闭窗根因修复）
- **根因定位**：
  1. `src-tauri/tauri.conf.json` 中的 `main` 与 `configurator` 均属于应用启动时创建的静态窗口。
  2. 点击 **Configurator** 或 **Main** 左上角红叉关闭窗口时，Tauri 默认的 `WindowEvent::CloseRequested` 会直接 **销毁（Destroy）** 该窗口实例；而开启 `tray_only_mode` 时进程仍驻留在系统托盘。
  3. 之后从托盘或 Configurator 再次点击 Launch / 打开 Configurator 时，`app.get_webview_window("main" | "configurator")` 返回 `None`，直接抛出 `"Main window handle not found"` / `"Configurator window handle not found"`。
  4. `Layer1TauriConfiguratorRecipe.tsx` 的 `handleSaveOnly` / `handleLaunchGame` / `handleResetDefault` 只有 `try ... finally` 没有 `catch`，导致异常变成未捕获的 `Unhandled Promise Rejection`，且托盘侧 `let _ = open_configurator_window(app)` 静默吞掉了错误，Terminal 无任何日志。
- **已修复内容**：
  - `src-tauri/src/lib.rs`：在 `.on_window_event` 拦截 `configurator` 的 `WindowEvent::CloseRequested`，调用 `api.prevent_close()` + `window.hide()` + `update_macos_activation_policy(..., Some("configurator"))`，确保红叉关闭 Configurator 仅隐藏不销毁。
  - `src-tauri/src/window_manager.rs`：新增 `ensure_webview_window(app, label)`，当 `main`（释放 WebGL/GPU/音频/隧道后被销毁）或 `configurator` 的句柄为 `None` 时，自动从 `app.config().app.windows` 配置通过 `WebviewWindowBuilder::from_config` 重建窗口。
  - `src-tauri/src/tray_manager.rs`：托盘菜单触发 `launch_main_window_with_config` / `open_configurator_window` 失败时通过 `log::error!` 与 `eprintln!` 显式输出错误。
  - `src/ui-core/recipes/Layer1TauriConfiguratorRecipe.tsx`：为 `handleSaveOnly`、`handleLaunchGame`、`handleResetDefault` 补齐 `catch (err)`，`console.error` 打印并通过 `setStatusMessage` 展示给用户。
- **注意**：需在本地重新编译一次 Rust（如 `pnpm tauri:dev:tray`）使 Tauri 侧修改生效。

### 2. Exhibition 6 人用户名同步 & 切模式通道 2（`VNextRosterManifestCodec`）全链路打通
- **Deep Scan 发现的断链点**：
  1. `AuthoritativeServerWorker.ts` 仅发射了通道 1（`VNextWorldFrameCodec`），但帧头硬编码 `metadataHash: 0`，且从未编码/下发通道 2（`VNextRosterManifestCodec`）。
  2. `reconcilerInboundPacketParser.ts` 已能解码通道 2 并触发 `ctx.onRosterUpdate({ type: 'vnext_roster_manifest', manifest, metadataHash })`，但 `OnlineSessionCoordinator.ts` 的 `onRosterUpdate` 只认 `msg.roster`（旧 JSON 结构），直接忽略了 `vnext_roster_manifest`。
  3. `ExperimentalMultiBotsRosterController` 生成了 `MatchExhibitionRosterManifest`，但从未转换成 `VNextRosterManifest` 推给 `AuthoritativeServerHost` -> `AuthoritativeServerWorker`。
- **已修复内容**：
  - `src/network/server/WorkerAuthorityBus.ts`：新增 `setRosterManifest`、`getMetadataHash`、`getRosterManifestBytes`、`sendRosterManifestToSession`、`broadcastRosterManifestToSessions`，使用 `VNextRosterManifestCodec.encode` 缓存通道 2 二进制包并同步更新 4 字节 `metadataHash`。
  - `src/network/server/workerActivityAndInputRouter.ts` & `AuthoritativeServerHost.ts`：新增 `set_roster_manifest` 命令与 `pushRosterManifest(...)` API，收到后立即编码并广播通道 2 到所有非 Bot 客户端会话。
  - `src/network/server/AuthoritativeServerWorker.ts`：`tick()` 中所有通道 1 帧头改为携带真实的 `authorityBus.getMetadataHash()`；并在 `registerWorkerChannel` 新客户端连入时立即补发一次缓存的通道 2 Manifest。
  - `src/game/modes/experimental-multi-bots/ExperimentalMultiBotsRosterController.ts` & `src/game/runtime/OnlineMatchControlBridge.ts`：在 Exhibition Roster 更新及广播期间 `switchRoomActivity` 切换模式时，自动通过 `toVNextRosterManifest` 推送通道 2 清单。
  - `src/game/runtime/OnlineSessionCoordinator.ts`、`OnlineRosterSyncCoordinator.ts`、`onlineSessionEnvironmentPipeline.ts`：客户端收到 `vnext_roster_manifest` 后，统一经 `fromVNextRosterManifest` 刷新 3D 车辆/涂装/幽灵车透明度、`projectRosterCarEntries`（HUD 1~6 + 3D NameTag）以及 `useExperimentalMultiBotsStore.spawnedCars`（Exhibition 6 人用户名、队别、Bot 类型与配色）。
- **后续可继续优化点**：目前通道 2 在“Roster 变更 / 切模式 / 新通道注册”时主动推送；若未来遇到 UDP/弱网单包丢失，可在 `reconcilerInboundPacketParser.ts` 发现 `frame.metadataHash !== 0 && frame.metadataHash !== localManifestHash` 时增加一次 `REQUEST_ROSTER_MANIFEST` 异步补拉请求，实现彻底的弱网自愈。

### 3. 全局静默报错检测（`radar` 硬性棘轮 `SilentCatches`）
- **现状扫描**：全库 `src/` 共扫描出 **231 处** 空 `catch {}` 或静默 `.catch(() => {})`（主要集中在音频播放/WebRTC/UI 状态存储等历史代码）。
- **Radar 已落地**：在 `scripts/radar/radarArchDebtGuards.ts` 与 `scripts/radar.ts` 新增 `SilentCatches` 棘轮守卫（`HARD_SILENT_CATCH_LIMIT = 231`），并在 `pnpm run radar:audit` 中输出前 20 处具体 `file:line` 与类型（`empty-catch` / `silent-promise-catch`）。任何新增的静默 `catch` 都会在 `pnpm run check` 阶段直接被拦截拒绝提交。
- **关于“雷达 90% + 人工/AST 复查 10%”的判断**：
  - 静态正则/AST 扫描可以 **100% 拦截** 空 `catch {}`、`catch (_) {}`、`.catch(() => {})` / `.catch(() => null)`。
  - 剩余 ~10% 属于“语义级静默吞错”（例如 `catch (err) { return fallback; }` 或 `if/else if/else` 兜底 `default` 分支里只做了状态重置却没有 `console.warn/error`）。建议后续通过 ESLint/TS AST 规则要求所有 `catch` 块必须包含 `console.warn/error`、`throw` 或显式 `// intentional-fallback:` 注释豁免，即可逐步将 `231` 棘轮压降到 `0`。

---

## 二、重点评估 1：Trajectory Predictor 在广播模式下的可行性与 Host 暂停触发方案

### 1. 为什么当前会在 Replay / Rewind / 广播观战时疯狂刷 `ALPHA / BRAVO` 报错？
- `GameRenderSubsystem.ts` 每帧无条件调用 `r.ballTrajectoryPredictor.update(curTick, r.arena?.ball?.position, winWidth, winHeight, r.physics)`。
- 在 Replay 回放（尤其是 Rewind 倒放）或客户端 Pure CSR 观战模式下，画面球的位置来自插值/回放缓冲，而 `r.physics`（本地 WASM 实例）并未同步步进或 `tick` 发生跳变/倒流。
- `BallTrajectorySamplingController` 将本地 WASM `BallPredTracker` 的预测轨迹与当前画面球位/速度对比，产生巨大 `posErrorUu` / `velErrorUuPerSec`，触发 `BallTrajectoryAnomalyDetector` 的 Alpha/Bravo Ping-Pong 缓冲锁存，进而通过 `ballTrajectoryAnomalyUiBridge.ts` 在右下角弹出 `[ALPHA · ...]` / `[BRAVO · ...]` 警告通知。
- **本次已做止血**：在 `GameRenderSubsystem.ts` 中检测 `matchMode === 'replay-viewer'` 或广播观战/Replay 活动时，自动调用 `acquireTemporarySuppression('replay-or-broadcast-spectate')`，彻底消除回放/倒放/观战时的 `ALPHA/BRAVO` 刷屏。

### 2. 广播模式下“Host 暂停手动触发”的推荐架构（Coaching 评估专用）
完全可行，且非常适合 Coaching 复盘场景。建议采用 **“Host 暂停态按需触发 + 客户端本地确定性推演”** 方案，既保留每帧推进 5 帧的丝滑动态轨迹动画，又零带宽开销、零状态污染：
1. **触发前置护栏**：仅当房间处于 **Paused 状态** 且由 **Host 手动触发**（例如快捷键或 Coaching 工具栏按钮）时激活。一旦 Host 点击 Resume / Seek / Rewind，立即自动关闭并 `clear()`。
2. **协议同步（极简，走通道 1/Workspace 或通道 3 Signal）**：
   - Host 广播 `{ trajectoryPreview: { enabled: true, anchorTick, ballState18 } }`（或在暂停态直接利用客户端当前已对齐的权威快照 `ball.pos / ball.vel / ball.angVel`）。
   - 带宽消耗几乎为 0（仅触发时发 1 次信令）。
3. **客户端本地推演与渲染**：
   - 客户端收到触发信号后，使用 `BallTrajectoryPredictor.setStaticPreviewTrajectory(sim, ...)`（该 API 在 `StaticTrainingArenaController.ts:277` 已有现成参考实现！），将本地空闲 WASM `physics` 仅作为**只读数学计算器**，从当前暂停点的球状态一次性向前推演 120~600 ticks，然后按每帧推进 5 frames 的节奏播放管线增长动效。
   - **关键隔离**：推演期间必须禁用 `BallTrajectoryAnomalyDetector`（`staticPreview` 模式本身不走 live anomaly 检测），避免任何 `ALPHA/BRAVO` 误报。

---

## 三、重点评估 2：GPU 资源消耗遥测（Telemetry）与可导出诊断报告方案

### 1. 现状盘点与瓶颈盲区
目前 `src/ui-core/stores/perfTelemetryStore.ts` 与 `scenarioWebGlAndPacingPipeline.ts` 仅读取 `renderer.info.render`（总 `calls` / `triangles`）和 `renderer.info.memory`（总 `geometries` / `textures` / `programs`）。这只能回答“总开销高”，无法回答用户关心的核心诊断问题：
> **“到底是场地（Stadium/Driftwoods）占用了资源，还是车模（GLB）、Boost 粒子、阴影贴图还是后处理（BoostBloom / EffectComposer）导致了高功耗？”**

此外，实时面板在操作比赛时根本来不及看，必须支持 **后台采样 + 离线导出报告**。

### 2. 推荐架构：`GpuTelemetryProfiler`（子系统分组耗时 + 显存归因 + 一键导出）
建议在 `src/game/diagnostics/` 下构建一个轻量、零侵入的 `GpuTelemetryProfiler`，分三层采集：

1. **按业务子系统（Pass / Object Group）归因 Draw Calls 与 Triangles**：
   - 利用 Three.js `Object3D.onBeforeRender` / `onAfterRender`（或按 `Scene` 顶层分组：`arenaStadiumGroup`、`vehiclesGroup`、`boostParticlesGroup`、`ballAndTrajectoryGroup`、`skyboxGroup`）以及 `EffectComposer` 的各个 Pass（`RenderPass`、`BoostBloom`、`SupersonicSpeedLinesPass`、`ShadowMap`）。
   - 在采样窗口内记录 `renderer.info.render.calls` 与 `triangles` 的**差值（delta）**，即可精确得出每个子系统每帧贡献了多少 Draw Calls 和多少万个三角形。
2. **WebGL2 GPU 真实耗时测量（`EXT_disjoint_timer_query_webgl2`）**：
   - CPU 侧 `performance.now()` 只能测 JS 提交耗时，无法反映 GPU 光栅化/Fragment Shader 压力（比如透明 Boost 粒子 Overdraw 或 Bloom 高斯模糊）。
   - 在支持 `EXT_disjoint_timer_query_webgl2` 的环境（macOS WKWebView / Chrome / Angle）下，对每个子系统/RenderPass 包裹 `TIME_ELAPSED_EXT` 查询，异步回收结果（延迟 2-3 帧读取，避免 `gl.finish()` 卡顿主线程），得到各 Pass 的真实 **GPU ms 占比**（同时在不支持该扩展的驱动上自动降级为 DrawCalls + Triangles + PixelFill 估算）。
3. **显存与资产预算归因（Geometry / Texture / RenderTarget）**：
   - 遍历 `scene` 按子系统统计去重后的 `BufferGeometry` 字节数与 `Texture` 分辨率/格式估算显存（MB），并单独列出 Top 10 最大纹理与 Top 10 最高面数 Mesh（例如可直接定位 `Driftwoods` 场地网格或某个高模车身 GLB）。
4. **后台录制与一键导出报告（JSON + Markdown）**：
   - 提供 `Start Capture (10s / 30s / Manual Stop)` 开关，采样期间仅写环形缓冲，不影响帧率；结束后自动生成并下载 `rc-gpu-telemetry-<timestamp>.md`（人类可读的排名与优化建议）和 `.json`（供脚本/Agent 分析）。

---

## 四、后续建议下一步
1. **本地验证 Tauri 窗口修复**：拉取 `af6c9f9` 后执行一次 `pnpm tauri:dev:tray` 重编 Rust，测试“红叉关闭 Configurator -> 托盘重开”及“红叉关闭主窗口 -> Configurator 重新 Launch”。
2. **联机实测 Exhibition 6 人用户名**：Host 开启 Exhibition 并广播，Client 进房后确认 6 辆车的 HUD 用户名、3D NameTag、车身颜色与切模式后 Roster 是否全量同步。
3. **排期落地**：按上述评估方案先后排期实现 **GpuTelemetryProfiler 导出报告**（先定位功耗大户）与 **Host 暂停触发 Trajectory Preview**。
