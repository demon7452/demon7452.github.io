---
layout: post
title: "Agent设计模式：ReAct与Tool Use最佳实践"
category: 技术
tags: [AI Agent, ReAct, Tool Use, 设计模式, LLM应用]
keywords: [ReAct, Tool Use, Agent Pattern, Function Calling, LLM Application]
description: "深入讲解ReAct（推理+行动）Agent设计模式，以及工具调用（Tool Use）的最佳实践，帮助构建可靠的生产级AI Agent。"
date: 2026-01-24
---

# Agent设计模式：ReAct与Tool Use最佳实践

## 引言

AI Agent的核心能力之一是**在推理的同时调用外部工具**完成任务。**ReAct（Reasoning + Acting）** 是最经典的Agent设计模式之一，它让模型能够交替进行思维推理和工具调用，形成一个完整的"思考-行动-观察"循环。

本文将深入讲解ReAct模式原理、工具调用最佳实践，并通过完整项目示例展示如何构建生产级Agent。

## ReAct模式核心原理

### "思考-行动-观察"循环

```
用户问题 → 思考(Thought) → 行动(Action) → 观察(Observation) → 思考 → ... → 最终答案
```

**代码实现：**

```python
from typing import TypedDict, Annotated, Sequence
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, ToolMessage
import operator

class ReactState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    max_iterations: int
    current_iteration: int

class ReactAgent:
    """ReAct Agent实现"""
    
    def __init__(self, llm, tools: list):
        self.llm = llm
        self.tools = {tool.name: tool for tool in tools}
        self.llm_with_tools = llm.bind_tools(tools)
    
    def run(self, query: str, max_iterations: int = 5) -> str:
        """执行ReAct循环"""
        messages = [HumanMessage(content=query)]
        
        for i in range(max_iterations):
            # 思考 + 可能调用工具
            response = self.llm_with_tools.invoke(messages)
            messages.append(response)
            
            # 检查是否需要调用工具
            if not response.tool_calls:
                return response.content  # 最终答案
            
            # 执行工具调用
            for tool_call in response.tool_calls:
                tool = self.tools[tool_call["name"]]
                tool_result = tool.invoke(tool_call["args"])
                
                # 将工具结果加入消息
                messages.append(ToolMessage(
                    content=str(tool_result),
                    tool_call_id=tool_call["id"]
                ))
        
        return "已达到最大迭代次数，但未能完成任务。"
```

## 实战：构建多功能文档处理Agent

### 完整项目：SmartDoc Agent

```python
import json
import datetime
from typing import List, Optional
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, ToolMessage

# ========== 1. 定义工具 ==========

@tool
def search_knowledge_base(query: str) -> str:
    """在知识库中搜索文档内容
    
    Args:
        query: 搜索关键词
    """
    # 模拟知识库
    kb = {
        "Python": "Python 3.12已发布，主要特性包括更好的错误信息、f-string改进...",
        "Docker": "Docker Compose v2.24支持GPU资源分配，使用deploy.resources...",
        "Kubernetes": "K8s 1.30版本新增了Sidecar容器特性，简化了日志收集..."
    }
    for key, value in kb.items():
        if key.lower() in query.lower():
            return value
    return "未找到相关信息"

@tool
def read_file(file_path: str) -> str:
    """读取文件内容
    
    Args:
        file_path: 文件路径
    """
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            return f.read()
    except FileNotFoundError:
        return f"文件不存在: {file_path}"

@tool
def calculate(expression: str) -> str:
    """执行数学计算
    
    Args:
        expression: 数学表达式，如 '2 + 3 * 4'
    """
    try:
        result = eval(expression)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {str(e)}"

@tool
def get_current_time() -> str:
    """获取当前时间"""
    return datetime.datetime.now().isoformat()

# ========== 2. ReAct Agent核心 ==========

class SmartDocAgent:
    """智能文档处理Agent"""
    
    def __init__(self, llm=None):
        self.llm = llm or ChatOpenAI(model="gpt-4o", temperature=0)
        self.tools = {
            "search_knowledge_base": search_knowledge_base,
            "read_file": read_file,
            "calculate": calculate,
            "get_current_time": get_current_time
        }
        self.llm_with_tools = self.llm.bind_tools(list(self.tools.values()))
    
    def process(self, query: str, max_steps: int = 5) -> dict:
        """
        执行ReAct循环处理用户请求
        
        Returns:
            {
                "answer": "最终回答",
                "steps": [{"thought": "...", "action": "..."}, ...],
                "tool_calls": [...]
            }
        """
        messages = [HumanMessage(content=query)]
        steps = []
        tool_calls_record = []
        
        for step in range(max_steps):
            response = self.llm_with_tools.invoke(messages)
            messages.append(response)
            
            # 记录当前步骤
            step_record = {
                "step": step + 1,
                "content": response.content if response.content else "(工具调用)"
            }
            
            # 检查是否结束
            if not response.tool_calls:
                step_record["type"] = "final_answer"
                steps.append(step_record)
                return {
                    "answer": response.content,
                    "steps": steps,
                    "tool_calls": tool_calls_record
                }
            
            # 执行工具调用
            step_record["type"] = "tool_call"
            step_record["tools"] = []
            
            for tool_call in response.tool_calls:
                tool_name = tool_call["name"]
                tool_args = tool_call["args"]
                
                tool = self.tools[tool_name]
                
                try:
                    result = tool.invoke(tool_args)
                except Exception as e:
                    result = f"工具调用错误: {str(e)}"
                
                step_record["tools"].append({
                    "name": tool_name,
                    "args": tool_args,
                    "result": result
                })
                
                tool_calls_record.append({
                    "tool": tool_name,
                    "args": tool_args,
                    "result_preview": str(result)[:200]
                })
                
                messages.append(ToolMessage(
                    content=str(result),
                    tool_call_id=tool_call["id"]
                ))
            
            steps.append(step_record)
        
        return {
            "answer": "任务超时，未能在规定步数内完成。",
            "steps": steps,
            "tool_calls": tool_calls_record
        }

# ========== 3. 使用示例 ==========

def demo_react_agent():
    agent = SmartDocAgent()
    
    # 测试1：需要搜索知识库
    print("=== 测试1：知识库搜索 ===")
    result1 = agent.process("Python 3.12有什么新特性？")
    print(f"回答：{result1['answer']}")
    print(f"步骤数：{len(result1['steps'])}")
    print(f"工具调用：{[tc['tool'] for tc in result1['tool_calls']]}")
    
    print("\n=== 测试2：需要计算 ===")
    result2 = agent.process("帮我计算：如果每天存100元，一年（365天）能存多少钱？")
    print(f"回答：{result2['answer']}")
    
    print("\n=== 测试3：不需要工具 ===")
    result3 = agent.process("你好，请介绍一下你自己")
    print(f"回答：{result3['answer']}")
    print(f"工具调用次数：{len(result3['tool_calls'])}")

if __name__ == "__main__":
    demo_react_agent()
```

## 工具设计最佳实践

### 1. 工具命名与描述

```python
# ✅ 推荐：清晰的名称和详细描述
@tool
def get_weather_by_city(city: str, date: Optional[str] = None) -> str:
    """获取指定城市的天气信息。
    
    使用天气预报API查询指定城市在指定日期的天气状况。
    返回温度、湿度、天气状况等信息。
    
    Args:
        city: 城市名称，如'北京'、'上海'
        date: 查询日期，格式'YYYY-MM-DD'，不传则查询今天
    
    Returns:
        天气信息字符串
    """
    pass

# ❌ 不推荐：模糊命名和描述
@tool
def weather(x: str) -> str:
    """查天气"""
    pass
```

### 2. 工具粒度设计

```python
# ✅ 推荐：单一职责工具
@tool
def search_products(query: str) -> str:
    """搜索商品"""
    pass

@tool
def get_product_detail(product_id: str) -> str:
    """获取商品详情"""
    pass

@tool
def check_inventory(product_id: str) -> str:
    """查询库存"""
    pass

# ❌ 不推荐：上帝工具（太大）
@tool
def product_operation(action: str, **kwargs) -> str:
    """处理商品所有操作：搜索/详情/库存/下单/退款"""
    pass
```

### 3. 工具错误处理

```python
from functools import wraps
import time

class ToolExecutor:
    """带错误处理的工具执行器"""
    
    def __init__(self, max_retries: int = 3, timeout: int = 30):
        self.max_retries = max_retries
        self.timeout = timeout
    
    def execute(self, tool_func, args: dict) -> str:
        """执行工具，带重试和超时"""
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                # 超时控制
                import signal
                signal.alarm(self.timeout)
                
                result = tool_func.invoke(args)
                
                signal.alarm(0)  # 取消超时
                return result
                
            except Exception as e:
                last_error = e
                
                if attempt < self.max_retries - 1:
                    # 指数退避重试
                    wait_time = 2 ** attempt
                    time.sleep(wait_time)
        
        # 所有重试都失败
        return json.dumps({
            "error": True,
            "message": f"工具执行失败（尝试{self.max_retries}次）",
            "last_error": str(last_error),
            "suggestion": "请稍后重试或检查工具配置"
        })
```

### 4. 工具结果格式化

```python
@tool
def format_tool_result(result: any) -> str:
    """将工具结果格式化为模型友好的字符串"""
    
    if isinstance(result, dict):
        return json.dumps(result, ensure_ascii=False, indent=2)
    elif isinstance(result, list):
        if len(result) > 5:
            return f"共{len(result)}条结果:\n" + "\n".join(
                f"{i+1}. {item}" for i, item in enumerate(result[:5])
            ) + f"\n...（仅显示前5条）"
        return "\n".join(f"- {item}" for item in result)
    else:
        return str(result)
```

## 高级模式：ReAct with Planning

```python
class PlanAndReactAgent:
    """结合Planning的ReAct Agent"""
    
    def __init__(self, llm, tools: list):
        self.llm = llm
        self.tools = tools
        self.planner_llm = llm  # 可以用不同的模型
    
    def plan(self, user_request: str) -> list:
        """先用LLM制定执行计划"""
        plan_prompt = f"""
        任务：{user_request}
        
        请制定一个分步执行计划：
        1. 分析任务需要哪些步骤
        2. 列出每一步需要调用的工具
        3. 确定步骤之间的依赖关系
        
        输出JSON格式的计划：
        {{
            "steps": [
                {{
                    "id": 1,
                    "description": "步骤描述",
                    "tool": "工具名称（如果需要工具）",
                    "depends_on": []
                }}
            ]
        }}
        """
        
        response = self.planner_llm.invoke(plan_prompt)
        plan = json.loads(response.content)
        return plan["steps"]
    
    def execute_plan(self, plan: list, context: dict = None) -> dict:
        """执行计划"""
        results = {}
        
        for step in plan:
            if step.get("tool"):
                tool_name = step["tool"]
                tool_args = self._resolve_args(step, results, context)
                
                tool = self.tools[tool_name]
                result = tool.invoke(tool_args)
                
                results[f"step_{step['id']}"] = result
        
        return results
    
    def _resolve_args(self, step: dict, results: dict, context: dict) -> dict:
        """解析参数，处理步骤间依赖"""
        # 如果参数中引用了之前步骤的结果，进行替换
        args = step.get("args", {})
        resolved_args = {}
        
        for key, value in args.items():
            if isinstance(value, str) and value.startswith("$step_"):
                step_ref = value[1:]
                resolved_args[key] = results.get(step_ref, value)
            else:
                resolved_args[key] = value
        
        return {**resolved_args, **(context or {})}
```

## 面试题

### 面试题1：请解释ReAct模式的工作原理，并分析它相比纯推理（Reasoning-only）和纯行动（Acting-only）方法的优势。

**答案：**

**ReAct工作原理：**

ReAct将推理（Thought）和行动（Action）交替进行，形成三个阶段的循环：

1. **Thought（思考）**：模型分析当前状态，决定下一步行动
2. **Action（行动）**：调用外部工具获取信息或执行操作
3. **Observation（观察）**：接收工具返回结果，更新状态

```
示例流程：
用户："北京今天天气怎么样？明天需要带伞吗？"
→ Thought: 需要查询今天和明天的天气
→ Action: get_weather("北京", "2026-01-24")
→ Observation: "晴，25°C，湿度40%"
→ Action: get_weather("北京", "2026-01-25")
→ Observation: "雷阵雨，22°C，湿度85%"
→ Thought: 明天有雨，需要带伞
→ Answer: "今天晴天适合出行。明天有雷阵雨，建议带伞。"
```

**三大优势：**

| 对比维度 | ReAct | Reasoning-only | Acting-only |
|---------|-------|----------------|-------------|
| **事实准确性** | 高（工具验证） | 低（可能幻觉） | 低（无推理） |
| **动态信息** | 支持（实时工具查询） | 不支持（知识截止） | 部分支持 |
| **错误恢复** | 能（观察结果调整推理） | 不能 | 能但缺乏判断 |
| **可解释性** | 强（思维链条可见） | 强 | 弱 |

**具体优势分析：**

1. **对抗幻觉**：通过工具获取真实数据，减少模型幻觉
   ```python
   # Reasoning-only: 可能编造天气
   # ReAct: 调用API获取真实天气
   ```

2. **处理动态数据**：能获取实时信息
   ```python
   # ReAct可查询：股票行情、天气、新闻等实时数据
   # Reasoning-only只能基于训练数据
   ```

3. **可调试性**：思维过程透明
   ```python
   # 可以追踪每一步的思考和行动
   for step in agent_result["steps"]:
       print(f"步骤{step['step']}: {step['type']}")
   ```

### 面试题2：在构建生产级AI Agent时，如何处理工具调用的失败和异常？请设计一个完整的容错方案。

**答案：**

**完整容错方案设计：**

```python
class FaultTolerantToolExecutor:
    """容错工具执行器"""
    
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=60
        )
        
    def execute_with_fallback(
        self,
        tool_name: str,
        args: dict,
        fallbacks: list = None
    ) -> str:
        """多级容错执行"""
        
        # 第1级：正常调用
        try:
            if self.circuit_breaker.is_open(tool_name):
                return self._handle_circuit_open(tool_name, args)
            
            result = self._execute_tool(tool_name, args)
            self.circuit_breaker.record_success(tool_name)
            return result
            
        except TemporaryError:
            # 第2级：临时错误 → 重试
            return self._retry_with_backoff(tool_name, args)
            
        except PermanentError:
            # 第3级：永久错误 → 降级
            self.circuit_breaker.record_failure(tool_name)
            return self._degrade(tool_name, args, fallbacks)
            
        except TimeoutError:
            # 第4级：超时 → 异步转同步或返回部分结果
            return self._handle_timeout(tool_name, args)
    
    def _retry_with_backoff(self, tool_name: str, args: dict) -> str:
        """指数退避重试"""
        for attempt in range(3):
            try:
                time.sleep(2 ** attempt)
                return self._execute_tool(tool_name, args)
            except TemporaryError:
                continue
        
        raise PermanentError(f"重试{3}次后仍失败")
    
    def _degrade(self, tool_name: str, args: dict, fallbacks: list) -> str:
        """降级处理"""
        # 策略1：使用降级工具
        if fallbacks:
            for fallback in fallbacks:
                try:
                    return self._execute_tool(fallback, args)
                except:
                    continue
        
        # 策略2：返回缓存（如果有）
        cached = self._get_cached_result(tool_name, args)
        if cached:
            return f"[缓存结果] {cached}"
        
        # 策略3：返回友好错误
        return json.dumps({
            "error": True,
            "message": f"工具 '{tool_name}' 暂时不可用",
            "suggestion": "请尝试简化查询或稍后重试"
        })
```

**四层容错架构图：**

| 层级 | 策略 | 适用场景 | 恢复时间 |
|------|------|---------|---------|
| L1 重试 | 指数退避重试（3次） | 网络抖动、临时故障 | 秒级 |
| L2 降级 | 使用备用工具/简化功能 | 主工具不可用 | 实时 |
| L3 缓存 | 返回历史缓存结果 | 读操作 | 实时 |
| L4 熔断 | 停止调用故障工具 | 持续故障 | 分钟级 |

**监控建议：**

```python
class ToolMetrics:
    def __init__(self):
        self.metrics = {
            "total_calls": 0,
            "success": 0,
            "failure": 0,
            "retries": 0,
            "degradations": 0,
            "latency_p50": [],
            "latency_p99": []
        }
    
    def should_alert(self) -> bool:
        failure_rate = self.metrics["failure"] / max(1, self.metrics["total_calls"])
        return failure_rate > 0.1  # 10%失败率触发告警
```

## 总结

ReAct模式是AI Agent开发的基石，掌握它意味着掌握了构建可靠Agent的核心能力：

1. **ReAct原理**：思考→行动→观察的循环，让模型具备实时信息获取和动态推理能力
2. **工具设计**：单一职责、清晰命名、详细描述、友好错误信息
3. **容错机制**：重试→降级→缓存→熔断的四层防护
4. **最佳实践**：工具粒度适中、结果格式化、参数类型约束

在构建生产级Agent时，建议从简单的ReAct模式开始，逐步引入Plan-and-Execute、Multi-Agent等高级模式。

## 参考资料

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [LangChain Tool Documentation](https://python.langchain.com/docs/modules/tools/)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
