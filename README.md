# P2P Remote Controller & LLM Control 诊断与修复报告

## 1. Live Chat 联合定位结论
通过频道 `ch_ec57ea3a` 实时协同排查，定位并验证了以下现象与根因：
1. **P2P 配置与连通性**：
   - 在 `EMQX: Live` -> `Settings` -> `Cellular P2P Channel` 中配置 Access UUID（不区分大小写）并保持 `Require TOTP` 与 `Passkey (Interface)` 关闭（Passkey 为预留接口，勾选会导致鉴权失败）。
   - 手机端使用与 PC 端一致的 EMQX Broker 参数 + Access UUID 成功建立 **WebRTC P2P Direct** 连接（双端同步 `28.8 KB (10)` 且 EMQX 中继流量不增长）。
2. **`Force URL` / 模型切换完全无反应**：
   - 实测控制台报错：`ax_execute_javascript: The variable jsResult is not defined. (OSStatus -2753)`。
   - 根因：`usePocketDeckBridge.ts` 的 `reload_default` 传入带末尾分号的 `window.location.href = "...";`，而 `SafariController.swift` 的 `executeJavaScript` 将其包裹进 `var __r=(...);` 导致 JS 语法错误，Safari 返回空值进而触发 AppleScript `-2753`。
3. **手机端切标签即断开 P2P & `Direct`/`LLM` 无响应**：
   - 根因：`useCellularP2pClient` 原本挂载在 `PocketDeckOutputsTab` 内部，切换至 `Direct`/`LLM` 标签时组件卸载导致断开并清空统计；同时 `Direct`/`LLM` 仅走局域网 `/api/action`，未接入 Cellular P2P RPC。
4. **Safari 窗口状态误报 `[Uninit]`**：
   - 根因：`applyInteractionDetectedToReadiness` 在检测到交互时将 `ready` 窗口降级为 `uninitialized`，且 `ManagedWindowSidebar` 未对已含 `/prompts/<id>` 的 URL 做兜底纠偏。
5. **Tools 按钮报错与权限警告**：
   - `Tools` 区域折叠或标题变化导致 `AXCheckBox` 正则匹配失败；`default.json` 缺少 `core:window:allow-set-focus` 权限。
6. **UI 布局与日志噪声**：
   - `.emqx-settings-grid` 列宽过窄且缺少 `min-width: 0` 导致输入框重叠；后台轮询 `ax_execute_javascript` 刷屏掩盖关键错误。

---

## 2. 代码修复详情（分支 `a2`，Verified Bot Commit `1815afa`）
- `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift`：支持语句/末尾分号脚本包裹，并在 `do JavaScript` 前初始化 `set jsResult to ""` 彻底消除 `OSStatus -2753`。
- `apps/p2p-remote-controller/src/hooks/usePocketDeckBridge.ts`：移除 `reload_default` 末尾分号并在导航后短暂等待再扫描。
- `apps/p2p-remote-controller/src/types/cellularP2p.ts`、`useCellularP2pHost.ts`、`useCellularP2pClient.ts`、`useEmqxBridge.ts`、`App.tsx`：新增 `pocket_deck_action` 与 `pocket_deck_get_snapshot` RPC 命令并在 Host 端注册处理器。
- `apps/p2p-remote-controller/src/components/PocketDeckView.tsx` & `PocketDeckOutputsTab.tsx`：将 `useCellularP2pClient` 提升至 `PocketDeckView`，实现跨标签保持 P2P 连接并将 `Direct`/`LLM`/`Bridge` 动作与快照同步接入 P2P 通道。
- `apps/p2p-remote-controller/src/services/aiStudioWindowState.ts` & `ManagedWindowSidebar.tsx`：对含 `/prompts/<id>` 的窗口不再降级或显示为 `uninitialized` / `[Uninit]`。
- `apps/p2p-remote-controller/src/components/PocketDeckLlmControlTab.tsx` & `src-tauri/capabilities/default.json`：自动展开折叠的 `Tools` 区域并放宽正则匹配，补充 `core:window:allow-set-focus` 权限。
- `apps/p2p-remote-controller/src/styles/app.css`、`CellularP2pSettingsCard.tsx`、`DebugConsoleTabs.tsx`：修复设置网格溢出重叠，并在 Console 增加 `Errors/Warnings only`、`Hide ax_execute_javascript info`、`Hide ax_scan_window info` 过滤开关。
- `apps/p2p-remote-controller/src/services/aiStudioWindowState.test.ts`：新增针对 `/prompts/<id>` 窗口状态转换的单元测试。

---

## 3. 验证结果
- `pnpm p2p:check` (`tsc --noEmit && eslint src`)：通过
- `pnpm --filter p2p-remote-controller test --run`：11 个测试文件、42 项测试全部通过
- `pnpm p2p:build`：生产构建通过
- GitHub Verified Bot Commit：`1815afa775be520c5ab726d9a9e1a55a25bfa461` (`signature.isValid: true`, `state: "VALID"`)
