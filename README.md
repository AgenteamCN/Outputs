# 消息时序判断与回应配对问题记录（Phenomenon & Problem Statement）

## 1. 现象描述
- 在多端（HMOS 语音输入 / Mac / Cloud Agent）通过 `agyWorker` (`/chat/send`, `/chat/poll`) 进行实时通信时，可能出现：
  1. **投递延迟与乱序**：用户侧发送时间（`sentAt`）较早的消息，因网络/重放/离线队列原因，在 Agent 侧 `receivedAt` 较晚才被拉取到。
  2. **回应配对模糊**：当用户连续发送多条消息，或 Agent 并发回复进度时，单凭到达顺序难以一眼确定「Agent 的哪一条回复是在回应用户的哪一条消息」。
  3. **多频道 / 重放混合**：`poll` 一次返回多个频道与历史 `replayed: true` 消息时，仅看数组顺序容易误判最新意图。

## 2. 潜在解决思路（供后续讨论与迭代）
1. **排序基准统一**：`agyWorker` `/chat/poll` 返回 `received` 数组时，按 `payload.sentAt` 升序排列（相同时间戳再按 `seq` 排序），并附带 `oldestSentAt` / `newestSentAt`。
2. **显式引用关联（`replyToMsgId`）**：发送消息协议增加可选字段 `replyToMsgId`（或数组 `replyToMsgIds`），Agent 回复用户某条具体消息时显式带上原消息的 `msgId`，前端即可精确高亮/串联「问-答」配对。
3. **发送端单调序号（`clientSeq`）**：同一发送端维护递增 `clientSeq`，即使两条消息在同一毫秒内发出，也能严格区分先后。
4. **消费游标（`sinceSentAt` / `lastAckSeq`）**：支持按游标拉取，避免重放消息与新消息混杂。
