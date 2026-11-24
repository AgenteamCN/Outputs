=== completed.md ===
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

=== backlog.md ===
# Active Task Backlog (TODO)

> Completed items have been archived to `/workspace/.chat/completed.md`.

## High Priority / Architecture
- **[#14] Worker 送达回执 & Agent Thinking 状态指示** (`agyWorker` + 前端)

## TS / Web / Scripts (Offloadable to Sub-Agents)
- **[#25] 多端版本协议兼容与双线验证** (iOS `3f1558d` / Pad `b8195354` / PC 最新)

## Rust / Mac Native (Requires `--with-rust` or Mac Local Build)
- **[#6] `AGY Terminal` 自动上报自身 `.app` 目录给 `Unified Tray` (`apps_dir` 零配置)**
- **[#8] `Unified Tray` 增加自定义命令 / 交互式 New Tab** (仅限本地 UI 手动新建)
- **[#11] `Unified Tray` 增加自身及 `AGY Terminal` 日志/状态监听 Tab**
- **[#12] `Unified Tray` 顶部原生托盘菜单三态指示 & 点击交互优化** (普通 `MenuItem` + 彩色状态图标 + `Stop (SIGINT) ▸` / `Force Stop ▸` + 红叉隐藏窗口)
- **[#17] 统一状态目录迁移：从 `~/.agy-devkit` 迁至 Terminal `.app` 同级 `.agy-state/`** (macOS `sun_path` <= 104B fallback `/tmp/` + `>= 90B` 警告)
- **[#19] `AGY Terminal` 首次拉起 `Unified Tray` 时 Dock 图标动态显隐**
- **[#20] `Unified Tray` 终端日志渲染优化** (ANSI 颜色 + 行号 + 虚拟滚动)
- **[#21] `Singbox GUI` 双内核无感热切换 (Dual-Core Graceful Handover)**
- **[#22] `Unified Tray` 无感热重启 / 升级不掉线（Runner 保活重连）**
- **[#24] `Singbox GUI` 超大日志保留（50 万行 / 按字节尺寸）与网络诊断安全接管**

## Worker / Multi-Client / UX Additions
- **[#35] iOS APNs Push Notification Worker** (及统一收拢 `getgithubapptoken` / `agentoutputposter` / `push` 到 `agyWorker` monorepo)
- **[#36] PC 16:9 宽屏左中右多栏布局 + 独立 Active TODO 实时同步栏** (`todo_sync` over EMQX)
- **[#37] 移动端 `quick_git` 分支连线图可视化** (将 `log_graph` 渲染为手机友好的分支/Merge 泳道图)
- **[#38] `sing-box` 无证书观测 + 凭据脱敏器 + Agent 自动生成分流/去追踪规则** (MITM/Rewrite 延后)

## Recovered from `AgenteamCN/Outputs` (`84ddb0f`)
- **[#39] Monorepo 吸收剩余仓库**: `P2PBrowser` (`apps/p2p-browser`, Vite `1424`)、`SingboxGUIV0` (`apps/singbox-gui`, Vite `1425`)、`UIStorybook` (`apps/`)
- **[#40] `agyWorker` 可靠性与安全增强**: `/chat/send` `pubackedCount === 0` 假阳性修复 (`820b2e1`) + EMQX 主动 `PINGREQ` 死连接探测 (`Close 1005`) + GitHub Token 代理与子 Agent 级联吊销 (`2dc9d7f`)
- **[#41] `Unified Tray` 迁入旧托盘 `Shortcuts` (EMQX CN/Global) & `Runner Apps Dir` 引导抛光**
- **[#42] AI Studio `Inject Live Chat` AX 路径标定** (需 Mac 真机 `scan_window`)
- **[#43] 独立 `EMQX Live Chat.app` 桌面壳** (PWA 已在 `apps/emqx-live-chat`，Tray 已预留 `.app` 位)
- **[#44] 局域网远程 Mac 语音听写快捷键** (`CGEventPost` + 实时输入框同步)

## Newly Reported Items (Live Chat 2026-11-24)
- **[#46] Approve 多端协同旋转状态同步 (`approving`)**: 移动端点击 Approve 且 PC 端收到后，PC 端广播 `approving` 状态，使所有已连接的 Pocket Deck 客户端同步显示旋转加载指示器。
- **[#47] `open_safari_windows` 隐形窗口安全/可观测修复 + `dispatch_aistudio_task` 静默空操作修复**: `open_safari_windows` 打开的窗口必须立即注册到 `managed` 列表（无需手动开启 `Include All Safari Windows`）；`dispatch_aistudio_task` 在返回 `ok` 前必须校验提示词已实际粘贴并发送。
- **[#48] `agy-preview-instant` 严重 Bug 修复与重命名**: 待 Mark 补充具体 Bug 详情与新名称。
- **[#49] `Unified Tray` 日志区增加 Copy 按钮 + `p2p-remote-controller` 增加屏蔽 `ax helper` 噪声日志开关**
- **[#50] `p2p-remote-controller` 顶层 React ErrorBoundary + `vite:preloadError` 自动刷新兜底** (防止 `quick_git pull_ff` 大批量热更新时白屏)
- **[#51] `Unified Tray` 关闭/隐藏窗口时图标退化成默认执行图标** (展开时正常，关闭/隐藏时丢失自定义 icon)
- **[#52] `Unified Tray` 托盘图标时有时无 / 自动消失** (`NSStatusItem` / `TrayIcon` 生命周期排查)
- **[#53] `agyWorker` `agent_presence` 增加 `working` 状态 + `/chat/poll` `includeMySubAgents: true` 自动展开子频道**
- **[#54] 聊天阅读位置保持 (Smart Scroll) + 消息头 `Quote` 引用追加按钮** (往上翻历史回复时不强行滚到底部，不与系统长按选字冲突)
- **[#55] 聊天三态动效抛光**：去掉状态条冗余 `#channel · chief`；用户消息在送达前最后一行转圈 -> `delivered` 停止转圈并显示转圈 `Thinking...` -> 回复到达后 `Thinking...` 消失
- **[#56] OTA Build 每次独立日志留存 + `commands.rs` `ENRICHED_PATH`**: `build_and_read_ota_dist` 使用 `ENRICHED_PATH` 执行 `pnpm exec vite build`，并将每次构建 stdout/stderr/耗时写入 `.agy-state/logs/ota-build-<ts>-<commit>.log`（保留最近 20 次）
- **[#57] `Unified Tray` 日志按 `seq` 严格去重 + `Clear View` `minSeq` 游标过滤** (解决 ring buffer 重复追加导致行号重复/卡顿)
- **[#58] 消息达到 200/300 条时自动触发 DO `/channel/backup` 检查点** (Worker DO 端已上线，待补自动触发/前端同步)
- **[#59] `Quote` 120-160字摘要截断 + `↑ Back to Quote` 锚点按钮** (发送后正常回底部，发送键上方显示 `↑ Back to Quote`，发下一条无 Quote 消息后自动清除)
- **[#60] `Thinking...` / `Working` 计时器精确到小数点后一位** (`Thinking... 0m 01.3s` 格式，100ms 刷新精度)
- **[#61] `Unified Tray` 一键 `Git Pull` & `Rollback (HEAD@{1})` 按钮 + 合入前 `cargo check` 门禁** (防止 `quick_git pull_ff` 拉到 Rust 编译失败提交后 `p2p-remote-controller` 失联无法远程自救)

=== subagent_prompts.md ===
# Sub-Agent Prompt Templates & Notes (from User, 2026-11-24T04:43:55Z)

## User's Architectural Direction for Sub-Agents
- Sub-agents are dispatched by **User** (from prompt generated by Chief Agent), NOT directly by Chief via HTTP.
- Auth headers (`agy_sub_...` in old templates) are stale/known in Durable Objects and need to be **revoked & newly signed**.
- Assigned sub-agent channels should be **derived from current channel** (e.g. hashed from `ch_b308b620` so Chief can derive where the sub-agent is, while the sub-agent cannot derive Chief's channel).
- Task dispatch should go through **EMQX dynamic dispatch** (sub-agent checks in / reports online, Chief assigns task, sub-agent acks & reports progress/offline status), rather than competing over a Durable Object task board.
- `initial user request` / default input for sub-agents needs updating: **Chief is the "user" for the sub-agent**, just as Human is the "user" for Chief.
- Worker deployment API token will be provided later to persist and iterate on the worker.

## Legacy Template 1: `atlas`
- Auth: `x-agy-request-id: agy_sub_0a5ca6ef-7427-498f-a602-e1b605dead39` (to be revoked/rotated)
- Channel: `ch_sub_3458d88135`
- Key features: `GET /channel/state`, `POST /chat/send` with `"drainInbox": true`, `PATCH /channel/state` (`update_status`, `in_review`, `resultCommitSha`, `resultSummary`), 45m standby (`10 * 4.5m` timeouts + 3 more + check `sessionStatus === "shutdown"`).

## Legacy Template 2: `coder_1`
- Auth: `x-agy-request-id: agy_sub_9a8195b9-bc3c-4296-b959-49cedf5edd35` (to be revoked/rotated)
- Channel: `ch_code_a2_worker1`
- Same `drainInbox` + `channel/state` + 45m standby pattern.

## Legacy System Instruction (`fix.sh`)
- `https://envinit.maboco-mark.workers.dev/` (deprecated; env init now built into each repo via `scripts/env/bootstrap.sh` / `doctor.mjs`).
- Useful rules retained: `replace_file_content` over `write_to_file`, never dump raw `git show`/`git diff` on whole dir, extract to `/tmp/branches/` for `diff -u`, never print large Base64 in tool args/stdout during GraphQL `createCommitOnBranch`.

=== emqx_split_blueprint.md ===
# EMQX Live Chat vs p2p-remote-controller Split Blueprint (#5)

> Audited by `Alpha` on `origin/a2` (`86ec113`).

- **Shared Pure-TS Package (`packages/emqx-chat-core`, zero Tauri/DOM deps)**:
  - Types: `BridgeChatMessage`, `EmqxConnectionState`, `E2eeMode`, `PocketDirectChatConfig`, `AgentBoardSnapshot`.
  - Transport/Crypto/Utils: `emqxClient.ts`, `emqxMessageParser.ts` (chat/status), `emqxBridgeUtils.ts` (`computeChatMessageId`, `mergeAndSortChatMessages`, `updateSendTargetForChannelSelection`), `agentBoard.ts`, `pollCountdown.ts`, `chatMessageCollapse.ts`, E2EE decrypt/encrypt.
- **Standalone `apps/emqx-live-chat` (Chat-only PWA/Web)**:
  - Hooks/UI: `usePocketDirectChat.ts` (with injectable storage/sound adapters), `usePollCountdown.ts`, `EmqxChatSection.tsx`, `AgentBoardBar.tsx`, `PocketDirectChatSettingsCard.tsx`.
- **Control Plane `apps/p2p-remote-controller` (Tauri Host + Pocket Deck)**:
  - Keeps `src-tauri/` (`lan_server.rs`, `commands.rs`, `ax_helper`), `axService.ts`, `emqxCommandRouter.ts`, `emqxCommandHandlers.ts`, `aiStudio*`, `useEmqxApprovalsAndChat.ts`, `usePocketDeckBridge.ts`, `usePocketDeckRemoteSync.ts`, `useLanAutoProbe.ts`, `useCellularP2p*`, `EmqxPendingApprovalsSection.tsx`, `PocketDeckView.tsx`.
- **Avoiding Fake Splits**: Extract `packages/emqx-chat-core` as a workspace package FIRST so `computeChatMessageId`, E2EE, and `BridgeChatMessage` never drift; keep `RemoteCommandRequest`/`emqxCommandRouter` strictly in `p2p-remote-controller`.
