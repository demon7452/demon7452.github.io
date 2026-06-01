---
layout: post
title: "Fine-tuning入门：LoRA与QLoRA原理及数据准备"
date: 2026-04-26
categories: [AI Agent, Fine-tuning, 应用开发]
tags: [LoRA, QLoRA, Fine-tuning, 数据准备, PEFT, Python]
---

在 AI Agent 开发中，Prompt Engineering 能解决 80% 的问题，但当你需要模型稳定地输出特定格式、掌握私有领域术语、或者遵循特殊的行为规范时，Fine-tuning 是更可靠的方案。而 LoRA 和 QLoRA 这两项技术让微调从「需要 8 张 A100」变成了「一张消费级显卡就能跑」。

## 为什么不用全量微调？

全量微调（Full Fine-tuning）的问题非常现实：

| 模型规模 | 全量微调显存需求 | LoRA 显存需求 |
|---------|----------------|--------------|
| Llama 3 8B (FP16) | ~60 GB | ~18 GB |
| Qwen 2.5 7B (FP16) | ~56 GB | ~16 GB |
| DeepSeek 7B (FP16) | ~56 GB | ~16 GB |

对于个人开发者和中小团队，全量微调的门槛太高。LoRA 的出现让微调民主化了。

## LoRA 原理：低秩分解

LoRA（Low-Rank Adaptation）的核心思想是：**模型在适配下游任务时，权重的更新量是低秩的**。因此不需要更新全部权重，只需训练两个小矩阵 A 和 B，让它们的乘积近似原始权重的更新量。

```
原始前向传播: h = W₀ · x
LoRA 前向传播: h = W₀ · x + (B · A) · x

其中: W₀ ∈ R^(d×d)   [冻结]
      A  ∈ R^(r×d)   [可训练]
      B  ∈ R^(d×r)   [可训练]
      r << d          [秩，通常取 8~64]
```

参数量对比：假设 d=4096, r=16，原始权重有 16M 参数，LoRA 矩阵只有 2×4096×16 ≈ 131K 参数，减少了 99%+。

## QLoRA：量化 + LoRA

QLoRA 在 LoRA 基础上加了两个关键技术：

1. **4-bit NormalFloat 量化**：将预训练权重压缩到 4-bit，进一步降低显存。
2. **双重量化**：对量化常数再做一次量化，榨干最后一点显存。

这让 7B 模型的微调显存从 ~18GB 进一步降到 ~10GB，RTX 3060（12GB）即可胜任。

```python
import torch
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    BitsAndBytesConfig,
    TrainingArguments,
    Trainer,
    DataCollatorForLanguageModeling
)
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from datasets import Dataset

# ---------- 1. 量化配置（QLoRA 的核心） ----------

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,                    # 4-bit 量化
    bnb_4bit_quant_type="nf4",            # NormalFloat4
    bnb_4bit_compute_dtype=torch.bfloat16, # 计算时用 BF16
    bnb_4bit_use_double_quant=True        # 双重量化
)

# ---------- 2. 加载模型 ----------

model_name = "Qwen/Qwen2.5-7B-Instruct"

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto",                   # 自动分配设备
    trust_remote_code=True
)

tokenizer = AutoTokenizer.from_pretrained(model_name, trust_remote_code=True)
tokenizer.pad_token = tokenizer.eos_token

# ---------- 3. LoRA 配置 ----------

# 为 k-bit 训练做准备
model = prepare_model_for_kbit_training(model)

lora_config = LoraConfig(
    r=16,                                # LoRA 秩
    lora_alpha=32,                       # 缩放因子
    target_modules=[                     # 插入 LoRA 的目标模块
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj"
    ],
    lora_dropout=0.05,                   # Dropout 防过拟合
    bias="none",                         # 不训练 bias
    task_type="CAUSAL_LM"
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# 输出: trainable params: 41,943,040 || all params: 7,657,291,776 || trainable%: 0.55%
```

## 数据准备：高质量微调数据的黄金法则

Fine-tuning 的成败 70% 取决于数据质量。以下是实际项目中验证过的数据准备流程。

### 数据格式

```python
# 对话式微调数据的标准格式
training_data = [
    {
        "messages": [
            {"role": "system", "content": "你是一个擅长Python代码审查的AI助手。"},
            {"role": "user", "content": "这段代码有什么问题？\n\ndef foo():\n    return 1/0"},
            {"role": "assistant", "content": "这段代码有一个潜在的除零错误..."}
        ]
    },
    # ... 更多样本
]
```

### 数据清洗 Pipeline

```python
import json
import re
from typing import Iterator

class DataCleaner:
    """Quality assurance pipeline for fine-tuning datasets."""

    MIN_RESPONSE_LENGTH = 20      # 最少回复字数
    MAX_RESPONSE_LENGTH = 2048    # 最多回复 token 数（估算）
    FORBIDDEN_PATTERNS = [
        r"作为AI.*无法",
        r"作为一个.*模型",
        r"对不起.*不能",
    ]

    def clean(self, raw_data: list[dict]) -> list[dict]:
        """Apply all cleaning rules."""
        cleaned = []
        for item in raw_data:
            if self._is_valid(item):
                cleaned.append(self._normalize(item))
        return cleaned

    def _is_valid(self, item: dict) -> bool:
        messages = item.get("messages", [])
        if len(messages) < 2:
            return False

        # 检查 assistant 的回复长度
        for msg in messages:
            if msg["role"] == "assistant":
                content = msg["content"]
                if len(content) < self.MIN_RESPONSE_LENGTH:
                    return False
                if len(content) > self.MAX_RESPONSE_LENGTH:
                    return False
                # 检查拒绝回答的模式
                for pattern in self.FORBIDDEN_PATTERNS:
                    if re.search(pattern, content):
                        return False

        return True

    def _normalize(self, item: dict) -> dict:
        """Normalize formatting: trim whitespace, fix encoding."""
        for msg in item["messages"]:
            msg["content"] = msg["content"].strip()
        return item
```

### 数据多样性策略

一个好的微调数据集应该覆盖以下维度：

```
任务多样性: 问答 / 摘要 / 代码生成 / 分类 / 翻译 ...
难度层次:  简单 → 中等 → 复杂（建议比例 2:5:3）
领域覆盖:  至少覆盖目标场景的 5 个子领域
风格变化:  正式 / 半正式 / 技术风格 (避免全是同一种口吻)
```

### 数据量经验值

| 任务复杂度 | 推荐数据量 | 说明 |
|-----------|-----------|------|
| 格式对齐 | 200-500 条 | 让模型学会输出特定 JSON/Table 格式 |
| 领域适配 | 1000-3000 条 | 注入领域术语和规范 |
| 行为调整 | 3000-10000 条 | 改变模型的回答风格和决策偏好 |
| 知识注入 | 10000+ 条 | 教模型全新的知识（建议考虑 RAG 替代） |

## 训练配置与启动

```python
training_args = TrainingArguments(
    output_dir="./qwen-lora-agent",
    per_device_train_batch_size=2,      # 根据显存调整
    gradient_accumulation_steps=4,      # 模拟更大的 batch size
    num_train_epochs=3,
    learning_rate=2e-4,
    fp16=True,                          # 混合精度训练
    logging_steps=10,
    save_strategy="epoch",
    warmup_ratio=0.03,
    lr_scheduler_type="cosine",
    optim="paged_adamw_8bit",           # QLoRA 专用优化器
    report_to="none"
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset,
    data_collator=DataCollatorForLanguageModeling(
        tokenizer=tokenizer, mlm=False
    ),
)

trainer.train()

# 保存 LoRA 权重
model.save_pretrained("./qwen-lora-agent-final")
tokenizer.save_pretrained("./qwen-lora-agent-final")
```

## 推理时加载 LoRA 权重

```python
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained(
    model_name,
    device_map="auto",
    torch_dtype=torch.bfloat16
)
model = PeftModel.from_pretrained(base_model, "./qwen-lora-agent-final")

inputs = tokenizer("请审查以下 Python 代码：\n\ndef divide(a, b): return a/b",
                    return_tensors="pt").to("cuda")
outputs = model.generate(**inputs, max_new_tokens=512)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

## LoRA 超参数调优指南

| 参数 | 推荐范围 | 作用 |
|------|---------|------|
| r (秩) | 8~32 | 越大学习能力越强，但过拟合风险增加 |
| lora_alpha | r × 2 | 缩放因子，一般设为 r 的 2 倍 |
| target_modules | 全 Attention 层 | 只微调 Attention 通常够用，加 MLP 层可提升上限 |
| lora_dropout | 0.05~0.1 | 数据少时适当增大防过拟合 |

## 面试题

**1. LoRA 为什么能在大幅减少可训练参数的同时保持微调效果？低秩假设在什么情况下可能不成立？**

<details>
<summary>点击展开答案</summary>

LoRA 有效的理论基础是 **Intrinsic Dimensionality Hypothesis**（内在维度假设）：预训练大模型在适配下游任务时，权重的更新矩阵 ΔW 虽然是高维的，但其有效信息可以用一个远小于原始维度的低秩空间来表达。

数学上，LoRA 将 ΔW 分解为 BA（B∈R^(d×r), A∈R^(r×d), r<<d），训练时只更新 A 和 B，推理时将 BA 合并到原始权重 W₀ 中。

低秩假设可能不成立的情况：
- **任务与预训练分布差异极大**：如让一个英文模型学一门全新的编程语言语法，需要的知识更新量可能超出低秩表达能力。
- **需要记忆大量新事实**：LoRA 通过调整已有知识表达来适配，而不是存储大量新事实。如果需要注入海量领域知识，全量微调或 RAG 可能更合适。
- **秩 r 选择过小**：如果 r 远小于任务实际需要的秩，会导致欠拟合。

</details>

**2. 在准备微调数据时，如果发现训练集中 60% 的样本都是「代码审查」任务，10% 是「写文档」，30% 是「回答技术问题」，这会导致什么问题？你会如何调整？**

<details>
<summary>点击展开答案</summary>

会导致两个问题：

1. **灾难性遗忘不均衡**：模型在「代码审查」上过度拟合，可能显著削弱「写文档」和「回答技术问题」的能力。微调不是加法，而是对已有能力的重新分配。

2. **实际使用中的偏差放大**：如果 Agent 上线后被调用的三类任务比例与训练分布不同（如实际 80% 是「回答技术问题」），模型会在高频场景中表现不佳。

调整策略：
- **上采样 / 下采样**：对少数类（写文档）做重复或数据增强，对多数类（代码审查）做随机下采样。
- **分层抽样**：按任务类型分层，每类取等量样本（如各 500 条），确保训练时每个 batch 的类别分布均衡。
- **损失加权**：对少数类的 loss 赋予更高权重，让模型更关注这些样本。
- **评测驱动**：在验证集中按实际线上分布设置各类任务比例，以验证集指标而非训练 loss 作为模型选择标准。

</details>