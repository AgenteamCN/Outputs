# 任务完成报告：Live Chat Worker 修复、超时配置统一与 Mobile Viewport 聊天增强

## 1. `agentlivechatchannel` Cloudflare Worker 修复与升级（已部署 `live-chat-v2`）

已从 Cloudflare Workers 拉取并诊断 `agentlivechatchannel.maboco-mark.workers.dev` 源码，定位并修复了导致“掉线 / 串台 / 重复收发 / 参数丢失 / 超时不明确”的根本原因，并已直接部署上线（保留全部 10 个原有环境变量绑定）：

1. **修复共享 `clientId` + `cleanSession: false` 导致的掉线与跨频道串台**：
   - **根因**：此前 `/chat/send`、`/chat/poll`、`/mqtt/rpc` 均共用同一个 `clientId: "agy_cloud_agent_v1"` 且 `cleanSession: false`。任何一次 `/chat/send` 或其他 Agent 轮询都会在 EMQX 上顶掉当前会话（`MQTT_SOCKET_CLOSED`），并把其他频道（如 `#ch_4a336536`）的离线队列消息带出。
   - **修复**：
     - `POST /chat/send` 改用每次请求唯一的临时 `clientId`（`agy_send_<uuid>`）与 `cleanSession: true`，发送后不再踢掉活跃轮询，也不再返回任何残留消息（实测 `received: []`）。
     - `POST /chat/poll` 按 `channel` 隔离持久化会话（`agy_poll_<sanitizedChannel>`），避免跨频道串台；同时默认将 `excludeSender` 设为 `[defaults.clientId, "agent"]`，并在单次会话内按 `(topic, payload)` 去重，防止 Agent 收到自己发出的消息。
2. **修复 `POST /cmd` 参数剥离问题**：
   - **根因**：此前 `POST /cmd` 仅展开 `body.payload`，导致顶层的 `windowId`、`level`、`regexFilter`、`maxChars`、`script`、`action`、`targetRegex`、`targetIndex`、`nodePath` 等字段被丢弃。
   - **修复**：自动透传除 Worker 控制字段（`CMD_CONTROL_KEYS`）之外的所有顶层命令参数到 MQTT 命令负载。已在实时会话中通过 `windowId: 56122`、`level: "standard"`、`regexFilter`、`maxChars: 6000` 验证生效（9.1 秒返回 Google AI Studio 窗口 319 个元素中的 27 个匹配控件，含 `Get API key`、`API key selected`、`Select primary model`、`Select Antigravity agent harness model`、`System instructions`、`Run settings`、`Stop` 等）。
3. **保留 4 分 30 秒强制长轮询并规范超时返回**：
   - 维持 `FORCED_WAIT_MS = 270000`（不可通过传参覆盖），确保用户输入有充足时间到达。
   - 到达 270000ms 未收到响应时，Worker 必定退出并返回统一结构（非流式 HTTP `408`，流式 `type: "result"`）：
     - `ok: false`, `timedOut: true`, `reason: "timeout"`, `errorCode: "WAIT_TIMEOUT"`
     - `message`: 提示用户可能未在 4m30s 内输入，建议最多重试 3 次（`retryPolicy: { maxRetries: 3, waitMs: 270000 }`），若仍无响应再在报告中说明。
4. **完善 `/help` 与 `/version` 文档**：
   - 在 `/help` 中补充 `timeoutPolicy`、`POST /cmd` 全量透传参数说明、`POST /mqtt/rpc` 请求体 schema、以及 `stream: true` + `curl -sN --max-time 290` 示例。

---

## 2. 仓库改动（`Antigravi-ty/agyPreviewInstant`，分支 `a2`，Verified Commit `897185c`）

### 2.1 独立超时配置文件与流式输出集成
- 新增 [scripts/env/timeouts.json](scripts/env/timeouts.json)：统一定义硬超时 `300s`（5 分钟）、软超时 `270s`（4 分 30 秒）及 `streamOutput: true`。
- 新增 [scripts/env/timeouts.mjs](scripts/env/timeouts.mjs)：提供 `loadTimeoutConfig()`、`runWithSoftTimeout()` 及 `--print-env` / `--json` CLI 输出。
- 更新 [scripts/env/bootstrap.sh](scripts/env/bootstrap.sh)：通过 `timeouts.mjs --print-env` 读取软/硬超时配置，使用 `timeout --signal=TERM --kill-after=...` + `stdbuf -oL -eL` 与 `--reporter=append-only` 保证流式输出并在 5 分钟硬限制前收敛。
- 更新 [scripts/env/doctor.mjs](scripts/env/doctor.mjs)：读取并校验 `Timeout policy`，输出 `soft=270s (4m30s), hard=300s (5m00s), stream=true`。

### 2.2 Mobile Viewport 聊天字号/加粗与临时 Channel ID 快速跳转
- 更新 [apps/p2p-remote-controller/src/components/PocketDeckView.tsx](apps/p2p-remote-controller/src/components/PocketDeckView.tsx) 与 [apps/p2p-remote-controller/src/components/PocketDeckBridgeTab.tsx](apps/p2p-remote-controller/src/components/PocketDeckBridgeTab.tsx)：
  - **字号与加粗选项**：在 `Agent Chat` 顶部增加 `12 / 14 / 16 / 18 / 20` 字号按钮与 `B` 加粗开关（通过 `localStorage` 持久化），同步应用于聊天消息正文、审批提示文本及底部输入框。
  - **临时 Channel ID 自动选中与快速跳转**：
    - 频道栏新增 `Auto` 开关（默认开启），当新发现 `ch_...` 临时频道时自动选中。
    - 聊天消息头部的 `#ch_xxx`、审批卡片的 `#ch_xxx` 徽章、以及消息/审批正文中的 `ch_xxx` 均支持点击直接跳转切换，当前选中频道以高亮样式标识。

---

## 3. 验证结果
- `node scripts/env/doctor.mjs`：全部必需检查通过，OverlayFS 占用 `8 MB`（预算 `<= 100 MB`）。
- `pnpm --filter p2p-remote-controller test -- --run`：9 个测试文件、34 个测试用例全部通过。
- `pnpm --filter p2p-remote-controller build`：`tsc && vite build` 构建成功。
- Git 提交：通过 GitHub GraphQL `createCommitOnBranch` 提交至 `a2`（`897185cdea86cc0fb5adea091db3c13ac4f5115d`，`signature: VALID`）。
