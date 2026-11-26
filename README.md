# agyWorker 频道漂移检测与并发时序打标技术方案（无 PII 版）

## 1. 背景与现象

### 1.1 频道 ID 漏传 / 漂移（`channelId` Drift / Omission）
- **现象**：在多频道协作中，Agent 在调用 `/chat/send`、`/chat/poll` 或 `/cmd` 时，偶发出现以下情况：
  1. 漏传 `channelId`，导致消息发往默认频道而非目标会话频道；
  2. `poll` 时传入的 `channelIds` 列表漏掉了活跃的用户频道；
  3. 上下文压缩恢复后使用了错误的 `channelId`。
- **后果**：Agent 无法感知自己未监听到用户所在频道，用户也无法判断 Agent 正在监听哪个频道，造成双向失联假象。

### 1.2 并发消息因果时序模糊（Pre-Reply / Concurrent Message）
- **现象**：
  - `t=1s`：Agent 发送消息 A；
  - `t=2s`：用户回复消息 B，Agent 收到后开始思考/执行工具（`t=2s..10s`）；
  - `t=7s`：Agent 尚未进入下一次 `poll`，用户补充发送消息 C（`sentAt = 7s`）；
  - `t=10s`：Agent 思考完毕，发出针对消息 B 的回复 D（`sentAt = 10s`）；
  - `t=11s`：Agent 调用 `/chat/poll` 拉取到消息 C（`sentAt = 7s`）。
- **问题**：消息 C（`7s`）是在回复 D（`10s`）发出**之前**就已发出的，并非针对回复 D 的 `User Response`。若仅按拉取顺序处理，Agent 或用户容易误判因果关系。

---

## 2. 技术方案设计

### 2.1 `previousChannelId` 漂移与漏监听检测
- **标识键设计**：
  - 主键采用 `(requestId, agentId)`（来自请求头 `x-agy-request-id` 与请求体 `agentId`），在 Worker Durable Object / 内存会话状态中持久化：
    - `previousChannelId`：该 `(requestId, agentId)` 最近一次发送/操作的主频道；
    - `recentActiveChannels`：该会话近期有消息往来的频道集合（含最新 `sentAt`）。
  - **关于出口 IP**：云端沙箱/容器可能存在共享 NAT 出口 IP 或快照重启后切换出口 IP 的情况，因此**不建议单独依赖 IP 作为唯一主键**，可将出口 IP（`CF-Connecting-IP`）仅作为辅助观测字段（可在后续 Agent Playground 实测出口 IP 稳定性与拿到的请求头数据后再评估权重）。
- **非阻塞告警机制（Non-blocking Warning）**：
  - 当 `/chat/send`、`/chat/poll` 或 `/cmd` 出现以下情况时，Worker 正常执行但额外在响应 JSON 中附带 `channelWarning` 与 `previousChannelId`：
    1. 漏传 `channelId`；
    2. 传入的 `channelId` 与 `previousChannelId` 不一致（发生频道切换/漂移）；
    3. `/chat/poll` 的 `channelIds` 漏掉了 `recentActiveChannels` 中近期有未读/活跃记录的用户频道。
  - 示例响应字段：
    ```json
    {
      "ok": true,
      "previousChannelId": "ch_example_user",
      "channelWarning": "channelId omitted or differs from previousChannelId (ch_example_user)"
    }
    ```

### 2.2 `lastAgentSentAt` 因果时序打标（`timingRelation`）
- **状态记录**：
  - Worker 在每次 `/chat/send` 或 `/chat/send_and_poll` 成功发送时，按 `(requestId, agentId, channelId)` 记录 `lastAgentSentAt`。
- **拉取标注**：
  - 在 `/chat/poll` 与 `/chat/send_and_poll` 返回 `received` 列表前：
    1. 先按消息 `sentAt` 升序排列（`seq` / `receivedAt` 兜底），并附带 `oldestSentAt` / `newestSentAt` 摘要；
    2. 对非本 Agent 发出的每条消息，比较 `msg.sentAt` 与 `lastAgentSentAt`：
       - `msg.sentAt < lastAgentSentAt` -> 标注 `timingRelation: "sent_before_last_reply"`（并发/发前追加消息，**非**对上一条回复的回应）；
       - `msg.sentAt >= lastAgentSentAt` -> 标注 `timingRelation: "sent_after_last_reply"`（上一条回复发出后的新消息）。
    3. 附带统计字段 `preReplyCount`，提醒 Agent 若仍在等待用户针对上一条回复的回应，应继续 `poll`。

---

## 3. 后续验证计划（Agent Playground 实测）
1. 在独立测试环境（Agent Playground）中验证沙箱连续请求时的 `CF-Connecting-IP` 变化频率与可用元数据；
2. 验证 `channelWarning` 与 `timingRelation` 在高并发/断连重放场景下的准确率与零误报率。
