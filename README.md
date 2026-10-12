# Zotero Agent Skills

> 让 AI agent（Codex / Claude Code / 任意 shell agent）直接检索、深读、综述你的**本地 Zotero 文献库**。
>
> Agent skills for Zotero: hybrid retrieval (semantic + BM25), cached full-text reading,
> and citation-ready synthesis — all local-first.

[为什么](#为什么) · [技能一览](#技能一览) · [架构](#架构) · [安装](#安装) · [配置](#配置) · [使用](#使用) · [常见问题](#常见问题) · [设计笔记](#设计笔记) · [发布到-GitHub](#发布到-github)

---

## 为什么

研究者的日常痛点：

- 想读一篇论文，要切窗口、找 PDF、翻半天；
- 让 AI "综述某个主题"时，它靠的是训练记忆，**不是你的文献库**；
- 通用 RAG 方案要自建一整条管线，且与 Zotero 工作流割裂。

本技能集把"你的 Zotero 库"变成 agent 的一等公民，用**两个技能**覆盖两类核心场景：

## 技能一览

| 技能 | 触发场景 | 做什么 |
|---|---|---|
| **zotero-read-paper** | "读一下《…》"、给 DOI / 关键词 | 检索定位 → MinerU Markdown 缓存命中（秒级）→ 未命中自动在线解析 → 输出核心信息摘要 |
| **zotero-library-search** | "从我库里找 / 综述关于 X 的研究" | 两级检索（核心库 → 全库兜底）→ 选篇 → 取全文 → 带 `[作者 年份]` 引用的综合回答 |

两者分工：**read-paper 负责"读某一篇"，library-search 负责"从全库找一组并综合"**。

## 架构

```
[检索层]  语义向量（如 BAAI/bge-m3）+ BM25 关键词 —— RRF 融合
   ├─ 核心库层：本地索引，秒级响应，不需要 Zotero 开着
   └─ 全库层  ：Zotero 插件 / MCP 服务，覆盖面更广（需 Zotero 桌面版运行）

[读取层]  MinerU Markdown 缓存命中（秒级）
          → 未命中：MinerU 在线解析（精准 → 轻量）→ 自动存缓存
          → 全部失败：PDF 直读兜底（MCP get_content 等）

[综合层]  agent 精读 3–5 篇命中文献 → 带章节/年份引用的结构化回答
```

**设计原则：检索与保真分离。** 检索层允许粗糙（只影响召回），引用与数据永远回到干净全文；
缓存优先（省 token、秒级）；一切可选依赖都带降级路径。

## 安装

### 前置条件

- **Zotero 7+ 桌面版**（设置 → 高级 → 勾选"允许其他应用与本机 Zotero 通信"）
- **Python ≥ 3.10**
- 可选（强烈建议）：
  - [MinerU](https://mineru.net) API Token —— 高质量 PDF → Markdown 解析；无 Token 时自动退化为 PDF 直读；
  - 任意 OpenAI 兼容 embedding 服务（如 SiliconFlow / OpenAI）或本地模型 —— 用于语义检索；
  - Zotero 侧插件：[cookjohn/zotero-mcp](https://github.com/cookjohn/zotero-mcp)（库内 MCP 服务）、llm-for-zotero（自动转 MD）等。

### 安装技能（复制目录即可）

```bash
git clone <this-repo> && cd zotero-agent-skills

# Codex / 读取 ~/.agents/skills 的 agent
cp -r skills/* ~/.agents/skills/

# Claude Code
cp -r skills/* ~/.claude/skills/

# 其他支持 Skills 的 agent：复制到其技能目录即可（无破坏性，纯文档）
```

Windows 下把 `~` 换成 `C:\Users\<you>`。

### 任意 shell agent（不需要安装）

把本 README 放进该 agent 的常驻提示，或直接让它执行下面"使用"一节的命令。

## 配置

技能中的示例使用如下占位符，请替换为你自己的路径：

| 占位符 | 含义 | 示例 |
|---|---|---|
| `$ZOTERO_MCP_HOME` | 配套工具脚本目录 | `C:\Users\<you>\.zotero-mcp` |
| `$ZOTERO_DATA` | Zotero 数据目录（含 `storage\`） | `C:\Users\<you>\Zotero` |
| `$MINERU_CACHE` | MinerU 解析缓存根目录 | `$ZOTERO_DATA\llm-for-zotero-mineru` |
| `MINERU_API_KEY` | MinerU Token（环境变量或文件） | 见 `skills/zotero-read-paper/mineru_api_key.txt` |

> 本仓库只包含**技能文档**（`SKILL.md`）。技能引用的配套脚本
> （`zread.py` 取文 / `zsearch.py` 两级检索 / `core_build.py` 建库）为参考实现，
> 可按 README 的架构自建——或等待本仓库后续开源（见 Roadmap）。

## 使用

配置好工具脚本后，日常只需对 agent 说人话：

- "**读一下《标题》**" / 给 DOI → 走 read-paper 流程；
- "**从我库里综述关于 X 的研究，引用到章节**" → 走 library-search 流程；
- "**找跟《标题》类似的文献**" → 相似检索。

配套脚本的典型命令（占位符替换后）：

```bash
# 取全文：输出 full.md 绝对路径
python "$ZOTERO_MCP_HOME/zread.py" get <itemKey>

# 两级语义检索（核心库优先 + 全库兜底）
python "$ZOTERO_MCP_HOME/zsearch.py" semantic "cryogenian manganese" --top 10

# 术语/符号精确检索（BM25，秒级）
python "$ZOTERO_MCP_HOME/zsearch.py" terms "LA-ICP-MS"

# 相似文献
python "$ZOTERO_MCP_HOME/zsearch.py" similar <itemKey>
```

输出约定：

- `RESULT/SAVED <路径>` = 取文结果（直接读取）；
- 来源标记 `[core]` / `[full]` / `[core+full]`；命中方式 `(v)` 向量 / `(b)` BM25 / `(vb)` 双命中；
- `--json` 输出结构化数据，便于程序化消费。

## 常见问题

**没有 MinerU Token / 不想用在线解析？**
技能自动降级为 PDF 直读；质量略降（可能有双栏/公式拼接问题），流程不中断。

**引用的文献没被检索到？**
检索是"精确召回"问题，常见原因：该文献未进索引（未解析 / 不在核心集合）。
两级设计的全库层用于兜底；回答时应说明覆盖率而不是假装"库里没有"。

**数据与隐私？**
全本地运行。只有在你配置了 MinerU / embedding 服务时才会把**待解析的 PDF 或待嵌入的文本**
发往对应服务；没有任何遥测。参考实现为单机、文件级存储，数据不出你指定的范围。

**支持 macOS / Linux？**
技能流程与平台无关；命令中的路径请按平台调整（脚本为 Python，可跨平台）。

**为什么不做成"一个万能 skill"？**
两类任务的上下文与流程差异很大（单篇深读 vs 跨文献综合），拆分能让触发更准、
上下文更省——agent 技能的本质是"按场景加载的说明书"。

## 设计笔记

1. **两级检索，互补而非替代**：核心库（小而干净）负责精度与速度；全库层负责覆盖与兜底。
2. **向量 + BM25 混合（RRF 融合）**：术语型查询（化学式、方法缩写、地名、编号）不掉召回；
   语义型查询不变差——实测中"漂移"的正主往往被 BM25 捞回。
3. **检索与保真分离**：RAG 片段只用于定位；引用、数字、公式一律回到 MinerU Markdown / 原文。
4. **缓存优先 + 自动维护**：读过的文献秒级复用；索引更新做成后台增量任务，人工环节为零。

## Roadmap

- [ ] 开源参考实现脚本（`zread.py` / `zsearch.py` / `core_build.py`，含环境变量配置化）
- [ ] 通用 MCP server 封装（一次配置，多 agent 复用）
- [ ] 更多语言的 SKILL.md 版本

## 发布到 GitHub

```bash
cd zotero-agent-skills
git init && git add -A && git commit -m "Zotero agent skills: read-paper + library-search"
git remote add origin https://github.com/<you>/zotero-agent-skills.git
git push -u origin main
```

建议为仓库加 topics：`zotero` `agent-skills` `rag` `claude-code` `codex` `literature-review`。

## 致谢与许可

- 解析：[MinerU](https://github.com/opendatalab/MinerU)（opendatalab）
- 嵌入模型示例：[BAAI/bge-m3](https://huggingface.co/BAAI/bge-m3)；BM25 实现：[bm25s](https://github.com/xhluca/bm25s)
- Zotero 生态：[cookjohn/zotero-mcp](https://github.com/cookjohn/zotero-mcp)、[54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp)
- 本技能集的公开版本由作者整理分享，许可：见 `LICENSE`（默认 MIT）。
