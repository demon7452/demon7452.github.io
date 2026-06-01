---
layout: post
title: "RAG进阶：Chunk策略与Rerank优化"
category: 技术
tags: [RAG, Chunk策略, Rerank, 语义分块, 召回优化]
keywords: RAG, Chunk策略, 语义分块, Rerank, 召回优化, Cross-Encoder, BGE-Reranker
description: 在上一期RAG基础之上，深入探讨Chunk切分策略的工程选择与Rerank二次精排的落地实现，覆盖语义分块、Late Chunking、BM25+向量混合召回、Cross-Encoder重排序等生产级优化技术。
---

## 一、核心概念

上一期我们覆盖了 RAG 的基础管道——向量检索与 Embedding 选型。本期聚焦两个决定 RAG 系统"上限"的环节：**Chunk 策略**（决定了检索的候选质量上限）和 **Rerank 优化**（决定了最终送入 LLM 的排序质量）。

简单类比：Chunk 策略是"怎么把书拆成卡片"，Rerank 是"从一堆卡片中挑出最相关的几张"。前者是信息架构问题，后者是精细化匹配问题。

## 二、Chunk 策略深度对比

### 2.1 固定大小分块（Fixed-size）

最基础的策略，按 token 数量截断：

```python
def fixed_size_chunk(text: str, chunk_size: int = 512, overlap: int = 50):
    tokens = text.split()  # 简化，实际用 tokenizer
    chunks = []
    start = 0
    while start < len(tokens):
        end = min(start + chunk_size, len(tokens))
        chunks.append(" ".join(tokens[start:end]))
        start = end - overlap
    return chunks
```

**优点**：实现简单、可预测。**缺点**：粗暴截断可能将一句话拆到两个 Chunk，破坏语义完整性。

### 2.2 递归字符分块（Recursive Character）

LangChain 的 `RecursiveCharacterTextSplitter` 按优先级逐级尝试分隔符：

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", ".", " ", ""],
    length_function=len
)

chunks = splitter.split_text(document)
```

核心思想：优先在段落边界（`\n\n`）切分；段落太长则在换行处切；再不行才在句子边界切。这是**工程实践中使用率最高的策略**。

### 2.3 语义分块（Semantic Chunking）

根据文本的语义相似度变化来决定切分点——当相邻句子之间的 Embedding 余弦相似度骤降时，说明主题切换，应在该处分块：

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("BAAI/bge-large-zh-v1.5")

def semantic_chunk(text: str, threshold_percentile: float = 90):
    sentences = [s.strip() for s in text.replace("\n", " ").split("。") if s.strip()]

    if len(sentences) <= 1:
        return [text]

    # 计算相邻句子的 Embedding 相似度
    embeddings = model.encode(sentences)
    similarities = []
    for i in range(len(embeddings) - 1):
        sim = np.dot(embeddings[i], embeddings[i+1]) / (
            np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[i+1])
        )
        similarities.append(sim)

    # 找到相似度显著下降的断裂点
    threshold = np.percentile(similarities, 100 - threshold_percentile)
    breakpoints = [i + 1 for i, sim in enumerate(similarities) if sim < threshold]

    # 按断裂点重组 Chunk
    chunks = []
    start = 0
    for bp in breakpoints + [len(sentences)]:
        chunk = "。".join(sentences[start:bp]) + "。"
        chunks.append(chunk)
        start = bp

    return chunks
```

**优点**：分块边界更自然，每个 Chunk 的语义内聚性高。**缺点**：计算量大（需对所有句子做 Embedding），不适合实时索引。

### 2.4 Late Chunking（2025年新范式）

传统分块是"先切再编码"（先读入文本→分块→每个块分别 Embedding）。Late Chunking 反其道而行——**先对全文做一次完整编码，再在向量层面对切分后的片段进行池化**。这样每个 Chunk 的向量中仍然保留了被切掉的上下文信息。

这是 Jina AI 在 2024 年底提出的技术，核心价值在于既不增加 Chunk 大小（保持检索精度），又减少边界信息丢失。

### 策略选择决策树

| 文档类型 | 推荐策略 | 典型 chunk_size |
|---------|---------|----------------|
| 技术文档 / 教程 | 递归字符 | 300-500 tokens |
| 法律合同 | 语义分块 | 500-800 tokens |
| FAQ / 短问答 | 固定大小 | 100-200 tokens |
| 多主题长文 | 语义分块 | 400-600 tokens |
| 追求极致质量 | Late Chunking | 256-512 tokens |

## 三、Rerank 优化实战

### 3.1 为什么需要 Rerank？

向量检索（Bi-Encoder）速度快但精度有损——查询和文档分别独立编码，缺乏交叉注意力。Rerank 用 Cross-Encoder 对候选集做二次精排，将 query 和 document 拼接后联合编码。

典型流程：向量检索返回 Top-30 → Rerank 精排到 Top-5 → 送入 LLM。

### 3.2 BGE-Reranker 实战

```python
from FlagEmbedding import FlagReranker

# 初始化 Reranker（Cross-Encoder 模型）
reranker = FlagReranker(
    "BAAI/bge-reranker-v2-m3",
    use_fp16=True  # 半精度加速
)

# 向量检索粗筛结果
query = "如何在 Python 中实现异步 HTTP 请求？"
candidates = [
    "使用 aiohttp 库可以方便地实现异步 HTTP 请求...",
    "Python 的 requests 库是同步的 HTTP 客户端...",
    "异步编程在 Python 中通过 asyncio 模块实现...",
    "HTTP 协议是应用层协议，基于 TCP/IP...",
    "使用 httpx 库的 AsyncClient 可以实现异步请求..."
]

# 构造 query-doc pair
pairs = [[query, doc] for doc in candidates]

# Rerank 打分
scores = reranker.compute_score(pairs, normalize=True)

# 按分数降序重排
ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)

print("=== Rerank 结果 ===")
for i, (doc, score) in enumerate(ranked):
    print(f"[{i+1}] Score={score:.4f} | {doc[:50]}...")
```

### 3.3 Cohere Rerank API（SaaS 方案）

对于不想自托管模型的团队：

```python
import cohere

co = cohere.Client("your-api-key")

results = co.rerank(
    query="AI Agent 开发框架对比",
    documents=candidates,
    top_n=5,
    model="rerank-v3.5"
)

for r in results.results:
    print(f"[{r.index}] relevance={r.relevance_score:.4f}")
```

### 3.4 Rerank 选型对比

| 模型 | 架构 | 精度 | 延迟 | 部署难度 |
|------|------|------|------|----------|
| bge-reranker-v2-m3 | Cross-Encoder | ★★★★★ | ~50ms | 低（本地 GPU/CPU） |
| bge-reranker-large | Cross-Encoder | ★★★★☆ | ~30ms | 低 |
| Cohere rerank-v3.5 | 闭源 API | ★★★★★ | ~100ms | 零（API 调用） |
| Jina Reranker v2 | Cross-Encoder | ★★★★☆ | ~40ms | 低 |

## 四、最佳实践

1. **检索量设置**：向量检索 Top-20~50，Rerank 后取 Top-3~5。检索太少会漏掉相关文档，太多则 Rerank 耗时线性增长。

2. **Rerank 与 BM25 的配合**：若 BM25 和向量检索的融合结果已足够好（50% 的 case 首位命中），Rerank 可只用于尾部分布的长尾 Query。

3. **缓存 Rerank 分数**：相同 query 的 Rerank 结果应缓存（TTL 按业务设置），尤其在 FAQ 场景下命中率极高。

4. **Late Chunking 的前置依赖**：需要长上下文 Embedding 模型（如 jina-embeddings-v3，支持 8192 tokens），确保模型能对整个长文档做完整编码。

---

## 面试题

### 题1：语义分块和递归字符分块的核心区别是什么？在什么场景下语义分块的优势会被削弱？

**答案**：核心区别在于切分决策的依据——递归字符分块依赖文档的**显式结构**（段落、换行、标点），本质是"尊重作者的排版意图"；语义分块依赖**Embedding 相似度变化**，本质是"理解内容的主题边界"。前者是规则驱动，后者是模型驱动。

语义分块的优势在以下场景会被削弱：(1) 文档本身是高度结构化的（如 API 文档、数据库 Schema），标题和段落边界已经完美表达了主题切换，递归字符分块就足够；(2) 短文档（<1000 tokens）场景，分块数量很少，语义分块的计算开销不值得；(3) 主题变化不明显的单一主题长文，语义相似度全程高且平稳，找不到有效断裂点；(4) 计算资源受限的实时索引场景，语义分块需要对全文逐句做 Embedding，成本较高。

### 题2：Cross-Encoder（Rerank）相比 Bi-Encoder（向量检索）为什么更准确？代价是什么？

**答案**：Bi-Encoder（如 BGE）在编码阶段对 Query 和 Document **独立编码**，然后通过向量点积/余弦相似度做匹配。这个架构的好处是 Document 向量可预计算和索引，检索时只需编码 Query 即可 ANN 搜索；缺点是 Query 和 Document 之间没有交互——模型看不到"Query 中的关键词在 Document 中是如何被使用的"。

Cross-Encoder 将 Query 和 Document **拼接为一段文本** `[CLS] query [SEP] document [SEP]`，通过 Transformer 的全自注意力机制让 Query 和 Document 的 token 之间充分交互，然后输出一个相关性分数。这种联合编码捕获了细粒度的语义匹配（如同义词消歧、否定句式识别、条件关系判断），因此精度显著更高。

代价：(1) 无法预计算和索引——每个 Query-Document pair 都需要完整的前向传播，耗时与候选集大小成正比；(2) 延迟大——对 30 个候选做 Rerank 需要 30 次 Cross-Encoder 推理（批量模式下可优化但仍有开销）。因此不能直接对全库做 Rerank，只能用于向量检索之后的二次精排。