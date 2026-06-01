---
layout: post
title: "MCP协议详解：Tool、Resource与Prompt三种能力"
category: 技术
tags: [MCP, Model Context Protocol, AI Agent, Tool, 协议设计]
keywords: MCP协议, Model Context Protocol, Tool, Resource, Prompt, Agent开发, Anthropic
description: 深入解析Anthropic提出的MCP（Model Context Protocol）协议架构，详解Tool、Resource、Prompt三种核心原语的设计思想与工程实现，附完整Python Server/Client代码示例。
---

## 一、核心概念

MCP（Model Context Protocol）是 Anthropic 于 2024 年底发布的开放协议，目标是成为 LLM 与外部工具、数据源之间的"USB-C 接口"——一套统一标准，取代每个 LLM 应用各自实现 Function Calling 适配层的碎片化现状。

MCP 架构采用 **Client-Server** 模式：

```
LLM Host (Claude/GPT/...)  ←→  MCP Client  ←→  MCP Server
                                                     ├── Tool A
                                                     ├── Resource B
                                                     └── Prompt C
```

- **MCP Host**：LLM 应用本身（如 Claude Desktop、自建 ChatBot）
- **MCP Client**：协议客户端，维护与 Server 的连接，将 Server 的能力暴露给 Host
- **MCP Server**：轻量级服务，暴露三种能力原语——Tool、Resource、Prompt

与 OpenAI Function Calling 的关键区别：MCP 是**双向**的。Server 不仅能响应 Tool 调用，还能主动向 LLM 注入上下文（Resource）和 Prompt 模板，实现更深的上下文整合。

## 二、三种能力原语详解

### 2.1 Tool（工具）

Tool 最直观：Server 暴露可执行的函数，LLM 通过 Client 调用它们。每个 Tool 需提供名称、描述和 JSON Schema 参数定义。

```python
from pydantic import BaseModel, Field

class GetWeatherInput(BaseModel):
    city: str = Field(description="城市名称，如 'Beijing'")

# MCP Tool 注册
weather_tool = {
    "name": "get_weather",
    "description": "获取指定城市的实时天气信息",
    "inputSchema": GetWeatherInput.model_json_schema()
}
```

### 2.2 Resource（资源）

Resource 是 MCP 最具创新性的原语——它让 Server 可以**主动向 LLM 注入上下文数据**，而不需要 LLM 显式调用。Resource 类似于 RESTful API 中的 `GET` 端点，但通过 URI 模板化：

```
weather://beijing/current    → 北京当前天气数据
docs://product/api-reference → API 参考文档
db://users/schema            → 用户表结构
```

LLM 在需要上下文时，Host 自动从 Resource 拉取并注入。这意味着无需在 System Prompt 中硬编码所有上下文，上下文可以动态、按需加载。

### 2.3 Prompt（提示模板）

Prompt 原语让 Server 可以提供结构化的 Prompt 模板，包含参数化和多轮对话支持：

```python
prompt_template = {
    "name": "code_review",
    "description": "代码审查 Prompt 模板",
    "arguments": [
        {"name": "language", "description": "编程语言", "required": True},
        {"name": "focus", "description": "审查关注点"}
    ]
}
```

这使得团队可以将 Prompt 作为资产统一管理、版本控制和复用，而不是散落在代码各处。

## 三、实战：构建 MCP Weather Server

以下实现一个完整的 MCP Server，暴露 Tool 和 Resource 两种能力：

```python
import json
import asyncio
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool, TextContent, Resource, ResourceTemplate,
    ListToolsResult, CallToolResult, ListResourcesResult
)

# ---------- 1. 创建 Server ----------
server = Server("weather-server")

# ---------- 2. 注册 Tool ----------
@server.list_tools()
async def handle_list_tools() -> ListToolsResult:
    return ListToolsResult(tools=[
        Tool(
            name="get_weather",
            description="获取指定城市的实时天气信息",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如 Beijing"
                    }
                },
                "required": ["city"]
            }
        )
    ])

@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> CallToolResult:
    if name == "get_weather":
        city = arguments["city"]
        # 模拟返回天气数据
        weather_data = {
            "city": city,
            "temperature": 22,
            "humidity": 65,
            "condition": "晴",
            "forecast": "未来三天持续晴好"
        }
        return CallToolResult(content=[
            TextContent(
                type="text",
                text=json.dumps(weather_data, ensure_ascii=False, indent=2)
            )
        ])
    raise ValueError(f"未知工具: {name}")

# ---------- 3. 注册 Resource ----------
@server.list_resources()
async def handle_list_resources() -> ListResourcesResult:
    return ListResourcesResult(resources=[
        Resource(
            uri="weather://forecast",
            name="全国天气预报",
            description="主要城市未来三天天气预报",
            mimeType="application/json"
        )
    ])

@server.read_resource()
async def handle_read_resource(uri: str) -> str:
    if uri == "weather://forecast":
        return json.dumps({
            "北京": ["晴 22°C", "多云 20°C", "晴 24°C"],
            "上海": ["小雨 18°C", "阴 19°C", "多云 22°C"]
        }, ensure_ascii=False)
    raise ValueError(f"未知资源: {uri}")

# ---------- 4. 启动 Server ----------
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationCapabilities(
                sampling={},
                experimental={},
                tools={},
                resources={}
            )
        )

if __name__ == "__main__":
    asyncio.run(main())
```

**对应的 MCP Client**：

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run():
    server_params = StdioServerParameters(
        command="python",
        args=["weather_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 列出可用 Tool
            tools = await session.list_tools()
            print(f"可用工具: {[t.name for t in tools.tools]}")

            # 调用 Tool
            result = await session.call_tool(
                "get_weather", {"city": "Beijing"}
            )
            print(f"天气结果: {result.content[0].text}")

            # 读取 Resource
            forecast = await session.read_resource("weather://forecast")
            print(f"预报: {forecast.contents[0].text}")

asyncio.run(run())
```

## 四、最佳实践

1. **Tool 粒度控制**：每个 Tool 只做一件事。MCP 的 Tool 注册是 JSON Schema 驱动的，粒度太粗（一个 Tool 处理多种逻辑）会导致参数 Schema 膨胀，LLM 难以准确填充。

2. **Resource 用于"被动上下文"**：资源型数据（文档、Schema、配置）走 Resource 原语，让 Host 按需拉取；执行型操作（发邮件、查数据库、调 API）走 Tool 原语。二者不应混用。

3. **Tool 的错误返回要结构化**：不要只返回字符串，带回一个标准错误对象 `{"error": "...", "code": 400, "suggestion": "..."}`，让 LLM 可以根据错误信息调整参数重试。

4. **Server 要无状态**：每个 Tool 调用应当是幂等的、无副作用的（除主动写入外）。避免在 Server 内存中维护跨调用的会话状态，否则 Client 断连重连后状态丢失。

5. **安全性**：MCP Server 运行在本地进程或受信网络中，但仍需对 Tool 的权限做最小化。如文件读写 Tool 应限制在白名单目录内，数据库 Tool 应使用只读账号。

---

## 面试题

### 题1：MCP 协议中的 Tool、Resource、Prompt 三种原语分别解决什么问题？与 OpenAI Function Calling 有何本质区别？

**答案**：三种原语各有分工——Tool 是"LLM 调用的函数"，解决"让 LLM 执行操作"的需求；Resource 是"LLM 可访问的数据源"，解决"动态注入上下文"的需求，Server 主动暴露数据，Host 按需拉取注入到 LLM 的上下文窗口；Prompt 是"可复用的提示模板"，解决"Prompt 集中管理"的需求。

与 OpenAI Function Calling 的本质区别：(1) MCP 是双向协议——Server 不仅能被调用（Tool），还能主动提供数据（Resource）和模板（Prompt），而 Function Calling 仅支持单向调用；(2) MCP 是独立于 LLM 厂商的开放标准，一个 MCP Server 可以被任何支持 MCP 的 Host（Claude、GPT、开源模型）使用，而 Function Calling 绑定在 OpenAI API 上；(3) MCP 有连接管理和生命周期，Server 进程常驻，支持多轮会话中的状态保持和资源预加载。

### 题2：在设计 MCP Server 时，什么场景下应该用 Resource 而不是 Tool？请举例说明。

**答案**：判断依据是"数据是否需要 LLM 主动执行操作来获取"。Resource 适合**被动、只读的数据暴露**——数据就放在那，LLM 的 Host 在需要上下文时自动拉取。Tool 适合**需要参数计算、有副作用的主动操作**。

Resource 适用场景示例：(1) API 参考文档——LLM 需要查阅函数签名时，Host 从 `docs://api-reference` Resource 拉取，无需 LLM 显式调用 Tool；(2) 数据库表结构——在多轮对话中，LLM 可能需要反复参考 Schema，Resource 提供 URI 化的 Schema 访问；(3) 用户配置——用户的偏好设置作为 Resource 注入。

Tool 适用场景：(1) 天气查询——需要城市参数并返回计算结果；(2) 发送邮件——有明确的副作用；(3) 数据库查询——需要 SQL 参数，返回动态结果集。

边界案例：如果需要"从知识库搜索文档"，这既是数据获取又涉及参数计算——建议用 Tool，因为搜索需要 query 参数且结果集动态变化，不符合 Resource 的"静态 URI 可寻址"特性。