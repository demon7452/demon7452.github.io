---
layout: post
title: "LangGraph深入：循环节点与Checkpoint机制"
category: 技术
tags: [LangGraph, Checkpoint, 循环节点, Agent状态管理, 断点续传]
keywords: LangGraph, Checkpoint, 循环节点, 状态持久化, 断点续传, 人机交互
description: 深入解析LangGraph中的循环节点实现原理与Checkpoint状态持久化机制，通过实战代码演示如何构建支持断点续传、人机交互审批和故障恢复的生产级Agent工作流。
---

## 一、核心概念

在上一篇 LangGraph 基础中，我们构建了线性 + 条件边的 Agent 图。但真正的生产级 Agent 系统有两个刚需是基础图无法满足的：

**循环节点**：ReAct 模式的本质就是"思考→行动→观察→思考"的循环，每个周期都在同一个子图中运行。LangGraph 的循环边让节点可以回到上游重新执行，直到满足终止条件。

**Checkpoint 机制**：Agent 执行可能跨越数分钟甚至需要人工审批。如果在第 4 步失败了，应该从第 4 步恢复而非重跑全部。Checkpoint 在每个超级步骤（Superstep）结束后自动保存 State 快照，支持精确恢复。

二者的结合让 LangGraph 从一个"任务执行器"升级为"有状态的工作流引擎"。

## 二、循环节点实战：ReAct Agent 的完整实现

以下用 LangGraph 原生的 `add_edge` 实现循环，而非依赖 `AgentExecutor` 的封装：

```python
from typing import TypedDict, List, Annotated
import operator
from langgraph.graph import StateGraph, END

# ---------- State ----------
class AgentState(TypedDict):
    messages: Annotated[List[dict], operator.add]
    iteration_count: int          # 防止无限循环

MAX_ITERATIONS = 5

# ---------- 节点 ----------
def think(state: AgentState) -> dict:
    """LLM 推理：分析当前状态，决定下一步"""
    iteration = state.get("iteration_count", 0) + 1

    # 模拟推理：如果消息中已有工具结果，判断是否足以回答
    messages = state["messages"]
    tool_results = [m for m in messages if m["role"] == "tool"]

    if len(tool_results) >= 2 or "Final Answer" in str(messages[-1].get("content", "")):
        return {
            "messages": [{
                "role": "assistant",
                "content": "已收集足够信息，准备给出最终答案。"
            }],
            "iteration_count": iteration
        }

    # 需要继续调用工具
    return {
        "messages": [{
            "role": "assistant",
            "content": "还需要更多信息，继续调用工具...",
            "function_call": {"name": "search", "arguments": {"query": "latest data"}}
        }],
        "iteration_count": iteration
    }

def act(state: AgentState) -> dict:
    """执行工具调用"""
    last_msg = state["messages"][-1]
    if not last_msg.get("function_call"):
        return {"messages": []}

    fc = last_msg["function_call"]
    result = f"[工具 {fc['name']} 结果] 搜索 '{fc['arguments']['query']}' 返回: ..."

    return {"messages": [{"role": "tool", "content": result}]}

def finalize(state: AgentState) -> dict:
    """汇总所有工具结果，生成最终回答"""
    tool_results = [m["content"] for m in state["messages"] if m["role"] == "tool"]
    summary = "基于以下信息给出答案:\n" + "\n".join(tool_results)
    return {"messages": [{"role": "assistant", "content": summary}]}

# ---------- 路由函数 ----------
def should_continue(state: AgentState) -> str:
    """核心循环控制：判断是继续循环还是结束"""
    iteration = state.get("iteration_count", 0)

    # 硬限制：到达最大迭代次数强制结束
    if iteration >= MAX_ITERATIONS:
        return "finalize"

    # 根据最后一条消息判断
    last_msg = state["messages"][-1]
    if last_msg["role"] == "assistant" and last_msg.get("function_call"):
        return "act"                      # 有工具调用 → 继续循环
    elif last_msg["role"] == "tool":
        return "think"                    # 工具结果返回 → 回到思考
    else:
        return "finalize"                 # 无工具调用 → 结束

# ---------- 构建图 ----------
graph = StateGraph(AgentState)

graph.add_node("think", think)
graph.add_node("act", act)
graph.add_node("finalize", finalize)

graph.set_entry_point("think")

# 关键：条件边形成循环
graph.add_conditional_edges("think", should_continue, {
    "act": "act",
    "finalize": "finalize"
})
graph.add_edge("act", "think")           # ← 循环回到思考节点
graph.add_edge("finalize", END)

app = graph.compile()

# 运行
result = app.invoke({
    "messages": [{"role": "user", "content": "帮我调研最新的技术趋势"}],
    "iteration_count": 0
})
print(f"总迭代次数: {result['iteration_count']}")
print(f"消息轮数: {len(result['messages'])}")
```

## 三、Checkpoint 机制深度解析

### 3.1 什么是 Checkpoint？

LangGraph 在每个**超级步骤**（Superstep）结束后自动保存 State 的完整快照。超级步骤定义为：从一个节点出发，经过所有同步边和条件边到达的下一个节点集合，直到遇到"阻塞点"（如 `interrupt_before` 标记）。

```python
from langgraph.checkpoint.memory import MemorySaver

# 创建带 Checkpoint 的编译
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)
```

每个 Checkpoint 包含：完整的 State、当前所在的节点、时间戳、父 Checkpoint ID（形成链式历史）。

### 3.2 断点续传实战

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, END

# ---------- 模拟一个需要审批的工作流 ----------
class WorkflowState(TypedDict):
    document: str         # 待审批文档内容
    review_decision: str  # 审批决定：approved / rejected / pending
    final_output: str

def draft(state: WorkflowState) -> dict:
    return {"document": "合同草案：甲方向乙方采购..."}

def human_review(state: WorkflowState) -> dict:
    """人工审批节点——此处仅占位，实际审批由人类通过 update_state 完成"""
    return {"review_decision": "pending"}

def execute_decision(state: WorkflowState) -> dict:
    decision = state["review_decision"]
    if decision == "approved":
        return {"final_output": f"已执行：{state['document']}"}
    return {"final_output": f"已驳回：{state['document']}"}

def route_after_review(state: WorkflowState) -> str:
    if state["review_decision"] == "pending":
        return END  # 等待人工审批
    return "execute_decision"

# ---------- 构建图 ----------
wf = StateGraph(WorkflowState)
wf.add_node("draft", draft)
wf.add_node("human_review", human_review)
wf.add_node("execute_decision", execute_decision)

wf.set_entry_point("draft")
wf.add_edge("draft", "human_review")
wf.add_conditional_edges("human_review", route_after_review, {
    "execute_decision": "execute_decision",
    END: END
})
wf.add_edge("execute_decision", END)

# ---------- 编译：在 review 节点前中断 ----------
memory = MemorySaver()
app = wf.compile(checkpointer=memory, interrupt_before=["human_review"])

# 配置线程 ID（用于跨调用恢复）
config = {"configurable": {"thread_id": "contract-001"}}

# 第一轮：执行到 human_review 前自动暂停
result = app.invoke({"document": "", "review_decision": "", "final_output": ""}, config)
print(f"暂停位置: {app.get_state(config).next}")  # ('human_review',)

# 查看当前 State
current_state = app.get_state(config)
print(f"文档内容: {current_state.values.get('document')}")

# 人工审批通过——通过 update_state 注入决策
app.update_state(config, {"review_decision": "approved"})

# 恢复执行
result = app.invoke(None, config)  # None = 从上次暂停位置继续
print(f"最终输出: {result['final_output']}")
```

### 3.3 Checkpoint 的持久化后端

| 后端 | 场景 | 特点 |
|------|------|------|
| MemorySaver | 开发/调试 | 进程内内存，重启丢失 |
| SqliteSaver | 单机生产 | 文件持久化，支持并发读 |
| PostgresSaver | 分布式生产 | 多实例共享，事务一致 |

```python
# SqliteSaver 示例
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

conn = sqlite3.connect("agent_checkpoints.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
app = wf.compile(checkpointer=checkpointer)
```

## 四、最佳实践

1. **循环必须有硬上限**：永远不要依赖 LLM 自行决定何时退出循环。`MAX_ITERATIONS` 设 5-10，并在 State 中跟踪迭代计数，这是防止死循环的最后一道防线。

2. **Checkpoint 命名要有业务含义**：`thread_id` 用 `"{业务类型}-{业务ID}-{时间戳}"` 格式，方便运维排查。如 `"order-approval-ORDER20260301-17120000"`。

3. **`interrupt_before` 用于"高风险节点"**：所有涉及写操作（发邮件、修改数据库、调用支付接口）的节点都应设 `interrupt_before`，由人类确认后再执行。这比事后回滚安全得多。

4. **利用 Checkpoint 做 A/B 回放**：从历史 Checkpoint 分叉出新的执行路径（通过 `update_state` 注入不同参数），可以复用昂贵的前置计算来对比不同决策的后果。

5. **线程安全**：MemorySaver 不是线程安全的，多并发场景必须用 SqliteSaver 或 PostgresSaver。

---

## 面试题

### 题1：LangGraph 的 Checkpoint 机制和普通数据库事务有什么异同？为什么 Agent 系统需要专门的 Checkpoint 而非直接用数据库存储？

**答案**：相同点——都需要保证状态的一致性、支持失败回滚、支持并发控制。不同点有三个层面：

(1) 粒度不同：数据库事务以 SQL 操作为边界，Checkpoint 以"超级步骤"为边界——一个超级步骤包含多个节点的执行和状态合并，是业务语义级别的原子单元。(2) 恢复方式不同：数据库事务失败是回滚到事务开始前的状态；Checkpoint 则是"暂停后继续"，失败后不是回滚而是从上一个成功的 Checkpoint 恢复并重试失败的超级步骤。(3) 历史链不同：Checkpoint 维护从根到当前状态的完整版本链（类似 Git），支持时间旅行和分叉，而数据库通常只维护当前状态。

Agent 系统需要专门的 Checkpoint 的原因：Agent 的执行是长时间（可能数分钟到数小时）、多阶段的，中间需要人机交互（审批、确认）。如果用数据库存储，开发者需要手动管理状态序列化、版本关联、断点位置标记等——这些正是 Checkpoint 抽象已经封装好的能力。

### 题2：循环节点中如何平衡"允许充分探索"和"防止无限循环"？列出至少三种防护机制。

**答案**：防护机制：(1) 硬性迭代上限——在 State 中维护 `iteration_count`，循环路由函数中 `if iteration >= MAX_ITERATIONS: return END`。这是最后防线，通常设 5-10。(2) 重复调用检测——如果连续两次调用了同一工具且传入了完全相同的参数，说明模型陷入了无意义的循环，应立即中断并向用户报告"Agent 陷入循环"。(3) Token 预算——设置单次会话的 Token 消耗上限（如 5000 output tokens），通过 LangChain 的 `max_token_limit` 回调或 LangGraph 的自定义 `interrupt_after` 实现。此外还有特征检测（识别 LLM 输出的无助循环特征，如反复说"让我再试一次"）和时间限制（单次 Agent 运行不超过 60 秒）。