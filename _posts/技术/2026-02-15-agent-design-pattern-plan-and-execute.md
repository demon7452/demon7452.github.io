---
layout: post
title: "Agent设计模式：Plan-and-Execute架构"
category: 技术
tags: [AI Agent, Plan-and-Execute, Agent设计模式, LLM应用, 任务规划]
keywords: [Plan-and-Execute, Agent Architecture, Task Planning, LangGraph, LLM Application]
description: "深入讲解Plan-and-Execute Agent架构，通过完整的项目实战演示如何实现任务规划、动态调整和执行监控。"
date: 2026-02-15
---

# Agent设计模式：Plan-and-Execute架构

## 引言

**Plan-and-Execute**（计划-执行）是一种更高级的Agent设计模式。与ReAct的"边走边看"不同，Plan-and-Execute强调"先规划、后执行"，类似项目管理中的WBS（工作分解结构）思路。这种模式在复杂多步骤任务中表现出色，能显著降低中间步骤的偏差累积。

本文将深入讲解Plan-and-Execute的核心原理、与ReAct的对比，并通过完整项目展示如何构建这样的Agent。

## 核心概念

### Plan-and-Execute vs ReAct

```python
# ReAct：边走边看
"""
Thought: 我需要先查天气
Action: get_weather("北京")
Observation: "晴，25°C"
Thought: 天气不错，可以推荐户外活动
Action: search_activities("户外")
...
"""

# Plan-and-Execute：先规划后执行
"""
Plan:
Step 1: 查询天气 (get_weather)
Step 2: 根据天气筛选活动 (search_activities)
Step 3: 排序推荐 (rank_recommendations)
Step 4: 生成最终回答

然后按计划逐步执行...
"""
```

### 架构图

```
┌──────────┐    Plan     ┌──────────┐    Steps    ┌──────────┐
│ Planner  │──────────►  │ Executor │──────────►  │ Replanner│
│ (LLM)    │             │          │             │ (判断)    │
└──────────┘             └──────────┘             └──────────┘
                                │                       │
                                ▼                       ▼
                          ┌──────────┐           需要调整？
                          │  Tools   │          ──┬── 是 → 重新规划
                          └──────────┘            └── 否 → 继续执行
```

## 实战：旅游规划Agent

### 完整代码实现

```python
import json
from typing import TypedDict, Annotated, Sequence, List, Optional
import operator
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, END

# ========== 1. 状态定义 ==========
class PlanExecuteState(TypedDict):
    """Plan-and-Execute状态"""
    input: str                           # 用户输入
    plan: List[dict]                     # 执行计划 [{"id":1, "step":"...", "tool":"..."}, ...]
    past_steps: Annotated[List[tuple], operator.add]  # 已执行步骤及结果
    response: str                        # 最终回答
    need_replan: bool                    # 是否需要重新规划

# ========== 2. 模拟工具 ==========
def search_attractions(city: str) -> str:
    """搜索旅游景点"""
    attractions_db = {
        "北京": "故宫(4.8分)、长城(4.9分)、颐和园(4.7分)、天坛(4.6分)、798艺术区(4.5分)",
        "上海": "外滩(4.8分)、迪士尼(4.7分)、东方明珠(4.5分)、南京路(4.4分)、城隍庙(4.6分)",
        "杭州": "西湖(4.9分)、灵隐寺(4.7分)、宋城(4.5分)、千岛湖(4.6分)、雷峰塔(4.4分)",
        "成都": "宽窄巷子(4.7分)、大熊猫基地(4.9分)、锦里(4.5分)、都江堰(4.8分)、青城山(4.6分)"
    }
    return attractions_db.get(city, "未找到该城市景点信息")

def search_hotels(city: str, budget: str = "中等") -> str:
    """搜索酒店"""
    hotels = {
        ("北京", "经济"): "如家酒店(￥200/晚)、汉庭酒店(￥220/晚)",
        ("北京", "中等"): "全季酒店(￥450/晚)、亚朵酒店(￥500/晚)",
        ("北京", "豪华"): "王府半岛(￥2000/晚)、国贸大酒店(￥2500/晚)",
        ("上海", "经济"): "格林豪泰(￥180/晚)、锦江之星(￥200/晚)",
        ("上海", "中等"): "桔子水晶(￥400/晚)、美居酒店(￥480/晚)",
        ("上海", "豪华"): "和平饭店(￥2200/晚)、浦东文华(￥2800/晚)",
    }
    return hotels.get((city, budget), f"{city} {budget}档次酒店：价格区间￥200-800/晚")

def search_weather(city: str, date: str = "") -> str:
    """查询天气"""
    weather_db = {
        "北京": "晴转多云，15-28°C，适宜旅游",
        "上海": "小雨，20-25°C，建议带伞",
        "杭州": "晴，18-30°C，非常适宜出游",
        "成都": "多云，16-26°C，适宜旅游"
    }
    return weather_db.get(city, "晴，20-28°C")

def calculate_budget(days: int, hotel_budget: str, people: int = 1) -> str:
    """计算预算"""
    hotel_cost = {"经济": 200, "中等": 500, "豪华": 2000}
    daily_hotel = hotel_cost.get(hotel_budget, 500)
    daily_food = 150 * people
    daily_transport = 100 * people
    daily_attractions = 200 * people
    
    total = (daily_hotel + daily_food + daily_transport + daily_attractions) * days
    
    return f"""预算估算：
- 酒店：￥{daily_hotel * days}（{daily_hotel}/天 × {days}天）
- 餐饮：￥{daily_food * days}（{daily_food}/天）
- 交通：￥{daily_transport * days}（{daily_transport}/天）
- 景点：￥{daily_attractions * days}（{daily_attractions}/天）
- 总计：约￥{total}"""

# ========== 3. Agent实现 ==========
class TravelPlanAgent:
    """旅游规划Agent - Plan-and-Execute模式"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
        self.planner_llm = ChatOpenAI(model="gpt-4o", temperature=0)
        self.tools = {
            "search_attractions": search_attractions,
            "search_hotels": search_hotels,
            "search_weather": search_weather,
            "calculate_budget": calculate_budget
        }
        self.graph = self._build_graph()
    
    def _build_graph(self):
        """构建Plan-and-Execute状态图"""
        workflow = StateGraph(PlanExecuteState)
        
        workflow.add_node("planner", self._plan_step)
        workflow.add_node("executor", self._execute_step)
        workflow.add_node("replanner", self._replan_step)
        workflow.add_node("finalizer", self._finalize)
        
        workflow.set_entry_point("planner")
        
        workflow.add_edge("planner", "executor")
        workflow.add_conditional_edges(
            "executor",
            self._should_continue,
            {
                "continue": "executor",
                "replan": "replanner",
                "end": "finalizer"
            }
        )
        workflow.add_edge("replanner", "executor")
        workflow.add_edge("finalizer", END)
        
        return workflow.compile()
    
    def _plan_step(self, state: PlanExecuteState) -> PlanExecuteState:
        """步骤1：制定计划"""
        plan_prompt = f"""你是一个旅游规划专家。

用户需求：{state['input']}

请制定一个详细的执行计划。每个步骤应该具体、可执行，并指定需要调用的工具。

可用工具：search_attractions(城市名), search_hotels(城市名, 预算), search_weather(城市名), calculate_budget(天数, 酒店预算, 人数)

输出JSON格式的计划：
{{
    "plan": [
        {{
            "id": 1,
            "step": "步骤描述",
            "tool": "工具名称（如果不需要工具则为null）",
            "tool_args": {{"arg1": "value1"}}
        }}
    ]
}}

要求：
1. 先查询天气，再推荐景点
2. 根据景点位置推荐酒店
3. 最后计算总预算
4. 步骤应该在5-8个之间
"""
        
        response = self.planner_llm.invoke(plan_prompt)
        
        # 解析计划
        plan_data = json.loads(response.content)
        
        return {
            "plan": plan_data["plan"],
            "past_steps": [],
            "need_replan": False
        }
    
    def _execute_step(self, state: PlanExecuteState) -> PlanExecuteState:
        """步骤2：执行计划中的下一步"""
        plan = state["plan"]
        past_steps = state["past_steps"]
        
        # 找出下一个未执行的步骤
        executed_ids = {step[0] for step in past_steps}
        
        for step in plan:
            if step["id"] not in executed_ids:
                current_step = step
                break
        else:
            # 所有步骤已执行
            return state
        
        # 执行步骤
        step_result = None
        tool_name = current_step.get("tool")
        
        if tool_name and tool_name in self.tools:
            tool = self.tools[tool_name]
            tool_args = current_step.get("tool_args", {})
            
            try:
                step_result = tool(**tool_args)
            except Exception as e:
                step_result = f"工具执行错误: {str(e)}"
        else:
            # LLM推理步骤
            step_result = f"[推理步骤] {current_step['step']}"
        
        # 记录执行结果
        return {
            "past_steps": [(current_step["id"], current_step["step"], step_result)]
        }
    
    def _should_continue(self, state: PlanExecuteState) -> str:
        """判断是否继续执行、重新规划或结束"""
        plan = state["plan"]
        past_steps = state["past_steps"]
        
        # 检查是否需要重新规划
        if state.get("need_replan", False):
            return "replan"
        
        # 检查是否所有步骤已执行
        executed_ids = {step[0] for step in past_steps}
        
        if len(executed_ids) >= len(plan):
            return "end"
        
        return "continue"
    
    def _replan_step(self, state: PlanExecuteState) -> PlanExecuteState:
        """步骤3：重新规划"""
        replan_prompt = f"""原计划执行过程中需要进行调整。

原始需求：{state['input']}

已执行步骤及结果：
{json.dumps(state['past_steps'], ensure_ascii=False, indent=2)}

剩余未执行步骤：
{json.dumps([s for s in state['plan'] if s['id'] not in {step[0] for step in state['past_steps']}], ensure_ascii=False, indent=2)}

请根据已执行结果调整剩余计划，输出调整后的完整后续步骤（JSON数组）。
注意：已执行的步骤不需要重复。"""
        
        response = self.planner_llm.invoke(replan_prompt)
        
        try:
            new_plan = json.loads(response.content)
        except:
            new_plan = state["plan"]
        
        return {
            "plan": new_plan,
            "need_replan": False
        }
    
    def _finalize(self, state: PlanExecuteState) -> PlanExecuteState:
        """步骤4：生成最终回答"""
        finalize_prompt = f"""基于以下执行结果，为用户生成一份完整的旅游建议。

用户需求：{state['input']}

执行结果：
{json.dumps(state['past_steps'], ensure_ascii=False, indent=2)}

请生成：
1. 天气情况
2. 推荐景点及评分
3. 酒店推荐
4. 预算估算
5. 行程建议（按天安排）

输出为结构化的旅游规划报告。
"""
        
        response = self.llm.invoke(finalize_prompt)
        
        return {"response": response.content}
    
    def run(self, user_input: str) -> str:
        """运行Agent"""
        initial_state = {"input": user_input}
        final_state = self.graph.invoke(initial_state)
        return final_state.get("response", "无法生成回答")

# ========== 4. 使用示例 ==========
def demo():
    agent = TravelPlanAgent()
    
    result = agent.run("我想去北京玩3天，中等预算，4个人。帮我规划一下。")
    print(result)

if __name__ == "__main__":
    demo()
```

## Plan-and-Execute最佳实践

### 1. 计划粒度控制

```python
# ✅ 推荐：中等粒度
GOOD_PLAN = [
    {"id": 1, "step": "查询目的地天气", "tool": "search_weather"},
    {"id": 2, "step": "搜索热门景点", "tool": "search_attractions"},
    {"id": 3, "step": "搜索合适酒店", "tool": "search_hotels"},
    {"id": 4, "step": "计算预算", "tool": "calculate_budget"},
    {"id": 5, "step": "生成行程建议", "tool": None}
]

# ❌ 不推荐：过细
BAD_PLAN_FINE = [
    {"id": 1, "step": "查询天气", "tool": "search_weather"},
    {"id": 2, "step": "分析温度数据", "tool": None},
    {"id": 3, "step": "判断是否适合出行", "tool": None},
    # ...太多细小步骤
]

# ❌ 不推荐：过粗
BAD_PLAN_COARSE = [
    {"id": 1, "step": "完成旅游规划", "tool": None}
    # 一个步骤包办一切，失去了规划的意义
]
```

### 2. 动态重规划触发条件

```python
class ReplanTrigger:
    """重规划触发器"""
    
    @staticmethod
    def should_replan(step_result: str, expected_output: str = None) -> bool:
        """判断是否需要重新规划"""
        
        # 条件1：工具执行失败
        if "错误" in step_result or "failed" in step_result.lower():
            return True
        
        # 条件2：结果不符合预期
        if expected_output and expected_output not in step_result:
            return True
        
        # 条件3：结果为空
        if not step_result or step_result.strip() == "":
            return True
        
        # 条件4：结果过短（异常）
        if len(step_result) < 20:
            return True
        
        return False
```

### 3. 计划缓存与复用

```python
import hashlib
from functools import lru_cache

class CachedPlanner:
    """带缓存的规划器"""
    
    def __init__(self, llm):
        self.llm = llm
        self.plan_cache = {}
    
    def plan(self, user_input: str) -> list:
        """制定计划（带缓存）"""
        # 生成缓存Key
        input_hash = hashlib.md5(user_input.encode()).hexdigest()
        
        if input_hash in self.plan_cache:
            print("使用缓存的计划")
            return self.plan_cache[input_hash]
        
        plan = self._generate_plan(user_input)
        self.plan_cache[input_hash] = plan
        
        return plan
```

## 面试题

### 面试题1：请比较ReAct和Plan-and-Execute两种Agent设计模式，分别说明各自的优劣势和适用场景。

**答案：**

| 维度 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| **决策方式** | 逐步决策，每步根据观察调整 | 先全局规划，再按计划执行 |
| **适应性** | 强，能快速应对变化 | 中等，需要显式重规划 |
| **效率** | 可能走弯路 | 路径更直接 |
| **复杂度** | 实现简单 | 实现较复杂 |
| **可预测性** | 低 | 高 |

**适用场景：**

**ReAct更适合：**
```python
# 场景1：探索性任务（不知最终形状）
"帮我找一份最适合我预算的笔记本电脑"
# → 需要根据搜索结果逐步缩小范围

# 场景2：简单对话任务
"今天天气怎么样？" 
# → 1-2步即可完成，无需规划
```

**Plan-and-Execute更适合：**
```python
# 场景1：结构化多步骤任务
"帮我做一份Q4销售分析报告"
# → 先规划：查数据→分析趋势→生成图表→撰写报告

# 场景2：依赖关系明确的复杂任务
"部署一个微服务到K8s"
# → 明确步骤：构建→测试→打包→部署→验证
```

**代码对比：**

```python
# ReAct的步数可能因探索而增加
react_steps = [think, act, observe, think, act, observe, ...]  # 6-10步

# Plan-and-Execute的步数可预估
plan_execute_steps = [plan, step1, step2, step3, finalize]  # 5步
```

### 面试题2：在Plan-and-Execute架构中，如何处理某一步执行失败导致的计划中断？请设计一个完整的容错方案。

**答案：**

```python
class FaultTolerantPlanExecutor:
    """容错计划执行器"""
    
    def __init__(self, max_retries: int = 3):
        self.max_retries = max_retries
        self.replan_threshold = 2  # 连续失败2次则重规划
    
    def execute_with_recovery(self, plan: list, tools: dict) -> dict:
        """带恢复机制的执行"""
        results = {}
        consecutive_failures = 0
        
        for step in plan:
            step_id = step["id"]
            
            # 尝试执行（带重试）
            for attempt in range(self.max_retries):
                try:
                    result = self._execute_single_step(step, tools, results)
                    results[step_id] = {"success": True, "result": result}
                    consecutive_failures = 0
                    break
                    
                except StepError as e:
                    if attempt == self.max_retries - 1:
                        # 方案A：跳过该步骤
                        if step.get("optional", False):
                            results[step_id] = {"success": False, "error": str(e), "skipped": True}
                            break
                        
                        # 方案B：使用降级工具
                        fallback = step.get("fallback_tool")
                        if fallback:
                            try:
                                result = tools[fallback](**step.get("fallback_args", {}))
                                results[step_id] = {"success": True, "result": result, "degraded": True}
                                break
                            except:
                                pass
                        
                        # 方案C：重规划
                        consecutive_failures += 1
                        if consecutive_failures >= self.replan_threshold:
                            return {
                                "status": "needs_replan",
                                "completed_steps": results,
                                "failed_at": step_id,
                                "reason": str(e)
                            }
                        
                        results[step_id] = {"success": False, "error": str(e)}
        
        return {"status": "completed", "results": results}
```

**容错策略层级：**

| 层级 | 策略 | 触发条件 | 恢复时间 |
|------|------|---------|---------|
| L1 | 重试（3次） | 网络/临时故障 | 秒级 |
| L2 | 跳过可选步骤 | 步骤标记为optional | 即时 |
| L3 | 降级工具 | 主工具失败，有备用工具 | 即时 |
| L4 | 重规划 | 连续失败2次 | 秒级 |
| L5 | 人工介入 | 重规划仍失败 | 分钟级 |

## 总结

Plan-and-Execute是构建复杂AI Agent的关键模式：

1. **核心优势**：先规划后执行，减少偏差累积，提高效率
2. **适用场景**：结构化多步骤任务、依赖关系明确的工作流
3. **关键机制**：计划制定 → 逐步执行 → 动态重规划 → 结果汇总
4. **生产建议**：配合重试、降级、缓存机制，构建容错执行器

与ReAct配合使用效果更佳——高层用Plan-and-Execute制定路线，底层用ReAct处理具体交互。

## 参考资料

- [Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601)
