---
layout: post
title: "Agent设计模式：ReAct与Tool Use最佳实践"
category: 技术
tags: [AI Agent, ReAct, Tool Use, 设计模式, LangChain]
keywords: ReAct, Tool Use, Agent设计模式, 工具调用, 自主Agent
description: 深入剖析Agent开发的两种核心设计模式——ReAct循环与Tool Use机制，从架构设计到生产落地，覆盖工具注册规范、错误恢复、并发调度和可观测性等关键工程实践。
---

## 一、核心概念

现代 AI Agent 的自主性来源于两个基础能力的组合：**推理**（Reasoning）和**行动**（Acting）。ReAct 模式将二者交织在一个迭代循环中，而 Tool Use 机制则为 Agent 提供了与外部世界交互的"手脚"。理解二者的关系是 Agent 开发的起点。

### 1.1 ReAct 循环

ReAct（Reasoning + Acting）由 Google DeepMind 于 2022 年提出，核心思想是 LLM 不直接输出最终答案，而是按以下模式迭代：

```
Thought → Action → Observation → Thought → Action → Observation → ... → Final Answer
```

- **Thought**（思考）：LLM 分析当前状态，决定接下来做什么
- **Action**（行动）：调用一个工具，传入参数
- **Observation**（观察）：工具返回结果

循环终止条件：LLM 认为已有足够信息回答用户问题，或达到最大迭代次数。

### 1.2 Tool Use 机制

Tool Use 是指 LLM 调用外部函数/API 的能力的工程实现，包含三个核心组件：

- **工具注册表**：定义每个工具的名称、描述、参数 Schema（JSON Schema 格式）
- **工具选择器**：LLM 根据上下文决定调用哪个工具（或直接回答）
- **工具执行器**：解析 LLM 输出的函数调用，执行并返回结果

## 二、实战示例：从零构建 ReAct Agent

```python
import json
from typing import List, Dict, Any

# ---------- 1. 工具定义 ----------
tools_registry = [
    {
        "name": "search_knowledge_base",
        "description": "搜索内部知识库，返回最相关的文档片段",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string",
                          "description": "搜索查询语句"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "query_database",
        "description": "执行 SQL 查询，返回数据库中的数据",
        "parameters": {
            "type": "object",
            "properties": {
                "sql": {"type": "string",
                        "description": "SELECT 查询语句"}
            },
            "required": ["sql"]
        }
    },
    {
        "name": "send_notification",
        "description": "向指定用户发送通知消息",
        "parameters": {
            "type": "object",
            "properties": {
                "user_id": {"type": "string"},
                "message": {"type": "string"}
            },
            "required": ["user_id", "message"]
        }
    }
]

# ---------- 2. 工具执行器 ----------
def execute_tool(name: str, args: Dict[str, Any]) -> str:
    if name == "search_knowledge_base":
        return f"[知识库结果] 关于「{args['query']}」的文档：..."
    elif name == "query_database":
        return f"[数据库结果] 执行 {args['sql']}，返回 3 行数据"
    elif name == "send_notification":
        return f"[已发送] 通知已发送给用户 {args['user_id']}"
    return f"未知工具: {name}"

# ---------- 3. ReAct 循环 ----------
def react_loop(user_query: str, max_iterations: int = 5) -> str:
    messages = [
        {"role": "system", "content": f"""你是一个 ReAct Agent。可用工具：
{json.dumps(tools_registry, ensure_ascii=False, indent=2)}

回复格式：
Thought: 你的推理过程
Action: 工具名称
Action Input: JSON 格式参数
或
Thought: 你的推理过程
Final Answer: 最终答案""" },
        {"role": "user", "content": user_query}
    ]

    for i in range(max_iterations):
        # 模拟 LLM 调用
        response = simulate_llm_call(messages)

        if "Final Answer:" in response:
            return response.split("Final Answer:")[1].strip()

        # 解析 Action
        action = extract_action(response)
        if not action:
            return "Agent 未能确定下一步行动"

        tool_result = execute_tool(action["name"], action["args"])

        messages.append({"role": "assistant", "content": response})
        messages.append({"role": "user",
                         "content": f"Observation: {tool_result}"})

    return "达到最大迭代次数，任务未完成。"

def simulate_llm_call(messages: List[dict]) -> str:
    """模拟 LLM：识别用户意图并规划工具调用"""
    user_msg = messages[-1]["content"]

    if "查询订单" in user_msg:
        if "Observation:" in user_msg:
            # 已拿到数据库结果，返回最终答案
            return """Thought: 已从数据库获取订单数据，汇总即可。
Final Answer: 订单汇总：共3笔订单，总金额 1,280 元。"""
        return """Thought: 用户要查询订单，需要访问数据库。
Action: query_database
Action Input: {"sql": "SELECT * FROM orders WHERE date >= '2026-01-01'"}"""

    return """Thought: 无法确定用户意图。
Final Answer: 请具体说明你的需求。"""

def extract_action(response: str) -> dict | None:
    """解析 Action 和 Action Input"""
    import re
    action_match = re.search(r"Action:\s*(\w+)", response)
    input_match = re.search(r"Action Input:\s*(\{.+?\})", response, re.DOTALL)
    if action_match and input_match:
        return {
            "name": action_match.group(1),
            "args": json.loads(input_match.group(1))
        }
    return None

# ---------- 4. 运行 ----------
result = react_loop("帮我查询2026年1月以来的订单")
print(result)
```

## 三、最佳实践

### 3.1 工具设计规范

- **命名要"意图明确"，不要"实现描述"**：用 `search_knowledge_base` 而非 `call_vector_db_api`。LLM 根据工具的 name + description 做路由决策，差的命名直接导致路由错误。
- **描述要包含"何时使用"**：好的描述是"当用户询问产品信息、技术文档、FAQ 时使用此工具"而非"搜索知识库"。
- **参数要带 example**：在参数的 description 中加入示例值，能显著提升 LLM 传参的准确性。
- **单一职责**：一个工具只做一件事。不要设计一个"万能工具"根据参数切换不同行为，LLM 不擅长复杂的参数逻辑推理。

### 3.2 错误处理与恢复

Agent 系统中工具调用失败是常态而非异常。要建立分级恢复策略：

```python
def execute_tool_with_retry(name: str, args: dict, max_retries: int = 2):
    for attempt in range(max_retries + 1):
        try:
            return execute_tool(name, args)
        except TimeoutError:
            if attempt < max_retries:
                continue
            return f"工具 {name} 超时，请尝试缩小查询范围"
        except ValueError as e:
            # 参数错误不回退重试，直接反馈给 LLM
            return f"参数错误: {str(e)}，请修正参数后重试"
```

### 3.3 工具调用并发

当 Agent 需要调用多个互不依赖的工具时，串行执行会浪费大量时间。生产系统中建议用并发调度：

```python
import asyncio

async def execute_parallel(tool_calls: List[dict]) -> List[str]:
    tasks = [
        asyncio.to_thread(execute_tool, tc["name"], tc["args"])
        for tc in tool_calls
    ]
    return await asyncio.gather(*tasks)
```

LangGraph 的 `Send` API 原生支持这种模式，无需手动管理 asyncio。

### 3.4 可观测性

Agent 生产化的最大障碍不是模型不够聪明，而是出了问题不知道哪一步错的。必须建立完整的观测链路：

- **每轮 Thought 要记录**：不仅是文本，还要包含时间戳和 token 消耗
- **工具调用要埋点**：调用哪个工具、传了什么参数、返回了什么、耗时多久
- **设置终止护栏**：最大迭代次数（默认 5-10）、最大 token 消耗（防止模型陷入死循环反复调用同一工具）

---

## 面试题

### 题1：ReAct 模式相比"先规划再执行"的 Plan-Execute 模式有什么优势和劣势？

**答案**：优势：(1) 灵活性高，ReAct 每一步都基于最新的观察（Observation）动态调整下一步行动，能应对执行过程中的意外结果和中间发现的新信息；(2) 容错性强，工具调用失败时，失败信息作为 Observation 反馈给 LLM，模型可以修正参数重试或换用其他工具；(3) 实现简单，不需要规划器和执行器的分离架构。

劣势：(1) 效率低，每步都需要一次 LLM 调用来思考，简单任务也要多轮 token 消耗，延迟高；(2) 缺乏全局视角，模型只看到当前步骤的上下文，可能走弯路或重复调用；(3) 不擅长需要多步预规划的任务（如"先收集数据、再分析趋势、再生成图表、最后发邮件"），每一步独立决策可能导致子目标遗漏或顺序错乱。

实际选择：简单交互型 Agent（客服、问答）用 ReAct；复杂多步任务（自动化报表、数据管道）用 Plan-Execute。LangGraph 支持二者混合，可在 ReAct 循环中嵌入子图做局部规划。

### 题2：在设计 Tool Use 的工具描述（description）时，有哪些工程原则？如果 LLM 频繁选错工具，如何排查和修复？

**答案**：工具描述的设计原则：(1) 意图导向——描述工具"能帮用户解决什么问题"，而非"工具内部如何实现"，如"获取指定城市的实时天气"优于"调用 weather API 返回 JSON"；(2) 明确触发条件——写清楚什么场景下该用此工具，如"当用户询问天气、温度、降雨概率时使用"；(3) 区分度——确保不同工具的描述之间有足够的差异，避免 LLM 在多个相似工具间摇摆。

排查工具选错问题的步骤：首先检查工具描述是否过于相似（最常见原因），必要时合并功能重复的工具或用更明确的描述区分；其次检查 Prompt 中工具的排列顺序——LLM 对列表中靠前的工具有偏向性，把高频使用的工具放在前面；第三，给工具增加参数示例（在 description 中嵌入 example usage）；最后，如果问题仍然存在，在 `system prompt` 中添加"工具选择规则"章节，用 Few-shot 示例告知模型在特定场景下应选哪个工具。