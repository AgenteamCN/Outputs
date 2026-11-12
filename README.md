# 修复报告：统一红色复制按钮为 "Copy Raw" 并保障原始敏感信息原样复制

## 1. 问题根因分析

在最近的提交中，针对任务输入复制功能存在以下几个核心问题：

1. **按钮文案不统一**：
   - Composer 界面红色按钮为 `"Copy Input to Clipboard"`。
   - Inspector 界面红色按钮为 `"Copy Real Input"`。
   - 用户期望将带有危险警示色（红底 `primer-btn-danger`）的原生复制按钮统一命名为 **"Copy Raw"**，明确传达“此按钮代表用户完全知情、将不经脱敏直接复制原始完整内容”的语义。

2. **用户 Prompt 正文中形似 Key/Token 的内容被误隐藏**：
   - 之前在 `parsePromptWithGithubAuth` 中对截取出的 `remainder`（即用户输入正文）调用了 `redactInlinePrivateKeys`。
   - `redactInlinePrivateKeys` 随后被挂接到了 `redactSensitiveText`，导致用户输入正文中的 GitHub Token (`ghp_...`)、Worker Token (`bk_...`)、JWT (`eyJ...`)、URL 参数密钥 (`?key=...`) 等全部被强制替换成了 `[REDACTED_...]`。
   - 当点击红色复制按钮时，虽然前面的 `# 1. Auth Configuration` 恢复了真实的认证凭据，但正文中的密钥却依然被脱敏覆盖，导致实际外发的内容被破坏。

3. **无 Auth 场景下逻辑拧巴**：
   - 原 Inspector 红色复制按钮判断 `if (!parsedInput.hasGithubAuth)` 时直接返回 `baseText`。
   - 但因为此前入库 `TaskRecord.fullPrompt` 时也经过了 `formattedDisplayPrompt` 脱敏处理，导致无 GitHub Auth 的任务在普通复制和红色复制时均被误脱敏。
   - 实际上红色按钮的职责非常明确纯粹：**无论任务是否包含 GitHub Auth，用户发出去什么内容，红色按钮就原封不动地复制什么内容**（包含 GitHub Auth 时完整拼接未经脱敏的用户正文与真实凭据；不包含时直接输出用户原始正文，零脱敏）。

4. **超大文件拆分与解耦**：
   - `TaskComposerView.tsx` 原为 938 行（34KB+），`TaskInspectorView.tsx` 原为 665 行（22KB+），超出了单一职责及避免超大文件（600~800 行、12~16KB+）的工程要求。

---

## 2. 核心修改与架构解耦

### 2.1 修复 `githubAuthBlock.ts` 避免正文误脱敏
- 修改 `parsePromptWithGithubAuth`：不再对 `remainder` 或 `safeRaw` 调用 `redactInlinePrivateKeys`。返回的 `userBody` 保持 100% 原始正文内容。
- 新增并导出 `buildRawPromptWithAuth(prompt, includeGithubAuth, authBlock)` 工具函数：
  - 若任务包含 GitHub Auth 或勾选了插入凭据，按结构提取未经任何脱敏破坏的 `userBody`，与真实的 `authBlock` 完整拼接后输出；
  - 若任务不包含 GitHub Auth，直接返回纯原始正文，不做任何脱敏替换。

### 2.2 统一红色复制按钮为 "Copy Raw"
- **TaskComposerView**：
  - 红色按钮统一改名为 `"Copy Raw"`（点击后显示 `"Copied"` 1.5s）。
  - 处理函数更新为 `handleCopyRawToClipboard`，使用 `buildRawPromptWithAuth` 产出完整 Raw Prompt。
- **TaskInspectorView**：
  - 红色按钮统一改名为 `"Copy Raw"`（点击后显示 `"Copied"` 1.5s）。
  - 保留普通白色按钮 `"Copy Input"` 负责复制截屏视图 (`formattedDisplayPrompt`)。

### 2.3 模块化解耦超大组件
1. **抽离 `TaskInputSnapshotPanel.tsx`**（位于 `src/components/inspector/`）：
   - 将 Inspector 的 Raw Snapshot Box（包含状态 Badge、普通 `Copy Input`、红色 `Copy Raw`、以及展示 pre 文本）解耦至独立组件中。
   - `TaskInspectorView.tsx` 行数从 665 行精简至 **529 行**，远低于 600 行警戒线。
2. **抽离 `TaskComposerCollapsedSettings.tsx`**（位于 `src/components/composer/`）：
   - 将 Composer 在右侧栏收起状态下的紧凑标题与 Tag Presets 区域抽离为独立组件。
   - `TaskComposerView.tsx` 行数从 938 行精简至 **799 行**。

---

## 3. 验证与测试

- **单元逻辑验证**：
  - 测试正文包含 `ghp_...`、`bk_...`、`?key=...` 且无 Auth 场景：`parsePromptWithGithubAuth` 严格保留原始未脱敏正文；`buildRawPromptWithAuth` 返回完整无脱敏字符串。
  - 测试正文包含 `[REDACTED_GITHUB_AUTH_AND_COMMIT_PREFERENCE]` 场景：`buildRawPromptWithAuth` 正确将真实认证块与完整未脱敏正文拼接。
  - 测试正文包含完整 `# 1. Auth Configuration` 块场景：`buildRawPromptWithAuth` 保持完整原样。
- **生产构建验证**：
  - 执行 `pnpm build`：TypeScript 严格类型检查与 Vite 生产打包 100% 通过（2231 modules transformed，0 errors）。

---

## 4. GitHub Verified Bot Commit 信息

- **Commit SHA**: `1ad46ab21cbd6d3f56e8bd1977bda6893755f8d1`
- **Branch**: `a2`
- **Verified Signature**: `true` (`antigravity-fine-grained-token[bot]`)
- **Co-authored-by**: `Mark <29165094+MaBoCoMark@users.noreply.github.com>`
