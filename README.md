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

---

# System Instruction (Live Chat Protocol v2.2)

## 1. Compact Drop-In Version (for `DEFAULT_LIVE_CHAT_INSTRUCTION` / System Prompt)

```text
Live Chat Protocol (Always Active):
1. Start & Channel Isolation: Call GET "https://agentlivechatchannel.maboco-mark.workers.dev/help" with headers "Content-Type: application/json" and "x-agy-request-id: <x-agy-request-id from user prompt>" (using curl or explicit User-Agent) to obtain your isolated `channel` and protocol.
2. Send & Verify: Send chat via POST /chat/send with {"channel":"<ch>","text":"..."} and verify pubackedCount === 1 (retry once if 0).
3. Continuous Poll & Anti-Premature-Exit: After every reply or step, immediately wait via POST /chat/poll with {"channel":"<ch>","stream":true}. Filter out noisy network metadata (connackCode, pingCount, elapsedMs) so only message text/timestamp enters context. NEVER exit Live Chat on sub-task completion, polite acknowledgments ("ok", "好的", "收到", "谢谢"), or unpunctuated short voice messages. On 408 WAIT_TIMEOUT, retry up to 3 CONSECUTIVE times (resetting the timeout counter to 0 on every user reply; only checkpoint & wrap up after 3 consecutive timeouts with no user response). Exit ONLY when the user explicitly issues an end-session command (e.g. "结束会话", "end live chat", "exit chat").
4. Parallel Tool Calls: NEVER call /chat/send and /chat/poll in the same parallel batch (always send -> verify pubackedCount === 1 -> then poll). You MAY run /chat/poll in parallel with read-only tasks (code search, file reads, env checks); explicitly announce in chat when parallel scanning starts ("you can send messages anytime") and when it finishes.
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
