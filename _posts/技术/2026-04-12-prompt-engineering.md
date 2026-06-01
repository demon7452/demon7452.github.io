---
layout: post
title: "Prompt Engineering：System Prompt设计原则与模板工程"
date: 2026-04-12
categories: [AI Agent, Prompt Engineering, 应用开发]
tags: [System Prompt, 模板工程, Prompt设计, Jinja2, Python]
---

System Prompt 是 AI Agent 的"宪法"。它定义了 Agent 的角色边界、行为准则、输出格式和工具使用策略。一个精心设计的 System Prompt 可以让同一个模型的能力表现出数量级的差异，而一个糟糕的 System Prompt 则会让 Agent 变得不可预测、啰嗦甚至危险。

## System Prompt 的核心设计原则

### 1. 分层结构化设计

将 System Prompt 按职责分层，每层独立、可替换：

```
[角色定义层]    → 身份、性格、风格
[核心规则层]    → 安全约束、执行策略
[工具使用层]    → 工具描述、路由规则
[输出规范层]    → 格式、语言、卡片协议
[领域知识层]    → 行业术语、业务规则
```

### 2. 正向指令优于负向禁令

与其说"不要做X"，不如说"遇到X时，应该做Y"。负向指令容易被模型忽略，正向指令给出明确路径。

```python
# ❌ 纯负向禁令，效果差
BAD_RULE = "不要胡编乱造、不要输出无关内容、不要使用表情符号"

# ✅ 正向指令 + 替代行为，效果好
GOOD_RULE = (
    "结论必须有数据或原文支撑。"
    "回答需直击痛点、条理清晰。"
    "除用户明确要求，禁止使用任何表情符号。"
)
```

### 3. 示例驱动（Few-Shot in Prompt）

对于格式化输出、工具调用等场景，在 Prompt 中嵌入示例比纯文字描述更有效。

```markdown
## 卡片输出格式
当展示文件列表时，**必须**使用以下格式：

```yyb-file-list
[文件A](<D:\路径\文件A.pdf>)
[文件B](<D:\路径\文件B.docx>)
```
```

### 4. 冲突消解优先级

当多层规则冲突时，必须有明确的优先级声明：

```markdown
## 规则优先级
1. 安全约束（最高优先级，覆盖所有其他规则）
2. 用户明确指令
3. 工具使用协议
4. 输出规范
```

## 模板工程：用 Jinja2 管理 Prompt

在实际项目中，System Prompt 不应该是一坨硬编码的字符串。引入模板引擎让 Prompt 可维护、可版本化、可动态组装。

### 基础模板结构

```
prompts/
├── base/
│   ├── role.j2           # 角色定义
│   ├── safety.j2         # 安全约束
│   └── output.j2         # 输出规范
├── domains/
│   ├── finance.j2        # 金融领域
│   └── medical.j2        # 医疗领域
├── tools/
│   ├── search.j2         # 搜索工具
│   └── file_ops.j2       # 文件操作工具
└── agents/
    ├── file_agent.j2     # 文件助手完整 Prompt
    └── code_agent.j2     # 代码助手完整 Prompt
```

### Python 实现

```python
import os
from jinja2 import Environment, FileSystemLoader, select_autoescape

class PromptBuilder:
    """Build System Prompt from modular Jinja2 templates."""

    def __init__(self, template_dir: str = "prompts"):
        self.env = Environment(
            loader=FileSystemLoader(template_dir),
            autoescape=select_autoescape(),
            trim_blocks=True,
            lstrip_blocks=True
        )

    def build(self, agent_type: str, **kwargs) -> str:
        """Assemble System Prompt by rendering the agent template."""
        template = self.env.get_template(f"agents/{agent_type}.j2")

        prompt = template.render(
            date=kwargs.get("date", "2026-04-12"),
            domain=kwargs.get("domain", "general"),
            tools=kwargs.get("tools", []),
            max_tokens=kwargs.get("max_tokens", 4096),
            workspace=kwargs.get("workspace", os.getcwd())
        )

        # 清理多余空行和缩进
        return self._sanitize(prompt)

    def _sanitize(self, text: str) -> str:
        """Remove excessive blank lines while preserving structure."""
        lines = text.split("\n")
        cleaned = []
        blank_count = 0
        for line in lines:
            if line.strip() == "":
                blank_count += 1
                if blank_count <= 1:
                    cleaned.append(line)
            else:
                blank_count = 0
                cleaned.append(line)
        return "\n".join(cleaned).strip()
```

### file_agent.j2 模板示例

```jinja2
{# agents/file_agent.j2 #}
{% include 'base/role.j2' %}

{% include 'base/safety.j2' %}

## 环境信息
- 当前日期: {{ date }}
- 工作目录: {{ workspace }}

{% if tools %}
## 可用工具
{% for tool in tools %}
- **{{ tool.name }}**: {{ tool.description }}
{% endfor %}
{% endif %}

{% include 'base/output.j2' %}

{% if domain != 'general' %}
{% include 'domains/' + domain + '.j2' %}
{% endif %}
```

### 使用示例

```python
builder = PromptBuilder(template_dir="./prompts")

# 构建文件助手的 System Prompt
file_agent_prompt = builder.build(
    agent_type="file_agent",
    date="2026-04-12",
    domain="finance",
    tools=[
        {"name": "read_file", "description": "读取复杂文件内容"},
        {"name": "search_file", "description": "语义/结构化文件搜索"},
        {"name": "write_file", "description": "写入文件内容"},
    ]
)

print(f"Prompt 长度: {len(file_agent_prompt)} 字符")
# 可以继续发送给 LLM API
```

## Prompt 版本管理与 A/B 测试

将 Prompt 模板纳入 Git 管理，配合 CI/CD 做自动化测试：

```python
import hashlib

class PromptRegistry:
    """Track prompt versions and enable A/B testing."""

    def __init__(self):
        self.versions = {}

    def register(self, name: str, content: str):
        version_hash = hashlib.sha256(content.encode()).hexdigest()[:8]
        self.versions[name] = {
            "content": content,
            "version": version_hash,
            "length": len(content)
        }
        return version_hash

    def get_for_ab_test(self, name_variants: list[str]):
        """Select a prompt variant for A/B testing."""
        import random
        return random.choice(name_variants)
```

## 常见陷阱与最佳实践

| 陷阱 | 表现 | 解决 |
|------|------|------|
| Prompt 过长 | 模型忽略后半部分内容 | 分层 + 动态注入，只加载当前任务相关规则 |
| 指令冲突 | 两个规则互相矛盾，模型行为不稳定 | 显式声明优先级，用「覆盖」「例外」等词标记 |
| 过度约束 | 模型变得过于保守，拒绝合理请求 | 用分级策略替代一刀切禁令 |
| 语言不一致 | System Prompt 英文，用户中文，输出混杂 | 添加语言一致性协议，要求模型以用户语言回复 |
| 示例偏差 | Few-shot 示例过于具体，模型机械模仿 | 示例要覆盖常见情况，并注明「仅作参考」 |

## 面试题

**1. 在设计 System Prompt 时，为什么推荐「正向指令」而非「负向禁令」？请结合大模型的行为特点解释。**

<details>
<summary>点击展开答案</summary>

大模型的训练目标是预测下一个 token，它对「应该做什么」的建模远强于「不应该做什么」。负向禁令（如「不要胡编」）存在两个问题：

1. **注意力稀释**：禁令本身可能被模型当作要处理的指令而非约束，尤其在长 Prompt 中容易被后文覆盖或遗忘。

2. **缺乏替代路径**：只告诉模型不能做什么，模型可能选择次优路径甚至拒绝响应。正向指令（如「结论必须有数据支撑」）给出了明确的替代行为，模型更容易遵循。

实战中的做法：把每条负向禁令改写为「遇到X场景→执行Y行为」的正向映射，同时对必须保留的禁令加上「最高优先级」标记。

</details>

**2. Jinja2 模板引擎在 Prompt 工程中的核心价值是什么？你如何设计一套可扩展的 Prompt 模板架构？**

<details>
<summary>点击展开答案</summary>

核心价值：

- **模块化**：将角色、安全规则、工具描述等拆分为独立模板文件，修改一处自动传播到所有 Agent。
- **动态组装**：根据运行时上下文（当前日期、用户权限、可用工具集）动态注入不同的 Prompt 片段，避免一个超长 Prompt 包含所有可能场景。
- **版本可控**：模板文件纳入 Git 管理，每次修改有迹可循，支持回滚和 diff 审查。

可扩展架构设计要点：

1. 按职责分目录（base/domains/tools/agents），每个目录下的 `.j2` 文件职责单一。
2. Agent 级别的模板通过 `{% include %}` 和 `{% if %}` 组合底层模块，支持按参数选择领域知识、工具集。
3. 用 `PromptRegistry` 记录每次构建的版本哈希，方便线上问题排查时回溯当时使用的 Prompt 版本。

</details>