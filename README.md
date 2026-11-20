# Live Chat 验证与移动端 E2EE / Pocket Deck 架构评估报告

## 1. Live Chat Worker (`agentlivechatchannel`) 评估与实测

- **`/help` 与 `/version` 审查结论**：协议设计完整、规范，无需修改 Worker，可直接投入使用。
  - 自动分配隔离频道：`ch_897a432c`
  - Topic 规范：`agy_p2p_v1/cmd/ch_897a432c/<command>`、`agy_p2p_v1/resp/ch_897a432c/<requestId>`、`agy_p2p_v1/chat/ch_897a432c`
  - 端点完备：`/help`、`/version`、`/cmd`（支持 `stream: true` NDJSON 心跳流，最长 `waitMs: 300000`）、`/chat/send`、`/chat/poll`、`/mqtt/rpc`
  - `supportedCommands` 覆盖 23 个核心指令（`ping`、`get_status`、`scan_window`、`perform_action`、`execute_javascript`、`initialize_aistudio`、`ask_user` 等）。
- **连通性实测**：
  - `POST /chat/send` 成功发送问候消息（`ok: true`, `connackCode: 0`, `pubackedCount: 1`, 耗时 895ms）。
  - `POST /cmd`（`ping`, `stream: true`）成功建连、订阅响应 Topic 并持续输出心跳流（15s 内未收到响应是因为 Mac 端尚未在 `ch_897a432c` 审批/应答）。

## 2. 仓库现状盘点（`Antigravi-ty/agyPreviewInstant`）

已运行 `pnpm radar -- --audit`（291 files，Cycles=0，LayerViolations=0，全部通过）。

### 已具备能力
1. **Pocket Deck 离线外壳缓存**：`apps/p2p-remote-controller/public/sw.js` 已实现壳资源与同源静态资产预缓存、导航离线回退，以及 `MANUAL_REFRESH_CACHE` 手动刷新。
2. **移动端指令与正则匹配**：`PocketDeckDirectTab.tsx` 已包含 `Open New (Mobile)`、`Focus Window`、`Refresh Page`、`Reload Default`、`Scan AX`、`Press Regex`（`targetRegex`）及 URL 预设；`emqxCommandHandlers.ts` 与 `usePocketDeckBridge.ts` 已支持 `targetRegex` + `targetIndex` 匹配。

### 待补齐的核心缺口
1. **外网/蜂窝网络回落链路**：`usePocketDeckRemoteSync.ts` 与 `dispatchPocketDeckAction` 目前仅访问局域网 `/api/state`、`/api/action` 与 `ws://host/ws`，外网环境下只能读取 `localStorage` 缓存快照，尚无 WebRTC P2P 或 EMQX Relay 连接能力。
2. **E2EE 与双向鉴权（UUID Token + TOTP）**：目前 EMQX 载荷为明文 JSON，缺少端到端加密、UUID Access Token 鉴权以及连接/操作级 TOTP 校验。
3. **AI Studio `partial` 状态回退**：`aiStudioWindowState.ts` 中 `networkConfigured = networkVerified || (previous?.networkConfigured ?? false)` 依赖内存态 `previous`；当 network 面板在 AX Tree 中仅显示单行且发生重扫/状态重置时，会退回 `partial`。

## 3. 技术方案评估与建议

1. **P2P vs Rust UDP vs EMQX Relay**：
   - **优先 WebRTC DataChannel**：iOS Safari 原生支持，自带 DTLS-SRTP 加密，低频十几 KB 数据能耗极低；比自研带 ACK 的 UDP 或 Rust UPnP 更省事且兼容蜂窝网络。
   - **降级 EMQX E2EE Relay**：当严格 NAT 导致 ICE 打洞失败时，自动回落到 EMQX 中转，并在应用层使用 AES-256-GCM 加密载荷（Broker 不可见明文）。
2. **ICE Token 防篡改与 TOTP 鉴权**：
   - 浏览器端无法直接访问 HTTPS 证书私钥，因此不建议用 HTTPS 证书做应用层签名。
   - 推荐在局域网首次配对时，由 Mac 端下发 **UUID Access Token + TOTP Secret + E2EE 主密钥/ECDH 公钥指纹** 至 iOS 端持久化；外网建连时通过 **ECDH 临时密钥交换 + HKDF** 派生会话密钥，并对 ICE 信令及每次操作（`perform_action`、`execute_javascript` 等）强制校验 **TOTP**；只读状态拉取可限定在 `source: "mobile"` 创建的窗口内免 TOTP。
3. **轮播图动态 ID 与 AI Studio `partial` 调试**：
   - 轮播图按钮可继续强化 `targetRegex` + `role` + 父容器锚点定位。
   - `initialize_aistudio` 成功调整 network 后，应在 Mac 端按 `windowId` 记录 `networkConfigured` 标志（页面 reload 时清除），并可通过 Live Chat 用 `execute_javascript` 实测直接读取 network 面板实际状态。

## 4. 下一步建议

- 专属 Live Chat 频道 `ch_897a432c` 已就绪。若你准备就绪，我们可以直接在 `p2p-remote-controller` 中通过 Live Chat 联调带轮播图的页面操作与 AI Studio `partial` 探测；如需为手机端隔离新建 EMQX 账号与 Topic 权限，也可随时提供新凭据继续开发。
