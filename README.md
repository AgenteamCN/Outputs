# GitHub App 细粒度权限隔离与私有仓库主分支保护（Verified Commit 锁）可行性调研报告

## 1. 核心结论
1. **GitHub 原生 Branch Protection / Rulesets（在当前免费组织私有仓库下）**：**不可行**（GitHub Free 组织的私有仓库不支持启用 Branch Protection 与 Repository Rulesets，API 实测返回 HTTP 403）。
2. **GitHub App 原生 Token 权限隔离**：**不支持按分支隔离**，GitHub App Installation Token 最小粒度为「仓库级（Repository-scoped）」。
3. **零额外付费的推荐落地方案**：**在 `agyWorker` 服务端实现 GitHub Commit Proxy（`POST /agents/github-commit`）**，即可在不升级 GitHub 付费套餐的前提下，实现**按分支隔离（禁止 Partner 直写 `a2`/`main`）+ 强制 Verified Bot Commit**。

---

## 2. 实测 API 证据（只读查询 `Antigravi-ty/agyDevKit` 与 `Antigravi-ty/agyWorker`）
- **仓库可见性与所属组织**：两个仓库均为 Organization `Antigravi-ty` 下的 `private` 仓库。
- **Repository Rulesets 检查**（`GET /repos/Antigravi-ty/agyDevKit/rulesets` 与 `GET /repos/.../rules/branches/{a2|main}`）：
  - 返回 `HTTP 403`：`"Upgrade to GitHub Pro or make this repository public to enable this feature."`
- **Branch Protection 检查**（`GET /repos/.../branches/{a2|main}/protection`）：
  - 返回 `HTTP 403`：`"Resource not accessible by integration"`（同时受限于免费组织私有仓限制及当前 App 未授予 `administration: write`）。
- **Installation Token 权限粒度检查**（`GET /installation/repositories`）：
  - 当前 App 授权覆盖 35 个仓库，签发出的 `ghs_...` token 携带仓库级 `contents: write` 权限。只要持有该 raw token，客户端理论上可以向该仓库任意分支（包括 `a2`/`main`）发起提交或合并，无法仅靠 GitHub App token 本身禁止推送主分支。

---

## 3. 两种可行改造方案对比

### 方案 A：GitHub 原生方案（需付费升级，成本较高）
1. 将组织 `Antigravi-ty` 升级至 GitHub Team / Enterprise 付费计划（解锁私有库 Rulesets / Branch Protection）。
2. 拆分为两个独立 GitHub App：`Partner App`（仅用于开发分支）与 `Chief/Merge App`（专门用于合并）。
3. 在 `a2` / `main` 上配置 Ruleset：开启 `Require signed commits` + `Restrict updates`（仅 Bypass list 中的 `Chief/Merge App` 可更新主分支）。

### 方案 B：agyWorker 服务端代理方案（零额外付费、立即可落地，强烈推荐）
1. **收敛 Partner 的 raw GitHub Token**：
   - `POST /agents/github-token` 对 Partner 仅返回只读 token（或不再直接下发具备写权限的 raw `ghs_...` token）。
2. **新增 `POST /agents/github-commit` 代理接口（Worker 侧代签）**：
   - Partner 在本地完成修改与 2x 门禁测试后，将待提交的文件变更（或 diff/tree）与目标分支名发给 Worker `POST /agents/github-commit`；
   - Worker 严格校验调用方身份的 `allowedBranches`（例如仅放行 `a2-*`、`feat/*`、`fix/*`，**硬拒绝直写 `a2` 和 `main`**）；
   - 校验通过后，由 Worker 使用服务端持有的 App Token 调用 GitHub GraphQL `createCommitOnBranch`，自动生成带 GPG 签名的 **Verified Bot Commit** 并返回 commit SHA。
3. **主分支合并门禁（Chief 专属）**：
   - `POST /repos/<owner>/<repo>/merges` 或等效合并接口仅限 `chief`（`isAdmin: true`）调用；
   - 合并前检查源分支最新 commit 满足 `signature.verified == true` 且已记录独立 Cross-Review PASS。
