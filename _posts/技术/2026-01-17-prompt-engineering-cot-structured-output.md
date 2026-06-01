---
layout: post
title: "Prompt Engineering：思维链(CoT)与结构化输出"
category: 技术
tags: [Prompt Engineering, Chain-of-Thought, 结构化输出, LLM应用]
keywords: [Prompt Engineering, CoT, Structured Output, Function Calling, JSON Mode]
description: "系统讲解思维链(CoT)提示工程技巧和结构化输出方法，帮助开发者提升LLM应用的可控性和可靠性。"
date: 2026-01-17
---

# Prompt Engineering：思维链(CoT)与结构化输出

## 引言

在AI Agent应用开发中，**Prompt Engineering（提示工程）** 是决定系统质量的关键技能。一个精心设计的Prompt可以让模型输出准确率提升30%以上，而糟糕的Prompt则会导致幻觉、格式错误、逻辑混乱等问题。

本文将系统讲解**思维链（Chain-of-Thought, CoT）** 提示技巧和**结构化输出**方法，帮助你构建更可控、更可靠的LLM应用。

## 核心概念

### 1. 什么是思维链（CoT）？

**思维链**是一种提示技术，通过引导模型"逐步思考"，显著提升复杂推理任务的准确率。

**Zero-shot CoT（零样本思维链）：**
```
请一步步思考，然后给出答案。
```

**Few-shot CoT（少样本思维链）：**
```
示例1:
问题：Roger有5个网球，他又买了2罐，每罐3个球。他现在有多少个球？
思考：Roger原有5个球。他买了2罐，每罐3个，所以买了2×3=6个新球。总共5+6=11个。
答案：11个

示例2:
问题：<你的问题>
思考：
```

### 2. 为什么CoT有效？

| 机制 | 说明 |
|------|------|
| **分解复杂问题** | 将多步推理拆解为单步，降低错误累积 |
| **自我纠错** | 逐步推导过程中可以自我检查 |
| **激活知识** | 逐步思考能更好地激活模型中的相关知识 |
| **可解释性** | 输出包含推理过程，便于调试 |

## 实战：CoT在不同场景的应用

### 场景1：数据分析Agent

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
import json

class DataAnalysisAgent:
    """数据分析Agent - 使用CoT进行数据解读"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
        
        # CoT Prompt模板
        self.prompt = ChatPromptTemplate.from_messages([
            ("system", """你是一个专业的数据分析师。
            
对于用户的每个问题，你必须按照以下步骤思考：
1. 理解问题：明确用户想了解什么
2. 分析数据：识别关键指标和趋势
3. 推导结论：基于数据得出合理结论
4. 给出建议：提供可操作的建议

请严格按照以下格式输出：
思考：<逐步推理过程>
分析：<数据分析结果>
结论：<核心结论>
建议：<行动建议>
"""),
            ("human", "{user_query}\n\n数据：\n{data}")
        ])
    
    def analyze(self, query: str, data: str) -> dict:
        """执行数据分析"""
        chain = self.prompt | self.llm
        response = chain.invoke({
            "user_query": query,
            "data": data
        })
        
        # 解析输出
        return self._parse_response(response.content)
    
    def _parse_response(self, text: str) -> dict:
        """解析CoT输出"""
        result = {}
        current_key = None
        current_value = []
        
        for line in text.split('\n'):
            line = line.strip()
            if line.startswith('思考：'):
                current_key = 'thinking'
                current_value = [line[3:].strip()]
            elif line.startswith('分析：'):
                if current_key:
                    result[current_key] = '\n'.join(current_value)
                current_key = 'analysis'
                current_value = [line[3:].strip()]
            elif line.startswith('结论：'):
                if current_key:
                    result[current_key] = '\n'.join(current_value)
                current_key = 'conclusion'
                current_value = [line[3:].strip()]
            elif line.startswith('建议：'):
                if current_key:
                    result[current_key] = '\n'.join(current_value)
                current_key = 'suggestion'
                current_value = [line[3:].strip()]
            elif current_key and line:
                current_value.append(line)
        
        # 保存最后一个key
        if current_key:
            result[current_key] = '\n'.join(current_value)
        
        return result

# 使用示例
def demo_data_analysis():
    agent = DataAnalysisAgent()
    
    data = """
    2025年Q4销售数据：
    - 10月：120万
    - 11月：150万
    - 12月：180万
    产品A占比：40%
    产品B占比：35%
    产品C占比：25%
    """
    
    result = agent.analyze("分析Q4销售趋势，并预测明年Q1", data)
    
    print("=== 思考过程 ===")
    print(result.get('thinking', ''))
    print("\n=== 分析结论 ===")
    print(result.get('conclusion', ''))
    print("\n=== 行动建议 ===")
    print(result.get('suggestion', ''))

if __name__ == "__main__":
    demo_data_analysis()
```

### 场景2：代码生成Agent（带自我审查）

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

class CodeGenerationAgent:
    """代码生成Agent - 使用CoT + 自我审查"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    def generate_code(self, requirement: str, language: str = "Python") -> dict:
        """
        生成代码（带CoT推理）
        
        Returns:
            {
                "thinking": "推理过程",
                "code": "生成的代码",
                "test_cases": "测试用例",
                "explanation": "代码说明"
            }
        """
        
        prompt = ChatPromptTemplate.from_messages([
            ("system", f"""你是一个{language}专家。
            
生成代码时，请遵循以下步骤：

步骤1：需求分析
- 明确输入/输出
- 识别边界条件
- 考虑异常处理

步骤2：算法设计
- 选择合适的数据结构
- 设计算法流程
- 分析时间/空间复杂度

步骤3：代码实现
- 编写清晰可读的代码
- 添加必要注释
- 遵循{language}最佳实践

步骤4：自我审查
- 检查边界条件处理
- 验证异常处理完整性
- 确认代码正确性

严格按照以下JSON格式输出：
{{
    "thinking": "步骤1/2/3/4的详细思考",
    "code": "完整代码",
    "test_cases": "测试用例",
    "explanation": "代码说明"
}}
"""),
            ("human", "需求：{requirement}")
        ])
        
        chain = prompt | self.llm
        response = chain.invoke({"requirement": requirement})
        
        # 解析JSON输出
        import json
        try:
            return json.loads(response.content)
        except:
            # 如果JSON解析失败，尝试提取
            return self._extract_json(response.content)
    
    def _extract_json(self, text: str) -> dict:
        """从文本中提取JSON"""
        import re
        json_match = re.search(r'\{.*\}', text, re.DOTALL)
        if json_match:
            return json.loads(json_match.group())
        return {"error": "JSON解析失败", "raw": text}

# 使用示例
def demo_code_generation():
    agent = CodeGenerationAgent()
    
    result = agent.generate_code(
        requirement="实现一个LRU缓存，支持get和put操作，时间复杂度O(1)",
        language="Python"
    )
    
    print("=== 思考过程 ===")
    print(result.get('thinking', ''))
    print("\n=== 生成代码 ===")
    print(result.get('code', ''))
    print("\n=== 测试用例 ===")
    print(result.get('test_cases', ''))

if __name__ == "__main__":
    demo_code_generation()
```

## 结构化输出技术

### 方法1：JSON Mode（推荐）

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field
from typing import List, Optional

# ========== 定义输出结构 ==========
class ExtractedEntity(BaseModel):
    """提取的实体"""
    name: str = Field(description="实体名称")
    type: str = Field(description="实体类型：人物/地点/组织/事件")
    confidence: float = Field(description="置信度 0-1")

class DocumentAnalysis(BaseModel):
    """文档分析结果"""
    summary: str = Field(description="文档摘要（100字以内）")
    entities: List[ExtractedEntity] = Field(description="提取的实体列表")
    sentiment: str = Field(description="情感倾向：正面/负面/中性")
    key_points: List[str] = Field(description="关键要点列表")
    word_count: int = Field(description="文档字数")

# ========== 使用JSON Mode ==========
def structured_extraction(text: str) -> DocumentAnalysis:
    """使用JSON Mode进行结构化提取"""
    
    llm = ChatOpenAI(
        model="gpt-4o",
        temperature=0,
        model_kwargs={
            "response_format": {"type": "json_object"}  # 启用JSON Mode
        }
    )
    
    prompt = f"""分析以下文档，输出严格的JSON格式。

要求：
1. 必须输出合法的JSON
2. 字段必须完整
3. 置信度为0-1之间的浮点数

文档内容：
{text}

输出JSON格式：
{{
    "summary": "...",
    "entities": [
        {{"name": "...", "type": "...", "confidence": 0.95}}
    ],
    "sentiment": "正面/负面/中性",
    "key_points": ["...", "..."],
    "word_count": 123
}}
"""
    
    response = llm.invoke(prompt)
    
    import json
    result = json.loads(response.content)
    return DocumentAnalysis(**result)
```

### 方法2：Function Calling（最稳定）

```python
from langchain_openai import ChatOpenAI
from langchain_core.utils.function_calling import convert_to_openai_function

class StructuredOutputWithFunction:
    """使用Function Calling实现结构化输出"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    def extract_with_schema(self, text: str, schema: dict) -> dict:
        """
        使用Function Calling强制输出结构
        
        Args:
            text: 待分析文本
            schema: JSON Schema定义
        """
        
        # 将schema转换为OpenAI function格式
        function = {
            "name": "extract_information",
            "description": "从文本中提取结构化信息",
            "parameters": schema
        }
        
        # 绑定function
        llm_with_functions = self.llm.bind(
            functions=[function],
            function_call={"name": "extract_information"}
        )
        
        prompt = f"从以下文本中提取信息：\n\n{text}"
        response = llm_with_functions.invoke(prompt)
        
        # 提取function_call的参数
        function_args = response.additional_kwargs["function_call"]["arguments"]
        
        import json
        return json.loads(function_args)

# 使用示例
def demo_function_calling():
    extractor = StructuredOutputWithFunction()
    
    # 定义输出schema
    schema = {
        "type": "object",
        "properties": {
            "summary": {"type": "string"},
            "entities": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "type": {"type": "string"}
                    }
                }
            }
        },
        "required": ["summary", "entities"]
    }
    
    text = "苹果公司CEO Tim Cook宣布将在上海开设新总部。"
    
    result = extractor.extract_with_schema(text, schema)
    print(json.dumps(result, ensure_ascii=False, indent=2))

if __name__ == "__main__":
    demo_function_calling()
```

### 方法3：LangChain Output Parser（最灵活）

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class AnswerWithConfidence(BaseModel):
    """带置信度的答案"""
    answer: str = Field(description="最终答案")
    confidence: float = Field(description="置信度 0-1", ge=0, le=1)
    reasoning: str = Field(description="推理过程")

def use_output_parser():
    """使用PydanticOutputParser"""
    
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    parser = PydanticOutputParser(pydantic_object=AnswerWithConfidence)
    
    prompt = ChatPromptTemplate.from_messages([
        ("system", "你是一个问答助手。{format_instructions}"),
        ("human", "{query}")
    ])
    
    chain = prompt | llm | parser
    
    result = chain.invoke({
        "query": "Python中如何读取大文件？",
        "format_instructions": parser.get_format_instructions()
    })
    
    print(f"答案：{result.answer}")
    print(f"置信度：{result.confidence}")
    print(f"推理：{result.reasoning}")

if __name__ == "__main__":
    use_output_parser()
```

## 最佳实践

### 1. CoT Prompt设计模板

```python
# ✅ 推荐：结构化CoT Prompt
COT_PROMPT_TEMPLATE = """你是一个{role}。

请按照以下步骤解决问题：

步骤1：{step_1_description}
- 具体行动1
- 具体行动2

步骤2：{step_2_description}
- 具体行动1
- 具体行动2

步骤3：{step_3_description}
...

输出格式：
{output_format}

现在，请解决以下问题：
{user_query}
"""

# ❌ 不推荐：模糊的CoT提示
BAD_COT = """请一步步思考，然后给出答案。
问题：{query}
"""
```

### 2. 结构化输出选择指南

| 方法 | 稳定性 | 灵活性 | 适用场景 |
|------|--------|--------|---------|
| **JSON Mode** | ⭐⭐⭐⭐ | ⭐⭐⭐ | 简单结构化输出 |
| **Function Calling** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 复杂结构、强制输出 |
| **Output Parser** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 需要自定义解析逻辑 |

### 3. 常见错误与修复

**错误1：JSON解析失败**

```python
# ❌ 问题代码
response = llm.invoke(prompt)
result = json.loads(response.content)  # 可能失败

# ✅ 修复方案：使用重试 + 提取
import json
import re

def robust_json_parse(text: str, max_retries: int = 3) -> dict:
    for attempt in range(max_retries):
        try:
            # 方法1：直接解析
            return json.loads(text)
        except:
            try:
                # 方法2：提取JSON块
                json_match = re.search(r'\{[^{}]*\{[^{}]*\}[^{}]*\}|\{.*\}', text, re.DOTALL)
                if json_match:
                    return json.loads(json_match.group())
            except:
                pass
            
            # 方法3：请求模型修复
            if attempt < max_retries - 1:
                fix_prompt = f"以下JSON有错误，请修复：\n{text}"
                text = llm.invoke(fix_prompt).content
    
    raise ValueError("无法解析JSON")
```

**错误2：输出不符合预期格式**

```python
# ✅ 使用Few-shot示例引导
FEW_SHOT_PROMPT = """以下是一些示例：

输入：分析苹果公司的财务状况
输出：
{{
    "company": "苹果公司",
    "analysis_type": "财务分析",
    "key_metrics": ["营收", "利润", "现金流"]
}}

现在请处理：
输入：{user_input}
输出：
"""
```

## 面试题

### 面试题1：请解释思维链（Chain-of-Thought）提示的工作原理，并说明在什么场景下应该使用Zero-shot CoT，什么场景下应该使用Few-shot CoT。

**答案：**

**思维链工作原理：**

思维链通过在Prompt中引导模型"逐步思考"，将复杂问题分解为多个中间步骤，让模型在输出最终答案前先展示推理过程。

**核心机制：**
1. **问题分解**：将多步推理任务拆解为单步
2. **中间状态**：每个推理步骤都有明确的中间结果
3. **错误修正**：逐步推导中可以自我纠正
4. **知识激活**：逐步思考更好地激活相关知识和推理能力

**Zero-shot CoT vs Few-shot CoT：**

| 维度 | Zero-shot CoT | Few-shot CoT |
|------|---------------|--------------|
| **使用方式** | 添加"Let's think step by step" | 提供2-5个带推理过程的示例 |
| **适用场景** | 通用任务、快速原型 | 专业领域、复杂推理 |
| **效果** | 中等提升（约20-30%） | 显著提升（约40-60%） |
| **成本** | 低（无示例token消耗） | 高（示例占用大量token） |
| **示例** | "问题：... 请一步步思考" | "示例1: 问题→思考→答案\n示例2: ..." |

**选择建议：**

```python
def choose_cot_strategy(task_type: str, has_examples: bool) -> str:
    if task_type in ["数学推理", "逻辑推理", "代码生成"]:
        return "Few-shot CoT"  # 需要展示推理模式
    elif task_type in ["文本分类", "情感分析"]:
        return "Zero-shot CoT"  # 任务相对简单
    elif has_examples:
        return "Few-shot CoT"  # 有示例数据
    else:
        return "Zero-shot CoT"  # 快速验证
```

**代码示例：**

```python
# Zero-shot CoT
prompt_zero_shot = f"""
问题：{query}

请一步步思考，然后给出答案。
"""

# Few-shot CoT
prompt_few_shot = f"""
示例1:
问题：小明有5个苹果，他吃了2个，又买了3个，他现在有几个？
思考：原有5个 → 吃了2个剩3个 → 买了3个变成6个
答案：6个

示例2:
问题：{query}
思考：
"""
```

### 面试题2：在构建生产级LLM应用时，如何确保模型输出严格符合预期的JSON格式？请列举至少三种方法，并说明各自的优缺点。

**答案：**

**方法1：JSON Mode（OpenAI）**

```python
# 实现
llm = ChatOpenAI(
    model="gpt-4o",
    response_format={"type": "json_object"}
)

# 优点：
# 1. 模型原生支持，稳定性高
# 2. 无需额外token（不占用输入token）
# 3. 速度快

# 缺点：
# 1. 仅OpenAI模型支持
# 2. 无法约束具体字段结构（只保证是合法JSON）
# 3. 需要配合详细的Prompt说明字段
```

**方法2：Function Calling（所有支持Function Calling的模型）**

```python
# 实现
function_schema = {
    "name": "extract_info",
    "parameters": {
        "type": "object",
        "properties": {
            "field1": {"type": "string"},
            "field2": {"type": "number"}
        },
        "required": ["field1"]
    }
}

llm_with_functions = llm.bind(
    functions=[function_schema],
    function_call={"name": "extract_info"}
)

# 优点：
# 1. 强制输出结构（100%符合schema）
# 2. 支持复杂嵌套结构
# 3. 多模型支持（OpenAI/Anthropic/Google）
# 4. 自动验证必填字段

# 缺点：
# 1. 占用function token（影响上下文长度）
# 2. 某些模型支持不完善
```

**方法3：Output Parser + 重试机制（通用方案）**

```python
# 实现
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser(pydantic_object=MySchema)

chain = (
    prompt 
    | llm 
    | parser  # 自动重试直到解析成功
)

# 优点：
# 1. 模型无关（适用于所有LLM）
# 2. 自动重试机制
# 3. 支持Pydantic模型验证
# 4. 错误信息友好

# 缺点：
# 1. 重试消耗token和时间
# 2. 依赖Prompt质量（模型可能不遵循格式）
# 3. 解析失败时会多次调用LLM
```

**方法4：Guardrails/Instructor（高级约束）**

```python
# 使用Instructor（推荐）
import instructor
from pydantic import BaseModel

client = instructor.from_openai(OpenAI())

class Output(BaseModel):
    answer: str
    confidence: float

# 自动验证+重试
result = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": query}],
    response_model=Output
)

# 优点：
# 1. 自动重试 + 错误修正
# 2. 支持复杂验证规则
# 3. 类型安全

# 缺点：
# 1. 需要额外依赖
# 2. 学习成本
```

**生产环境推荐方案：**

```python
def production_structured_output(query: str) -> dict:
    """
    生产级结构化输出方案
    
    策略：多层保障
    1. 优先使用Function Calling（最稳定）
    2. 失败时使用JSON Mode + 解析重试
    3. 最终降级为手动解析
    """
    
    try:
        # 第1层：Function Calling
        result = function_calling_method(query)
        return result
    except Exception as e:
        print(f"Function Calling失败: {e}")
        
        try:
            # 第2层：JSON Mode + 健壮解析
            result = json_mode_method(query)
            return robust_json_parse(result)
        except Exception as e2:
            print(f"JSON Mode失败: {e2}")
            
            # 第3层：降级处理
            return {"error": "解析失败", "raw_output": str(e2)}
```

## 总结

本文系统讲解了Prompt Engineering中的两个核心技术：

1. **思维链（CoT）**：通过引导模型逐步思考，显著提升复杂推理任务的准确率
   - Zero-shot CoT：快速原型、通用任务
   - Few-shot CoT：专业领域、复杂推理

2. **结构化输出**：确保模型输出可控、可解析
   - JSON Mode：简单场景、OpenAI模型
   - Function Calling：生产环境、复杂结构
   - Output Parser：通用方案、自动重试

**实践建议：**
- 在开发阶段使用Few-shot CoT快速验证效果
- 生产环境优先使用Function Calling保证输出稳定性
- 始终添加解析失败的重试和降级机制

## 参考资料

- [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)
- [OpenAI Guide: Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [LangChain Output Parsers](https://python.langchain.com/docs/modules/model_io/output_parsers/)
- [Instructor Documentation](https://github.com/jxnl/instructor)

---

*如果觉得这篇文章对你有帮助，欢迎点赞、收藏和转发！有任何问题也欢迎在评论区留言讨论。*
