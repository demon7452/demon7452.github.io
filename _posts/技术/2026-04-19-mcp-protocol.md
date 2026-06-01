---
layout: post
title: "MCP协议实战：构建Server与Client全流程"
date: 2026-04-19
categories: [AI Agent, MCP, 应用开发]
tags: [MCP, Model Context Protocol, Server, Client, Python, asyncio]
---

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，旨在标准化 AI 模型与外部工具、数据源之间的交互方式。它解决了「每个 LLM 平台都有一套自己的 Function Calling 格式」的碎片化问题。2026年，MCP 已经被 OpenAI、Google 等主流平台采纳，成为 Agent 开发的通用基础设施。

## MCP 协议架构

```
┌─────────────┐     JSON-RPC 2.0      ┌─────────────┐
│  MCP Client  │ ◄──────────────────► │  MCP Server  │
│  (AI Host)   │   via stdio / SSE    │  (Tool Host) │
└─────────────┘                       └─────────────┘
       │                                      │
       ▼                                      ▼
  LLM 模型                              外部 API / 数据库
  (Claude/GPT)                          / 文件系统 / 服务
```

MCP 的核心通信基于 **JSON-RPC 2.0**，支持两种传输方式：
- **stdio**：标准输入输出，适合本地进程间通信
- **SSE**：Server-Sent Events，适合远程服务

## 构建 MCP Server

我们来实现一个天气预报 MCP Server，提供 `get_weather` 和 `list_cities` 两个工具。

### Server 端实现

```python
import asyncio
import json
import sys
from typing import Any

class MCPServer:
    """Minimal MCP Server implementation over stdio transport."""

    def __init__(self, name: str, version: str):
        self.name = name
        self.version = version
        self.tools: dict[str, dict] = {}
        self._handlers: dict[str, callable] = {}

    def register_tool(self, name: str, description: str,
                      input_schema: dict, handler: callable):
        """Register a tool that the server can expose to clients."""
        self.tools[name] = {
            "name": name,
            "description": description,
            "inputSchema": input_schema
        }
        self._handlers[name] = handler

    async def start(self):
        """Start the MCP server, listening on stdin."""
        # Send initialization message
        await self._send({
            "jsonrpc": "2.0",
            "method": "initialized",
            "params": {}
        })

        # Read JSON-RPC messages from stdin line by line
        reader = asyncio.StreamReader()
        protocol = asyncio.StreamReaderProtocol(reader)
        await asyncio.get_event_loop().connect_read_pipe(
            lambda: protocol, sys.stdin
        )

        while True:
            line = await reader.readline()
            if not line:
                break
            await self._handle_message(json.loads(line.decode()))

    async def _handle_message(self, msg: dict):
        """Route JSON-RPC messages to appropriate handlers."""
        msg_id = msg.get("id")
        method = msg.get("method")

        if method == "tools/list":
            await self._send({
                "jsonrpc": "2.0",
                "id": msg_id,
                "result": {"tools": list(self.tools.values())}
            })

        elif method == "tools/call":
            tool_name = msg["params"]["name"]
            arguments = msg["params"].get("arguments", {})
            result = await self._call_tool(tool_name, arguments)
            await self._send({
                "jsonrpc": "2.0",
                "id": msg_id,
                "result": {
                    "content": [{"type": "text", "text": json.dumps(result)}]
                }
            })

    async def _call_tool(self, name: str, arguments: dict) -> Any:
        handler = self._handlers.get(name)
        if not handler:
            return {"error": f"Unknown tool: {name}"}
        if asyncio.iscoroutinefunction(handler):
            return await handler(**arguments)
        return handler(**arguments)

    async def _send(self, message: dict):
        sys.stdout.write(json.dumps(message, ensure_ascii=False) + "\n")
        sys.stdout.flush()


# ---------- 业务逻辑 ----------

async def get_weather(city: str, unit: str = "celsius") -> dict:
    """Simulate weather API call."""
    # 实际项目中替换为真实的天气 API 调用
    weather_data = {
        "北京": {"celsius": 22, "fahrenheit": 72, "condition": "晴"},
        "上海": {"celsius": 26, "fahrenheit": 79, "condition": "多云"},
        "深圳": {"celsius": 30, "fahrenheit": 86, "condition": "阵雨"},
    }
    default = {"celsius": 20, "fahrenheit": 68, "condition": "未知"}
    city_data = weather_data.get(city, default)
    return {
        "city": city,
        "temperature": city_data[unit] if unit in ("celsius", "fahrenheit") else city_data["celsius"],
        "unit": unit,
        "condition": city_data["condition"]
    }

async def list_cities() -> dict:
    return {"cities": ["北京", "上海", "深圳", "广州", "成都", "杭州"]}


# ---------- 启动 Server ----------

async def main():
    server = MCPServer(name="weather-service", version="1.0.0")

    server.register_tool(
        name="get_weather",
        description="获取指定城市的天气信息，支持摄氏度和华氏度",
        input_schema={
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"],
                         "description": "温度单位"}
            },
            "required": ["city"]
        },
        handler=get_weather
    )

    server.register_tool(
        name="list_cities",
        description="列出所有支持查询天气的城市",
        input_schema={"type": "object", "properties": {}},
        handler=list_cities
    )

    await server.start()

if __name__ == "__main__":
    asyncio.run(main())
```

## 构建 MCP Client

Client 负责与 LLM 交互，将模型输出的 tool call 转发给 MCP Server，再把结果返回模型。

```python
import asyncio
import json
import subprocess

class MCPClient:
    """MCP Client that manages connection to an MCP Server via subprocess."""

    def __init__(self, server_command: list[str]):
        self.server_command = server_command
        self.process: subprocess.Popen | None = None
        self.tools: list[dict] = []
        self._request_id = 0
        self._pending: dict[int, asyncio.Future] = {}

    async def connect(self):
        """Launch MCP Server as subprocess and perform handshake."""
        self.process = await asyncio.create_subprocess_exec(
            *self.server_command,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        # 监听 stdout
        asyncio.create_task(self._read_responses())

        # 等待初始化完成
        init_msg = await self._read_next_message(timeout=5.0)
        assert init_msg.get("method") == "initialized", \
            f"Expected 'initialized', got: {init_msg}"

        # 获取工具列表
        self.tools = await self.list_tools()
        print(f"Connected. Available tools: {[t['name'] for t in self.tools]}")

    async def list_tools(self) -> list[dict]:
        result = await self._send_request("tools/list", {})
        return result.get("tools", [])

    async def call_tool(self, name: str, arguments: dict) -> dict:
        result = await self._send_request("tools/call", {
            "name": name,
            "arguments": arguments
        })
        # Parse text content from MCP response
        for item in result.get("content", []):
            if item["type"] == "text":
                return json.loads(item["text"])
        return result

    async def _send_request(self, method: str, params: dict) -> dict:
        self._request_id += 1
        req_id = self._request_id
        request = {
            "jsonrpc": "2.0",
            "id": req_id,
            "method": method,
            "params": params
        }

        future: asyncio.Future = asyncio.get_event_loop().create_future()
        self._pending[req_id] = future

        self.process.stdin.write(
            (json.dumps(request, ensure_ascii=False) + "\n").encode()
        )
        await self.process.stdin.drain()

        return await asyncio.wait_for(future, timeout=30.0)

    async def _read_responses(self):
        """Continuously read responses from server's stdout."""
        while self.process and self.process.stdout:
            line = await self.process.stdout.readline()
            if not line:
                break
            msg = json.loads(line.decode())
            msg_id = msg.get("id")
            if msg_id and msg_id in self._pending:
                future = self._pending.pop(msg_id)
                if "result" in msg:
                    future.set_result(msg["result"])
                elif "error" in msg:
                    future.set_exception(RuntimeError(msg["error"]))

    async def _read_next_message(self, timeout: float) -> dict:
        """Read one JSON-RPC message from stdout (for initialization)."""
        raw = await asyncio.wait_for(
            self.process.stdout.readline(), timeout=timeout
        )
        return json.loads(raw.decode())

    async def close(self):
        if self.process:
            self.process.terminate()
            await self.process.wait()


# ---------- 使用示例 ----------

async def demo():
    client = MCPClient(["python", "weather_server.py"])
    await client.connect()

    # 调用工具
    weather = await client.call_tool("get_weather", {"city": "深圳"})
    print(f"深圳天气: {weather}")

    cities = await client.call_tool("list_cities", {})
    print(f"支持城市: {cities['cities']}")

    await client.close()

if __name__ == "__main__":
    asyncio.run(demo())
```

## MCP vs 传统 Function Calling

| 维度 | 传统 Function Calling | MCP 协议 |
|------|----------------------|----------|
| 标准化 | 各平台格式不同（OpenAI / Anthropic / Google） | 统一 JSON-RPC 2.0 |
| 工具发现 | 硬编码在请求中 | 动态 `tools/list` 发现 |
| 资源管理 | 无标准 | `resources/list` + `resources/read` |
| 提示模板 | 无标准 | `prompts/list` + `prompts/get` |
| 传输层 | API 耦合 | stdio / SSE 可插拔 |
| 生态复用 | 每个平台写一套 | 一次开发，多平台使用 |

## 生产环境考量

1. **错误处理**：对 JSON-RPC error 对象做完整的类型守卫，避免 Server 崩溃导致 Client 卡死。
2. **超时与重试**：为每个 `tools/call` 设置合理超时，对幂等操作实现自动重试。
3. **认证**：MCP 协议本身不定义认证机制，生产环境建议在传输层加 mTLS 或 API Key 校验。
4. **日志与监控**：记录每次工具调用的耗时、成功率和错误类型，用于性能优化和问题排查。

## 面试题

**1. MCP 协议为什么要选择 JSON-RPC 2.0 作为通信协议？与 RESTful API 相比有什么优势？**

<details>
<summary>点击展开答案</summary>

选择 JSON-RPC 2.0 的原因：

1. **双向通信天然支持**：JSON-RPC 的 request/response 模型天然适配「Client 发请求 → Server 返回结果」的模式，同时 Server 也可以主动发 notification（无 id 的消息）。RESTful 的 HTTP 请求-响应模型是单向的，Server 无法主动推送。

2. **传输层无关**：JSON-RPC 2.0 不绑定 HTTP，可以跑在 stdio、WebSocket、SSE 等多种传输层上。MCP 利用这一点实现了本地进程通信（stdio）和远程服务通信（SSE）的统一。

3. **极简规范**：整个 JSON-RPC 2.0 规范只有一页纸，实现成本低，调试方便。相比之下 RESTful 在资源设计、状态码、HATEOAS 等方面有大量约定，适合 Web API 但对 Agent-Tool 通信来说过于厚重。

4. **批量请求**：JSON-RPC 2.0 支持 Batch（数组形式的多个请求），MCP 可以利用这个特性减少往返次数。

</details>

**2. 在实现 MCP Client 时，如果 MCP Server 是一个远程服务（通过 SSE/HTTP 连接），与本地 stdio 连接相比，需要考虑哪些额外的工程问题？**

<details>
<summary>点击展开答案</summary>

远程连接相比 stdio 需要额外考虑：

1. **网络可靠性**：需要实现连接断开后的重连机制（指数退避），以及请求超时和重试策略。stdio 模式下进程崩溃是明确的信号，但网络连接可能处于半开状态。

2. **认证与安全**：stdio 依赖操作系统进程隔离，远程连接必须加认证层（Bearer Token、mTLS）。还需要考虑传输加密（TLS），防止工具参数中的敏感数据被窃听。

3. **并发连接管理**：本地 stdio 通常是一对一，远程 Server 需要处理多个 Client 同时连接，要考虑连接池、请求排队和资源隔离。

4. **SSE 的长连接保活**：SSE 连接可能被中间代理（nginx、负载均衡器）超时关闭，需要实现心跳机制（定期发送空注释行 `: heartbeat`）保持连接活跃。

5. **工具调用的幂等性**：网络环境下重试不可避免，需要确保非幂等操作（如创建资源）有去重机制（如请求 ID 去重）。

</details>