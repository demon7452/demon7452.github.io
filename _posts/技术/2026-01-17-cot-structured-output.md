---
layout: post
title: "Prompt Engineering：思维链(CoT)与结构化输出"
date: 2026-01-17 10:00:00 +0800
category: 技术
tags: [Prompt Engineering, CoT, 思维链, 结构化输出, AI Agent]
keywords: 思维链, Chain-of-Thought, 结构化输出, Prompt工程, AI Agent开发
description: 深入讲解思维链(CoT) Prompt原理与结构化输出技术，结合实战案例展示如何让LLM输出可靠、可解析的JSON/XML结构，助力Agent开发。
---

## 核心概念

Prompt Engineering 是 AI Agent 开发中最基础也最容易被低估的技能。一个好的 Prompt 不仅能提升回答质量，还能让 LLM 的输出变得**可预测、可解析、可编排**——这正是 Agent 系统对 LLM 的核心诉求。

### 思维链（Chain-of-Thought, CoT）

CoT 的本质是让 LLM 在给出最终答案之前，先展示推理过程。这相当于把一个复杂的多步推理任务从隐式思考变成了显式步骤，大幅降低了模型"跳步"或"猜测"的概率。

CoT 的几种主流范式：

| 范式 | 说明 | 适用场景 |
|------|------|----------|
| **Zero-shot CoT** | 在 Prompt 末尾加 "Let's think step by step" | 通用推理任务 |
| **Few-shot CoT** | 提供 2-3 个带推理过程的示例 | 格式固定、有标准解法的任务 |
| **Self-Consistency** | 多次采样取多数答案，结合 CoT 进一步提升可靠性 | 高准确性要求的任务 |
| **Tree-of-Thought** | 在每个推理步骤探索多条路径，剪枝后取最优 | 复杂策略规划任务 |

### 结构化输出的价值

在 Agent 系统中，LLM 的输出通常不是给用户直接看的，而是要被下游代码解析和使用的。如果 LLM 返回自由格式的自然语言，解析就会变得极其脆弱。结构化输出要求 LLM 按照预定义的 Schema 返回数据（如 JSON），让下游可以可靠地反序列化。

## 实战示例

### 示例一：Zero-shot CoT + 结构化输出的组合

下面是一个客服意图分析 Agent，要求 LLM 同时展示推理过程和输出结构化结果：

```python
from openai import OpenAI
import json

client = OpenAI()

system_prompt = """你是一个客服意图分析系统。

分析规则：
1. 如果用户提到"退款"/"退货"/"质量"，标记为 priority=high, intent=refund
2. 如果用户提到"多少钱"/"价格"/"优惠"，标记为 priority=medium, intent=product_inquiry
3. 如果用户提到"怎么用"/"教程"/"帮助"，标记为 priority=low, intent=usage_help
4. 其他情况标记为 priority=low, intent=general

你必须严格按照以下JSON格式输出，不要输出任何JSON之外的内容：
{
  "reasoning": "你的逐步推理过程",
  "result": {
    "intent": "refund | product_inquiry | usage_help | general",
    "priority": "high | medium | low",
    "keywords": ["提取的关键词"],
    "confidence": 0.0-1.0
  }
}
"""

def analyze_intent(user_message: str) -> dict:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"请分析以下用户消息：\n\n{user_message}\n\nLet's think step by step."}
        ],
        temperature=0,  # 结构化输出必须设为0
        response_format={"type": "json_object"}  # OpenAI 原生 JSON 模式
    )
    return json.loads(response.choices[0].message.content)

# 测试
test_messages = [
    "我上周买的手机屏幕有坏点，我要退款！",
    "请问你们会员价格是多少？有没有优惠活动？",
    "不知道怎么绑定设备，有没有教程？",
]

for msg in test_messages:
    result = analyze_intent(msg)
    print(f"用户: {msg}")
    print(f"推理: {result['reasoning']}")
    print(f"结果: intent={result['result']['intent']}, "
          f"priority={result['result']['priority']}, "
          f"confidence={result['result']['confidence']}")
    print("---")
```

### 示例二：Few-shot CoT 实现复杂实体提取

对于需要复杂推理的实体提取任务，Few-shot CoT 比直接输出更可靠：

```python
few_shot_prompt = """你是一个合同信息提取系统。从合同文本中提取关键实体。

请先逐步分析合同条款，再输出JSON结果。

--- 示例 1 ---
合同文本：甲方张三向乙方李四借款人民币10万元，年利率5%，借款期限自2025年1月1日至2025年12月31日。

推理：这段文本是一个借贷合同。甲方是出借方，乙方是借款方。金额是10万元人民币，利率5%，借款期限跨越2025年全年。合同类型为借贷合同。

输出：
{
  "contract_type": "借贷合同",
  "parties": [
    {"role": "出借方", "name": "张三"},
    {"role": "借款方", "name": "李四"}
  ],
  "amount": {"value": 100000, "currency": "CNY", "text": "10万元"},
  "interest_rate": {"value": 5.0, "type": "年利率"},
  "period": {"start": "2025-01-01", "end": "2025-12-31"}
}

--- 示例 2 ---
合同文本：甲方科技有限公司将位于北京市海淀区的办公场地租赁给乙方创业公司使用，月租金3万元，租赁期24个月，自2025年3月1日起。

推理：（请仿照样例格式推理）
输出：（请仿照样例格式输出）
"""

def extract_contract_info(contract_text: str) -> dict:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": few_shot_prompt},
            {"role": "user", "content": f"合同文本：{contract_text}"}
        ],
        temperature=0,
    )
    # 从响应中提取 JSON
    content = response.choices[0].message.content
    # 找到 JSON 块
    json_start = content.rfind('{')
    json_end = content.rfind('}') + 1
    return json.loads(content[json_start:json_end])
```

### 示例三：结构化输出的防御性解析

生产环境中不能假设 LLM 一定返回合法 JSON，必须加防御层：

```python
import re

def safe_json_parse(llm_output: str, max_retries: int = 2) -> dict:
    """防御性 JSON 解析：处理 LLM 常见的输出格式问题"""
    for attempt in range(max_retries + 1):
        try:
            # 策略1: 直接解析
            return json.loads(llm_output)
        except json.JSONDecodeError:
            pass

        try:
            # 策略2: 提取 ```json ... ``` 代码块
            match = re.search(r'```(?:json)?\s*(.*?)\s*```', llm_output, re.DOTALL)
            if match:
                return json.loads(match.group(1))
        except (json.JSONDecodeError, AttributeError):
            pass

        try:
            # 策略3: 提取第一个完整 JSON 对象
            match = re.search(r'\{.*\}', llm_output, re.DOTALL)
            if match:
                return json.loads(match.group(0))
        except json.JSONDecodeError:
            pass

        if attempt < max_retries:
            continue

    raise ValueError(f"无法从LLM输出中解析JSON: {llm_output[:200]}...")
```

## 最佳实践

1. **temperature=0 是结构化输出的底线**：任何需要解析的 LLM 输出场景，temperature 必须设为 0 或使用 `response_format={"type": "json_object"}`，否则输出格式不稳定。
2. **System Prompt 中明确 Schema**：用 JSON 示例或字段说明在 system prompt 中定义输出格式，比在 user prompt 中定义更稳定（system prompt 的指令遵循优先级更高）。
3. **防御性解析不可省略**：即使用了 `response_format={"type": "json_object"}`，仍应保留 `safe_json_parse` 兜底。模型可能在 JSON 值中嵌套特殊字符导致解析失败。
4. **Few-shot > Zero-shot（复杂任务）**：对于需要多步推理或格式要求严格的任务，2-3 个示例的 Few-shot CoT 效果远超 Zero-shot。示例的选择应覆盖边界情况，而不是最典型的 case。
5. **CoT 推理过程也要结构化**：对于需要 audit log 的生产系统，让推理过程也成为 JSON 的一个字段（如 `"reasoning": "..."`），而不是让 LLM 自由输出后再截断。一个 JSON 输出包含所有信息，解析和日志记录都更方便。
6. **验证层必不可少**：解析 JSON 后必须用 Pydantic 做 Schema 校验，确保所有必填字段存在且类型正确。

## 面试题

**Q1：为什么在 Prompt 中要求 LLM 输出 JSON 时，必须将 temperature 设为 0？如果 temperature > 0 可能产生什么问题？**

**A1**：temperature 控制 LLM 输出 token 的随机性。temperature > 0 时，模型在每个 token 的选择上会引入随机采样，这可能导致两个严重后果：

1. **格式不稳定**：JSON 对语法要求极其严格，多一个逗号、少一个引号、键名拼写错误都会导致解析失败。temperature > 0 时这些问题的概率大幅上升，在多次调用中表现不一致。
2. **值的不确定性**：对于需要一致决策的场景（如意图分类），temperature > 0 可能导致同一输入产生不同分类结果，系统行为不可预测。

这也就是为什么 OpenAI 的 JSON Mode（`response_format={"type": "json_object"}`）在底层也会强制约束 token 采样策略，确保输出有效 JSON。

---

**Q2：Few-shot CoT 和 Zero-shot CoT 的核心区别是什么？在什么情况下必须使用 Few-shot CoT？**

**A2**：核心区别在于示例的注入。Zero-shot CoT 仅靠一句"Let's think step by step"触发推理，模型自主决定推理路径和格式；Few-shot CoT 通过人工编写的 2-3 个完整示例（包含推理步骤和最终输出），同时教会模型**怎么推理**和**输出什么格式**。

必须使用 Few-shot CoT 的场景：

- 输出格式复杂且非标准（如特定的嵌套 JSON Schema），Zero-shot 无法可靠遵循。
- 任务推理路径有明确的领域规范（如法律合同的要素提取顺序），需要示例来对齐。
- 边界 case 多，需要示例来展示对模糊输入的处理方式。

本质区别：Zero-shot CoT 告诉模型"你要思考"，Few-shot CoT 告诉模型"你要这样思考，并输出这样的结果"。
