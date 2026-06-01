---
layout: post
title: "Agent设计模式：Plan-and-Execute架构实战"
category: 技术
tags: [AI Agent, Plan-and-Execute, 设计模式, LangGraph, 任务规划]
keywords: Plan-and-Execute, Agent设计模式, 任务规划, ReWOO, LLMCompiler, LangGraph
description: 深入拆解Plan-and-Execute架构的设计理念和工程实现，对比ReAct模式的优劣，通过完整Python代码演示如何构建可规划、可执行、可重规划的智能Agent系统。
---

## 一、核心概念

ReAct 模式是一步一想、边走边看——灵活但缺乏全局规划。面对多步骤、有依赖关系的复杂任务（如"调研三家竞品、汇总对比表、生成分析报告、发邮件给老板"），ReAct 容易走弯路、遗漏步骤甚至死循环。

Plan-and-Execute（P&E）正是为解决这一问题而生。它将 Agent 的执行过程分为两个阶段：

```
User Query → Planner → Plan (步骤列表) → Executor → Step1 → Step2 → ... → Replanner（可选）
```

- **Planner**：分析任务，生成结构化的执行计划（步骤列表，每步包含目标、依赖、所需工具）
- **Executor**：按计划逐步执行，每步完成后将结果反馈给上下文
- **Replanner**（可选）：在执行过程中发现计划有误或环境变化时，动态修正剩余步骤

P&E 的核心优势：**全局视角、可追踪、可并行**。LLM 在规划阶段看到的是完整任务，能识别出哪些步骤可以并行、哪些步骤依赖前置结果。

## 二、P&E 的三种主流变体

| 变体 | 特点 | 代表实现 |
|------|------|----------|
| 经典 P&E | 全量规划后再执行，计划固定 | LangChain PlanAndExecute |
| ReWOO | 规划时预分配变量，减少冗余推理 | ReWOO Paper (2023) |
| LLMCompiler | 构建任务 DAG，编译器优化并行执行 | LLMCompiler Paper (2024) |

本节重点实现经典 P&E 和 ReWOO。

## 三、实战：基于 LangGraph 的 P&E Agent

```python
from typing import TypedDict, List, Annotated
import operator
import json

# ---------- State ----------
class AgentState(TypedDict):
    user_query: str
    plan: List[dict]       # [{"step_id": 1, "description": "...", "tool": "...", "depends_on": []}]
    completed_steps: Annotated[List[dict], operator.add]
    final_answer: str

# ---------- Planner ----------
def planner(state: AgentState) -> dict:
    """分析任务，生成结构化执行计划"""
    query = state["user_query"]

    # 实际项目中这里是 LLM 调用
    plan = generate_plan(query)

    return {"plan": plan}

def generate_plan(query: str) -> List[dict]:
    """模拟 Planner：根据查询生成步骤列表"""
    if "竞品" in query and "报告" in query:
        return [
            {
                "step_id": 1,
                "description": "搜索竞品 A 的产品信息",
                "tool": "web_search",
                "arguments": {"query": "竞品A 产品功能 定价 2026"},
                "depends_on": []
            },
            {
                "step_id": 2,
                "description": "搜索竞品 B 的产品信息",
                "tool": "web_search",
                "arguments": {"query": "竞品B 产品功能 定价 2026"},
                "depends_on": []
            },
            {
                "step_id": 3,
                "description": "将两个竞品信息汇总为对比表格",
                "tool": "summarize",
                "arguments": {"source_steps": [1, 2]},
                "depends_on": [1, 2]
            },
            {
                "step_id": 4,
                "description": "生成竞品分析报告（Markdown）",
                "tool": "generate_report",
                "arguments": {"source_step": 3},
                "depends_on": [3]
            }
        ]
    return [
        {
            "step_id": 1,
            "description": "直接回答",
            "tool": "direct_answer",
            "arguments": {},
            "depends_on": []
        }
    ]

# ---------- Executor ----------
def executor(state: AgentState) -> dict:
    """按拓扑顺序执行计划中的步骤"""
    plan = state["plan"]
    completed = {s["description"]: s for s in state.get("completed_steps", [])}

    for step in plan:
        step_id = step["step_id"]

        # 检查依赖是否都已执行
        deps = step.get("depends_on", [])
        deps_met = all(
            any(s.get("step_id") == dep for s in state.get("completed_steps", []))
            for dep in deps
        )
        if not deps_met:
            continue  # 依赖未满足，跳过（实际应排队等待）

        # 收集依赖步骤的结果
        context = ""
        for dep_id in deps:
            dep_step = next(
                (s for s in state.get("completed_steps", [])
                 if s.get("step_id") == dep_id), None
            )
            if dep_step:
                context += f"\n[{dep_step['step_id']}的结果]: {dep_step.get('result', '')}"

        # 执行步骤
        result = execute_step(step, context)
        completed[step["description"]] = {"step_id": step_id, "result": result}

    return {"completed_steps": list(completed.values())}

def execute_step(step: dict, context: str) -> str:
    """模拟工具执行"""
    tool = step["tool"]
    args = step.get("arguments", {})

    if tool == "web_search":
        return f"已搜索「{args['query']}」: [模拟结果] 产品功能丰富，定价中等..."
    elif tool == "summarize":
        source_ids = args.get("source_steps", [])
        return f"汇总步骤 {source_ids} 的结果：| 维度 | 竞品A | 竞品B |\n| 功能 | ★★★★ | ★★★ |\n| 定价 | 中等 | 较高 |"
    elif tool == "generate_report":
        return f"# 竞品分析报告\n\n基于步骤结果生成完整报告..."
    return "已完成"

# ---------- Replanner ----------
def replanner(state: AgentState) -> dict:
    """检查执行结果，决定是否需要修正计划"""
    plan = state["plan"]
    completed = state.get("completed_steps", [])

    # 检查是否有步骤失败或结果异常
    for step in completed:
        if "失败" in step.get("result", "") or "错误" in step.get("result", ""):
            # 在实际项目中调用 LLM 重新生成后续步骤
            return {"plan": plan, "final_answer": "部分步骤执行失败，已重新规划"}
    return {"final_answer": "所有步骤执行完成。"}

# ---------- 主流程 ----------
def run_plan_execute(query: str):
    state = {"user_query": query, "plan": [], "completed_steps": [], "final_answer": ""}

    # Phase 1: 规划
    state.update(planner(state))
    print("=== Plan ===")
    for step in state["plan"]:
        deps = f" (依赖步骤: {step['depends_on']})" if step.get("depends_on") else ""
        print(f"  Step {step['step_id']}: {step['description']}{deps}")

    # Phase 2: 执行
    state.update(executor(state))
    print("\n=== Results ===")
    for step in state["completed_steps"]:
        print(f"  Step {step['step_id']}: {step['result'][:60]}...")

    return state["final_answer"]

run_plan_execute("帮我做竞品A和B的分析，生成对比报告")
```

## 四、ReWOO 模式：减少冗余推理

经典 P&E 的一个问题是：Executor 每执行一步都要做一次 LLM 推理来决定如何执行。ReWOO（Reasoning WithOut Observation）将冗余推理前移到规划阶段——Planner 不仅生成步骤，还为每个步骤分配**具名变量**，Executor 直接按变量填充即可，无需额外推理：

```python
# ReWOO Planner 输出示例
rewoo_plan = {
    "plan": [
        "#E1 = web_search('竞品A 2026功能')",
        "#E2 = web_search('竞品B 2026功能')",
        "#E3 = summarize(#E1, #E2)",       # E3 直接引用 E1、E2 的结果
        "#E4 = generate_report(#E3)"
    ]
}
```

Executor 看到 `#E3 = summarize(#E1, #E2)` 时，直接将 `#E1` 和 `#E2` 的值代入，无需 LLM 再次推理"summarize 需要什么参数"。

**ReWOO 的核心价值**：将 LLM 调用次数从"N 步执行 × 每步 1 次推理"减少为"1 次规划 + N 次纯执行"，在简单重复工具调用场景中可节省 60% 以上的 token 消耗。

## 五、最佳实践

1. **规划粒度要适中**：每步应当是"一个可独立执行的原子操作"——调用一个工具、执行一个脚本、查询一次 API。不要设计"分析并报告"这种模糊步骤。

2. **依赖声明要显式**：每个步骤的 `depends_on` 字段必须在规划阶段明确写清。Executor 应按拓扑排序执行，确保依赖步骤的结果在需要时已就绪。

3. **并行优化**：不互相依赖的步骤应标记为可并行。P&E 的最大工程价值之一就是天然支持并行——Planner 规划时即可识别，Executor 调度时自动并发。

4. **失败隔离**：某一步失败不应中断整个流程。Executor 应捕获异常，将错误信息记录到步骤结果中，由 Replanner 决定是重试、跳过还是替换方案。

5. **P&E vs ReAct 的选择**：任务步骤 ≥3 且步骤间有明显依赖关系 → 用 P&E；任务开放式、探索性强、步骤数不固定 → 用 ReAct。LangGraph 支持在同一个 Graph 中混合使用，将 ReAct 作为某个步骤的子循环。

---

## 面试题

### 题1：Plan-and-Execute 相比 ReAct 的优势和劣势分别是什么？在什么场景下你应该选择 P&E？

**答案**：优势：(1) 全局规划避免走弯路——Planner 在起始阶段就看到完整任务，可以规划最优路径，而 ReAct 每一步都是局部决策；(2) 天然支持并行——Planner 可以标注哪些步骤独立、Executor 自动并发执行，ReAct 的串行推理无法利用并行性；(3) 可追踪和可审计——计划作为显式的中间产物，方便调试和人工审核。

劣势：(1) 初始规划可能不准确——如果任务本身需要探索性操作（如"帮我查一下这个问题，可能需要多个数据源"），预先生成的计划可能遗漏关键步骤；(2) 对意外情况应变慢——Plan 是静态的，如果执行中途发现新信息需要改变策略，依赖 Replanner 的二次规划，增加了延迟；(3) 简单任务下 overhead 大——查询天气这种单步骤任务，P&E 的规划+执行反而比 ReAct 的一步到位更慢。

选择 P&E 的场景：(1) 多步骤且有明确依赖关系的任务（调研→汇总→报告）；(2) 可并行化的批量操作（同时对多个数据源做相同操作）；(3) 需要人工审批的工作流（计划先生成供审批再执行）；(4) 对延迟不敏感但要求结果完整性的批处理任务。

### 题2：ReWOO 模式如何减少 LLM 推理调用？它的核心设计思想是什么？

**答案**：ReWOO 的核心设计思想是"将执行时的推理负担前置到规划阶段"。经典 P&E 中，Executor 每执行一个步骤都需要调用 LLM 来决定如何调用工具、传什么参数——即使在规划阶段这些决策的"答案"已经隐含在计划描述中了。ReWOO 将计划写成带变量引用的伪代码形式（如 `#E3 = summarize(#E1, #E2)`），Executor 直接做字符串替换即可执行，无需 LLM 推理。

具体来说，ReWOO 将 LLM 调用从 O(N) 降到 O(1)（N 为执行步骤数）：Plan 阶段调用 1 次 LLM 生成带变量的完整计划；Execute 阶段每个步骤只做参数替换 + 工具调用，不调用 LLM。当 N=5 时，经典 P&E 需要 1（规划）+ 5（每步推理）= 6 次 LLM 调用，ReWOO 仅需 1 次。

局限性：如果工具的调用方式不固定（如"根据用户输入动态选择参数"），变量替换不够灵活，此时仍需回退到每次执行前做 LLM 推理。因此 ReWOO 最适合"工具调用模式可预测"的场景，如数据分析管道、文档处理流水线。