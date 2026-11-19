# 实时联调与 PWA HTTPS / EMQX 修复总结

本次通过 Cloudflare Worker (`agentplayground.maboco-mark.workers.dev/mqtt/rpc`) 与 EMQX (`e9c769c2.ala.cn-hangzhou.emqxsl.cn:8883`) 完成了实时双向联调，逐步定位并解决了 EMQX 自动连接、PWA Leaf Certificate 签发部署、iOS PWA 图标与灵动岛适配，以及手机端/桌面端多行输入与换行显示问题。

## 1. 已推送的 Verified Bot Commits（分支 `a2`）

| Commit | 说明 | 关键修改文件 |
| --- | --- | --- |
| `bdeaf79` | 修复 React StrictMode 卸载后未清空 `emqxClientRef.current` 导致二次挂载跳过自动重连的问题 | `apps/p2p-remote-controller/src/App.tsx` |
| `34350d7` | 使用 `setTimeout(..., 0)` + `clearTimeout` 延迟自动连接，避免 StrictMode 首次卸载强行中断 `CONNECTING` 中的 WebSocket 并产生报错 | `apps/p2p-remote-controller/src/App.tsx` |
| `9810eb4` | 为桌面端 Chat 消息与 Pending Reply 卡片添加 `white-space: pre-wrap; word-break: break-word;` 保留换行 | `apps/p2p-remote-controller/src/components/EmqxBridgePanel.tsx`, `apps/p2p-remote-controller/src/styles/app.css` |
| `00addef` | 复用 Tauri 应用图标作为 PWA 图标（512x512 / 256x256），并为独立模式 Pocket Deck 添加 iOS `env(safe-area-inset-*)` 灵动岛与底部横条安全区留白 | `apps/p2p-remote-controller/public/apple-touch-icon.png`, `apps/p2p-remote-controller/public/icon-256.png`, `apps/p2p-remote-controller/index.html`, `apps/p2p-remote-controller/public/manifest.json`, `apps/p2p-remote-controller/src/components/PocketDeckView.tsx` |
| `1585352` | 手机端 Pocket Deck 改用多行 `<textarea>`、添加 `white-space: pre-wrap`，并为 `ask_user` 精简为 `Send Reply` / `Dismiss` 按钮 | `apps/p2p-remote-controller/src/components/PocketDeckView.tsx` |
| `3f1983e` | 手机端独立模式 Pocket Deck 按键盘 Return/换行不再误触发发送，改为插入换行并由 Send 按钮提交 | `apps/p2p-remote-controller/src/components/PocketDeckView.tsx` |

## 2. PWA Leaf Certificate 签发与部署（已验证）

- **Root CA**：`MaBoCo_P_V1.1.crt` + `dec_MaBoCo_P_V1.1.pem`
- **关键参数**：
  - 有效期：`365` 天（`Nov 19 2026 GMT` – `Nov 19 2027 GMT`，满足 Apple `<= 825` 天及推荐 `<= 398` 天限制）
  - 扩展：`basicConstraints = critical, CA:FALSE`、`keyUsage = critical, digitalSignature, keyEncipherment`、`extendedKeyUsage = serverAuth`
  - SAN：`DNS:mark.ma, DNS:*.mark.ma, DNS:localhost, IP:127.0.0.1`
- **验证结果**：`openssl verify -CAfile MaBoCo_P_V1.1.crt leaf.crt` 返回 `leaf.crt: OK`，已放置于 `/Users/seqma/Library/Application Support/com.antigravity.p2premotecontroller/certs`（`leaf.crt` + `leaf.key`），LAN Sharing 状态已确认为 `HTTPS / WSS Active`。

## 3. Live Communication 机制确认

- 当 Agent 通过 `/mqtt/rpc` 挂起长轮询（`waitMs` 最长 50s）时，`ask_user` 与 `chat` 均可实现 ~200ms 级别实时送达；若 Agent 处于两轮间隙，EMQX 通过 `cleanSession: false` 与 `QoS 1` 缓存消息直至下一轮拉取。
- 传输层全程走 TLS（Mac 端 `wss://...:8084/mqtt`，Worker 端 `8883` MQTT over TLS），第三方网络抓包无法读取明文（但 EMQX Broker 服务端本身可见 payload）。
