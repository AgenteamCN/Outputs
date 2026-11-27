# 文叔叔 (Wenshushu) API 下载方案 & 本次任务结果备份

## 1. 文叔叔 (c.wss.ink / wenshushu.cn) 匿名下载 API 原理与实现

通过分析文叔叔前端 JS（`app.ff50e1f6.js` 中的模块 `fca7` 和 `7d40`），无密码公开分享链接（如 `https://c.wss.ink/f/obantulwkfw` -> `https://www.wenshushu.cn/f/obantulwkfw`，`tid = obantulwkfw`）可通过以下 4 步纯 HTTP 接口直接获取直链并下载：

1. **获取匿名 Token**：`POST https://www.wenshushu.cn/ap/login/anonymous`
   - 请求体：`{"dev_info": "{}"}`
   - 返回：`data.token`，后续请求在请求头携带 `x-token: <token>`
2. **获取任务信息**：`POST https://www.wenshushu.cn/ap/task/mgrtask`
   - 请求体：`{"tid": "<task_id>", "password": ""}`
   - 返回：`data.boxid`、`data.ufileid`
3. **获取文件列表**（新版接口为 `/ap/ufile/nlist`，非旧版 `/ap/ufile/list`）：
   - `POST https://www.wenshushu.cn/ap/ufile/nlist`
   - 请求体：
     ```json
     {
       "start": 0,
       "sort": {"name": "asc"},
       "bid": "<boxid>",
       "pid": "<ufileid>",
       "options": {"uploader": "true"},
       "size": 50
     }
     ```
   - 返回：`data.fileList[0].fid`、`data.fileList[0].fname`
4. **签发下载直链**：`POST https://www.wenshushu.cn/ap/dl/sign`
   - 请求体：`{"ufileid": "<fid>", "kid": "", "type": 1}`
   - 返回：`data.url`（CDN 签名直链，直接 GET 流式下载即可）

### 完整 Python 参考脚本

```python
import os
import requests

def download_wenshushu(tid: str, save_dir: str = "/workspace") -> str:
    s = requests.Session()
    s.headers.update({
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
    })
    tok = s.post("https://www.wenshushu.cn/ap/login/anonymous", json={"dev_info": "{}"}).json()["data"]["token"]
    s.headers.update({"x-token": tok})

    mgr = s.post("https://www.wenshushu.cn/ap/task/mgrtask", json={"tid": tid, "password": ""}).json()["data"]
    boxid, ufileid = mgr["boxid"], mgr["ufileid"]

    nlist = s.post("https://www.wenshushu.cn/ap/ufile/nlist", json={
        "start": 0,
        "sort": {"name": "asc"},
        "bid": boxid,
        "pid": ufileid,
        "options": {"uploader": "true"},
        "size": 50,
    }).json()
    item = nlist["data"]["fileList"][0]
    fid, fname = item["fid"], item["fname"]

    dl = s.post("https://www.wenshushu.cn/ap/dl/sign", json={"ufileid": fid, "kid": "", "type": 1}).json()
    url = dl["data"]["url"]

    out_path = os.path.join(save_dir, fname)
    with s.get(url, stream=True) as resp:
        resp.raise_for_status()
        with open(out_path, "wb") as f:
            for chunk in resp.iter_content(chunk_size=1024 * 1024):
                f.write(chunk)
    return out_path
```

---

## 2. 本次统计任务产出结果

1. **2026 秋季/第一份数据（`学生课程注册明细20261126152146.csv.zip`）**
   - 原始记录：265,509 条
   - 汇总结果：1,499 行（52 个学习中心），总费用 `7,022,127.5`
   - Pixeldrain 下载：https://pixeldrain.com/u/ZcKbDpoE

2. **2026 春季/第二份数据（`2026春季学生课程注册明细（掰各家）20260801.xlsx.zip`）**
   - 原始总表（`合5008715元`）：190,698 条（经核对与其余 45 个分表总行数完全一致）
   - 过滤规则：剔除“选课次数”非“1”及“课程名称”含“#”的记录
   - 汇总结果：1,378 行（51 个学习中心），总费用 `5,008,715` 元（与原表名完全一致）
   - Pixeldrain 下载：https://pixeldrain.com/u/E93goekA
