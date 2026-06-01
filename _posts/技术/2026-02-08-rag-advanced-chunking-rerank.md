---
layout: post
title: "RAG进阶：Chunk策略与Rerank优化"
category: 技术
tags: [RAG, Chunking, Rerank, 检索增强生成, Embedding优化]
keywords: [RAG, Chunking Strategy, Rerank, Cross-Encoder, Semantic Search]
description: "深入探讨RAG系统中Chunk分块策略和Rerank重排序技术，从语义分块到混合检索重排序，全面提升RAG系统检索质量。"
date: 2026-02-08
---

# RAG进阶：Chunk策略与Rerank优化

## 引言

在上一篇RAG文章中我们讲解了向量检索的基础架构，但实际生产环境中，**Chunk策略**和**Rerank优化**往往是决定RAG系统质量的关键因素。不合理的分块会导致语义断裂，缺少Rerank则会让不相关的文档混入LLM上下文。

本文将深入讲解这两种进阶技术，通过实战代码展示如何将RAG检索质量提升到生产级别。

## 一、Chunk策略深度剖析

### 1.1 常见分块策略对比

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    MarkdownHeaderTextSplitter,
    TokenTextSplitter,
    SemanticChunker
)

class ChunkingStrategyComparison:
    """分块策略对比"""
    
    @staticmethod
    def fixed_size_chunk(text: str, chunk_size: int = 500) -> list:
        """策略1：固定大小分块（最简单）"""
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=100,
            separators=["\n\n", "\n", "。", "！", "？", " ", ""]
        )
        return splitter.split_text(text)
    
    @staticmethod
    def semantic_chunk(text: str) -> list:
        """策略2：语义分块（推荐）"""
        from langchain_openai import OpenAIEmbeddings
        
        splitter = SemanticChunker(
            embeddings=OpenAIEmbeddings(),
            breakpoint_threshold_type="percentile",
            breakpoint_threshold_amount=95
        )
        return splitter.split_text(text)
    
    @staticmethod
    def token_based_chunk(text: str, max_tokens: int = 256) -> list:
        """策略3：Token级别分块"""
        import tiktoken
        
        encoding = tiktoken.get_encoding("cl100k_base")
        tokens = encoding.encode(text)
        
        chunks = []
        for i in range(0, len(tokens), max_tokens):
            chunk_tokens = tokens[i:i + max_tokens]
            chunks.append(encoding.decode(chunk_tokens))
        
        return chunks
    
    @staticmethod
    def markdown_aware_chunk(text: str) -> list:
        """策略4：Markdown结构感知分块"""
        headers_to_split_on = [
            ("#", "Header 1"),
            ("##", "Header 2"),
            ("###", "Header 3"),
        ]
        
        splitter = MarkdownHeaderTextSplitter(
            headers_to_split_on=headers_to_split_on
        )
        return splitter.split_text(text)
```

### 1.2 实战：自适应分块策略

```python
class AdaptiveChunker:
    """自适应分块器——根据文档类型选择最优策略"""
    
    def __init__(self):
        self.strategies = {
            "markdown": self._markdown_chunk,
            "pdf": self._semantic_chunk,
            "code": self._ast_chunk,
            "default": self._fixed_chunk
        }
        self.min_chunk_size = 100
        self.max_chunk_size = 2000
    
    def chunk(self, text: str, doc_type: str = "default") -> list:
        """根据文档类型分块"""
        strategy = self.strategies.get(doc_type, self.strategies["default"])
        raw_chunks = strategy(text)
        
        # 后处理：合并过小块、拆分过大块
        return self._post_process(raw_chunks)
    
    def _semantic_chunk(self, text: str) -> list:
        """语义分块（PDF/文章专用）"""
        from langchain_openai import OpenAIEmbeddings
        
        splitter = SemanticChunker(
            embeddings=OpenAIEmbeddings(),
            breakpoint_threshold_type="interquartile",
            breakpoint_threshold_amount=1.5
        )
        return splitter.split_text(text)
    
    def _markdown_chunk(self, text: str) -> list:
        """Markdown分块：保持结构"""
        headers_to_split_on = [
            ("#", "h1"), ("##", "h2"), ("###", "h3")
        ]
        splitter = MarkdownHeaderTextSplitter(headers_to_split_on)
        docs = splitter.split_text(text)
        
        chunks = []
        for doc in docs:
            header_path = " > ".join(
                doc.metadata.get(h, "") for h in ["h1", "h2", "h3"]
                if doc.metadata.get(h)
            )
            chunks.append({
                "content": doc.page_content,
                "header": header_path,
                "metadata": doc.metadata
            })
        
        return chunks
    
    def _ast_chunk(self, text: str) -> list:
        """代码AST分块：按函数/类粒度"""
        import ast
        
        tree = ast.parse(text)
        chunks = []
        
        for node in ast.walk(tree):
            if isinstance(node, (ast.FunctionDef, ast.ClassDef)):
                chunks.append(ast.get_source_segment(text, node))
        
        return chunks or [text]
    
    def _fixed_chunk(self, text: str) -> list:
        """固定大小分块（兜底策略）"""
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=800,
            chunk_overlap=150,
            separators=["\n\n", "\n", "。", "!", "?", ";", " ", ""]
        )
        return splitter.split_text(text)
    
    def _post_process(self, chunks: list) -> list:
        """后处理：严格控制块大小"""
        processed = []
        
        for chunk in chunks:
            content = chunk["content"] if isinstance(chunk, dict) else chunk
            
            # 跳过空块
            if len(content.strip()) < 10:
                continue
            
            # 合并过小块
            if len(content) < self.min_chunk_size and processed:
                prev = processed[-1]
                prev_content = prev["content"] if isinstance(prev, dict) else prev
                
                if len(prev_content) + len(content) < self.max_chunk_size:
                    prev["content"] = prev_content + "\n\n" + content
                    continue
            
            processed.append(
                chunk if isinstance(chunk, dict) else {"content": chunk, "header": ""}
            )
        
        return processed

# 使用示例
chunker = AdaptiveChunker()

# 分块技术文档
tech_doc = """
# API Reference

## Authentication
使用Bearer Token进行认证...

## Endpoints

### GET /users
获取用户列表...

### POST /users
创建新用户...
"""

chunks = chunker.chunk(tech_doc, doc_type="markdown")
for i, chunk in enumerate(chunks):
    print(f"Chunk {i+1} [{chunk['header']}]: {chunk['content'][:80]}...")
```

### 1.3 分块策略选择指南

```python
CHUNKING_GUIDE = {
    "技术文档/API文档": {
        "strategy": "Markdown感知分块",
        "chunk_size": "500-800",
        "overlap": "100-150",
        "reason": "需要保持Header层级结构，方便上下文定位"
    },
    "法律合同": {
        "strategy": "语义分块",
        "chunk_size": "800-1200",
        "overlap": "200-300",
        "reason": "条款间语义关联强，语义分块能更好保持连贯性"
    },
    "客服对话": {
        "strategy": "固定大小 + 对话轮次",
        "chunk_size": "300-500",
        "overlap": "50-100",
        "reason": "对话轮次是自然分界点，固定大小保证检索粒度"
    },
    "新闻/文章": {
        "strategy": "语义分块",
        "chunk_size": "1000-1500",
        "overlap": "200",
        "reason": "段落间语义关联强，语义分块保证主题完整性"
    }
}
```

## 二、Rerank重排序技术

### 2.1 为什么需要Rerank？

```
向量检索 (First Stage)          重排序 (Second Stage)
┌──────────────────┐           ┌──────────────────┐
│ 快速但粗粒度       │  ────►   │ 慢但精粒度         │
│ 基于向量相似度     │           │ 基于深度语义理解    │
│ 适合大规模召回     │           │ 适合Top-K精排       │
│ 召回100条 → Top20 │           │ 精排20条 → Top5    │
└──────────────────┘           └──────────────────┘
```

### 2.2 实战：多种Rerank方案

```python
from typing import List
from dataclasses import dataclass
import numpy as np

@dataclass
class Document:
    content: str
    score: float
    metadata: dict = None

class RerankPipeline:
    """重排序流水线"""
    
    def __init__(self):
        self.rerankers = {}
    
    # ========== 方案1：Cross-Encoder Rerank（推荐） ==========
    def cross_encoder_rerank(
        self,
        query: str,
        documents: List[Document],
        model_name: str = "BAAI/bge-reranker-v2-m3",
        top_k: int = 5
    ) -> List[Document]:
        """使用Cross-Encoder重排序"""
        from sentence_transformers import CrossEncoder
        
        model = self._get_model(model_name, lambda: CrossEncoder(model_name))
        
        # 构建 (query, document) 对
        pairs = [(query, doc.content) for doc in documents]
        
        # Cross-Encoder打分
        scores = model.predict(pairs, show_progress_bar=False)
        
        # 更新分数并排序
        for doc, score in zip(documents, scores):
            doc.score = float(score)
        
        documents.sort(key=lambda x: x.score, reverse=True)
        return documents[:top_k]
    
    # ========== 方案2：LLM Rerank（最高精度） ==========
    def llm_rerank(
        self,
        query: str,
        documents: List[Document],
        top_k: int = 5
    ) -> List[Document]:
        """使用LLM进行重排序"""
        from langchain_openai import ChatOpenAI
        
        llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
        
        # 构建Prompt
        doc_texts = "\n\n---\n\n".join(
            f"[{i+1}] {doc.content[:300]}" 
            for i, doc in enumerate(documents)
        )
        
        prompt = f"""查询：{query}

以下是与查询相关的文档列表。请根据相关性从高到低排序。

文档：
{doc_texts}

请返回排序后的文档编号（只返回数字，用逗号分隔，如：3,1,5,2,4）。
最相关的排在前面。
"""
        
        response = llm.invoke(prompt)
        
        # 解析排序结果
        rankings = [int(x.strip()) - 1 for x in response.content.split(",")]
        
        # 重排序
        reranked = [documents[i] for i in rankings if i < len(documents)]
        
        # 重新打分
        n = len(reranked)
        for i, doc in enumerate(reranked):
            doc.score = (n - i) / n
        
        return reranked[:top_k]
    
    # ========== 方案3：MMR多样性重排序 ==========
    def mmr_rerank(
        self,
        query_embedding: np.ndarray,
        documents: List[Document],
        embeddings: List[np.ndarray],
        top_k: int = 5,
        lambda_param: float = 0.7
    ) -> List[Document]:
        """
        MMR(Maximal Marginal Relevance)重排序
        
        平衡相关性和多样性，避免重复内容
        
        Args:
            lambda_param: 0=纯多样性, 1=纯相关性
        """
        n = len(documents)
        selected = []
        remaining = list(range(n))
        
        while len(selected) < min(top_k, n):
            mmr_scores = []
            
            for idx in remaining:
                # 相关性分数
                relevance = np.dot(query_embedding, embeddings[idx])
                
                # 多样性惩罚（与已选文档的最大相似度）
                diversity_penalty = 0
                if selected:
                    max_sim = max(
                        np.dot(embeddings[idx], embeddings[s])
                        for s in selected
                    )
                    diversity_penalty = lambda_param * max_sim
                
                mmr_score = lambda_param * relevance - diversity_penalty
                mmr_scores.append(mmr_score)
            
            # 选择MMR最高的
            best_idx = remaining[np.argmax(mmr_scores)]
            selected.append(best_idx)
            remaining.remove(best_idx)
        
        return [documents[i] for i in selected]
    
    def _get_model(self, name: str, factory):
        """懒加载模型"""
        if name not in self.rerankers:
            self.rerankers[name] = factory()
        return self.rerankers[name]

# ========== 完整RAG Pipeline ==========
class OptimizedRAG:
    """优化后的RAG流水线：分块 → 检索 → 重排序"""
    
    def __init__(self):
        self.chunker = AdaptiveChunker()
        self.reranker = RerankPipeline()
        self.vector_store = None
    
    def process(
        self,
        query: str,
        documents: List[str],
        top_k: int = 5,
        rerank_method: str = "cross_encoder"
    ) -> List[Document]:
        """完整的RAG处理流程"""
        
        # Step 1: 分块
        all_chunks = []
        for doc in documents:
            chunks = self.chunker.chunk(doc)
            all_chunks.extend(chunks)
        
        print(f"分块完成：共{len(all_chunks)}个块")
        
        # Step 2: 向量检索（First Stage）
        retrieved = self._vector_search(query, all_chunks, top_k=20)
        print(f"向量检索：召回{len(retrieved)}个候选")
        
        # Step 3: 重排序（Second Stage）
        if rerank_method == "cross_encoder":
            final = self.reranker.cross_encoder_rerank(query, retrieved, top_k)
        elif rerank_method == "llm":
            final = self.reranker.llm_rerank(query, retrieved, top_k)
        else:
            final = retrieved[:top_k]
        
        print(f"重排序后：返回Top-{len(final)}个结果")
        
        return final
    
    def _vector_search(self, query: str, chunks: list, top_k: int) -> list:
        """向量检索（模拟）"""
        # 实际实现会使用向量数据库
        results = []
        for i, chunk in enumerate(chunks[:top_k * 2]):
            content = chunk["content"] if isinstance(chunk, dict) else chunk
            results.append(Document(
                content=content,
                score=0.9 - (i * 0.02),
                metadata=chunk if isinstance(chunk, dict) else {}
            ))
        return results[:top_k]
```

## 三、最佳实践：Chunk + Rerank组合拳

### 3.1 推荐的两阶段流程

```python
class ProductionRAG:
    """生产级RAG系统"""
    
    def __init__(self):
        self.chunker = AdaptiveChunker()
        self.reranker = RerankPipeline()
        
        # Stage 1配置：宽召回
        self.recall_top_k = 50
        
        # Stage 2配置：精排
        self.final_top_k = 5
        
        # Stage 3配置：去重
        self.dedup_threshold = 0.85
    
    def query(self, query: str) -> List[Document]:
        """完整查询流程"""
        
        # Stage 1: 向量检索（宽召回）
        candidates = self._dense_retrieve(query, top_k=self.recall_top_k)
        
        # Stage 2: Rerank精排
        reranked = self.reranker.cross_encoder_rerank(query, candidates)
        
        # Stage 3: 去重
        deduped = self._deduplicate(reranked)
        
        # Stage 4: 截取Top-K
        return deduped[:self.final_top_k]
    
    def _deduplicate(self, documents: List[Document]) -> List[Document]:
        """基于内容相似度去重"""
        from sklearn.feature_extraction.text import TfidfVectorizer
        from sklearn.metrics.pairwise import cosine_similarity
        
        if len(documents) <= 1:
            return documents
        
        texts = [doc.content for doc in documents]
        vectorizer = TfidfVectorizer()
        tfidf_matrix = vectorizer.fit_transform(texts)
        
        # 构建相似度矩阵
        sim_matrix = cosine_similarity(tfidf_matrix)
        
        keep = [True] * len(documents)
        for i in range(len(documents)):
            if not keep[i]:
                continue
            for j in range(i + 1, len(documents)):
                if sim_matrix[i][j] > self.dedup_threshold:
                    keep[j] = False
        
        return [doc for doc, k in zip(documents, keep) if k]
    
    def _dense_retrieve(self, query: str, top_k: int) -> list:
        """向量检索实现"""
        # 实际对接向量数据库
        pass
```

### 3.2 性能权衡指南

| 方案 | 延迟 | 精度 | 成本 | 推荐场景 |
|------|------|------|------|---------|
| 纯向量检索 | ~50ms | 中等 | $ | 实时对话 |
| 向量检索 + Cross-Encoder | ~200ms | 高 | $$ | 常规RAG |
| 向量检索 + LLM Rerank | ~2s | 最高 | $$$ | 高精度需求 |
| 向量检索 + MMR | ~100ms | 中等 | $ | 需要多样性 |

## 面试题

### 面试题1：为什么在RAG系统中需要两阶段检索（Recall + Rerank），而不是直接做单阶段高精度检索？

**答案：**

**核心原因：**

1. **效率与精度的Trade-off**
```python
# 单阶段：用Cross-Encoder检索100万条 → 太慢（需要N×M次推理）
# 两阶段：向量检索快筛 → 对Top-50用Cross-Encoder → 速度快1000x
```

2. **检索层次对比：**

| 阶段 | 方法 | 速度 | 精度 | 适用规模 |
|------|------|------|------|---------|
| Stage1: Recall | Bi-Encoder/ANN | ms级 | 中等 | 百万级 |
| Stage2: Rerank | Cross-Encoder | 100ms级 | 高 | 十到百级 |

3. **技术原因：**
- Bi-Encoder生成独立向量，可预计算和索引 → 大规模快速检索
- Cross-Encoder将查询和文档拼接输入 → 深度交互，精度高但无法预计算
- 两阶段结合实现了效率与精度的帕累托最优

4. **类比：搜索引擎**
```
搜索引擎也是两阶段：
Stage1: 倒排索引快速召回（类似向量检索）
Stage2: 精排模型排序（类似Rerank）
```

### 面试题2：在Chunk策略中，overlap（重叠）的作用是什么？如何根据文档类型设置合理的overlap大小？

**答案：**

**Overlap的核心作用：**

1. **保持上下文连贯性**
```python
# 无Overlap：文档被硬截断
Chunk1: "Python是一种解释型语言，它的特点包括简"
Chunk2: "洁的语法和丰富的生态系统..."
# → "简洁"被拆开，语义断裂

# 有Overlap：上下文平滑过渡
Chunk1: "Python是一种解释型语言，它的特点包括简洁的语法和丰富的生态系统"
Chunk2: "它的特点包括简洁的语法和丰富的生态系统，广泛应用于..."
# → Overlap保持了语义完整
```

2. **Overlap设置建议：**

| 文档类型 | chunk_size | overlap | overlap比例 | 原因 |
|---------|-----------|---------|------------|------|
| 技术文档 | 500-800 | 100-150 | ~20% | 逻辑紧密，需要较多重叠 |
| 法律法规 | 800-1200 | 200-300 | ~25% | 条款间交叉引用多 |
| 新闻文章 | 1000-1500 | 150-200 | ~15% | 段落相对独立 |
| 代码 | 800-2000 | 300-500 | ~25% | 函数间调用关系需保持 |
| 对话记录 | 300-500 | 50-100 | ~20% | 轮次间上下文重要 |

3. **动态Overlap策略：**

```python
class DynamicOverlapChunker:
    def chunk_with_adaptive_overlap(self, text: str) -> list:
        """根据文本特征动态调整overlap"""
        
        # 计算句子密度（每千字符的句子数）
        sentences = text.replace(".", ".").split("。")
        density = len(sentences) / (len(text) / 1000)
        
        if density > 5:  # 句子密集 → 增大overlap
            overlap_ratio = 0.25
        elif density > 3:
            overlap_ratio = 0.20
        else:
            overlap_ratio = 0.15
        
        chunk_size = 800
        overlap = int(chunk_size * overlap_ratio)
        
        print(f"句子密度={density:.1f}, overlap={overlap}")
        
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=overlap
        )
        return splitter.split_text(text)
```

## 总结

Chunk策略和Rerank优化是RAG系统从"能用"到"好用"的关键：

1. **Chunk策略**：根据文档类型选择合适的分块方式，语义分块是大多数场景的最佳选择
2. **Overlap设置**：保持20%左右的overlap，确保上下文连贯性
3. **两阶段检索**：宽召回+精排的组合，在效率和精度间取得最优平衡
4. **Rerank选择**：常规场景用Cross-Encoder，高精度场景用LLM Rerank

## 参考资料

- [BGE-Reranker](https://huggingface.co/BAAI/bge-reranker-v2-m3)
- [LangChain SemanticChunker](https://python.langchain.com/docs/modules/data_connection/document_transformers/semantic-chunker)
- [MMR论文](https://www.cs.cmu.edu/~jgc/publication/The_Use_MMR_Diversity_Based_LTMIR_1998.pdf)
