# 📚 paper-hub

> **用 GitHub Issues + Actions + LLM 打造的「arXiv 论文 / 技术博客」阅读‑笔记‑推荐一体化系统**
>
> * **数据来源**：arXiv、各类技术博客。
> * **核心理念**：笔记 = Issue，Issue = 结构化数据行 → 可编程、可追溯、便于协作。

---

## 🌟 功能亮点

| 功能              | 说明                                                                          |
| --------------- | --------------------------------------------------------------------------- |
| **统一模板**        | 通过 `ISSUE_TEMPLATE/reading.yml` 约束「题目 / 链接 / 关键词 / 方法 / 摘要 / 思考」等字段         |
| **标签体系**        | `source/{arxiv,blog}`、`status/{todo,reading,done}`、`field/{LLM,CV,NLP,…}` 等 |
| **Projects 看板** | 表格视图 = 数据库；可按「状态 / 评分 / 关键词」过滤排序                                            |
| **自动周报**        | `summarize_cluster.yml`：每周 UTC‑0 周日 00:00 把最近 10 篇笔记聚合成一条 Digest Issue      |
| **智能推荐**        | `recommend_next.yml`：每天基于向量检索 + arXiv API 推荐 5 篇相关文章并自动起草 Issue             |
| **本地备份**        | `sync_to_obsidian.yml`：push 后同步 Markdown → Obsidian Vault                   |

---

## 🗂️ 仓库结构

```text
paper-hub/
├─ .github/
│  ├─ ISSUE_TEMPLATE/
│  │   └─ reading.yml        # 单篇阅读模板（见下文示例）
│  ├─ workflows/
│  │   ├─ summarize_cluster.yml   # 周聚合摘要
│  │   ├─ recommend_next.yml      # 每日推荐
│  │   └─ sync_to_obsidian.yml    # 笔记双向同步
│  └─ project.yml            # Projects Beta 配置文件
├─ scripts/
│  ├─ summarize.py           # 调 OpenAI GPT-4o 生成聚合摘要
│  ├─ recommend.py           # 调 arXiv / OpenAlex + 向量检索产出推荐
│  └─ utils.py               # 公共工具函数，集中管理参数
├─ README.md                 # ← 你正在看的这个文件
└─ LICENSE
```

---

## 📝 Issue 模板示例 (`reading.yml`)

```yaml
name: 📑 阅读笔记
title: "[Paper] ${title}"
labels: [paper, "source/arxiv"]
body:
  - type: input
    id: url
    attributes:
      label: 原文链接 (arXiv / Blog)
  - type: input
    id: authors
    attributes:
      label: 作者
  - type: input
    id: tags
    attributes:
      label: 关键字 (逗号分隔)
  - type: textarea
    id: summary
    attributes:
      label: 摘要 & 主要贡献 (150~300 字)
  - type: textarea
    id: critique
    attributes:
      label: 个人思考 / 质疑 / TODO 实验
  - type: dropdown
    id: rating
    attributes:
      label: 推荐星级
      options: [⭐, ⭐⭐, ⭐⭐⭐, ⭐⭐⭐⭐, ⭐⭐⭐⭐⭐]
  - type: markdown
    attributes:
      value: |
        <!--
        kg_triples: 自动写入 (YAML 三元组；勿手改)
        -->
```

* **隐藏字段**：创建 Issue 后，`recommend_next.yml` 会调用 `scripts/utils.py` 生成文本 embedding 并写入隐藏注释块，供相似度检索。

---

## 🕸️ 知识图谱流水线

本仓库采用“**摘要 → 标签/三元组 → Neo4j 图数据库 → 图 + 向量混合检索**”的轻量方案。

1. **抽取** (`abstract_tag.yml`)：对新建 Issue

   1. GPT‑3.5 -turbo 将正文截断为 ≤200 token 摘要；
   2. 同一次调用输出 **YAML 标签** (method／task／dataset…)；
   3. 再调用一个小模型将摘要转换为 **RDF 三元组** 写入隐藏注释 `<!-- kg_triples: ... -->`。
2. **入库** (`graph_ingest.yml`)：每日 cron

   1. 解析所有新 `kg_triples`；
   2. 用 `neo4j-admin import` 或 Cypher `MERGE` 把实体/关系写入本地 Docker Neo4j；
   3. 同时生成 **稀疏标签向量** 存进 `data/vectors.parquet` 供快速 KNN。
3. **推荐** (`recommend_next.yml`)

   1. 最近 N 篇 Issue → 聚合其标签向量求质心；
   2. 用 pgvector/HNSW 检索候选 50 篇；
   3. 在 Neo4j 图中计算两跳共现度 + PageRank 重排序；
   4. 生成 `[Suggest]` Issue 草稿。
4. **周报** (`summarize_cluster.yml`)：与旧方案相同，但附加一张 **知识图谱子图** PNG（由 `scripts/draw_subgraph.py` 读取 Neo4j、NetworkX 绘制）。
5. **同步** (`sync_to_obsidian.yml`)：保持不变。

> **费用**：每日摘要+标签+三元组约 500 token/篇，周 50 篇 ≈ 25 k token ⇒ 0.013 USD；Neo4j 本地运行，零云成本。

---

\----- | ---- | -------- | ---- |
\| `summarize_cluster.yml` | `schedule: weekly` | GraphQL 拉取近 N 篇 **done** 状态 Issue → `scripts/summarize.py` → 新建 `Weekly-Digest #编号` Issue | 聚合摘要（研究方向 / 方法/数据集 / 共性与趋势） |
\| `recommend_next.yml` | `schedule: daily` | 读取最近 20 条笔记 embedding → arXiv API (query=关键词) → 相似度 rerank → `scripts/recommend.py` → 起草 Issue 草稿 | `[Suggest]` 标题的待阅 Issue |
\| `sync_to_obsidian.yml` | `push` | 把所有 Issue 转 Markdown → `vault/✈️_inbox/` | 本地离线阅读 |

全部工作流均在 **UTC 时间** 执行，遵守 [https://docs.github.com/actions/using-workflows/about-workflows](https://docs.github.com/actions/using-workflows/about-workflows)。

---

## 🧠 高效利用 AI & GitHub Issues 指南

> 以下技巧与最佳实践无需立即实现任何新脚本，但能帮助你和 AI 在「读 → 记 → 总结 → 推荐」循环中合作更流畅。

### 1. 给 AI 下指令的 3‑步节奏

1. **定位** — 在 Issue body 顶部用 `<!-- context:xxx -->` 隐藏注释告诉 AI 本条笔记的角色（如 *survey* / *dataset* / *method*）。在多篇聚合任务中，AI 先按此字段分 bucket 再总结。
2. **收敛** — 评论里使用 "`@copilot summarize lines:1-30`"（或 Copilot Chat 快捷命令）让 AI 精简某段文字，只输出关键 bullet，避免全文浪费 token。
3. **触发** — 使用 Issue comment 指令 `gh workflow run summarize_cluster.yml -f issue=123` 临时重跑聚合，而非改 YAML；人保持 control loop，成本更低。

### 2. 标签驱动的自动化思维

* **Label = 状态机**：`status/todo → reading → done` 三态可直接作为 GitHub Projects 的看板列；AI 可监听 webhooks 在状态变化时执行动作（写摘要、打分等）。
* **Field = 过滤器**：在 `project.yml` 定义 Field `impact_score` / `method_type`，AI 可在同模板中读写 → 日后做多维交叉统计。

### 3. ChatOps + gh CLI 快捷键

| 操作                          | 命令示例                                           |                                                                            |
| --------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------- |
| 批量生成 Issue（从 DOI 列表）        | \`cat doi.txt                                  | xargs -I{} gh issue create -F scripts/new\_from\_doi.md -t "\[Paper] {}"\` |
| 让 AI 回答“这 3 篇有没有共用数据集？”     | `gh copilot question --context issue:10,11,12` |                                                                            |
| 直接在评论引用代码 commit 并让 AI 解释差异 | `gh copilot question --diff HEAD~1..HEAD`      |                                                                            |

### 4. 提示工程模板

```text
系统
你是我的科研助手，需用中文回答，并严格给出引用索引。

用户
下面是我最近读的 5 篇笔记，请总结共同方法论趋势，列出不足，并推荐 3 篇 arXiv 新论文：
<<<
{{issues_bodies}}
>>>
```

* **分隔符** `<<< >>>` 帮 AI 清晰识别 chunk；`{{issues_bodies}}` 通过脚本注入多篇 Issue 正文。
* 在 Prompt 末尾附 "**输出格式**"（如 markdown 表格 + 引脚注）能让结果直接贴入 README 或 Wiki。

### 5. 版本控制中的 AI 产物

* **存档原则**：AI 自动生成的摘要/推荐统一以 **comment** 形式附在原 Issue；当人确认无误后再复制到 Issue body 或 README，避免主文档反复 churn。
* **Diff 评审**：将 `summarize_cluster.yml` 产出的 Digest 以 Pull Request 更新 `docs/Weekly-Digest.md`，你可像审代码那样 Review & Comment。

### 6. 性能 & 成本小贴士

| 场景          | 建议                                           |
| ----------- | -------------------------------------------- |
| 大规模历史笔记向量化  | 先跑离线脚本，每 1,000 条批量写入；GitHub Actions 仅处理增量数据。 |
| 控制 GPT 调用费用 | 在脚本里做 **token 预算预估**，若正文 >3k 字先让 AI 自摘要再分类。  |
| 升级模型        | 用环境变量 `GPT_MODEL=...` 集中管理；脚本读取后统一切换。        |

> **记住**：让 AI 做「高层抽象与提炼」，把「机械 ETL、批量 API 调用」交给脚本或 Actions；人类核心工作是提出好问题与审核结果。

| Workflow                | 触发                 | 主要步骤                                                                                              | 输出                          |
| ----------------------- | ------------------ | ------------------------------------------------------------------------------------------------- | --------------------------- |
| `summarize_cluster.yml` | `schedule: weekly` | GraphQL 拉取近 N 篇 **done** 状态 Issue → `scripts/summarize.py` → 新建 `Weekly‑Digest #编号` Issue         | 聚合摘要（研究方向 / 方法/数据集 / 共性与趋势） |
| `recommend_next.yml`    | `schedule: daily`  | 读取最近 20 条笔记 embedding → arXiv API (query=关键词) → 相似度 rerank → `scripts/recommend.py` → 起草 Issue 草稿 | `[Suggest]` 标题的待阅 Issue     |
| `sync_to_obsidian.yml`  | `push`             | 把所有 Issue 转 Markdown → `vault/✈️_inbox/`                                                          | 本地离线阅读                      |

全部工作流均在 **UTC 时间** 执行，遵守 [https://docs.github.com/actions/using-workflows/about-workflows](https://docs.github.com/actions/using-workflows/about-workflows)。

---

## 🧑‍💻 本地开发

```bash
# 1. 克隆仓库
$ git clone git@github.com:<you>/paper-hub.git && cd paper-hub

# 2. 安装依赖
$ python -m venv .venv && source .venv/bin/activate
$ pip install -r requirements.txt  # openai, arxiv, httpx, pyyaml, numpy, scikit-learn

# 3. 配置环境变量 (放 .env 或 GH Secrets)
OPENAI_API_KEY=...
ARXIV_RSS_URL=https://export.arxiv.org/rss/cs.CL

# 4. 运行推荐脚本
$ python scripts/recommend.py --top_k 5
```

所有脚本遵循你的个人代码规范：

* **PEP 8 + 注释充分**（见 `scripts/utils.py` 示例）。
* **参数集中管理**：脚本入口只接受命名参数；内部禁用裸 `get()`。
* **Fail‑Fast**：任何异常立即抛出，Actions 会捕获并标红。

---

## 🤝 贡献指南

1. **创建新笔记**：在 Issues → `New Issue` 选择📑模板，按字段填写，状态默认 `status/todo`。
2. **阅读进行中**：把标签改为 `status/reading`，在评论区追加实时笔记或代码片段。
3. **阅读完成**：关闭 Issue 或打标签 `status/done`，自动被周报工作流抓取。
4. **脚本 / 工作流改动**：请走 Pull Request，遵循 Conventional Commits + 通过 `pre‑commit` 检查。

---

## ❓ FAQ

| 问题                | 答案                                                |
| ----------------- | ------------------------------------------------- |
| **为何不用 PDF？**     | arXiv / Blog 多为 HTML/LaTeX，可直接抓正文，无版权困扰。          |
| **embedding 放哪？** | 当前存进 Issue 隐藏注释，未来可迁移到 Supabase 向量表。              |
| **移动端如何看？**       | GitHub App 即可，或查看 Obsidian Vault 的 `✈️_inbox` 目录。 |
| **私密笔记？**         | 将仓库设为私有或拆分 `private-notes` 子模块。                   |

---

## 📝 License

本仓库除特别说明外，采用 **MIT License**。如引用他人博客内容，请保留原作者署名并遵守其许可协议。

---

> *What I cannot create, I do not understand.*
>
> —— 你的 AI 读研伙伴
