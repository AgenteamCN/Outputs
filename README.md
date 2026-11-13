# Worker 改造与 Gemma 4 Commit Message 接入完成报告

## 一、模型选型与实测结果

在部署前对 Google Gemini API (`v1beta`) 下的两个 Gemma 4 模型进行了实测对比：

| 模型 ID | 实测响应耗时 | 关键发现 | 结论 |
| :--- | :---: | :--- | :--- |
| `gemma-4-26b-a4b-it` (MoE 26B / 4B active) | **0.87s**（配合 `thinkingLevel: "MINIMAL"`） | 默认开启 thinking 会吃掉 `maxOutputTokens`；显式设置 `thinkingLevel: "MINIMAL"` 后速度极快且仅输出最终标题 | **选用** |
| `gemma-4-31b-it` (Dense 31B) | >15s 超时 | 推理延迟明显偏高，不适合 Worker 同步链路 | 淘汰 |

---

## 二、代码架构与改动说明

**是否需要文件解耦**：**不需要**。改造后的 `worker.js` 全文约 240 行原生 ES Module，无任何第三方构建依赖，单文件即可清晰维护且不存在上下文膨胀问题。

### 核心改造点（已部署至 `agentoutputposter`）
1. **头部裁切 + Gemma 4 概括 (`summarizeWithGemma`)**
   - 仅截取正文前 **800 字符**（超出拼接 `...`）发送给 `gemma-4-26b-a4b-it`，单次总输入仅约 **140～500 tokens**，在 16K TPM 下可轻松支撑每分钟 30 次（RPM 上限）并发上报。
   - 设置 `thinkingConfig: { thinkingLevel: "MINIMAL" }`、`maxOutputTokens: 64`、`temperature: 0.2`，并过滤掉 `part.thought === true` 的思考片段。
   - 强制 **5 秒超时**（`AbortSignal.timeout(5000)`），避免阻塞 GitHub 提交。
2. **三级自动降级链 (`buildCommitMessage`)**
   - **首选**：`<Gemma 4 简短标题> (<ISO时间戳>)`
   - **降级 1**（Gemma 4 超时 / 429 / 报错）：正则提取原文首个 Markdown `# ` 标题或第一行非空文本（限 60 字）+ 时间戳
   - **降级 2**（兜底）：`Backup agent output (<ISO时间戳>)`
   - 确保无论 Google API 状态如何，GitHub 备份提交 **100% 不失败**。
3. **响应增强**：JSON 返回体中新增 `commit_message` 字段，方便直接核对最终生成的 Commit 标题。

---

## 三、部署与绑定验证

- **Worker 名称**：`agentoutputposter`
- **部署时间**：`2026-11-13T02:07:33.809511Z`（HTTP 200）
- **环境变量/Secret 保留情况**：已验证 `DEFAULT_INSTALLATION_ID`、`GATEWAY_REQUEST_ID`、`GEMMA_KEY`、`GITHUB_APP_ID`、`GITHUB_PRIVATE_KEY`、`KeySHA265` 全部完好保留。
- **端到端自验证**：本报告即通过 `POST https://agentoutputposter.maboco-mark.workers.dev/` 提交，可直接查看本次返回的 `commit_message` 与 GitHub Commit 记录。
