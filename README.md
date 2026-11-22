# Live Chat Session Overview & Status Index

- **Request ID**: `5cd24140-d0d2-49b8-a1ae-54393dcc0b7a`
- **Channel**: `ch_6eb92a46`
- **Target Repository**: `https://github.com/Antigravi-ty/agyPreviewInstant` (latest inspected commit: `6f85635ab0264566145b3667386f763cf09167b6`)
- **Backup Gateway**: `https://agentoutputposter.maboco-mark.workers.dev/`

## Topic Completion Status
- `[DONE]` **System Instruction (`system_instruction.md`)**: Finalized anti-premature-exit protocol, `send` -> `poll` sequencing, read-only parallel rules, voice-input tolerance, compaction recovery + user notification, and Git safety.
- `[DONE]` **Human Instruction (`human_instruction.md`)**: Finalized bidirectional guide covering parallel-mode high-frequency messaging, soft `"说完了"` / `"over"` turn-taking, interleaving factual questions, and explicit control signals.
- `[DONE]` **Compaction & Disk Persistence (`compaction_persistence.md`)**: Finalized Agent-only raw buffer with pruning, semantic topic filenames, JSON metadata filtering, and disk stitching for super-large reports.
- `[DONE]` **Pocket Deck Top-Right Menu & OTA Version Check Scan (`pocket_deck_ota_menu_scan.md`)**: Located root cause in `PocketDeckHeader.tsx` & `PocketDeckView.tsx` (menu button only toggles local state; `activeSnapshot.commitId` only updates on initial P2P connect or manual `Fetch`) and documented fix direction (no repo code modified per user instruction).
- `[DONE]` **Web Audio Notification Sound & Auditioning Tools (`web_audio_sound_tools.md`)**: Documented zero-file `AudioContext` synthesis in `notificationSounds.ts` and recorded ZzFX / Jsfxr / Tone.js for future sound auditioning.
- `[OPEN]` **Future Roadmap & Open Items (`future_roadmap_and_open_items.md`)**: Tracked AI Studio `Inject Live Chat` AX path bug and Unified Main Tray + Dedicated AGY Terminal (Multi-Tab PTY) for upcoming discussion.
- `[IN_PROGRESS]` **Unified Tray, AGY Terminal, Quick Git, Sing-box & Standalone Live Chat App (`unified_tray_terminal_and_control_plane.md`)**: Captured architectural vision for fault-isolated Control Plane (standalone EMQX Live Chat app), Quick Git update/rollback commands, and Sing-box integration.
- `[OPEN]` **Wide-Screen Layout, Font Scaling & Remote Hotkey (`ui_layout_and_remote_hotkey.md`)**: Recorded 16:9/3:2 Mobile Viewport layout & distance font-size ideas; verified `MacosAxHelper` currently lacks hotkey posting and documented `CGEventPost` + `kCGEventFlagsChanged` feasibility.
- `[DONE - Scan]` **Pocket Deck Chat Auto-Scroll & Stick-to-Bottom Toggle (`pocket_deck_chat_autoscroll_scan.md`)**: Located missing new-message scroll effect, missing `onScroll` handler, and one-shot `<ArrowDown />` button in `PocketDeckView.tsx`; documented exact state machine.

---

# System Instruction (Live Chat Protocol v2.2)

## 1. Compact Drop-In Version (for `DEFAULT_LIVE_CHAT_INSTRUCTION` / System Prompt)

```text
Live Chat Protocol (Always Active):
1. Start & Channel Isolation: Call GET "https://agentlivechatchannel.maboco-mark.workers.dev/help" with headers "Content-Type: application/json" and "x-agy-request-id: <x-agy-request-id from user prompt>" (using curl or explicit User-Agent) to obtain your isolated `channel` and protocol.
2. Send & Verify: Send chat via POST /chat/send with {"channel":"<ch>","text":"..."} and verify pubackedCount === 1 (retry once if 0).
3. Continuous Poll & Anti-Premature-Exit: At the end of every reply, alignment checkpoint, or completed task phase, immediately wait via POST /chat/poll with {"channel":"<ch>","stream":true}. (During active multi-step execution, you MAY send non-blocking progress updates via /chat/send and immediately proceed to the next step without waiting for /chat/poll.) Filter out noisy network metadata (connackCode, pingCount, elapsedMs) so only message text/timestamp enters context. NEVER exit Live Chat on sub-task completion, polite acknowledgments ("ok", "好的", "收到", "谢谢"), or unpunctuated short voice messages. On 408 WAIT_TIMEOUT, retry up to 3 CONSECUTIVE times (resetting the timeout counter to 0 on every user reply; only checkpoint & wrap up after 3 consecutive timeouts with no user response). Exit ONLY when the user explicitly issues an end-session command (e.g. "结束会话", "end live chat", "exit chat").
4. Parallel Tool Calls: NEVER call /chat/send and /chat/poll in the same parallel batch (always send -> verify pubackedCount === 1 -> then poll). You MAY run /chat/poll in parallel with long read-only tasks (code search, file reads, env checks), but do NOT pair a 270s /chat/poll with a fast 2-second check if you intend to immediately report the result; explicitly announce in chat when parallel scanning starts ("you can send messages anytime") and when it finishes.
5. Voice-Input Tolerance & Turn-Taking: Expect unstable voice dictation (homophones, typos, missing punctuation). Infer intent from context and confirm ambiguous identifiers rather than guessing or aborting. Allow interleaving/bursts: wait for soft end-of-burst cues ("说完了" / "over") during multi-part thoughts, while answering self-contained factual questions immediately.
6. Disk Persistence & Compaction Recovery: Continuously persist raw user requirements and categorized semantic topic files (with [DONE]/[OPEN] status, outside the tracked repo, e.g. `/workspace/.live_chat/`). If context compaction occurs, immediately restore state from those semantic files, briefly notify the user in chat that compaction occurred and state was restored, then continue /chat/poll.
7. Alignment & Git Safety: Always align in chat and announce briefly before modifying code; push code only via Verified Bot Commit (never local git push).
```

## 2. Design Rationale (Hard Constraints vs. Dynamic Pacing)
- **Hard constraints belong in System Instruction**: Preventing premature exit, enforcing `send` before `poll`, retrying `408 WAIT_TIMEOUT`, and compaction recovery are irreversible if violated—once an Agent exits the `/chat/poll` loop, the user cannot correct it in the next turn.
- **Soft pacing belongs in chat + Human Instruction**: Over-constraining conversational pacing in System Instruction dilutes attention and makes the model rigid. Instead, Human Instruction empowers the user to steer pacing ("hold on", "dig deeper", "说完了") dynamically.

---

# Human Instruction (Bidirectional Collaboration Guide)

## 中文版（操作者速览）
1. **并行工作与高频连发**
   - 当 Agent 播报“正在并行扫描/检查，你可以随时发消息”时，你可以用语音连续、高频地补充想法，不用“说一句等一句”。
   - 当 Agent 播报“扫描完成”时，代表已回到正常对话节奏。
   - 完全允许插队提问（例如查某个 commit ID 或文件路径），独立明确的问题 Agent 会直接秒答。
2. **语音输入（iOS 听写）配合技巧**
   - 无需刻意补全标点，也不必担心同音错别字（Agent 已配置语义纠偏容错）。
   - 如果需要分段连发长想法，可以在最后一句加上 **“说完了”**（或 **“over”**），作为软信号提醒 Agent 统一回复，避免中途被打断。
   - 涉及精确的 URL、Token、仓库名或关键变量名时，建议文字补充或让 Agent 复述确认。
3. **明确的控制口令（防误触退出）**
   - **日常确认**（“好的”“收到”“谢谢”“继续”）：**不会**结束会话，Agent 会继续留在 Live Chat。
   - **暂停动手**：“先别改代码 / 先只扫描讨论”——阻止 Agent 提前修改文件或赶进度定稿。
   - **正式结束**：明确说出 **“结束会话” / “end live chat” / “exit chat”**，Agent 才会退出轮询并收尾。
4. **超时与静默预期**
   - 单次 `/chat/poll` 最长等待 4 分 30 秒，Agent 执行本地命令或扫描时会有 5~30 秒静默，属正常现象；如遇到上下文压缩，Agent 恢复本地落盘后会在对话中主动报备一句。

## English Version
1. **Parallel Work & High-Frequency Messaging**
   - When the Agent announces it is running read-only scans/checks in parallel, you may send multiple voice messages continuously without waiting turn-by-turn.
   - The Agent will announce when parallel scanning finishes so you know normal pacing has resumed.
   - Interleaving self-contained questions (e.g. checking a commit ID) is always welcome; the Agent will answer them directly.
2. **Voice Dictation Tips**
   - Missing punctuation and dictation homophones are tolerated automatically.
   - For multi-part voice bursts, end your final burst with **"说完了"** (or **"over"**) as a soft cue so the Agent doesn't interrupt mid-thought.
   - For exact URLs, tokens, or identifiers, type them or ask the Agent to echo them back.
3. **Explicit Control Signals**
   - Casual acknowledgments ("ok", "got it", "thanks") will **not** end the session.
   - To hold implementation: say **"pause implementation / don't edit code yet"**.
   - To terminate Live Chat: explicitly say **"end live chat" / "exit chat" / "结束会话"**.
4. **Timeouts & Compaction Transparency**
   - Each `/chat/poll` waits up to 4m30s (up to 3 consecutive timeouts = ~13.5m, resetting on every reply); brief silences during local scans are normal. If context compaction occurs, the Agent will restore from disk and briefly notify you in chat.
5. **Temporary Departure & Session Handoff**
   - Stepping away briefly / switching to Pocket Deck mobile: say **"我先离开一下，保持在线"** (Agent keeps polling).
   - Stepping away long-term / resuming in a new sandbox later: say **"我要出门了，先做个接力快照并备份"** (Agent updates `session_status.md` and POSTs a single stitched checkpoint report to `agentoutputposter`).

---

# Compaction & Disk Persistence Architecture

## 1. Why Live Chat Needs Disk Persistence
Unlike single-prompt tasks, Live Chat sessions generate **most of their concrete requirements dynamically inside `/chat/poll` tool outputs**. When context compaction triggers, early tool outputs and chat turns are collapsed into a lossy summary—risking loss of the `channel` ID, exact user requirements, and which topics are settled vs. still open.

## 2. Persistence Rules
1. **Store outside the tracked repository** (e.g. `/workspace/.live_chat/`) so working files never pollute `git status` or commits.
2. **Record high-signal content only (skip verbose Agent chatter)**:
   - **Raw User Scratchpad (`ch_<id>_raw_buffer.md`, Agent-only, excluded from final report)**: Verbatim incoming user messages with timestamps. Once a message's requirement has been distilled into a semantic topic file, **prune/delete** that settled entry from the raw buffer so the buffer stays tiny and only holds unabsorbed/active messages.
   - **Semantic Topic Files** (never generic `1.md`, `2.md`): e.g. `system_instruction.md`, `human_instruction.md`, `compaction_persistence.md`, `pocket_deck_ota_menu_scan.md`. Semantic filenames allow a post-compaction Agent to run `ls` and immediately know what each file holds.
3. **Filter Poll/Send JSON Noise**:
   - Parse `/chat/poll` and `/chat/send` responses in-script to print only `sentAt` + `text` (and errors on failure), stripping `connackCode`, `pingCount`, `elapsedMs`, `qos`, etc. to preserve context budget and delay compaction.

## 3. Compaction Recovery Protocol
1. **Detect**: Agent notices older turns/tool outputs were replaced by a compaction summary, or feels uncertain about `channel` ID / prior decisions.
2. **Restore**: Read `/workspace/.live_chat/ch_<id>.md` and relevant semantic topic files (`ls /workspace/.live_chat/`).
3. **Notify**: Briefly inform the Human in chat that context compaction occurred and state has been restored from disk, so the Human can accurately judge the workflow state.
4. **Resume**: Continue `/chat/poll`.

## 4. Super-Large Report Assembly (Disk Stitching)
- Write each topic/finding as an independent semantic markdown file as the session progresses.
- At the deterministic finalization moment, stitch them directly on disk via shell (`cat ... > /tmp/output.md`) without forcing the LLM to regenerate tens of thousands of tokens in one response, then `curl --data-binary @/tmp/output.md` to `https://agentoutputposter.maboco-mark.workers.dev/`.

---

# Module 4: Pocket Deck Top-Right Menu & OTA Version Check Scan

## Problem Statement (User Requirement)
- When tapping the top-right menu button on Pocket Deck (mobile P2P client), the client should send a message/RPC to the PC Host to check the latest version (`commitId`).
- Current symptom: after loading completes, the version status only updates on the **first** P2P connection, so subsequent Host updates are not reflected when opening the top-right menu.

## Code Location & Root Cause Analysis (Read-Only Scan)
1. **Top-right menu button (`apps/p2p-remote-controller/src/components/PocketDeckHeader.tsx`)**:
   - The `<Menu />` button `onClick` only executes `setMenuOpen((prev) => !prev)` (no callback/RPC is triggered when opening the menu).
   - The `hasHostUpdate` highlight and `Update (#<hostCommitId>)` label inside the dropdown depend on `hasHostUpdate` and `hostCommitId` passed from `PocketDeckView`.
2. **Version comparison source (`apps/p2p-remote-controller/src/components/PocketDeckView.tsx`)**:
   - `hasHostUpdate` is computed as `!embedded && Boolean(activeSnapshot.commitId && activeSnapshot.commitId !== commitId)`.
   - In P2P mode (`!wsConnected && p2pConnected`), `activeSnapshot` comes from `p2pSnapshot`, which is only populated on initial P2P connection (or when the user manually clicks `Fetch` / `fetchP2pSnapshot()`).
   - Therefore, if the PC Host updates after the first P2P connection, `activeSnapshot.commitId` remains stale until a manual `Fetch` or full snapshot refresh.
3. **Host OTA endpoint (`apps/p2p-remote-controller/src/hooks/useCellularP2pHost.ts` & `src/services/p2pOtaService.ts`)**:
   - Host already supports `pocket_deck_ota_manifest` (which returns `bundle.manifest.commitId`) as well as snapshot queries.
   - However, `PocketDeckHeader` has no `onMenuOpen` / `onCheckUpdate` hook wired to `fetchP2pSnapshot()` or a lightweight `pocket_deck_ota_manifest` / version-check RPC when the top-right menu opens.

## Recommended Fix Direction (For Report / Future Implementation)
- Add an `onOpenMenu` (or `onCheckHostVersion`) callback prop to `PocketDeckHeader` invoked when `menuOpen` transitions from `false` -> `true`.
- In `PocketDeckView`, when `!embedded && !wsConnected && p2pConnected`, trigger `fetchP2pSnapshot()` (or a lightweight version/manifest query) on menu open so `activeSnapshot.commitId` and `hasHostUpdate` refresh immediately.
- Optional: debounce/throttle menu-open checks (e.g. 3–5s) to avoid redundant P2P RPC traffic on rapid toggles.

---

# Pocket Deck Chat Auto-Scroll & Stick-to-Bottom Toggle Scan

## Problem Statement
On mobile Pocket Deck (`PocketDeckView.tsx`), incoming messages do not auto-scroll to the bottom, and the `<ArrowDown />` ("Scroll to bottom") button is only a one-shot scroll rather than a coupled auto-follow toggle.

## Code Location & Root Cause (`apps/p2p-remote-controller/src/components/PocketDeckView.tsx`)
1. **Missing new-message auto-scroll effect**:
   - Currently `scrollContainerRef.current.scrollTop = scrollHeight` only runs when switching into `mode === "bridge"` (lines 192–200) or when the textarea height changes while already near bottom (lines 204–224). There is **no effect** triggered by incoming `chatMessages` updates.
2. **One-shot `<ArrowDown />` button (lines 974–986)**:
   - Clicking `<ArrowDown />` only calls `el.scrollTo({ top: el.scrollHeight, behavior: "smooth" })` and does not track or toggle a `stickToBottom` state.
3. **Missing `onScroll` listener on `scrollContainerRef` (line 885)**:
   - Manual scroll-up cannot disengage auto-follow because `scrollContainerRef` has no `onScroll` handler.

## Expected State Machine (For Future Implementation)
- Introduce `stickToBottom` state (default `true`, visually indicated on `<ArrowDown />` or active/inactive state).
- **On incoming messages**: if `stickToBottom === true`, scroll `scrollContainerRef` to bottom.
- **On manual scroll (`onScroll`)**: if user scrolls up away from bottom (`scrollHeight - scrollTop - clientHeight > threshold`), set `stickToBottom = false`.
- **On clicking `<ArrowDown />`**: scroll to bottom and set `stickToBottom = true`.
- **On sending a chat message**: set `stickToBottom = true` and scroll to bottom.

---

# Web Audio Notification Sound Architecture & Synthesis Tools

## 1. Current Implementation (`apps/p2p-remote-controller/src/utils/notificationSounds.ts`)
- Zero external audio files (`.mp3` / `.wav`); synthesized in real time via Web Audio API (`AudioContext` + `OscillatorNode` + `GainNode` + `DynamicsCompressorNode`).
- Presets:
  - `chime` (Soft Chime C5-E5): dual `sine` waves (`523.25 Hz` -> `659.25 Hz`, 0.11s offset)
  - `ping` (Crystal Ping): `triangle` wave (`880 Hz`)
  - `marimba`: triple `triangle` arpeggio (`523.25` -> `659.25` -> `783.99 Hz`)
  - `alert` (Double Alert): dual `sine` waves (`783.99 Hz` x2)
- Previewable in `EmqxBridgeSettingsCard.tsx` (`Notification Sound & Boost`).

## 2. Zero-File / Low-Effort Sound Synthesis Tools (For Future Auditioning)
- **ZzFX** (`https://killedbyapixel.github.io/ZzFX/`): Interactive web auditioning & random generator; exports a single tiny array of parameters (tens of bytes) to reproduce any sound.
- **Jsfxr** (`https://sfxr.me/`): Classic browser-based sound generator with presets and minimal JS export.
- **Tone.js** (`https://tonejs.github.io/`): Richer Web Audio synth presets (e.g., PolySynth, PluckSynth, FMSynth with filters/reverb) with interactive docs.

---

# Unified Tray + AGY Terminal + Standalone EMQX Live Chat Control Plane

## 1. Core Architectural Vision
Separate the **Control Plane** (always-on, stable) from **Managed Target Apps / Repos** (`agyPreviewInstant`, `p2p-remote-controller`, `Sing-box`, etc.), so that a broken `git checkout` or crash in any managed app never takes down remote communication or recovery capabilities.

## 2. Key Subsystems
1. **Standalone EMQX Live Chat Application**:
   - Extract the now-stable EMQX Live Chat into its own standalone `.app` bundle with auto-reconnect to EMQX Live.
   - Built-in management capabilities: dispatch commands to inspect error logs, trigger repo `checkout` / updates, or roll back a broken commit even when target apps fail to launch.
   - Fault isolation: updating `p2p-remote-controller` only restarts `p2p-remote-controller`; updating `agyPreviewInstant` only restarts `agyPreviewInstant`; the Live Chat Control Plane stays untouched.
2. **Unified Main Tray + Dedicated AGY Terminal (Multi-Tab PTY)**:
   - **Process Tree & Tray Design Options (No Circular Launch)**:
     - *Option 1 (Tray Built into `AGY Terminal.app` — Single Root)*: `AGY Terminal.app` hosts the system tray + multi-tab PTY UI (window can close to tray). Clicking a project in the tray opens a PTY Tab inside `AGY Terminal` to run `tauri dev` (`flock` dedup by `identifier`). Only one `.app` needs macOS AX permission.
     - *Option 2 (`Global Remote Controller.app` + `AGY Terminal.app` Split — Max Fault Isolation)*: `Global Remote Controller.app` hosts Tray + EMQX Live Chat + Sing-box + Quick Git, and commands `AGY Terminal.app` via local IPC to open PTY Tabs for `tauri dev`. Even if `AGY Terminal` crashes or breaks during UI edits, the Control Plane stays online (requires one-time AX grant on both `.app` bundles).
   - **Security & Permission Isolation**: Because every `tauri dev` child process stays inside `AGY Terminal.app`'s process tree, all dev apps cleanly inherit `AGY Terminal.app`'s macOS AX permission without granting broad AX permissions to the global system `Terminal.app`.
   - **Quick Git / Remote Commit Sync**:
     - Remote `checkout` / update / rollback commands (no local commit push needed).
     - **Commit Graph Visualization (Mobile & Remote)**: Display branch topology (divergent branches from parallel Agents, octopus merges, commit parents) so mobile/remote users can see the exact graph structure before checking out a commit rather than only seeing a bare commit hash.
3. **Remote Co-Debugging & Direct Log Inspection (Cloud Linux Agent <-> Local macOS)**:
   - **Rationale**: Cloud Linux sandboxes cannot realistically simulate macOS (TCC/AX, Safari, WebView, Swift helpers). Instead of asking the Human to manually copy-paste logs, the Agent should directly issue remote commands to read `AGY Terminal` PTY output or target log files.
4. **Controller Split Analysis (`Safari Remote Controller` vs `Global Remote Controller`)**:
   - **Option A (Two Apps / Fault-Isolated)**: Keep `Global Remote Controller` (hosting EMQX Live Chat, AGY Terminal control, Quick Git, Sing-box, and log reader) as a minimal, rarely-changed Control-Plane `.app`, and separate `Safari Remote Controller` (Safari AX/JS/Net Recorder) as a managed target app. If a Safari-controller edit breaks build/startup, the Global Controller remains online to read build logs and roll back via Quick Git.
   - **Option B (Single App)**: Lower packaging overhead, but a broken UI/build in Safari features could take down the Live Chat / Terminal controller too.
5. **Sing-box (`sing-box`) Network Service Integration**:
   - Integrate `sing-box` lifecycle management directly into the same unified tray / terminal control plane.
6. **Log Capture & Tray Execution Model**:
   - **Terminal Logs vs. Web Console (Do NOT rely on AX for logs)**:
     - *Terminal Logs*: Since `AGY Terminal` owns the `portable-pty` master fd in Rust, tee `stdout`/`stderr` directly into a ring buffer / log file—100% lossless and far more reliable than macOS AX screen scraping.
     - *Web Console*: macOS AX cannot reliably read internal JS `console.log`/`error` stacks; use lightweight frontend console hooks or `execute_javascript` forwarding over IPC/EMQX.
   - **Tray Launch Strategy (Hybrid)**:
     - *Via AGY Terminal PTY Tab* (for interactive/dev processes like `agy:dev`, `p2p:dev`): preserves full TTY (colors, interactive keys), lets humans inspect/take over anytime, and inherits `AGY Terminal.app` AX permissions.
     - *Via Direct Background Spawn* (for daemons & one-shot CLI like `sing-box`, `git checkout`, `git log`): lightweight, zero UI clutter, stdout/stderr piped to log buffer.

---

# UI Layout, Remote Hotkey & LAN Synced Dictation

## 1. `[OPEN]` Wide-Screen PC Layout for Mobile Viewport (16:9 / 3:2)
- Current vertical stacking underutilizes horizontal space on PC 16:9 / 3:2 screens, while naive horizontal layout feels awkward.
- Needs a dedicated responsive multi-pane / adaptive layout design later.

## 2. `[DROPPED]` Global Distance-Reading Font Scaling on PC
- User decided against forcing large-font distance reading on PC screens—reading directly on the phone (Pocket Deck) is superior.

## 3. `[OPEN - High Value, LAN-Only]` Remote Mac Dictation Hotkey + Real-Time Synced Input Box
- **User Scenario**: When on LAN (at home/office, away from the desk), use the phone (Pocket Deck) to read output, while triggering the superior **Mac dictation engine** remotely instead of using iOS dictation.
- **Scope**: **LAN-only** (not supported over remote P2P/EMQX relay, avoiding EMQX quota waste on keystroke-level sync).
- **Two Coupled Features**:
  1. **Remote Hotkey Trigger (`CGEventPost`)**: Phone taps a button -> LAN WS -> PC Host focuses the target input box and triggers `MacosAxHelper` (`send_hotkey` via `CGEventPost` + `kCGEventFlagsChanged` for `Control + Command`).
  2. **Real-Time Synced Input Box**: Text dictated into the Mac input box syncs live over LAN WebSocket back to the phone's input box, where the user can review/edit and tap Send.
- **Feasibility Evaluation**:
  - **Latency & Cost**: LAN WS has <5ms latency and zero EMQX quota cost, ideal for real-time `onChange` text mirroring.
  - **Focus Guarantee (No external AX traversal needed)**: Since the synced input lives inside our own Tauri app (`p2p-remote-controller`), do NOT rely on external macOS AX tree traversal to locate it. Instead: (1) Rust `app.webview_windows()` calls `show()` + `set_focus()` to bring the app window to OS foreground; (2) Frontend focuses the input via React `ref.current?.focus()` on event; (3) optionally add `data-ui-action` / `aria-label` so Self UI scanner can also index it; (4) then invoke `CGEventPost` to start Mac dictation.
  - **Permission**: `CGEventPost` reuses the existing macOS Accessibility (AX) permission.

---

# Future Roadmap & Open Items

## 1. `[OPEN]` AI Studio `Inject Live Chat` Auto-Injection Path Bug
- **Symptom**: Clicking `Inject Live Chat` in `SafariControlStrip` fails when performing macOS AX injection against a real AI Studio window (AX path misaligned and missing complete Automation trajectory).
- **Next Step**: Run `scan_window` on a live AI Studio Safari window to verify exact AX `role`/`title` paths and fix the injection sequence.
- **Current Workaround**: Copy/paste the instruction manually from the bottom of the Self UI tab.

## 2. `[OPEN]` Unified Main Tray + Dedicated AGY Terminal (Multi-Tab PTY)
- **Goal**: Build a lightweight unified system tray + multi-tab pseudo-terminal app (Rust `portable-pty` + `xterm.js`), packaged as a fixed `.app` bundle with one-time macOS Accessibility (AX) permission.
- **Mechanism**: Dev child processes launched from this terminal automatically inherit macOS AX permissions, deduplicate via `flock` (keyed by `identifier`), and sync the managed process whitelist to P2P Remote Controller via Rust IPC.
- **Status**: Expanded in `unified_tray_terminal_and_control_plane.md`.

## 3. `[OPEN]` Live Chat Worker `POST /chat/send_and_poll` Endpoint
- **Idea**: Add an optional combined `POST /chat/send_and_poll` endpoint to `agentlivechatchannel` Worker to cut standard conversational turns from 2 tool calls to 1.
- **Coexistence Rule**: Keep standalone `/chat/send` (fire-and-forget progress updates during active coding without blocking 4m30s) and standalone `/chat/poll` (silent listen & `408` retries).
