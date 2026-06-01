---
layout: post
title: "Agent设计模式：Multi-Agent协作架构"
category: 技术
tags: [AI Agent, Multi-Agent, 协作架构, 角色分工, LangGraph]
keywords: Multi-Agent, 多智能体, 角色分工, 协作模式, Supervisor, Debate, 群聊
description: 深入剖析Multi-Agent协作架构的三种核心模式——Supervisor监督、顺序流水线和群聊辩论，通过完整的Python代码演示如何构建分工明确、可扩展的多智能体协作系统。
---

## 一、核心概念

单 Agent 有天然的能力天花板——一个 LLM 既要分析问题、又要查询数据、还要计算和生成报告，容易顾此失彼。Multi-Agent 架构将复杂的全局任务拆解为多个**专家 Agent**，每个 Agent 只负责自己擅长的子任务，通过协作完成整体目标。

三种经典协作模式：

| 模式 | 协作方式 | 适用场景 |
|------|---------|---------|
| **Supervisor（监督者）** | 一个主管 Agent 分配任务、调度执行、仲裁冲突 | 任务边界清晰、可预先分工 |
| **Sequential（流水线）** | Agent 按固定顺序执行，前一个的输出作为后一个的输入 | 数据处理管道、文档生成流水线 |
| **Debate / Chat（辩论/群聊）** | 多个 Agent 自由交换信息、互相挑战观点 | 创意发散、风险评估、多视角分析 |

## 二、架构一：Supervisor 模式实战

Supervisor 是生产中最常用的 Multi-Agent 模式。以下用 LangGraph 构建一个包含研究员（Researcher）、分析师（Analyst）和报告写手（Writer）的三 Agent 协作系统：

```python
from typing import TypedDict, List, Annotated, Literal
import operator
from langgraph.graph import StateGraph, END

# ---------- State ----------
class TeamState(TypedDict):
    messages: Annotated[List[dict], operator.add]
    task: str
    research_data: str        # 研究员产出
    analysis_result: str      # 分析师产出
    final_report: str         # 最终报告
    next_agent: str           # Supervisor 决策

# ---------- Agent 节点 ----------
def researcher(state: TeamState) -> dict:
    """研究员：搜索和收集原始数据"""
    task = state["task"]
    result = f"[研究员] 针对「{task}」进行了资料收集：\n"
    result += "- 竞品 A：市场份额 35%，定价策略中等\n"
    result += "- 竞品 B：市场份额 28%，定价策略激进\n"
    result += "- 行业趋势：AI 驱动型产品增速 45%\n"
    return {"research_data": result, "messages": [{"role": "researcher", "content": result}]}

def analyst(state: TeamState) -> dict:
    """分析师：基于研究数据做深度分析"""
    data = state.get("research_data", "")
    result = f"[分析师] 基于研究数据分析：\n"
    result += "- 竞品 A 优势：品牌认知度高、渠道覆盖广\n"
    result += "- 竞品 A 弱点：创新速度慢、定价灵活度低\n"
    result += "- 市场机会：中端价格带存在空白\n"
    result += "- 风险提示：竞品 B 可能在 Q2 大幅降价\n"
    return {"analysis_result": result, "messages": [{"role": "analyst", "content": result}]}

def writer(state: TeamState) -> dict:
    """写手：整合研究与分析，生成最终报告"""
    research = state.get("research_data", "")
    analysis = state.get("analysis_result", "")

    report = f"""# 竞品分析报告

## 市场概况
{research}

## 深度分析
{analysis}

## 行动建议
1. 优先抢占中端价格带
2. 加速 AI 功能迭代，拉开与竞品 A 的差距
3. 建立价格预警机制应对竞品 B 的降价策略
"""
    return {"final_report": report, "messages": [{"role": "writer", "content": report}]}

# ---------- Supervisor ----------
def supervisor(state: TeamState) -> dict:
    """Supervisor：决策下一步由哪个 Agent 执行"""
    completed_steps = len([m for m in state["messages"]
                           if m["role"] in ("researcher", "analyst", "writer")])

    # 第一步 → 研究员
    if not state.get("research_data"):
        return {"next_agent": "researcher"}
    # 有数据但缺分析 → 分析师
    elif not state.get("analysis_result"):
        return {"next_agent": "analyst"}
    # 有数据有分析但缺报告 → 写手
    elif not state.get("final_report"):
        return {"next_agent": "writer"}
    # 全部完成
    return {"next_agent": "FINISH"}

# ---------- 路由函数 ----------
def route_supervisor(state: TeamState) -> str:
    agent = state["next_agent"]
    if agent == "FINISH":
        return END
    return agent

# ---------- 构建图 ----------
builder = StateGraph(TeamState)

builder.add_node("researcher", researcher)
builder.add_node("analyst", analyst)
builder.add_node("writer", writer)
builder.add_node("supervisor", supervisor)

builder.set_entry_point("supervisor")

# Supervisor → 路由到各 Agent
builder.add_conditional_edges("supervisor", route_supervisor, {
    "researcher": "researcher",
    "analyst": "analyst",
    "writer": "writer",
    END: END
})

# 各 Agent 执行完 → 回到 Supervisor 继续调度
builder.add_edge("researcher", "supervisor")
builder.add_edge("analyst", "supervisor")
builder.add_edge("writer", "supervisor")

app = builder.compile()

# 运行
result = app.invoke({
    "task": "分析 AI SaaS 市场竞争格局并生成报告",
    "messages": [],
    "research_data": "",
    "analysis_result": "",
    "final_report": "",
    "next_agent": ""
})

print(result["final_report"])
```

## 三、架构二：Sequential 流水线模式

当任务的子目标有严格先后顺序时（如 ETL 管道），流水线比 Supervisor 更高效——省去了中间的调度开销：

```python
def build_pipeline():
    """构建严格顺序的流水线：提取 → 清洗 → 分析 → 报告"""
    builder = StateGraph(TeamState)

    builder.add_node("extract", extract_data)
    builder.add_node("clean", clean_data)
    builder.add_node("analyze", analyze_data)
    builder.add_node("report", generate_report)

    builder.set_entry_point("extract")
    builder.add_edge("extract", "clean")
    builder.add_edge("clean", "analyze")
    builder.add_edge("analyze", "report")
    builder.add_edge("report", END)

    return builder.compile()
```

## 四、架构三：群聊辩论模式

适用于需要多视角碰撞的场景——如风险评估、产品 brainstorm。每个 Agent 持有不同的角色和观点，通过多轮对话互相挑战：

```python
AGENT_PROFILES = {
    "optimist": "你是一个乐观主义者，总是看到机会和积极面。",
    "pessimist": "你是一个悲观主义者，擅长发现风险和潜在问题。",
    "realist": "你是一个现实主义者，基于数据和事实做判断。"
}

def debate_round(state: TeamState, agent_role: str) -> dict:
    """一轮辩论：指定角色发表观点"""
    system_prompt = AGENT_PROFILES[agent_role]
    context = "\n".join([m["content"] for m in state["messages"]])
    response = f"[{agent_role}观点] 基于上下文给出 {agent_role} 视角的分析..."

    return {"messages": [{"role": agent_role, "content": response}]}
```

## 五、Multi-Agent 架构对比与选型

| 维度 | Supervisor | Sequential | Debate |
|------|-----------|------------|--------|
| 灵活性 | 高（动态调度） | 低（固定顺序） | 最高（自由交互） |
| 延迟 | 中（每次切换需 Supervisor 决策） | 低（无调度开销） | 高（多轮交互） |
| Token 消耗 | 中 | 低 | 高 |
| 适用场景 | 通用型复杂任务 | 数据处理管道 | 创意发散、风险评估 |
| 实现复杂度 | 中 | 低 | 高（收敛控制难） |

## 六、最佳实践

1. **Agent 数量控制在 3-5 个**：Agent 越多协调成本越高，Supervisor 的调度复杂度指数增长。超过 5 个时应考虑将部分 Agent 合并或分层（引入 Sub-Supervisor）。

2. **Supervisor 决策要可审计**：每个调度决策附带 `reason` 字段记录选择该 Agent 的原因，方便排查"为什么走了这条路径"。

3. **Agent 间通信走 State 不走直接调用**：Agent 只读写全局 State，不耦合彼此的接口。这样新增 Agent 时无需修改已有 Agent 的代码。

4. **设置全局终止条件**：Supervisor 模式下必须有"任务完成"的判定标准（如报告已生成、所有子任务已完成），否则多 Agent 可能陷入无限反馈循环。

5. **共享工具注册表**：多个 Agent 可能用到相同的工具（如搜索、数据库查询）。统一在 State 中维护工具注册表，避免每个 Agent 独立注册造成不一致。

---

## 面试题

### 题1：Supervisor 模式和 Sequential 流水线模式的核心区别是什么？在什么场景下 Supervisor 比 Sequential 更合适？

**答案**：核心区别在于**执行路径的确定性**。Sequential 的执行路径是编译期固定的——A→B→C→END，无论 State 中发生了什么变化，这个顺序不会改变。Supervisor 的执行路径是运行时动态的——Supervisor 每轮根据 State 决定下一步由哪个 Agent 执行，路径可能因中间结果而变化。

Supervisor 比 Sequential 更合适的场景：(1) 任务存在条件分支——如"先分析数据，如果异常则调调查员进一步排查，否则直接生成报告"，Sequential 无法表达这种条件逻辑；(2) 需要动态重试——如"研究员返回的数据不足，Supervisor 可以要求研究员补充搜索后再交给分析师"；(3) 任务步骤数不固定——如"继续调研直到收集到至少 3 个竞品的信息"，Sequential 无法表达"直到"这种循环语义。

Sequential 更合适的场景：步骤固定、无分支、对延迟敏感的数据处理管道——如标准的 ETL 流水线或文档生成流程（搜集→整理→排版→导出）。

### 题2：Multi-Agent 系统中的"共同记忆"（Shared Memory）如何设计？为什么不能简单地让所有 Agent 共用一个 messages 列表？

**答案**：Shared Memory 的设计需要解决两个问题——信息过载和信息隔离。简单共用一个 messages 列表会导致每个 Agent 看到所有历史信息（包括其他 Agent 的中间推理过程），造成 Token 浪费和注意力分散。

推荐的设计：(1) 分层 State——顶层 State 放全局共享的关键结果（如 `research_data`、`analysis_result`），每个 Agent 只读写自己关心的字段；(2) Agent 私有工作区——每个 Agent 有独立的工作消息列表，仅当产出最终结果时才写入全局 State；(3) Supervisor 做信息中转——Supervisor 在调度时从全局 State 中提取目标 Agent 需要的上下文，组装为精简的 Task 描述传递下去。

这样设计的好处是：研究员不需要看到分析师的推导过程，分析师不需要看到写手的格式调整，每个 Agent 只接收最小必要信息，既节省 Token 又提高专注度。