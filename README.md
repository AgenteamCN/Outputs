# agyPreviewInstant 同步增强与 Cellular P2P 通道预演报告

**Commit**: `a81fedabf893f212955fe56169202389c3c6f434` (`Antigravi-ty/agyPreviewInstant` @ `a2`, Verified Bot Commit)

---

## 一、Part 1：AGY ↔ Remote Controller 同步增强、后台自动捕获与截断排查

### 1.1 Task Overview 备注栏与来源 URL 同步
- **后端与存储 (`src-tauri/src/models/interaction.rs`, `storage/task_registry.rs`, `commands/internal_channel.rs`, `commands/task_commands.rs`)**：
  - 在 `TaskRecord` 与 `TaskIndexEntry` 中新增 `notes: Option<String>` 字段，并在索引/详情合并逻辑中完整持久化。
  - `InternalImportPayload` 新增 `sourceUrl` 字段；Remote Controller 同步任务时自动将页面 URL 写入 `notes`（若任务已存在且 `notes` 为空则自动补全）。
  - 新增 `update_task_notes` Tauri 命令并在 `lib.rs` 注册，支持在 AGY 端手动编辑/保存备注。
- **前端 UI (`src/components/inspector/TaskOverviewNotesField.tsx`, `TaskOverviewSidebar.tsx`)**：
  - 在 Task Overview 中新增 **Notes / Source URL** 栏：自动识别备注中的 `https://...` 链接并提供 **Open**（直接调用系统浏览器打开原页面以便开启下一轮对话）、**Copy** 与内联 **Edit / Save** 功能。
- **Remote Controller 侧 (`apps/p2p-remote-controller/src/services/networkRecorderScript.ts`, `internalAgyChannel.ts`, `src-tauri/src/commands.rs`, `src/App.tsx`)**：
  - 捕获 `CreateInteraction` 时同步记录当时的 `location.href`（`pageUrl`），并通过 `send_interaction_to_agy` 的 `sourceUrl` 参数传递给 AGY。

### 1.2 Remote Controller 后台自动同步（无需打开 Network Tab）
- **原问题根因**：
  1. `createDefaultWindowNetState` 与窗口列表默认 `autoInject = false`，已打开的 Safari 窗口不会自动注入网络录制脚本。
  2. `useNetworkRecorderPolling` 仅在 `netRecording && selectedWindowId !== null` 时轮询**当前选中窗口**，后台未选中的窗口即使已注入也不会被轮询，导致其 `v1_...` 交互从未同步到 AGY。
- **修复方案**：
  - `safariWindowHelpers.ts` 将托管窗口默认 `autoInject` 设为 `true`（仍支持手动关闭）。
  - `useNetworkRecorder.ts` 新增后台自动注入 Effect：只要 `safariRunning && autoSyncToAgy` 为真，自动为所有可注入的托管窗口注入网络录制器。
  - `useNetworkRecorderPolling.ts` 新增后台多窗口轮询 Loop（每 3.5 秒遍历非当前选中但已开启录制的窗口），静默提取新发现的 `v1_...` interaction 并立即同步至 AGY，无需用户切换到 Network Tab 或逐个点击窗口。

### 1.3 User Input 被截断的原因与通信机制详解

#### 截断根因
在 `apps/p2p-remote-controller/src/services/networkRecorderScript.ts` 中：
- `extractCreateInteractionInput()` 末尾硬编码了 `.slice(0, 12000)`
- `addInteraction()` 中又一次执行了 `.slice(0, 12000)`

因此只要单次输入超过 **12,000 字符**（例如粘贴数万至数十万字符的报错日志或代码上下文），就会在 Safari 页面内脚本提取阶段被直接截断，后续链路根本收不到完整内容。
- **修复**：已移除上述两处 `.slice(0, 12000)` 限制，并在 `networkRecorderScript.test.ts` 中新增了 **30,000 字符**超长输入的回归测试用例（实测完整保留）。

#### 两个应用（Remote Controller ↔ AGY）目前的通信机制
两者目前通过**纯本地跨进程通道（Local Internal Channel）**通信，不经过任何外部网络或 EMQX：
1. **Safari 页面拦截层（JS Injection）**：Remote Controller 通过 macOS Apple Events（`osascript` `do JavaScript`）向 Safari 标签页注入 `window.__agyNetRecorder`，Hook `XMLHttpRequest` / `fetch`（同时拦截并屏蔽 `play.google.com` 等 tracker 请求）。当检测到 `CreateInteraction` 请求时，从 Protobuf-JSON 请求体 `parsed[3]` 递归提取用户输入文本，并从响应中提取 `v1_...` 交互 ID。
2. **Remote Controller 轮询与脱敏层（Tauri Frontend）**：`useNetworkRecorderPolling` 定期调用 `pollRecorderInWindow` 获取快照，经 `networkSanitizer.ts` 脱敏（擦除 API Key / GitHub Token 等敏感凭据）后，调用 `sendInteractionToAgyLocal()`。
3. **IPC 传输层（Unix Domain Socket + 文件 Inbox 双通道）**：
   - Remote Controller 的 Rust 后端（`send_interaction_to_agy`）优先连接 AGY 在应用配置目录创建的 **Unix Domain Socket**（`com.antigravity.previewinstant/internal_channel.sock`），发送单行 NDJSON（`{"interactionId", "input", "sentAt", "sourceUrl"}\n`）。该通道基于 Tokio `BufReader::read_line` 与 `serde_json`，**本身没有 12KB 文字上限**，可承载数 MB 级长文本。
   - 若 AGY 当时尚未启动或 Socket 不可用，则原子写入（先写 `.tmp` 再 `rename`）到 `com.antigravity.previewinstant/internal_inbox/<id>.json`，由 AGY 启动时通过 `drain_internal_inbox_sync` 消费。
4. **AGY 接收与状态广播层（`internal_channel.rs`）**：`ingest_internal_import` 将任务持久化至本地 `tasks` 注册表，并通过 Tauri Event `internal:import-task` 实时推送给 AGY 前端展示。

---

## 二、Part 2：Cellular P2P Channel 预演、鉴权体系与 Outputs 预览

### 2.1 已实现的预演功能模块
- **PC 常驻端（`useCellularP2pHost.ts`, `useEmqxConnection.ts`, `useEmqxBridge.ts`, `CellularP2pSettingsCard.tsx`）**：
  - 复用现有 EMQX CN 连接，监听 `{topicPrefix}/cellular/req/#`。
  - 收到手机端请求后，支持 **WebRTC `RTCPeerConnection` + `RTCDataChannel`** 建连；若处于对称型 NAT 无法直连，则无缝降级为 **EMQX CN Relay（AES-256-GCM E2EE 端到端加密）**。
  - 在 EMQX 设置面板中可管理：**Access UUID**（不区分大小写、一键轮换）、**TOTP 开关与 `otpauth://` URI**、**Passkey 接口预留开关**、**GitHub API Token**（提升限额）、**APN Token 持久化与 JSON 导出**，以及实时查看 **P2P Direct vs EMQX CN Relay 字节/消息统计**。
- **移动端 PocketDeck PWA（`useCellularP2pClient.ts`, `PocketDeckOutputsTab.tsx`, `PocketDeckView.tsx`, `public/gfm.css`, `public/sw.js`）**：
  - 新增 **Outputs** 标签页，支持**按需 Connect / Disconnect**（用完即断，最大限度节省 EMQX CN 配额）。
  - 顶部状态徽章明确区分 **`P2P Direct`**（绿色，零 EMQX 流量）与 **`EMQX CN (E2EE Relay)`**（黄色），并实时展示双方流量统计。
  - 支持检索 `AgenteamCN/Outputs` 的 **Commit List**，点击任意 Commit 即拉取 `README.md` 并渲染为干净的 **GFM HTML**（零 GitHub Tracker/Heartbeat 脚本），由 Service Worker 离线缓存 `/gfm.css`。

---

### 2.2 鉴权方案对比与推荐：UUID vs TOTP vs Passkey

| 维度 | 轮换 UUID（已实现） | TOTP 动态口令（已实现接口） | Passkey / WebAuthn（已预留接口） |
| :--- | :--- | :--- | :--- |
| **核心原理** | 高熵随机 128-bit 令牌，通过 URL Query / Header / 信令包携带；服务端不区分大小写比对，并派生 AES-256-GCM 密钥 | 基于共享密钥（Base32）+ 当前 30 秒时间窗口计算 `HMAC-SHA1` 6 位动态码，防重放 | 基于非对称密钥对（FIDO2 / WebAuthn）：私钥硬件级隔离在 iPhone Secure Enclave 中，通过 Face ID / Touch ID 对服务器下发的随机 `challenge` 进行 ECDSA P-256 签名 |
| **无感程度** | **极高（完全无感）**：首次扫码或粘贴后持久化在 PWA `localStorage`，后续请求自动携带 | **较低（有感）**：每次连接需切去 Authenticator 复制 6 位数（或依赖系统自动填充） | **高（生物识别无感）**：iOS Safari / PWA 弹窗直接面容 ID 一键通过，但**依赖 HTTPS + 固定可信域名（RP ID）** |
| **安全性** | 依赖信道加密（HTTPS/WSS + E2EE）；若设备丢失需在 PC 端轮换 UUID | 即使静态凭据泄露，攻击者也无法在无 TOTP 密钥时通过验证 | **最高**：私钥永不离开 iPhone，防钓鱼、防中间人，天然绑定域名 |
| **适用阶段** | **当前首选主力方案**（配合 E2EE 与定期轮换） | 作为高敏感操作（如远程执行危险命令）的**可选二次校验** | 适合未来绑定固定个人域名后的终极无感鉴权 |

**Passkey 落地注意事项**：WebAuthn 规范强制要求在 `https://` 且拥有稳定 `rpId`（域名）的环境下运行（不能是变动的内网 IP 或纯自签名未受信任证书），而目前 iOS 端与 Apple Silicon Mac 之间若通过 Apple ID 钥匙串同步，在固定 HTTPS 域名下即可实现跨设备无感登录。当前已在 `cellularP2p.ts` 中预留了 `PasskeyChallengeProvider` 与 `PasskeyAssertionPayload` 接口，后续可无缝接入。

---

### 2.3 获取 GitHub 公开仓库 Commit List 与渲染 Markdown 的备选方案（不使用 Cloudflare Workers）

1. **当前采用方案：PC Host 代理 + `api.github.com` + `raw.githubusercontent.com`**
   - Commit List：`GET https://api.github.com/repos/AgenteamCN/Outputs/commits`（未鉴权 60 req/hr，填入个人 GitHub PAT 后提升至 **5,000 req/hr**）。
   - README 原文：`GET https://raw.githubusercontent.com/AgenteamCN/Outputs/{sha}/README.md`（**不计入 REST API 60 req/hr 限额**，速度极快），在 PC 端直接渲染为纯洁 GFM HTML 返回给手机，手机无需开代理。
2. **备选方案 A：GitHub Atom / RSS Feed（零 API 限额、免 Token）**
   - 直接请求 `https://github.com/AgenteamCN/Outputs/commits/main.atom`（XML 格式），可拿到最近 ~20 次 Commit 的 SHA、标题、时间与作者，不走 `api.github.com`，无 60 req/hr 限制。
3. **备选方案 B：第三方 CDN / 镜像加速（面向第三方用户，稳定直连）**
   - **jsDelivr CDN**：`https://cdn.jsdelivr.net/gh/AgenteamCN/Outputs@{sha}/README.md`（全球 CDN 缓存，国内连通性通常显著优于 `raw.githubusercontent.com`，且支持按 Commit SHA 精确不可变缓存）。
   - **GitHub REST Markdown API**：`POST https://api.github.com/markdown`（`mode: "gfm"`）可获得与 GitHub 官网 100% 一致的 HTML（但会消耗 API 额度，不如本地 GFM 渲染器省额度）。
4. **备选方案 C：公开 Git 镜像 / Gitea / GitLab Mirror**
   - 将 `AgenteamCN/Outputs` 自动 Mirror 到国内可稳定访问的 Git 托管或自建轻量 Gitea 实例，第三方用户即可通过标准 Git/HTTP API 拉取 Commit 与 Markdown，彻底脱离对 GitHub 官方直连的依赖。

---

### 2.4 EMQX CN 流量与每日 20 条 / 20 MB 估算可行性评估

- **单次典型交互体积测算**：
  - WebRTC 建连信令（Offer + Answer + 2~4 个 ICE Candidate）：合计约 **3 ~ 5 KB**。
  - 若 WebRTC P2P 直连成功：后续所有 Commit List 与 README 请求走 DataChannel，**EMQX CN 流量为 0 B**。
  - 若被迫走 EMQX CN E2EE Relay：
    - 1 次 Commit List（20 条）：JSON 约 **5 ~ 8 KB**，AES-GCM + Base64 封装后约 **8 ~ 11 KB**。
    - 1 篇典型报告 `README.md`（约 10~30 KB 文本 / 渲染后 HTML 约 15~45 KB）：E2EE Base64 后约 **20 ~ 60 KB**；即使是一篇包含大量日志的超长报告（200 KB），加密后也仅约 **270 KB**。
- **每日 20 条总量评估**：
  - 正常 20 条预览（含信令 + 列表 + 20 篇 README）：**约 0.6 MB ~ 1.5 MB / 天**。
  - 极端重度场景（每篇均达 500 KB 超大 HTML 且全部走 Relay）：20 × 670 KB ≈ **13.4 MB / 天**。
- **结论**：您预估的 **“每日 20 条 ≤ 20 MB”非常现实且留有充裕余量**。配合“手机端按需 Connect / Disconnect”以及“P2P 优先、EMQX CN 仅作信令与兜底”，实际 EMQX CN 日消耗通常可控制在 **1 MB 以内**。

---

## 三、验证结果

- `pnpm check`（主工程 `tsc --noEmit` + `eslint` + `vitest` 3 files / 19 tests）：**全部通过**
- `pnpm p2p:check`（`p2p-remote-controller` `tsc --noEmit` + `eslint`）：**全部通过（0 errors, 0 warnings）**
- `pnpm --filter p2p-remote-controller test`（`vitest` 11 files / 39 tests，含 30,000 字符无截断测试与 GFM/UUID/TOTP 测试）：**全部通过**
- `pnpm p2p:build` & `pnpm build`：**构建成功**
