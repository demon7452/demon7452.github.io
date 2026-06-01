---
layout: post
title: RAG 与 Agentic RAG 技术解析
category: 技术
tags: RAG AgenticRAG LLM AI
keywords: RAG Agentic RAG 检索增强生成 大模型
description:
---

### 参考文档
* [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
* [What is Agentic RAG?](https://weaviate.io/blog/what-is-agentic-rag)
* [LangGraph: Build Agentic RAG](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_agentic_rag/)

### RAG（检索增强生成）

RAG（Retrieval-Augmented Generation）是一种将信息检索与大语言模型生成相结合的技术范式，核心流程如下：

```text
用户查询 → 向量检索（从知识库召回相关文档）→ 拼接上下文 → LLM 生成回答
```

**关键环节：**

1. **文档分块（Chunking）**：将长文档切分为合适粒度的文本片段
2. **向量化（Embedding）**：将文本片段转为高维向量存入向量数据库
3. **检索（Retrieval）**：对用户查询向量化后，在向量库中做相似度搜索，取 Top-K 结果
4. **增强生成（Augmented Generation）**：将检索结果作为上下文注入 Prompt，由 LLM 生成最终回答

**RAG 的优势：**
- 有效缓解 LLM 的幻觉问题，回答有据可查
- 知识可动态更新，无需重新训练模型
- 适用于企业知识库问答、文档助手等场景

**RAG 的局限：**
- 单次检索固定，无法根据中间结果动态调整
- 对多步推理、需要综合多源信息的复杂问题支持较弱
- 检索质量高度依赖分块策略和 Embedding 模型

### Agentic RAG（智能体增强检索生成）

Agentic RAG 在传统 RAG 的基础上引入 **Agent（智能体）** 概念，赋予系统自主规划、动态决策和多轮交互的能力。

```text
用户查询 → Agent 规划（拆解子问题）→ 多轮检索（工具调用/切换数据源）
         → 结果验证与筛选 → 综合推理 → 生成回答
```

**核心增强：**

1. **动态路由（Routing）**：Agent 根据查询类型自动选择最佳检索策略（向量检索 / 关键词检索 / 结构化查询）
2. **多步推理（Multi-hop Reasoning）**：将复杂问题拆解为子问题，逐步检索并聚合信息
3. **工具调用（Tool Use）**：不仅检索文档，还能调用 API、查询数据库、执行计算等
4. **自我反思（Self-Reflection）**：对检索结果评估质量，不满足时自动重试或调整策略
5. **记忆与上下文管理**：在多轮对话中保持状态，避免重复检索

**典型架构（以 LangGraph 为例）：**

```python
# Agentic RAG 核心流程图
START → retrieve → grade_documents → [相关] → generate → END
                    ↓ [不相关]
                  transform_query → retrieve（重新检索）
```

**适用场景：**
- 需要跨多数据源综合分析的研究型问答
- 需要动态决策路径的客服系统（如先查 FAQ，再查知识库，最后转人工）
- 涉及实时数据查询 + 静态文档检索的混合场景

### RAG vs Agentic RAG 对比

| 维度 | RAG | Agentic RAG |
|------|-----|-------------|
| 检索次数 | 单次固定 | 多轮动态 |
| 决策方式 | 预定义流程 | Agent 自主规划 |
| 工具集成 | 仅向量检索 | 检索 + API + 计算等 |
| 适应复杂度 | 简单事实型查询 | 多步推理、综合分析 |
| 实现复杂度 | 低 | 中高 |
| 典型框架 | LangChain / LlamaIndex | LangGraph / CrewAI |

### 总结

传统 RAG 适合标准化的知识库问答场景，架构成熟、部署简单。Agentic RAG 在 RAG 之上叠加了 Agent 的规划与推理能力，能够应对更复杂的查询需求，但相应地增加了系统复杂度和延迟。实际选型时，应根据业务复杂度在二者之间权衡，而非一味追求 Agentic。
*（内容由AI生成，仅供参考）*
*（内容由AI生成，仅供参考）*
