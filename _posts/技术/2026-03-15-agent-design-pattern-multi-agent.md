---
layout: post
title: "Agent设计模式：Multi-Agent协作架构"
category: 技术
tags: [Multi-Agent, Agent协作, AI Agent架构, 任务分发, 角色分工]
keywords: [Multi-Agent System, Agent Collaboration, Task Orchestration, Role-based Agent, LangGraph]
description: "深入讲解Multi-Agent协作架构，通过完整的团队协作系统，演示如何让多个AI Agent分工合作完成复杂任务。"
date: 2026-03-15
---

# Agent设计模式：Multi-Agent协作架构

## 引言

单Agent能处理的任务有限。当面对需要多种技能、多维度分析的复杂任务时，**Multi-Agent协作**是必然选择。就像人类团队有产品经理、工程师、测试一样，AI Agent团队也需要分工协作。

本文将讲解Multi-Agent架构的三种核心模式，并通过完整项目展示如何构建Agent团队。

## 一、Multi-Agent架构模式

### 1.1 三种协作模式

```python
"""
模式1：顺序流水线（Sequential Pipeline）
Agent_A → Agent_B → Agent_C → 最终输出

模式2：路由分发（Router-Dispatcher）
                 ┌→ Agent_A（销售类）
用户输入 → Router ─┼→ Agent_B（技术类）
                 └→ Agent_C（客服类）

模式3：层级协作（Hierarchical Team）
         Supervisor（管理者）
        /    |    \
    Agent1  Agent2  Agent3
"""
```

## 二、实战：软件研发团队Agent

### 完整代码实现

```python
from typing import TypedDict, Annotated, Sequence, List, Literal
import operator
from langgraph.graph import StateGraph, END
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI
import json

# ========== 1. 团队状态定义 ==========
class TeamState(TypedDict):
    """Multi-Agent团队状态"""
    task: str                              # 原始任务
    messages: Annotated[list, operator.add]  # 所有Agent的通信消息
    current_role: str                      # 当前活跃的Agent角色
    code: str                              # 生成的代码
    review_feedback: str                   # 审查反馈
    test_report: str                       # 测试报告
    next_step: str                         # 下一步：continue/review/fix/deploy/end
    final_output: str                      # 最终交付物
    iteration: int                         # 迭代次数

# ========== 2. Agent角色定义 ==========
class AgentRole:
    """Agent角色基类"""
    
    def __init__(self, name: str, role: str, llm: ChatOpenAI):
        self.name = name
        self.role = role
        self.llm = llm
    
    def think(self, task_context: str) -> str:
        """角色思考 - 生成该角色的专业分析"""
        pass
    
    def act(self, task_context: str, **kwargs) -> str:
        """角色行动 - 执行具体操作"""
        pass


class DeveloperAgent(AgentRole):
    """开发者Agent——负责编码"""
    
    def __init__(self, llm):
        super().__init__("Developer", "资深Python开发工程师", llm)
    
    def think(self, task: str) -> str:
        """分析需求，制定技术方案"""
        prompt = f"""你是一位{self.role}。

任务需求：{task}

请完成：
1. 分析技术难点
2. 设计代码结构
3. 列出关键依赖

输出清晰的技术方案。
"""
        return self.llm.invoke(prompt).content
    
    def act(self, task: str, tech_plan: str = "") -> str:
        """根据需求编写代码"""
        prompt = f"""你是一位{self.role}。

任务需求：{task}

技术方案：{tech_plan}

请编写完整的、可运行的Python代码。要求：
1. 包含完整的类型注解
2. 包含错误处理
3. 包含使用示例
4. 代码注释清晰

只输出代码，不要解释。
"""
        response = self.llm.invoke(prompt)
        # 提取代码块
        code = response.content
        if "```python" in code:
            code = code.split("```python")[1].split("```")[0].strip()
        return code


class ReviewerAgent(AgentRole):
    """审查者Agent——负责代码审查"""
    
    def __init__(self, llm):
        super().__init__("Reviewer", "资深代码审查专家", llm)
    
    def think(self, code: str) -> str:
        """代码分析"""
        pass
    
    def act(self, code: str, task: str) -> str:
        """审查代码"""
        prompt = f"""你是一位{self.role}。

原始任务：{task}

待审查代码：
```python
{code}
```

请从以下维度审查：
1. 功能正确性：代码是否满足需求？
2. 代码质量：命名、结构、可读性
3. 错误处理：是否有充分的异常处理？
4. 安全性：是否存在安全漏洞？
5. 性能：是否有明显的性能问题？

输出格式：
审查结果：[通过 / 需要修改]
问题列表：
- [严重程度] 问题描述（建议修复方案）
- ...
"""
        return self.llm.invoke(prompt).content


class TesterAgent(AgentRole):
    """测试者Agent——负责测试"""
    
    def __init__(self, llm):
        super().__init__("Tester", "资深测试工程师", llm)
    
    def act(self, code: str, task: str) -> str:
        """编写和执行测试"""
        prompt = f"""你是一位{self.role}。

原始任务：{task}

待测试代码：
```python
{code}
```

请：
1. 编写完整的pytest测试用例
2. 覆盖正常路径和边界情况
3. 包含集成测试
4. 给出预期测试结果

输出：
- 测试用例代码
- 预期通过/失败的用例数
- 代码覆盖率预估
"""
        return self.llm.invoke(prompt).content


class SupervisorAgent(AgentRole):
    """主管Agent——负责协调和决策"""
    
    def __init__(self, llm):
        super().__init__("Supervisor", "技术主管", llm)
    
    def act(self, state: dict) -> dict:
        """根据当前状态决策下一步"""
        prompt = f"""你是一位{self.role}。当前项目状态：

任务：{state.get('task', '')}
当前角色：{state.get('current_role', 'start')}
已有代码：{state.get('code', '无')[:200]}
审查反馈：{state.get('review_feedback', '无')[:200]}
测试报告：{state.get('test_report', '无')[:200]}
迭代次数：{state.get('iteration', 0)}

请决策下一步：
- develop：需要开发者编写代码
- review：需要审查代码
- test：需要运行测试
- fix：需要开发者修复问题
- end：任务完成

只回复一个词：develop/review/test/fix/end
"""
        response = self.llm.invoke(prompt)
        return response.content.strip().lower()

# ========== 3. Multi-Agent工作流 ==========
class SoftwareTeamGraph:
    """软件开发团队Multi-Agent工作流"""
    
    def __init__(self, max_iterations: int = 5):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
        self.developer = DeveloperAgent(self.llm)
        self.reviewer = ReviewerAgent(self.llm)
        self.tester = TesterAgent(self.llm)
        self.supervisor = SupervisorAgent(self.llm)
        self.max_iterations = max_iterations
        self.graph = self._build_graph()
    
    def _build_graph(self):
        """构建Multi-Agent状态图"""
        workflow = StateGraph(TeamState)
        
        workflow.add_node("supervisor", self._supervisor_node)
        workflow.add_node("developer", self._developer_node)
        workflow.add_node("reviewer", self._reviewer_node)
        workflow.add_node("tester", self._tester_node)
        workflow.add_node("finalizer", self._finalizer_node)
        
        workflow.set_entry_point("supervisor")
        
        # 主管路由到各Agent
        workflow.add_conditional_edges(
            "supervisor",
            self._supervisor_router,
            {
                "develop": "developer",
                "review": "reviewer",
                "test": "tester",
                "fix": "developer",
                "end": "finalizer"
            }
        )
        
        # 各Agent执行后回到主管
        workflow.add_edge("developer", "supervisor")
        workflow.add_edge("reviewer", "supervisor")
        workflow.add_edge("tester", "supervisor")
        workflow.add_edge("finalizer", END)
        
        return workflow.compile()
    
    def _supervisor_node(self, state: TeamState) -> TeamState:
        """主管节点：分析状态并决策"""
        
        # 检查迭代次数
        if state.get("iteration", 0) >= self.max_iterations:
            return {"next_step": "end"}
        
        decision = self.supervisor.act(state)
        
        return {
            "next_step": decision,
            "current_role": "supervisor",
            "iteration": state.get("iteration", 0) + 1
        }
    
    def _supervisor_router(self, state: TeamState) -> str:
        """主管路由判断"""
        return state.get("next_step", "end")
    
    def _developer_node(self, state: TeamState) -> TeamState:
        """开发者节点"""
        task = state.get("task", "")
        existing_code = state.get("code", "")
        review_feedback = state.get("review_feedback", "")
        
        if not existing_code:
            # 首次开发
            tech_plan = self.developer.think(task)
            code = self.developer.act(task, tech_plan)
            
            return {
                "code": code,
                "current_role": "developer",
                "messages": [AIMessage(content=f"[Developer] 已完成代码编写\n代码长度：{len(code)}字符")]
            }
        else:
            # 修复模式
            fix_prompt = f"""根据审查意见修复代码：

审查意见：{review_feedback}

原代码：
{existing_code}

请输出修复后的完整代码。
"""
            response = self.llm.invoke(fix_prompt)
            code = response.content
            if "```python" in code:
                code = code.split("```python")[1].split("```")[0].strip()
            
            return {
                "code": code,
                "current_role": "developer",
                "messages": [AIMessage(content=f"[Developer] 已根据反馈修复代码")]
            }
    
    def _reviewer_node(self, state: TeamState) -> TeamState:
        """审查者节点"""
        code = state.get("code", "")
        task = state.get("task", "")
        
        if not code:
            return {
                "review_feedback": "没有代码可供审查",
                "current_role": "reviewer"
            }
        
        feedback = self.reviewer.act(code, task)
        
        return {
            "review_feedback": feedback,
            "current_role": "reviewer",
            "messages": [AIMessage(content=f"[Reviewer] {feedback[:200]}")]
        }
    
    def _tester_node(self, state: TeamState) -> TeamState:
        """测试者节点"""
        code = state.get("code", "")
        task = state.get("task", "")
        
        if not code:
            return {
                "test_report": "没有代码可供测试",
                "current_role": "tester"
            }
        
        test_report = self.tester.act(code, task)
        
        return {
            "test_report": test_report,
            "current_role": "tester",
            "messages": [AIMessage(content=f"[Tester] {test_report[:200]}")]
        }
    
    def _finalizer_node(self, state: TeamState) -> TeamState:
        """最终汇总节点"""
        final_output = f"""# 软件研发团队交付报告

## 原始需求
{state.get('task', '')}

## 交付代码
```python
{state.get('code', '')}
```

## 审查报告
{state.get('review_feedback', '无')}

## 测试报告
{state.get('test_report', '无')}

## 统计
- 迭代次数：{state.get('iteration', 0)}
- 参与角色：Developer, Reviewer, Tester
"""
        
        return {
            "final_output": final_output,
            "messages": [AIMessage(content="[Supervisor] 任务完成")]
        }
    
    def run(self, task: str) -> str:
        """运行Multi-Agent团队"""
        initial_state = {
            "task": task,
            "messages": [HumanMessage(content=task)],
            "iteration": 0
        }
        
        result = self.graph.invoke(initial_state)
        return result["final_output"]

# ========== 4. 使用示例 ==========
def demo():
    team = SoftwareTeamGraph(max_iterations=5)
    
    result = team.run("""
    开发一个函数，接收一个包含交易记录的CSV文件路径，
    返回以下统计数据：
    1. 总交易额
    2. 平均交易额
    3. 交易数量
    4. 最高单笔交易
    
    要求：
    - 处理CSV可能不存在的情况
    - 处理CSV格式错误
    - 使用pandas库
    - 包含完整的类型注解
    """)
    
    print(result)

if __name__ == "__main__":
    demo()
```

## 三、Multi-Agent通信机制

### 3.1 Agent间消息传递

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class MessagePriority(Enum):
    HIGH = 1    # 紧急（如错误）
    NORMAL = 2  # 正常通信
    LOW = 3     # 低优先级（如进度更新）

@dataclass
class AgentMessage:
    """Agent间消息"""
    sender: str        # 发送者
    receiver: str      # 接收者（None=广播）
    content: str       # 消息内容
    priority: MessagePriority
    requires_response: bool = False
    context: dict = None  # 附加上下文

class AgentCommunicationBus:
    """Agent通信总线"""
    
    def __init__(self):
        self.message_queue = []
        self.agents = {}
    
    def register_agent(self, name: str, agent):
        """注册Agent"""
        self.agents[name] = agent
    
    def send(self, message: AgentMessage):
        """发送消息"""
        self.message_queue.append(message)
        self.message_queue.sort(key=lambda m: m.priority.value)
    
    def receive(self, agent_name: str) -> list:
        """接收消息"""
        messages = []
        
        for msg in self.message_queue[:]:
            if msg.receiver == agent_name or msg.receiver is None:
                messages.append(msg)
                self.message_queue.remove(msg)
        
        return messages
    
    def broadcast(self, sender: str, content: str):
        """广播消息"""
        self.send(AgentMessage(
            sender=sender,
            receiver=None,
            content=content,
            priority=MessagePriority.NORMAL
        ))
```

### 3.2 任务委托与反馈循环

```python
class TaskDelegator:
    """任务委托系统"""
    
    def __init__(self):
        self.agent_capabilities = {}
    
    def register_capability(
        self, 
        agent_name: str, 
        skills: list, 
        capacity: int = 5
    ):
        """注册Agent能力"""
        self.agent_capabilities[agent_name] = {
            "skills": skills,
            "capacity": capacity,
            "current_load": 0,
            "completed": 0,
            "failed": 0
        }
    
    def find_best_agent(self, task: str) -> list:
        """找到最适合的Agent"""
        from langchain_openai import OpenAIEmbeddings
        from sklearn.metrics.pairwise import cosine_similarity
        
        embeddings = OpenAIEmbeddings()
        task_emb = embeddings.embed_query(task)
        
        scores = []
        for name, caps in self.agent_capabilities.items():
            if caps["current_load"] >= caps["capacity"]:
                continue  # 跳过负载已满的Agent
            
            skills_text = " ".join(caps["skills"])
            skill_emb = embeddings.embed_query(skills_text)
            
            similarity = cosine_similarity([task_emb], [skill_emb])[0][0]
            load_penalty = caps["current_load"] / max(caps["capacity"], 1) * 0.2
            
            scores.append({
                "agent": name,
                "score": similarity - load_penalty,
                "load": caps["current_load"],
                "skills": caps["skills"]
            })
        
        scores.sort(key=lambda x: x["score"], reverse=True)
        return scores
```

## 面试题

### 面试题1：在Multi-Agent系统中，如何解决Agent之间的通信和协调问题？请设计一个通信架构。

**答案：**

**通信架构设计：**

```
┌─────────────────────────────────────────┐
│            Message Bus (消息总线)         │
│  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │ Queue │  │ Pub/ │  │ RPC  │          │
│  │       │  │ Sub  │  │      │          │
│  └──────┘  └──────┘  └──────┘          │
└─────────────────────────────────────────┘
        │          │          │
   ┌────▼──┐  ┌───▼───┐  ┌──▼─────┐
   │Agent A│  │Agent B│  │Agent C │
   └───────┘  └───────┘  └────────┘
```

**三种通信模式对比：**

| 模式 | 机制 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| 消息队列 | FIFO队列 | 解耦、可靠 | 延迟较高 | 异步任务 |
| 发布订阅 | 主题订阅 | 广播、灵活 | 难追踪 | 状态更新 |
| RPC | 直接调用 | 低延迟、同步 | 紧耦合 | 子任务委托 |

**解决协调问题：**

```python
class CoordinationManager:
    """协调管理器"""
    
    def __init__(self):
        self.agents = {}
        self.shared_context = {}
        self.locks = {}
        self.task_dependencies = {}
    
    def acquire_lock(self, resource: str, agent_name: str) -> bool:
        """获取共享资源锁（避免冲突）"""
        if resource not in self.locks:
            self.locks[resource] = agent_name
            return True
        return False
    
    def update_shared_context(self, agent_name: str, key: str, value):
        """更新共享上下文（信息同步）"""
        self.shared_context[key] = {
            "value": value,
            "updated_by": agent_name,
            "timestamp": time.time()
        }
    
    def wait_for_dependency(self, task_id: str, depends_on: list) -> bool:
        """等待依赖任务完成"""
        for dep_id in depends_on:
            if self.task_dependencies.get(dep_id) != "completed":
                return False
        return True
```

### 面试题2：请设计一个Multi-Agent系统处理客户投诉，至少包含3个专业Agent角色，并说明它们如何协作。

**答案：**

```python
class ComplaintHandlingTeam:
    """客户投诉处理团队"""
    
    def __init__(self):
        self.classifier = ClassifierAgent()    # 分类Agent
        self.investigator = InvestigatorAgent()  # 调查Agent
        self.responder = ResponderAgent()       # 回复Agent
        self.supervisor = SupervisorAgent()     # 审核Agent
    
    def handle_complaint(self, complaint: str, customer_history: dict) -> str:
        """
        处理投诉流程：
        1. Classifier: 分类投诉类型和紧急程度
        2. Investigator: 调查问题（查订单、日志）
        3. Responder: 生成回复草案
        4. Supervisor: 审核回复（涉及赔偿时需要）
        """
        
        # Step 1: 分类
        classification = self.classifier.act(complaint)
        # 返回：{"type": "refund", "severity": "high", "category": "product_quality"}
        
        # Step 2: 调查
        investigation = self.investigator.act(
            complaint=complaint,
            customer_history=customer_history,
            classification=classification
        )
        
        # Step 3: 生成回复
        draft_response = self.responder.act(
            complaint=complaint,
            investigation=investigation
        )
        
        # Step 4: 审核（高严重度需要）
        if classification["severity"] == "high":
            approval = self.supervisor.act(draft_response, investigation)
            
            if not approval["approved"]:
                # 重新生成回复
                draft_response = self.responder.act(
                    complaint=complaint,
                    investigation=investigation,
                    supervisor_feedback=approval["feedback"]
                )
        
        return draft_response


class ClassifierAgent:
    """分类Agent：分析投诉类型"""
    
    def act(self, complaint: str) -> dict:
        prompt = f"""分析以下客户投诉并分类：

投诉内容：{complaint}

输出JSON：
{{
    "type": "refund/exchange/complaint/inquiry",
    "severity": "high/medium/low",
    "category": "product_quality/delivery_delay/customer_service/other",
    "sentiment": "angry/disappointed/neutral",
    "requires_compensation": true/false,
    "reason": "分类依据"
}}
"""
        # LLM调用...
        pass


class InvestigatorAgent:
    """调查Agent：查订单、查询问题原因"""
    
    def act(self, complaint: str, customer_history: dict, classification: dict) -> dict:
        # 查询订单
        order = self._query_order_system(customer_history.get("order_id"))
        
        # 查询日志
        logs = self._query_delivery_logs(customer_history.get("order_id"))
        
        return {
            "order_status": order.get("status"),
            "delivery_status": logs.get("delivery_status"),
            "issue_found": True,
            "root_cause": "配送延迟是由于仓库库存不足",
            "recommended_action": "退款+优惠券补偿"
        }
    
    def _query_order_system(self, order_id: str) -> dict:
        pass
    
    def _query_delivery_logs(self, order_id: str) -> dict:
        pass


class ResponderAgent:
    """回复Agent：生成专业回复"""
    
    def act(self, complaint: str, investigation: dict, 
            supervisor_feedback: str = "") -> str:
        prompt = f"""你是一位客户服务专家。

投诉内容：{complaint}
调查结果：{investigation}
{"主管反馈：" + supervisor_feedback if supervisor_feedback else ""}

请生成专业回复：
1. 道歉并共情
2. 说明调查结果
3. 提出解决方案
4. 提供后续保障

回复要真诚、专业、具体。
"""
        pass
```

**协作流程总结：**

1. **Classifier → Investigator**：分类结果指导调查方向
2. **Investigator → Responder**：调查结果为回复提供事实依据
3. **Responder → Supervisor**：高风险回复需人工审核
4. **Supervisor → Responder**：反馈驱动回复优化

## 总结

Multi-Agent协作是构建复杂AI系统的关键能力：

1. **三种协作模式**：流水线、路由分发、层级协作
2. **角色设计**：每个Agent有明确的职责和专业能力
3. **通信机制**：消息总线 + 共享上下文 + 任务委托
4. **容错设计**：反馈循环、人工审核、降级策略

建议从小团队（2-3个Agent）开始，逐步扩展到更复杂的协作场景。

## 参考资料

- [AutoGen Multi-Agent Framework](https://github.com/microsoft/autogen)
- [LangGraph Multi-Agent Examples](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/)
- [CrewAI](https://github.com/joaomdmoura/crewAI)
