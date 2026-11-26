# 一键启动与真机验收指南（agyDevKit `a2` 最终汇总）

> 本文档已严格脱敏，不含任何 Token、UUID、真实域名或个人隐私信息。

## 一、本轮已合入 `Antigravi-ty/agyDevKit`（`a2` 分支）核心成果
最新 `origin/a2` 提交：`5d809186`（所有合入均走 Verified Bot Commit / Merge）。

### 1. `singbox-gui` 增强模式、MITM 与性能
- **#160d / #160f**：MITM 改写引擎 + **Strict Option A 权限模型**（`sing-box` 永不以 `root` 运行；仅 `agy-tun-helper` 在冷启动和手动 `off->on` 时通过 `osascript` Touch ID 弹窗提权并通过 `SCM_RIGHTS` 传递 `utun` fd，取消弹窗自动优雅退回普通模式）。
- **#160e**：JS Script-Rewrite 骨架（带 64 KiB 与超时边界）、Soft MITM（仅对新连接生效 + UI `Close Active (N)` 手动断连按钮）、Smart Domain Prefer（`OPTIONS`/`HEAD` 探测 + 默认关闭的可选双发 `GET`）、局域网 Wi-Fi IP 与 `198.18.0.2` DNS 展示、流式 SSE/Chunked Tee（8 KiB 切片防 OOM）。
- **#187**：有界 `127.0.0.1` 回环压测（并发<=4、总包<=4 MiB、总耗时 0.06s < 15s；64 KiB chunked/SSE 达 33.46 MiB/s）。
- **#168 / #169**：Windows CA 信任引导 + EMQX Keepalive 与代理关闭修复。

### 2. 移动端持久化、多端并集同步与托盘收口
- **#188**：修复纯移动端直连 EMQX 时 `localStorage` 空数组 `"[]"` 阻塞 IndexedDB 水合的问题，持久化上限提升至 500 条，持久化 `channelStatuses`（发现的频道列表），并在重连时通过 `msgId` 实现 Mac <-> Mobile 双向差量并集同步。
- **#173 / #184 / #185-audit & #185-fix**：`unified-tray` 多行命令输入、新增 `EMQX CN (Private)` / `EMQX Global (Private)` 无痕快捷入口、清理 `p2p-remote-controller` 废弃 `build_tray()` 死代码与旧图标，并将 `keep_awake` 抽离至 `agy_runner_core` 且为 `unified-tray` 补齐 `keep_awake` 与 `open_devtools`。

### 3. Task Manager、Browser 脱敏、AX 自动化与 Windows 拆分
- **#183**：`agy-task-manager` 适配 `error`/`deadline_exceeded` 终态（不再卡在 `in_progress`）、`step.stop`、`usage` 解析与缺失时间戳显式展示 `未提供`。
- **#178**：`p2p-browser` Rust/TS 抓包与日志全量脱敏（`Authorization`/`Cookie`/`Bearer`/`ghs_` 等）+ `127.0.0.1`/`localhost` CDP 安全绑定。
- **#186**：`agy-ax-terminal` macOS `Info.plist` 补齐 `NSAppleEventsUsageDescription` 与 `--automation-preflight`（`System Events` / `Safari` 权限预检）。
- **#174 / #179 / #175 / #171**：Windows 独立 `apps/win-admin-controller`（含 Scope 绑定与 DPAPI `CredentialVault`）、Windows UIA/CDP 终端、Windows 一键引导与自升级脚本。
- **#181 / #170 / #176 / Audit Pass B**：全局 `config/runtime-limits.json`（`300s - 30s = 270s`）与 `watchdog.mjs`、`rustls`/`minimatch` 依赖安全更新、根 `README.md` 平台矩阵，以及 `p2p-remote-controller` `logExport.ts` 脱敏与 `types/lanSharing.ts` 循环依赖消除。

### 4. 暂留特性分支（未合入 `main`）与只读调研
- `Antigravi-ty/agyWorker` 特性分支：`feat/causal-tag-and-channel-hint`（#163）、`feat/multi-human-routing`（#177）。
- 只读调研：#180（Free Org 规则集 403 -> 建议走 Worker `github-commit` 代理）、#182（HiveMQ Serverless 2026-12-31 EOL -> 建议保留 EMQX）。

---

## 二、一键启动与拉取更新

### macOS 端
```bash
git checkout a2 && git pull --ff-only origin a2
node scripts/env/doctor.mjs --auto-restore
pnpm run env:check
pnpm run check:gates
# 分别启动所需应用（例如 unified-tray / p2p-remote-controller / singbox-gui / agy-task-manager）
```
- 仅 `agy-terminal` / `agy-ax-terminal` 需重新打包 `.app`（含本次新增的 `NSAppleEventsUsageDescription`），其余前端/Tauri 开发态可直接源码运行或通过 P2P OTA 更新手机端。

### Windows 端
```powershell
pwsh -File scripts/windows/bootstrap.ps1
node scripts/windows/preflight.mjs
```

---

## 三、真机验收清单
1. **iOS / HMOS 手机端（直连 EMQX + 断线重连）**：
   - 在无 Mac 局域网主机情况下，用手机端收发数条消息后杀掉 App/切后台重开，确认历史消息与已发现频道完整保留；
   - 再连接 Mac 端，确认双端缺失消息按 `msgId` 自动取并集补齐。
2. **macOS `unified-tray`**：
   - 检查托盘菜单中 `Shortcuts -> EMQX CN (Private) / EMQX Global (Private)` 可正常打开无痕窗口；
   - 确认 `Keep-Awake` 与 `Open DevTools Console` 可正常切换/打开。
3. **macOS `singbox-gui` 增强模式与 Soft MITM**：
   - 切换 Enhanced Mode 触发 `agy-tun-helper` Touch ID 弹窗：点“允许”正常建立 `utun`，点“取消”优雅退回普通模式不崩溃；
   - 控制台确认显示局域网 Wi-Fi IP 与 `198.18.0.2` DNS；修改 MITM 规则确认不断开存量连接，点击 `Close Active (N)` 可手动清理。
4. **`agy-task-manager`**：
   - 导入含 `error` (`deadline_exceeded`) 与 `step.stop` 的 SSE 样本，确认任务正确落入 `failed` 终态且缺失时间戳显示 `未提供`。

---

## 四、后续遗留跟进项（7 项 Rust P2 审计小修）
以下 7 处已在 Audit Pass A2 定位，因部分 Rust 沙盒容器静默失联且 Chief 容器未装 Rust 工具链，留待下一轮有 Rust 工具链的 Specialist 一并收口：
1. `apps/p2p-remote-controller/src-tauri/src/lan_server.rs:1263-1266`（`let _ = serve_connection` 改记错误日志）
2. `apps/unified-tray/src-tauri/src/main.rs:2076-2079`（`let _ = fs::write` 改记错误日志）
3. `apps/agy-task-manager/src-tauri/src/commands/lan_commands.rs:690, 739`（`let _ =` 改记错误日志）
4. `apps/p2p-browser/src-tauri/src/{host_relay.rs, download.rs, webview/mod.rs}`（`emit_to` / `eval` `let _ =` 改记错误日志）
5. `apps/singbox-gui/src-tauri/src/commands/mitm_rewrite.rs:221-227`（`GzDecoder` 加 `.take(MAX_DECOMPRESSED_BYTES + 1)` 防 gzip-bomb OOM）
6. `apps/agy-ax-terminal/src-tauri/src/cdp_client.rs:90-94`（`stream.take(4 MiB)` 防无界读取）
7. `apps/singbox-gui/src-tauri/src/commands/singbox.rs:194-195`（清理 `ManagedSingboxProcess` `dead_code` 告警）
