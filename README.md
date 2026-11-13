# 紧急修复与后续防崩溃工程方案报告

已完成针对 Commit `4b248ddfa72e343a9e0cf0c2a8a274194988ded7` 的紧急修复与 `Agent Name / Agent Model` 快速选择器优化，并已通过 Verified Bot Commit 推送至 `origin/a2`（Commit: `fab1672907c6c004ad3e2952437270561cfbd0aa`）。

---

## 一、崩溃根因分析

Commit `4b248dd` 引入了 `useBackdropClose`（内部调用了 `useRef`），在多个弹窗组件中直接把 `const backdropProps = useBackdropClose(...)` 写在了**组件底部的 `return (` 之前**，而不是**提前返回 `if (!isOpen) return null;` 之前**，直接违反了 React **Rules of Hooks**：

1. `CredentialsSettingsModal.tsx`（**Manage Key**）：`if (!isModalOpen) return null;` 在第 107 行，而 `useBackdropClose` 被放在第 273 行。打开弹窗时 Hook 数量从 24 个突然变为 25 个（多出 `useRef`），触发 `Rendered more hooks than during the previous render` 崩溃。
2. `StatementSnippetModal.tsx`（**Insert Repository / Insert Scan** 共用弹窗）：`if (!isOpen) return null;` 在第 83 行，而 `useBackdropClose` 被放在第 145 行，同样在打开时崩溃。
3. `RequestQuotaModal.tsx`（**Request Quota**）：`if (!isModalOpen) return null;` 在第 81 行，而 `useBackdropClose` 被放在第 129 行，同样存在相同隐患。

> **为什么 AI/开发者容易踩这个坑？**
> 这三个文件都是 **570~1075 行、20~38 KB 的超大文件**，`if (!isOpen) return null;` 在文件上半部分（80~110 行），而 `return (` 在 130~280 行之后，局部修改时极易忽略上方的 early return。

---

## 二、本次紧急修复与功能优化（Commit `fab1672`）

### 1. 修复 Rules of Hooks 崩溃
- `CredentialsSettingsModal.tsx`、`StatementSnippetModal.tsx`、`RequestQuotaModal.tsx`：将 `useBackdropClose(...)` 全部上移至 `if (!isOpen) return null;` 之前。
- 全量扫描 `src/**/*.tsx`，确认已无任何组件在 `return null` 之后调用 React Hook（`Hook order issues found: 0`）。

### 2. Agent Model & Agent Name 快速选择器优化
- **移除 `Custom` 按钮与 `Empty` 选项，保留 `<SquareDashed />` 虚线小框**：
  - 修改 `AgentModelField.tsx`：移除 `Custom` 按钮，仅保留预设标签按钮 + `<SquareDashed />` 虚线小框，节省横向布局空间。
  - 新增 `AgentNameField.tsx`（从 `TaskConfigurationPanel.tsx` 拆出）：同样提供预设标签按钮 + `<SquareDashed />` 虚线小框。
  - **统一语义**：只要当前输入值不匹配任何预设（`!matchesPreset`，包括任意自定义输入或非预设文本），`<SquareDashed />` 虚线小框即自动高亮（`primer-btn-primary`）；点击虚线小框直接聚焦并全选输入框。
- **Agent Name 预设与默认值配置**：
  - 扩展 `presets.ts`、`agentDefaults.ts`、`defaultProfile.json`、`presetStore.ts`、`workspaceDraft.ts`，支持 `agentNamePresets` 持久化、排序与默认 Agent Name 同步。
  - 新增独立设置分区 `AgentNamesSection.tsx`，并在 `AppSettingsModal.tsx` 注册 `Agent Names` 入口。

### 3. 验证结果
- `pnpm build`（`tsc && vite build`）：**通过**（0 TS 错误，Vite 打包成功）。
- 全量 Hook 顺序静态扫描：**0 个问题**。

---

## 三、后续防崩溃与工程约束方案（如何避免再次发生）

不建议为了防这种问题去做全量重写（成本高、风险大），建议用**「硬性静态门禁 + 渐进式拆小文件 + 关键交互烟雾测试」**三层约束，低成本彻底堵住同类问题：

### 1. 立即引入 ESLint + `eslint-plugin-react-hooks`（最高性价比，1 天落地）
- 当前 `package.json` **只有 `tsc && vite build`，没有 `lint` 脚本**。`tsc` 查不出 Hook 顺序错误！
- 引入 `eslint-plugin-react-hooks`（`react-hooks/rules-of-hooks: "error"`）：只要有人在 `if (...) return null;` 之后写任何 `useXxx()`，CI / 本地 `pnpm lint` 会**直接报错拦截**，从根本上消灭此类崩溃。
- 可顺带加一条自定义/ESLint 规则：`max-lines: ["warn", { max: 400, skipBlankLines: true, skipComments: true }]`，限制单文件膨胀。

### 2. 用“弹窗外壳组件”替代手动调用 `useBackdropClose`（架构防呆）
- 与其要求每个 Modal 自己记得在 `return null` 前调用 `useBackdropClose`，不如抽一个统一的 `<PrimerModalShell isOpen={...} onClose={...} maxWidth={...}>`：
  - 外壳组件内部统一处理 `if (!isOpen) return null;`、`useBackdropClose`、`Escape` 键关闭、滚动锁定。
  - 子弹窗只在 `isOpen === true` 时才挂载内部表单状态，天然隔离 Hook 生命周期，不再需要每个 Modal 重复写 backdrop 逻辑。

### 3. 渐进式拆分超大文件（指哪儿打哪儿）
目前 `src/` 中超过 16 KB / 500 行的文件有 30+ 个，建议按**“改到哪、拆到哪”**的原则逐步拆分（优先拆高频修改区）：
- **`CredentialsSettingsModal.tsx`（37.4 KB / 1074 行）**：按凭据域拆为 `GeminiKeySection`、`GithubAppSection`、`EmqxSection`、`BemfaWebhookSection` + 外壳。
- **`TaskOverviewSidebar.tsx`（30.8 KB / 865 行）** & **`TaskConfigurationPanel.tsx`（17.7 KB / 521 行）**：按表单字段/卡片拆成独立小组件（本次已先拆出 `AgentNameField.tsx`）。
- **`taskStore.ts`（51.7 KB / 1625 行）**：按职责拆分为 `taskSlice`、`draftSlice`、`syncSlice`。

### 4. 引入轻量级前端测试流程（Vitest + React Testing Library）
- 不需要上来就做重型 E2E，先补 **Modal 开关烟雾测试**（10~15 个测试用例即可）：
  - 对每个弹窗执行 `render(closed) -> rerender(open) -> rerender(closed)`。
  - 只要存在 Hook 顺序错误或初始化异常，Vitest 会当场报错（`Rendered more hooks`），5 秒内跑完全部弹窗。
- 顺带清理 `src-tauri` 两条 `unused import: redact_sensitive_blocks` warning，保持构建输出干净。
