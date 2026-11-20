# Live Chatting & `/cmd` 排查分析报告

## 一、 测试与验证结果摘要

1. **Worker 源码与接口状态**
   - `agentplayground.maboco-mark.workers.dev`：已下线迁移，返回 HTTP `410` (`MIGRATED`)，指向 `agentlivechatchannel.maboco-mark.workers.dev`。
   - `agentlivechatchannel.maboco-mark.workers.dev`：运行 `live-chat-v2`（884 行），本次分配隔离频道 `ch_007666bd`。

2. **并发自测（Self-Test，证明 Worker 核心链路可正常 Early Return）**
   - **`/chat/poll` + `/chat/send` 自测（频道 `ch_selftest_01`）**：
     - 先启动 `/chat/poll`（`stream: true`），2 秒后由模拟客户端 `sender: "user_sim"` 调用 `/chat/send`。
     - `/chat/send` 耗时 `877ms`（`pubackedCount: 1`）；`/chat/poll` 在收到消息后立即 **early return**（`elapsedMs: 2678`，总耗时 `3065ms`）。
   - **`/cmd` + `/resp` 自测（频道 `ch_selftest_02`）**：
     - 模拟端通过 `/mqtt/rpc` 订阅 `agy_p2p_v1/cmd/ch_selftest_02/#`，并发调用 `/cmd` (`command: "ping"`)，模拟端收到 `requestId` 后回传到 `agy_p2p_v1/resp/ch_selftest_02/<requestId>`。
     - `/cmd` 在收到回包后立即 **early return**（`elapsedMs: 2090`，总耗时 `4263ms`）。

3. **真实频道 `ch_007666bd` 连通性尝试**
   - 已通过 `/chat/send` 发送上线通知与进度同步（均返回 `ok: true, pubackedCount: 1`）。
   - 已通过 `/cmd` 发送 `ping`（等待 20s，`subacked: true`，5 次 heartbeat，0 响应）。
   - 已连续 **3 次** 调用 `/chat/poll` 监听 `ch_007666bd`（分别等待 12s、30s、25s，`subacked: true`，均收到 0 条入站消息）。

---

## 二、 为什么会出现“消息发不出 / 无 Early Return / 连续戛然而止”

结合 `agentlivechatchannel` 源码、`agyPreviewInstant` 客户端源码以及你提供的历史消息记录，定位到以下 **4 个核心根因**：

### 1. EMQX 持久会话冷启动竞态（直接导致 `ch_3f7cb72a` 戛然而止）
查看你提供的 `ch_3f7cb72a` 时间线：
- `10:24:24.571Z`：Agent 通过 `/chat/send` 发送 `我现在开始监听 ch_3f7cb72a，请问需要扫描哪个页面？`
- `10:24:25.838Z`（**仅 1.267 秒后**）：你在 UI 回复 `你这效率也太快了吧！ID 47694。`
- 之后 Agent 彻底失联。

**原因**：
- `/help` 的 `quickStart` 引导 Agent 先调用 `/chat/send`，再调用 `/chat/poll`。
- `/chat/send` 使用临时客户端 `agy_send_<uuid>`（`cleanSession: true`），不会为接收端建会话。
- `/chat/poll` 使用持久客户端 `agy_poll_<channel>`（`cleanSession: false`）。**但在第一次 `/chat/poll` 连上 EMQX 并 `SUBSCRIBE` 之前，EMQX 上根本不存在 `agy_poll_ch_3f7cb72a` 的离线队列！**
- 因此你在 1.27 秒内的快速回复直接被 EMQX 丢弃；随后 Agent 启动 `/chat/poll` 干等 4 分 30 秒直至超时，完全不知道你已经回复过。

### 2. 频道隔离不对称：桌面端默认 `All` 发往默认频道，Worker 只听 `#ch_xxxx`
- 在 `agentlivechatchannel` 中，传入 `channel: "ch_007666bd"` 时，`/chat/poll` **只订阅** `agy_p2p_v1/chat/ch_007666bd`。
- 在 `agyPreviewInstant` 桌面端 `EmqxBridgePanel.tsx` 中，`selectedChannel` 默认为 `"all"`，且**没有**移动端那样的 `autoFollowChannel` 自动切换逻辑。
- 如果你在桌面端 `All`（或 `default`）视图直接回复，`handleSendChatSubmit` 会传入 `channel: undefined`，消息实际发往 `agy_p2p_v1/chat`，**Worker 的 `/chat/poll` 永远收不到，自然无法 early return**。

### 3. 移动端默认停留在 `llm` 页签，看不到 `Bridge Sync` 审批与聊天
- 在 `PocketDeckView.tsx`（第 60 行）中，移动端默认选中的是第 3 个页签 `mode = "llm"`（LLM Control）。
- `/cmd` 下发到 `agy_p2p_v1/cmd/<channel>/<command>` 后，客户端 `useEmqxConnection` 会正常收到并放入 `pendingApprovals`（桌面端与移动端快照均可见），但若移动端停留在 `llm` 页签且未切回 `Bridge Sync`，用户侧感知不到新命令到达，导致 `/cmd` 一直等待 270 秒。

### 4. `/cmd` 的 `SUBSCRIBE` 与 `PUBLISH` 并发竞态 & `/help` 文档误导
- **Worker `/cmd` 竞态**：`worker.mjs` 第 287-299 行在收到 `CONNACK` 后，连续发送 `SUBSCRIBE` 和 `PUBLISH` 而**未等待 `SUBACK`**。若响应端极快回包，可能抢在 EMQX 完成 `SUBACK` 前到达。
- **`/help` 引导顺序问题**：`/help` 仅提示 `stream: true` 与 `channel`，未提示“必须先建立 `agy_poll_<channel>` 订阅再发首条消息”，也未说明必须使用 `stream: true` 防止代理层在 60-100 秒无字节传输时断连。

---

## 三、 后续修复建议

### A. `agentlivechatchannel` Worker 优化建议
1. **消除 `/chat/send` -> `/chat/poll` 冷启动丢消息**：
   - 当 `/chat/send` 携带 `channel` 时，先用 `agy_poll_<channel>`（`cleanSession: false`）完成一次快速 `CONNECT + SUBSCRIBE` 再发消息（或由 `/help?channel=...` / `/chat/send` 预热持久会话），确保用户 1 秒内秒回也不会被 EMQX 丢弃。
2. **`/cmd` 等待 `SUBACK` 后再 `PUBLISH`**：
   - 在 `runMqttSession` 中，若同时有 `subscribeTopics` 和 `publishMessages`，等收到 `ptype === 9` (`SUBACK`) 后再发送 `PUBLISH`。
3. **优化 `/help` 返回**：
   - 在 `quickStart` 和 `note` 中明确写出：**先调用一次 `/chat/poll`（或使用已预热会话的 `/chat/send`）再等待用户回复**，并强调桌面端需选中 `#<channel>` 芯片后再回复。

### B. `agyPreviewInstant` 客户端优化建议
1. **桌面端 `EmqxBridgePanel.tsx` 补齐 `autoFollowChannel`**：与移动端 `PocketDeckView.tsx` 对齐，发现新 `ch_*` 临时频道时自动切换或提供一键高亮切换，避免回复误发到 `agy_p2p_v1/chat`。
2. **移动端 `PocketDeckView.tsx` 未读/待审批提醒**：在 `llm` 默认页签顶部 Tab 已有 `Bridge Sync (N)` 计数的基础上，可增加新命令/新聊天到达时的提示或快捷跳转。
