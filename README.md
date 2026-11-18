# 问题解答与二次优化报告

## 1. 是否需要单独跑 `pnpm ax-helper:build`？
- **不需要**。`apps/p2p-remote-controller/src-tauri/tauri.conf.json` 中的 `beforeDevCommand` 已经配置为：
  `pnpm --workspace-root ax-helper:build && pnpm dev`
- 因此运行 `pnpm p2p:dev:watch` 时会自动重新编译 `macos-ax-helper` 并启动开发模式；并且 `scripts/devWatch.ts` 也监听了 `packages/macos-ax-helper/`，后续代码有改动时会自动重启并重编。

---

## 2. 为什么之前显示 `Injection Failed / Not Installed`？
- 从你贴出的现象来看：
  1. 控制台已经不再报 `undefined is not an object (evaluating 'res.result.length')`，说明 `macos-ax-helper` **已经成功重编且 JS 执行链路已打通**。
  2. `Poll / Verify Net` 返回 `Recorder not installed in this page (or page navigated)`，说明 `ax_execute_javascript` 成功在 Safari 当前页执行了轮询脚本并拿到了合法 JSON（`installed: false`），只是当前这个 Safari 页面还没有点击 **Record Net** 注入录制脚本（或者注入后发生了页面刷新/跳转）。
- **UI 易混淆点已修复**（Verified Bot Commit `640bffd70a735650a81b58f6b7bed9ebdb5e4268` on `a2`）：
  - 将 `Not Installed`（黄色 `warn` 状态）与真正的 `Injection Failed`（红色 `err` 状态）拆分开，不再误导为注入报错。
  - 在右侧 **Network** 标签页操作栏中，直接在 `Poll / Verify Net` 旁边新增了 **`Inject & Record Net`** / **`Stop Net`** 按钮，不用再回到顶部工具栏找按钮。
  - 若 Safari 页面内执行 JS 抛出异常（`__jsError`），也会直接在 Network 详情中展示具体 JS 错误信息。

---

## 3. 关于 EMQX 历史清空与 Worker 轮询机制的确认
1. **重启 App 后 EMQX 历史清空**：
   - **完全符合预期**。Mac 端每次连接使用随机 `clientId`（`cleanSession = true`），且所有消息均为 `retain = false`，因此重启后不会收到过期历史消息，避免误执行旧指令。
2. **Cloudflare Worker 工作方式**：
   - Worker 内部**没有死循环/定时轮询**。每次需要通信时，由云端 Agent 发起一次 `POST /mqtt/rpc`（带 `waitMs`，例如 3~25 秒），Worker 在等待期间属于 **I/O Wait（不计入 Cloudflare 的 10ms CPU 时间上限）**，拿到 EMQX 下推的排队消息或超时后立即关闭连接并返回。
   - 由于云端侧使用 `cleanSession = false` + 固定 `clientId` + `QoS 1`，在云端两次轮询间隔期间你在 Mac 端发送的消息会由 EMQX Broker 暂存并在下次连上时立刻投递，既不丢消息也不消耗空闲 CPU。
