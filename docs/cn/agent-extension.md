# ValueCell Agent 开发与扩展

## 学习目标

阅读本文后，你将能够：

- 说明一个可工作的 Agent 需要哪些组成部分
- 独立实现一个最小可用 Agent 并在系统中运行
- 正确配置 Agent Card 和 Agent YAML
- 使用 `streaming.*` 辅助函数发出各类事件
- 在 Agent 中集成 LLM 和工具（Tool）
- 排查 Agent 接入过程中的常见问题

---

## 1. 先理解：新增 Agent 不只是写一个类

在 ValueCell 中，一个可工作的 Agent 至少涉及**三个组成部分**：

| 组成部分 | 位置 | 职责 |
|----------|------|------|
| **Python Agent 实现** | `python/valuecell/agents/<agent_name>/` | 核心业务逻辑、流式响应 |
| **Agent Card** | `python/configs/agent_cards/<name>.json` | 服务发现、能力声明、前端展示 |
| **Agent YAML 配置** | `python/configs/agents/<name>.yaml` | 模型选择、参数覆盖、环境变量映射 |

三者的关系是：**代码实现提供能力，Agent Card 声明能力，YAML 配置运行参数。** 缺少任何一个，Agent 都无法正常工作。

> **为什么需要三份文件？** 这种设计将"能力是什么"（Card）、"用什么模型跑"（YAML）、"怎么跑"（代码）三者解耦。你可以不改代码就切换模型，也可以不改配置就替换实现。

---

## 2. 最小接入路径

下面以创建一个 `HelloAgent` 为例，展示完整的接入过程。

### 2.1 创建 Agent 目录

```bash
mkdir -p python/valuecell/agents/hello_agent
touch python/valuecell/agents/hello_agent/__init__.py
touch python/valuecell/agents/hello_agent/__main__.py
touch python/valuecell/agents/hello_agent/core.py
```

典型目录结构：

```
python/valuecell/agents/hello_agent/
├── __init__.py          # 包初始化
├── __main__.py          # 独立运行入口
└── core.py              # Agent 核心逻辑
```

### 2.2 实现 Agent 逻辑

在 `core.py` 中，继承 `BaseAgent` 并实现 `stream()` 方法：

```python
# file: python/valuecell/agents/hello_agent/core.py
from typing import AsyncGenerator, Optional, Dict
from valuecell.core.types import BaseAgent, StreamResponse
from valuecell.core.agent import streaming


class HelloAgent(BaseAgent):
    async def stream(
        self,
        query: str,                    # 用户查询内容
        conversation_id: str,          # 会话 ID
        task_id: str,                  # 任务 ID
        dependencies: Optional[Dict] = None,  # 可选上下文（语言、时区等）
    ) -> AsyncGenerator[StreamResponse, None]:
        """
        处理用户查询并返回流式响应。

        核心约定：
        - 方法必须是 async 的
        - 使用 yield 逐条返回事件
        - 最后必须 yield streaming.done() 表示完成
        """
        # 发送消息片段
        yield streaming.message_chunk("思考中...")
        yield streaming.message_chunk(f"你说的是：{query}")

        # 必须以 done 结束
        yield streaming.done()
```

**关于 `stream()` 方法的几个关键点：**

- **异步**：必须是 `async def`，因为内部可能涉及网络调用
- **流式**：返回 `AsyncGenerator`，用 `yield` 逐条产出事件
- **参数固定**：四个参数的含义由系统契约决定，不要随意增删
- **必须调用 `done()`**：缺少 `done()` 会导致前端无法判断响应是否结束

> **为什么不写成同步函数直接 return？** 因为 ValueCell 的架构主线就是流式的。后端的 Orchestrator、Event 层、前端渲染器都围绕流式事件设计。如果 Agent 返回一次性结果，就脱离了这条主线，前端也无法正确展示过程。

### 2.3 创建独立运行入口

在 `__main__.py` 中，使用 `create_wrapped_agent()` 包装并启动：

```python
# file: python/valuecell/agents/hello_agent/__main__.py
import asyncio
from valuecell.core.agent import create_wrapped_agent
from .core import HelloAgent


if __name__ == "__main__":
    agent = create_wrapped_agent(HelloAgent)
    asyncio.run(agent.serve())
```

`create_wrapped_agent()` 做了什么：

- 将你的 Agent 类包装为标准的 A2A 服务
- 根据对应的 Agent Card 读取监听地址和端口
- 统一处理传输协议和事件序列化

> **重要**：`create_wrapped_agent()` 必须放在 `__main__.py` 中，因为这种模式使得：
> - 可以通过 `uv run -m valuecell.agents.hello_agent` 独立启动
> - 后端服务器能自动发现并注册该 Agent
> - 传输协议和事件发射方式保持一致

### 2.4 配置 Agent Card

在 `python/configs/agent_cards/` 下创建 `hello_agent.json`：

```json
{
  "name": "HelloAgent",
  "display_name": "Hello Agent",
  "url": "http://localhost:10010",
  "description": "一个最小的示例 Agent，回显用户输入。",
  "capabilities": {
    "streaming": true,
    "push_notifications": false
  },
  "default_input_modes": ["text"],
  "default_output_modes": ["text"],
  "version": "1.0.0",
  "skills": [
    {
      "id": "echo",
      "name": "Echo",
      "description": "回显用户输入为流式片段。",
      "tags": ["example", "echo"]
    }
  ]
}
```

### 2.5 配置 Agent YAML

在 `python/configs/agents/` 下创建 `hello_agent.yaml`：

```yaml
name: "Hello Agent"
enabled: true

models:
  primary:
    model_id: "anthropic/claude-haiku-4.5"
    provider: "openrouter"

env_overrides:
  HELLO_AGENT_MODEL_ID: "models.primary.model_id"
  HELLO_AGENT_PROVIDER: "models.primary.provider"
```

> **命名约定**：YAML 文件名（不含 `.yaml` 后缀）建议与 Agent 目录名一致。`get_model_for_agent("hello_agent")` 会自动加载 `hello_agent.yaml` 的配置。

---

## 3. Agent Card 字段详解

Agent Card 不只是展示配置，它会影响 Agent 的**发现、连接和能力描述**。

### 3.1 必填字段

| 字段 | 说明 | 注意事项 |
|------|------|----------|
| `name` | Agent 类名 | **必须与 Python 类名完全一致**（如 `HelloAgent`） |
| `url` | 服务监听地址 | 包含端口，多个 Agent 端口不能冲突 |
| `description` | Agent 功能描述 | 会影响规划器（Planner）如何选择 Agent |

### 3.2 重要字段

| 字段 | 说明 |
|------|------|
| `display_name` | 前端展示名（可以包含中文、空格） |
| `capabilities.streaming` | 是否支持流式输出 |
| `capabilities.push_notifications` | 是否支持推送通知 |
| `skills` | 能力描述数组，包含 `id`、`name`、`description`、`examples`、`tags` |
| `enabled` | 是否启用（`false` 可禁用而不删除配置） |
| `metadata.local_agent_class` | 本地类路径（如 `valuecell.agents.hello_agent.core:HelloAgent`） |

### 3.3 当前已注册 Agent 的端口分配

| Agent | Card 文件名 | `name` 字段 | 默认端口 |
|-------|-------------|-------------|----------|
| 研究 Agent | `investment_research_agent.json` | `ResearchAgent` | 10004 |
| 新闻 Agent | `news_agent.json` | `NewsAgent` | 10005 |
| 提示策略 Agent | `prompt_strategy_agent.json` | `PromptBasedStrategyAgent` | 10006 |
| 网格策略 Agent | `grid_agent.json` | `GridStrategyAgent` | 10007 |

新增 Agent 时，请选择未被占用的端口。

---

## 4. 事件系统（Event System）

### 4.1 为什么事件系统很重要

ValueCell 的 Agent 不是"返回一个字符串就结束"，而是通过**结构化事件流**与前端通信。事件类型决定了前端用什么渲染器展示内容。

### 4.2 可用的事件类型

所有事件类型定义在 `valuecell.core.types` 中，通过 `streaming.*` 辅助函数产出：

| 事件类型 | 辅助函数 | 说明 |
|----------|----------|------|
| `MESSAGE_CHUNK` | `streaming.message_chunk(text)` | 文本消息片段 |
| `TOOL_CALL_STARTED` | `streaming.tool_call_started(id, name)` | 工具调用开始 |
| `TOOL_CALL_COMPLETED` | `streaming.tool_call_completed(result, id, name)` | 工具调用完成 |
| `COMPONENT_GENERATOR` | `streaming.component_generator(type, content)` | 富 UI 组件 |
| `DONE` | `streaming.done()` | 流式输出结束（成功） |
| `FAILED` | `streaming.failed(content)` | 流式输出结束（失败） |

此外，`notification` 辅助函数用于推送通知场景：

| 辅助函数 | 说明 |
|----------|------|
| `notification.message(content)` | 推送文本通知 |
| `notification.component_generator(type, content)` | 推送组件通知 |
| `notification.done()` | 推送通知完成 |
| `notification.failed(content)` | 推送通知失败 |

### 4.3 富 UI 组件事件

`COMPONENT_GENERATOR` 事件允许 Agent 发送纯文本以外的 UI 组件。当前支持的组件类型：

| 组件类型 | 说明 |
|----------|------|
| `report` | 研究报告，支持 Markdown 格式内容 |
| `profile` | 公司或股票档案 |
| `filtered_line_chart` | 可交互的折线图 |
| `filtered_card_push_notification` | 带筛选选项的通知卡片 |
| `scheduled_task_controller` | 定时任务控制面板 |
| `scheduled_task_result` | 定时任务结果展示 |

**示例：发送折线图组件**

```python
yield streaming.component_generator(
    component_type="filtered_line_chart",
    content={
        "title": "股价走势",
        "data": [
            ["日期", "AAPL", "GOOGL"],
            ["2025-01-01", 150.5, 2800.3],
            ["2025-01-02", 152.1, 2815.7],
        ],
        "create_time": "2025-01-15 10:30:00",
    },
)
```

**示例：发送报告组件**

```python
yield streaming.component_generator(
    component_type="report",
    content={
        "title": "Q4 财务分析",
        "data": "## 摘要\n\n营收同比增长 15%...",
        "create_time": "2025-01-15 10:30:00",
    },
)
```

---

## 5. 在 Agent 中使用 LLM 和工具

### 5.1 加载配置的模型

使用 `get_model_for_agent()` 加载与 Agent YAML 配置匹配的模型：

```python
from valuecell.utils.model import get_model_for_agent


class MyAgent(BaseAgent):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        # 自动加载 my_agent.yaml 中配置的模型
        # "my_agent" 必须与 YAML 文件名一致
        self.model = get_model_for_agent("my_agent")
```

### 5.2 使用 Agno Agent 框架集成 LLM 和工具

ValueCell 使用 [Agno](https://docs.agno.com)（v2.x）作为 Agent 构建框架。Agno 提供了 `Agent` 类来封装 LLM 调用、工具注册和知识库集成。

**Agno 在 ValueCell 中的角色**：

- ValueCell 的 `BaseAgent` 定义了与编排层通信的接口（`stream()` 方法）
- Agno 的 `Agent` 封装了与 LLM 交互的细节（模型调用、工具执行、知识检索）
- 每个 ValueCell Agent 通常在内部创建一个 `agno.Agent` 实例

以下是一个完整示例：

```python
from agno.agent import Agent
from valuecell.core.types import BaseAgent
from valuecell.core.agent import streaming
from valuecell.utils.model import get_model_for_agent


def search_stock_info(ticker: str) -> str:
    """
    根据股票代码查询信息。

    Args:
        ticker: 股票代码，如 "AAPL"、"GOOGL"

    Returns:
        股票信息字符串
    """
    return f"Stock info for {ticker}"


class ResearchAgent(BaseAgent):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.inner = Agent(
            model=get_model_for_agent("research_agent"),
            tools=[search_stock_info],  # 注册工具函数
        )

    async def stream(self, query, conversation_id, task_id, dependencies=None):
        # 启动 Agno Agent 的流式推理
        response_stream = self.inner.arun(
            query,
            stream=True,
            stream_intermediate_steps=True,
            session_id=conversation_id,
        )

        async for event in response_stream:
            if event.event == "RunContent":
                # 文本内容片段
                yield streaming.message_chunk(event.content)

            elif event.event == "ToolCallStarted":
                # 工具调用开始
                yield streaming.tool_call_started(
                    event.tool.tool_call_id,
                    event.tool.tool_name,
                )

            elif event.event == "ToolCallCompleted":
                # 工具调用完成
                yield streaming.tool_call_completed(
                    event.tool.result,
                    event.tool.tool_call_id,
                    event.tool.tool_name,
                )

        # 必须以 done 结束
        yield streaming.done()
```

> **Agno 事件到 ValueCell 事件的映射**：Agno 的 `arun()` 方法发出 `RunContent`、`ToolCallStarted`、`ToolCallCompleted` 等内部事件。你的 `stream()` 方法负责把这些内部事件转换为 ValueCell 的 `streaming.*` 格式。这种分层设计意味着你可以不使用 Agno（比如用裸 HTTP 调用 LLM），只要你的 `stream()` 方法产出的仍然是 ValueCell 事件契约。

- **清晰的文档字符串**：Agno 会将 `docstring` 作为工具描述传给 LLM，它决定了 LLM 是否能正确选择和调用工具
- **完整的类型注解**：所有参数和返回值都要有类型注解
- **单一职责**：每个工具只做一件事
- **健壮的错误处理**：工具内部的异常不应导致 Agent 崩溃

### 5.4 运行时配置覆盖

可以通过环境变量临时覆盖 Agent 的模型配置，无需修改任何文件：

```bash
export HELLO_AGENT_MODEL_ID="anthropic/claude-3.5-sonnet"
export HELLO_AGENT_TEMPERATURE="0.9"

cd python
uv run -m valuecell.agents.hello_agent
```

---

## 6. 调试新 Agent 的推荐步骤

### 第一步：单独运行 Agent

```bash
cd python
uv run -m valuecell.agents.hello_agent
```

先确认它能独立启动、不报错。

### 第二步：启动后端，验证发现

```bash
bash start.sh --no-frontend
```

观察后端日志，确认系统能发现并连接你的 Agent。

### 第三步：从前端验证完整体验

启动完整系统，在 Web UI 中验证：

- Agent 是否出现在界面中
- 发起对话后是否收到流式内容
- 事件渲染（工具调用、推理等）是否符合预期

### 使用调试模式

在 `.env` 中设置：

```bash
AGENT_DEBUG_MODE=true
```

调试模式会记录：

- 发送给模型的完整 Prompt
- 工具调用详情
- 中间步骤
- 提供商返回的元数据

> **注意**：调试模式会记录敏感的输入/输出内容，并显著增加日志量和延迟。仅在本地开发环境使用，不要在生产环境开启。

---

## 7. 常见接入错误

### 7.1 只有代码，没有 Agent Card

**现象**：系统无法发现或展示该 Agent。

**原因**：Agent Card 是系统发现 Agent 的关键契约。缺少 Card 等于 Agent 不存在。

**解决**：在 `python/configs/agent_cards/` 下创建对应的 JSON 文件。

### 7.2 类名与 Card `name` 不一致

**现象**：包装或发现链路失败。

**原因**：`create_wrapped_agent()` 依赖 Card 中的 `name` 字段来匹配类。

**解决**：确保 Card 中的 `"name": "HelloAgent"` 与 `class HelloAgent(BaseAgent)` 完全一致。

### 7.3 端口冲突

**现象**：Agent 服务启动失败或被错误复用。

**解决**：检查 Card 中的 `url` 字段，确保端口未被其他 Agent 占用（参考第 3.3 节的端口分配表）。

### 7.4 不返回流式事件

**现象**：前端体验差，消息不完整。

**原因**：直接返回字符串而非使用 `streaming.*` 辅助函数。

**解决**：始终使用 `streaming.message_chunk()` 发送内容，并以 `streaming.done()` 结束。

### 7.5 忘记创建 Agent YAML 配置

**现象**：Agent 能启动但无法正确使用模型。

**原因**：缺少 YAML 配置导致 `get_model_for_agent()` 找不到配置。

**解决**：在 `python/configs/agents/` 下创建对应的 YAML 文件。

---

## 8. 扩展现有 Agent

如果你不是新增 Agent，而是扩展现有 Agent，建议先明确三个问题：

1. **改的是什么？** 是扩展能力描述（Agent Card 的 `skills`），还是扩展内部执行逻辑（`core.py`）？
2. **是否改变了事件结构？** 如果你新增了组件类型或改变了事件序列，前端渲染器是否需要同步更新？
3. **配置是否需要同步修改？** 新增工具或更改模型参数时，是否需要更新 Agent YAML？

### 当前 Agent 的关键设计模式

注意 `prompt_strategy_agent.json` 和 `grid_agent.json` 中的 `metadata.planner_passthrough` 字段：

```json
{
  "metadata": {
    "planner_passthrough": true
  }
}
```

**`planner_passthrough` 的含义**：当设为 `true` 时，该 Agent 会跳过完整的 Planner 推理流程，直接将用户请求原样交给 Agent 执行。这适用于交互简单、不需要复杂规划的 Agent。

ValueCell 当前至少存在两种执行模式：

- **标准规划模式**：先由 Planner 分析用户意图、生成执行计划，再执行
- **直通模式**（passthrough）：跳过规划，直接生成单任务计划并执行

理解这一点对调试很重要：某些 Agent 的请求可能根本不会进入完整的 Planner 推理流程。

---

## 9. 一个务实的最小闭环

如果你要创建第一个自定义 Agent，建议的目标不是"一步做到很复杂"，而是先完成这个闭环：

1. 新建 Agent 目录（`__init__.py`、`__main__.py`、`core.py`）
2. 实现最简单的 `stream()` —— 只返回一条消息和一个 `done()`
3. 创建 Agent Card（JSON 文件）
4. 创建 Agent YAML 配置
5. 单独启动验证
6. 接入后端验证
7. 从前端看到可用响应

先跑通闭环，再逐步加入工具调用、推理输出、富 UI 组件和定时任务等高级功能。

---

## 10. 推荐阅读

- 英文原文：[`docs/CONTRIBUTE_AN_AGENT.md`](../CONTRIBUTE_AN_AGENT.md)
- 系统架构：[architecture.md](./architecture.md)
- 源码导读：[source-guide.md](./source-guide.md)
- 配置详解：[configuration.md](./configuration.md)
