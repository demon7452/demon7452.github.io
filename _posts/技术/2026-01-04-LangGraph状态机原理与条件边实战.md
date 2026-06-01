---
layout: post
title: "LangGraph状态机原理与条件边实战"
category: 技术
tags: [LangGraph, AI Agent, 状态机, 条件边, Python]
keywords: LangGraph, 状态机, 条件边, Agent开发, 工作流编排
description: 深入解析LangGraph中的状态机模型与条件边机制，通过完整代码示例演示如何构建灵活的Agent工作流，覆盖多轮对话、动态路由和并行执行等核心场景。
---

## 一、核心概念

LangGraph 是 LangChain 生态中的有向图状态机框架，用节点（Node）和边（Edge）来建模 Agent 的执行流程。与传统的 DAG（有向无环图）不同，LangGraph 支持**循环边**，让 Agent 的执行图天然适配多轮推理和工具调用场景。

### 1.1 状态（State）

LangGraph 中的 State 是一个带类型注解的字典，贯穿整个图的执行过程。每个节点读取 State、返回部分更新，LangGraph 自动将这些更新合并回全局 State。State 的 Schema 通过 `TypedDict` 定义：

```python
from typing import TypedDict, List, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[List[dict], operator.add]  # 累加合并
    next_step: str                                   # 覆盖合并
    tool_results: dict                               # 覆盖合并
```

`Annotated` 标注决定合并策略：`operator.add` 表示追加，默认行为是覆盖。这是整个框架最核心的设计，理解它才能驾驭复杂工作流。

### 1.2 条件边（Conditional Edge）

条件边是 LangGraph 区别于普通 DAG 的关键。它允许从一个节点出发后，根据 State 中的某个字段动态决定跳转到哪个下游节点。比如 LLM 调用的返回结果中包含了 `function_call`，条件边就可以根据函数名路由到不同的工具节点。

```python
from langgraph.graph import StateGraph, END

def router(state: AgentState) -> str:
    last_msg = state["messages"][-1]
    if last_msg.get("function_call"):
        func_name = last_msg["function_call"]["name"]
        return func_name          # 路由到对应工具节点
    return END                    # 无函数调用则结束
```

## 二、实战示例：天气查询 + 搜索整合 Agent

以下实现一个完整的多工具 Agent：用户提问后，LLM 决定是查询天气还是搜索网页，最后汇总结果。

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolExecutor
from typing import TypedDict, List, Annotated
import operator

# ---------- State 定义 ----------
class AgentState(TypedDict):
    messages: Annotated[List[dict], operator.add]
    next_step: str

# ---------- 模拟工具 ----------
def get_weather(city: str) -> str:
    return f"{city}今天晴，25°C"

def web_search(query: str) -> str:
    return f"关于「{query}」的搜索结果：..."

# ---------- 节点 ----------
def call_llm(state: AgentState) -> dict:
    """LLM 推理 + 决定调用哪个工具"""
    # 模拟 LLM 的推理过程
    user_input = state["messages"][-1]["content"]
    if "天气" in user_input:
        return {"messages": [{
            "role": "assistant",
            "function_call": {"name": "get_weather",
                              "arguments": {"city": "北京"}}
        }]}
    else:
        return {"messages": [{
            "role": "assistant",
            "function_call": {"name": "web_search",
                              "arguments": {"query": user_input}}
        }]}

def execute_tool(state: AgentState) -> dict:
    """执行具体工具"""
    fc = state["messages"][-1]["function_call"]
    args = fc["arguments"]
    if fc["name"] == "get_weather":
        result = get_weather(**args)
    else:
        result = web_search(**args)
    return {"messages": [{"role": "tool", "content": result}]}

def summarize(state: AgentState) -> dict:
    """汇总结果"""
    tool_msg = state["messages"][-1]["content"]
    return {"messages": [{"role": "assistant",
                           "content": f"根据工具返回，总结如下：{tool_msg}"}]}

# ---------- 条件路由 ----------
def route_after_llm(state: AgentState) -> str:
    """核心：条件边，根据 LLM 输出动态路由"""
    last_msg = state["messages"][-1]
    if last_msg.get("function_call"):
        return "execute_tool"
    return END

# ---------- 构建图 ----------
graph = StateGraph(AgentState)

graph.add_node("call_llm", call_llm)
graph.add_node("execute_tool", execute_tool)
graph.add_node("summarize", summarize)

graph.set_entry_point("call_llm")

# 条件边：LLM → 工具 or 结束
graph.add_conditional_edges("call_llm", route_after_llm, {
    "execute_tool": "execute_tool",
    END: END
})

# 工具执行后 → 汇总 → 结束
graph.add_edge("execute_tool", "summarize")
graph.add_edge("summarize", END)

app = graph.compile()

# ---------- 运行 ----------
result = app.invoke({"messages": [{"role": "user", "content": "北京今天天气怎么样？"}]})
for msg in result["messages"]:
    print(f"[{msg['role']}]: {msg.get('content', msg.get('function_call', ''))}")
```

## 三、最佳实践

1. **State 设计先行**：先画好数据流图，再定义 State。合并策略（add vs 覆盖）要在 TypedDict 中显式标注，不要依赖隐式行为。

2. **条件边函数保持纯函数**：路由函数只读 State、返回字符串，不要在路由函数中修改 State 或调用外部服务，否则调试会非常痛苦。

3. **节点粒度适中**：一个节点做一件事。LLM 调用、工具执行、结果汇总应当拆分，便于单测和条件路由。

4. **善用 `interrupt_before` / `interrupt_after`**：LangGraph 支持在图执行中插入断点，这在需要人类审批的关键步骤（如发送邮件、执行删除操作）中至关重要。

5. **并行节点用 `Send` API**：当多个工具调用互不依赖（如同时查天气和搜索），LangGraph 的 `Send` 机制可以自动并行它们，显著降低延迟。

---

## 面试题

### 题1：LangGraph 中的 `Annotated[list, operator.add]` 是什么意思？为什么这样设计？

**答案**：这是 Python `typing` 的类型标注语法，`Annotated` 第一个参数是基础类型（列表中元素积累），第二个参数 `operator.add` 指明了 State 合并策略——当多个节点返回同名键时，使用列表追加（add）而非覆盖。这样设计解决了多节点并发更新同一字段时的竞态问题。例如在聊天历史场景中，LLM 节点和工具节点都会往 `messages` 列表追加新消息，用 `operator.add` 保证消息顺序完整保留。若用默认覆盖策略，后执行的节点会直接覆盖前一个节点的输出，导致消息丢失。

### 题2：什么是"条件边"？它和普通边有什么区别？在什么场景下必须使用条件边？

**答案**：条件边是 LangGraph 中一种动态路由机制。普通边（`add_edge`）的跳转目标是编译期固定的；条件边（`add_conditional_edges`）在运行时根据 State 的当前值决定跳转目标。关键区别在于确定性——普通边永远是 A→B，条件边是 A→（B 或 C 或 D...）。

必须使用条件边的场景包括：(1) LLM 输出函数调用后，需要根据函数名路由到不同工具节点；(2) 需要循环迭代的场景，如 ReAct 循环中"推理→行动→观察→推理"，必须用条件边判断何时跳出循环到 END；(3) 工作流中存在分支逻辑，如审批流中的"通过/驳回"分叉。

实现上，条件边需要一个路由函数 `fn(state) -> str`，返回的字符串映射到节点名称或 END。