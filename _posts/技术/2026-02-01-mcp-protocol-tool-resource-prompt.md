---
layout: post
title: "MCP协议详解：Tool/Resource/Prompt三种能力"
category: 技术
tags: [MCP, Model Context Protocol, AI Agent, 工具协议, LLM集成]
keywords: [MCP, Model Context Protocol, Tool, Resource, Prompt, AI Agent, LLM Integration]
description: "深入解析MCP（Model Context Protocol）协议的三种核心能力：Tool、Resource和Prompt，并通过实战示例演示如何构建MCP Server和Client。"
date: 2026-02-01
---

# MCP协议详解：Tool/Resource/Prompt三种能力

## 引言

**MCP（Model Context Protocol）** 是由Anthropic提出的一种开放协议，旨在标准化AI应用程序与外部数据源和工具之间的通信方式。它类似于"AI应用领域的USB-C接口"，让不同的LLM客户端能够以统一的方式连接各种后端服务。

本文将深入解析MCP的三种核心能力：**Tool（工具）**、**Resource（资源）**和**Prompt（提示模板）**，并通过实战代码演示如何高效使用它们。

## MCP协议架构概述

### 核心架构

```
┌─────────────┐     MCP Protocol      ┌─────────────┐
│             │◄─────────────────────►│             │
│  MCP Client │   JSON-RPC over       │  MCP Server │
│  (LLM App)  │   stdio/HTTP          │  (Tool/DB)  │
│             │                       │             │
└─────────────┘                       └─────────────┘
```

**Client与Server通信采用JSON-RPC 2.0协议，支持两种传输方式：**
- **stdio**：标准输入输出（进程通信）
- **HTTP + SSE**：网络通信（远程服务）

### 三种核心能力对比

| 能力 | 用途 | 驱动方 | 典型示例 |
|------|------|--------|---------|
| **Tool** | LLM主动调用的函数 | 模型驱动 | 查天气、发邮件、操作数据库 |
| **Resource** | Client读取的结构化数据 | 应用驱动 | 读取文件、获取数据库Schema |
| **Prompt** | 预定义的提示模板 | 用户驱动 | 标准化提示、交互式工作流 |

## 能力一：Tool（工具）

### 概念

Tool是MCP中最核心的能力，它允许LLM调用服务器上的函数来执行操作。

### 实现示例：文件系统MCP Server

```python
# file_system_server.py
import json
import os
import sys
from typing import Any

class MCPFileSystemServer:
    """MCP文件系统服务器"""
    
    def __init__(self):
        self.tools = {
            "read_file": {
                "name": "read_file",
                "description": "读取指定文件的内容",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "path": {
                            "type": "string",
                            "description": "文件路径"
                        }
                    },
                    "required": ["path"]
                }
            },
            "list_directory": {
                "name": "list_directory",
                "description": "列出目录内容",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "path": {
                            "type": "string",
                            "description": "目录路径"
                        },
                        "recursive": {
                            "type": "boolean",
                            "description": "是否递归",
                            "default": False
                        }
                    },
                    "required": ["path"]
                }
            },
            "search_files": {
                "name": "search_files",
                "description": "按文件名模式搜索文件",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "pattern": {
                            "type": "string",
                            "description": "文件匹配模式，如 *.py"
                        },
                        "directory": {
                            "type": "string",
                            "description": "搜索目录"
                        }
                    },
                    "required": ["pattern"]
                }
            }
        }
    
    def handle_request(self, request: dict) -> dict:
        """处理JSON-RPC请求"""
        method = request.get("method", "")
        req_id = request.get("id")
        
        if method == "initialize":
            return self._handle_initialize(req_id)
        elif method == "tools/list":
            return self._handle_tools_list(req_id)
        elif method == "tools/call":
            return self._handle_tool_call(request["params"], req_id)
        else:
            return self._error_response(req_id, -32601, "Method not found")
    
    def _handle_initialize(self, req_id: Any) -> dict:
        """初始化握手"""
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "result": {
                "protocolVersion": "2024-11-05",
                "capabilities": {
                    "tools": {}
                },
                "serverInfo": {
                    "name": "file-system-server",
                    "version": "1.0.0"
                }
            }
        }
    
    def _handle_tools_list(self, req_id: Any) -> dict:
        """返回工具列表"""
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "result": {
                "tools": list(self.tools.values())
            }
        }
    
    def _handle_tool_call(self, params: dict, req_id: Any) -> dict:
        """调用工具"""
        tool_name = params["name"]
        tool_args = params.get("arguments", {})
        
        try:
            if tool_name == "read_file":
                result = self._read_file(tool_args["path"])
            elif tool_name == "list_directory":
                result = self._list_directory(
                    tool_args["path"],
                    tool_args.get("recursive", False)
                )
            elif tool_name == "search_files":
                result = self._search_files(
                    tool_args["pattern"],
                    tool_args.get("directory", ".")
                )
            else:
                return self._error_response(req_id, -32602, f"Unknown tool: {tool_name}")
            
            return {
                "jsonrpc": "2.0",
                "id": req_id,
                "result": {
                    "content": [{"type": "text", "text": result}]
                }
            }
        except Exception as e:
            return {
                "jsonrpc": "2.0",
                "id": req_id,
                "result": {
                    "content": [{"type": "text", "text": f"Error: {str(e)}"}],
                    "isError": True
                }
            }
    
    def _read_file(self, path: str) -> str:
        with open(path, 'r', encoding='utf-8') as f:
            return f.read()
    
    def _list_directory(self, path: str, recursive: bool = False) -> str:
        items = os.listdir(path)
        result = []
        for item in items:
            full_path = os.path.join(path, item)
            item_type = "📁" if os.path.isdir(full_path) else "📄"
            size = os.path.getsize(full_path) if os.path.isfile(full_path) else 0
            result.append(f"{item_type} {item} ({self._format_size(size)})")
        return "\n".join(result)
    
    def _search_files(self, pattern: str, directory: str = ".") -> str:
        import fnmatch
        matches = []
        for root, _, files in os.walk(directory):
            for file in files:
                if fnmatch.fnmatch(file, pattern):
                    matches.append(os.path.join(root, file))
        return "\n".join(matches) if matches else "No matches found"
    
    @staticmethod
    def _format_size(size: int) -> str:
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size < 1024:
                return f"{size:.1f}{unit}"
            size /= 1024
        return f"{size:.1f}TB"
    
    @staticmethod
    def _error_response(req_id: Any, code: int, message: str) -> dict:
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "error": {"code": code, "message": message}
        }

# 启动服务器
if __name__ == "__main__":
    server = MCPFileSystemServer()
    
    # 从stdin读取请求
    for line in sys.stdin:
        request = json.loads(line)
        response = server.handle_request(request)
        print(json.dumps(response), flush=True)
```

## 能力二：Resource（资源）

### 概念

Resource用于暴露只读的结构化数据。与Tool不同，Resource是**应用驱动**的——客户端主动拉取数据，而Tool是**模型驱动**的。

### 实现示例：数据库Schema Resource

```python
class MCPDatabaseResourceServer:
    """MCP数据库资源服务器"""
    
    def __init__(self, db_connection_string: str):
        self.connection_string = db_connection_string
        
        # 定义资源
        self.resources = {
            "schema://tables": {
                "uri": "schema://tables",
                "name": "Table List",
                "description": "All database table names",
                "mimeType": "application/json"
            },
            "schema://tables/{table_name}": {
                "uri": "schema://tables/{table_name}",
                "name": "Table Schema",
                "description": "Schema for a specific table",
                "mimeType": "application/json"
            }
        }
    
    def handle_request(self, request: dict) -> dict:
        method = request.get("method", "")
        req_id = request.get("id")
        
        if method == "resources/list":
            return self._handle_resources_list(req_id)
        elif method == "resources/read":
            return self._handle_resource_read(request["params"], req_id)
        
        # ... 其他方法和之前相同
    
    def _handle_resources_list(self, req_id: Any) -> dict:
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "result": {
                "resources": list(self.resources.values())
            }
        }
    
    def _handle_resource_read(self, params: dict, req_id: Any) -> dict:
        uri = params["uri"]
        
        if uri == "schema://tables":
            content = self._get_all_tables()
        elif uri.startswith("schema://tables/"):
            table_name = uri.replace("schema://tables/", "")
            content = self._get_table_schema(table_name)
        else:
            return self._error_response(req_id, -32602, f"Unknown resource: {uri}")
        
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "result": {
                "contents": [{
                    "uri": uri,
                    "mimeType": "application/json",
                    "text": json.dumps(content, indent=2)
                }]
            }
        }
    
    def _get_all_tables(self) -> list:
        """获取所有表名"""
        # 模拟数据库查询
        return [
            {"name": "users", "row_count": 15000},
            {"name": "orders", "row_count": 89000},
            {"name": "products", "row_count": 3200},
        ]
    
    def _get_table_schema(self, table_name: str) -> dict:
        """获取表结构"""
        schemas = {
            "users": {
                "columns": [
                    {"name": "id", "type": "INTEGER", "nullable": False},
                    {"name": "username", "type": "VARCHAR(100)", "nullable": False},
                    {"name": "email", "type": "VARCHAR(255)", "nullable": False},
                    {"name": "created_at", "type": "TIMESTAMP", "nullable": False}
                ],
                "primary_key": ["id"],
                "indexes": ["email"]
            },
            "orders": {
                "columns": [
                    {"name": "id", "type": "INTEGER", "nullable": False},
                    {"name": "user_id", "type": "INTEGER", "nullable": False},
                    {"name": "total", "type": "DECIMAL(10,2)", "nullable": False},
                    {"name": "status", "type": "VARCHAR(20)", "nullable": False}
                ],
                "primary_key": ["id"],
                "foreign_keys": [{"column": "user_id", "references": "users(id)"}]
            }
        }
        
        return schemas.get(table_name, {"error": f"Table '{table_name}' not found"})
```

## 能力三：Prompt（提示模板）

### 概念

Prompt能力允许MCP Server提供**预定义的提示模板**，让用户能以标准化的方式与LLM交互。

### 实现示例：代码审查Prompt模板

```python
class MCPCodeReviewServer:
    """MCP代码审查提示模板服务器"""
    
    def __init__(self):
        self.prompts = {
            "code_review": {
                "name": "code_review",
                "description": "生成代码审查提示",
                "arguments": [
                    {
                        "name": "language",
                        "description": "编程语言",
                        "required": True
                    },
                    {
                        "name": "focus_areas",
                        "description": "审查重点，逗号分隔",
                        "required": False
                    }
                ]
            },
            "bug_report": {
                "name": "bug_report",
                "description": "生成Bug报告模板",
                "arguments": [
                    {
                        "name": "severity",
                        "description": "严重程度：critical/high/medium/low",
                        "required": True
                    },
                    {
                        "name": "component",
                        "description": "相关组件名称",
                        "required": False
                    }
                ]
            }
        }
    
    def handle_request(self, request: dict) -> dict:
        method = request.get("method", "")
        req_id = request.get("id")
        
        if method == "prompts/list":
            return self._handle_prompts_list(req_id)
        elif method == "prompts/get":
            return self._handle_prompt_get(request["params"], req_id)
        
        # ... 其他方法
    
    def _handle_prompts_list(self, req_id: Any) -> dict:
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "result": {
                "prompts": list(self.prompts.values())
            }
        }
    
    def _handle_prompt_get(self, params: dict, req_id: Any) -> dict:
        prompt_name = params["name"]
        arguments = params.get("arguments", {})
        
        template = self._build_prompt(prompt_name, arguments)
        
        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "result": {
                "messages": [
                    {
                        "role": "user",
                        "content": {
                            "type": "text",
                            "text": template
                        }
                    }
                ]
            }
        }
    
    def _build_prompt(self, name: str, args: dict) -> str:
        if name == "code_review":
            language = args.get("language", "Python")
            focus = args.get("focus_areas", "代码质量,安全性,性能")
            
            return f"""请对以下{language}代码进行全面审查。

审查重点：{focus}

请按以下维度评估：
1. **代码质量**：命名规范、函数设计、注释
2. **安全性**：注入风险、敏感信息泄露、权限控制
3. **性能**：算法复杂度、内存使用、IO效率
4. **可维护性**：模块化、测试覆盖、文档

输出格式：每个维度给出等级评级（优秀/良好/需改进/严重问题）和具体建议。
"""
        
        elif name == "bug_report":
            severity = args.get("severity", "medium")
            component = args.get("component", "未指定")
            
            return f"""请帮助生成以下Bug的详细报告：

严重程度：{severity.upper()}
影响组件：{component}

报告应包含：
1. **问题描述**：清晰说明Bug表现
2. **复现步骤**：详细步骤
3. **预期行为**：正确的行为应该是什么
4. **实际行为**：当前的错误行为
5. **环境信息**：操作系统、浏览器、版本等
6. **建议修复**：技术层面的修复建议
"""
```

## 实战：构建完整MCP Client

```python
# mcp_client.py
import json
import subprocess
import asyncio
from typing import Optional

class MCPClient:
    """MCP客户端实现"""
    
    def __init__(self, server_command: list):
        """
        初始化MCP客户端
        
        Args:
            server_command: 启动MCP服务器的命令
                例如：["python", "file_system_server.py"]
        """
        self.process = subprocess.Popen(
            server_command,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        self.request_id = 0
        self.initialized = False
        
    def initialize(self) -> dict:
        """初始化连接"""
        response = self._send_request("initialize", {})
        self.initialized = True
        return response
    
    def list_tools(self) -> list:
        """获取可用工具列表"""
        response = self._send_request("tools/list", {})
        return response.get("tools", [])
    
    def call_tool(self, tool_name: str, arguments: dict) -> str:
        """调用工具"""
        response = self._send_request("tools/call", {
            "name": tool_name,
            "arguments": arguments
        })
        
        # 从响应中提取文本内容
        contents = response.get("content", [])
        for content in contents:
            if content.get("type") == "text":
                return content["text"]
        
        return json.dumps(response)
    
    def list_resources(self) -> list:
        """获取可用资源列表"""
        response = self._send_request("resources/list", {})
        return response.get("resources", [])
    
    def read_resource(self, uri: str) -> dict:
        """读取资源内容"""
        return self._send_request("resources/read", {"uri": uri})
    
    def list_prompts(self) -> list:
        """获取可用提示模板列表"""
        response = self._send_request("prompts/list", {})
        return response.get("prompts", [])
    
    def get_prompt(self, name: str, arguments: dict = None) -> str:
        """获取提示模板内容"""
        response = self._send_request("prompts/get", {
            "name": name,
            "arguments": arguments or {}
        })
        
        messages = response.get("messages", [])
        if messages:
            return messages[0]["content"]["text"]
        return ""
    
    def _send_request(self, method: str, params: dict) -> dict:
        """发送JSON-RPC请求"""
        self.request_id += 1
        
        request = {
            "jsonrpc": "2.0",
            "id": self.request_id,
            "method": method,
            "params": params
        }
        
        # 发送请求
        request_json = json.dumps(request) + "\n"
        self.process.stdin.write(request_json)
        self.process.stdin.flush()
        
        # 读取响应
        response_line = self.process.stdout.readline()
        response = json.loads(response_line)
        
        if "error" in response:
            raise MCPError(response["error"])
        
        return response.get("result", {})
    
    def close(self):
        """关闭连接"""
        self.process.stdin.close()
        self.process.terminate()


class MCPError(Exception):
    """MCP错误"""
    pass


# 使用示例
def demo_mcp_client():
    """演示MCP客户端使用"""
    
    # 创建客户端
    client = MCPClient(["python", "file_system_server.py"])
    
    try:
        # 1. 初始化连接
        init_result = client.initialize()
        print(f"Server: {init_result.get('serverInfo', {})}")
        
        # 2. 获取工具列表
        tools = client.list_tools()
        print(f"\n可用工具 ({len(tools)}):")
        for tool in tools:
            print(f"  - {tool['name']}: {tool['description']}")
        
        # 3. 调用工具
        result = client.call_tool("read_file", {"path": "README.md"})
        print(f"\n读取文件结果: {result[:200]}...")
        
        # 4. 获取资源
        resources = client.list_resources()
        print(f"\n可用资源 ({len(resources)}):")
        for resource in resources:
            print(f"  - {resource['name']}: {resource['uri']}")
        
        # 5. 获取提示模板
        prompts = client.list_prompts()
        print(f"\n可用提示模板 ({len(prompts)}):")
        for prompt in prompts:
            print(f"  - {prompt['name']}: {prompt['description']}")
        
    finally:
        client.close()
```

## 面试题

### 面试题1：请解释MCP协议中Tool、Resource、Prompt三种能力的区别和各自适用场景。

**答案：**

**核心区别：**

| 维度 | Tool | Resource | Prompt |
|------|------|----------|--------|
| **驱动方** | 模型驱动（LLM决定何时调用） | 应用驱动（程序主动读取） | 用户驱动（用户选择模板） |
| **数据流向** | 双向（执行并返回结果） | 单向（服务端→客户端） | 单向（模板注入到对话） |
| **是否有副作用** | 可能有（写操作） | 无（只读） | 无 |
| **返回内容** | 文本结果 | 结构化数据 | 提示文本 |

**适用场景：**

**Tool：**
```python
# 场景：LLM需要执行操作
- 查询实时天气 → tool: get_weather
- 发送邮件 → tool: send_email
- 操作数据库 → tool: execute_sql
```

**Resource：**
```python
# 场景：应用需要上下文数据
- 读取某个文件内容 → resource: file:///doc/readme.md
- 获取数据库表结构 → resource: schema://tables/users
- 获取系统状态信息 → resource: system://status
```

**Prompt：**
```python
# 场景：提供标准化的交互模板
- 代码审查 → prompt: code_review
- Bug报告 → prompt: bug_report
- 翻译任务 → prompt: translate
```

**选择决策树：**
```
需要LLM执行操作？
├── 是 → Tool
└── 否 → 需要提供上下文数据？
    ├── 是 → Resource
    └── 否 → 需要标准化交互？
        └── 是 → Prompt
```

### 面试题2：在MCP协议中，Client和Server之间如何进行安全认证和授权？请设计一个方案。

**答案：**

**方案设计：**

```python
import jwt
import time

class SecureMCPClient(MCPClient):
    """带认证的MCP客户端"""
    
    def __init__(self, server_command: list, api_key: str):
        super().__init__(server_command)
        self.api_key = api_key
        self.token = self._generate_token()
    
    def _generate_token(self) -> str:
        """生成JWT Token"""
        payload = {
            "client_id": f"mcp-client-{os.getpid()}",
            "exp": int(time.time()) + 3600,  # 1小时过期
            "iat": int(time.time()),
            "permissions": ["tools:read", "tools:execute", "resources:read"]
        }
        
        return jwt.encode(payload, self.api_key, algorithm="HS256")
    
    def _send_request(self, method: str, params: dict) -> dict:
        """发送带认证的请求"""
        # 在请求中添加认证信息
        auth_params = {
            **params,
            "_auth": {
                "token": self.token,
                "client_id": f"mcp-client-{os.getpid()}"
            }
        }
        
        return super()._send_request(method, auth_params)


class SecureMCPServer:
    """带认证的MCP服务器"""
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.allowed_origins = ["localhost", "trusted-service.com"]
    
    def authenticate(self, request: dict) -> bool:
        """验证请求"""
        auth_params = request.get("params", {}).get("_auth")
        
        if not auth_params:
            return False
        
        token = auth_params.get("token")
        client_id = auth_params.get("client_id")
        
        try:
            # 验证JWT
            payload = jwt.decode(token, self.api_key, algorithms=["HS256"])
            
            # 检查过期
            if payload.get("exp", 0) < time.time():
                return False
            
            # 检查权限
            required_permission = self._get_required_permission(request.get("method"))
            if required_permission not in payload.get("permissions", []):
                return False
            
            return True
            
        except jwt.InvalidTokenError:
            return False
    
    def _get_required_permission(self, method: str) -> str:
        """根据方法获取所需权限"""
        permission_map = {
            "tools/list": "tools:read",
            "tools/call": "tools:execute",
            "resources/list": "resources:read",
            "resources/read": "resources:read",
        }
        return permission_map.get(method, "basic")
```

**安全层次设计：**

| 层次 | 机制 | 说明 |
|------|------|------|
| L1 传输层 | TLS/SSL（HTTP模式下） | 加密传输 |
| L2 认证 | JWT Token | 身份验证 |
| L3 授权 | 基于权限的访问控制 | 细粒度权限 |
| L4 审计 | 操作日志 | 事后追溯 |

## 总结

MCP协议通过标准化的Tool、Resource、Prompt三种能力，为AI应用提供了统一的外部集成接口：

1. **Tool**：让LLM能执行操作——核心交互模式
2. **Resource**：为应用提供结构化数据——上下文的来源
3. **Prompt**：标准化交互模板——提升效率和质量

在实际项目中，建议先实现Tool能力满足基本需求，再逐步引入Resource提升数据集成效率，最后用Prompt实现标准化的工作流。

## 参考资料

- [MCP协议规范](https://spec.modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Anthropic MCP介绍](https://www.anthropic.com/news/model-context-protocol)
