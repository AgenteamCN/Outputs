# P2P Remote Controller：EMQX 断连（1005）、自动重连与手机端离线判定修复报告

**Commit**: `5ed3f5489386d5023505af1296ad6f1909d919ec`（分支 `a2`，`signature.isValid: true`）  
**URL**: https://github.com/Antigravi-ty/agyPreviewInstant/commit/5ed3f5489386d5023505af1296ad6f1909d919ec

---

## 一、`Closed (code 1005)` 是什么？为什么经常断连？

1. **`1005`（`No Status Received`）的含义**：
   - 根据 RFC 6455 WebSocket 协议，`1005` 不是服务端发来的具体错误码，而是客户端 WebView / 浏览器在 **底层 TCP/TLS 连接被直接断开、但没有收到对方发送带状态码的 WebSocket Close 帧** 时自动合成的关闭码。
2. **结合你的日志时间线看根因**：
   - 你的日志里出现了 `16:34:26 -> 17:02:47`（约 28 分钟）和 `17:02:48 -> 02:06:00`（约 9 小时）后触发 `Closed (code 1005)`。
   - 主要原因有三个：
     1. **macOS 休眠 / App Nap / 网络切换**：Mac 休眠或切换 Wi-Fi 后，JS 定时器暂停，EMQX 收不到 30s KeepAlive 心跳（`1.5 × 30s = 45s` 超时），EMQX 直接断开死链接；Mac 唤醒后 WebView 检测到 socket 已死，抛出 `code 1005`。
     2. **网关 / 代理 / EMQX Serverless 空闲回收**：云端负载均衡或 NAT 网关在长连接静默/半开状态下静默丢弃 TCP 映射。
     3. **旧代码缺少自动重连与心跳超时检测**：`apps/p2p-remote-controller/src/services/emqxClient.ts` 和 `useEmqxConnection.ts` 此前只在 `ws.onclose` 时打印 `DISCONNECTED: Closed (code 1005)`，**没有任何自动重连逻辑**，也没有检查 `PINGRESP` 是否按时返回。

---

## 二、方案评估：“交错双通道（5 分钟双长连接）” vs “持久化会话 + 自动重连”

你提到的思路是：**建立两条交错长连接（间隔 5 分钟）以防止断连漏消息**。从 MQTT 协议与实际网络工程来看，**不建议同时维持两条 WebSocket 长连接**，原因如下：
- **无法防住同机休眠/断网**：`1005` 绝大多数来自本机休眠或 Wi-Fi 波动，两条连接在同一台 Mac 上会**同时断开**。
- **双倍流量与配额消耗**：EMQX Serverless 按连接数、消息数和流量计费，双订阅会让所有下发消息、心跳包全部翻倍，且客户端必须对每条消息做去重。

**更好的原生 MQTT 替代方案（本次已直接落地）**：
1. **固定实例 Client ID + `cleanSession: false`（MQTT 持久化会话）+ QoS 1**：
   - 之前每次连接都调用 `generateRandomClientId()` 换新 ID 且 `cleanSession = true`，导致断连期间 EMQX 直接丢弃离线消息。
   - 现在同一运行实例在自动重连时**复用同一个 `clientId`** 并设置 `cleanSession: false`（订阅 QoS 1）。只要不是人工主动断开，短暂掉线期间发布到 `cmd/#`、`chat/#` 的 QoS 1 消息会由 **EMQX Broker 自动暂存**，重连成功瞬间立即补发，**无需双通道也不会漏消息**。
2. **自动重连 + 指数退避 + `PINGRESP` 看门狗 + 唤醒/联网事件监听**：
   - 发送 `PINGREQ` 后若 25s 内未收到 `PINGRESP`，主动关闭僵尸 socket 触发重连。
   - 监听浏览器 `online` 和 `visibilitychange` 事件，Mac 唤醒或网络恢复时**零等待立即重连**，否则按 `2s -> 4s -> 8s -> 15s` 退避重连。

---

## 三、EMQX 点对点独立新账户推荐配置

为了把 `p2p-remote-controller` 的远程控制/P2P 信令与主应用的 Telemetry 彻底隔离，建议在 EMQX 控制台单独创建一个专用账户：

- **推荐用户名（Username）**：`agy_p2p_bridge`（或 `p2p_remote_ctrl`）
- **推荐 Topic 前缀（Topic Prefix）**：`agy_p2p_v1`（与客户端默认值一致，涵盖 `agy_p2p_v1/cmd/#`、`agy_p2p_v1/resp/#`、`agy_p2p_v1/chat/#`、`agy_p2p_v1/cellular/req/#`、`agy_p2p_v1/cellular/res/#`）
- **推荐 ACL 权限模式（按顺序从上到下匹配，Allow + Deny 兜底）**：
  1. **Allow** | Action: `Publish & Subscribe` (`All`) | Topic: `agy_p2p_v1/#`
  2. **Deny** | Action: `Publish & Subscribe` (`All`) | Topic: `#`
- **连接注意事项**：Mac 端与手机端共用该用户名即可，两端均使用随机生成的独立 `Client ID`（`p2p_mac_...` 与 `pocket_cell_...`），不会互踢；只需确保 EMQX 未限制单用户名最大连接数 <= 1。

---

## 四、本次代码修复与功能落地（已推送到 `a2`）

### 1. EMQX 实时通信自动重连 & 持久化开关
- **`EmqxBridgeConfig`**（`apps/p2p-remote-controller/src/types/emqxBridge.ts`、`utils/emqxBridgeUtils.ts`）：
  - 新增 `autoReconnect: boolean`，**默认 `true`**，通过 `localStorage`（`p2p_emqx_bridge_config_v1`）持久化落盘。
- **`EmqxWebSocketClient`**（`services/emqxClient.ts`）：
  - 支持 `cleanSession` 参数，并新增 `PINGRESP` 看门狗（超时主动 `4000: PINGRESP timeout` 关闭僵尸连接）。
- **`useEmqxConnection`**（`hooks/useEmqxConnection.ts`）：
  - 区分手动断开与异常断连；异常断连且 `autoReconnect === true` 时，复用同实例 `clientId` + `cleanSession: false` 自动指数退避重连，并监听 `online` / `visibilitychange` 快速恢复。
- **UI Toggle Switch**：
  - 在 `EmqxBridgePanel.tsx` 顶部状态栏和 `EmqxBridgeSettingsCard.tsx` 设置卡片中均添加了 **Auto-Reconnect** 开关。

### 2. 手机端（Cellular P2P）主机在线判定 & 自动重连
- **修复“主机不在线却误显示走 EMQX Relay”的问题**（`utils/cellularP2pConfig.ts`、`hooks/useCellularP2pClient.ts`）：
  - 新增 `resolveCellularFallbackOutcome` 纯函数与 `hostRespondedRef` 状态追踪。
  - 手机端收到来自主机的 `webrtc_answer`、`webrtc_ice (from: "host")`、`e2ee_rpc_res` 或 DataChannel 打开时，才标记 `hostResponded = true`；在 `force_relay_only` 模式下也会先发送加密 `ping` 验证主机存活。
  - 超时（5.5s）后若 `!hostResponded`，**不再切换为 `emqx_cn_relay`**，而是保持 `disconnected` 并明确提示：
    `Host offline (no response from host via EMQX)`。
  - 仅当 **主机有回应（在线）但 WebRTC DataChannel 无法建立（NAT/网络状况限制）** 时：
    - `auto` 模式回退到 `emqx_cn_relay`（提示 `Host online — EMQX CN E2EE Relay active (P2P NAT fallback)`）；
    - `force_p2p_only` 模式保持 `disconnected`（提示 `Host online, but P2P Direct failed (network/NAT; Relay disabled)`）。
- **手机端持久化 Auto-Reconnect 开关**：
  - `MobileCellularSavedAuth` 新增 `autoReconnect`（默认 `true`，落盘到 `p2p_cellular_mobile_auth_v1`）。
  - 在 `PocketDeckOutputsTab.tsx` 添加 **Auto-Reconnect** 开关；仅在已建立连接后意外掉线时触发自动重连（主机本来就离线时不空转重连）。

---

## 五、验证结果

- `pnpm --filter p2p-remote-controller check`（`tsc --noEmit && eslint src`）：通过（0 errors）
- `pnpm --filter p2p-remote-controller test`（`vitest run`）：**12 个测试文件、47 个测试用例全部通过**（含新增 `emqxBridgeUtils.test.ts` 持久化测试与 `cellularP2pConfig.test.ts` 主机离线/Relay 判定测试）
- `pnpm --filter p2p-remote-controller build`（`tsc && vite build`）：构建成功
