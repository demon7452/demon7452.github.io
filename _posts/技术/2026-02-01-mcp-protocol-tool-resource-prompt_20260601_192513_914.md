---
layout: post
title: "MCP协议详解：Tool/Resource/Prompt三种能力"
date: 2026-02-01 10:00:00 +0800
category: 技术
tags: [MCP, 协议, AI Agent, Tool, Resource, Prompt]
keywords: MCP协议, Model Context Protocol, Tool能力, Resource能力, Prompt能力, AI Agent标准化
description: 深入解析MCP（Model Context Protocol）协议的三种核心能力——Tool、Resource和Prompt，通过完整代码示例展示如何构建MCP Server和Client，实现AI Agent的标准化工具集成。
---

## 核心概念

MCP（Model Context Protocol）是 Anthropic 于 2024 年底推出的开源协议，旨在为 AI 模型与外部工具/数据源之间建立标准化的通信接口。可以把它理解为 AI 世界的"USB-C 协议"——不同的工具和数据源只要实现 MCP 接口，就能被任何支持 MCP 的 AI 应用即插即用。

### 协议架构

MCP 采用 Client-Server 架构：

```
AI Host (Claude Desktop / IDE / Agent)
    │
    ▼
MCP Client (协议层，内嵌在 Host 中)
    │
    ├──► MCP Server A (文件系统)
    ├──► MCP Server B (数据库)
    ├──► MCP Server C (API 网关)
    └──► MCP Server D (自定义工具)
```

通信方式支持两种传输：
- **stdio**：通过标准输入/输出，适合本地进程
- **HTTP + SSE**：通过 HTTP 请求和 Server-Sent Events，适合远程服务

### 三种核心能力

MCP 定义了三种标准化能力（Primitives），每一种都对应 AI 应用中的一个基础需求：

| 能力 | 类比 | 控制方 | 典型场景 |
|------|------|--------|----------|
| **Tool** | API 端点 | 模型主动调用 | 查数据库、发邮件、调 API |
| **Resource** | 文件读取 | 模型被动获取 | 读文档、获取 schema、查看日志 |
| **Prompt** | 模板 | 用户或模型触发 | 预置的对话模板、工作流模板 |

三者之间的核心区别：**Tool 是模型驱动的操作（模型决定何时调用），Resource 是应用驱动的数据暴露（应用决定何时提供），Prompt 是用户驱动的交互模板（用户选择使用）。**

## 实战示例

下面构建一个完整的 MCP Server（提供天气查询和文件管理能力）和对应的 Client。

### 第一部分：MCP Server 实现

```python
#!/usr/bin/env python3
"""
weather_server.py - MCP Server 示例
提供 Tool（天气查询）和 Resource（城市列表）
"""

import json
import sys
import asyncio
from typing import Any
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    TextContent,
    Resource,
    ResourceTemplate,
    Prompt,
    PromptMessage,
    GetPromptResult,
)

# 创建 Server 实例
server = Server("weather-server")

# ============ 1. Tool 能力：天气查询 ============
@server.list_tools()
async def handle_list_tools() -> list[Tool]:
    """向 Client 暴露所有可用工具"""
    return [
        Tool(
            name="get_weather",
            description="查询指定城市的实时天气信息",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如 '北京'、'上海'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位",
                        "default": "celsius"
                    }
                },
                "required": ["city"]
            }
        ),
        Tool(
            name="get_forecast",
            description="查询指定城市未来3天天气预报",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称"
                    }
                },
                "required": ["city"]
            }
        )
    ]

@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[TextContent]:
    """处理工具调用请求"""
    if name == "get_weather":
        city = arguments["city"]
        unit = arguments.get("unit", "celsius")

        # 模拟天气数据
        weather_db = {
            "北京": {"temp": 25, "condition": "晴", "humidity": 40},
            "上海": {"temp": 28, "condition": "多云", "humidity": 65},
            "深圳": {"temp": 30, "condition": "阵雨", "humidity": 80},
        }

        data = weather_db.get(city, {"temp": 20, "condition": "未知", "humidity": 50})

        if unit == "fahrenheit":
            data["temp"] = data["temp"] * 9/5 + 32

        result = (
            f"{city}天气：{data['condition']}，"
            f"温度{data['temp']}°{'F' if unit == 'fahrenheit' else 'C'}，"
            f"湿度{data['humidity']}%"
        )
        return [TextContent(type="text", text=result)]

    elif name == "get_forecast":
        city = arguments["city"]
        result = f"{city}未来3天预报：\n- 明天：晴转多云 22-30°C\n- 后天：多云 20-28°C\n- 大后天：小雨 18-25°C"
        return [TextContent(type="text", text=result)]

    else:
        return [TextContent(type="text", text=f"未知工具: {name}")]

# ============ 2. Resource 能力：城市列表和天气数据 ============
@server.list_resources()
async def handle_list_resources() -> list[Resource]:
    """暴露可用的资源列表"""
    return [
        Resource(
            uri="weather://cities",
            name="支持的城市列表",
            description="所有支持天气查询的城市",
            mimeType="application/json"
        ),
        Resource(
            uri="weather://stats",
            name="天气统计信息",
            description="全局天气数据统计",
            mimeType="application/json"
        ),
        ResourceTemplate(
            uriTemplate="weather://city/{name}",
            name="城市详细天气",
            description="指定城市的完整天气数据"
        )
    ]

@server.read_resource()
async def handle_read_resource(uri: str) -> str:
    """读取资源内容"""
    if uri == "weather://cities":
        return json.dumps({
            "cities": ["北京", "上海", "深圳", "广州", "杭州", "成都"],
            "total": 6
        }, ensure_ascii=False)

    elif uri == "weather://stats":
        return json.dumps({
            "total_queries": 12580,
            "avg_temp": 24.5,
            "most_queried": "北京"
        }, ensure_ascii=False)

    elif uri.startswith("weather://city/"):
        city = uri.split("/")[-1]
        return json.dumps({
            "city": city,
            "current": {"temp": 25, "condition": "晴"},
            "forecast": ["22-30°C", "20-28°C", "18-25°C"]
        }, ensure_ascii=False)

    return json.dumps({"error": "资源不存在"})

# ============ 3. Prompt 能力：预置查询模板 ============
@server.list_prompts()
async def handle_list_prompts() -> list[Prompt]:
    """暴露可用的 Prompt 模板"""
    return [
        Prompt(
            name="weather_report",
            description="生成城市天气报告模板",
            arguments=[
                {"name": "city", "description": "城市名称", "required": True}
            ]
        ),
        Prompt(
            name="travel_advice",
            description="旅行天气建议模板",
            arguments=[
                {"name": "city", "description": "目的地城市", "required": True},
                {"name": "days", "description": "旅行天数", "required": False}
            ]
        )
    ]

@server.get_prompt()
async def handle_get_prompt(
    name: str, arguments: dict | None
) -> GetPromptResult:
    """获取 Prompt 模板内容"""
    if name == "weather_report":
        city = arguments.get("city", "北京") if arguments else "北京"
        return GetPromptResult(
            messages=[
                PromptMessage(
                    role="user",
                    content=TextContent(
                        type="text",
                        text=f"请帮我查询{city}的天气，并生成一份简洁的天气报告"
                    )
                )
            ]
        )

    elif name == "travel_advice":
        city = arguments.get("city", "北京") if arguments else "北京"
        days = arguments.get("days", "3") if arguments else "3"
        return GetPromptResult(
            messages=[
                PromptMessage(
                    role="user",
                    content=TextContent(
                        type="text",
                        text=f"我计划去{city}旅游{days}天，请查询当地天气并给出穿衣和出行建议"
                    )
                )
            ]
        )

    return GetPromptResult(messages=[])


# ============ 启动 Server ============
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationCapabilities(
                sampling={},
                experimental={},
                roots={},
            )
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### 第二部分：MCP Client 实现

```python
#!/usr/bin/env python3
"""
mcp_client_demo.py - MCP Client 示例
连接 weather_server 并使用其 Tool/Resource/Prompt
"""

import asyncio
import json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def demo_tool_use(session: ClientSession):
    """演示 Tool 能力"""
    print("\n===== Tool 能力演示 =====")

    # 获取工具列表
    tools = await session.list_tools()
    print(f"可用工具: {[t.name for t in tools.tools]}")

    # 调用 get_weather
    result = await session.call_tool("get_weather", {"city": "深圳", "unit": "celsius"})
    print(f"天气查询结果: {result.content[0].text}")

    # 调用 get_forecast
    result = await session.call_tool("get_forecast", {"city": "北京"})
    print(f"天气预报结果: {result.content[0].text}")

async def demo_resource_use(session: ClientSession):
    """演示 Resource 能力"""
    print("\n===== Resource 能力演示 =====")

    # 获取资源列表
    resources = await session.list_resources()
    print(f"可用资源: {[r.name for r in resources.resources]}")

    # 读取城市列表
    content = await session.read_resource("weather://cities")
    data = json.loads(content.contents[0].text)
    print(f"城市列表: {data}")

    # 读取城市详情（使用 ResourceTemplate）
    content = await session.read_resource("weather://city/上海")
    data = json.loads(content.contents[0].text)
    print(f"上海详情: {data}")

async def demo_prompt_use(session: ClientSession):
    """演示 Prompt 能力"""
    print("\n===== Prompt 能力演示 =====")

    # 获取 Prompt 列表
    prompts = await session.list_prompts()
    print(f"可用 Prompt: {[p.name for p in prompts.prompts]}")

    # 获取天气报告模板
    prompt_result = await session.get_prompt("weather_report", {"city": "杭州"})
    for msg in prompt_result.messages:
        print(f"Prompt 模板内容: {msg.content.text}")

    # 获取旅行建议模板
    prompt_result = await session.get_prompt("travel_advice", {"city": "成都", "days": "5"})
    for msg in prompt_result.messages:
        print(f"Prompt 模板内容: {msg.content.text}")

async def main():
    server_params = StdioServerParameters(
        command="python",
        args=["weather_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # 初始化会话
            await session.initialize()

            # 演示三种能力
            await demo_tool_use(session)
            await demo_resource_use(session)
            await demo_prompt_use(session)

if __name__ == "__main__":
    asyncio.run(main())
```

## 最佳实践

1. **能力分离原则**：严格区分 Tool/Resource/Prompt。不要用 Tool 返回本来应该用 Resource 暴露的静态数据，也不要用 Resource 来执行有副作用的操作。混乱的能力分工会让 LLM 产生错误的行为预期。
2. **Tool 描述要包含"何时使用"**：不仅描述工具做什么，还要说明在什么场景下应该选择这个工具而不是其他工具。这直接影响 LLM 的工具选择准确率。
3. **Resource URI 设计要有命名空间**：使用 `scheme://category/identifier` 格式（如 `weather://city/beijing`），避免 URI 冲突，也方便后续扩展。
4. **Prompt 模板要模块化**：不要把一个 Prompt 做得过于庞大，应该拆分为多个小 Prompt，让用户可以灵活组合。例如"天气报告"和"旅行建议"分开，而非混在一起。
5. **传输层选择**：本地工具用 stdio（零网络开销、天然安全），远程服务用 HTTP+SSE（支持分布式部署）。不要强行把所有 Server 都做成 HTTP。
6. **错误标准化**：工具返回错误时，使用结构化格式 `{"error": true, "message": "...", "retryable": false}`，让 LLM 能够基于错误类型做出合理决策。

## 面试题

**Q1：MCP 协议中 Tool、Resource、Prompt 三种能力的本质区别是什么？如果让你设计一个文件管理 MCP Server，你会如何分配这三种能力？**

**A1**：本质区别在于**控制权归属**和**是否有副作用**：

- **Tool**：模型驱动、可能有副作用（写入/删除）。模型主动决定调用时机，类比 REST API 的 POST/PUT/DELETE。
- **Resource**：应用驱动、无副作用（只读）。应用决定暴露哪些数据，类比 REST API 的 GET 请求。模型被动获取。
- **Prompt**：用户驱动、无副作用。用户选择使用哪个模板发起对话，类比预置的对话快捷方式。

文件管理 MCP Server 的能力分配：

- **Tool**：创建文件、删除文件、移动文件、重命名文件（都有副作用）
- **Resource**：读取文件内容、列出目录结构、获取文件元数据（只读）
- **Prompt**：代码审查模板、文档生成模板、文件整理模板（用户触发）

---

**Q2：为什么 MCP 选择 stdio 和 HTTP+SSE 两种传输方式，而不是统一用 HTTP REST？stdio 在 AI Agent 场景下有什么独特优势？**

**A2**：两种传输方式对应两种部署场景：

**stdio 的独特优势**：
- **零配置安全性**：进程间通信不暴露网络端口，无需认证、TLS、防火墙配置，杜绝了网络攻击面。
- **进程生命周期绑定**：Server 随 Client 启动而启动、终止而终止，资源管理极其简单，没有孤儿进程问题。
- **极低延迟**：无网络栈开销，适合高频交互的本地工具（如每轮 Agent 循环都要读文件的场景）。
- **离线可用**：不依赖网络连通性。

**HTTP+SSE 的必要性**：当 MCP Server 需要远程部署（如团队共享的数据库 Server、云端 API 网关），或者需要独立扩容时，必须走网络。SSE 用于 Server 向 Client 推送通知（如资源变更），补足了纯 HTTP REST 单向请求的不足。

设计哲学：本地优先用 stdio（安全默认），远程才用 HTTP（按需开启），而不是一刀切用 HTTP 然后让开发者自己处理安全问题。