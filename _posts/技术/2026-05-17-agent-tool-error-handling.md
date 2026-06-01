---
layout: post
title: "Agent工具调用：错误处理、重试与降级策略"
date: 2026-05-17
categories: [AI Agent, 工程实践]
tags: [Agent, 工具调用, 错误处理, 重试, 降级, Python]
---

## 引言

AI Agent 的核心能力之一是调用外部工具（API、数据库、文件系统等）完成复杂任务。然而，外部工具不可靠——API 超时、数据库死锁、权限不足、返回格式异常等问题层出不穷。一个健壮的 Agent 必须具备完善的错误处理、重试和降级机制。本文将深入探讨这些策略的工程实现。

## 工具调用失败的分类

在设计错误处理策略前，先对失败类型做系统分类：

| 失败类型 | 示例 | 是否可重试 | 推荐策略 |
|---------|------|-----------|---------|
| 瞬时故障 | 网络超时、503 错误 | 是 | 指数退避重试 |
| 限流错误 | 429 Too Many Requests | 是 | 等待后重试 |
| 参数错误 | 400 Bad Request | 否 | 修正参数后重试 |
| 权限错误 | 401/403 | 否 | 降级或人工介入 |
| 资源不存在 | 404 Not Found | 否 | 降级或跳过 |
| 业务逻辑错误 | 余额不足、状态不允许 | 否 | 降级或终止 |

## 构建健壮的工具调用层

下面构建一个生产级的工具调用包装器，集成重试、超时、熔断和降级：

```python
import asyncio
import functools
import time
import random
from enum import Enum
from typing import Any, Callable, Dict, Optional, TypeVar
from dataclasses import dataclass, field
import logging

logger = logging.getLogger(__name__)

T = TypeVar("T")


class ErrorCategory(Enum):
    """错误类别"""
    TRANSIENT = "transient"       # 瞬时故障，可重试
    RATE_LIMIT = "rate_limit"     # 限流，等待后可重试
    CLIENT_ERROR = "client_error" # 客户端错误，不可重试
    FATAL = "fatal"               # 致命错误，不可恢复


@dataclass
class RetryConfig:
    """重试配置"""
    max_retries: int = 3
    base_delay: float = 1.0       # 基础等待秒数
    max_delay: float = 30.0       # 最大等待秒数
    backoff_factor: float = 2.0   # 退避因子
    jitter: bool = True           # 是否添加随机抖动
    retryable_exceptions: tuple = (TimeoutError, ConnectionError)


@dataclass
class CircuitBreakerConfig:
    """熔断器配置"""
    failure_threshold: int = 5     # 连续失败阈值
    recovery_timeout: float = 60.0 # 恢复等待时间（秒）
    half_open_max_calls: int = 3   # 半开状态最大试探请求数


@dataclass
class CircuitBreaker:
    """熔断器实现"""
    config: CircuitBreakerConfig = field(default_factory=CircuitBreakerConfig)
    failure_count: int = 0
    last_failure_time: float = 0.0
    state: str = "closed"  # closed | open | half_open
    half_open_calls: int = 0

    def call(self, func: Callable, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.config.recovery_timeout:
                self.state = "half_open"
                self.half_open_calls = 0
                logger.info("熔断器进入半开状态，尝试恢复")
            else:
                raise CircuitBreakerOpenError(
                    f"熔断器已打开，{self.config.recovery_timeout:.0f}秒后尝试恢复"
                )

        if self.state == "half_open":
            if self.half_open_calls >= self.config.half_open_max_calls:
                raise CircuitBreakerOpenError("半开状态已达到最大试探次数")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception:
            self._on_failure()
            raise

    def _on_success(self):
        self.failure_count = 0
        self.state = "closed"
        self.half_open_calls = 0

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.state == "half_open":
            self.half_open_calls += 1
        if self.failure_count >= self.config.failure_threshold:
            self.state = "open"
            logger.warning(f"熔断器打开，连续失败 {self.failure_count} 次")


class CircuitBreakerOpenError(Exception):
    pass


def classify_error(exception: Exception) -> ErrorCategory:
    """根据异常类型分类"""
    if isinstance(exception, (TimeoutError, ConnectionError)):
        return ErrorCategory.TRANSIENT
    if hasattr(exception, "status_code"):
        status = exception.status_code
        if status == 429:
            return ErrorCategory.RATE_LIMIT
        if status in (500, 502, 503, 504):
            return ErrorCategory.TRANSIENT
        if status in (400, 422):
            return ErrorCategory.CLIENT_ERROR
        if status in (401, 403):
            return ErrorCategory.FATAL
    return ErrorCategory.FATAL


def tool_call(
    retry_config: Optional[RetryConfig] = None,
    circuit_breaker: Optional[CircuitBreaker] = None,
    fallback: Optional[Callable] = None,
    timeout: float = 30.0
):
    """
    工具调用的装饰器，集成重试、超时、熔断和降级

    使用示例:
        @tool_call(retry_config=RetryConfig(max_retries=3), timeout=10.0)
        async def search_database(query: str) -> list:
            ...
    """
    retry_config = retry_config or RetryConfig()

    def decorator(func: Callable):
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            last_exception = None

            for attempt in range(retry_config.max_retries + 1):
                try:
                    # 熔断检查
                    if circuit_breaker:
                        return circuit_breaker.call(
                            lambda: asyncio.get_event_loop().run_until_complete(
                                asyncio.wait_for(func(*args, **kwargs), timeout=timeout)
                            )
                        )

                    result = await asyncio.wait_for(
                        func(*args, **kwargs),
                        timeout=timeout
                    )
                    return result

                except asyncio.TimeoutError as e:
                    last_exception = e
                    wait = _calc_wait(attempt, retry_config)
                    logger.warning(
                        f"工具调用超时 (尝试 {attempt + 1}/{retry_config.max_retries + 1})，"
                        f"{wait:.1f}秒后重试: {func.__name__}"
                    )
                    await asyncio.sleep(wait)

                except Exception as e:
                    last_exception = e
                    category = classify_error(e)
                    logger.error(
                        f"工具调用失败 [{category.value}] (尝试 {attempt + 1}): "
                        f"{func.__name__} - {e}"
                    )

                    if category in (ErrorCategory.CLIENT_ERROR, ErrorCategory.FATAL):
                        break  # 不可重试的错误，直接退出重试循环

                    if attempt < retry_config.max_retries:
                        wait = _calc_wait(attempt, retry_config)
                        if category == ErrorCategory.RATE_LIMIT:
                            # 限流时使用更长的等待时间
                            wait = max(wait, 5.0)
                        await asyncio.sleep(wait)

            # 所有重试都失败，执行降级
            if fallback:
                logger.warning(
                    f"工具 {func.__name__} 所有重试失败，启用降级策略"
                )
                return await fallback(*args, **kwargs)

            raise last_exception

        @functools.wraps(func)
        def sync_wrapper(*args, **kwargs):
            last_exception = None

            for attempt in range(retry_config.max_retries + 1):
                try:
                    if circuit_breaker:
                        return circuit_breaker.call(func, *args, **kwargs)
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    category = classify_error(e)

                    if category in (ErrorCategory.CLIENT_ERROR, ErrorCategory.FATAL):
                        break

                    if attempt < retry_config.max_retries:
                        wait = _calc_wait(attempt, retry_config)
                        time.sleep(wait)

            if fallback:
                return fallback(*args, **kwargs)
            raise last_exception

        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper

    return decorator


def _calc_wait(attempt: int, config: RetryConfig) -> float:
    """计算指数退避等待时间（带随机抖动）"""
    wait = min(config.base_delay * (config.backoff_factor ** attempt), config.max_delay)
    if config.jitter:
        wait *= random.uniform(0.5, 1.5)
    return wait
```

## 降级策略设计

降级策略是保证 Agent 可用性的最后防线。以下是几种常见的降级模式：

```python
from typing import List, Dict, Any


class ToolDegradation:
    """工具降级策略管理器"""

    @staticmethod
    async def fallback_search_engine(primary: str, secondary: str, query: str) -> List[Dict]:
        """搜索引擎降级：主引擎失败时切换到备选引擎"""
        engines = {
            "elasticsearch": lambda q: [],
            "meilisearch": lambda q: [],
        }
        try:
            return await engines[primary](query)
        except Exception:
            logger.warning(f"搜索引擎 {primary} 不可用，降级到 {secondary}")
            return await engines[secondary](query)

    @staticmethod
    def fallback_cache_hit(cache: Dict, key: str) -> Optional[Any]:
        """缓存降级：数据库不可用时返回缓存数据"""
        return cache.get(key, None)

    @staticmethod
    def fallback_static_response(query_type: str) -> str:
        """静态响应降级：所有工具都不可用时的兜底回复"""
        static_responses = {
            "weather": "抱歉，天气服务暂时不可用，请稍后重试",
            "stock": "抱歉，股票查询服务正在维护中",
            "default": "很抱歉，当前服务不可用，请稍后再试",
        }
        return static_responses.get(query_type, static_responses["default"])

    @staticmethod
    def fallback_partial_result(primary_results: List, backup_results: List) -> List:
        """部分结果降级：合并主结果和备用结果"""
        if primary_results:
            return primary_results
        logger.info("主数据源无结果，使用备用数据源")
        return backup_results


# Agent 中的实际使用
class RobustAgent:
    def __init__(self):
        self.cb = CircuitBreaker()
        self.result_cache = {}

    @tool_call(
        retry_config=RetryConfig(max_retries=2, base_delay=0.5),
        circuit_breaker=CircuitBreaker(),
        fallback=lambda q: ToolDegradation.fallback_cache_hit(
            self.result_cache, q
        ),
        timeout=5.0
    )
    async def search_knowledge_base(self, query: str) -> List[str]:
        """搜索知识库（带完整的错误处理链路）"""
        # 实际搜索逻辑
        results = await self._call_vector_search(query)
        # 更新缓存
        self.result_cache[query] = results
        return results

    async def _call_vector_search(self, query: str) -> List[str]:
        """模拟向量搜索调用"""
        await asyncio.sleep(0.1)
        return [f"结果1 for {query}", f"结果2 for {query}"]
```

## Agent 级别的错误编排

在 Agent 层面，还需要对工具调用链做整体编排。LangChain 和 LangGraph 都提供了相应的机制：

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from typing import TypedDict, List


class AgentState(TypedDict):
    messages: List[dict]
    tool_errors: List[dict]
    fallback_used: bool


def handle_tool_error(state: AgentState) -> AgentState:
    """处理工具调用错误的专用节点"""
    last_error = state["tool_errors"][-1] if state["tool_errors"] else None
    if last_error:
        error_msg = last_error.get("message", "未知错误")
        state["messages"].append({
            "role": "system",
            "content": f"工具调用失败: {error_msg}。请尝试使用替代方法或告知用户当前限制。"
        })
        state["fallback_used"] = True
    return state
```

关键设计原则：
1. **错误传播要有边界**：工具的错误不应直接抛给用户，Agent 应翻译成用户可理解的表述
2. **部分成功优于完全失败**：如果 3 个工具中 2 个成功 1 个失败，应返回部分结果而非整体报错
3. **记录所有失败上下文**：便于后续分析和优化

## 总结

构建健壮的 Agent 工具调用层需要三层防护：第一层是重试机制处理瞬时故障，第二层是熔断器防止级联失败，第三层是降级策略保证基本可用性。三者结合，才能让 Agent 在生产环境中稳定运行。

## 面试题

**1. 在 Agent 工具调用中，为什么要使用指数退避（Exponential Backoff）而非固定间隔重试？**

指数退避的核心优势是平衡"快速恢复"和"避免雪崩"。在网络拥塞或服务端过载场景下，如果所有客户端都按固定间隔重试，会在同一时刻产生请求高峰，加剧服务压力。指数退避让重试间隔逐渐增大（如 1s → 2s → 4s → 8s），给服务端留出恢复时间。此外，加上随机抖动（Jitter）可以进一步打散重试请求的时间分布，避免"惊群效应"。

**2. 熔断器（Circuit Breaker）的三种状态分别是什么？在实际 Agent 系统中如何配置阈值？**

三种状态：
- **Closed（闭合）**：正常状态，请求正常通过。连续失败计数达到阈值后切换到 Open。
- **Open（打开）**：熔断状态，所有请求直接拒绝（快速失败），经过 recovery_timeout 后切换到 Half-Open。
- **Half-Open（半开）**：允许少量试探请求通过。如果试探成功→切回 Closed；失败→切回 Open。

配置建议：
- `failure_threshold`：建议设为 3-5 次。对延迟敏感的 Agent（如实时对话）设 3，批处理 Agent 可设 5。
- `recovery_timeout`：建议 30-120 秒。需结合工具的平均恢复时间，太短会导致频繁开关切换。
- `half_open_max_calls`：建议 1-3 个请求，确保恢复验证不会给刚恢复的服务造成二次冲击。