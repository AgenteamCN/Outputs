# Live Chat Communication Cadence (`ack-before-work`) Retrospective

## 1. Problem Observed
During rapid Live Chat iteration, when Chief went straight into multi-step background execution (cloning repos, reviewing sub-agent diffs, running `tsc`/`vitest`/`cargo check`, pushing GraphQL commits) without sending a brief notice first, the user experienced 30–120 seconds of silence and repeatedly wondered whether their client had disconnected (*"我是不是又掉线了？"*).

## 2. Root Cause
- In synchronous HTTP/MQTT Live Chat, the user has no visibility into tool calls happening inside the agent sandbox.
- Any silent gap >15–20 seconds feels indistinguishable from a dropped WebSocket (`Close code 1005`) or crashed worker.
- Additionally, splitting `send` and `poll` into two separate tool calls introduced extra round-trip latency between replying and resuming listening.

## 3. Rules & Patterns Adopted
1. **Mandatory `ack-before-work` Rule**:
   - **Always send a brief 1–2 line notice BEFORE** starting any multi-step background action (diff review, test/build suite, GraphQL merge, or long report generation), stating what is being done and roughly how long it will take.
   - **Skip extra interim ACKs** only for fast (<3s) read-only checks or direct Q&A.
2. **Single-Call `POST /chat/send_and_poll`**:
   - Combined reply + listen into one atomic `send_and_poll` call (`--max-time 150`, `stream: true`, `NotificationTimeoutSeconds: 170`), eliminating the unmonitored gap after sending a message.
3. **Context Compaction Transparency & Disk Persistence**:
   - Persisted key decisions and open backlogs to `/workspace/.chat/backlog_notes.md`.
   - Explicitly informed the user in Live Chat immediately after context compaction occurred, preventing silent drift.

## 4. Recommendations for Future Instructions
- **System / Agent Instruction**: Codify `ack-before-work` as a hard pre-condition before any tool chain expected to exceed ~10 seconds, and standardize `send_and_poll` as the default turn-closing call.
- **Human Instruction**: Encourage users to end multi-part voice dictation with an explicit completion cue (e.g., *"说完了"*) when batching thoughts, and reassure them that brief `"去跑检查，1分钟回"` messages mean the agent is actively working, not idle.
