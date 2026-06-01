---
layout: post
title: "RAG架构中的向量检索与Embedding模型选型"
date: 2026-01-10 10:00:00 +0800
category: 技术
tags: [RAG, Embedding, 向量检索, 语义搜索, AI Agent]
keywords: RAG架构, Embedding模型选型, 向量检索, 语义搜索, 知识库构建
description: 系统讲解RAG架构中向量检索的核心原理，对比主流Embedding模型的选型策略，并给出工程落地的最佳实践。
---

## 核心概念

RAG（Retrieval-Augmented Generation，检索增强生成）是当前 AI Agent 应用中最主流的架构模式。其核心理念是：在 LLM 生成回答之前，先从外部知识库中检索相关信息，将检索结果注入 Prompt 作为上下文，从而让 LLM 输出更准确、更具时效性的回答。

### 向量检索的工作原理

向量检索是整个 RAG 管道的起点，其流程如下：

1. **文档切分（Chunking）**：将长文档切成适当大小的文本块（chunk）。
2. **向量化（Embedding）**：用 Embedding 模型将每个 chunk 转化为固定维度的向量。
3. **索引构建（Indexing）**：将所有向量存入向量数据库（如 Milvus、Qdrant、Chroma）。
4. **查询检索（Retrieval）**：用户查询同样经过 Embedding 模型向量化，然后在向量库中进行相似度搜索（通常用余弦相似度或欧氏距离），召回 Top-K 个最相关的 chunk。

### Embedding 模型的核心指标

选 Embedding 模型不能只看排行榜分数，要从以下几个工程维度综合评估：

| 维度 | 说明 | 关键问题 |
|------|------|----------|
| **检索精度** | 在目标领域数据上的 NDCG/Recall 指标 | 通用模型在垂直领域可能严重退化 |
| **向量维度** | 输出向量的维度数 | 维度越高存储成本越大，检索速度越慢 |
| **最大 Token 数** | 单次可编码的最大文本长度 | 超过限制需截断，信息丢失 |
| **推理速度** | 每秒可处理的文本量 | 大批量入库时直接影响构建效率 |
| **多语言支持** | 对中文/混合语言的处理能力 | 中文场景必须关注 C-MTEB 榜单 |
| **部署方式** | 本地部署 vs API 调用 | 数据安全的权衡 |

## 实战示例

下面通过一个完整的 RAG 检索管道，对比不同 Embedding 模型在中英文混合场景下的表现：

```python
import numpy as np
from typing import List, Tuple
from sentence_transformers import SentenceTransformer
import time

# ============ 1. 文档准备与切分 ============
documents = [
    "LangGraph is a framework for building stateful, multi-actor applications with LLMs.",
    "LangGraph 是基于 LangChain 的状态机 Agent 框架，支持条件边和循环执行。",
    "RAG architecture combines retrieval with generation to produce grounded responses.",
    "RAG 架构通过向量检索从知识库中获取相关文档，再交给大模型生成回答。",
    "MCP (Model Context Protocol) standardizes how applications provide context to LLMs.",
    "MCP 协议定义了 Tool、Resource、Prompt 三种能力，是 AI Agent 的标准化接口。",
    "The Transformer architecture relies entirely on self-attention mechanisms.",
    "向量数据库如 Milvus 和 Qdrant 为语义搜索提供了高效的近似最近邻检索能力。",
]

queries = [
    "LangGraph 是什么框架？",
    "Explain RAG architecture briefly.",
    "MCP 协议有哪些核心能力？",
    "向量数据库在 RAG 中的作用是什么？",
]

# ============ 2. 模型加载与对比 ============
models_to_test = [
    ("all-MiniLM-L6-v2", "sentence-transformers/all-MiniLM-L6-v2"),
    ("bge-large-zh-v1.5", "BAAI/bge-large-zh-v1.5"),
    ("multilingual-e5-large", "intfloat/multilingual-e5-large"),
]

# 使用 query 前缀（bge/e5 系列需要）
query_instruction = {
    "bge-large-zh-v1.5": "为这个句子生成表示以用于检索相关文章：",
    "multilingual-e5-large": "query: ",
}

def encode_with_model(
    model_name: str, model_path: str, texts: List[str], is_query: bool = False
) -> np.ndarray:
    """加载模型并编码文本，返回归一化向量"""
    model = SentenceTransformer(model_path)
    prefix = ""
    if is_query and model_name in query_instruction:
        prefix = query_instruction[model_name]
        texts = [prefix + t for t in texts]

    embeddings = model.encode(texts, normalize_embeddings=True, show_progress_bar=False)
    return embeddings, model

def evaluate_retrieval(
    doc_embeddings: np.ndarray,
    query_embeddings: np.ndarray,
    k: int = 3
) -> List[List[Tuple[int, float]]]:
    """余弦相似度检索，返回每个 query 的 top-k 结果"""
    results = []
    for q_emb in query_embeddings:
        # 余弦相似度（已归一化，点积等价于余弦相似度）
        scores = np.dot(doc_embeddings, q_emb)
        top_k_idx = np.argsort(scores)[::-1][:k]
        top_k_scores = [(int(idx), float(scores[idx])) for idx in top_k_idx]
        results.append(top_k_scores)
    return results

# ============ 3. 执行对比评估 ============
for model_name, model_path in models_to_test:
    print(f"\n{'='*60}")
    print(f"模型: {model_name}")
    print(f"{'='*60}")

    start = time.time()
    doc_embeddings, model = encode_with_model(model_name, model_path, documents)
    query_embeddings, _ = encode_with_model(model_name, model_path, queries, is_query=True)
    elapsed = time.time() - start

    print(f"向量维度: {doc_embeddings.shape[1]}")
    print(f"编码耗时: {elapsed:.2f}s")

    results = evaluate_retrieval(doc_embeddings, query_embeddings, k=3)

    for i, (query, top_k) in enumerate(zip(queries, results)):
        print(f"\n  Query {i+1}: {query}")
        for rank, (doc_idx, score) in enumerate(top_k, 1):
            print(f"    #{rank} (score={score:.4f}): {documents[doc_idx][:60]}...")

# ============ 4. 向量库批量入库示例 ============
import chromadb

def build_vector_index(
    documents: List[str],
    model: SentenceTransformer,
    collection_name: str = "knowledge_base"
):
    """使用 ChromaDB 构建向量索引"""
    client = chromadb.Client()
    # 如果已存在则删除重建
    try:
        client.delete_collection(collection_name)
    except Exception:
        pass

    collection = client.create_collection(
        name=collection_name,
        metadata={"hnsw:space": "cosine"}
    )

    embeddings = model.encode(documents, normalize_embeddings=True)

    # 批量添加
    collection.add(
        embeddings=embeddings.tolist(),
        documents=documents,
        ids=[f"doc_{i}" for i in range(len(documents))]
    )

    return collection

# 构建索引
model_for_index = SentenceTransformer("BAAI/bge-large-zh-v1.5")
collection = build_vector_index(documents, model_for_index)

# 查询
query_text = "LangGraph 和状态机"
query_embedding = model_for_index.encode(
    ["为这个句子生成表示以用于检索相关文章：" + query_text],
    normalize_embeddings=True
)

search_results = collection.query(
    query_embeddings=query_embedding.tolist(),
    n_results=3
)

print("\n============ ChromaDB 检索结果 ============")
for i, (doc, dist) in enumerate(zip(
    search_results["documents"][0],
    search_results["distances"][0]
)):
    print(f"#{i+1} (距离={dist:.4f}): {doc[:80]}...")
```

## 最佳实践

1. **中文优先选 BGE 系列**：`bge-large-zh-v1.5` 在中文检索任务上表现优异，且支持 `query_instruction` 前缀提升检索精度。
2. **多语言混合选 multilingual-e5**：如果知识库同时包含中英文，`multilingual-e5-large` 是不错的选择，跨语言检索效果稳定。
3. **成本敏感选 all-MiniLM-L6-v2**：384 维向量、轻量级模型，适合原型验证和小规模场景，但中文效果一般。
4. **Query 前缀不能忘**：BGE 和 E5 系列都需要在 query 侧加前缀（`为这个句子生成表示以用于检索相关文章：`），但 document 侧通常不需要或使用不同的前缀。混用会导致检索精度骤降。
5. **向量维度与存储的权衡**：1024 维向量检索精度更高，但存储和计算开销是 384 维的近 3 倍。应根据实际精度需求和硬件资源选择。
6. **预热与批处理**：大批量入库时使用 `model.encode(batch, batch_size=64)` 并开启 `show_progress_bar=True`，利用 GPU 批处理能力提升吞吐量。

## 面试题

**Q1：RAG 系统中 Embedding 模型的 query 侧和 document 侧为什么要使用不同的前缀？如果混用会有什么后果？**

**A1**：BGE 和 E5 系列 Embedding 模型在训练时使用了非对称的指令微调策略——训练数据中 query 侧和 document 侧的文本角色不同，因此模型学会了根据前缀区分编码模式。query 前缀告诉模型"这是搜索查询，请将它与文档区分开"，从而在向量空间中形成更合理的 query-document 对齐关系。

如果混用——比如 query 不加前缀而 document 加了前缀，模型的编码行为会偏离训练分布，导致 query 向量和 document 向量的语义对齐关系被破坏，检索精度可能下降 10%-30%。这就是为什么生产环境中必须在代码里显式管理 query 和 document 的编码逻辑。

---

**Q2：当知识库包含大量长文档时，如何设计 Chunk 策略来平衡检索精度和完整性？**

**A2**：长文档 Chunk 策略涉及三个核心决策：

1. **Chunk Size**：一般设为 512-1024 tokens。太小（如 128）会导致语义碎片化，检索结果缺乏上下文；太大（如 4096）会稀释关键信息，降低检索精度。推荐从 512 起步，根据文档类型 A/B 测试调优。

2. **Overlap**：chunk 之间保留 10%-20% 的重叠（如 512 tokens chunk + 64 tokens overlap）。这确保跨 chunk 边界的关键信息不被切断，提高召回完整性。

3. **层级索引策略**：对于超长文档（如 100 页报告），使用两级索引——先对小粒度 chunk（如 256 tokens）做向量检索召回，再对召回 chunk 所属的父文档或更大粒度 chunk 做上下文扩展。这种"小检索 + 大上下文"模式在 LangChain 的 Parent Document Retriever 中有标准实现。

核心原则：检索时用小块追求精度精准命中，生成时用大块保证上下文完整。
