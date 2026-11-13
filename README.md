已完成 `Antigravi-ty/agyPreviewInstant`（分支 `a2`，Verified Bot Commit `4b248dd`）的改动与构建验证（`pnpm build` 通过）：

---

### 一、代码改动总结

1. **右侧 Configuration — Agent Model 预设与自定义传参机制**
   - **设置面板 (`AgentModelsSection.tsx`)**：在 Settings 新增 **Agent Models** 分类，支持添加/编辑/排序/删除模型预设，每条预设包含 **Short Tag**（按钮短标签）与 **Model Name**（实际传给 API 的模型名）；顶部支持配置新任务的默认模型（`Profile Default`、任意预设、或 `Custom` 自定义文本）。
   - **配置面板 (`AgentModelField.tsx` / `TaskConfigurationPanel.tsx`)**：在右侧 Configuration 的 `Agent Model` 区域，提供与 `Task Title` 一致的快捷标签按钮栏 + `Custom` 自定义文本传参输入选项 + 一键恢复默认（`Restore`）。
   - **状态与草稿同步 (`presetStore.ts` / `workspaceDraft.ts`)**：持久化模型预设及默认选择，并在新建草稿及设置变更时自动同步默认模型。

2. **Environment Manager Header 存储上限解锁调节**
   - **默认锁定态 (`EnvironmentManagerModal.tsx`)**：在 Header 新增锁头按钮（默认锁定）。锁定状态下不显示输入框，下方的 Usage Banner 正常显示当前用量与上限（默认 `2048 MiB`）。
   - **解锁编辑态 (`environmentStore.ts` / `environmentSorting.ts` / `AppTopHeader.tsx`)**：点击锁头解锁后，Header 内直接显示 `MiB` 数值输入框（及重置为 `2048 MiB` 的按钮），修改后持久化至本地并实时联动更新 Usage 进度条与顶部 `AppTopHeader` 的悬浮提示/百分比。

3. **Overlay 弹窗拖拽框选误关闭修复 (`useBackdropClose.ts`)**
   - 为所有 `.modal-backdrop` 遮罩层（包括 `AppSettingsModal`、`DraftBatchDuplicateModal`、`EnvironmentManagerModal`、`CredentialsSettingsModal`、`StatementSnippetModal`、`RequestQuotaModal`、`LanQrScannerModal`、`ImportTaskModal`、`TaskDeleteConfirmModal`、`TaskAbortRefreshModal`、`EnvironmentDeleteConfirmModal`）统一接入 `mousedown` + `mouseup` 校验逻辑。
   - 仅当鼠标**按下与松开都在阴影遮罩区本身**时才关闭弹窗；在弹窗内部开始拖拽文本并拖到框外阴影区释放时，不再误关闭窗口，保留用户的文本选中状态。

---

### 二、Gemini 4 Hydrogen vs Gemini 4 Argon vs Gemini 3.8 Flash 对比与选型排序

#### 1. 核心区别与能力变化：是“新功能拓展”还是“模型本身水平提升”？

* **架构层面（同代共享，非全新模态拓展）**：
  * **Gemini 4 Hydrogen** 与 **Gemini 4 Argon**（以及 4 Carbon）共享同一代 Gemini 4 架构：统一的 **1M token 上下文窗口**、支持长输出（通过 continuation 最高 1M output tokens）、多模态输入（文本/图像/音频/视频/文件）及内置工具调用能力 [1.1.12, 1.2.11, 3.1.16]。
* **能力与水平提升（主要是模型智能度、Agent 执行力与性价比的跃升）**：
  * **相比 3.8 Flash**：Google 正式用 Hydrogen 取代 Flash 品牌 [3.2.8]。Hydrogen 的核心升级在于**模型本身智能水平与 Agent 可靠性的质变**——任务完成速度提升 **2.5 倍** [1.1.8, 3.1.11]、Terminal-Bench 4.1 编码得分从 18.2% 跃升至 **57%** [3.2.20]、缺失上下文时的幻觉/编造率降低 **5 倍以上** [3.1.22]，并增强了**并行工具调用批处理（parallel tool-call batching）**与 4 档思考深度控制（`low` / `medium` / `high` / `xhigh`）[1.1.12, 3.1.22]。
  * **相比 4 Argon**：Argon 发布于 2026 年 10 月，定位为早期前沿推理模型（$2/$10 每百万 tokens 起，部分列表为 $4/$20）[1.2.3, 1.2.7, 3.2.15]。而 11 月 12 日发布的 Hydrogen 在默认 `high` 思考深度下，Artificial Analysis Intelligence Index 跑分已反超 Argon（**51 vs 50**，`xhigh` 可达 **54**）[3.1.11, 3.1.14]，输出吞吐更快（约 198 tok/s vs 64 tok/s）[3.2.15]，且价格更低（**$1.50 / $7.50** 每百万 tokens）[3.1.14, 3.2.15]。Argon 仅在部分超长周期（long-horizon）多步任务中仍保有微弱的上下文保持优势 [3.2.19]。
  * **需要注意的短板**：Hydrogen（以及整个 Gemini 4 系列）的首字思考等待时间（TTFT）较长（Hydrogen `xhigh` 约 41.1s，Argon 约 39.9s），而 3.8 Flash 约 17.1s [3.2.13]；因此在“需要秒级首字流式响应的简单对话”场景下，Gemini 4 系列反而不如 3.8 Flash 轻快 [3.2.13]。

#### 2. 管理类似（Agentic 编码 / 多轮任务编排）项目的模型推荐排序

1. **第一推荐：`gemini-4-hydrogen`（Gemini 4 Hydrogen）**
   * **理由**：目前性价比、编码能力（Terminal-Bench 57%）、工具并行调用和任务完成速度综合最优的模型，在常规开发、多步 Agent 工作流、代码审查与重构中全面胜出，且费用显著低于 Argon [3.1.22, 3.2.15, 3.2.20]。
2. **第二推荐：`gemini-4-argon`（Gemini 4 Argon）**
   * **理由**：仅在少数需要超长周期（long-horizon）持续推理、且你发现 Hydrogen 在长链条下出现细节遗忘时作为备选；否则多数场景已被 Hydrogen 以更低成本超越 [3.2.15, 3.2.19]。（注：若预算充足且遇到极难架构设计、深层 Bug 或安全审计，同系列的 **Gemini 4 Carbon** 是更强的上限选择 [3.1.16, 3.2.15]。）
3. **第三推荐：`gemini-3.8-flash`（Gemini 3.8 Flash）**
   * **理由**：在 2027 年 1 月 1 日调价之前，3.8 Flash 价格（$0.75/$3.75）是 Hydrogen 的一半，且首字延迟更低，仅适合**高频、简单、低复杂度**的辅助任务或轻量分类/摘要；不建议用于复杂 Agent 编码（2027 年 1 月 1 日涨价至与 Hydrogen 同价后将无保留价值）[3.1.13, 3.1.15, 3.2.13]。

---

**参考来源：**
* [1] [Introducing Gemini 4 Hydrogen for Agentic Workflows - Google Blog](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-hydrogen/)
* [2] [Gemini 4 Hydrogen - API Pricing & Providers - OpenRouter](https://openrouter.ai/google/gemini-4-hydrogen)
* [3] [Gemini 4 Argon - API Pricing & Benchmarks - OpenRouter](https://openrouter.ai/google/gemini-4-argon)
* [4] [Gemini 4 Hydrogen vs Carbon vs Argon - TeamoRouter](https://teamorouter.com/blogs/gemini-4-hydrogen-vs-carbon-argon)
* [5] [Gemini 4 Hydrogen vs Gemini 4 - OrcaRouter](https://www.orcarouter.ai/blog/gemini-4-hydrogen-vs-gemini-4)
* [6] [Google's Gemini 4 Hydrogen runs 2.5x faster than 3.8 Flash - Crypto Briefing](https://cryptobriefing.com/gemini-4-hydrogen-faster-than-3-8-flash/)
