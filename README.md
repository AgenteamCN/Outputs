# 技术备忘录：通过 Gemini Environments API 跨容器读取/下载环境快照文件

整理时间：2026-11-26T07:02Z

## 1. 核心价值与适用场景
当旧的 Agent Cloud 容器已结束、上下文丢失或需要跨容器迁移 `/workspace/.chat/` 等持久化资产时，**无需重启旧对话**，只要拥有原环境的 `environment_id`（例如 `300cc1fb88c3d546c5b4861e4c594b7d`）和有效的 Gemini API Key，即可通过 **Gemini Environments Files REST API** 直接列举并下载该环境快照中的任意文件。

同时，在 Interactions API 中，`environment="<env_id>"` 参数与 `previous_interaction_id` 完全独立，可在不依赖旧对话历史的情况下挂载或读取指定环境快照。

## 2. API 接口规范（`v1beta`）

Base URL: `https://generativelanguage.googleapis.com`  
Auth Header: `x-goog-api-key: <GEMINI_API_KEY>`（或 `?key=<GEMINI_API_KEY>`）

### 2.1 列举目录 / 获取文件元数据
- **Endpoint**:
  ```
  GET /v1beta/environments/{environment_id}/files/{path}
  ```
- **Path 说明**:
  - `path` 为相对于环境根目录（`/`）或工作区的路径。例如要读取 `/workspace/.chat` 下的文件，可传 `workspace/.chat`。
- **Query 参数（注意必须使用 snake_case）**:
  - `recursive=true`：递归列出目录下所有文件与子目录（非常关键，避免逐层遍历）。
  - `page_size=200`：每页最大条目数（注意：**不能**用驼峰 `pageSize`，否则会报参数无效）。
  - `page_token=<token>`：分页游标（当返回结果包含 `nextPageToken` / `next_page_token` 时继续翻页）。

### 2.2 下载单个文件原始内容
- **Endpoint**:
  ```
  GET /v1beta/environments/{environment_id}/files/{file_path}?alt=media
  ```
- **关键注意事项**:
  - 必须附带 `?alt=media` 查询参数（或直接使用列举接口返回的 `downloadUrl`），否则接口默认只返回 JSON 格式的文件元数据，而非文件原始字节流。
  - 若路径包含 `.` 开头的隐藏目录（如 `workspace/.chat/...`），在 URL 中直接保留或正常 URL 编码即可。

## 3. 快速实操示例（`curl` / `python3`）

### 3.1 递归列举旧环境 `/workspace/.chat`
```bash
curl -sS \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  "https://generativelanguage.googleapis.com/v1beta/environments/${ENV_ID}/files/workspace/.chat?recursive=true&page_size=200"
```

### 3.2 批量下载到本地恢复目录
```python
import json, os, urllib.request, urllib.parse

API_KEY = os.environ["GEMINI_API_KEY"]
ENV_ID = "300cc1fb88c3d546c5b4861e4c594b7d"
BASE = f"https://generativelanguage.googleapis.com/v1beta/environments/{ENV_ID}/files"

def api_get(url: str) -> bytes:
    req = urllib.request.Request(url, headers={"x-goog-api-key": API_KEY})
    with urllib.request.urlopen(req, timeout=30) as resp:
        return resp.read()

# 1. 递归列举（支持分页）
entries, page_token = [], None
while True:
    qs = {"recursive": "true", "page_size": "200"}
    if page_token:
        qs["page_token"] = page_token
    url = f"{BASE}/workspace/.chat?{urllib.parse.urlencode(qs)}"
    data = json.loads(api_get(url).decode())
    entries.extend(data.get("files", []))
    page_token = data.get("nextPageToken") or data.get("next_page_token")
    if not page_token:
        break

# 2. 过滤非目录条目并加上 ?alt=media 下载原始内容
for item in entries:
    if item.get("isDirectory"):
        continue
    rel_path = item["path"]  # 例如 workspace/.chat/master_backlog.md
    raw = api_get(f"{BASE}/{urllib.parse.quote(rel_path, safe='/')}?alt=media")
    out_path = os.path.join("/workspace/.chat/previous/restored", rel_path)
    os.makedirs(os.path.dirname(out_path), exist_ok=True)
    with open(out_path, "wb") as f:
        f.write(raw)
```

## 4. SDK 参考（`google-genai` 3.6.1）
本地预装的 Python `google-genai` 包已在 `google/genai/_gaos/files.py` 中封装了该能力：
- `client.files.list(environment=..., path=..., page_size=200, recursive=True)`
- 底层路径模板：`/{api_version}/environments/{environment}/files/{path}`

## 5. 安全操作规范（必守）
1. 若用户为跨容器拉取临时签发了 Gemini API Key，**拉取完成后必须立即通知用户吊销该临时 Key**。
2. 严禁将任何 `AIza...` API Key 写入 `/workspace/.chat/`、Git 仓库或备份网关日志中；仅限在当前进程内存或临时变量中使用。
3. 恢复后的旧环境文件建议统一归档至 `/workspace/.chat/previous/<env_short_id>/`，并在 `/workspace/.chat/recover/recovery_index.md` 中登记索引。
