# agyDevKit #1 ~ #70 待办完成情况阶段报告
Generated: 2026-11-24T15:21:00Z
Repo: `Antigravi-ty/agyDevKit` (`a2` HEAD: `d89e03e`) | `Antigravi-ty/agyWorker` (`main` HEAD: `3939f3a`, live version `84902513`)

## 1. 总体进度统计
- 总计追踪项：**#1 ~ #70**（其中 `#63` 已由 Mark 取消，实际有效 **69 项**）
- **已完成并合入 `a2` / 部署上线：`60 项` (~87%)**
- **剩余待办 / 进行中：`9 项`**

## 2. 已完成里程碑摘要（按模块）
- **核心架构与 Monorepo (#5, #17, #25, #39 部分)**:
  - 拆分独立 `apps/emqx-live-chat` 与 `apps/p2p-remote-controller`，共享 `@agy/emqx-chat-core`
  - 统一状态目录迁至 `.agy-state/`（macOS `sun_path <= 104B` 自动回退 `/tmp/`）
  - `#25` 多端旧版协议兼容矩阵 (`normalizePocketDeckSnapshot` / `protocolCompat.test.ts`)
  - `#39` 已吸收 `apps/p2p-browser` 与 `apps/singbox-gui`（含 `#24` 50万行 / 64MB `LogRingBuffer`）
- **Live Chat & 多端协同 (#3, #7, #9, #13, #14, #18, #23, #26, #27, #29, #36, #37, #46, #54, #55, #58, #59, #60, #64, #65, #67, #68, #69)**:
  - 严格 `ackedSentAt` 逐条消息状态、0.1s 精度 `Thinking...` 等宽计时气泡
  - 智能滚动 (`Smart Scroll`) + `Quote` 140字截断预览 + `↑ Back to Quote` 锚点
  - PC 16:9 三栏布局（`>=1080px`）+ 可折叠 `ActiveTodoPanel` + 移动端 `TODO <N>` 弹窗
  - WebRTC P2P 二进制 `ArrayBuffer` OTA（`AGOT` 帧、自动降包 `16KB->8KB->4KB`、8KB JSON 回退）
  - 全局 300 条消息阈值 + DO `/channel/backup` + `backup_verified` 自动清零
- **Unified Tray & Mac 原生工具链 (#6, #8, #10, #11, #12, #15, #19, #20, #22, #41, #47, #48, #49, #50, #51, #52, #56, #57, #61, #62, #66, #70, task_bravo_021)**:
  - 5 状态原生托盘菜单（Stopped / Starting / Foreground / Background / Crashed）+ `Stop (SIGINT)` / `Force Stop`
  - 三端（`p2p-remote-controller` / `singbox-gui` / `p2p-browser`）`AGY_TRAY_MANAGED=1` 关闭隐藏到托盘 & Dock 重开
  - `git_watchdog.rs`：Git Pull / `HEAD` 变更后 45 秒健康检查与自动 `git reset --hard` 回滚看门狗
  - `Tray & Terminal` ANSI 虚拟滚动日志、`seq` 去重、`Clear View` 游标、`reattach.rs` PID 保活重连
- **Cloudflare Worker (`agyWorker`) (#16, #30, #31, #33, #34, #40 部分, #53, #58)**:
  - 多频道 `/chat/poll`、`working`/`standby` 在线状态 + `includeMySubAgents`
  - DO SQLite `/channel/backup` + `backup_verified` EMQX 广播修复（已部署 `84902513`）
  - `todoSync` 透传支持

## 3. 剩余 9 项待办
1. **#21** `Singbox GUI` 双内核无感热切换（*已派发给 Bravo `task_bravo_022`*）
2. **#35** iOS APNs Push Notification Worker + 统一收拢 `getgithubapptoken` / `agentoutputposter` 到 `agyWorker`
3. **#38** `sing-box` 无证书观测 + 凭据脱敏器 + Agent 自动生成分流/去追踪规则
4. **#39（剩余）** `UIStorybook` 吸收进 `apps/`
5. **#40（剩余）** `agyWorker` `/chat/send` `connackCode === null` 自动重试一次 + EMQX `PINGREQ` 死连接探测 + `POST /cmd` `working` 心跳
6. **#42** AI Studio `Inject Live Chat` AX 路径标定（需 Mac 真机 `scan_window`）
7. **#43** 独立 `EMQX Live Chat.app` 桌面壳
8. **#44** 局域网远程 Mac 语音听写快捷键（`CGEventPost` + 实时输入框同步）
9. **#66（剩余）** 全仓重复/未复用实现扫描（*已派发给 Alpha `task_alpha_058`*，同步排查 TODO 布局）

---
## 附录 A：已完成归档明细 (`completed.md`)
# Completed & Archived Tasks

> Moved out of active TODO (`backlog.md`) to keep the active list clean and scannable while preserving auditability.

- **[#1] PC 端提示音静音排查与修复** (`194255f`, verified on `a2`)
  - `apps/p2p-remote-controller/src/App.tsx`: added `unlockNotificationAudio()` on `pointerdown`/`touchstart`/`keydown`.
  - `apps/p2p-remote-controller/src/utils/notificationSounds.ts` (+ `.test.ts`): update `lastPlayedAtMs` only after `AudioContext.resume()` resolves.
- **[#2] OverlayFS 真实占用与 `env:check` 误报** (`194255f`, verified on `a2`)
  - `scripts/env/tmpfsLayout.mjs`: measure real Linux OverlayFS usage via `df -Pm /` (fallback `measureDirSizeMB`).
  - Purged unused `--with-rust` apt `.deb`s and `/var/lib/apt/lists/*`; `/` OverlayFS reduced from 551 MB -> 56 MB.
- **[#9] HarmonyOS 设备无法持久化 Direct Chat EMQX 配置与数据** (`11c8d90`, verified on `a2`)
  - `apps/p2p-remote-controller/src/utils/pocketDirectChatUtils.ts` (+ `.test.ts`) & `usePocketDirectChat.ts`: `IndexedDB` write-through mirror + async mount hydration for Direct Chat config, messages, and `agentBoard`.
- **[#18] Approve 审批幂等与并发防重** (`194255f`, verified on `a2`)
  - `apps/p2p-remote-controller/src/services/emqxCommandRouter.ts` (+ `.test.ts`): single-flight lock + 500-entry completed `requestId` cache in `executeEmqxDecision`.
- **[#26] HarmonyOS (HMOS) PWA 每次打开新窗口问题** (`11c8d90`, verified on `a2`)
  - `apps/p2p-remote-controller/public/manifest.json`: added `"id": "/?view=pocket-deck"` and `"launch_handler": { "client_mode": ["focus-existing", "auto"] }`.
- **[#27] LAN 局域网同步每秒 ~719 KB 流量风暴排查与优化** (`11c8d90`, verified on `a2`)
  - `lanBroadcastThrottle.ts` (+ `.test.ts`), `usePocketDeckBridge.ts`, `usePocketDeckRemoteSync.ts`: snapshot deduplication + 350ms non-urgent coalescing (approvals/chat/clipboard immediate) + 2000ms `saveCachedSnapshot` debounce.
- **[#30] `agyWorker` `/chat/poll` 支持 `channels: string[]` 多频道同时监听** (already live in `Antigravi-ty/agyWorker`)
  - `POST /chat/poll` accepts `"channels": [...]` and subscribes via `agy_poll_${agentId}_multi`. (Remaining: document in tiered `/help`.)
- **[#32] `AgentBoardBar` 启动时未显示 & 置顶/置底位置优化** (`11c8d90`, verified on `a2`)
  - `AgentBoardBar.tsx` (+ `.test.tsx`), `EmqxChatSection.tsx`, `EmqxBridgePanel.tsx`, `DebugConsoleTabs.tsx`, `App.tsx`: mounted `AgentBoardBar` at top of desktop `EmqxChatSection` and rendered compact standby bar when `board === null`.
- **[Base] `runner-core` Unix 子进程组信号清理** (`72a5a96`, verified on `a2` & Mac real-device checks #1/#2/#3)
  - `packages/runner-core/src/process_manager.rs`: `process_group(0)` on Unix child spawn and `-pgid` in `send_graceful_stop` / `send_force_kill`.
- **[#13] Command 与 Response 卡片支持展开完整参数与命令行输出** (`c7c5612` / `78f5b0d`, verified on `a2`)
  - `emqxBridge.ts`, `emqxBridgeUtils.ts`, `pocketDirectChatUtils.ts`, `emqxMessageParser.ts`, `emqxCommandRouter.ts`, `EmqxChatSection.tsx`, `PocketDeckBridgeTab.tsx`: populated `details` (up to 16k chars) and added Show/Hide details toggle.
- **[#28] `dispatch_agent_task` 在窄窗口下 Run settings 折叠导致找不到 model 按钮** (`c7c5612` / `78f5b0d`, verified on `a2`)
  - `aiStudioAxHelpers.ts` (+ `.test.ts`): auto-expand `Toggle run settings panel` and rescan when `Select Antigravity agent harness model` is missing.
- **[#29] P2P UI `All` 频道过滤与发送目标解耦** (`c7c5612` / `78f5b0d`, verified on `a2`)
  - `emqxBridgeUtils.ts` (+ `.test.ts`), `EmqxBridgePanel.tsx`, `EmqxChatSection.tsx` (+ `.test.tsx`): decoupled `sendTargetChannel` from `All` filter and added active `Send -> #channel` indicator.
- **[#3] 超长消息双击折叠 (Double-tap to collapse) & 折叠状态持久化** (`9f7c959` / `0550cea`, verified on `a2`)
  - `chatMessageCollapse.ts` (+ `.test.ts`), `EmqxChatSection.tsx` (+ `.test.tsx`), `PocketDeckBridgeTab.tsx`: >50-line double-tap/double-click collapse & expand with preview + `localStorage` persistence keyed by stable message ID.
- **[#4] 长耗时脚本 4.5 分钟 Watchdog (`scripts/env/watchdog.mjs`)** (`9f7c959` / `0550cea`, verified on `a2`)
  - `scripts/env/watchdog.mjs` (+ `.test.mjs`), `package.json`: 270s watchdog streaming stdout/stderr in real time with graceful termination and diagnostic summary (exit 124).
- **[#15] `quick_git` 增强：只读 `diff` 查看与受审批的指定文件丢弃改动** (`603cdd2` / `0c34db7`, verified on `a2`)
  - `emqxBridge.ts`, `emqxMessageParser.ts`, `quickGitUtils.ts` (+ `.test.ts`), `emqxCommandRouter.ts`, `commands.rs`, `EmqxPendingApprovalsSection.tsx` (+ `.test.tsx`), `PocketDeckBridgeTab.tsx`: read-only bounded `diff` (with `--summary`/`hasFilemodeChanges`) and human-approved `restore`/`discard` with strict path validation + red/amber destructive Approve card UI.
- **[#23] Pocket Deck 回家时从 Direct EMQX 自动探测并切回 LAN 模式** (`86ec113` / `5442c44`, verified on `a2`)
  - `lanAutoProbe.ts` (+ `.test.ts`), `useLanAutoProbe.ts`, `usePocketDeckRemoteSync.ts`, `PocketDirectChatSettingsCard.tsx` (+ `.test.tsx`), `PocketDeckView.tsx`: exponential backoff (10s->120s) + foreground/online `/api/status` probe, 15s hysteresis cooldown, and manual `Try LAN` / `Switch to LAN` button.
- **[#7] 移动端（HMOS）同步延迟排查与修复** (`64e1bc4` / `8190b4c`, verified on `a2`)
  - `lan_server.rs`: on `RecvError::Lagged(_)`, immediately push `latest_snapshot` instead of dropping.
  - `usePocketDeckRemoteSync.ts` (+ `.test.ts`): `visibilitychange`/`online` `/api/state` resync + stale WS recovery.
  - `usePocketDirectChat.ts` (+ `.test.ts`): allow all valid remote approval commands (including `quick_git`) into `directApprovals`.
- **[#16] Worker `/cmd` 被 `chat` / 自回显提前打断丢失 `resp` 的 Bug** (`ca3f467` on `agyWorker` `main`, deployed `a1760ab9`)
  - Defaulted `alsoListenChat` to `false` (opt-in only) and included `defaults.clientId` in `cmdExcludeSender`.
- **[#31] `agyWorker` `/chat/poll` 单频道固定 `clientId` 互踢 + 缺 `text` 校验 + 分层 `/help`** (`ca3f467` on `agyWorker` `main`, deployed `a1760ab9`)
  - Per-agent single-channel `clientId` (`agy_poll_${identity.agentId}_${primaryChannel}`), `400 BAD_REQUEST` structured hint when `text` is missing, and tiered `GET /help?topic=cmd|chat|agents|backup|all`.
- **[#33] `agyWorker` `/chat/poll` 默认 `excludeSender` 误屏蔽 Chief (`agy_cloud_agent_v1`) Bug** (`ca3f467` on `agyWorker` `main`, deployed `a1760ab9`)
  - Non-admin default `excludeSender` now only excludes `${identity.agentId} (${identity.role})`.
- **[#34] `agyWorker` `wrangler.toml` `keep_vars = true`** (verified already present & deployed with `--keep-vars`).
- **[DO Backup] `ChannelCoordinatorDO` SQLite `channel_checkpoints` (`GET/POST /channel/backup`)** (`ca3f467` on `agyWorker` `main`, deployed `a1760ab9`)
  - `INSERT OR REPLACE` by `(channel, kind)`, 512 KB row cap, SHA-256 verification, selective metadata listing.
- **[#5] 拆分独立 `EMQX Live Chat` 与 `p2p-remote-controller`** (`c77847b` + `be8b546`, verified on `a2`)
  - `packages/emqx-chat-core` (`@agy/emqx-chat-core`): pure-TS shared types, E2EE, message IDs, agentBoard, pollCountdown, chatMessageCollapse.
  - `apps/emqx-live-chat`: standalone React+Vite PWA (zero Tauri/AX deps, `manifest.json` `focus-existing`, IndexedDB+localStorage persistence, responsive desktop 16:9 2-column & mobile compact layout; `tsc --noEmit`, 4/4 tests, `vite build` 59.57 kB gzip).
- **[#10] `ENRICHED_PATH` in `process_manager.rs`** (verified already implemented at `packages/runner-core/src/process_manager.rs:522`).
- **[QoL Audio] `task_alpha_017` Channel-aware notification sound routing & WebAudio previews** (`39c6e3f` on `a2`)
- **[#46] Multi-device `approving` spinner sync** (`35f8315` on `a2`)
- **[#36] PC 16:9 3-column layout + Active TODO live sync (`todo_sync` over EMQX)** (`35f8315` on `a2`)
- **[#50] White-screen recovery (`ErrorBoundary` + `vite:preloadError` auto-reload)** (`af6242b` on `a2`)
- **[#49 Frontend] Suppress `[ax-helper]` / `[ax-bridge]` noise toggle in Debug Console** (`af6242b` on `a2`)
- **[#37] Mobile `quick_git` branch graph visualization (`QuickGitGraphView`)** (`af6242b` on `a2`)
- **[#47] Safari `open_safari_windows` immediate managed registration, post-init readiness cache, & prompt submission verification** (`32e1471` on `a2`)
- **[#17] Unified state directory migration (`~/.agy-devkit` -> `.agy-state/`) & macOS `sun_path <= 104B` fallback** (`e2b45fa` on `a2`)
- **[#54] Smart Scroll (preserve reading position + `↓ Bottom` button) & per-message `Quote` button** (`3be8d12` on `a2`)
- **[#20 & #49 Tray Frontend] `Unified Tray` ANSI color terminal log rendering, line numbers, virtual scroll & Copy Logs button** (`4f965f7` on `a2`)
- **[#55] Chat 3-state delivery/thinking spinner flow & 1h offline threshold** (`7e44f34` on `a2`)
- **[#57] `Unified Tray` log `seq` deduplication & `Clear View` `clearedAfterSeq` cursor** (`6633482` on `a2`)
- **[#41 Frontend] `Unified Tray` `Runner Apps Dir` onboarding & EMQX shortcuts UI polish** (`83a4f1a` on `a2`)
- **[#58 Tracker & Auto-Kind] Auto-detect `[in progress]`/`[Resuming from compaction]` kind & 200/300-msg backup threshold tracker** (`9e8f3d2` on `a2`)
- **[4-State Presence & Offline Retention]** (`e7302d2` on `a2`): 4-state `Working (Not listening)` support + never auto-hide `>1h` offline agents.
- **[CDP Layout Audit]** (`2f9a681` on `a2`): Headless Chrome CDP zero-screenshot geometry audit script `scripts/env/verify-layout.mjs` (0 overlap/overflow at `375x812` & `1920x1080`).
- **[#59] Quote 140-char truncation + normal scroll-to-bottom on send + `↑ Back to Quote` anchor button** (`74f46c4` on `a2`)
- **[#58 Frontend/Core] Global 300-msg `backupEventId` + `unbackedCount` + `backup_verified` reset** (`task_alpha_033` merged to `a2`)
- **[#60] `Thinking...` / `Working` 0.1s precision elapsed timer (`0m 01.3s`, 100ms refresh)** (`task_alpha_034` merged to `a2`)
- **[#62] `--with-rust` tmpfs `libc6-dev` / `linux-libc-dev` / Tauri sysroot headers** (`task_alpha_035` merged to `a2`; `p2p-remote-controller` `cargo check` passes in Linux sandbox)
- **[Parser Bugfix] Restored 8 dropped `RemoteCommandRequest` fields (`x`, `y`, `width`, `height`, `targetId`, `label`, `includeHidden`, `model`) + `params`/`meta` passthrough** (`task_alpha_036` merged to `a2`)
- **[#25] Multi-device legacy protocol compatibility & `normalizePocketDeckSnapshot` across all LAN/WS/HTTP/Tauri paths** (`task_alpha_037` merged to `a2`)
- **[#64] `INBOUND · thinking` bubble (`SF Mono` timer), centered `.spin` fix, removed bottom `Replied` bar & fixed stuck `OUTBOUND` pending spinner** (`task_alpha_038` merged to `a2`)
- **[#65] Mobile `AgentBoardBar` `TODO <N>` uncompleted count badge + `TODOs` modal drawer + `todoSnapshots` LAN/EMQX sync** (`task_alpha_039` merged to `a2`)
- **[#68 & #67] HMOS landscape `safe-area-inset-right` + `@media (hover: hover)` touch hover fix & remote Approve desktop focus-steal guard** (`task_alpha_040` merged to `a2`)

- **task_bravo_003** (`a9c6cfd`, `2026-11-24T11:44:02Z`): `Unified Tray` `p2p:dev:watch` + `devWatch.ts` + `#56` OTA `ENRICHED_PATH` & 20-log retention (merged `a2-bravo` `7142c5d` -> `a2`).
- **task_alpha_041** (`226ead3`, `2026-11-24T11:45:13Z`): `#69` strict `ackedSentAt` independent per-message spinner + `get_bridge_diagnostics` read-only `/cmd` (merged `a2-alpha` `e0fe221` -> `a2`).
- **task_bravo_004** (`c1bff9b` merged into `a2`, `2026-11-24T11:47:08Z`): `#51` Dock/window close icon revert fix, `#52` `TrayIconState` + menu dedup, `#70` `TerminalLogViewer` `requestAnimationFrame` scrollHeight + 8px spacer, `#61` `Unified Tray` 1-click `Git Pull` & `Rollback (HEAD@{1})`.
- **task_alpha_042** (`7e4348a` merged into `a2`, `2026-11-24T11:49:09Z`): Fixed silent drop of non-string DataChannel frames (`decodeDataChannelPayload`) + `dc.readyState === "open"` fallback in `useCellularP2pHost.ts`, exported `addConsoleLog` and instrumented `[P2P-OTA]`, `[P2P-DC]`, `[P2P-RPC]`, `[P2P-Relay]`, `[P2P-WebRTC]`.
- **task_bravo_005** (`11ca4ad9` merged into `a2`, `2026-11-24T11:52:20Z`): `#66` `open_safari_windows` reusing idle `Open New` windows + dedup scan; `#48` `CreateInteraction` -> `CreatePrompt` -> `GetInteractionStream` capture & `pageUrl` upgrade.
- **task_alpha_043** (`fad331c` merged into `a2`, `2026-11-24T11:53:13Z`): `useNetworkRecorderPolling.ts` exponential backoff cooldown (`15s->120s`) on AX RPC timeout + skip background poll when no non-selected windows have `recording` enabled + unit tests.
- **task_alpha_044** (`9b10cea` merged into `a2`, `2026-11-24T11:57:28Z`): `#54` Smart Scroll + `Jump to latest` pill + `#59` `Quote` 140-char truncation & `↑ Back to Quote` anchor in `EmqxChatSection.tsx` + interactive jsdom tests.
- **task_bravo_006** (`54c19e4` merged into `a2`, `2026-11-24T11:57:59Z`): `#6` AGY Terminal auto-reporting `.app` parent dir to `~/.agy-state/apps_dir.txt`; `#19` dynamic `Regular`/`Accessory` Dock icon policy; `#11` `Unified Tray` `Tray & Terminal` log/status tab.
- **task_bravo_007** (`d4b4bf2` merged into `a2`, `2026-11-24T12:00:52Z`): `#49` Copy Logs & `suppressAxNoise` verified; `#50` `ErrorBoundary` + `vite:preloadError` verified; `#57` strict `seq` dedup + `clearedAfterSeq` `minSeq` cursor extended to `Tray & Terminal` system logs tab.
- **task_alpha_045** (`71509b9` merged into `a2`, `2026-11-24T12:01:38Z`): WebRTC DataChannel `16KB` chunking/reassembly (`cellularDataChannelChunking.ts`) + `<= 48 KB` cellular snapshot cap for legacy `3f1558d` compatibility + `pc.sctp?.maxMessageSize` logging + 158/158 vitest passing.
- **task_alpha_046** (`9add55a` merged into `a2`, `2026-11-24T12:06:24Z`): `#46` multi-client `approving` sync + `pruneStaleApprovals` clearing stuck spinners on WS/P2P/EMQX reconnect or PC `commitId` change (`AGENTS.md` untouched).
- **task_bravo_008** (`bdb96b1` merged into `a2`, `2026-11-24T12:06:42Z`): Fixed missing `agentStatus` prop wiring in `EmqxBridgePanel` & `PocketDeckView`, added `[AgentStatus]` console debug logs, and added PC + HMOS `INBOUND · thinking` jsdom simulation tests (160/160 passing).
- **task_bravo_009** (`8e85add` merged into `a2`, `2026-11-24T12:08:34Z`): `#12` Unified Tray native menu tri-state indicators + `Stop (SIGINT)` / `Force Stop` submenus + `CloseRequested` Dock visibility; `#20` ANSI SGR/CSI/OSC span rendering & virtualized scrolling verified.
- **task_alpha_047** (`d72ec9b` merged into `a2`, `2026-11-24T12:09:38Z`): `#36` PC 16:9 widescreen `>=1280px` 3-column layout + independent `ActiveTodoPanel` live `todo_sync` column (161/161 tests passing).
- **task_bravo_010** (`9e864df` merged into `a2`, `2026-11-24T12:10:37Z`): `#37` `QuickGitGraphView.tsx` SVG swimlane connectors (`merge_right`/`merge_left`), commit nodes, `HEAD`/`origin/*` ref pills, and `QuickGitGraphView.test.tsx` (163/163 passing).
- **task_bravo_011** (`5c68c04` merged into `a2`, `2026-11-24T12:15:56Z`): `#22` Unified Tray `reattach.rs` PID/preset persistence in `~/.agy-state/`, alive-PID re-attachment, IPC socket retry, and `cargo test` + `cargo check` passing.
- **task_alpha_048** (`015169f` merged into `a2`, `2026-11-24T12:16:58Z`): `#41` Unified Tray Shortcuts (`PocketDeck`, `EMQX CN`, `EMQX Global`) + `tray_pick_apps_dir` folder picker; `#8` `+ New Tab` custom command launcher (16/16 vitest + `cargo check` passing).
- **task_bravo_012** (`09459a7` merged into `a2`, `2026-11-24T12:18:41Z`): Mobile lock-screen P2P disconnect detection (`pc.oniceconnectionstatechange`, `pagehide` suspect marking, `shouldReconnectOnForegroundResume()` on `visibilitychange(visible)`/`pageshow`/`online`, 5 unit tests, 170/170 passing).
- **task_alpha_049** (`9e25674` merged into `a2` at `85fde67`): Removed `.badge`/`.timeline-badge` pill background from `INBOUND` & `Thinking...` indicators and updated monospace font stack (`"SF Mono", "JetBrains Mono", ui-monospace, ...`).
- **task_bravo_013** (`d1da3c8` merged into `a2` at `754ede7`): Capped `OTA_CHUNK_SIZE = 8192` in `p2pOtaService.ts` for legacy WebRTC compatibility + added `recentP2pLogs` in `get_bridge_diagnostics` (172/172 passing).
- **task_alpha_050** (`db58322` merged into `a2` at `8864d88`): Telegram-style dismissible Quote preview bar above composer, isolated inner-container `↑ Back to Quote` scroll (`scrollContainerToChildElement`), and `pending` reply stability on consecutive sends (`@agy/emqx-chat-core` 26/26, `p2p-remote-controller` 173/173, `emqx-live-chat` 13/13).
- **task_bravo_014** (`ce224be` merged into `a2` at `d37bb7c`): WebRTC P2P binary `ArrayBuffer` OTA chunking (`AGOT` frame codec, `pc.sctp?.maxMessageSize`-aware `16KB` initial chunk, `dc.send`/ACK/`bufferedAmount` failure detection, `16KB -> 8KB -> 4KB` half-packet retry, and automatic `8KB` JSON `ota_chunk` fallback; 48 files / 178 tests passing).
- **task_alpha_051** (`370cc19` merged into `a2` at `4ed1db3`): Lowered 3-column `.main-grid` breakpoint to `1080px` and added collapsible `ActiveTodoPanel` (44px narrow rail on `>=1080px` / compact 1-line bar on `<1080px`, auto-collapsing when 0 TODOs, auto-expanding when items arrive; 48 files / 179 tests passing).
- **task_bravo_015** (`b6c5858` merged into `a2` at `3228ad8`): Added paired DataChannel E2E binary OTA tests (`p2pOtaE2e.test.ts`: 16KB AGOT chunking, 16KB->8KB->4KB half-packet retry, 8KB JSON fallback), extended `pocket_footer` horizontal safe-area insets on mobile landscape, and polished `QuickGitGraphView` narrow-screen wrapping (49 files / 181 tests passing).
- **task_alpha_052** (`1d4728c` merged into `a2` at `986d77d`): Made `emqx-live-chat` `TodoSyncPanel` collapsible (auto-collapse on 0 items, 48px narrow rail on `>=1024px`) and verified `INBOUND · thinking` clean typography parity (`tsc`, 13/13 tests, `vite build` passing).
- **task_bravo_016** (`d844b46` merged into `a2` at `f135294`): Added `Hide [AgentStatus]` (default checked) and `Hide [P2P-*]` checkboxes in `DebugConsoleTabs.tsx` with `localStorage` persistence in `logExport.ts` (49 files / 183 tests passing).
- **task_alpha_054** (`64f16a0` merged into `a2` at `63b2bdd`): `#39` Absorbed `Antigravi-ty/P2PBrowser` (`a1`) into `apps/p2p-browser` (Vite `1420`, pnpm workspace, `tsc`, `vite build`, and `cargo check` passing).
- **task_bravo_017** (`1a6f7bf` merged into `a2` at `415ccec`): Replaced `🟢`/`🟡`/`🔴`/`⚪` emojis in `apps/unified-tray/src-tauri/src/main.rs:544-548` with aligned non-emoji symbols (`●`/`◐`/`×`/`○`), and `#39` absorbed `Antigravi-ty/SingboxGUIV0` (`main`) into `apps/singbox-gui` (reassigned Vite/Tauri dev port `1422 -> 1424`, `tsc`, `vite build`, and `cargo check` passing).
- **task_bravo_018** (`1b5244c` merged into `a2` at `bafe826`): `#24` Implemented `LogRingBuffer` (up to 500,000 lines / 64 MB cap) in `apps/singbox-gui` Rust backend (`singbox.rs`) + virtualized `ConsoleRealtimeLogPanel` with retention selector (5/5 Rust tests, `cargo check`, `tsc`, and `vite build` passing).
- **task_alpha_055+056** (`5d64fdd` merged into `a2` at `0cdd05e`): Kept `"SF Mono", "JetBrains Mono", ui-monospace` + `tabular-nums` on `thinking` timer span while reusing normal inbound bubble sizing/padding, `var(--text-muted)` color, and `<Loader2 size="1em" />` across `EmqxChatSection.tsx`, `PocketDeckBridgeTab.tsx`, and `ChatStreamPanel.tsx` (`p2p-remote-controller` 183/183, `emqx-live-chat` 13/13).
- **task_bravo_019+020** (`438b974` merged into `a2` at `3f1cfa4`): `#12` Implemented 5-state `Unified Tray` menu (`  ` Stopped, `-` Starting/Stopping, `✓` Running foreground, `○` Running background -> unhide & focus, `×` Crashed) + `Stop (SIGINT)` / `Force Stop` submenus + macOS `CloseRequested` hide-on-close in `p2p-remote-controller` (`unified-tray` 6/6 Rust tests + `cargo check` passing).
- **task_alpha_057** (`c08870a` merged into `a2` at `b5e6c53`): Added `AGY_TRAY_MANAGED=1` `CloseRequested` hide + `ExitRequested` keep-alive + macOS `RunEvent::Reopen` restore across `p2p-remote-controller`, `singbox-gui`, and `p2p-browser` (all 3 `cargo check`s passing).

### task_bravo_021 (`9840c4c` -> `d89e03e`)
- Unified Tray (`apps/unified-tray`): added `git_watchdog.rs` + `main.rs` integration for `last_good_commit.txt`, Git Pull / `HEAD` change detection, 45s health-check grace window, and automatic `git reset --hard <prev_good_commit>` rollback on crash or Rust/Vite compile failure.

