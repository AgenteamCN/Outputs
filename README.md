# 任务完成报告（分支 `a2` · Commit `9d83956`）

## 1. 关于 `recentUserActions` 携带十几条 AX 事件的机制说明

**结论：是的，只要你在 Remote Controller（或 Pocket Deck）里点击审批/回复，或者在 Agent Chat 里发送消息，Mac 端就会主动把增量 AX 事件打包进那条 MQTT 消息里，并不是 `/mqtt/rpc` 网关自己附加的。**

具体代码路径在 `apps/p2p-remote-controller/src/App.tsx` 与 `apps/p2p-remote-controller/src/utils/emqxBridgeUtils.ts`：

1. **两个触发入口**：
   - `handleEmqxDecision`（点击 Approve / Deny / Reply 按钮发送 `.../resp/<requestId>` 时）
   - `handleSendEmqxChat`（在 Agent Chat 发送聊天消息到 `.../chat` 时）
2. **数据筛选与条数上限（`buildPiggybackTimelineDiff`）**：
   - 每次发送前，Mac 端会从 `getTimelineEvents()` 取出 `id > lastSyncedTimelineIdRef.current` 的所有未同步 AX / 用户操作事件，发送后更新 `lastSyncedTimelineIdRef.current`。
   - 条数上限由 `defaultLevel` 决定：**默认 `compact` 模式最多携带最近 15 条**（`standard` 40 条、`full` 100 条），所以通常就是“十几条 AX 事件 JSON，单条消息几 KB”。
   - 如果在刚重启应用后（`lastSyncedTimelineIdRef.current` 重置为 `0`），或两次回复之间浏览器/AX 产生了较多事件，第一次回复/发消息就会把最近最多 15 条事件顺带捎过来；如果连续发两条消息且中间没有新 AX 事件，第二条的 `recentUserActions` 就会是 `[]`。

---

## 2. Notifications & Chat 持久化最近 200 条消息

为避免新提交代码、重启 Mac 端后丢失交流上下文，已在 `p2p-remote-controller` 中将 `Notifications & Chat` 消息列表持久化到 `localStorage`（上限 **200 条**）：

- **`apps/p2p-remote-controller/src/utils/emqxBridgeUtils.ts`**：
  - 新增 `MAX_PERSISTED_CHAT_MESSAGES = 200` 及存储键 `p2p_emqx_bridge_chat_v1`。
  - 新增 `loadEmqxChatMessages()`（启动时反序列化并校验字段合法性，截断保留最近 200 条）与 `saveEmqxChatMessages()`（写入最近 200 条）。
- **`apps/p2p-remote-controller/src/App.tsx`**：
  - `chatMessages` 状态改为通过 `loadEmqxChatMessages()` 惰性初始化。
  - 新增 `useEffect` 在 `chatMessages` 变化时自动调用 `saveEmqxChatMessages(chatMessages)`（点击 `Clear` 清空时也会同步清空持久化存储）。
  - `appendBridgeMessage` 的内存保留上限从 150 条提升至 `MAX_PERSISTED_CHAT_MESSAGES`（200 条）。

---

## 3. 移动端 Pocket Deck 右上角 Commit ID 与相对时间修复

**原问题原因**：`PocketDeckView.tsx` 原先显示的是 `#{activeSnapshot.commitId || commitId}`。其中 `activeSnapshot` 是 Mac 宿主机通过 `/api/state` 和 `/ws` 实时推送过来的状态，导致手机端在未点击 `Update` 刷新 PWA 缓存的情况下，依然会显示 Mac 端最新 check out 的 commit ID，从而造成“以为已经自动更新”的误导。

**本次改动**：
- **`apps/p2p-remote-controller/vite.config.ts` & `src/vite-env.d.ts` & `src/services/lanService.ts`**：
  - 构建/打包前端时通过 `git show -s --format=%ct HEAD` 注入 `__APP_COMMIT_TIME__`（若无 git 上下文则回退到构建时间），并提供 `getAppCommitTime()`。
- **`apps/p2p-remote-controller/src/utils/emqxBridgeUtils.ts`**：
  - 新增 `formatRelativeTimeCompact(isoTime, nowMs)`，将提交时间格式化为 `just now`、`15m ago`、`3h ago`、`2d ago`。
- **`apps/p2p-remote-controller/src/components/PocketDeckView.tsx`**：
  - 右上角改为**始终显示移动端当前实际运行包自身的 `commitId` 与相对时间**（例如 `#9d83956 · 5m ago`），每 60 秒自动刷新一次相对时间。
  - 当宿主机推送的 `activeSnapshot.commitId` 与手机当前运行的 `commitId` 不一致时，不再篡改显示的 commit ID，而是高亮 `Update` 按钮（`className="primary"`）并在 tooltip 中提示宿主机当前版本，方便明确判断是否需要手动点击 `Update`。

---

## 4. 验证与提交记录

- **检查与测试**：
  - `pnpm --filter p2p-remote-controller check`（`tsc --noEmit` + `eslint src`）：0 errors
  - `pnpm --filter p2p-remote-controller test`（Vitest）：2 个测试文件、10 个用例全部通过（新增 200 条消息持久化截断测试与相对时间格式化测试）
  - `pnpm --filter p2p-remote-controller build`（`tsc && vite build`）：构建成功
- **Git 提交**：
  - 仓库与分支：`Antigravi-ty/agyPreviewInstant` (`a2`)
  - Verified Bot Commit：[`9d83956`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/9d83956a1352c6f042d84bbd9495d10aa9f4264c)（`signature.isValid: true`）
