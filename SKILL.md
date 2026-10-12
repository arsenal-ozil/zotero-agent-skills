---
name: zotero-read-paper
description: >
  从 Zotero 查找论文并读取全文。当用户给出论文标题、关键词或 DOI 时触发。
  优先走 MinerU MD 缓存（质量更高、省上下文），无缓存时自动用 MinerU 精准解析 API
  在线解析 PDF，失败再回退轻量解析 API，最后才 PDF 直读。适用于地质学、地球化学等
  领域的学术文献查询。
---

# Zotero 论文读取流程

## 第一步：搜索 Zotero

使用 MCP 工具 `mcp__zotero-mcp__zotero_search_items` 搜索用户给出的论文标题或关键词。

- 参数：`query` 必填；`qmode` 用 `titleCreatorYear`（默认，标题/作者/年份）或 `everything`（含摘要等）；`limit` 控制条数（默认 10）
- 标题不确定时先用 `titleCreatorYear` 小步尝试，找不到再换 `everything`

从结果中提取：

- **`itemKey`** — 条目唯一 key（结果里 `**Item Key:**` 后的值，如 `ABCD1234`）
- **`attachmentKey`** — PDF 附件的 key。用 `mcp__zotero-mcp__zotero_get_item_children`（`item_key=itemKey`）列出子条目，取 `itemType == "attachment"` 且 `contentType` 为 PDF 的 `key`（如 `EFGH5678`），用于匹配 MinerU 缓存、定位本地 PDF
- **标题、作者、年份** — 用于确认找到的是正确文献

如果结果不明确，向用户确认后再继续。

## 第二步：取全文（一条命令 + 兜底）

首选封装脚本（自动完成：缓存查找 → 精准解析 → 轻量解析 → 自动保存缓存）：

```powershell
python "C:\Users\<you>\.zotero-mcp\zread.py" find <itemKey|attachmentKey>   # 只查缓存，秒级
python "C:\Users\<you>\.zotero-mcp\zread.py" get  <itemKey>                # 未命中则在线解析（几分钟）
```

- 输出里 `RESULT <路径>` / `SAVED <路径>` 行为 `full.md` 绝对路径，直接读。
- 精准解析 Token 由脚本自动读取（环境变量 `MINERU_API_KEY` 或技能目录 key 文件），绝不回显。
- 脚本报错时按下面的手动兜底。

### 手动兜底（zread.py 不可用时）

1. **缓存查找**：在 `C:/Users/<you>/Zotero/llm-for-zotero-mineru/` 下扫 `_llm_source.json`：

   ```powershell
   Get-ChildItem "C:/Users/<you>/Zotero/llm-for-zotero-mineru" -Recurse -Filter _llm_source.json |
     Select-String -Pattern "ITEMKEY|ATTACHMENTKEY" -List | Select-Object -ExpandProperty Path
   ```

   命中 → 直接读该目录 `full.md`（图片在同目录 `images/`）。

2. **在线解析**：MinerU 精准 API（`POST https://mineru.net/api/v4/file-urls/batch` → PUT 上传 → 轮询 `GET /api/v4/extract-results/batch/{batch_id}`，Header 带 `Authorization: Bearer {token}`）；失败退轻量 API（`POST https://mineru.net/api/v1/agent/parse/file` → PUT 上传 → 轮询 `GET /api/v1/agent/parse/{task_id}`）。成功后按 `zread.py` 的格式存缓存目录（编号 + `full.md` + `_llm_source.json`）。

3. **PDF 直读**（最后保底）：本地 PDF 在 `C:/Users/<you>/Zotero/storage/{attachmentKey}/`；文本读取用 MCP `mcp__zotero-mcp__zotero_get_item_fulltext`（`item_key=…`），或插件 `get_content(itemKey=…, mode="complete")`。可能含全角字符、双栏拼接错乱，需清理后呈现。

## 第三步：呈现给用户

- 优先呈现结构清晰的内容：核心信息摘要（研究问题、方法、关键数据、结论）
- 告知数据来源：MinerU MD 缓存 / 精准解析 API / 轻量解析 API / PDF 直读
- 需要看图时提醒：截图在 `full.md` 同目录 `images/` 下

## 边界情况

- **Zotero 中未找到该论文** → 告知用户，建议先添加到 Zotero
- **缓存未命中** → `zread.py get` 自动在线解析，无需用户干预
- **在线解析全失败 / 超时** → 自动回退 PDF 直读；内容被截断时说明已尽力覆盖
