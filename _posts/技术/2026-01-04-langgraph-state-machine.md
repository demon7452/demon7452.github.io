---
layout: post
title: "LangGraph状态机原理与条件边实战"
date: 2026-01-04 10:00:00 +0800
category: 技术
tags: [LangGraph, 状态机, AI Agent, 条件边, 工作流]
keywords: LangGraph, 状态机, 条件边, AI Agent开发, 工作流编排
description: 深入解析LangGraph状态机核心原理，通过实战案例演示条件边的设计与实现，帮助开发者掌握AI Agent工作流编排的关键技术。
---

## 核心概念

LangGraph 是 LangChain 生态中用于构建**有状态、多参与者**Agent 应用的核心框架。它的设计灵感来源于图计算中的状态机模型——将 Agent 的决策流程建模为一个有向图，其中节点代表计算步骤，边代表状态转移条件。

### 状态机三要素

1. **State（状态）**：在整个图执行过程中共享的数据结构，通常是一个 TypedDict 或 Pydantic 模型，承载消息历史、中间变量、执行上下文等。
2. **Nodes（节点）**：封装具体逻辑的执行单元，可以是 LLM 调用、工具调用、数据处理函数等。
3. **Edges（边）**：定义节点之间的转移规则，分为普通边（固定转移）和条件边（动态路由）。

### 条件边（Conditional Edge）的本质

条件边是 LangGraph 区别于传统 DAG 工作流的关键特性。它允许根据当前 State 的内容动态选择下一个要执行的节点，从而实现类似 `if-else`、`switch-case` 甚至循环的执行逻辑。条件边通过一个路由函数（Router Function）实现，该函数接收当前 State 并返回下一个目标节点的名称。

## 实战示例

下面通过一个客服 Agent 实战案例，演示条件边的完整用法。该 Agent 需要根据用户意图将请求路由到不同模块：

- 用户需要人工客服 → 转人工模块
- 用户咨询产品信息 → 知识库检索模块
- 用户请求退款 → 退款流程模块
- 其他情况 → LLM 直接回复

```python
from typing import TypedDict, Literal, Annotated
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage

# 1. 定义共享状态
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    intent: str                    # 路由意图
    user_profile: dict             # 用户信息
    refund_status: str             # 退款状态

# 2. 意图识别节点
def intent_classifier(state: AgentState) -> AgentState:
    """使用 LLM 识别用户意图"""
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    last_message = state["messages"][-1].content

    prompt = f"""分析以下用户消息的意图，仅返回以下之一：
    - human_support   (要求人工客服)
    - product_inquiry (产品咨询)
    - refund          (退款/售后)
    - general         (其他)

    用户消息：{last_message}"""

    response = llm.invoke(prompt)
    intent = response.content.strip().lower()
    return {"intent": intent}

# 3. 各业务节点（简化实现）
def human_support_node(state: AgentState) -> AgentState:
    return {"messages": [AIMessage(content="已为您转接人工客服，请稍候...")]}

def product_inquiry_node(state: AgentState) -> AgentState:
    """产品咨询：先检索知识库，再生成回复"""
    last_message = state["messages"][-1].content
    # 模拟知识库检索
    knowledge = f"关于「{last_message[:20]}...」的检索结果：产品A售价299元..."
    return {"messages": [AIMessage(content=knowledge)]}

def refund_node(state: AgentState) -> AgentState:
    return {"messages": [AIMessage(content="已为您创建退款工单，预计3个工作日内处理。")]}

def general_reply_node(state: AgentState) -> AgentState:
    llm = ChatOpenAI(model="gpt-4o")
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# 4. 条件路由函数——核心！
def route_by_intent(state: AgentState) -> Literal[
    "human_support", "product_inquiry", "refund", "general_reply"
]:
    """根据意图动态选择下一个节点"""
    intent_map = {
        "human_support": "human_support",
        "product_inquiry": "product_inquiry",
        "refund": "refund",
    }
    return intent_map.get(state["intent"], "general_reply")

# 5. 构建图
builder = StateGraph(AgentState)

# 添加节点
builder.add_node("intent_classifier", intent_classifier)
builder.add_node("human_support", human_support_node)
builder.add_node("product_inquiry", product_inquiry_node)
builder.add_node("refund", refund_node)
builder.add_node("general_reply", general_reply_node)

# 设置入口
builder.set_entry_point("intent_classifier")

# 添加条件边——根据意图路由
builder.add_conditional_edges(
    "intent_classifier",   # 源节点
    route_by_intent,       # 路由函数
    {
        "human_support": "human_support",
        "product_inquiry": "product_inquiry",
        "refund": "refund",
        "general_reply": "general_reply",
    }
)

# 所有业务节点完成后 → END
builder.add_edge("human_support", END)
builder.add_edge("product_inquiry", END)
builder.add_edge("refund", END)
builder.add_edge("general_reply", END)

# 编译图（带记忆检查点）
memory = MemorySaver()
graph = builder.compile(checkpointer=memory)

# 6. 执行测试
config = {"configurable": {"thread_id": "user-session-001"}}

# 测试退款意图
result = graph.invoke(
    {"messages": [HumanMessage(content="我买的商品有质量问题，要求退款")]},
    config
)
print(result["messages"][-1].content)
# 输出: 已为您创建退款工单，预计3个工作日内处理。

# 测试产品咨询
result2 = graph.invoke(
    {"messages": [HumanMessage(content="你们的产品A多少钱？")]},
    config
)
print(result2["messages"][-1].content)
# 输出: 关于「你们的产品A多少钱？」的检索结果：产品A售价299元...
```

### 条件边的高级模式：带循环的 Agent

实际 Agent 场景中，经常需要**循环执行**——比如 Tool Use Agent 的标准流程："LLM 思考 → 调用工具 → LLM 基于工具结果再思考 → 再调工具 → ... → 完成"。这需要条件边支持"回到自身或前置节点"：

```python
def should_continue(state: AgentState) -> Literal["tools", END]:
    """判断是否需要继续调用工具"""
    last_message = state["messages"][-1]
    # 如果最后一条消息包含 tool_calls，则继续执行工具
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END

# 工具节点执行后回到 LLM 节点，形成循环
builder.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
builder.add_edge("tools", "agent")  # 工具执行后回到 agent
```

## 最佳实践

1. **State 设计尽量扁平**：避免深层嵌套导致的可序列化问题和调试困难。优先使用 TypedDict 明确字段类型。
2. **路由函数保持纯净**：条件边函数应该是纯函数（仅依赖 State 输入），不要在路由函数内部做副作用操作（如调用外部 API）。
3. **使用 Literal 类型约束返回值**：为路由函数标注 `Literal["node_a", "node_b"]` 类型，IDE 可提供自动补全，运行时也能更早发现拼写错误。
4. **利用 checkpointer 实现断点续传**：生产环境应配置持久化 checkpointer（如 SQLite、Postgres），确保 Agent 执行中断后可恢复。
5. **单节点单职责**：每个节点只做一件事。意图识别就是一个节点，知识检索是另一个节点，LLM 生成回复又是另一个节点。节点职责清晰，图的可维护性和可测试性都会大幅提升。
6. **给条件边加兜底路由**：路由函数必须覆盖所有可能的返回值，否则图编译或运行时会抛出异常。建议始终保留一个 "fallback" 分支。

## 面试题

**Q1：LangGraph 中的普通边和条件边有什么区别？分别在什么场景下使用？**

**A1**：普通边（`add_edge`）定义的是固定转移——无论 State 内容如何，执行完节点 A 后总是跳到节点 B。适用于确定性的工作流步骤（如"生成回复后结束"）。

条件边（`add_conditional_edges`）根据 State 运行时内容动态选择目标节点。它配合路由函数实现 `if-else` 逻辑，适用于需要动态路由的场景（如意图分发、工具调用循环、权限校验分流等）。条件边是构建自适应 Agent 的关键机制——它让图不再是一条"死路"，而是能够根据上下文灵活调整执行路径。

---

**Q2：在 LangGraph 中实现一个 Tool Use Agent 的标准循环模式，需要哪几个核心组件？请画出节点和边的关系。**

**A2**：标准 Tool Use Agent 循环需要以下组件：

- **节点**：`agent`（LLM 推理节点）、`tools`（工具执行节点）
- **边关系**：
  - `agent` → 条件边 → 如果 LLM 输出包含 `tool_calls` → `tools`；否则 → `END`
  - `tools` → 普通边 → `agent`（形成闭环）

流程：用户输入 → `agent`（LLM 决定是否需要工具）→ 需要工具 → `tools`（执行）→ `agent`（基于工具结果继续推理）→ 不需要工具 → `END`（返回最终答案）。这个循环的核心就是条件边 `should_continue` 函数，它检查最后一条消息是否包含 tool_calls 来决定是继续循环还是结束。
