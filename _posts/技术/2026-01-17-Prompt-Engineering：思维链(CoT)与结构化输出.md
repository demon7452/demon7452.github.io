---
layout: post
title: "Prompt Engineering：思维链(CoT)与结构化输出"
category: 技术
tags: [Prompt Engineering, CoT, 结构化输出, Function Calling, AI Agent]
keywords: Prompt Engineering, 思维链, CoT, 结构化输出, Function Calling, JSON Mode
description: 系统梳理Prompt Engineering中的两大核心技法——思维链推理与结构化输出控制，结合Python代码演示如何在实际Agent开发中落地，涵盖Few-shot CoT、约束解码与Pydantic输出校验。
---

## 一、核心概念

Prompt Engineering 不是写几句提示词那么简单。在生产级 Agent 系统中，它包含两个正交维度：

**思维链（Chain of Thought, CoT）**：控制 LLM 的推理过程——"怎么想"。通过要求模型逐步推理而非直接输出答案，大幅提升复杂推理任务的准确率。

**结构化输出**：控制 LLM 的输出格式——"怎么说"。Function Calling、JSON Mode、Constrained Decoding 都是结构化输出的不同实现路径。

二者结合才能构建可控、可靠的 Agent：CoT 保证推理质量，结构化输出保证下游系统能消费 LLM 结果。

## 二、实战示例

### 2.1 思维链（CoT）实现

最基础的 CoT 通过在 Prompt 中加入"让我们一步步思考"触发，但在生产环境中，**Few-shot CoT** 更可控：

```python
analysis_prompt = """你是一个代码审查专家。分析以下代码并给出 review。

请按以下步骤逐步推理：
1. 代码意图：这段代码要解决什么问题？
2. 逻辑缺陷：是否存在边界条件遗漏或逻辑错误？
3. 安全隐患：是否存在 SQL 注入、XSS 等安全风险？
4. 性能问题：是否存在不必要的循环或 I/O 操作？
5. 综合评分：1-10 分，并给出修改建议。

代码：
def get_user(user_id):
    query = "SELECT * FROM users WHERE id = " + user_id
    return db.execute(query)
"""

# 调用 LLM（伪代码，替换为实际 API）
# response = llm.invoke(analysis_prompt)
```

**CoT 的两个关键变体**：

- **Zero-shot CoT**：仅加"Let's think step by step"，在算术推理上准确率即可提升 20%+
- **Few-shot CoT**：提供完整的推理示例（问答对 + 推理过程），精度提升更显著
- **Self-Consistency**：对同一问题采样多条 CoT 路径，取多数投票结果，进一步减少推理偏差

### 2.2 结构化输出实战

使用 Instructor 库（基于 Pydantic v2）实现类型安全的 LLM 输出：

```python
from pydantic import BaseModel, Field
from typing import List, Optional
import instructor
from openai import OpenAI

# ---------- 定义输出 Schema ----------
class BugReport(BaseModel):
    severity: str = Field(
        description="严重程度：critical / high / medium / low"
    )
    line_number: Optional[int] = Field(
        description="问题所在行号"
    )
    category: str = Field(
        description="分类：logic / security / performance / style"
    )
    description: str = Field(
        description="问题描述，50字以内"
    )
    suggestion: str = Field(
        description="修复建议"
    )

class CodeReviewOutput(BaseModel):
    summary: str = Field(description="一句话总结代码质量")
    score: int = Field(ge=1, le=10, description="综合评分 1-10")
    bugs: List[BugReport] = Field(description="发现的问题列表")

# ---------- 使用 Instructor 调用 ----------
client = instructor.from_openai(OpenAI())

review = client.chat.completions.create(
    model="gpt-4o",
    response_model=CodeReviewOutput,
    messages=[
        {"role": "system", "content": "你是代码审查专家。严格按指定格式输出。"},
        {"role": "user", "content": analysis_prompt}
    ],
    max_tokens=2000
)

print(f"评分: {review.score}/10")
print(f"总结: {review.summary}")
for bug in review.bugs:
    print(f"  [{bug.severity}] {bug.category}: {bug.description}")
```

**核心机制**：Instructor 通过重试 + 校验实现结构化输出的可靠性。当 LLM 输出不符合 Pydantic Schema 时，Instructor 自动将校验错误作为新消息追加到对话历史中，请求 LLM 重新生成，直到通过校验或达到最大重试次数。这个过程被称为 **Re-ask** 模式。

### 2.3 Function Calling 作为结构化输出的补充

当输出类型不固定（如 Agent 需要动态选择返回体结构）时，Function Calling 是更灵活的选择。通过定义多个 `tool` schema，LLM 可以自主决定返回哪个结构：

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "report_bug",
            "description": "报告一个代码缺陷",
            "parameters": BugReport.model_json_schema()
        }
    },
    {
        "type": "function",
        "function": {
            "name": "report_safe",
            "description": "代码无问题时的响应",
            "parameters": {
                "type": "object",
                "properties": {
                    "message": {"type": "string"}
                }
            }
        }
    }
]
```

## 三、最佳实践

1. **Prompt 模板化管理**：将所有 Prompt 抽离到 `.yaml` 文件中，用 `jinja2` 渲染变量。这样 Prompt 的迭代不再依赖代码变更，产品经理和领域专家也可以参与调试。

2. **输出 Schema 即契约**：用 Pydantic 定义输出结构后，自动生成 JSON Schema 嵌入 Prompt 的 system message。让下游消费者（前端、微服务）与 Prompt 在编译期就建立类型约束。

3. **CoT 长度权衡**：Few-shot CoT 的示例推理步骤不宜过长。研究表明，2-3 个步骤的 CoT 示例效果最佳，过多步骤反而让模型"过度推理"引入噪音。每个示例控制在 150 字以内。

4. **约束解码 > 后处理**：结构化输出的首选方案是 LLM 服务端原生支持的约束解码（如 OpenAI JSON Mode、vLLM guided decoding），其次用 Instructor/Outlines 做客户端重试，最后才考虑正则后处理。

5. **CoT + ReAct 融合**：在 Agent 工具调用场景中，把 CoT 嵌入 ReAct 的"思考"步骤。先让模型思考需要什么信息、调用什么工具、预期什么结果，然后再行动。这种"Think-Act-Observe"三步循环是 LangChain/LangGraph ReAct Agent 的默认模式。

---

## 面试题

### 题1：Zero-shot CoT 和 Few-shot CoT 的区别是什么？分别在什么场景下更适合使用？

**答案**：Zero-shot CoT 仅通过一句触发语（如"让我们逐步思考"）激活模型的链式推理能力，不需要任何示例。它的优势在于零配置、通用性好，适合任务类型多变或没有标注数据的场景。Few-shot CoT 需要在 Prompt 中提供 2-5 个完整的推理示例（问题 + 推理步骤 + 答案），让模型模仿特定格式和深度来推理。它的优势是输出格式更可控、推理深度更一致，适合有固定任务模板的生产场景（如客服工单分类、合同条款审查）。

效率对比：Zero-shot CoT 对推理增益约为 15%-25%（相比无 CoT），Few-shot CoT 可达 30%-50%。但 Few-shot 增加了 Prompt 长度和推理成本。实际选择时，如果任务多样性高且模型能力强（如 GPT-4/Claude），Zero-shot CoT 已足够；如果任务固定且对准确率要求严苛，Few-shot CoT 是更优选择。

### 题2：Instructor 库如何保证 LLM 输出符合 Pydantic Schema？描述其 Re-ask 重试机制。

**答案**：Instructor 的可靠性源于"校验-反馈-重试"闭环。首次调用时，Instructor 将 Pydantic Schema 序列化为 JSON Schema，嵌入到 API 请求的 `response_format` 或 function `parameters` 中，引导 LLM 按 Schema 输出。返回结果后，Instructor 尝试用 `model_validate()` 将 JSON 解析为 Pydantic 对象；若解析失败（类型不匹配、缺少必填字段、约束校验不通过），Instructor 提取具体的 `ValidationError` 信息（如"score 字段值 11 超出范围 1-10"），构造一条新的 user message 追加到对话历史，重新请求 LLM 修正。默认最多重试 2 次，可在 `from_openai()` 中通过 `max_retries` 参数调整。

这个机制的有效性基于两个前提：(1) LLM 具备理解校验错误并针对性修正的能力（当前主流模型已满足）；(2) 重试不会无限递归——实践中 90% 以上的错误在 1 次重试后即可修正。对于剩余顽固案例，Instructor 抛出 `InstructorRetryException`，开发者需兜底处理。