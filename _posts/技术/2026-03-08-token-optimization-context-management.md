---
layout: post
title: "Token优化：上下文窗口管理与成本控制"
category: 技术
tags: [Token优化, 上下文管理, LLM成本, Prompt压缩, 缓存策略]
keywords: [Token Optimization, Context Management, LLM Cost, Prompt Compression, Caching]
description: "系统讲解LLM应用中的Token优化策略，包括上下文窗口管理、Prompt压缩、缓存策略和成本控制方案。"
date: 2026-03-08
---

# Token优化：上下文窗口管理与成本控制

## 引言

LLM API按Token计费，上下文窗口有限。一个生产级Agent每天处理数万次请求，Token成本可能成为最大开支。本文系统讲解Token优化策略，帮助你在保证质量的前提下大幅降低成本。

## 一、Token消耗分析

### 1.1 Token消耗来源

```python
class TokenAnalyzer:
    """Token消耗分析器"""
    
    def __init__(self):
        from langchain_openai import OpenAIEmbeddings
        self.encoder = tiktoken.encoding_for_model("gpt-4o")
    
    def analyze_message_history(self, messages: list) -> dict:
        """分析对话历史的Token消耗"""
        analysis = {
            "total_tokens": 0,
            "by_role": {},
            "by_position": [],
            "largest_messages": []
        }
        
        for i, msg in enumerate(messages):
            role = msg.get("role", "unknown")
            content = msg.get("content", "")
            
            tokens = len(self.encoder.encode(content))
            
            analysis["total_tokens"] += tokens
            analysis["by_role"][role] = analysis["by_role"].get(role, 0) + tokens
            analysis["by_position"].append({
                "index": i,
                "role": role,
                "tokens": tokens,
                "preview": content[:50]
            })
        
        # 找出Token消耗最大的消息
        analysis["by_position"].sort(key=lambda x: x["tokens"], reverse=True)
        analysis["largest_messages"] = analysis["by_position"][:5]
        
        return analysis
    
    def estimate_cost(self, token_count: int, model: str = "gpt-4o") -> dict:
        """估算成本"""
        pricing = {
            "gpt-4o": {"input": 0.0025, "output": 0.01},  # 每1K tokens
            "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
            "claude-3.5-sonnet": {"input": 0.003, "output": 0.015},
            "gemini-1.5-pro": {"input": 0.00125, "output": 0.005}
        }
        
        model_pricing = pricing.get(model, pricing["gpt-4o"])
        
        # 假设输入输出各占50%
        input_tokens = token_count * 0.5
        output_tokens = token_count * 0.5
        
        cost = (input_tokens / 1000 * model_pricing["input"] + 
                output_tokens / 1000 * model_pricing["output"])
        
        return {
            "model": model,
            "total_tokens": token_count,
            "estimated_cost_usd": cost,
            "estimated_cost_rmb": cost * 7.2
        }

# 使用示例
analyzer = TokenAnalyzer()

messages = [
    {"role": "system", "content": "你是一个AI助手..." * 100},
    {"role": "user", "content": "帮我分析这份文档..." * 50},
    {"role": "assistant", "content": "好的，我来帮你分析..." * 200},
    # ... 更多历史消息
]

analysis = analyzer.analyze_message_history(messages)
print(f"总Token数：{analysis['total_tokens']}")
print(f"估算成本：{analyzer.estimate_cost(analysis['total_tokens'])}")
```

### 1.2 上下文窗口限制

| 模型 | 上下文窗口 | 建议保留Token | 最大输出 |
|------|-----------|--------------|---------|
| GPT-4o | 128K | ~100K | 16K |
| GPT-4o-mini | 128K | ~100K | 16K |
| Claude 3.5 Sonnet | 200K | ~180K | 8K |
| Gemini 1.5 Pro | 2M | ~1.8M | 8K |

## 二、上下文窗口管理策略

### 2.1 滑动窗口策略

```python
class SlidingWindowManager:
    """滑动窗口上下文管理器"""
    
    def __init__(self, max_tokens: int = 100000):
        self.max_tokens = max_tokens
        self.encoder = tiktoken.encoding_for_model("gpt-4o")
    
    def truncate_messages(
        self, 
        messages: list, 
        preserve_recent: int = 10
    ) -> list:
        """
        滑动窗口截断
        
        Args:
            messages: 完整消息历史
            preserve_recent: 保留最近N条消息
        """
        if not messages:
            return messages
        
        # 计算总Token
        total_tokens = sum(
            len(self.encoder.encode(m.get("content", ""))) 
            for m in messages
        )
        
        if total_tokens <= self.max_tokens:
            return messages
        
        # 策略：保留系统消息 + 最近N条
        system_msgs = [m for m in messages if m.get("role") == "system"]
        non_system = [m for m in messages if m.get("role") != "system"]
        
        # 保留最近N条
        recent = non_system[-preserve_recent:] if non_system else []
        
        # 尝试保留更多历史
        truncated = system_msgs + recent
        
        # 如果还是超，从最旧的开始删
        while self._count_tokens(truncated) > self.max_tokens and len(truncated) > 1:
            # 删除最旧的非系统消息
            for i, msg in enumerate(truncated):
                if msg.get("role") != "system":
                    truncated.pop(i)
                    break
        
        return truncated
    
    def _count_tokens(self, messages: list) -> int:
        return sum(
            len(self.encoder.encode(m.get("content", ""))) 
            for m in messages
        )
```

### 2.2 智能摘要压缩

```python
class ContextCompressor:
    """上下文压缩器"""
    
    def __init__(self, llm):
        self.llm = llm
        self.encoder = tiktoken.encoding_for_model("gpt-4o")
    
    def compress_history(
        self, 
        messages: list, 
        max_tokens: int = 4000
    ) -> list:
        """
        压缩历史对话
        
        策略：
        1. 保留最近N轮完整对话
        2. 对更早的对话生成摘要
        """
        if len(messages) <= 6:  # 少于3轮对话不压缩
            return messages
        
        # 分离：保留最近3轮 + 压缩更早的
        recent = messages[-6:]  # 最近3轮（每轮2条）
        to_compress = messages[:-6]
        
        if not to_compress:
            return messages
        
        # 生成摘要
        history_text = "\n".join(
            f"{m['role']}: {m['content']}" 
            for m in to_compress
        )
        
        summary_prompt = f"""请压缩以下对话历史，保留关键信息：

{history_text}

要求：
1. 保留用户的关键需求和问题
2. 保留AI的重要回答和决策
3. 去除冗余和重复信息
4. 输出为简洁的摘要（200字以内）

输出格式：
[历史摘要] <压缩后的内容>
"""
        
        summary_response = self.llm.invoke(summary_prompt)
        summary = summary_response.content
        
        # 构建压缩后的消息
        compressed = [
            {
                "role": "system", 
                "content": f"历史对话摘要：\n{summary}"
            }
        ] + recent
        
        return compressed
    
    def selective_compress(
        self, 
        messages: list, 
        importance_scores: list = None
    ) -> list:
        """
        选择性压缩：根据重要性分数保留/压缩
        
        importance_scores: 每条消息的重要性分数（0-1）
        """
        if importance_scores is None:
            # 默认：系统消息和最近消息更重要
            importance_scores = [
                1.0 if m.get("role") == "system" else
                0.8 if i >= len(messages) - 4 else
                0.3
                for i, m in enumerate(messages)
            ]
        
        # 高重要性：保留原样
        # 中重要性：简短摘要
        # 低重要性：删除
        
        result = []
        for msg, score in zip(messages, importance_scores):
            if score >= 0.7:
                result.append(msg)
            elif score >= 0.4:
                # 生成简短版本
                short_version = self._summarize_message(msg)
                result.append({
                    "role": msg["role"],
                    "content": f"[摘要] {short_version}"
                })
            # score < 0.4: 删除
        
        return result
    
    def _summarize_message(self, message: dict) -> str:
        """生成单条消息的摘要"""
        content = message.get("content", "")
        
        if len(content) <= 100:
            return content
        
        prompt = f"请用20字以内摘要以下内容：\n{content}"
        response = self.llm.invoke(prompt)
        return response.content
```

### 2.3 向量检索压缩（RAG场景）

```python
class RAGContextOptimizer:
    """RAG场景的上下文优化"""
    
    def __init__(self):
        self.encoder = tiktoken.encoding_for_model("gpt-4o")
    
    def optimize_retrieved_docs(
        self, 
        query: str, 
        documents: list, 
        max_tokens: int = 8000
    ) -> list:
        """
        优化检索到的文档
        
        策略：
        1. 按相关性排序
        2. 去重（相似度>阈值）
        3. 截断超长文档
        4. 优先保留最相关的
        """
        from sklearn.metrics.pairwise import cosine_similarity
        from langchain_openai import OpenAIEmbeddings
        
        embeddings = OpenAIEmbeddings()
        
        # 1. 按相关性排序（假设已有score）
        sorted_docs = sorted(
            documents, 
            key=lambda x: x.get("score", 0), 
            reverse=True
        )
        
        # 2. 去重
        unique_docs = self._deduplicate_docs(sorted_docs)
        
        # 3. 在Token预算内选择
        selected = []
        used_tokens = 0
        
        for doc in unique_docs:
            doc_tokens = len(self.encoder.encode(doc["content"]))
            
            if used_tokens + doc_tokens > max_tokens:
                # 尝试截断
                max_chars = (max_tokens - used_tokens) * 3  # 粗略估算
                if max_chars > 100:
                    doc["content"] = doc["content"][:int(max_chars)] + "..."
                    selected.append(doc)
                break
            
            selected.append(doc)
            used_tokens += doc_tokens
        
        return selected
    
    def _deduplicate_docs(self, documents: list, threshold: float = 0.85) -> list:
        """去重相似文档"""
        from langchain_openai import OpenAIEmbeddings
        import numpy as np
        
        embeddings = OpenAIEmbeddings()
        
        # 计算所有文档的embedding
        doc_texts = [d["content"][:500] for d in documents]  # 取前500字符
        doc_embeddings = embeddings.embed_documents(doc_texts)
        
        # 去重
        keep = [True] * len(documents)
        
        for i in range(len(documents)):
            if not keep[i]:
                continue
            for j in range(i + 1, len(documents)):
                if not keep[j]:
                    continue
                
                sim = cosine_similarity(
                    [doc_embeddings[i]], 
                    [doc_embeddings[j]]
                )[0][0]
                
                if sim > threshold:
                    keep[j] = False
        
        return [d for d, k in zip(documents, keep) if k]
```

## 三、Prompt压缩技术

### 3.1 自动Prompt压缩

```python
class PromptCompressor:
    """自动Prompt压缩"""
    
    def __init__(self, llm):
        self.llm = llm
    
    def compress_prompt(
        self, 
        original_prompt: str, 
        compression_ratio: float = 0.5
    ) -> str:
        """
        压缩Prompt
        
        Args:
            compression_ratio: 目标压缩比例
        """
        prompt = f"""请压缩以下Prompt，在保持核心指令不变的前提下，去除冗余表达。

原始Prompt：
{original_prompt}

要求：
1. 保留所有关键指令和约束
2. 去除重复的说明
3. 简化示例（保留1个即可）
4. 目标长度：原始长度的{int(compression_ratio * 100)}%以内

直接输出压缩后的Prompt，不要解释。
"""
        
        response = self.llm.invoke(prompt)
        return response.content
    
    def compress_few_shot_examples(
        self, 
        examples: list, 
        max_examples: int = 3
    ) -> list:
        """
        压缩Few-shot示例
        
        策略：
        1. 保留最具代表性的示例
        2. 去除相似示例
        3. 简化示例输出
        """
        if len(examples) <= max_examples:
            return examples
        
        # 使用LLM选择最多样化的示例
        examples_text = "\n\n".join(
            f"示例{i+1}：\n输入：{e['input']}\n输出：{e['output']}"
            for i, e in enumerate(examples)
        )
        
        select_prompt = f"""从以下示例中选择{max_examples}个最具代表性的（覆盖不同场景）：

{examples_text}

请返回选中的示例编号（如：1, 3, 5），并说明选择理由。
"""
        
        response = self.llm.invoke(select_prompt)
        
        # 解析选中的编号
        import re
        numbers = re.findall(r'\d+', response.content)
        selected_indices = [int(n) - 1 for n in numbers[:max_examples]]
        
        return [examples[i] for i in selected_indices if i < len(examples)]
```

### 3.2 缓存策略

```python
from functools import lru_cache
import hashlib
import json

class PromptCache:
    """Prompt缓存——避免重复计算"""
    
    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size
    
    def get_cache_key(self, prompt: str, **kwargs) -> str:
        """生成缓存Key"""
        cache_data = {
            "prompt": prompt,
            "kwargs": sorted(kwargs.items()) if kwargs else {}
        }
        return hashlib.md5(
            json.dumps(cache_data, ensure_ascii=False).encode()
        ).hexdigest()
    
    def get(self, prompt: str, **kwargs) -> str:
        """获取缓存结果"""
        key = self.get_cache_key(prompt, **kwargs)
        return self.cache.get(key)
    
    def set(self, prompt: str, result: str, **kwargs):
        """设置缓存"""
        if len(self.cache) >= self.max_size:
            # LRU淘汰：删除最早的一个
            oldest_key = next(iter(self.cache))
            del self.cache[oldest_key]
        
        key = self.get_cache_key(prompt, **kwargs)
        self.cache[key] = result
    
    def cached_llm_call(self, llm, prompt: str, **kwargs) -> str:
        """带缓存的LLM调用"""
        cached = self.get(prompt, **kwargs)
        if cached:
            print("Cache hit!")
            return cached
        
        result = llm.invoke(prompt, **kwargs).content
        
        self.set(prompt, result, **kwargs)
        return result


# 使用LLM API自带的缓存（推荐）
"""
OpenAI支持Prompt Caching（自动）：
- 相同前缀的Prompt会自动缓存
- 缓存命中时费用降低90%
- 缓存TTL：5-10分钟

最佳实践：
1. 将不变的System Prompt放在最前面
2. 变化的部分（用户输入）放在最后
3. 避免在每个请求中都修改System Prompt

示例：
messages = [
    {"role": "system", "content": VERY_LONG_SYSTEM_PROMPT},  # 会被缓存
    {"role": "user", "content": user_input}  # 每次变化
]
"""
```

## 四、成本控制方案

### 4.1 模型路由策略

```python
class ModelRouter:
    """智能模型路由——根据任务复杂度选择模型"""
    
    def __init__(self):
        self.complexity_thresholds = {
            "simple": 0.3,      # 简单任务 → 用最便宜的模型
            "medium": 0.7,      # 中等任务 → 用中等模型
            "complex": 1.0       # 复杂任务 → 用最强模型
        }
        
        self.model_pricing = {
            "gpt-4o-mini": 0.00015,  # $/1K input tokens
            "gpt-4o": 0.0025,
            "claude-3.5-haiku": 0.00025,
            "claude-3.5-sonnet": 0.003
        }
    
    def route(self, task: str, estimated_tokens: int) -> str:
        """
        路由到合适的模型
        
        Returns:
            模型名称
        """
        complexity = self._estimate_complexity(task)
        
        if complexity < self.complexity_thresholds["simple"]:
            return "gpt-4o-mini"  # 最便宜
        elif complexity < self.complexity_thresholds["medium"]:
            return "claude-3.5-haiku"  # 中等
        else:
            return "gpt-4o"  # 最强
    
    def _estimate_complexity(self, task: str) -> float:
        """估算任务复杂度（0-1）"""
        indicators = {
            "high": ["分析", "推理", "规划", "创作", "设计", "优化"],
            "medium": ["总结", "翻译", "提取", "分类", "改写"],
            "low": ["回答", "查询", "简单计算", "格式转换"]
        }
        
        task_lower = task.lower()
        
        for indicator in indicators["high"]:
            if indicator in task_lower:
                return 0.8
        
        for indicator in indicators["medium"]:
            if indicator in task_lower:
                return 0.5
        
        return 0.2
    
    def estimate_savings(
        self, 
        tasks: list, 
        default_model: str = "gpt-4o"
    ) -> dict:
        """估算使用路由策略的节省金额"""
        total_default_cost = 0
        total_routed_cost = 0
        
        for task in tasks:
            tokens = len(task.split()) * 1.3  # 粗略估算
            
            # 默认模型成本
            default_cost = tokens / 1000 * self.model_pricing[default_model]
            total_default_cost += default_cost
            
            # 路由后成本
            routed_model = self.route(task, int(tokens))
            routed_cost = tokens / 1000 * self.model_pricing[routed_model]
            total_routed_cost += routed_cost
        
        savings = total_default_cost - total_routed_cost
        savings_rate = savings / total_default_cost * 100 if total_default_cost > 0 else 0
        
        return {
            "default_cost_usd": total_default_cost,
            "routed_cost_usd": total_routed_cost,
            "savings_usd": savings,
            "savings_rate": f"{savings_rate:.1f}%"
        }
```

### 4.2 Token预算控制

```python
class TokenBudgetManager:
    """Token预算管理器"""
    
    def __init__(self, daily_budget_usd: float = 10.0):
        self.daily_budget_usd = daily_budget_usd
        self.used_today = 0.0
        self.request_count = 0
    
    def check_budget(self, estimated_tokens: int, model: str) -> bool:
        """检查是否在预算内"""
        estimated_cost = self._estimate_cost(estimated_tokens, model)
        
        if self.used_today + estimated_cost > self.daily_budget_usd:
            return False
        
        return True
    
    def record_usage(self, tokens: int, model: str):
        """记录用量"""
        cost = self._estimate_cost(tokens, model)
        self.used_today += cost
        self.request_count += 1
    
    def _estimate_cost(self, tokens: int, model: str) -> float:
        """估算成本（USD）"""
        pricing = {
            "gpt-4o": {"input": 0.0025, "output": 0.01},
            "gpt-4o-mini": {"input": 0.00015, "output": 0.0006}
        }
        
        model_pricing = pricing.get(model, pricing["gpt-4o"])
        input_cost = tokens * 0.5 / 1000 * model_pricing["input"]
        output_cost = tokens * 0.5 / 1000 * model_pricing["output"]
        
        return input_cost + output_cost
    
    def get_usage_report(self) -> dict:
        """获取用量报告"""
        return {
            "date": datetime.now().strftime("%Y-%m-%d"),
            "budget_usd": self.daily_budget_usd,
            "used_usd": self.used_today,
            "remaining_usd": self.daily_budget_usd - self.used_today,
            "usage_rate": f"{self.used_today / self.daily_budget_usd * 100:.1f}%",
            "request_count": self.request_count,
            "avg_cost_per_request": self.used_today / max(self.request_count, 1)
        }
```

## 面试题

### 面试题1：在生产环境中，如何降低LLM API的成本？请给出至少5种策略。

**答案：**

| 策略 | 节省比例 | 实现难度 | 质量影响 |
|------|---------|---------|---------|
| 1. 使用更便宜的模型 | 80-90% | 低 | 中等 |
| 2. Prompt缓存 | 50-90% | 低 | 无 |
| 3. 上下文压缩 | 30-50% | 中等 | 低 |
| 4. 批量请求 | 20-30% | 中等 | 无 |
| 5. Few-shot → Zero-shot | 10-20% | 低 | 中等 |
| 6. 模型路由 | 40-60% | 高 | 低 |

**代码示例：模型路由**

```python
class CostOptimizer:
    """成本优化器"""
    
    def __init__(self):
        self.cache = PromptCache()
    
    def optimize(self, prompt: str, complexity: str = "auto") -> str:
        """优化单次请求"""
        
        # 1. 检查缓存
        cached = self.cache.get(prompt)
        if cached:
            return cached
        
        # 2. 选择模型
        model = self._select_model(complexity)
        
        # 3. 压缩Prompt
        compressed = self._compress_if_needed(prompt, model)
        
        # 4. 发起请求
        result = model.invoke(compressed)
        
        # 5. 缓存结果
        self.cache.set(prompt, result)
        
        return result
    
    def _select_model(self, complexity: str):
        if complexity == "simple":
            return ChatOpenAI(model="gpt-4o-mini")
        elif complexity == "medium":
            return ChatOpenAI(model="gpt-4o-mini")
        else:
            return ChatOpenAI(model="gpt-4o")
```

**真实案例：**

某客服机器人优化前后对比：
- 优化前：GPT-4o，平均每次请求5000 tokens，日成本$200
- 优化后：
  - 简单问题 → GPT-4o-mini（节省80%）
  - 启用Prompt缓存（节省50%）
  - 上下文压缩（节省30%）
  - 日成本降至$60（节省70%）

### 面试题2：请解释LLM的上下文窗口限制，以及如何有效管理长对话历史。

**答案：**

**上下文窗口限制：**

```python
# 问题：上下文窗口有限
"""
GPT-4o: 128K tokens ≈ 96,000个汉字
假设每次对话平均500 tokens：
- 可以支持约192轮对话

但实际中：
- System Prompt: 1000 tokens
- 每轮对话: 500 tokens (用户输入+AI回答)
- 可用轮次: (128K - 1000) / 500 ≈ 254轮

超过后需要截断或压缩！
"""
```

**管理策略对比：**

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 滑动窗口 | 实现简单 | 丢失早期信息 | 短对话 |
| 摘要压缩 | 保留关键信息 | 有压缩损失 | 长对话 |
| 向量检索 | 精准召回 | 实现复杂 | 有文档的对话 |
| 分层存储 | 支持超长对话 | 系统复杂 | 企业级应用 |

**推荐方案：混合策略**

```python
class HybridContextManager:
    """混合上下文管理"""
    
    def __init__(self, max_tokens: int = 100000):
        self.max_tokens = max_tokens
        self.summary = ""  # 历史摘要
    
    def manage(self, messages: list) -> list:
        """管理上下文"""
        total_tokens = self._count_tokens(messages)
        
        if total_tokens <= self.max_tokens:
            return messages
        
        # 策略1：保留系统消息
        system_msgs = [m for m in messages if m["role"] == "system"]
        
        # 策略2：保留最近N轮
        recent = messages[-6:]
        
        # 策略3：压缩中间部分
        middle = messages[len(system_msgs):-6]
        if middle and not self.summary:
            self.summary = self._compress_to_summary(middle)
        
        # 构建最终上下文
        result = system_msgs
        if self.summary:
            result.append({
                "role": "system",
                "content": f"历史对话摘要：{self.summary}"
            })
        result.extend(recent)
        
        return result
```

## 总结

Token优化是LLM应用降本增效的关键：

1. **上下文管理**：滑动窗口 + 智能摘要 + 向量检索
2. **Prompt压缩**：自动压缩 + Few-shot优化 + 缓存策略
3. **成本控制**：模型路由 + 预算管理 + 批量请求

建议从启用缓存和模型路由开始，逐步引入更精细的优化策略。

## 参考资料

- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
- [LangChain Token Counting](https://python.langchain.com/docs/modules/model_io/token_counting/)
- [LLM Cost Optimization Patterns](https://www.anthropic.com/news/prompt-caching)
