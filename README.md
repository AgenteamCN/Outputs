# 实施汇报：Plan Alpha 修复上线、本地 HTTPS + Pocket Deck 推送 & 架构深度解析

---

## 一、已完成并上线的两项核心变更

### 1. Plan Alpha：Cloudflare Worker (`agentplayground`) 热修复已部署
- **部署状态**：已通过 Cloudflare API 更新 `agentplayground`（`modified_on: 2026-11-19T01:47:15Z`）。
- **在线验证**：`GET https://agentplayground.maboco-mark.workers.dev/version` 返回：
  ```json
  {"ok":true,"worker":"emqx_worker","plan":"alpha-v1","burstFix":true}
  ```
- **修复内容**：
  - 修复了 `emqx_worker.mjs` 中 `burstDeadline = Math.min(deadline, Date.now() + 250)` 导致永不提前 `break` 的 Bug，改为记录首次收到消息的 `firstReceivedAt`，在 `burstMs`（默认 `150ms`）或 `returnOnFirstMessage: true` 时**立即退出并返回结果**，不再空等剩余的 55 秒。
  - 响应中新增 `elapsedMs` 字段，便于观测真实往返耗时。

### 2. 本地 HTTPS Server + Pocket Deck + Mobile Viewport + 离线版本锁定已推送至 GitHub
- **GitHub Verified Bot Commit**：[`fe4e928`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/fe4e928a6389fb161c8ff6bbf741cb908e06f63b)（分支 `a2`，`signature.isValid: true`）。
- **新增与改造内容（`apps/p2p-remote-controller`）**：
  - **本地可选 HTTPS/WSS + HTTP/WS 服务端（`src-tauri/src/lan_server.rs`）**：
    - 使用 `hyper` + `tokio-rustls` + `rustls-pemfile` 实现，默认监听 `0.0.0.0:14210`。
    - 自动检测证书目录（优先 `apps/p2p-remote-controller/src-tauri/certs/`，已加入 `.gitignore`；打包后回退至系统应用配置目录 `certs/`）。若存在 `leaf.crt` 与 `leaf.key` 则自动启用 **HTTPS + WSS**，否则平滑降级为 HTTP + WS 并在界面提示证书路径。
  - **LAN Sharing 极简设置弹窗（`src/components/LanSharingModal.tsx`）**：
    - 一键开启/关闭 `0.0.0.0` 局域网暴露、端口设置、家用信任模式（`Open LAN` 免 PIN 或 `6-Digit PIN`）、TLS 状态与证书目录一键复制、Pocket Deck 访问 URL。
  - **Pocket Deck 竖屏双模式终端（`src/components/PocketDeckView.tsx`）**：
    - **Bridge Sync**：同步展示待审批队列（支持 `Approve` / `Reply & Approve` / `Reply Only` / `Deny`）与 Agent 实时对话。
    - **Direct Control**：竖屏专用 `Controls` / `Timeline` / `Console` 三合一控制台。
  - **Mobile Viewport 独立窗口**：
    - 顶部工具栏新增 `Mobile Viewport` 按钮，调用 `lan_open_mobile_viewport` 打开 `390x844` 独立 Tauri 窗口，实时同步展示 Pocket Deck 布局与状态。
  - **Commit ID 锁定 + 不自动刷新的离线 PWA（`public/sw.js` + `vite.config.ts`）**：
    - 构建时注入 `__APP_COMMIT_ID__`，Service Worker 采用严格 Cache-First 策略，首次加载后即使断网或切换网络也**永不自动刷新**；仅当用户手动点击右上角 `Update` 按钮时才清理旧缓存并拉取新版本。

---

## 二、问题解答 1：本地 HTTPS 部署需要哪些证书文件？

除了密钥对（RSA 或 ECDSA）之外，你需要向 `apps/p2p-remote-controller/src-tauri/certs/`（已 `.gitignore`，不会上传）放入以下 **2 个 PEM 文件**：

1. **`leaf.crt`**（叶子证书 + 可选中间证书链，PEM 格式，`-----BEGIN CERTIFICATE-----`）
   - **关键要求（iOS Safari 强制校验）**：签发 `leaf.crt` 时**必须包含 `SubjectAlternativeName` (SAN)**，把你的 Mac 局域网 IP 和本地域名都写进去，例如：
     ```ini
     subjectAltName = IP:192.168.1.100, IP:127.0.0.1, DNS:localhost, DNS:your-mac.local
     ```
   - 有效期建议不超过 **825 天**（Apple 对 TLS 叶子证书的硬性上限，建议 365 天）。
2. **`leaf.key`**（叶子证书对应的私钥，PEM 格式，`-----BEGIN PRIVATE KEY-----` 或 `-----BEGIN RSA PRIVATE KEY-----`）
   - **不要设置密码（Passphrase）**，否则服务端后台启动时无法无交互读取。
3. **iOS 设备侧操作**：
   - 将你的 **Root CA 公钥证书**（不要放私钥！）安装到 iPhone/iPad 上，并在「设置 -> 通用 -> 关于本机 -> 证书信任设置」中打开针对该 Root CA 的完全信任开关即可。

---

## 三、问题解答 2：是否需要同步修改 Cloudflare Pages？

- **纯局域网使用 + 外出离线操作/审批（通过缓存好的 PWA + EMQX/Worker 通信）**：**不需要**单独修改或部署 Cloudflare Pages。Mac 本地的 HTTPS Server 已经提供了完整的 `manifest.json` 和 `sw.js`，iOS 在家连 Wi-Fi 安装到主屏幕后，页面壳会被当前 Commit ID 永久锁存在本地缓存中；出门后不刷新页面即可继续使用。
- **只有以下场景才需要 Cloudflare Pages（或 Cloudflare Tunnel）**：
  1. 你在外面**没有提前在家里安装过 PWA**，需要通过公网 HTTPS 地址直接打开；
  2. 你需要 **iOS Web Push 锁屏弹窗通知**（Apple Push 要求推送源与已安装的 HTTPS PWA Origin 匹配，用 Cloudflare Pages 或 `cloudflared` Tunnel 绑定固定 HTTPS 域名最稳妥）。
  - 即便部署到 Cloudflare Pages，也**复用同一套 `apps/p2p-remote-controller` 前端构建产物（`dist/`）**，无需维护两套代码。

---

## 四、问题解答 3：Worker 10ms CPU 限制 & 使用 Durable Objects (Plan Bravo) 的区别

### 1. 关于 Cloudflare Worker 10ms CPU 限制
- Cloudflare Workers Free 计划限制的是 **10ms 纯 CPU 计算时间**，**不包含** `await reader.read()` 或 `fetch()` 等待网络 I/O 的时间。
- 因此在 Plan Alpha 中哪怕 `waitMs` 设到 55 秒，只要没有在循环里死算，实际消耗的 CPU 通常只有 **1–3ms**，不会触发 10ms CPU 超限。

### 2. 使用 vs 不使用 Durable Objects (DO) 的核心区别

| 维度 | 不使用 DO（当前 Plan Alpha 无状态 Worker） | 使用 DO（Plan Bravo：`MqttBridgeDO`） |
| :--- | :--- | :--- |
| **连接模型** | 每次 `POST /mqtt/rpc` 重新 TCP+TLS+MQTT 握手，结束即断开 | DO 实例在内存中长期持有与 EMQX 的**单一持久连接**（或直接做 WebSocket Hub） |
| **休眠与计费** | 每次请求必须完整跑完握手流程 | 支持 **WebSocket Hibernation API**：空闲连接不断开，休眠期间**不占 CPU 配额、不计费** |
| **端到端延迟** | 每次建连耗时约 300ms–1500ms；Agent 思考间隙离线 | **0ms 握手延迟**；消息实时缓存在 DO 内存，收到回复微秒级唤醒返回 |
| **iOS Web Push** | Worker 断开后无法感知新消息，无法主动推送 | DO 持续在线，收到 `ask_user` / 待审批指令时可**立即触发 Web Push** |
| **门槛** | **免费**（Free 计划可用） | 需 **Workers Paid 计划（$5/月起）** |

---

## 五、问题解答 4：Plan Charlie（Fly.io / Railway）是免费服务吗？

- **Fly.io**：**已不再提供无限制的永久免费套餐**。新账号通常需绑定信用卡，按实际资源用量计费（例如一台最小 `shared-cpu-1x` / 256MB RAM 的常驻容器约 **$1.94/月**）。
- **Railway**：**非永久免费**。提供 30 天（或 $5 额度）免费试用，之后 Hobby 计划最低 **$5/月**（包含 $5 用量额度）。
- **零成本替代方案**：
  - 既然你的 Apple Silicon Mac 本身就在家里常驻运行，配合 **Plan Alpha（免费 Cloudflare Worker）** 解决 55 秒卡死问题；或者在本地 Mac 上运行 **Cloudflare Tunnel（`cloudflared`，完全免费）** 将本地 HTTPS/WS 服务映射为公网 HTTPS 域名，即可实现 **0 元月费**的永久长连接与外网访问。
