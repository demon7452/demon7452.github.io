---
layout: post
title: "LangGraph深入：循环节点与checkpoint机制"
category: 技术
tags: [LangGraph, Agent, Checkpoint, 持久化, 断点续传]
keywords: [LangGraph, Checkpoint, Persistence, State Graph, Agent Memory, 断点续传]
description: "深入讲解LangGraph中的循环节点设计和checkpoint持久化机制，实现Agent的状态保存与断点续传功能。"
date: 2026-03-01
---

# LangGraph深入：循环节点与checkpoint机制

## 引言

在构建生产级Agent时，**状态持久化**和**断点续传**是必不可少的能力。当Agent因网络中断、模型超时或用户暂停而停止时，checkpoint机制能确保状态不丢失，从断点继续执行。LangGraph内置了强大的checkpoint系统，本文将深入讲解循环节点的设计和checkpoint机制。

## 一、循环节点（Loop Node）

### 1.1 循环节点的典型场景

```python
# 场景1：迭代优化（代码Review Agent）
"""
Node: Review Code → Loop(需要修改？) → Node: Fix Code → Loop(需要修改？) → END
"""

# 场景2：多步骤累积（数据收集Agent）  
"""
Node: Search → Loop(更多数据？) → Node: Filter → Loop(更多数据？) → END
"""

# 场景3：自修正（Self-Refine）
"""
Node: Generate → Loop(需要改进？) → Node: Critique → Node: Refine → Loop → END
```
```

### 1.2 实战：代码审查Agent（带循环）

```python
from typing import TypedDict, Annotated, Sequence
import operator
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI

# ========== 状态定义 ==========
class CodeReviewState(TypedDict):
    code: str                          # 原始代码
    review_comments: Annotated[list, operator.add]  # 审查意见（累加）
    fixed_code: str                    # 修复后的代码
    iteration: int                     # 当前迭代次数
    max_iterations: int                # 最大迭代次数
    issues_found: int                  # 发现的问题数
    final_result: str                  # 最终结果

class CodeReviewAgent:
    """带循环迭代的代码审查Agent"""
    
    def __init__(self, max_iterations: int = 3):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
        self.max_iterations = max_iterations
        self.graph = self._build_graph()
    
    def _build_graph(self):
        workflow = StateGraph(CodeReviewState)
        
        # 添加节点
        workflow.add_node("review", self._review_code)
        workflow.add_node("fix", self._fix_code)
        workflow.add_node("summarize", self._summarize)
        
        workflow.set_entry_point("review")
        
        # 条件边：需要修复则循环
        workflow.add_conditional_edges(
            "review",
            self._should_fix,
            {
                "fix": "fix",           # 发现问题 → 修复
                "done": "summarize",    # 无问题/达到上限 → 总结
            }
        )
        
        workflow.add_edge("fix", "review")     # 修复后重新审查（循环）
        workflow.add_edge("summarize", END)
        
        # 使用checkpointer
        memory = MemorySaver()
        return workflow.compile(checkpointer=memory)
    
    def _review_code(self, state: CodeReviewState) -> CodeReviewState:
        """审查代码"""
        prompt = f"""你是一位资深代码审查者。请审查以下代码。

代码：
{state['code']}

之前发现的已修复问题：
{state.get('review_comments', [])}

请按以下维度审查：
1. Bug和逻辑错误
2. 性能问题
3. 安全问题
4. 代码可读性
5. 最佳实践遵循

如果代码已经完美，请明确说明"代码质量优秀，无需修改"。

输出格式：
发现问题数：N
详细意见：
- [严重程度] 问题描述 (位置: 第X行)
- ...
"""
        
        response = self.llm.invoke(prompt)
        
        # 解析问题数
        content = response.content
        issues_found = 0
        if "发现问题数：" in content:
            try:
                issues_found = int(content.split("发现问题数：")[1].split("\n")[0].strip())
            except:
                issues_found = 0
        
        return {
            "review_comments": [content],
            "issues_found": issues_found,
            "iteration": state.get("iteration", 0) + 1
        }
    
    def _should_fix(self, state: CodeReviewState) -> str:
        """判断是否需要修复"""
        
        # 条件1：无问题 → 结束
        if state["issues_found"] == 0:
            return "done"
        
        # 条件2：超过最大迭代次数 → 结束
        if state["iteration"] >= state.get("max_iterations", self.max_iterations):
            return "done"
        
        # 条件3：有问题且未超过上限 → 修复
        return "fix"
    
    def _fix_code(self, state: CodeReviewState) -> CodeReviewState:
        """根据审查意见修复代码"""
        prompt = f"""根据以下审查意见修复代码。

原始代码：
{state['code']}

审查意见：
{state['review_comments'][-1] if state['review_comments'] else '无'}

请生成修复后的完整代码。只输出代码，不要解释。
"""
        
        response = self.llm.invoke(prompt)
        
        return {
            "fixed_code": response.content,
            "code": response.content  # 更新代码供下一轮审查
        }
    
    def _summarize(self, state: CodeReviewState) -> CodeReviewState:
        """生成审查总结"""
        summary = f"""代码审查完成

迭代次数：{state["iteration"]}
总发现问题数：{state["issues_found"]}
最终代码质量：{"通过" if state["issues_found"] == 0 else "需要人工复审"}

审查历史：
{chr(10).join(f"--- 第{i+1}轮 ---{chr(10)}{c}" for i, c in enumerate(state['review_comments']))}
"""
        
        return {"final_result": summary}
    
    def review(self, code: str, thread_id: str = "default") -> str:
        """执行代码审查（支持断点续传）"""
        config = {"configurable": {"thread_id": thread_id}}
        
        initial_state = {
            "code": code,
            "review_comments": [],
            "max_iterations": self.max_iterations
        }
        
        final_state = self.graph.invoke(initial_state, config)
        return final_state["final_result"]

# 使用示例
agent = CodeReviewAgent(max_iterations=3)

code = """
def calculate_average(numbers):
    sum = 0
    for i in range(len(numbers)):
        sum = sum + numbers[i]
    return sum / len(numbers)

def process_data(data):
    result = []
    for item in data:
        result.append(item * 2)
    return result
"""

result = agent.review(code, thread_id="review-001")
print(result)
```

## 二、Checkpoint机制详解

### 2.1 Checkpoint类型

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver
import sqlite3

# ========== 1. 内存Checkpoint（开发和测试用） ==========
memory_checkpointer = MemorySaver()
# 优点：速度最快，零配置
# 缺点：进程重启丢失

# ========== 2. SQLite Checkpoint（单机生产用） ==========
sqlite_checkpointer = SqliteSaver(
    sqlite3.connect("agent_checkpoints.db", check_same_thread=False)
)
# 优点：持久化存储，单文件部署
# 缺点：不支持并发写入

# ========== 3. PostgreSQL Checkpoint（分布式生产用） ==========
# postgres_checkpointer = PostgresSaver(
#     connection_string="postgresql://user:pass@localhost:5432/agent_db"
# )
# 优点：支持高并发，分布式部署
# 缺点：需要独立数据库
```

### 2.2 自定义Checkpoint后端

```python
from langgraph.checkpoint.base import BaseCheckpointSaver, CheckpointTuple
from typing import AsyncIterator, Iterator, Optional
import json

class RedisCheckpointSaver(BaseCheckpointSaver):
    """基于Redis的自定义Checkpoint"""
    
    def __init__(self, redis_client, ttl: int = 86400 * 7):
        super().__init__()
        self.redis = redis_client
        self.ttl = ttl  # 7天过期
    
    def _key(self, thread_id: str, checkpoint_ns: str = "") -> str:
        return f"checkpoint:{thread_id}:{checkpoint_ns}"
    
    def get_tuple(self, config: dict) -> Optional[CheckpointTuple]:
        """获取checkpoint"""
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config.get("configurable", {}).get("checkpoint_ns", "")
        
        data = self.redis.get(self._key(thread_id, checkpoint_ns))
        if not data:
            return None
        
        checkpoint_data = json.loads(data)
        return CheckpointTuple(
            config=config,
            checkpoint=checkpoint_data["checkpoint"],
            metadata=checkpoint_data.get("metadata", {}),
            parent_config=checkpoint_data.get("parent_config")
        )
    
    def put(
        self,
        config: dict,
        checkpoint: dict,
        metadata: dict,
        new_versions: dict
    ) -> dict:
        """保存checkpoint"""
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config.get("configurable", {}).get("checkpoint_ns", "")
        
        data = {
            "checkpoint": checkpoint,
            "metadata": metadata,
            "parent_config": getattr(self, "_parent_config", None)
        }
        
        self.redis.setex(
            self._key(thread_id, checkpoint_ns),
            self.ttl,
            json.dumps(data)
        )
        
        return config
    
    def list(
        self,
        config: Optional[dict] = None,
        *,
        filter: Optional[dict] = None,
        before: Optional[dict] = None,
        limit: Optional[int] = None
    ) -> Iterator[CheckpointTuple]:
        """列出checkpoints（简化实现）"""
        if config:
            thread_id = config["configurable"]["thread_id"]
            data = self.redis.get(self._key(thread_id))
            if data:
                checkpoint_data = json.loads(data)
                yield CheckpointTuple(
                    config=config,
                    checkpoint=checkpoint_data["checkpoint"],
                    metadata=checkpoint_data.get("metadata", {})
                )
```

### 2.3 Checkpoint实战应用

```python
class CheckpointManager:
    """Checkpoint管理器 —— 提供断点续传和管理功能"""
    
    def __init__(self, checkpointer):
        self.checkpointer = checkpointer
    
    def list_all_checkpoints(self, thread_id: str) -> list:
        """列出某个线程的所有checkpoint"""
        config = {"configurable": {"thread_id": thread_id}}
        
        checkpoints = []
        for cp in self.checkpointer.list(config):
            checkpoints.append({
                "checkpoint_id": cp.checkpoint.get("id"),
                "timestamp": cp.metadata.get("timestamp"),
                "step": cp.metadata.get("step"),
                "source": cp.metadata.get("source"),
            })
        
        return checkpoints
    
    def rollback_to_checkpoint(self, thread_id: str, step: int) -> dict:
        """回滚到指定步骤的checkpoint"""
        checkpoints = self.list_all_checkpoints(thread_id)
        
        target = None
        for cp in checkpoints:
            if cp["step"] == step:
                target = cp
                break
        
        if not target:
            raise ValueError(f"未找到step={step}的checkpoint")
        
        config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_id": target["checkpoint_id"]
            }
        }
        
        return self.checkpointer.get_tuple(config)
    
    def export_checkpoints(self, thread_id: str, output_path: str):
        """导出checkpoint到文件"""
        config = {"configurable": {"thread_id": thread_id}}
        
        checkpoints_data = []
        for cp in self.checkpointer.list(config):
            checkpoints_data.append({
                "checkpoint": cp.checkpoint,
                "metadata": cp.metadata
            })
        
        with open(output_path, "w", encoding="utf-8") as f:
            json.dump(checkpoints_data, f, ensure_ascii=False, indent=2)
        
        return f"已导出{len(checkpoints_data)}个checkpoint到{output_path}"
    
    def import_checkpoints(self, input_path: str):
        """从文件导入checkpoint"""
        with open(input_path, "r", encoding="utf-8") as f:
            checkpoints_data = json.load(f)
        
        for cp_data in checkpoints_data:
            config = cp_data["metadata"].get("config", {})
            self.checkpointer.put(
                config=config,
                checkpoint=cp_data["checkpoint"],
                metadata=cp_data["metadata"],
                new_versions={}
            )
        
        return f"已导入{len(checkpoints_data)}个checkpoint"

# ========== 实战：带Checkpoint的多轮对话Agent ==========
class CheckpointedConversationAgent:
    """支持断点续传的对话Agent"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0.7)
        self.checkpoint_manager = CheckpointManager(MemorySaver())
        self.graph = self._build_conversation_graph()
    
    def _build_conversation_graph(self):
        from langgraph.graph import StateGraph, END
        
        class ConversationState(TypedDict):
            messages: Annotated[list, operator.add]
            context: dict
            current_topic: str
            summary: str
        
        workflow = StateGraph(ConversationState)
        
        workflow.add_node("process_message", self._process_message)
        workflow.add_node("update_context", self._update_context)
        workflow.add_node("generate_response", self._generate_response)
        
        workflow.set_entry_point("process_message")
        workflow.add_edge("process_message", "update_context")
        workflow.add_edge("update_context", "generate_response")
        workflow.add_edge("generate_response", END)
        
        return workflow.compile(checkpointer=self.checkpoint_manager.checkpointer)
    
    def send_message(self, message: str, conversation_id: str) -> str:
        """发送消息（自动checkpoint）"""
        config = {"configurable": {"thread_id": conversation_id}}
        
        state = {"messages": [HumanMessage(content=message)]}
        result = self.graph.invoke(state, config)
        
        return result["messages"][-1].content
    
    def resume_conversation(self, conversation_id: str) -> str:
        """恢复对话（从checkpoint）"""
        config = {"configurable": {"thread_id": conversation_id}}
        
        checkpoints = self.checkpoint_manager.list_all_checkpoints(conversation_id)
        if not checkpoints:
            return "未找到历史对话记录"
        
        latest = checkpoints[-1]
        return f"已恢复对话，最后更新时间：{latest['timestamp']}，已进行{latest['step']}步"
    
    def _process_message(self, state):
        return state
    
    def _update_context(self, state):
        return state
    
    def _generate_response(self, state):
        response = self.llm.invoke(state["messages"])
        return {"messages": [response]}
```

## 三、最佳实践

### 1. Checkpoint序列化

```python
class SerializableCheckpoint:
    """确保checkpoint数据可序列化"""
    
    @staticmethod
    def sanitize_state(state: dict) -> dict:
        """清理状态，确保可JSON序列化"""
        import datetime
        
        def serialize_value(v):
            if isinstance(v, datetime.datetime):
                return v.isoformat()
            if isinstance(v, BaseMessage):
                return {"type": type(v).__name__, "content": v.content}
            if isinstance(v, (list, tuple)):
                return [serialize_value(item) for item in v]
            if isinstance(v, dict):
                return {k: serialize_value(val) for k, val in v.items()}
            return v
        
        return serialize_value(state)
```

### 2. Checkpoint策略选择

| 场景 | 推荐方案 | Checkpoint频率 |
|------|---------|---------------|
| 开发调试 | MemorySaver | 每个节点后 |
| 单机生产 | SQLite | 关键节点后 |
| 分布式 | PostgreSQL/Redis | 可配置间隔 |
| 长任务(>10min) | PostgreSQL + S3备份 | 每N步 + 异常时 |

## 面试题

### 面试题1：LangGraph中checkpoint的实现原理是什么？它如何保证断点续传的可靠性？

**答案：**

**核心原理：**

```python
# LangGraph在每个节点执行后自动保存checkpoint
"""
执行流程：
Node_A执行 → 保存checkpoint_1 → 边路由 → 
Node_B执行 → 保存checkpoint_2 → 边路由 → 
...（异常中断）...

续传流程：
加载checkpoint_2 → 恢复状态 → 从Node_B之后继续执行
"""
```

**可靠性保证机制：**

1. **原子性写入**：checkpoint在当前节点完全执行后才保存，避免部分执行状态
2. **版本控制**：每个checkpoint关联parent checkpoint，形成版本链
3. **幂等设计**：重放相同状态产生相同结果

**源码级原理：**

```python
# LangGraph内部简化逻辑
class PregelLoop:
    def __init__(self, graph, checkpointer):
        self.checkpointer = checkpointer
    
    def invoke(self, input_state, config):
        # 1. 尝试加载已有checkpoint
        saved = self.checkpointer.get_tuple(config)
        if saved:
            state = saved.checkpoint["channel_values"]
            next_nodes = saved.metadata.get("next_nodes", [])
        else:
            state = input_state
            next_nodes = [graph.entry_point]
        
        # 2. 执行步骤
        while next_nodes:
            for node in next_nodes:
                # 执行节点
                state = graph.nodes[node].invoke(state)
                
                # 保存checkpoint
                self.checkpointer.put(
                    config=config,
                    checkpoint={"channel_values": state},
                    metadata={
                        "step": current_step,
                        "source": node,
                        "parent_checkpoint": parent_id
                    }
                )
            
            # 3. 计算下一步节点
            next_nodes = self._get_next_nodes(state)
```

**保证可靠性的三个关键设计：**

| 机制 | 作用 |
|------|------|
| 每步保存 | 最小化丢失量（最多丢失1步） |
| parent链 | 支持时间旅行和回滚 |
| 幂等执行 | 重放不产生副作用 |

### 面试题2：在循环节点中如何防止无限循环？请给出至少三种防护措施。

**答案：**

```python
class LoopGuard:
    """循环防护机制"""
    
    def __init__(self):
        # 措施1：最大迭代次数限制
        self.max_iterations = 10
        
        # 措施2：状态变化检测
        self.previous_states = []
        
        # 措施3：超时控制
    
    def should_continue(self, state: dict, iteration: int) -> tuple:
        """判断是否继续循环，返回 (是否继续, 停止原因)"""
        
        # 1. 迭代次数限制
        if iteration >= self.max_iterations:
            return False, f"达到最大迭代次数{self.max_iterations}"
        
        # 2. 状态变化检测（防震荡）
        state_hash = self._hash_state(state)
        self.previous_states.append(state_hash)
        
        if len(self.previous_states) > 3:
            last_3 = self.previous_states[-3:]
            if len(set(last_3)) <= 2:  # 最近3次中有重复
                return False, "检测到状态震荡（重复状态）"
        
        # 3. 收敛检测
        if self._is_converged(state):
            return False, "状态已收敛"
        
        # 4. 关键指标恶化检测
        if self._is_degrading(state):
            return False, "检测到质量下降"
        
        return True, None
    
    def _hash_state(self, state: dict) -> str:
        """状态哈希"""
        import hashlib
        state_str = json.dumps(state, sort_keys=True, default=str)
        return hashlib.md5(state_str.encode()).hexdigest()
    
    def _is_converged(self, state: dict) -> bool:
        """收敛检测"""
        # 例如：代码审查中连续两次无新问题
        return state.get("issues_found", 1) == 0
    
    def _is_degrading(self, state: dict) -> bool:
        """质量退化检测"""
        # 例如：代码修复后问题反而增多
        pass
```

**三种防护措施总结：**

| 措施 | 实现方式 | 适用场景 |
|------|---------|---------|
| 最大迭代次数 | `max_iterations` 硬限制 | 所有场景（必须） |
| 状态震荡检测 | 检查最近N步状态是否重复 | 优化类循环 |
| 收敛条件 | 设定明确的退出标准 | 搜索/优化类循环 |

## 总结

1. **循环节点**是Agent实现自修正、迭代优化的核心机制
2. **Checkpoint**是生产级Agent的必需品，提供断点续传和状态回滚能力
3. 生产环境建议使用SQLite/PostgreSQL持久化checkpoint
4. 循环节点必须配备多层防护防止无限循环

## 参考资料

- [LangGraph Documentation - Checkpoints](https://langchain-ai.github.io/langgraph/concepts/persistence/)
- [LangGraph How-tos](https://langchain-ai.github.io/langgraph/how-tos/)
