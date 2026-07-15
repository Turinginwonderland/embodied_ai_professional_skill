# New Phase 2：原文层向量检索落地计划

> 状态：待触发 | 创建：2026-07-15
> 触发条件见 `.claude/skills/input-knowledge-evolution/SKILL.md` §7
> 这份文档是"什么时候做、怎么做"的执行手册，条件未满足前不执行任何步骤。

---

## 0. 什么时候启动本计划

满足**任一**条件即启动：
1. `docs/raw_materials/` 原文素材累计超过 **~10 份**（PPT/PDF/网页快照/md 均算）
2. 单次研究任务需要 Claude 同时参考 **3 份以上**原文素材
3. 用户明确要求"回溯某条目出处的原文段落"，且该段落无法靠条目层的`来源`字段直接定位

条件未满足时，维持现状（条目层全量读入），不预装、不预建。

---

## 1. 目标

给 `docs/raw_materials/` 的**原文层**建向量检索，让 `input-knowledge-evolution` 在核对条目出处时能从大量原文中快速召回相关段落。

**明确边界（再次强调）**：向量检索只碰原文层，绝不碰条目层 `kb_*.md`。条目层永远全量读入 + schema 过滤。理由见 SKILL.md §7"关键原则"。

---

## 2. 技术选型

| 项 | 选择 | 理由 |
|----|------|------|
| 向量库 | **chromadb**（PersistentClient） | 用户历史项目用过，顺手；纯本地、无服务依赖 |
| 框架 | **不用 LangChain** | 抽象层太高、会切碎结构、难调试；自己写 <200 行更可控 |
| 切分策略 | 按文档类型分别处理（见 §4） | 不能用粗暴的"每 500 字切一段"——会切断研报/PPT 的逻辑结构 |
| 嵌入模型 | chromadb 默认 embedding function（起步） | 先跑通；若召回质量不够再换 bge-m3 等中文模型 |
| Python | 已有 3.14.6 | 无需另装 |

---

## 3. 目录结构（落地后）

```
具身智能项目/
├── docs/
│   ├── raw_materials/              # 原文层（向量库索引对象）
│   │   ├── 2025_具身智能_研报_A.pdf
│   │   └── 2026_供应链大会_纪要.md
│   └── outputs/
│       └── knowledge_base/
│           ├── kb_*.md              # 条目层（不进向量库）
│           └── vector_store/        # 向量库持久化（gitignore）
└── .claude/skills/input-knowledge-evolution/
    └── scripts/
        ├── search_raw.py            # 检索入口
        ├── build_index.py           # 建库/增量更新入口
        └── splitters.py             # 按类型的切分逻辑
```

---

## 4. 落地步骤（按顺序执行）

### 步骤 1：装依赖
```bash
pip install chromadb
```
仅此一个。不装 LangChain、不装 faiss（chromadb 自带）。

### 步骤 2：补 .gitignore
在项目根 `.gitignore` 追加：
```
# 向量库持久化（可重建，勿提交）
docs/outputs/knowledge_base/vector_store/
```

### 步骤 3：写切分器 `scripts/splitters.py`
**关键**：不能无脑按字数切，要按文档类型保留逻辑结构：
- **PDF/研报**：按"标题层级 + 段落"切，保留每段的标题路径作为 metadata
- **PPT**：按"幻灯片"切，每页一个 chunk，metadata 记页码
- **网页快照/md**：按 markdown 标题层级切
- 每个 chunk 的 metadata 必含：`source`（文件名）、`locator`（页码/章节路径/偏移量，用于回溯定位）

返回 `List[{id, text, metadata}]`。

### 步骤 4：写建库脚本 `scripts/build_index.py`
```python
import chromadb
from splitters import split_documents   # 上一步的切分器

client = chromadb.PersistentClient(path="docs/outputs/knowledge_base/vector_store")
collection = client.get_or_create_collection(
    name="raw_materials",
    # 起步用默认 embedding；质量不够再换 bge-m3
)

def add_file(path):
    chunks = split_documents(path)   # 返回 [{id, text, metadata}]
    collection.add(
        ids=[c["id"] for c in chunks],
        documents=[c["text"] for c in chunks],
        metadatas=[c["metadata"] for c in chunks],
    )

if __name__ == "__main__":
    import sys, glob
    files = sys.argv[1:] or glob.glob("docs/raw_materials/**/*", recursive=True)
    for f in files:
        add_file(f)
    print(f"indexed {len(files)} files")
```
支持增量：重复 add 同 id 会覆盖，不会重复入库。

### 步骤 5：写检索入口 `scripts/search_raw.py`
```python
import chromadb, sys, json

client = chromadb.PersistentClient(path="docs/outputs/knowledge_base/vector_store")
collection = client.get_collection("raw_materials")

def search(query, top_k=5):
    res = collection.query(query_texts=[query], n_results=top_k)
    hits = []
    for doc, meta, dist in zip(
        res["documents"][0], res["metadatas"][0], res["distances"][0]):
        hits.append({
            "text": doc,
            "source": meta.get("source"),
            "locator": meta.get("locator"),   # 页码/章节/偏移，回溯定位用
            "distance": dist,
        })
    return hits

if __name__ == "__main__":
    query = sys.argv[1]
    for h in search(query):
        print(json.dumps(h, ensure_ascii=False))
```

### 步骤 6：在 SKILL.md §7 把"尚未实现"改为"已实现"
把 §7 的"预留接口契约"段从"未实现"标记为可用，补上实际调用示例。

---

## 5. 与条目层的协作流（落地后怎么用）

当用户问"这条结论的原文到底是哪一段说的"时：
1. 从条目层读出该条目的`内容`和`来源`字段
2. 若`来源`字段已能定位（如"§一.2"），直接打开原文跳转，**不走向量库**
3. 若`来源`模糊（如只写"某研报"）或要找"还有哪些原文支持/反对这条"：
   `python scripts/search_raw.py "<条目内容关键词>"`
   → 拿到原文层最相关的 N 段 + locator → 回溯核实

**绝不**用向量召回的原文段直接写报告——报告只从条目层取数（synthesis-output-generation 的铁律）。向量库只是"回溯核实"工具，不是"取数"工具。

---

## 6. 召回质量不行怎么办（兜底）

起步用 chromadb 默认 embedding，若发现：
- 中文研报召回不准
- 切分太粗/太细导致语义断裂

按顺序升级：
1. 换中文嵌入模型：`embedding_function=SentenceTransformerEmbeddingFunction(model_name="BAAI/bge-m3")`
2. 调切分粒度（splitters.py 的阈值）
3. 对 query 加"查询扩展"（用 Claude 把一个问句扩成多个检索词）

---

## 7. 何时升级到阶段三（全自动流水线）

仅当出现以下**真实痛点**时再考虑，不要为自动化而自动化：
- 每周新增原文 >5 份，手工跑 `build_index.py` 成负担
- skill 摄取条目时，希望自动"扫一遍已有原文看有没有重复/冲突证据"

阶段三的形态：watch `docs/raw_materials/` → 新文件自动切片建库 → 触发 skill 提炼条目。到那时再说，现在不规划细节。

---

## 检查清单（启动前对照）

- [ ] `docs/raw_materials/` 是否已达 ~10 份？或单任务需 3+ 原文？
- [ ] `pip install chromadb` 是否已执行？
- [ ] `.gitignore` 是否已加 `docs/outputs/knowledge_base/vector_store/`？
- [ ] splitters.py 是否按类型保留 metadata 的 `source` + `locator`？
- [ ] 条目层 `kb_*.md` 是否确认**未被**纳入向量库？
