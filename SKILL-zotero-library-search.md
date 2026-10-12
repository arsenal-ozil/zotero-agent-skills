---
name: zotero-library-search
description: >
  对 Zotero 文库做库级检索（语义相似 + 全文关键词）并综合多篇文献回答问题。当用户要求
  "从我的库里找/综述关于 X 的文献、证据、研究"、"库里还有什么相关文献"、"跨文献比较/梳理"，
  或给出主题词要求全库检索（papers in my library about X, literature review from my library）时触发。
  与 zotero-read-paper 分工：那个负责读"某一篇"，本技能负责"从全库找一组文献并综合"。
---

# Zotero 文库检索与综述

## 能力与前提

- **两级检索**：
  ① 核心库（本地）：按主题收藏夹圈定的核心文献（MinerU 干净解析，向量 + BM25 双索引，带章节信息）；
  ② 全库层（Zotero MCP 插件或服务）：覆盖更广的语义索引（需 Zotero 桌面版运行）。
- `--index auto`（默认）先查核心库、自动并回全库兜底；**核心库不需要 Zotero 开着**，
  全库级需要。Zotero 未开时 auto 自动退化为仅核心库。
- 检索工具（固定路径，两个 agent 通用）：`C:\Users\<you>\.zotero-mcp\zsearch.py`（用 `python` 运行）。
- 维护：核心库配了**自动增量计划任务**（`ZoteroCoreRAG-Update`，登录后每 6 小时尝试一次，Zotero 关闭则跳过）；
  一般无需手动（含 BM25 索引自动重建）。需要手动时跑 `python "C:\Users\<you>\.zotero-mcp\core_build.py"`（增量，秒~分钟级）；`--force` 才全量重建。

## 第一步：检索

把用户问题拆成 2–4 个互补查询串（主题、机制、材料/地点、指标各一），中英各跑一轮更稳。

```powershell
python "C:\Users\<you>\.zotero-mcp\zsearch.py" ping
python "C:\Users\<you>\.zotero-mcp\zsearch.py" semantic "cryogenian manganese" --top 10
python "C:\Users\<you>\.zotero-mcp\zsearch.py" semantic "continental weathering lithium isotope" --top 8
python "C:\Users\<you>\.zotero-mcp\zsearch.py" terms "LA-ICP-MS" --top 8   # 术语/符号精确检索（秒级）
```

- `semantic <query>`（默认 `--index auto`）：两级合并输出，每条带 `[core]` / `[full]` / `[core+full]` 标记；核心库侧已为**向量+BM25 融合**，`[core]` 条目附 `(v)`/`(b)`/`(vb)` 指明命中来源；`--index core|full` 可强制单查。
- `terms "<术语>"`：BM25 精确检索（化学式、方法缩写、地名、编号、数字），本地秒级，适合符号型查询。
- `similar <itemKey>` / `passages "<精确短语>"`：插件全库功能（passages 较慢，约 1–3 分钟）。
- 打分口径：`[full]` 条目=余弦相似度（>0.55 通常高度相关，可用 `--min-score` 放宽）；`[core]` 条目=RRF 融合分（相对排序，无绝对阈值）。

## 第二步：筛选与确认

- 依据命中片段挑 3–8 篇代表性文献：覆盖不同观点/时段/方法，别只挑高分。
- 除非用户说"直接综述"，先给候选清单（标题 + 年份 + 一句入选理由）让用户确认。

## 第三步：取全文（一条命令）

```powershell
python "C:\Users\<you>\.zotero-mcp\zread.py" get <itemKey>
```

- 输出 `RESULT/SAVED <路径>` 即 `full.md` 绝对路径，直接读取；未命中时会自动走 MinerU 在线解析并存入缓存。
- 脚本报错时回退：插件 `get_content(itemKey=…, mode="complete")` 或独立版 `zotero_get_item_fulltext`。
- 单轮精读 3–5 篇为宜；更多时先快速扫描（开头/摘要）再挑重点精读。

## 第四步：综合回答

- 结构：按主题 / 时间线 / 争论点组织要点 → 每条挂 `[作者 年份]` → 末尾"来源清单"
  （完整标题 + itemKey，便于回跳 Zotero）。
- 明确区分"文献原文说了什么"与"基于文献的推断"。
- 结尾注明覆盖面：全库层语义索引覆盖已解析/有全文的那批文献；未命中 ≠ 库里没有。

## 边界情况

- **Zotero 没开** → auto 仍可用核心库（全库级自动跳过）；可提醒用户开 Zotero 以获得全库覆盖。
- 预期内的重复条目（同一论文被收录多次）→ 脚本已按标题去重；跨查询的重复人工合并。
- 检索不到相关文献 → 英文复述换词、降 `--min-score`、或改用 `terms`（术语）/ `passages`（精确短语）。
- 用户随后要读单篇细节 → 转 zotero-read-paper 流程。
