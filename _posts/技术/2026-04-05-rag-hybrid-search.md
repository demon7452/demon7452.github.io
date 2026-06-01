---
layout: post
title: "RAG混合检索：向量+关键词+结构化查询实战"
date: 2026-04-05
categories: [AI Agent, RAG, 应用开发]
tags: [RAG, 混合检索, 向量数据库, Elasticsearch, Python]
---

在构建 AI Agent 的知识问答能力时，RAG（Retrieval-Augmented Generation）已经成为事实上的标配架构。然而，单一的向量检索在很多实际场景中会暴露出明显短板——纯语义相似度匹配无法处理精确关键词匹配、时间范围过滤、结构化字段筛选等需求。混合检索（Hybrid Search）正是解决这些问题的关键方案。

## 为什么需要混合检索？

向量检索擅长理解语义，即使查询词与文档用词不完全一致，也能找到相关内容。但以下场景纯向量检索会失效：

- **精确匹配**：搜索包含「RFC 7231」的文档，向量模型可能把它和「RFC 7230」混为一谈。
- **结构化过滤**：只查2026年Q1的报告、只查某个部门的文档。
- **关键词命中**：法律条款编号「第39条第2款」、API名称「user.logout」等精确标识符。
- **实体名称**：人名、地名、产品型号的精确匹配。

混合检索的核心思路很简单：**同时执行向量检索和关键词/结构化检索，然后将两路结果融合排序**。

## 混合检索的三种武器

### 1. 向量检索（Dense Retrieval）

使用 embedding 模型将查询和文档都映射到高维向量空间，通过余弦相似度或欧氏距离计算相关性。

### 2. 关键词检索（Sparse Retrieval）

基于倒排索引的 BM25 算法，精确匹配词项，对罕见词、专有名词效果极佳。

### 3. 结构化查询（Structured Filtering）

利用元数据字段（日期、分类、标签、作者等）进行精确过滤，这是传统数据库的强项。

## 实现方案：Elasticsearch + 向量数据库

### 方案一：Elasticsearch 8.x 单引擎方案

ES 8.x 原生支持 dense_vector 类型和 kNN 搜索，可以在一个查询中同时完成关键词检索、向量检索和结构化过滤。

```python
from elasticsearch import Elasticsearch
import numpy as np

es = Elasticsearch("http://localhost:9200")

# 定义索引映射：同时支持全文搜索、向量搜索和结构化字段
index_mapping = {
    "mappings": {
        "properties": {
            "title": {"type": "text", "analyzer": "ik_max_word"},
            "content": {"type": "text", "analyzer": "ik_max_word"},
            "content_vector": {
                "type": "dense_vector",
                "dims": 768,          # 向量维度，取决于 embedding 模型
                "index": True,
                "similarity": "cosine"
            },
            "publish_date": {"type": "date"},
            "category": {"type": "keyword"},
            "tags": {"type": "keyword"},
            "author": {"type": "keyword"}
        }
    }
}

es.indices.create(index="ai_docs", body=index_mapping)
```

### 混合检索查询

以下查询同时执行：向量搜索（找出语义相似文档）+ BM25 全文搜索（匹配「微服务」关键词）+ 结构化过滤（仅2026年的文档 + 后端分类）：

```python
def hybrid_search(query_text: str, query_vector: list[float],
                  category: str = None, date_from: str = None):
    """Execute hybrid search combining vector, BM25 and structured filtering."""

    must_clauses = []

    # 结构化过滤：分类
    if category:
        must_clauses.append({"term": {"category": category}})

    # 结构化过滤：日期范围
    if date_from:
        must_clauses.append({
            "range": {"publish_date": {"gte": date_from}}
        })

    search_body = {
        "query": {
            "bool": {
                "must": must_clauses,
                "should": [
                    # 向量搜索分支
                    {
                        "knn": {
                            "field": "content_vector",
                            "query_vector": query_vector,
                            "k": 20,
                            "num_candidates": 100
                        }
                    },
                    # BM25 关键词搜索分支
                    {
                        "multi_match": {
                            "query": query_text,
                            "fields": ["title^3", "content"],
                            "type": "best_fields"
                        }
                    }
                ]
            }
        },
        "size": 10
    }

    return es.search(index="ai_docs", body=search_body)
```

### 方案二：Milvus + Elasticsearch 双引擎方案

对于追求极致向量检索性能的场景，可以用 Milvus 专攻向量检索，ES 负责关键词和元数据过滤。

```python
from pymilvus import Collection, connections
from elasticsearch import Elasticsearch

class HybridSearchEngine:
    """Dual-engine hybrid search: Milvus for vector + ES for keyword/metadata."""

    def __init__(self, milvus_host="localhost", milvus_port="19530",
                 es_host="http://localhost:9200"):
        connections.connect(host=milvus_host, port=milvus_port)
        self.collection = Collection("ai_docs")
        self.es = Elasticsearch(es_host)
        self.rrf_k = 60  # RRF 融合参数

    def search(self, query_text: str, query_vector: list[float],
               top_k: int = 10) -> list[dict]:
        # 1. Milvus 向量检索
        self.collection.load()
        vector_results = self.collection.search(
            data=[query_vector],
            anns_field="content_vector",
            param={"metric_type": "COSINE", "params": {"nprobe": 16}},
            limit=top_k * 2,
            output_fields=["doc_id", "title"]
        )

        # 2. ES BM25 检索
        es_results = self.es.search(index="ai_docs", body={
            "query": {"multi_match": {"query": query_text,
                                       "fields": ["title^3", "content"]}},
            "size": top_k * 2
        })

        # 3. RRF (Reciprocal Rank Fusion) 融合
        return self._rrf_merge(vector_results, es_results, top_k)

    def _rrf_merge(self, vec_results, es_results, top_k: int) -> list[dict]:
        """Reciprocal Rank Fusion for result merging."""
        scores = {}
        doc_map = {}

        # 向量检索结果
        for hits in vec_results:
            for rank, hit in enumerate(hits):
                doc_id = hit.entity.get("doc_id")
                scores[doc_id] = scores.get(doc_id, 0) + 1 / (self.rrf_k + rank + 1)
                doc_map[doc_id] = hit.entity

        # ES 检索结果
        for rank, hit in enumerate(es_results["hits"]["hits"]):
            doc_id = hit["_source"]["doc_id"]
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (self.rrf_k + rank + 1)
            if doc_id not in doc_map:
                doc_map[doc_id] = hit["_source"]

        # 按融合分数排序
        sorted_docs = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [{"doc_id": did, "score": sc, "data": doc_map[did]}
                for did, sc in sorted_docs[:top_k]]
```

## 融合排序策略对比

| 策略 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| RRF | 基于排名的倒数求和 | 无需校准分数，简单鲁棒 | 丢弃了原始分数信息 |
| 加权求和 | 对各路分数归一化后加权 | 可调节权重偏好 | 需要调参，分数尺度需对齐 |
| 学习排序 | 训练模型预测最终相关性 | 效果最优 | 需要标注数据，工程复杂 |

## 实际落地方案建议

对于大多数 AI Agent 应用场景，推荐以下分层策略：

1. **原型阶段**：直接用 Elasticsearch 8.x 单引擎，降低架构复杂度。
2. **规模化阶段**：拆分向量引擎（Milvus/Qdrant），ES 退化为关键词+元数据引擎，用 RRF 做结果融合。
3. **终极优化**：引入 reranker 模型（如 Cohere Rerank、BGE-Reranker）对融合结果做二次精排。

混合检索不是炫技，而是让 RAG 系统真正可靠的必要工程手段。当你的 Agent 需要从海量文档中精准找到答案时，单靠向量检索是不够的——就像你不能只用一把锤子来修理所有东西。

## 面试题

**1. 混合检索中 RRF（Reciprocal Rank Fusion）的工作原理是什么？为什么它在多路召回融合中比简单的分数加权更鲁棒？**

<details>
<summary>点击展开答案</summary>

RRF 的计算公式为：`score(d) = Σ 1/(k + rank_i(d))`，其中 `k` 是常数（通常取 60），`rank_i(d)` 是文档 d 在第 i 路检索结果中的排名。

RRF 更鲁棒的原因：
- **无需分数归一化**：不同检索引擎的分数分布差异很大（向量余弦相似度 0~1，BM25 分数可以超过 100），直接加权需要复杂的校准。RRF 只关心相对排名，天然跨引擎可比。
- **对异常不敏感**：如果某一路检索给某个文档打了极高分数（可能是噪音），在加权求和中会主导最终排序；RRF 只取排名，削弱了极端分数的影响。
- **超参数少**：只需调一个 k 值，比多路加权求和的调参工作量小得多。

</details>

**2. 在 RAG 系统中，什么时候应该优先使用混合检索而非纯向量检索？请给出三个典型场景。**

<details>
<summary>点击展开答案</summary>

三个典型场景：

1. **精确标识符查询**：用户搜索「RFC 7231 第5.3节」或「API: user.deactivate」，向量模型可能无法区分相邻编号，BM25 的精确词项匹配更可靠。

2. **多条件结构化过滤**：查询「2026年Q1 后端组的故障复盘报告」，需要在语义相关性的基础上叠加日期范围、部门标签的结构化约束。纯向量检索无法处理这种「语义 + 元数据」的组合查询。

3. **领域专有名词密集的文档库**：如法律文书（条款编号）、医疗文献（药物名称、ICD编码）、技术文档（类名、函数签名）。这些术语的语义空间分布可能不够分散，关键词检索能提供更好的区分度。

</details>