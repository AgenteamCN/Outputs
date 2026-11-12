# Fix Summary: Unify "Copy Raw", Preserve Unredacted Input Keys, and View Decoupling

## 1. 问题分析与修复背景

### 1.1 问题根本原因 (Root Cause)
在原先的实现中，用户点击红色复制按钮（用于导出带鉴权凭据的原始 Input）时出现了敏感信息被意外遮蔽或截断的问题：
1. **前端 Prompt 解析脱敏过度**：`src/utils/githubAuthBlock.ts` 中解析 Prompt 时，对用户自定义输入的 body 内容（`remainder`）调用了 `redactInlinePrivateKeys` 等脱敏过滤，导致用户自行输入的密钥、API Key、Worker Token 被替换为 `[REDACTED_SECRET]`。
2. **Rust 后端存储脱敏破坏 Input 原文**：在 `src-tauri/src/google_api/client.rs` 与 `output_recovery.rs` 中，后端在收到 `req.prompt` 构造任务记录以及拉取恢复任务交互时，错误地对 `req.prompt` 与 `existing.full_prompt` 调用了 `redact_sensitive_blocks`，导致原本用户发出的输入内容在后端被二次遮蔽。
3. **按钮命名与行为不一致**：部分视图命名为 "Copy Input to Clipboard"，部分为 "Copy Real Input"，未能统一明确为 **"Copy Raw"**。红色警示样式表示该操作会原样导出完整凭据与敏感字段。
4. **组件单文件膨胀与耦合严重**：`TaskComposerView.tsx`（>800 lines, 29KB）与 `TaskInspectorView.tsx`（>660 lines, 23KB）超过了模块化标准（<600 lines, <16KB 规范），业务横幅、动作栏、标签栏混杂在一起，后续改动易相互干扰。

---

## 2. 解决方案与修改内容

### 2.1 彻底保留原始输入凭据与密钥
- **Rust 后端 (`src-tauri/src/google_api`)**:
  - 在 `response_parser.rs` 中导出 `pub fn collapse_github_auth_header_block`。
  - 在 `client.rs`（创建任务与合并交互）与 `output_recovery.rs`（恢复输出交互）中，将 Prompt 的脱敏由全局敏感字段替换 `redact_sensitive_blocks` 调整为仅折叠 GitHub Auth 头部块 `collapse_github_auth_header_block`。
  - 用户正文中的 API 密钥、私钥、Token 原封不动保存在 `full_prompt`，真正做到“发出去是什么东西，复制出来就是什么东西”。
- **前端工具库 (`src/utils`)**:
  - `src/utils/githubAuthBlock.ts`: 移除对 `remainder` 正文的 `redactInlinePrivateKeys` 篡改，实现 `buildRawPromptWithAuth` 恢复真实完整的 GitHub Auth 头。
  - `src/utils/rawPromptCopy.ts`: 新建统一的异步 Raw 复制工具函数 `copyRawTaskPrompt(prompt, includeGithubAuth)`，自动处理缓存与 Safari/WebKit 瞬态激活兼容。

### 2.2 统一红色复制按钮为 "Copy Raw"
- 统一在 `TaskComposerView`（底部动作栏）与 `TaskInspectorView`（输入快照面板）中将红色按钮文案与 `aria-label` / `title` 命名为 **"Copy Raw"**。
- 灰色按钮保持为安全或折叠后的普通复制（"Copy Input"）。

### 2.3 拆分解耦超大组件文件 (低耦合、按功能区域拆分)
严格限制所有文件在 600 行以内、16KB 以内：
1. **`src/components/composer/ComposerBanners.tsx`** (350 lines, 12KB):
   - 提取草稿编辑横幅 (`editingSavedDraft`)、Hook 状态警示横幅 (`!enableToolHooks`)、GitHub Auth 状态横幅、凭据缺失警示横幅以及多轮 Continuation 冲突提示。
2. **`src/components/composer/ComposerActionBar.tsx`** (128 lines, 3.9KB):
   - 提取底部字数统计、红色 **"Copy Raw"** 按钮、"Save as Draft" 与 "Dispatch Task" 调度按钮。
3. **`src/components/composer/TaskComposerCollapsedSettings.tsx`** (154 lines, 4.8KB):
   - 提取折叠侧边栏状态下的标题与标签输入卡片。
4. **`src/components/composer/TaskComposerView.tsx`** (精简至 370 lines, 13KB):
   - 聚焦于主编辑器输入框、快捷代码片段插入以及状态调度。
5. **`src/components/inspector/TaskInspectorTabBar.tsx`** (223 lines, 6.5KB):
   - 提取检查器顶栏 Tab 切换、轮次导航 (Round Prev/Next)、刷新与重试操作。
6. **`src/components/inspector/TaskInputSnapshotPanel.tsx`** (148 lines, 4.4KB):
   - 提取输入快照查看器，集成灰色 "Copy Input" 与红色 **"Copy Raw"**。
7. **`src/components/inspector/TaskInspectorView.tsx`** (精简至 358 lines, 12KB):
   - 专注文档流编排与各子面板调度。

---

## 3. 文件尺寸合规核验

| 文件路径 | 行数 | 文件大小 | 状态 |
| :--- | :--- | :--- | :--- |
| `src/components/composer/TaskComposerView.tsx` | 370 行 | 13 KB | ✅ 严格符合要求 (<600 lines, <16KB) |
| `src/components/composer/ComposerBanners.tsx` | 350 行 | 12 KB | ✅ 符合要求 |
| `src/components/composer/ComposerActionBar.tsx` | 128 行 | 3.9 KB | ✅ 符合要求 |
| `src/components/composer/TaskComposerCollapsedSettings.tsx` | 154 行 | 4.8 KB | ✅ 符合要求 |
| `src/components/inspector/TaskInspectorView.tsx` | 358 行 | 12 KB | ✅ 符合要求 |
| `src/components/inspector/TaskInspectorTabBar.tsx` | 223 行 | 6.5 KB | ✅ 符合要求 |
| `src/components/inspector/TaskInputSnapshotPanel.tsx` | 148 行 | 4.4 KB | ✅ 符合要求 |
| `src/utils/rawPromptCopy.ts` | 50 行 | 1.5 KB | ✅ 符合要求 |
| `src/utils/githubAuthBlock.ts` | 251 行 | 7.7 KB | ✅ 符合要求 |

---

## 4. 构建与验证

- `pnpm build`: 执行 `tsc && vite build`，2235 模块完整编译通过，零类型错误，产物正常输出。
- GitHub GraphQL Verified Commit 已创建并同步至远端分支 `a2`：
  - **Commit OID**: `6753e645249468424b28818b774d8a4d3a0a06f7`
  - **Commit URL**: https://github.com/Antigravi-ty/agyPreviewInstant/commit/6753e645249468424b28818b774d8a4d3a0a06f7
