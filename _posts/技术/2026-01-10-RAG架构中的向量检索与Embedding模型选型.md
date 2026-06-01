---
layout: post
title: "RAG架构中的向量检索与Embedding模型选型"
category: 技术
tags: [RAG, 向量检索, Embedding, 语义搜索, AI Agent]
keywords: RAG, 向量检索, Embedding, 语义搜索, Milvus, Chroma, 模型选型
description: 从工程视角深入剖析RAG架构中的向量检索全流程，对比主流Embedding模型选型策略，提供生产环境的检索增强实战方案与性能调优指南。
---

## 一、核心概念

RAG（Retrieval-Augmented Generation）是当前 AI Agent 落地最成熟的技术范式之一，核心流程可拆解为三个阶段：

**索引阶段**：将文档切片（Chunk），通过 Embedding 模型将文本转为稠密向量，存入向量数据库。

**检索阶段**：用户查询同样经过 Embedding 转为向量，在向量库中用相似度搜索（ANN）召回 Top-K 相关文档切片。

**生成阶段**：将召回的切片拼入 Prompt，交由 LLM 生成最终回答。

三个阶段的瓶颈完全不同：索引阶段受限于 Embedding 模型的吞吐和向量库的写入性能；检索阶段的关键是召回的**准确率**（Precision）和**召回率**（Recall）；生成阶段则受上下文窗口和 Prompt 拼接策略影响。

Embedding 模型是整个链条的"翻译官"，它决定了文本与向量之间的语义映射质量。选错 Embedding 模型，即使向量库再快、LLM 再强，检索环节的输入也是噪声。

## 二、实战示例：基于 Chroma + BGE 的本地 RAG 系统

以下代码从零构建一个完整的本地 RAG 检索管道：

```python
from chromadb import PersistentClient
from chromadb.utils import embedding_functions
import numpy as np

# ---------- 1. 初始化向量库 ----------
client = PersistentClient(path="./chroma_db")

# 使用 BGE 中文 Embedding（通过 HuggingFace 本地推理）
bge_ef = embedding_functions.HuggingFaceEmbeddingFunction(
    model_name="BAAI/bge-large-zh-v1.5",
    api_key=None  # 本地模型不需要
)

collection = client.get_or_create_collection(
    name="tech_docs",
    embedding_function=bge_ef,
    metadata={"hnsw:space": "cosine"}  # 余弦相似度
)

# ---------- 2. 文档索引 ----------
documents = [
    "LangGraph 是 LangChain 生态中的有向图状态机框架，支持循环边和条件路由。",
    "BGE 模型由智源研究院发布，在中文语义检索任务上表现优异。",
    "向量数据库的选择需综合考虑召回精度、写入吞吐和集群扩展能力。",
    "Chroma 是一个轻量级向量数据库，适合原型开发和中小规模生产部署。"
]

chunks = [doc for doc in documents]
ids = [f"doc_{i}" for i in range(len(chunks))]

collection.add(
    documents=chunks,
    ids=ids,
    metadatas=[{"source": "blog"} for _ in chunks]
)

# ---------- 3. 检索 ----------
query = "什么是向量数据库？有哪些选择？"
results = collection.query(
    query_texts=[query],
    n_results=3,
    include=["documents", "distances"]
)

print(f"Query: {query}\n")
for i, (doc, dist) in enumerate(zip(results["documents"][0],
                                       results["distances"][0])):
    print(f"[Rank {i+1}] Distance: {dist:.4f}")
    print(f"  {doc}\n")
```

## 三、Embedding 模型选型指南

针对中文场景，以下是经过实际项目验证的选型建议：

| 模型 | 维度 | 中文表现 | 推理速度 | 适用场景 |
|------|------|----------|----------|----------|
| BAAI/bge-large-zh-v1.5 | 1024 | ★★★★★ | 中等 | 高精度检索，生产环境首选 |
| BAAI/bge-m3 | 1024 | ★★★★★ | 中等 | 多语言混合场景，8192 token 上下文 |
| text2vec-large-chinese | 1024 | ★★★★☆ | 快 | 对延迟敏感的在线服务 |
| shibing624/text2vec-base-chinese | 768 | ★★★☆☆ | 极快 | 资源受限环境，边缘部署 |

**选型决策树**：
1. 若需要多语言支持 → `bge-m3`（支持中英日韩等 100+ 语言）
2. 若纯中文、高精度 → `bge-large-zh-v1.5`
3. 若 GPU 资源有限或要求低延迟 → `text2vec-large-chinese`

## 四、最佳实践

### 4.1 Chunk 策略

切片大小是 RAG 系统中**调优收益最大的超参数**。过小（<100 tokens）丢失上下文，过大（>1000 tokens）稀释语义。推荐策略：

- **技术文档**：256-512 tokens，按段落边界切分，加 10% 重叠
- **法律/合同**：512-768 tokens，按条款切分，不要截断条款内部
- **FAQ/短文本**：保持原文长度，不要强行切片

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", ".", " ", ""]  # 按优先级切分
)
```

### 4.2 混合检索（Hybrid Search）

纯向量检索在精确匹配场景（如产品型号"PS-3000A"、订单号）中表现不佳。推荐混合检索：

- **稀疏检索**（BM25 / TF-IDF）：精确关键词匹配
- **稠密检索**（Embedding）：语义相似度
- **融合策略**：RRF（Reciprocal Rank Fusion），`score = Σ 1/(k+rank_i)`，k 通常取 60

### 4.3 重排序（Rerank）

初筛 Top-20 后，用更强的 Rerank 模型二次精排到 Top-5，精度提升显著而成本可控：

```python
from FlagEmbedding import FlagReranker

reranker = FlagReranker("BAAI/bge-reranker-v2-m3")
scores = reranker.compute_score([[query, doc] for doc in candidates])
```

### 4.4 向量库选型

| 向量库 | 规模 | 部署复杂度 | 特点 |
|--------|------|------------|------|
| Chroma | <10万 | 极低 | 嵌入式，零配置，快速原型 |
| Milvus | >百万 | 中等 | 分布式，十亿级向量，GPU 索引 |
| Qdrant | >10万 | 低 | Rust 实现，过滤查询强 |

---

## 面试题

### 题1：RAG 系统中，为什么 Chunk 大小会影响检索质量？应该如何选择？

**答案**：Chunk 大小直接影响两个维度的质量——语义完整性和检索精度。过小的 Chunk（如 64 tokens）语义碎片化，Embedding 向量无法准确表示完整观点，导致"大海捞针"——可能在向量空间中找到相关片段，但片段本身缺乏上下文，LLM 无法理解。过大的 Chunk（如 2000 tokens）则语义过于混杂，一个 Chunk 包含多个主题，Embedding 向量变成"平均特征"，检索到它时 LLM 被迫处理大量无关内容，且容易超出上下文窗口。

选择策略：(1) 根据文档类型——技术文档 256-512 tokens，法律条款 512-768 tokens；(2) 按自然边界切分——段落、标题层级，不要截断句子；(3) 加 10%-20% 重叠保证边界信息不丢失；(4) 通过 A/B 测试确定最佳值，评估指标用 Recall@K 和 Answer Correctness。

### 题2：向量检索 + BM25 的混合检索为什么比纯向量检索更好？RRF 融合算法如何工作？

**答案**：纯向量检索的致命弱点是无法处理精确匹配。例如产品型号"iPhone 15 Pro Max 256GB 蓝色"，向量模型将其编码为语义向量后，相近的表述（如"苹果手机大容量版"）也可能被召回，但精确搜索"256GB"时，BM25 可以直接命中包含该关键词的文档。两个检索方式互补：向量检索捕获语义相似度，BM25 捕获关键词精确度。

RRF（Reciprocal Rank Fusion）是一种无监督融合算法，不依赖分数分布假设。公式为 `RRF_score(d) = Σ 1/(k + rank_i(d))`，其中 k 是平滑常数（通常为 60），`rank_i` 是文档在第 i 个排序列表中的排名。RRF 只关心相对排名而非绝对分数，天然适合融合不同量纲的检索结果。例如某文档在向量检索中排第 3、BM25 中排第 8，RRF_score = 1/(60+3) + 1/(60+8) ≈ 0.0159 + 0.0147 = 0.0306，综合排序优于仅在单个列表中靠前的文档。