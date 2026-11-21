# Live Chat 协作与 P2P Remote Controller / Cloudflare Worker 改造总结报告

**仓库**：`Antigravi-ty/agyPreviewInstant`（分支 `a2`）
**Verified Bot Commits**：
1. `5c2366c63ccb29a19c6acf4e5e0d89521015e5a8` — AI Studio 初始化与窗口隔离、PocketDeck UI 持久化、Bridge 默认全量与滚动、Debug 控制台 per-command `console.info` 过滤
2. `1f9cfe3e7187a30636edc08d0f4178676dbd3ee3` — No-Trust Worker E2EE（`off` / `dual` / `strict`）+ WebAuthn Passkey 10 分钟滑动信任窗口 + 手机端隔离 Admin EMQX 凭据
3. `71a9f9053334e4169efa4622a12f877822fbe429` — 修复 PocketDeck 手机端 LLM 控制（`Unlink API Key`、`Thinking Level`、`Run` settle 延迟与 `Toggle run settings panel` 安全检查）

**Cloudflare Worker**：`agentlivechatchannel`（`live-chat-v2.1`，已部署生效）

---

## 一、已完成的核心需求与落地细节

### 1. GitHub Authentication Token（移动端扫描限流）
- 已指导签发只读 Fine-grained PAT（`Outputs Reader P2P`，仅授权 `AgenteamCN/Outputs` 的 `Contents: Read-only` 与 `Metadata: Read-only`），将 API 限额从 60 req/hr 提升至 5,000 req/hr。用户已在手机端验证可正常拉取 commits。

### 2. Cloudflare Worker `stream` 与 `heartbeat` 优化
- 实测确认 `execute_bash` + `curl` 会缓冲 stdout，10 秒周期性 `heartbeat` 仅增加 Agent 上下文噪音。
- 已更新 `agentlivechatchannel` Worker（`/tmp/worker.mjs` 已部署）：`heartbeat` 默认改为 `false`（仅显式传 `heartbeat: true` 时输出诊断心跳），`stream: true` 仅输出 `connected` 与 `result`，并同步更新 `/help` 文档。

### 3. EMQX 分离权限与手机端凭据隔离
- 复核并确认 EMQX CN ACL 配置：
  - `CNremoteController`（Admin/Host/Worker）：Allow Pub/Sub `agy_p2p_v1/#`，Deny `#`。
  - `CNremoteP2PCellularOnly`（Mobile Cellular）：仅 Allow Pub `agy_p2p_v1/cellular/req/#`、`agy_p2p_v1/cellular/req` 与 Allow Sub `agy_p2p_v1/cellular/res/#`，Deny `#`。
- 修复 `apps/p2p-remote-controller/src/hooks/useCellularP2pClient.ts`：不再从 `loadEmqxBridgeConfig()` 继承 `username` 与 `password`，确保手机端绝不接触 Admin 凭据。用户已在手机端配置受限账户并验证 P2P 正常连接。

### 4. No-Trust Worker E2EE（EMQX 全程密文）
- **加密方案**：256-bit `E2EE_PSK` + 每条消息随机 16 字节 `salt` + `HKDF-SHA256`（`info = "agy-worker-e2ee-v1"`）派生一次性 `AES-256-GCM` 密钥 + 随机 12 字节 `iv`，线上格式 `{"e2ee":"v1","s":...,"iv":...,"ct":...}`。
- **双模平滑过渡与严格模式**：支持 `off` / `dual` / `strict` 三种模式。已在 Mac 端（`EmqxBridgeSettingsCard.tsx`、`useEmqxConnection.ts`、`useEmqxApprovalsAndChat.ts`、`emqxCommandRouter.ts`、`p2pSecurity.ts`）与 Cloudflare Worker（`E2EE_PSK`、`E2EE_MODE`）同步落地，并经 Live Chat 双向验证 `dual` 与 `strict` 模式均可在 ~230ms 内正常加解密。
- **安全边界说明**：当前方案防的是 **EMQX Broker 不可信**（EMQX 仅见随机密文），不具备跨端前向保密（若 `E2EE_PSK` 本身泄露，历史抓包可被解密）；如需单端密钥隔离，后续可升级为双端 ECIES P-256。

### 5. P2P Passkey（读免密 + 控制 10 分钟滑动窗口）
- 新增 `apps/p2p-remote-controller/src/utils/webauthnPasskey.ts`，并在 `PocketDeckHeader.tsx` 与 `PocketDeckView.tsx` 接入：
  - **读操作免密**：`select_window`、`window_command: "scan"`、查看 Bridge / Outputs / Snapshot 均无需生物识别。
  - **控制操作门禁**：其余控制类动作在已注册 Passkey 时触发 `ensurePasskeyAuthorizedForControl()`，通过 `window.location.hostname` 绑定平台级 WebAuthn（FaceID / TouchID），并维护 10 分钟滑动信任窗口（支持顶部倒计时徽章与手动立即锁定）。
- **边界说明**：当前 Passkey 为手机端 PWA 本地生物识别门禁，Mac Host 端尚未做 WebAuthn 公钥签名验签；用户已实测 10 分钟滑动窗口体验良好。

### 6. AI Studio 初始化、窗口隔离、UI 与手机端 LLM 控制修复
- **窗口隔离与批量保护（`5c2366c`）**：非 AI Studio URL 预过滤、操作后立即还原焦点、`ensureRunSettingsPanelOpen`、手机端 `open_new_window` 标记 `source: "mobile"`（`[Mobile]` 标签）且不进批量 `All`、`emqxCommandRouter.ts` 与 `usePocketDeckBridge.ts` 严格过滤仅暴露托管窗口。
- **UI 与日志（`5c2366c`）**：PocketDeck Tab 状态持久化、Bridge 默认显示全部并自动滚底、`axService.ts` + `DebugConsoleTabs.tsx` 增加按命令持久化的 `console.info` 过滤。
- **手机端 LLM 控制修复（`71a9f90`）**：修复 `PocketDeckLlmControlTab.tsx` 中 `ensureRunSettingsOpen`（仅在按钮存在时点击）、`handleUnlinkApiKey`（先展开 Run Settings 且放宽 `API key selected` 角色匹配）、新增 `handleSelectThinkingLevel`（自动先展开 `Thinking Level` 下拉框再选 `Minimal`/`Medium`/`High`），并在 `handleSendPrompt` 注入文本后增加 180ms 稳定等待。

---

## 二、验证结果
- `pnpm exec tsc --noEmit`：通过（0 errors）。
- `pnpm test -- --run`：13 个测试文件、49 项单元测试全部通过（含 `p2pSecurity.test.ts` 的 Worker E2EE HKDF+AES-GCM 测试与 `webauthnPasskey.test.ts` 的 10 分钟滑动窗口测试）。
- Cloudflare Worker `agentlivechatchannel`：`node --check /tmp/worker.mjs` 通过，已部署并通过 `/version` 与 `strict` 模式 Live Chat 实测验证。
