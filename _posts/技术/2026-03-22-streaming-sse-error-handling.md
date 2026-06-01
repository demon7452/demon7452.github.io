---
layout: post
title: "流式输出：SSE协议细节与错误处理"
category: 技术
tags: [流式输出, SSE, Server-Sent Events, LLM流式, 错误处理]
keywords: [SSE, Streaming, Server-Sent Events, LLM Streaming, Error Handling, Chunked Response]
description: "深入讲解LLM流式输出的实现原理，包括SSE协议细节、客户端实现、错误处理和断点续传策略。"
date: 2026-03-22
---

# 流式输出：SSE协议细节与错误处理

## 引言

流式输出（Streaming）是提升AI产品用户体验的关键技术。将LLM响应逐Token返回，用户瞬间看到结果，感知延迟从"几秒"降至"即时"。本文深入讲解SSE协议、流式Agent实现、以及生产环境的错误处理策略。

## 一、SSE协议基础

### 1.1 SSE协议格式

```python
"""
SSE (Server-Sent Events) 协议格式：

data: 消息内容\n
id: 事件ID\n
event: 事件类型\n
retry: 重连时间(ms)\n
\n

规则：
1. 每个字段以 field: value 格式
2. 每条消息以双换行 \n\n 结束
3. data 可以多行（表示同一条消息的不同部分）
4. event 默认为 "message"
"""

# SSE流示例：
"""
data: {"token": "Hello", "index": 0}
\n
data: {"token": " World", "index": 1}
\n
event: done
data: {"status": "completed", "total_tokens": 150}
\n
"""
```

### 1.2 服务端实现

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
import asyncio
import json
import time

app = FastAPI()

# ========== 基础SSE服务端 ==========
@app.post("/api/chat/stream")
async def chat_stream(request: Request):
    """SSE流式聊天接口"""
    body = await request.json()
    messages = body.get("messages", [])
    
    async def generate():
        """SSE生成器"""
        try:
            # 1. 发送开始事件
            yield f"event: start\ndata: {json.dumps({'status': 'started'})}\n\n"
            
            # 2. 流式生成Token
            async for token_data in stream_tokens(messages):
                # 2.1 发送Token数据
                yield f"data: {json.dumps(token_data)}\n\n"
                
                # 2.2 发送进度更新（event区分）
                if token_data.get("index", 0) % 10 == 0:
                    yield (
                        f"event: progress\n"
                        f"data: {json.dumps({'progress': token_data['index']})}\n\n"
                    )
            
            # 3. 发送完成事件
            yield (
                f"event: done\n"
                f"data: {json.dumps({'status': 'completed', 'total_tokens': 150})}\n\n"
            )
            
        except Exception as e:
            # 4. 发送错误事件
            yield (
                f"event: error\n"
                f"data: {json.dumps({'error': str(e), 'code': 'STREAM_ERROR'})}\n\n"
            )
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # 禁用Nginx缓冲
        }
    )

async def stream_tokens(messages: list):
    """模拟Token流式生成"""
    sample_response = "AI Agent是构建在大型语言模型之上的智能系统，能够自主感知环境、制定计划、调用工具并执行任务。"
    
    for i, char in enumerate(sample_response):
        await asyncio.sleep(0.02)  # 模拟延迟
        yield {
            "token": char,
            "index": i,
            "timestamp": time.time()
        }
```

### 1.3 客户端实现

```python
class SSEClient:
    """SSE客户端——处理流式事件"""
    
    def __init__(self):
        self.buffer = ""
        self.event_handlers = {
            "message": self._handle_data,
            "progress": self._handle_progress,
            "done": self._handle_done,
            "error": self._handle_error,
            "start": self._handle_start
        }
    
    async def connect(self, url: str, data: dict, timeout: int = 30):
        """建立SSE连接"""
        import aiohttp
        
        self.timeout = timeout
        self.full_response = []
        
        async with aiohttp.ClientSession() as session:
            async with session.post(
                url, 
                json=data,
                timeout=aiohttp.ClientTimeout(total=timeout)
            ) as response:
                async for line in response.content:
                    await self._process_line(line.decode("utf-8"))
        
        return "".join(self.full_response)
    
    async def _process_line(self, line: str):
        """处理SSE行"""
        line = line.rstrip()
        
        # 空行 = 事件结束
        if line == "" and self.buffer:
            await self._process_event(self.buffer)
            self.buffer = ""
            return
        
        # 追加到缓冲区
        self.buffer += line + "\n"
    
    async def _process_event(self, event_text: str):
        """处理完整的SSE事件"""
        event_type = "message"  # 默认事件类型
        data = ""
        
        for line in event_text.strip().split("\n"):
            if line.startswith("event:"):
                event_type = line[6:].strip()
            elif line.startswith("data:"):
                data += line[5:].strip()
        
        # 调用对应处理器
        handler = self.event_handlers.get(
            event_type, 
            self._handle_unknown
        )
        
        await handler(data, event_type)
    
    async def _handle_start(self, data: str, event_type: str):
        """处理开始事件"""
        parsed = json.loads(data)
        print(f"Stream started: {parsed}")
    
    async def _handle_data(self, data: str, event_type: str):
        """处理Token数据"""
        parsed = json.loads(data)
        token = parsed.get("token", "")
        self.full_response.append(token)
        
        # 实时输出（展示流式效果）
        print(token, end="", flush=True)
    
    async def _handle_progress(self, data: str, event_type: str):
        """处理进度更新"""
        parsed = json.loads(data)
        print(f"\n[Progress: {parsed.get('progress')}]")
    
    async def _handle_done(self, data: str, event_type: str):
        """处理完成事件"""
        parsed = json.loads(data)
        print(f"\nStream completed: {parsed}")
    
    async def _handle_error(self, data: str, event_type: str):
        """处理错误"""
        parsed = json.loads(data)
        raise Exception(f"Stream error: {parsed.get('error')}")
    
    async def _handle_unknown(self, data: str, event_type: str):
        """处理未知事件"""
        print(f"Unknown event [{event_type}]: {data}")


# ========== 使用OpenAI原生SSE客户端 ==========
from openai import OpenAI

class OpenAIStreamClient:
    """OpenAI流式客户端（带错误处理）"""
    
    def __init__(self, api_key: str):
        self.client = OpenAI(api_key=api_key)
    
    def stream_chat(
        self, 
        messages: list, 
        model: str = "gpt-4o",
        on_token: callable = None
    ) -> str:
        """
        流式聊天
        
        Args:
            messages: 消息列表
            on_token: Token回调函数（可选）
        
        Returns:
            完整响应文本
        """
        full_response = []
        
        try:
            stream = self.client.chat.completions.create(
                model=model,
                messages=messages,
                stream=True,
                stream_options={"include_usage": True}
            )
            
            for chunk in stream:
                # 处理不同chunk类型
                if chunk.choices and chunk.choices[0].delta.content:
                    token = chunk.choices[0].delta.content
                    full_response.append(token)
                    
                    if on_token:
                        on_token(token)
                
                # 处理Token用量统计
                if hasattr(chunk, "usage") and chunk.usage:
                    print(f"\nUsage: {chunk.usage}")
            
        except Exception as e:
            # 流式中断处理
            raise StreamError(f"Stream interrupted: {str(e)}")
        
        return "".join(full_response)
    
    def stream_with_retry(
        self, 
        messages: list, 
        max_retries: int = 3
    ) -> str:
        """带重试的流式调用"""
        for attempt in range(max_retries):
            try:
                return self.stream_chat(messages)
            except StreamError as e:
                if attempt == max_retries - 1:
                    raise
                print(f"Retry {attempt + 1}/{max_retries}: {e}")
                time.sleep(2 ** attempt)  # 指数退避
        
        return ""

class StreamError(Exception):
    """流式错误"""
    pass
```

## 二、高级流式场景

### 2.1 多路流式输出

```python
class MultiStreamAgent:
    """多路流式输出Agent"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", streaming=True)
    
    async def stream_with_tool_calls(self, query: str):
        """流式输出 + 工具调用"""
        
        async def event_generator():
            # 1. 流式输出思考过程（Thinking Stream）
            yield f"event: thinking\ndata: 正在分析问题...\n\n"
            
            # 2. 检测到需要工具调用
            tool_call = self._needs_tool(query)
            if tool_call:
                # 2.1 通知工具调用
                yield (
                    f"event: tool_call\n"
                    f"data: {json.dumps({'tool': tool_call['name'], 'args': tool_call['args']})}\n\n"
                )
                
                # 2.2 执行工具
                import asyncio
                await asyncio.sleep(0.5)  # 模拟工具执行
                
                # 2.3 工具结果
                result = f"工具 {tool_call['name']} 执行完成"
                yield (
                    f"event: tool_result\n"
                    f"data: {json.dumps({'result': result})}\n\n"
                )
            
            # 3. 流式输出最终回答（Answer Stream）
            sample = "根据分析结果，为您推荐以下方案..."
            for i, char in enumerate(sample):
                await asyncio.sleep(0.03)
                yield f"data: {json.dumps({'token': char, 'index': i})}\n\n"
            
            # 4. 完成
            yield f"event: done\ndata: {json.dumps({'status': 'ok'})}\n\n"
        
        return StreamingResponse(
            event_generator(),
            media_type="text/event-stream"
        )
    
    def _needs_tool(self, query: str) -> dict:
        """判断是否需要工具调用"""
        if "天气" in query:
            return {"name": "get_weather", "args": {"city": "北京"}}
        return None


# 多路流式客户端
class MultiStreamClient:
    """多路流式客户端"""
    
    def __init__(self):
        self.thinking_text = []
        self.answer_text = []
        self.tool_calls = []
    
    async def consume(self, url: str, data: dict):
        """消费多路流式输出"""
        import aiohttp
        
        async with aiohttp.ClientSession() as session:
            async with session.post(url, json=data) as response:
                async for line in response.content:
                    line = line.decode("utf-8").rstrip()
                    
                    if line.startswith("event: thinking"):
                        print("[Thinking] 正在思考...")
                    
                    elif line.startswith("event: tool_call"):
                        tool_data = line.split("data: ")[1] if "data: " in line else "{}"
                        tool = json.loads(tool_data)
                        self.tool_calls.append(tool)
                        print(f"[Tool] 调用 {tool.get('tool')}: {tool.get('args')}")
                    
                    elif line.startswith("event: tool_result"):
                        result_data = line.split("data: ")[1] if "data: " in line else "{}"
                        result = json.loads(result_data)
                        print(f"[Tool Result] {result.get('result')}")
                    
                    elif line.startswith("data: "):
                        token_data = line.split("data: ")[1]
                        token = json.loads(token_data).get("token", "")
                        self.answer_text.append(token)
                        print(token, end="", flush=True)
                    
                    elif line.startswith("event: done"):
                        print("\n[Done]")
```

### 2.2 流式断点续传

```python
class StreamResumableClient:
    """支持断点续传的流式客户端"""
    
    def __init__(self):
        self.received_tokens = 0
        self.checkpoint = None
        self.max_retries = 5
    
    async def resume_from_checkpoint(
        self, 
        messages: list,
        checkpoint: dict = None
    ) -> str:
        """
        从checkpoint恢复流式接收
        
        checkpoint格式：
        {
            "received_tokens": 150,
            "last_message_id": "msg_abc123",
            "partial_response": "已完成的部分文本"
        }
        """
        
        if checkpoint:
            self.received_tokens = checkpoint["received_tokens"]
            self.partial_response = checkpoint["partial_response"]
            
            # 请求时告知服务端从指定位置继续
            messages.append({
                "role": "system",
                "content": f"请从第{self.received_tokens}个token之后继续。"
            })
        
        try:
            remaining_response = await self._stream_with_timeout(messages)
            
            if checkpoint:
                return checkpoint["partial_response"] + remaining_response
            return remaining_response
            
        except StreamTimeoutError:
            # 超时后保存checkpoint
            self.checkpoint = {
                "received_tokens": self.received_tokens,
                "partial_response": self.partial_response,
                "timestamp": time.time()
            }
            raise
    
    async def _stream_with_timeout(
        self, 
        messages: list, 
        timeout: float = 30.0
    ) -> str:
        """带超时的流式请求"""
        import asyncio
        
        try:
            response = await asyncio.wait_for(
                self._do_stream(messages),
                timeout=timeout
            )
            return response
        except asyncio.TimeoutError:
            raise StreamTimeoutError(
                f"Stream timeout after {timeout}s",
                checkpoint=self.checkpoint
            )

class StreamTimeoutError(Exception):
    """流式超时异常"""
    
    def __init__(self, message: str, checkpoint: dict):
        super().__init__(message)
        self.checkpoint = checkpoint
```

## 三、错误处理策略

### 3.1 完整错误处理框架

```python
from enum import Enum

class StreamErrorType(Enum):
    NETWORK_ERROR = "network"        # 网络中断
    TIMEOUT_ERROR = "timeout"        # 超时
    MODEL_ERROR = "model"            # 模型错误
    RATE_LIMIT = "rate_limit"        # 频率限制
    CONTENT_FILTER = "content_filter"  # 内容过滤
    UNKNOWN = "unknown"              # 未知错误

class StreamErrorHandler:
    """流式错误处理器"""
    
    def __init__(self):
        self.fallback_text = "抱歉，生成回复时遇到问题，请稍后重试。"
    
    def handle(
        self, 
        error_type: StreamErrorType, 
        partial_response: str,
        attempt: int
    ) -> dict:
        """
        处理流式错误
        
        Returns:
            {
                "action": "retry/fallback/end",
                "delay": 2,  # 重试延迟(秒)
                "message": "...,"
            }
        """
        
        handlers = {
            StreamErrorType.NETWORK_ERROR: self._handle_network,
            StreamErrorType.TIMEOUT_ERROR: self._handle_timeout,
            StreamErrorType.RATE_LIMIT: self._handle_rate_limit,
            StreamErrorType.MODEL_ERROR: self._handle_model_error,
            StreamErrorType.CONTENT_FILTER: self._handle_content_filter,
            StreamErrorType.UNKNOWN: self._handle_unknown
        }
        
        handler = handlers.get(error_type, self._handle_unknown)
        return handler(partial_response, attempt)
    
    def _handle_network(self, partial: str, attempt: int) -> dict:
        """网络错误：重试"""
        if attempt < 3:
            return {
                "action": "retry",
                "delay": 2 ** attempt,
                "message": "网络中断，正在重试..."
            }
        return {
            "action": "fallback",
            "message": partial + "\n\n[回复中断]" if partial else self.fallback_text
        }
    
    def _handle_timeout(self, partial: str, attempt: int) -> dict:
        """超时：使用部分结果或重试"""
        if partial and len(partial) > 50:
            return {
                "action": "partial",
                "message": partial + "..."
            }
        elif attempt < 2:
            return {
                "action": "retry",
                "delay": 1,
                "message": "请求超时，正在重试..."
            }
        return {"action": "fallback", "message": self.fallback_text}
    
    def _handle_rate_limit(self, partial: str, attempt: int) -> dict:
        """频率限制：等待后重试"""
        if attempt < 3:
            return {
                "action": "retry",
                "delay": 5 * (attempt + 1),  # 5, 10, 15秒
                "message": "请求频繁，稍等片刻..."
            }
        return {"action": "fallback", "message": self.fallback_text}
    
    def _handle_model_error(self, partial: str, attempt: int) -> dict:
        """模型错误：重试或降级"""
        if attempt < 2:
            return {
                "action": "retry",
                "delay": 1,
                "message": "模型暂时不可用，正在重试..."
            }
        # 降级到备选模型
        return {
            "action": "switch_model",
            "fallback_model": "gpt-4o-mini",
            "message": "正在切换到备选模型..."
        }
    
    def _handle_content_filter(self, partial: str, attempt: int) -> dict:
        """内容过滤：不重试"""
        return {
            "action": "end",
            "message": "回复内容不符合安全策略，无法生成。"
        }
    
    def _handle_unknown(self, partial: str, attempt: int) -> dict:
        """未知错误"""
        if attempt < 1:
            return {
                "action": "retry",
                "delay": 2,
                "message": "发生未知错误，正在重试..."
            }
        return {"action": "fallback", "message": self.fallback_text}
```

### 3.2 健壮的流式Agent

```python
class RobustStreamAgent:
    """健壮的流式Agent"""
    
    def __init__(self):
        self.error_handler = StreamErrorHandler()
        self.primary_model = ChatOpenAI(model="gpt-4o", streaming=True)
        self.fallback_model = ChatOpenAI(model="gpt-4o-mini", streaming=True)
    
    async def chat(
        self, 
        query: str, 
        on_token: callable = None,
        on_error: callable = None
    ) -> str:
        """健壮的流式聊天"""
        
        messages = [HumanMessage(content=query)]
        partial_response = ""
        attempt = 0
        
        while True:
            try:
                model = self.primary_model if attempt == 0 else self.fallback_model
                
                async for chunk in model.astream(messages):
                    if chunk.content:
                        partial_response += chunk.content
                        if on_token:
                            on_token(chunk.content)
                
                # 成功完成
                return partial_response
                
            except Exception as e:
                error_type = self._classify_error(e)
                
                result = self.error_handler.handle(
                    error_type, 
                    partial_response, 
                    attempt
                )
                
                if result["action"] == "retry":
                    attempt += 1
                    if on_error:
                        on_error(result["message"])
                    await asyncio.sleep(result["delay"])
                    continue
                
                elif result["action"] == "switch_model":
                    attempt += 1
                    if on_error:
                        on_error(result["message"])
                    continue
                
                elif result["action"] == "partial":
                    return result["message"]
                
                else:  # fallback or end
                    if on_error:
                        on_error(result["message"])
                    return result["message"]
    
    def _classify_error(self, error: Exception) -> StreamErrorType:
        """分类错误"""
        error_str = str(error).lower()
        
        if "timeout" in error_str:
            return StreamErrorType.TIMEOUT_ERROR
        elif "rate" in error_str or "429" in error_str:
            return StreamErrorType.RATE_LIMIT
        elif "content" in error_str and "filter" in error_str:
            return StreamErrorType.CONTENT_FILTER
        elif "connection" in error_str or "network" in error_str:
            return StreamErrorType.NETWORK_ERROR
        elif "model" in error_str:
            return StreamErrorType.MODEL_ERROR
        else:
            return StreamErrorType.UNKNOWN
```

## 面试题

### 面试题1：请解释SSE协议与WebSocket的区别，为什么LLM流式输出通常选择SSE？

**答案：**

| 维度 | SSE | WebSocket |
|------|-----|-----------|
| 方向 | 单向（服务器→客户端） | 双向 |
| 协议 | HTTP（标准协议） | 独立协议（ws://） |
| 重连 | 自动（浏览器内置） | 需手动实现 |
| 代理支持 | 完美（HTTP代理） | 需特殊配置 |
| 复杂度 | 低 | 高 |
| 适用场景 | 服务端推送 | 实时双向通信 |

**为什么LLM流式输出选SSE：**

```python
# LLM流式输出的特点：单向数据流
"""
客户端请求 → 服务器开始生成 → 服务器推送Tokens → 结束

这种模式天然适合SSE：
1. 数据流方向单一（服务器→客户端）
2. HTTP协议兼容性好（无需特殊网络配置）
3. 浏览器原生支持（EventSource API）
4. 自动重连机制
"""

# 何时用WebSocket：
"""
需要双向通信的场景：
- 实时协作编辑
- 多人聊天
- 客服系统（用户可随时打断）
"""
```

### 面试题2：在流式输出中，如何优雅地处理生成内容被截断（如网络中断、用户取消）的情况？

**答案：**

```python
class GracefulStreamHandler:
    """优雅的流式中断处理"""
    
    def __init__(self):
        self.partial_response = ""
        self.is_complete = False
    
    async def stream_with_cancellation(
        self, 
        messages: list,
        cancel_event: asyncio.Event = None
    ) -> dict:
        """
        支持取消的流式处理
        
        Returns:
            {
                "status": "complete/partial/cancelled",
                "content": "文本内容",
                "truncated": false/true,
                "completed_tokens": 150
            }
        """
        
        try:
            async for chunk in self._stream_chunks(messages):
                # 检查取消信号
                if cancel_event and cancel_event.is_set():
                    return {
                        "status": "cancelled",
                        "content": self.partial_response,
                        "truncated": True,
                        "completed_tokens": len(self.partial_response)
                    }
                
                self.partial_response += chunk
            
            return {
                "status": "complete",
                "content": self.partial_response,
                "truncated": False,
                "completed_tokens": len(self.partial_response)
            }
            
        except asyncio.CancelledError:
            # 异步任务被取消
            return {
                "status": "cancelled",
                "content": self.partial_response,
                "truncated": True
            }
        
        except Exception as e:
            # 网络错误 → 尝试优雅降级
            if len(self.partial_response) > 100:
                # 部分结果足够长 → 尝试用LLM补全
                completion = await self._complete_partial(
                    self.partial_response
                )
                return {
                    "status": "partial_completed",
                    "content": completion,
                    "truncated": False
                }
            else:
                return {
                    "status": "error",
                    "content": "回复生成失败，请重试",
                    "truncated": True
                }
    
    async def _complete_partial(self, partial_text: str) -> str:
        """补全截断的文本"""
        prompt = f"""以下文本被意外截断，请自然地补全剩余内容：

{partial_text}

要求：
1. 保持原有风格和语气
2. 补全截断的句子
3. 自然收尾

只输出补全后的完整文本。
"""
        response = await self.llm.ainvoke(prompt)
        return response.content
```

**前端处理：**

```javascript
// 前端流式中断处理
class StreamClient {
    constructor() {
        this.abortController = null;
        this.partialText = '';
    }
    
    async streamWithCancel(query) {
        this.abortController = new AbortController();
        
        try {
            const response = await fetch('/api/chat/stream', {
                method: 'POST',
                body: JSON.stringify({ query }),
                signal: this.abortController.signal
            });
            
            const reader = response.body.getReader();
            
            while (true) {
                const { done, value } = await reader.read();
                if (done) break;
                
                const text = new TextDecoder().decode(value);
                this.partialText += text;
                this.updateUI(text);
            }
            
            // 完整响应
            return { status: 'complete', text: this.partialText };
            
        } catch (error) {
            if (error.name === 'AbortError') {
                // 用户取消 → 保留已显示的部分
                return { 
                    status: 'cancelled', 
                    text: this.partialText + '\n\n[已停止生成]' 
                };
            }
            
            // 网络错误 → 显示部分结果
            if (this.partialText.length > 50) {
                return { 
                    status: 'partial', 
                    text: this.partialText + '...' 
                };
            }
            
            return { status: 'error', text: '请求失败，请重试' };
        }
    }
    
    cancel() {
        if (this.abortController) {
            this.abortController.abort();
        }
    }
}
```

**三级降级策略：**

```
L1: 完整响应（理想状态）
L2: 部分响应补全（截断时）
L3: 显示部分结果 + 重试按钮（严重错误时）
```

## 总结

1. **SSE协议**是LLM流式输出的最佳选择，简单高效，兼容性好
2. **事件类型区分**（data/progress/done/error）让客户端能有针对性地处理不同消息
3. **多路流式**支持工具调用、思考过程等复杂场景的实时展示
4. **健壮的错误处理**是生产级流式应用的关键，需要分级降级策略
5. **用户体验**优先：优先显示部分结果，而非直接报错

## 参考资料

- [SSE Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [OpenAI Streaming](https://platform.openai.com/docs/api-reference/streaming)
- [FastAPI StreamingResponse](https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse)
