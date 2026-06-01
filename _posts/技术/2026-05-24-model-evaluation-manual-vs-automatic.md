---
layout: post
title: "模型评估：人工评估vs自动评估框架对比"
date: 2026-05-24
categories: [AI Agent, 模型评估]
tags: [模型评估, 人工评估, 自动评估, LLM-as-Judge, Python]
---

## 引言

当你上线了一个 AI Agent 后，如何知道它是否真的变好了？Prompt 调整、模型切换、RAG 参数优化——每一次迭代都需要评估来验证效果。评估方法分为两大阵营：人工评估和自动评估。本文从工程实践角度对比两者的优劣，并给出可落地的混合评估方案。

## 评估维度梳理

在讨论方法之前，先明确 AI Agent 应用需要评估哪些维度：

| 维度 | 说明 | 示例指标 |
|------|------|---------|
| 准确性 | 回答是否正确 | 事实正确率、幻觉率 |
| 完整性 | 是否覆盖用户需求 | 信息覆盖率 |
| 相关性 | 回答是否切题 | 语义相似度 |
| 安全性 | 是否有害/违规 | 拒答率、越狱成功率 |
| 效率 | 响应速度 | 首 token 延迟、端到端延迟 |
| 工具调用 | 工具选择和参数准确性 | 工具调用成功率 |
| 对话连贯性 | 多轮对话表现 | 上下文遗忘率 |

## 人工评估

### 实施方式

人工评估的核心是标注人员根据评分标准对模型输出打分。常见的评分方式有：

- **A/B 对比**：标注员看到两个模型对同一问题的回答，选择更好的一个
- **Likert 量表**：1-5 分李克特量表，标注员对单个回答的每个维度打分
- **排序法**：对多个回答进行偏好排序

### 评分标准的标准化

```python
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from enum import IntEnum


class ScoreLevel(IntEnum):
    """Likert 5 级评分"""
    VERY_POOR = 1
    POOR = 2
    ACCEPTABLE = 3
    GOOD = 4
    EXCELLENT = 5


@dataclass
class RubricCriterion:
    """评分维度定义"""
    name: str
    description: str
    score_descriptions: Dict[int, str] = field(default_factory=dict)


# 定义 Agent 回答评估的评分标准
AGENT_RUBRIC = [
    RubricCriterion(
        name="准确性",
        description="回答中的事实陈述是否准确无误",
        score_descriptions={
            5: "所有事实准确，无任何错误",
            4: "核心事实准确，极少数次要细节不精确",
            3: "主要事实基本准确，存在个别偏差但不影响理解",
            2: "存在明显事实错误，可能误导用户",
            1: "大量事实错误，回答基本不可信",
        }
    ),
    RubricCriterion(
        name="完整性",
        description="回答是否完整覆盖了用户问题的所有关键方面",
        score_descriptions={
            5: "完全覆盖所有方面，并有适当扩展",
            4: "覆盖所有关键方面",
            3: "覆盖了主要方面，遗漏了部分次要信息",
            2: "只覆盖了部分关键方面，有重要遗漏",
            1: "基本未回答用户问题",
        }
    ),
    RubricCriterion(
        name="工具使用",
        description="是否正确选择和调用了所需的工具",
        score_descriptions={
            5: "工具选择完全正确，参数精准",
            4: "工具选择正确，参数基本合适",
            3: "工具选择正确但参数可优化",
            2: "工具选择有误或遗漏必要工具",
            1: "完全选错工具或未使用应有的工具",
        }
    ),
    RubricCriterion(
        name="表达质量",
        description="语言表达的清晰度、逻辑性和专业性",
        score_descriptions={
            5: "表达清晰流畅，逻辑严密，专业得体",
            4: "表达清晰，逻辑通顺，偶有小瑕疵",
            3: "基本可理解，但逻辑或表达上有改进空间",
            2: "表达混乱，逻辑跳跃，影响理解",
            1: "完全无法理解或严重不专业",
        }
    ),
]
```

### 人工评估的优劣势

**优势**：
- 能捕捉细微的语义差异和主观体验（如语气是否得体）
- 可以发现自动评估难以量化的新问题类型
- 评分标准可以灵活调整

**劣势**：
- 成本高：每轮迭代都需要标注，耗时耗力
- 一致性差：不同标注员对同一回答的打分可能差异大（需计算 Inter-Annotator Agreement）
- 规模受限：通常只能评估数百条样本

## 自动评估

自动评估通过程序化手段对模型输出打分，主要包括以下几类：

### 1. 基于规则的评估

```python
import re
from typing import List, Tuple


class RuleBasedEvaluator:
    """基于规则的自动评估器"""

    def __init__(self):
        # 定义检查规则
        self.safety_patterns = [
            (re.compile(r"(密码|password)\s*[=:]\s*\S+", re.I), "疑似密码泄露"),
            (re.compile(r"(\d{17}[\dXx])", re.I), "疑似身份证号"),
        ]
        self.quality_patterns = [
            (re.compile(r"作为.*AI.*我.*不"), "包含不必要的 AI 身份声明"),
            (re.compile(r"很抱歉.*无法"), "过度道歉"),
        ]

    def evaluate_safety(self, response: str) -> List[Tuple[str, str]]:
        """检查安全合规性"""
        issues = []
        for pattern, description in self.safety_patterns:
            if pattern.search(response):
                issues.append((description, pattern.search(response).group()))
        return issues

    def evaluate_quality(self, response: str) -> List[str]:
        """检查回答质量"""
        issues = []
        for pattern, description in self.quality_patterns:
            if pattern.search(response):
                issues.append(description)
        # 检查回答长度
        if len(response) < 50:
            issues.append("回答过短，可能不够详尽")
        # 检查是否包含代码块但未闭合
        if response.count("```") % 2 != 0:
            issues.append("Markdown 代码块未正确闭合")
        return issues

    def evaluate(self, query: str, response: str) -> dict:
        return {
            "safety_issues": self.evaluate_safety(response),
            "quality_issues": self.evaluate_quality(response),
            "length": len(response),
        }
```

### 2. 基于语义相似度的评估

```python
from sentence_transformers import SentenceTransformer
import numpy as np


class SemanticEvaluator:
    """基于语义相似度的评估器"""

    def __init__(self, model_name: str = "BAAI/bge-large-zh-v1.5"):
        self.model = SentenceTransformer(model_name)

    def compute_relevance(self, query: str, response: str) -> float:
        """计算 query-response 的语义相关性"""
        q_emb = self.model.encode(query, normalize_embeddings=True)
        r_emb = self.model.encode(response, normalize_embeddings=True)
        return float(q_emb @ r_emb.T)

    def compute_faithfulness(self, response: str, source: str) -> float:
        """
        计算回答与源文档的一致性（RAG 场景）
        使用 sentence-level 对齐
        """
        response_sents = [s.strip() for s in response.split("。") if s.strip()]
        source_emb = self.model.encode(source, normalize_embeddings=True)

        scores = []
        for sent in response_sents:
            sent_emb = self.model.encode(sent, normalize_embeddings=True)
            max_sim = float(np.max(sent_emb @ source_emb.T))
            scores.append(max_sim)

        return float(np.mean(scores)) if scores else 0.0


# 使用示例
evaluator = SemanticEvaluator()

query = "什么是糖尿病"
response_good = "糖尿病是一种以高血糖为特征的代谢性疾病..."
response_bad = "今天天气不错，适合户外运动"

relevance_good = evaluator.compute_relevance(query, response_good)
relevance_bad = evaluator.compute_relevance(query, response_bad)

print(f"好回答的相关性: {relevance_good:.3f}")   # 预期 > 0.6
print(f"差回答的相关性: {relevance_bad:.3f}")   # 预期 < 0.3
```

### 3. LLM-as-Judge 评估

```python
from typing import List
import json


class LLMJudge:
    """使用 LLM 作为评估法官"""

    EVALUATION_PROMPT = """你是一个 AI 回答质量的评估专家。请根据以下标准评估给定的回答。

评分标准（每个维度 1-5 分）：
- 准确性：回答中的事实陈述是否准确
- 完整性：是否完整覆盖了用户问题的所有关键方面
- 相关性：回答是否紧扣用户问题
- 安全性：是否包含任何有害、偏见或不当内容
- 表达质量：语言是否清晰、专业、逻辑严密

请以 JSON 格式输出评估结果：
{
    "accuracy": <1-5>,
    "completeness": <1-5>,
    "relevance": <1-5>,
    "safety": <1-5>,
    "expression": <1-5>,
    "overall": <1-5>,
    "brief_reason": "<一句话总结>"
}

用户问题：{query}

待评估回答：
{response}

请给出你的评估："""

    def __init__(self, llm_client):
        self.llm = llm_client

    async def evaluate(self, query: str, response: str) -> dict:
        prompt = self.EVALUATION_PROMPT.format(query=query, response=response)
        result = await self.llm.generate(prompt)
        try:
            # 提取 JSON
            json_start = result.find("{")
            json_end = result.rfind("}") + 1
            return json.loads(result[json_start:json_end])
        except json.JSONDecodeError:
            return {"error": "Failed to parse judge output", "raw": result}

    async def batch_evaluate(
        self, samples: List[Tuple[str, str]]
    ) -> List[dict]:
        """批量评估"""
        import asyncio
        tasks = [self.evaluate(query, response) for query, response in samples]
        return await asyncio.gather(*tasks)
```

### LLM-as-Judge 的注意事项

1. **位置偏差**：LLM 倾向于偏好第一个出现的候选回答，应随机交换 A/B 顺序并取平均
2. **长度偏差**：LLM 倾向于给更长的回答更高分，应在 prompt 中明确要求忽略长度
3. **自利偏差**：同一厂商的 LLM 评估自己生成的回答会偏高，建议使用异构 Judge 模型
4. **一致性校验**：同一对回答应多次评估取一致性，Kappa 系数 < 0.6 说明 Judge 不适合该任务

## 混合评估方案

实际项目中，推荐采用"自动评估做初筛 + 人工评估做精判"的混合方案：

```python
from dataclasses import dataclass, field
from typing import List, Optional
import statistics


@dataclass
class HybridEvaluationPipeline:
    """混合评估流水线"""

    rule_evaluator: RuleBasedEvaluator = field(default_factory=RuleBasedEvaluator)
    semantic_evaluator: SemanticEvaluator = field(default_factory=SemanticEvaluator)
    llm_judge: Optional[LLMJudge] = None

    # 阈值配置
    min_relevance: float = 0.5
    min_faithfulness: float = 0.4
    overall_pass_threshold: float = 3.5

    async def evaluate_sample(
        self, query: str, response: str, source_doc: Optional[str] = None
    ) -> dict:
        """对一个样本执行完整评估"""
        result = {
            "query": query,
            "response": response[:200] + "...",
        }

        # 第一层：规则检查（快速过滤明显问题）
        rule_result = self.rule_evaluator.evaluate(query, response)
        result["rule_check"] = rule_result

        if rule_result["safety_issues"]:
            result["status"] = "FAILED_SAFETY"
            result["score"] = 1.0
            return result

        # 第二层：语义相似度评估
        relevance = self.semantic_evaluator.compute_relevance(query, response)
        result["relevance"] = round(relevance, 3)

        if relevance < self.min_relevance:
            result["status"] = "FAILED_RELEVANCE"
            result["score"] = 2.0
            return result

        if source_doc:
            faithfulness = self.semantic_evaluator.compute_faithfulness(
                response, source_doc
            )
            result["faithfulness"] = round(faithfulness, 3)

        # 第三层：LLM Judge 精细评估
        if self.llm_judge:
            judge_result = await self.llm_judge.evaluate(query, response)
            result["judge_scores"] = judge_result
            result["score"] = judge_result.get("overall", 3)

        # 判定
        result["status"] = (
            "PASSED" if result.get("score", 0) >= self.overall_pass_threshold
            else "FAILED"
        )
        return result

    def generate_report(self, results: List[dict]) -> dict:
        """生成评估报告"""
        scores = [r.get("score", 0) for r in results]
        passed = sum(1 for r in results if r.get("status") == "PASSED")

        return {
            "total_samples": len(results),
            "pass_rate": f"{passed / len(results) * 100:.1f}%",
            "mean_score": round(statistics.mean(scores), 2) if scores else 0,
            "median_score": round(statistics.median(scores), 2) if scores else 0,
            "std_dev": round(statistics.stdev(scores), 2) if len(scores) > 1 else 0,
            "fail_breakdown": {
                "safety": sum(1 for r in results if r.get("status") == "FAILED_SAFETY"),
                "relevance": sum(1 for r in results if r.get("status") == "FAILED_RELEVANCE"),
                "quality": sum(1 for r in results if r.get("status") == "FAILED"),
            }
        }
```

## 评估频率与场景匹配

| 场景 | 推荐评估方式 | 评估规模 | 频率 |
|------|-------------|---------|------|
| Prompt 微调 | 自动评估（LLM-as-Judge） | 100-500 条 | 每次调整后 |
| 模型切换 | 自动评估 + 人工抽检 | 500-2000 条 | 切换前后对比 |
| 上线前验收 | 人工评估为主 | 200+ 条 | 每次大版本 |
| 线上监控 | 自动评估（规则+语义） | 抽样 | 持续/每日 |

## 总结

评估不是一蹴而就的工作，而是贯穿 AI Agent 全生命周期的持续工程实践。自动评估提供了规模和速度，人工评估提供了精度和洞察力。一个成熟的评估体系应当将两者有机结合：用自动评估覆盖每一次迭代的快速反馈，用人工评估为关键决策提供可靠依据。

## 面试题

**1. 使用 LLM-as-Judge 进行自动评估时，有哪些常见的偏差（Bias）？如何缓解？**

常见偏差和缓解方法：
- **位置偏差（Position Bias）**：LLM 倾向于偏好 A/B 测试中排在前面的回答。缓解方法：对每对样本做两次评估，交换顺序后取平均；或要求 Judge 先分别评分再对比。
- **长度偏差（Length Bias）**：Judge 倾向于给更长的回答更高分。缓解方法：在 prompt 中明确指令"忽略回答长度，仅评估质量"；或在评分时使用长度归一化。
- **自利偏差（Self-Enhancement Bias）**：GPT-4 作为 Judge 更偏好 GPT-4 生成的回答。缓解方法：使用异构 Judge（如用 Claude 评估 GPT 的输出）；或者在多个 Judge 之间取共识。
- **格式偏差**：Judge 可能因输出格式差异（如 Markdown vs 纯文本）给出不同分数。缓解方法：统一输出的格式后再评估。

**2. 人工评估中，如何衡量和提升标注一致性（Inter-Annotator Agreement, IAA）？**

核心方法：
- **衡量指标**：分类任务用 Cohen's Kappa 或 Fleiss' Kappa（多标注员）；Likert 量表用加权 Kappa 或 Intraclass Correlation Coefficient（ICC）。一般要求 Kappa > 0.6 为可接受，> 0.8 为优秀。
- **提升一致性**：①制定详细的评分标准（Rubric），包含每个分数级别的具体示例；②组织标注前培训，让标注员在 10-20 条样本上对齐理解；③设置锚定样本（Gold Standard），每个批次混入 5%-10% 已知得分的标准样本，监控标注质量；④对一致性低的维度做归因分析——是标准不清晰还是标注员理解偏差，针对性修正。
- **争议处理**：对 Kappa < 0.4 的样本，应由第三方专家仲裁，并将仲裁结果同步给所有标注员以持续对齐。