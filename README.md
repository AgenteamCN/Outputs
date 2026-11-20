# 调查结论与重构报告：`Send to AGY` 机制、明文泄露风险与本地内部通道改造

## 1. 你的猜测完全正确（原机制存在严重问题）

在对 `Antigravi-ty/agyPreviewInstant`（分支 `a2`）进行完整代码审计后，确认**改造前**的 `Send to AGY` 机制确实存在你担心的两个问题：

1. **确实走了 EMQX**：
   - `apps/p2p-remote-controller/src/hooks/useEmqxConnection.ts` 中的 `publishInteractionToAgy` 将任务通过 EMQX WebSocket (`wss://...`) 发布到 `${prefix}/agy/import_task`（默认 `agy_p2p_v1/agy/import_task`）。
   - `apps/p2p-remote-controller/src/hooks/useNetworkRecorderPolling.ts` 中的 `autoSyncToAgy` 甚至要求 `isEmqxConnected()` 为真才自动同步。
   - 主端 `src/stores/telemetryBrokerCoordinator.ts` 订阅了 `+/agy/import_task`，收到后通过 `src/stores/telemetryStore.ts` 触发 `importTaskById`。
2. **确实将 `input` 以明文 JSON 发给了 EMQX Broker**：
   - 原 `publishInteractionToAgy` 直接构造明文 JSON：
     ```json
     {
       "type": "remote_interaction_detected",
       "interactionId": "...",
       "input": "<你的完整明文输入>",
       "sourceUrl": "...",
       "foundAt": "...",
       "windowId": ...,
       "sentAt": "..."
     }
     ```
     并通过 `client.publish(topic, payload, 1, false)` 明文发送至 EMQX。
   - **安全建议**：如果你此前开启过 `Send to AGY` 或 `autoSyncToAgy` 且连接过 EMQX，**所有曾经通过该通道发送过的 `input` 中包含的敏感凭据（Token/Key/Password 等）均已在 EMQX 通道中明文传输过，建议按你的判断及时吊销/轮换。**

---

## 2. 已完成的架构改造（Commit `3a56ff9`，Verified Bot Signature）

分支：`a2`
提交：[`3a56ff9337e1c7858ace946510fd59d27f98c97e`](https://github.com/Antigravi-ty/agyPreviewInstant/commit/3a56ff9337e1c7858ace946510fd59d27f98c97e) (`signature: VALID`)

### 2.1 本地 Internal Channel（零 EMQX、零明文外发、零轮询资源占用）
- **通道路径**：复用了两个 Tauri 应用在同一台设备上共享的配置目录 `<config_dir>/com.antigravity.previewinstant/`。
- **实时通道（Unix Domain Socket）**：
  - 主端 `src-tauri/src/commands/internal_channel.rs` 在启动时绑定 `internal_channel.sock`（权限 `0600`），使用 `tokio::net::UnixListener::accept().await` 异步阻塞等待连接，空闲时零 CPU / 零轮询。
  - `p2p-remote-controller` 新增 Tauri 命令 `send_interaction_to_agy`（`apps/p2p-remote-controller/src-tauri/src/commands.rs`），优先直连 `internal_channel.sock` 写入单行 JSON 并等待 `{"ok":true}` 确认。
- **离线队列兜底（`internal_inbox/`）**：
  - 当 AGY 主端未启动（socket 不存在或连接失败）时，`send_interaction_to_agy` 自动以 `0600` 权限原子写入 `internal_inbox/<interactionId>.json`。
  - AGY 主端在启动及前端挂载调用 `drain_internal_inbox` 时自动消费并删除队列文件。
- **彻底移除 EMQX 导入通道**：
  - 移除了 `p2p-remote-controller` 中 `useEmqxConnection.ts` 的 `publishInteractionToAgy` 与 `useNetworkRecorderPolling.ts` 对 `isEmqxConnected()` 的依赖（`Send to AGY` 与自动同步不再需要连接 EMQX）。
  - 移除了 AGY 主端 `telemetryBrokerCoordinator.ts` 的 `+/agy/import_task` 订阅及 `telemetryStore.ts` 的远端导入处理。

### 2.2 直接植入 Task History（不立即发起网络请求）
- AGY 主端 `ingest_internal_import` 收到本地通道传来的 `{ interactionId, input, sentAt }` 后：
  - **直接本地构造/更新 `TaskRecord`**（写入 `interactionId`、`fullPrompt = input`、`promptPreview`、提取首行作为 `title`、标记 `importedFromRemote: true`），持久化到 `agy_tasks_index.json` 和 `tasks/<id>.json`。
  - 通过 Tauri 事件 `internal:import-task` 推送给前端 `ingestInternalImportedTask`，直接插入 Task History 列表顶部。
  - **不会在导入时自动请求 Google API**，用户可在需要时手动点击刷新。

### 2.3 Task History 卡片右上角显示 Imported Icon
- 在 Rust 模型（`TaskRecord`、`TaskIndexEntry`、`task_registry.rs`）和前端类型（`TaskRecord`、`taskPersistence.ts`）中新增并持久化 `importedFromRemote` 标志，跨刷新保留。
- `src/components/layout/TaskGroupCard.tsx`：当 `task.importedFromRemote` 为 `true` 时，卡片右上角状态徽章不再显示步骤数 `effectiveStepCount`，而是显示导入图标（`<Download size={11} />`）。

### 2.4 刷新失败感叹号与成功获取后自动消除
- 排查发现此前 `task_registry.rs`（`merge_task_records`）和 `src/utils/taskPersistence.ts`（`protectTaskRecordLocalFields`）在后续刷新成功、返回 `errors: []` 时会无条件保留旧的 `existing.errors`，导致感叹号一旦出现就无法消去。
- 现已修复：
  - Rust 新增 `upsert_task_record_from_fetch`，并在 `import_task`、`refresh_task`、`refresh_task_output_only` 成功返回且状态非 `failed`、无接口错误时，自动清理历史上的瞬态 `"refresh"` / `"refresh_output_only"` / `"import"` 错误记录。
  - 前端 `protectTaskRecordLocalFields` 同步在收到健康的拉取结果（`status !== "failed"` 且 `hasDetailLoaded || hasIoLoaded`）时过滤掉上述瞬态刷新错误，感叹号随之自动消失；若拉取失败，仍正常保留感叹号提示。

---

## 3. 工具链安装与验证结果

已在沙箱中补齐 `pnpm`（`12.21.0`）、`rustup`/`cargo`/`rustc`（`1.100.0`，持久化在 `/workspace/.rust`）及 `gcc`/`pkg-config`/`libgtk-3-dev`/`libwebkit2gtk-4.1-dev`。

所有检查均已通过：
- `pnpm check`（`tsc --noEmit && eslint src && vitest run`）：**19/19 测试通过**（含新增 `src/test/internalImportAndErrorClear.test.ts` 的 3 个单测），0 errors。
- `pnpm p2p:check`（`tsc --noEmit && eslint src`）：0 errors。
- `cargo check --manifest-path src-tauri/Cargo.toml`：**通过**。
- `TAURI_CONFIG='{"bundle":{"externalBin":[]}}' cargo check --manifest-path apps/p2p-remote-controller/src-tauri/Cargo.toml`：**通过**。
