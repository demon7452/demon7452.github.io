---
layout: post
title: "Agent设计模式：ReAct与Tool Use最佳实践"
date: 2026-01-24 10:00:00 +0800
category: 技术
tags: [AI Agent, ReAct, Tool Use, 设计模式, Function Calling]
keywords: ReAct模式, Tool Use, Function Calling, AI Agent设计模式, 工具调用
description: 深入解析AI Agent两大核心设计模式——ReAct与Tool Use，通过完整代码实战展示如何构建可靠的工具调用系统，并总结生产环境下的最佳实践。
---

## 核心概念

AI Agent 区别于普通 LLM 应用的核心特征，在于它能够**主动使用工具**并与外部环境交互。这一能力的工程落地依赖两种紧密相关的设计模式：ReAct 和 Tool Use。

### ReAct 模式

ReAct（Reasoning + Acting）由 Google DeepMind 在 2022 年提出，核心思想是将 LLM 的推理过程和行动过程交错执行：

```
Thought（思考）→ Action（行动）→ Observation（观察）→ Thought → Action → ... → Final Answer
```

每次循环中，Agent 先"思考"当前需要什么信息，然后"行动"（调用工具），接着观察工具的返回结果，再基于新信息继续思考。这个循环直到 Agent 认为信息充足时终止，输出最终答案。

### Tool Use / Function Calling

Tool Use 是 ReAct 中 Action 环节的工程实现。LLM 不直接执行工具，而是输出一个结构化的工具调用请求（函数名 + 参数），由外部运行时解析并执行，结果返回给 LLM 继续推理。OpenAI 的 Function Calling 和 Anthropic 的 Tool Use 都是这一模式的标准实现。

### ReAct vs 传统 Few-shot 的区别

| 维度 | 传统 Few-shot | ReAct |
|------|-------------|-------|
| 信息获取 | 仅依赖训练数据 | 运行时动态获取外部信息 |
| 时效性 | 知识截止于训练日期 | 可访问实时数据 |
| 准确性 | 可能产生幻觉 | 基于工具返回的真实数据 |
| 可追溯性 | 黑盒输出 | 每步推理和行动可审计 |

## 实战示例

下面实现一个完整的 ReAct Agent，它能搜索互联网、执行计算、查询天气：

```python
import json
import re
from typing import List, Dict, Any
from openai import OpenAI

client = OpenAI()

# =========== 1. 定义工具 ===========
def web_search(query: str) -> str:
    """模拟网络搜索"""
    # 生产环境替换为真实的搜索 API
    knowledge_base = {
        "Python 3.13": "Python 3.13 于 2024 年 10 月发布，引入了新的 JIT 编译器...",
        "LangGraph": "LangGraph 是 LangChain 生态中的状态机 Agent 框架...",
        "GPT-4o": "GPT-4o 是 OpenAI 的多模态模型，支持文本、图片、音频...",
    }
    for key in knowledge_base:
        if key.lower() in query.lower():
            return knowledge_base[key]
    return f"未找到与 '{query}' 相关的结果。"

def calculator(expression: str) -> str:
    """安全的数学计算器"""
    allowed_chars = set("0123456789+-*/().%^ ")
    if not all(c in allowed_chars for c in expression):
        return "错误：表达式包含不允许的字符"
    try:
        # 安全 eval（仅允许数学表达式）
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"

def get_weather(city: str) -> str:
    """模拟天气查询"""
    weather_data = {
        "北京": "晴，25°C，湿度40%",
        "上海": "多云，28°C，湿度65%",
        "深圳": "阵雨，30°C，湿度80%",
    }
    return weather_data.get(city, f"未找到 {city} 的天气数据")

# 工具注册表
TOOLS = {
    "web_search": web_search,
    "calculator": calculator,
    "get_weather": get_weather,
}

# 工具 Schema（用于 Function Calling）
TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "搜索互联网获取最新信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculator",
            "description": "执行数学计算",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "数学表达式，如 '2+3*4'"}
                },
                "required": ["expression"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询指定城市的天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名"}
                },
                "required": ["city"]
            }
        }
    }
]

# =========== 2. ReAct Agent 核心循环 ===========
class ReActAgent:
    def __init__(self, max_iterations: int = 10):
        self.max_iterations = max_iterations
        self.history: List[Dict[str, Any]] = []

    def run(self, user_query: str) -> str:
        """执行 ReAct 循环"""
        messages = [
            {"role": "system", "content": self._get_system_prompt()},
            {"role": "user", "content": user_query}
        ]
        self.history = []

        for iteration in range(self.max_iterations):
            print(f"\n--- 迭代 {iteration + 1} ---")

            response = client.chat.completions.create(
                model="gpt-4o",
                messages=messages,
                tools=TOOL_SCHEMAS,
                tool_choice="auto",
                temperature=0
            )

            choice = response.choices[0]
            msg = choice.message

            # 如果 LLM 决定调用工具
            if msg.tool_calls:
                messages.append(msg)  # 添加 assistant 消息（含 tool_calls）

                for tool_call in msg.tool_calls:
                    func_name = tool_call.function.name
                    func_args = json.loads(tool_call.function.arguments)

                    print(f"  调用工具: {func_name}({func_args})")

                    # 执行工具
                    tool_func = TOOLS.get(func_name)
                    if tool_func is None:
                        result = f"错误：未知工具 '{func_name}'"
                    else:
                        result = tool_func(**func_args)

                    print(f"  工具返回: {result[:100]}...")

                    self.history.append({
                        "iteration": iteration + 1,
                        "tool": func_name,
                        "arguments": func_args,
                        "result": result
                    })

                    # 添加工具结果消息
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": result
                    })

            # 如果 LLM 直接返回文本（没有 tool_calls）
            else:
                final_answer = msg.content
                print(f"  最终答案: {final_answer}")
                return final_answer

        return "已达到最大迭代次数，无法完成任务。"

    def _get_system_prompt(self) -> str:
        return """你是一个 ReAct Agent。你可以使用以下工具：

1. web_search(query) - 搜索互联网获取信息
2. calculator(expression) - 执行数学计算
3. get_weather(city) - 查询城市天气

ReAct 原则：
- 将复杂问题拆解为多个步骤
- 每次只调用一个必要的工具
- 基于工具返回的信息继续推理
- 当你拥有足够信息回答用户时，直接给出答案
- 不要制造信息，只能基于工具返回的真实数据回答"""

# =========== 3. 测试 ===========
agent = ReActAgent(max_iterations=10)

queries = [
    "Python 3.13 有什么新特性？",
    "北京今天天气怎么样？",
    "计算 (1234 + 5678) * 0.15 的结果",
    "深圳的天气和北京比哪个更热？",
]

for q in queries:
    print(f"\n{'='*50}")
    print(f"用户提问: {q}")
    print(f"{'='*50}")
    answer = agent.run(q)
    print(f"\n>>> 最终回答: {answer}")
```

### 工具调用的错误处理模式

生产环境中工具调用可能失败，需要实现优雅的降级：

```python
import functools
import time

def tool_retry(max_retries: int = 2, backoff: float = 1.0):
    """工具调用重试装饰器"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    if attempt < max_retries:
                        time.sleep(backoff * (2 ** attempt))
            return f"工具执行失败（重试{max_retries}次后仍失败）: {last_error}"
        return wrapper
    return decorator

@tool_retry(max_retries=2, backoff=0.5)
def web_search_robust(query: str) -> str:
    """带重试的搜索工具"""
    # 实际调用可能超时、网络错误等
    return web_search(query)
```

## 最佳实践

1. **工具描述决定调用质量**：LLM 是否能正确选择工具，80% 取决于工具描述的质量。描述要说明工具的能力边界、参数含义、返回值格式，以及**什么时候应该使用这个工具**（而不是另一个）。
2. **限制单次返回的工具数量**：一次给 LLM 超过 10 个工具时，选择准确率会显著下降。对于大型工具集，应使用分层路由——先用分类器缩小工具范围，再在子集中做 Function Calling。
3. **工具结果要精简**：工具返回给 LLM 的结果应控制在 1000 tokens 以内。如果原始结果很大（如搜索返回全文），应先做摘要或截断，否则会撑爆上下文窗口。
4. **实现工具调用超时**：每个工具调用都应设置超时（如 30 秒），防止某个工具卡住导致整个 Agent 挂起。
5. **记录工具调用链**：每次工具调用的参数和返回值都应记录到结构化日志中，方便后续审计、调试和成本分析。
6. **安全边界在工具层**：不要在 Prompt 里试图阻止 LLM 做危险操作，而应在工具函数内部做权限控制（如 calculator 限制可执行的操作类型）。

## 面试题

**Q1：ReAct 模式中的 "Observation" 环节如果返回了错误信息（如工具超时、参数无效），Agent 应当如何处理？**

**A1**：Observation 返回错误信息时，Agent 不应直接输出"我不知道"，而应进入**纠错循环**：

1. **识别错误类型**：LLM 收到错误信息后，首先判断是参数问题（可修复）还是服务问题（需降级）。
2. **参数修正**：如果是参数错误（如城市名拼写错误、日期格式不对），LLM 应修正参数后重新调用同一工具。
3. **工具降级**：如果是服务不可用（超时、限流），LLM 应尝试调用备用工具或调整策略（如用 web_search 代替 get_weather）。
4. **优雅告知**：如果所有尝试均失败，LLM 应向用户坦白说明原因，而非编造答案。

关键设计原则：工具的错误信息必须是结构化的（如 `{"error": "timeout", "detail": "..."}`），帮助 LLM 做出正确的后续决策。字符串形式的模糊错误（如 "Error"）会让 LLM 难以判断下一步行动。

---

**Q2：为什么推荐工具数量超过 10 个时使用分层路由而非直接将所有工具暴露给 LLM？**

**A2**：两个核心原因：

1. **注意力稀释**：LLM 的上下文窗口虽然大，但有效注意力集中在开头和结尾。当 tool_choice 列表过长时，每个工具的描述获得的"注意力预算"被严重稀释，LLM 可能忽略关键工具或混淆相似工具。实验表明，工具从 5 个增加到 20 个时，选择准确率可下降 15%-25%。

2. **Token 成本**：每个工具的 Schema 描述消耗 input tokens，10 个复杂工具可能占 2000+ tokens。每次对话都要携带全部工具的 Schema，多轮对话中成本累积极快。

分层路由的标准方案：第一层用一个轻量级分类 LLM（如 gpt-4o-mini）做意图识别，将用户请求映射到某个工具组（如"数据查询组"/"文件操作组"/"外部 API 组"）；第二层在该组内（通常 3-5 个工具）用 Function Calling 做精确匹配。这样每个 LLM 调用只需要处理少量工具，准确率和成本都更优。