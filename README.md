### 问题原因

1. **清空默认选项未同步到后端持久化**：在 `src/stores/presetStore.ts` 中，`writeOptionalId`（用于 `agy_default_sys_inst_preset_id_v1` 等默认项）和 `writeStringValue` 在值为空或 `null` 时调用了 `localStorage.removeItem(key)`。而 `src/utils/lanPairingUrl.ts` 中的 `collectLocalPresetsSnapshot` 只收集 `typeof localStorage.getItem(key) === "string"` 的键，导致清空默认 System Instruction 后该键从同步快照中缺席，`src-tauri/src/storage/lan_store.rs` 中的 `save_synced_lan_presets` 仍保留旧值并在下次打开页面或重启应用时恢复旧默认值。
2. **桌面端启动时未优先应用后端同步的持久化配置**：在 `src/stores/lanStore.ts` 的 `initLanSession` 桌面端分支中，启动时调用的是 `hydrateMissingPresetsInLocalStorage(backendPresets)`，只填充 `localStorage` 中缺失或为空的键。当通过终端 `Ctrl+C` 终止或窗口关闭前 WebView 尚未将 `localStorage` 刷盘时，已同步写入 `lan_config.json` 的新默认值会被 `localStorage` 中的旧值忽略，随后又被 `saveSyncedPresetsToBackend(collectLocalPresetsSnapshot())` 用旧值覆盖回后端。

### 修复内容

- **`src/stores/presetStore.ts`**：修改 `writeOptionalId` 与 `writeStringValue`，在清空或置为 `null` 时写入空字符串 `""` 而非 `removeItem`，使 `readOptionalId` 仍返回 `null`，同时确保 `collectLocalPresetsSnapshot` 能将清空状态同步至后端 `lan_config.json` 与 LAN 客户端。
- **`src/stores/lanStore.ts`**：桌面端 `initLanSession` 启动同步改用 `applySyncedPresetsToLocalStorage(backendPresets)`（与 Web 端一致），确保 `lan_config.json` 中同步落盘的最新默认配置覆盖未刷盘或陈旧的 `localStorage`，并移除未使用的 `hydrateMissingPresetsInLocalStorage` 导入。
- **`src/utils/lanPairingUrl.ts`**：将 `agy_agent_name_presets_v1`、`agy_default_agent_name_preset_id_v1`、`agy_custom_default_agent_name_v1` 补全到 `SYNCED_PRESET_STORAGE_KEYS`，保证所有预设默认项同步一致。
- **`src/test/presetPersistence.test.ts`**：新增单元测试，覆盖默认 System Instruction 的设置、清空为 `null` 时快照包含 `""`、以及后端持久化值覆盖陈旧 `localStorage` 的场景。

### 验证与提交

- **工具链**：安装全局 `pnpm`（`12.21.0`）并执行 `pnpm install --frozen-lockfile`。
- **检查与测试**：执行 `pnpm check`（`tsc --noEmit && pnpm lint && pnpm test`）全部通过（0 errors，2 个测试文件共 16 项测试通过），`pnpm build` 构建通过。
- **远程提交**：通过 GitHub GraphQL `createCommitOnBranch` 提交并签名推送至 `a2` 分支（`fb9df3f56a752098de781de6f2f5f2c5ba5050a1`，`verified: true`），本地已 `git fetch && git reset --hard origin/a2`。
