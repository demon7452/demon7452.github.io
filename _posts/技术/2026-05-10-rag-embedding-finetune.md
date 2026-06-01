---
layout: post
title: "RAG中的Embedding模型微调与领域适配"
date: 2026-05-10
categories: [AI Agent, RAG, Embedding]
tags: [RAG, Embedding, 微调, 领域适配, Python]
---

## 引言

RAG（Retrieval-Augmented Generation）是当前 AI Agent 最核心的技术范式之一。然而，通用 Embedding 模型在垂直领域往往表现不佳——法律术语、医学缩写、金融专有名词等领域的语义相似度判断可能与通用场景大相径庭。本文聚焦 Embedding 模型的领域微调策略，结合 Python 代码展示从数据准备到模型评估的完整流程。

## 为什么需要微调 Embedding 模型

通用 Embedding 模型（如 `text-embedding-3-small`、`bge-large-zh`）在通用语料上训练，对垂直领域存在三个典型问题：

1. **术语误判**：将「PT（Physical Therapy）」与「PT（Prothrombin Time）」视为相似，因为通用语料中 PT 常作为缩写出现
2. **语境缺失**：不理解领域特有的命名实体关系，如「某药厂-A 与药厂-B 是竞品关系」
3. **长尾失效**：对低频但关键的专业术语，向量表示不够紧凑，检索召回率低

## 微调数据准备

微调 Embedding 模型的核心是构建高质量的 (query, positive, negative) 三元组。下面是一个实用的数据构建脚本：

```python
import json
import random
from typing import List, Tuple


class EmbeddingDatasetBuilder:
    """构建 Embedding 微调数据集"""

    def __init__(self, domain_name: str):
        self.domain_name = domain_name
        self.pairs = []  # (query, positive_doc)
        self.hard_negatives = {}  # query -> [hard_negative_docs]

    def add_positive_pair(self, query: str, document: str):
        """添加正样本对：用户问题 -> 相关文档"""
        self.pairs.append((query, document))

    def add_hard_negative(self, query: str, document: str):
        """添加难负样本：看起来相关但实际不相关的文档"""
        if query not in self.hard_negatives:
            self.hard_negatives[query] = []
        self.hard_negatives[query].append(document)

    def build_triplets(self, easy_negative_pool: List[str],
                       num_easy_negatives: int = 3) -> List[dict]:
        """构建 (query, positive, negative) 三元组训练数据"""
        triplets = []

        for query, positive in self.pairs:
            # 合并难负样本和简单负样本
            negatives = self.hard_negatives.get(query, []).copy()
            # 从简单负样本池随机采样
            easy_samples = random.sample(
                [d for d in easy_negative_pool if d != positive],
                min(num_easy_negatives, len(easy_negative_pool))
            )
            negatives.extend(easy_samples)

            for neg in negatives:
                triplets.append({
                    "query": query,
                    "positive": positive,
                    "negative": neg
                })

        return triplets

    def save(self, output_path: str):
        """保存为 JSONL 格式"""
        triplets = self.build_triplets([])
        with open(output_path, "w", encoding="utf-8") as f:
            for t in triplets:
                f.write(json.dumps(t, ensure_ascii=False) + "\n")
        print(f"已生成 {len(triplets)} 条三元组，保存至 {output_path}")


# 示例：医疗领域数据构建
builder = EmbeddingDatasetBuilder("medical")

# 正样本：医生常见问题和对应的病历段落
builder.add_positive_pair(
    "胰岛素抵抗如何诊断",
    "胰岛素抵抗的诊断标准包括：空腹胰岛素水平>25μU/mL、HOMA-IR指数>2.5..."
)
builder.add_positive_pair(
    "二甲双胍的副作用有哪些",
    "二甲双胍常见副作用包括胃肠道反应如恶心、腹泻，罕见但严重的副作用为乳酸酸中毒..."
)

# 难负样本：看起来相关但实际是关于其他药物的
builder.add_hard_negative(
    "二甲双胍的副作用有哪些",
    "阿卡波糖的副作用主要为腹胀、排气增多，偶见转氨酶升高..."
)

builder.save("./medical_triplets.jsonl")
```

## 使用 Sentence-Transformers 进行微调

Sentence-Transformers 提供了多种损失函数适配不同的微调场景：

| 损失函数 | 适用场景 | 特点 |
|---------|---------|------|
| `MultipleNegativesRankingLoss` | 有正样本对，无标注负样本 | 使用 batch 内其他样本作为负样本 |
| `TripletLoss` | 有三元组 (query, pos, neg) | 需要显式标注负样本 |
| `CoSENTLoss` | 有相似度分数标注 | 适合排序任务 |
| `MatryoshkaLoss` | 需要多维度向量 | 支持可变维度输出 |

```python
from sentence_transformers import (
    SentenceTransformer,
    SentenceTransformerTrainer,
    SentenceTransformerTrainingArguments
)
from sentence_transformers.losses import MultipleNegativesRankingLoss
from sentence_transformers.training_args import BatchSamplers
from datasets import Dataset


def train_embedding_model(
    base_model: str = "BAAI/bge-large-zh-v1.5",
    train_data_path: str = "./medical_triplets.jsonl",
    output_dir: str = "./medical-embedding-finetuned",
    epochs: int = 3,
    batch_size: int = 16,
    learning_rate: float = 2e-5,
):
    """使用 Sentence-Transformers 微调 Embedding 模型"""

    # 1. 加载预训练模型
    model = SentenceTransformer(base_model)
    print(f"加载基座模型: {base_model}")

    # 2. 加载训练数据
    dataset = Dataset.from_json(train_data_path)
    print(f"训练样本数: {len(dataset)}")

    # 3. 定义损失函数
    # MultipleNegativesRankingLoss 将 batch 内其他样本作为负样本
    loss = MultipleNegativesRankingLoss(model)

    # 4. 训练参数配置
    args = SentenceTransformerTrainingArguments(
        output_dir=output_dir,
        num_train_epochs=epochs,
        per_device_train_batch_size=batch_size,
        learning_rate=learning_rate,
        warmup_ratio=0.1,
        logging_steps=10,
        save_strategy="epoch",
        evaluation_strategy="no",
        # 避免过拟合
        weight_decay=0.01,
        # 混合精度训练
        fp16=True,
    )

    # 5. 创建 Trainer 并开始训练
    trainer = SentenceTransformerTrainer(
        model=model,
        args=args,
        train_dataset=dataset,
        loss=loss,
    )

    trainer.train()

    # 6. 保存模型
    model.save_pretrained(output_dir)
    print(f"模型已保存至: {output_dir}")

    return model
```

## 评估与对比

微调后需在领域测试集上评估检索效果：

```python
import numpy as np
from typing import List


def evaluate_retrieval(
    model,
    test_queries: List[str],
    corpus: List[str],
    relevant_indices: List[List[int]],
    k_values: List[int] = [1, 3, 5, 10]
) -> dict:
    """
    评估检索效果

    Args:
        test_queries: 测试查询列表
        corpus: 候选文档库
        relevant_indices: 每个查询对应的相关文档在 corpus 中的索引
        k_values: 要计算的 Recall@K 的 K 值列表
    """
    # 编码所有语料（可提前做并缓存）
    corpus_embeddings = model.encode(corpus, normalize_embeddings=True)
    query_embeddings = model.encode(test_queries, normalize_embeddings=True)

    # 计算相似度矩阵
    scores = query_embeddings @ corpus_embeddings.T  # [num_queries, num_corpus]

    results = {}
    for k in k_values:
        recalls = []
        for i, relevant in enumerate(relevant_indices):
            # 取 top-k
            top_k_indices = np.argsort(scores[i])[::-1][:k]
            # 计算 Recall@k
            hits = len(set(top_k_indices) & set(relevant))
            recall = hits / len(relevant) if relevant else 0.0
            recalls.append(recall)
        results[f"Recall@{k}"] = np.mean(recalls)

    return results


# 使用示例
test_queries = ["糖尿病的诊断标准是什么", "胰岛素如何使用"]
corpus = [
    "糖尿病诊断标准：空腹血糖≥7.0mmol/L...",
    "胰岛素注射应在餐前15-30分钟进行...",
    "高血压的诊断标准与分级...",
    "阿司匹林在心血管疾病中的应用..."
]
relevant_indices = [[0], [1]]  # query0 的相关文档是 corpus[0]

# 对比基座模型和微调模型
base_model = SentenceTransformer("BAAI/bge-large-zh-v1.5")
finetuned_model = SentenceTransformer("./medical-embedding-finetuned")

print("基座模型评估:", evaluate_retrieval(base_model, test_queries, corpus, relevant_indices))
print("微调模型评估:", evaluate_retrieval(finetuned_model, test_queries, corpus, relevant_indices))
```

## 微调最佳实践

1. **数据质量 > 数据数量**：200-500 条高质量的三元组往往优于 5000 条低质量数据。每条正样本应确保 query 和 document 有明确的语义关联。

2. **难负样本是关键**：使用 BM25 或现有模型检索出与 query 相似但实际不相关的文档作为难负样本，能让模型学会更精细的语义区分。

3. **学习率要小**：Embedding 模型微调通常使用 1e-5 到 5e-5 的学习率，过大容易破坏预训练的语义空间。

4. **防止灾难性遗忘**：可在训练数据中混入 5%-10% 的通用数据，或使用较小的学习率 + 较少的 epoch（通常 2-5 个 epoch 足够）。

5. **对比评估要有基准**：微调后必须在统一的测试集上对比基座模型、微调模型以及同领域其他模型（如 `bge-m3`）的 Recall 和 MRR 指标。

## 总结

Embedding 模型微调是提升 RAG 系统在垂直领域检索质量的关键手段。通过精心构建 (query, positive, negative) 三元组数据，使用 Sentence-Transformers 框架配合合适的损失函数，通常能在 2-5 个 epoch 内显著提升领域 Recall。微调后的模型需要系统性评估，确保检索质量有实质提升而非幻觉式优化。

## 面试题

**1. 为什么 RAG 系统中 Embedding 模型的微调比 LLM 的微调更常见也更优先？**

原因有三：第一，Embedding 模型参数量通常远小于 LLM（数百 MB vs 数十 GB），微调成本低、速度快。第二，Embedding 模型直接决定了检索质量——检索阶段如果召回不到正确文档，后续 LLM 再强也无法生成正确答案，属于"上游瓶颈"。第三，LLM 微调可能导致幻觉增加和安全性下降，而 Embedding 微调的风险边界更可控。实际工程中，优化检索（Embedding 微调 + 混合检索 + 重排序）的 ROI 往往高于微调 LLM。

**2. 在 Embedding 微调中，如何处理正负样本不平衡的问题？具体有哪些技巧？**

处理策略：
- **In-batch negatives**：使用 `MultipleNegativesRankingLoss`，将同一 batch 内其他样本的 positive document 作为当前样本的负样本，无需显式标注负样本，天然缓解不平衡。
- **Hard negative mining**：用 BM25 或当前模型检索 top-50，人工标注其中不相关的作为难负样本，而不是随机采样。
- **Temperature scaling**：在损失函数中引入温度参数，控制对难负样本的关注程度。较小温度（如 0.05）会让模型更关注难负样本。
- **梯度累积**：增大有效 batch size（通过梯度累积），让每个正样本能接触到更多负样本，缓解 batch 内正负比过高的问题。