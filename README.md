# 问题排查、Context 消耗分析与 `8ec04ad` 修复报告

## 1. 为什么上一轮突然结束且没收到“拜拜”？
- **并非 Context Window Overflow 或系统强行重置**，而是我在上一轮收尾时认为主要任务已完成，主动结束了对话。
- 没收到 Chat 里的“拜拜”，是因为我最后一次发告别消息时给 Worker 传了 `"waitMs": 500`。而从 Cloudflare Worker 到 EMQX (`8883`) 的 **TLS + MQTT `CONNACK` 握手通常需要 ~900–1200ms**，结果 500ms 超时直接断开了连接（`connackCode: null, pubackedCount: 0`），导致告别消息根本没发出去。
- 本轮已用 `waitMs: 5000` 重新发送了打招呼与修复说明，并确认已投递。

## 2. 手机端消息本身不显示换行的原因与修复 (`8ec04ad`)
- **原因**：`apps/p2p-remote-controller/src/components/PocketDeckView.tsx` 第 380 行渲染 Agent Chat 消息体的 `<div>{msg.text}</div>` 漏掉了 `white-space: pre-wrap`，导致输入时虽然带 `\n`，但显示时被 HTML 默认折叠成空格。
- **修复**：已补充 `style={{ whiteSpace: "pre-wrap", wordBreak: "break-word" }}` 并推送至 `a2` 分支（commit **`8ec04ad`**）。`git pull` 后在手机端右上角点一下 **Update** 刷新 Service Worker 缓存即可生效。

## 3. Context Window 为什么会消耗到 ~4M？（以及如何极简接入降低消耗）
4M 是**多轮工具调用累积输入 Token** 的总和（每轮都会把完整对话历史 + 之前所有工具输出重新喂给模型），主要消耗在：
1. **每次 `/mqtt/rpc` 返回的 `recentUserActions` 体积很大**（几十条 AX 事件 JSON，每条 ~200B，一次就是几 KB）；
2. **多次等待超时与重拉**（每一轮 `execute_bash` 都会产生一次模型回合，把前面所有历史再重算一遍，轮次越多呈平方级增长）；
3. **前期探索 Worker 源码与仓库结构**（读取 Cloudflare Worker 脚本、多个 TSX/Rust 文件）。

### 下次开新窗口最省 Token 的描述方式
以后在新窗口直接给 **1 份紧凑连接卡片** 即可（不需要贴历史对话，也不需要让 Agent 去查 Cloudflare Worker 源码）：
```text
Repo: https://github.com/Antigravi-ty/agyPreviewInstant (branch: a2)
EMQX RPC Worker: POST https://agentplayground.maboco-mark.workers.dev/mqtt/rpc
- host: e9c769c2.ala.cn-hangzhou.emqxsl.cn, port: 8883
- auth: CNremoteController / weqgum-2qijmY-didnem
- clientId: agy_cloud_agent_v1 (cleanSession: false)
- topics: 发指令 agy_p2p_v1/cmd/ask_user | 发聊天 agy_p2p_v1/chat | 收回复 agy_p2p_v1/resp/#, agy_p2p_v1/chat
- 注意: waitMs 至少 >= 3000（发消息也不要用 500ms）；长轮询用 waitMs: 50000, returnOnFirstMessage: true。
```
这样 Agent 第一步就能直连 Worker，省掉读 Worker 源码和踩 `500ms` / `resp/#` 坑的轮次；如果要进一步压低 Token，后续也可以在本地或 Worker 侧裁剪 `recentUserActions`（或只在 `ask_user` 需要时附带）。
