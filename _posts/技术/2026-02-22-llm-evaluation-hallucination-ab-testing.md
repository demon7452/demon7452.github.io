---
layout: post
title: "LLM评估方法：幻觉检测与A/B测试"
category: 技术
tags: [LLM评估, 幻觉检测, A/B测试, AI Agent评测, Prompt评测]
keywords: [LLM Evaluation, Hallucination Detection, A/B Testing, AI Agent Benchmark, Prompt Evaluation]
description: "系统讲解LLM应用中的幻觉检测方法和A/B测试框架，帮助团队建立科学的Agent评估体系，确保生产环境质量。"
date: 2026-02-22
---

# LLM评估方法：幻觉检测与A/B测试

## 引言

AI Agent上线后，如何**量化评估**其表现是一个核心难题。与传统软件不同，LLM输出具有不确定性，传统的单元测试和集成测试无法覆盖。**幻觉检测**和**A/B测试**是解决这一问题的两种关键方法。

本文将系统讲解这两种评估方法，提供可直接使用的评估框架代码。

## 一、幻觉检测

### 1.1 幻觉分类

```python
from enum import Enum
from dataclasses import dataclass

class HallucinationType(Enum):
    """幻觉类型"""
    FACTUAL_ERROR = "事实错误"       # 编造不存在的事实
    ATTRIBUTION_ERROR = "归因错误"   # 张冠李戴
    LOGICAL_ERROR = "逻辑错误"       # 推理矛盾
    NUMERICAL_ERROR = "数值错误"     # 数字、日期错误
    CITATION_ERROR = "引用错误"      # 引用不存在的来源

@dataclass
class HallucinationResult:
    has_hallucination: bool
    type: HallucinationType
    description: str
    source_text: str
    hallucinated_text: str
    confidence: float
```

### 1.2 基于LLM的幻觉检测器

```python
from langchain_openai import ChatOpenAI
import json

class HallucinationDetector:
    """幻觉检测器"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    def detect_with_ground_truth(
        self,
        generated_text: str,
        ground_truth: str
    ) -> HallucinationResult:
        """方法1：基于Ground Truth的检测"""
        
        prompt = f"""你是一个专业的幻觉检测器。请将生成文本与参考文本(ground truth)进行对比。

【参考文本（Ground Truth）】
{ground_truth}

【生成文本】
{generated_text}

请逐句检测生成文本中是否存在幻觉（与参考文本不一致的内容）。

检测要点：
1. 事实性错误：编造不存在的事实
2. 数值错误：数字、日期、金额等与参考不一致
3. 归因错误：将A的事说成B的
4. 逻辑矛盾：生成文本内部自相矛盾

输出JSON：
{{
    "has_hallucination": true/false,
    "hallucinations": [
        {{
            "type": "factual_error/numerical_error/attribution_error/logical_error",
            "source_truth": "参考文本中的原始内容",
            "hallucinated_text": "生成文本中的错误内容",
            "explanation": "详细说明问题"
        }}
    ],
    "overall_score": 0-100  （100表示完全准确）
}}
"""
        
        response = self.llm.invoke(prompt)
        result = json.loads(response.content)
        
        return HallucinationResult(
            has_hallucination=result["has_hallucination"],
            type=result["hallucinations"][0]["type"] if result["hallucinations"] else None,
            description=result["hallucinations"][0]["explanation"] if result["hallucinations"] else "",
            source_text=result["hallucinations"][0]["source_truth"] if result["hallucinations"] else "",
            hallucinated_text=result["hallucinations"][0]["hallucinated_text"] if result["hallucinations"] else "",
            confidence=result["overall_score"] / 100.0
        )
    
    def detect_without_ground_truth(
        self,
        generated_text: str,
        context: str = None
    ) -> HallucinationResult:
        """方法2：无Ground Truth的自我一致性检测"""
        
        # 策略1：多次采样检测一致性
        samples = []
        for _ in range(3):
            prompt = f"基于以下上下文回答问题：\n{context}\n\n请回答。"
            response = self.llm.invoke(prompt)
            samples.append(response.content)
        
        # 策略2：让LLM自我检查
        check_prompt = f"""请仔细检查以下文本是否存在幻觉（与常识或逻辑矛盾的内容）。

文本：{generated_text}

检查要点：
1. 数字、日期是否合理
2. 实体名称是否存在
3. 逻辑是否自洽
4. 是否有无法验证的断言

输出JSON：
{{
    "has_hallucination": true/false,
    "issues": ["问题1", "问题2"],
    "confidence": 0-100
}}
"""
        
        response = self.llm.invoke(check_prompt)
        result = json.loads(response.content)
        
        return HallucinationResult(
            has_hallucination=result["has_hallucination"],
            type=None,
            description="; ".join(result.get("issues", [])),
            source_text="",
            hallucinated_text=generated_text,
            confidence=result.get("confidence", 50) / 100.0
        )
    
    def detect_batch(
        self,
        test_cases: list
    ) -> dict:
        """批量检测并生成报告"""
        results = []
        
        for case in test_cases:
            result = self.detect_with_ground_truth(
                case["generated"],
                case["ground_truth"]
            )
            results.append({
                "case_id": case["id"],
                "has_hallucination": result.has_hallucination,
                "type": result.type.value if result.type else "none",
                "confidence": result.confidence
            })
        
        # 生成统计报告
        total = len(results)
        hallucinated = sum(1 for r in results if r["has_hallucination"])
        
        return {
            "total_cases": total,
            "hallucination_count": hallucinated,
            "hallucination_rate": f"{hallucinated / total * 100:.1f}%",
            "avg_confidence": sum(r["confidence"] for r in results) / total,
            "details": results
        }
```

## 二、A/B测试框架

### 2.1 A/B测试架构

```python
import random
from typing import Callable, Dict
from dataclasses import dataclass, field

@dataclass
class ABTestConfig:
    """A/B测试配置"""
    name: str
    variants: Dict[str, Callable]  # {"variant_a": func_a, "variant_b": func_b}
    metrics: List[str]             # 评估指标
    traffic_split: Dict[str, float] = None  # 流量分配比例
    min_sample_size: int = 100

@dataclass  
class TestResult:
    variant: str
    metrics: Dict[str, float]
    latency_ms: float
    token_usage: int
    user_feedback: int = 0

class ABTestRunner:
    """A/B测试运行器"""
    
    def __init__(self, config: ABTestConfig):
        self.config = config
        self.results = {v: [] for v in config.variants}
        
        if not config.traffic_split:
            # 默认均匀分配
            n = len(config.variants)
            self.config.traffic_split = {v: 1.0/n for v in config.variants}
    
    def route_request(self, user_id: str = None) -> str:
        """根据流量分配路由到对应变体"""
        if user_id:
            # 基于用户ID的确定性路由（同一用户始终看到同一版本）
            variant_list = list(self.config.variants.keys())
            idx = hash(user_id) % len(variant_list)
            return variant_list[idx]
        else:
            # 随机分配
            rand = random.random()
            cumulative = 0
            for variant, weight in self.config.traffic_split.items():
                cumulative += weight
                if rand <= cumulative:
                    return variant
    
    def run_test_case(self, test_input: str, true_output: str = None) -> Dict:
        """对单个测试用例运行所有变体"""
        results = {}
        
        for variant_name, handler in self.config.variants.items():
            # 执行变体
            import time
            start = time.time()
            
            try:
                output = handler(test_input)
            except Exception as e:
                output = f"ERROR: {str(e)}"
            
            latency = (time.time() - start) * 1000  # 转换为ms
            
            # 计算指标
            metrics = {}
            if true_output:
                metrics["exact_match"] = 1.0 if output == true_output else 0.0
                metrics["rouge_l"] = self._compute_rouge_l(output, true_output)
                metrics["semantic_similarity"] = self._compute_semantic_sim(output, true_output)
            
            metrics["latency_ms"] = latency
            metrics["output_length"] = len(output)
            
            results[variant_name] = {
                "output": output,
                "metrics": metrics
            }
        
        return results
    
    def _compute_rouge_l(self, generated: str, reference: str) -> float:
        """计算ROUGE-L分数"""
        from collections import Counter
        
        gen_words = generated.lower().split()
        ref_words = reference.lower().split()
        
        # LCS计算（简化版）
        lcs_length = self._lcs_length(gen_words, ref_words)
        
        if not gen_words or not ref_words:
            return 0.0
        
        recall = lcs_length / len(ref_words)
        precision = lcs_length / len(gen_words)
        
        if recall + precision == 0:
            return 0.0
        
        f1 = 2 * recall * precision / (recall + precision)
        return f1
    
    @staticmethod
    def _lcs_length(a: list, b: list) -> int:
        """最长公共子序列长度"""
        m, n = len(a), len(b)
        dp = [[0] * (n + 1) for _ in range(m + 1)]
        
        for i in range(1, m + 1):
            for j in range(1, n + 1):
                if a[i-1] == b[j-1]:
                    dp[i][j] = dp[i-1][j-1] + 1
                else:
                    dp[i][j] = max(dp[i-1][j], dp[i][j-1])
        
        return dp[m][n]
    
    def _compute_semantic_sim(self, text1: str, text2: str) -> float:
        """计算语义相似度（使用Embedding）"""
        from langchain_openai import OpenAIEmbeddings
        import numpy as np
        
        embeddings = OpenAIEmbeddings()
        emb1 = np.array(embeddings.embed_query(text1))
        emb2 = np.array(embeddings.embed_query(text2))
        
        return float(np.dot(emb1, emb2) / (np.linalg.norm(emb1) * np.linalg.norm(emb2)))
    
    def generate_report(self) -> str:
        """生成A/B测试报告"""
        report = f"# A/B测试报告：{self.config.name}\n\n"
        report += f"测试样本数：{self.config.min_sample_size}\n\n"
        
        report += "## 各变体指标对比\n\n"
        report += "| 指标 | " + " | ".join(self.config.variants.keys()) + " |\n"
        report += "|------|" + "|".join(["------" for _ in self.config.variants]) + "|\n"
        
        for metric in self.config.metrics:
            row = f"| {metric} |"
            for variant in self.config.variants:
                values = [r[metric] for r in self.results[variant] if metric in r]
                avg = sum(values) / len(values) if values else 0
                row += f" {avg:.4f} |"
            report += row + "\n"
        
        return report

# 使用示例
def demo_ab_test():
    """A/B测试示例：比较两种Prompt策略"""
    
    def prompt_strategy_a(request: str) -> str:
        """策略A：简洁Prompt"""
        llm = ChatOpenAI(model="gpt-4o-mini")
        return llm.invoke(f"回答问题：{request}").content
    
    def prompt_strategy_b(request: str) -> str:
        """策略B：详细CoT Prompt"""
        llm = ChatOpenAI(model="gpt-4o-mini")
        prompt = f"""请仔细阅读问题，然后分步骤思考和回答。

问题：{request}

请按以下格式回答：
思考：<逐步推理>
答案：<最终答案>
"""
        return llm.invoke(prompt).content
    
    config = ABTestConfig(
        name="Prompt策略对比",
        variants={
            "简洁Prompt": prompt_strategy_a,
            "CoT Prompt": prompt_strategy_b
        },
        metrics=["latency_ms", "output_length"],
        min_sample_size=50
    )
    
    runner = ABTestRunner(config)
    
    # 运行测试
    test_cases = [
        "Python中如何实现单例模式？",
        "解释RESTful API的设计原则"
    ]
    
    for case in test_cases:
        results = runner.run_test_case(case)
        print(json.dumps(results, indent=2, ensure_ascii=False))
    
    # 生成报告
    print(runner.generate_report())

if __name__ == "__main__":
    demo_ab_test()
```

### 2.2 离线评估：预定义测试集

```python
class OfflineEvaluator:
    """离线评估器——使用预定义测试集"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    def evaluate_with_golden_dataset(
        self,
        agent_func: Callable,
        test_cases: List[dict]
    ) -> dict:
        """
        使用金标准数据集评估Agent
        
        test_cases格式：
        [
            {
                "id": "case_001",
                "input": "用户问题",
                "expected_output": "期望回答",
                "expected_tool_calls": ["tool_name"],
                "expected_facts": ["事实1", "事实2"]
            }
        ]
        """
        results = []
        
        for case in test_cases:
            try:
                output = agent_func(case["input"])
            except Exception as e:
                output = f"ERROR: {str(e)}"
            
            # 多维度打分
            scores = {}
            
            # 1. 事实准确性
            fact_score = self._check_facts(output, case.get("expected_facts", []))
            scores["fact_accuracy"] = fact_score
            
            # 2. 回答完整性
            completeness = self._check_completeness(
                output, case["expected_output"]
            )
            scores["completeness"] = completeness
            
            # 3. 格式合规性
            format_score = self._check_format(output)
            scores["format"] = format_score
            
            # 综合得分
            overall = (fact_score * 0.5 + completeness * 0.3 + format_score * 0.2)
            
            results.append({
                "case_id": case["id"],
                "scores": scores,
                "overall": overall,
                "output": output[:200]
            })
        
        # 汇总统计
        overall_scores = [r["overall"] for r in results]
        
        return {
            "total_cases": len(test_cases),
            "avg_score": sum(overall_scores) / len(overall_scores),
            "pass_rate": sum(1 for s in overall_scores if s >= 0.7) / len(overall_scores),
            "details": results
        }
    
    def _check_facts(self, output: str, expected_facts: list) -> float:
        """检查输出是否包含预期事实"""
        if not expected_facts:
            return 1.0
        
        matched = 0
        for fact in expected_facts:
            if fact.lower() in output.lower():
                matched += 1
        
        return matched / len(expected_facts)
    
    def _check_completeness(self, output: str, expected: str) -> float:
        """检查回答完整性"""
        # 使用语义相似度
        return self._compute_semantic_sim(output, expected)
    
    def _check_format(self, output: str) -> float:
        """检查格式质量"""
        score = 1.0
        
        # 检查长度（太短可能不完整）
        if len(output) < 50:
            score -= 0.3
        
        # 检查是否有明显错误标志
        error_keywords = ["error", "错误", "I don't know", "无法回答"]
        for kw in error_keywords:
            if kw.lower() in output.lower():
                score -= 0.2
                break
        
        return max(0, score)
```

## 面试题

### 面试题1：请说明LLM幻觉的主要类型，以及如何构建一个有效的幻觉检测系统。

**答案：**

**主要幻觉类型：**

| 类型 | 说明 | 示例 |
|------|------|------|
| 事实捏造 | 编造不存在的事实 | "Python 4.0于2025年发布"（实际不存在） |
| 实体混淆 | 将属性/事件错误归因 | "Google发布了ChatGPT"（实际是OpenAI） |
| 数值错误 | 数字计算或日期错误 | "100*50=50000"（应该是5000） |
| 逻辑矛盾 | 前后文不一致 | "A比B大...B比A大" |
| 来源伪造 | 引用不存在的论文/URL | "根据Smith 2023论文..."（不存在） |

**幻觉检测系统架构：**

```python
# 三层检测架构
class MultiLayerDetector:
    """多层幻觉检测"""
    
    def layer1_rule_based(self, text: str) -> list:
        """第1层：规则检测（快速）"""
        rules = [
            ("日期检查", r"\d{4}年\d{1,2}月\d{1,2}日"),
            ("引用检查", r"根据.*\d{4}"),
            ("数值检查", r"\d+[\+\-\*/]\d+=\d+")
        ]
        issues = []
        for name, pattern in rules:
            matches = re.findall(pattern, text)
            for m in matches:
                # 快速规则验证
                if not self._validate(m):
                    issues.append(f"[{name}] 可疑：{m}")
        return issues
    
    def layer2_llm_based(self, text: str) -> list:
        """第2层：LLM检测（中速）"""
        # 使用专门训练的幻觉检测Prompt
        pass
    
    def layer3_human_review(self, text: str) -> bool:
        """第3层：人工抽检（慢速但最准）"""
        # 抽样10%进行人工审核
        pass
```

**技术指标：**

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 检测召回率 | >90% | 能发现90%以上的幻觉 |
| 检测精确率 | >80% | 误报率低于20% |
| 检测延迟 | <500ms | 不影响用户体验 |

### 面试题2：设计一个AI Agent的A/B测试方案，包括流量分配、指标选择、统计显著性判断。

**答案：**

```python
class ProductionABTest:
    """生产级A/B测试方案"""
    
    def __init__(self):
        self.config = {
            "variants": {
                "control": {"weight": 0.9},    # 90%流量（稳定版）
                "experiment": {"weight": 0.1}  # 10%流量（实验版）
            },
            "metrics": {
                "primary": ["task_success_rate", "user_satisfaction"],
                "secondary": ["latency_p50", "latency_p99", "token_cost"]
            },
            "statistical": {
                "confidence_level": 0.95,
                "min_sample_size": 384  # 效应量0.2, 功效0.8
            }
        }
    
    def calculate_sample_size(self, baseline_rate: float, 
                              expected_lift: float) -> int:
        """计算所需样本量"""
        from scipy import stats
        
        # 使用比例检验的样本量公式
        z_alpha = stats.norm.ppf(0.975)  # α=0.05 (双尾)
        z_beta = stats.norm.ppf(0.80)    # β=0.20 (80%功效)
        
        p1 = baseline_rate
        p2 = baseline_rate * (1 + expected_lift)
        p_pooled = (p1 + p2) / 2
        
        n = ((z_alpha * (2*p_pooled*(1-p_pooled))**0.5 + 
              z_beta * (p1*(1-p1) + p2*(1-p2))**0.5)**2) / (p2 - p1)**2
        
        return int(n)
    
    def check_significance(self, control_results: list, 
                          experiment_results: list) -> dict:
        """检查统计显著性"""
        from scipy import stats
        
        control_rate = sum(control_results) / len(control_results)
        experiment_rate = sum(experiment_results) / len(experiment_results)
        
        # Z检验
        n1, n2 = len(control_results), len(experiment_results)
        p_pooled = (sum(control_results) + sum(experiment_results)) / (n1 + n2)
        
        se = (p_pooled * (1 - p_pooled) * (1/n1 + 1/n2)) ** 0.5
        z_score = (experiment_rate - control_rate) / se
        p_value = 2 * (1 - stats.norm.cdf(abs(z_score)))
        
        return {
            "control_rate": control_rate,
            "experiment_rate": experiment_rate,
            "lift": f"{(experiment_rate - control_rate) / control_rate * 100:+.1f}%",
            "p_value": p_value,
            "significant": p_value < 0.05,
            "confidence_interval": (
                experiment_rate - control_rate - 1.96*se,
                experiment_rate - control_rate + 1.96*se
            )
        }
```

**流量分配策略：**

| 阶段 | 控制组 | 实验组 | 持续时间 |
|------|--------|--------|---------|
| 灰度验证 | 95% | 5% | 1-3天 |
| 扩大测试 | 80% | 20% | 3-7天 |
| 全量观察 | 50% | 50% | 7-14天 |

## 总结

1. **幻觉检测**是LLM应用质量保障的基石，建议从Ground Truth检测开始，逐步引入多层次检测体系
2. **A/B测试**是科学迭代的方法论，流量分配、指标选择、统计显著性缺一不可
3. 建议将两者结合：A/B测试时自动检测幻觉率，作为核心评估指标

## 参考资料

- [HaluEval: A Large-Scale Hallucination Evaluation Benchmark](https://arxiv.org/abs/2305.11747)
- [A/B Testing Guide](https://www.evanmiller.org/ab-testing/)
- [LangSmith Evaluation](https://docs.smith.langchain.com/evaluation)
