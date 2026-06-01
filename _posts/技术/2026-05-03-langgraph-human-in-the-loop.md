---
layout: post
title: "LangGraph Human-in-the-Loop 模式实战"
date: 2026-05-03
categories: [AI Agent, LangGraph, Human-in-the-Loop]
tags: [LangGraph, Agent, 人机协同, Python]
---

## 引言

在 AI Agent 应用开发中，完全自主的 Agent 往往难以应对复杂业务场景中的边缘情况。Human-in-the-Loop（HITL）模式通过在关键决策节点引入人工干预，在自动化效率与人工判断精度之间取得平衡。LangGraph 作为 LangChain 生态中的有状态图执行框架，为构建 HITL Agent 提供了原生支持。

本文将结合 Python 代码示例，展示如何使用 LangGraph 的 `interrupt` 机制构建一个带人工审批流程的 Agent 应用。

## 什么是 Human-in-the-Loop

HITL 的核心思想是：Agent 在遇到高风险的决策点时暂停执行，将决策权交给人类操作员，待人工确认或修正后再继续执行后续步骤。

典型应用场景包括：
- 财务审批：Agent 自动生成报销单，但需要人工审核后才能提交
- 内容发布：Agent 撰写文章后，需要编辑确认再发布
- 代码部署：Agent 生成代码变更后，需要开发者 review 再合并

## LangGraph 的 interrupt 机制

LangGraph 提供了 `interrupt` 功能，可以在图的任意节点执行前/后挂起执行流。中断后，图状态会被持久化，等待外部（人类）通过 `Command` 恢复执行。

### 基础用法

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command


class AgentState(TypedDict):
    content: str
    approved: bool
    feedback: str


def generate_content(state: AgentState) -> AgentState:
    """模拟 Agent 生成内容"""
    state["content"] = "这是一份自动生成的季度销售报告：\n- Q1 营收增长 15%\n- 客户满意度 92%"
    return state


def human_approval(state: AgentState) -> AgentState:
    """人工审批节点：使用 interrupt 暂停执行"""
    approved = interrupt({
        "message": "请审核以下内容，确认后回复 approve 或 reject",
        "content": state["content"]
    })
    state["approved"] = approved
    return state


def publish_content(state: AgentState) -> AgentState:
    """发布内容"""
    if state["approved"]:
        state["feedback"] = "内容已发布"
    else:
        state["feedback"] = "内容已驳回"
    return state


# 构建图
builder = StateGraph(AgentState)
builder.add_node("generate", generate_content)
builder.add_node("approve", human_approval)
builder.add_node("publish", publish_content)

builder.add_edge(START, "generate")
builder.add_edge("generate", "approve")
builder.add_edge("approve", "publish")
builder.add_edge("publish", END)

# 使用 MemorySaver 持久化状态
memory = MemorySaver()
graph = builder.compile(checkpointer=memory)
```

### 执行与恢复

```python
# 第一步：运行到中断点
config = {"configurable": {"thread_id": "thread-001"}}
result = graph.invoke({"content": "", "approved": False, "feedback": ""}, config)
# 此时执行会挂起在 approve 节点，result 包含中断信息

print(f"中断信息：{result}")

# 第二步：人工决策后恢复执行
# 人类操作员审核后，通过 Command(resume=...) 恢复
graph.invoke(
    Command(resume={"approved": True}),
    config
)
print("审批通过，内容已发布")
```

## 构建实用的审批工作流

下面展示一个更贴近生产环境的示例——文档发布审批系统，包含多级审批和时间戳记录。

```python
from typing_extensions import TypedDict, Annotated
from datetime import datetime
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command


class DocumentState(TypedDict):
    title: str
    body: str
    author: str
    editor_approved: bool
    admin_approved: bool
    audit_log: Annotated[list, "追加审计日志"]


def draft_document(state: DocumentState) -> DocumentState:
    """起草文档"""
    state["title"] = "Q3 产品路线图"
    state["body"] = "## 产品路线图\n\n### 7月\n- 上线 Agent 工作流引擎\n- 完成 RAG 检索优化\n\n### 8月\n- 多模态支持\n- 性能基准测试"
    state["audit_log"].append(f"[{datetime.now().isoformat()}] 文档起草完成，作者: {state['author']}")
    return state


def editor_review(state: DocumentState) -> DocumentState:
    """编辑审核（第一级审批）"""
    decision = interrupt({
        "stage": "编辑审核",
        "title": state["title"],
        "body": state["body"],
        "options": ["approve", "reject", "request_changes"]
    })
    state["editor_approved"] = decision.get("action") == "approve"
    state["audit_log"].append(
        f"[{datetime.now().isoformat()}] 编辑审核结果: {decision.get('action')}, "
        f"备注: {decision.get('note', '无')}"
    )
    if decision.get("action") == "request_changes":
        # 可以在这里加入修改逻辑
        pass
    return state


def admin_review(state: DocumentState) -> DocumentState:
    """管理员审核（第二级审批）"""
    if not state["editor_approved"]:
        state["audit_log"].append(f"[{datetime.now().isoformat()}] 编辑未通过，跳过管理员审核")
        return state

    decision = interrupt({
        "stage": "管理员终审",
        "title": state["title"],
        "editor_status": "已通过",
        "options": ["approve", "reject"]
    })
    state["admin_approved"] = decision.get("action") == "approve"
    state["audit_log"].append(
        f"[{datetime.now().isoformat()}] 管理员审核结果: {decision.get('action')}"
    )
    return state


def finalize(state: DocumentState) -> DocumentState:
    """最终处理"""
    state["audit_log"].append(
        f"[{datetime.now().isoformat()}] "
        f"最终状态: {'已发布' if state['admin_approved'] else '已驳回'}"
    )
    return state


# 构建多级审批图
builder = StateGraph(DocumentState)
builder.add_node("draft", draft_document)
builder.add_node("editor_review", editor_review)
builder.add_node("admin_review", admin_review)
builder.add_node("finalize", finalize)

builder.add_edge(START, "draft")
builder.add_edge("draft", "editor_review")
builder.add_edge("editor_review", "admin_review")
builder.add_edge("admin_review", "finalize")
builder.add_edge("finalize", END)

memory = MemorySaver()
approval_graph = builder.compile(checkpointer=memory)
```

## 生产环境注意事项

1. **持久化后端**：`MemorySaver` 仅适用于开发和测试，生产环境应使用 `SqliteSaver` 或 `PostgresSaver`，确保服务重启后中断状态不丢失。

2. **超时与过期**：应为中断点设置合理的超时时间，配合定时任务自动取消过期审批。

3. **通知机制**：实际项目中，`interrupt` 触发后应通过 WebSocket、邮件或消息推送通知审批人，而非仅靠轮询。

4. **幂等性**：`Command(resume=...)` 的恢复操作应保证幂等，避免重复提交导致双写。

5. **审计追踪**：所有中断和恢复事件都应记录到审计日志中，满足合规要求。

## 总结

LangGraph 的 `interrupt` 机制为构建 HITL Agent 提供了简洁而强大的原语。通过将人工决策建模为图中的挂起节点，开发者可以在不破坏 Agent 整体架构的前提下，灵活引入人工审批流程。结合持久化后端和通知机制，可以构建出生产级的智能审批系统。

## 面试题

**1. LangGraph 中 `interrupt` 和普通函数的 `input()` 调用有什么本质区别？为什么不能用 `input()` 替代？**

`interrupt` 会将图执行状态序列化到 checkpointer 中持久化，支持跨进程、跨服务恢复。而 `input()` 只是阻塞当前线程，一旦服务重启状态就丢失了。此外，`interrupt` 支持通过 `Command(resume=...)` 从外部 API 恢复执行，便于构建异步审批工作流，这是 `input()` 无法做到的。

**2. 在 HITL Agent 中，如何设计审批超时和自动降级策略？请给出具体方案。**

关键设计点：
- 在 `interrupt` 触发后记录时间戳到状态中，通过外部调度器（如 APScheduler、Celery Beat）定时扫描超时的中断状态。
- 超时后的降级策略可根据业务场景选择：①自动驳回（保守策略，适用于审批类）；②自动通过（适用于非关键通知类）；③升级到更高级别审批人。
- 恢复时使用 `graph.update_state()` 注入降级决策，再调用 `graph.invoke(Command(resume=...), config)` 继续执行。
- 所有超时和降级事件必须记录到审计日志。