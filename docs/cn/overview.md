# ValueCell 项目概览

## 学习目标

阅读本文后，你将能够：

- 准确描述 ValueCell 的产品定位和核心价值
- 区分"已实现"和"规划中"的能力
- 解释系统各层的职责和协作方式
- 理解智能体（Agent）、编排器（Orchestrator）、规划器（Planner）等核心概念
- 避免新手常见的认知误区

---

## 1. 项目是什么

ValueCell 是一个**社区驱动的、面向金融场景的多智能体（Multi-Agent）平台**。它的目标不是做一个聊天机器人，而是构建一个由多个专业 AI 智能体（Agent）协同工作的金融助手团队。

从仓库当前实现来看，ValueCell 由以下部分组成：

| 层级 | 职责 | 技术 |
|------|------|------|
| **前端层** | 用户界面、流式渲染（Streaming Response）、状态管理 | React Router v7、TypeScript、Vite、**Tauri v2**（桌面打包） |
| **API 层** | 服务入口、路由、初始化 | FastAPI、uvicorn |
| **运行时层** | 编排、规划、执行、事件处理 | Python asyncio |
| **Agent 层** | 具体金融能力实现 | **Agno** 框架（智能体（Agent）构建）、**A2A** 协议（智能体（Agent）通信） |

### 1.1 核心目标

1. **统一入口**：用户通过一个界面与不同类型的智能体（Agent）交互
2. **Agent 协作**：支持研究、新闻、策略等不同能力的智能体（Agent）协同工作
3. **数据本地化**：配置、数据库、知识库等敏感数据保存在本地系统目录
4. **实时反馈**：通过流式响应（Streaming Response）持续展示推理过程、工具调用和任务结果
5. **桌面应用**：通过 Tauri v2 打包为 macOS / Windows 原生应用，也可通过浏览器访问

### 1.2 在线版本

ValueCell 已推出在线服务，提供 A 股深度研究和市场分析功能，无需本地部署，直接访问 [valuecell.ai](https://valuecell.ai) 即可使用。

---

## 2. 它不是什么

为了避免误解，明确以下几点：

- 它**不是**单一模型提供商的固定应用——仓库支持 **9 个** LLM 提供商接入
- 它**不是**只做前端展示——核心价值在于后端的编排、规划、任务执行和 Agent 协议集成
- 它**不是**纯命令式任务脚本——带有规划、流式事件、对话持久化和人机确认（Human-in-the-Loop, HITL）机制
- 它**不是**把所有状态写在仓库根目录——使用系统级应用目录保存 `.env`、SQLite、LanceDB 和知识数据

---

## 3. 当前已实现的 Agent

根据仓库中 `python/configs/agent_cards/` 下的 Agent Card 配置，当前有以下 4 个 Agent：

### 3.1 ResearchAgent（研究 Agent）

- **Agent Card**：`investment_research_agent.json`
- **代码位置**：`python/valuecell/agents/research_agent/`
- **监听端口**：10004
- **默认模型**：`google/gemini-2.5-flash`（通过 OpenRouter）
- **嵌入模型**：`Qwen/Qwen3-Embedding-4B`（通过 SiliconFlow）
- **执行模式**：标准规划（无 `planner_passthrough`）
- **前端可见**：是（无 `metadata.hidden`）
- **推送通知**：不支持（Agent Card 中无 `capabilities` 字段）

Card 中声明的 Skills（`skills` 数组）：

| Skill ID | 名称 | 说明 |
|----------|------|------|
| `extract_financials` | Extract financial line items | 从 10-K、10-Q 等财报中提取营收、EPS、现金流等指标 |
| `analyze_13f_filings` | Analyze 13F holdings | 解析机构持仓、仓位变动 |
| `synthesize_context` | Synthesize context and commentary | 结合知识库搜索结果提供解读和上下文 |
| `monitor_and_alert` | Monitor filings and notify | 跟踪公司文件提交并发送摘要 |

> **注意**：Research Agent 访问 SEC EDGAR 数据需要配置 `SEC_EMAIL` 环境变量（用于 SEC 的 User-Agent 标识）。如果未配置，Agent 会输出警告但仍可运行（部分功能受限）。

### 3.2 NewsAgent（新闻 Agent）

- **Agent Card**：`news_agent.json`
- **代码位置**：`python/valuecell/agents/news_agent/`
- **监听端口**：10005
- **默认模型**：`qwen/qwen3-max`（通过 OpenRouter）
- **搜索模型**：`perplexity/sonar`（通过 OpenRouter，用于网络搜索）
- **执行模式**：标准规划
- **前端可见**：是
- **推送通知**：不支持（`capabilities` 中仅有 `streaming: true`）

Card 中声明的 Skills：

| Skill ID | 名称 | 说明 |
|----------|------|------|
| `web_search` | Real-time news search | 通过网络搜索获取当前新闻 |
| `get_breaking_news` | Breaking news monitoring | 监控并获取紧急新闻更新 |
| `get_financial_news` | Financial market news | 综合性金融市场和商业新闻 |

### 3.3 PromptBasedStrategyAgent（提示策略 Agent）

- **Agent Card**：`prompt_strategy_agent.json`
- **代码位置**：`python/valuecell/agents/prompt_strategy_agent/`
- **监听端口**：10006
- **执行模式**：直通模式（`metadata.planner_passthrough: true`）
- **前端可见**：否（`metadata.hidden: true`）
- **推送通知**：支持（`capabilities.push_notifications: true`）

Card 中声明的 Skills：

| Skill ID | 名称 | 说明 |
|----------|------|------|
| `strategy_run` | Run Strategy | 根据自然语言描述运行交易策略 |

> **注意**：此 Agent 没有 Agent YAML 配置文件，使用系统默认模型。

### 3.4 GridStrategyAgent（网格策略 Agent）

- **Agent Card**：`grid_agent.json`
- **代码位置**：`python/valuecell/agents/grid_agent/grid_agent.py`
- **监听端口**：10007
- **执行模式**：直通模式（`metadata.planner_passthrough: true`）
- **前端可见**：否（`metadata.hidden: true`）
- **推送通知**：支持（`capabilities.push_notifications: true`）
- **Skills**：空数组 `[]`（不声明特定能力，由 Planner 直通处理）

> **注意**：此 Agent 也没有 Agent YAML 配置文件，使用系统默认模型。策略类 Agent 涉及实盘交易，当前状态因交易所而异（见第 5 节），请谨慎使用。

---

## 4. 模型提供商支持

ValueCell 通过统一的配置接口支持多个 LLM 提供商，当前已配置 **9 个**提供商：

| 提供商 | 配置文件 | 默认模型 | 默认嵌入模型 | 特点 |
|--------|----------|----------|-------------|------|
| **OpenRouter** | `providers/openrouter.yaml` | `qwen/qwen3-max` | `qwen/qwen3-embedding-4b` | 聚合多模型，模型最全 |
| **SiliconFlow** | `providers/siliconflow.yaml` | `deepseek-ai/DeepSeek-V3.1-Terminus` | `Qwen/Qwen3-Embedding-4B` | 性价比高，支持中文模型 |
| **Google** | `providers/google.yaml` | `gemini-2.5-flash` | `gemini-embedding-001` | Gemini 系列 |
| **OpenAI** | `providers/openai.yaml` | `gpt-5-2025-08-07` | `text-embedding-3-small` | GPT 系列 |
| **Azure** | `providers/azure.yaml` | `gpt-5-2025-08-07` | `text-embedding-3-small` | 企业级部署 |
| **DeepSeek** | `providers/deepseek.yaml` | `deepseek-chat` | — | DeepSeek 模型 |
| **DashScope** | `providers/dashscope.yaml` | `qwen3-max` | `text-embedding-v4` | 阿里云百炼（Qwen 系列） |
| **Ollama** | `providers/ollama.yaml` | `qwen3:4b` | — | 本地模型部署 |
| **OpenAI-Compatible** | `providers/openai-compatible.yaml` | `qwen3-max` | `text-embedding-v4` | 兼容接口的自定义提供商 |

> **注意**：上表中的默认模型是 YAML 配置文件中的初始值，实际使用时可通过 `.env` 或环境变量覆盖。Agent 级别的模型配置（如 Research Agent 默认使用 `google/gemini-2.5-flash`）会覆盖提供商级别的默认值。

系统支持**自动检测**（根据可用的 API Key 选择提供商）和**降级链**（主提供商失败时自动切换到备选）。

---

## 5. 交易所支持

ValueCell 支持多个加密货币交易所的实盘/模拟交易：

| 交易所 | 状态 | 说明 |
|--------|------|------|
| **Binance** | ✅ 已测试 | USDT 永续合约，仅支持国际站 |
| **Hyperliquid** | ✅ 已测试 | USDC 保证金，需主钱包地址 + API 钱包私钥 |
| **OKX** | ✅ 已测试 | USDT 永续合约，需 API Key + Secret + 密码 |
| Coinbase | 🟡 部分测试 | 代码已完成，未完全测试 |
| Gate.io | 🟡 部分测试 | 代码已完成，未完全测试 |
| MEXC | 🟡 部分测试 | 代码已完成，未完全测试 |

> **重要**：当前仅支持合约交易（现货以 1 倍合约实现）。使用前务必确保合约账户余额充足，并在模拟盘验证策略后再切换到实盘。

---

## 6. 核心概念

理解 ValueCell 时，建议先把以下概念分清楚。

### 6.1 智能体（Agent）

在 ValueCell 中，智能体（Agent）是一个能够接收用户查询、执行内部逻辑、并以**流式事件**产出结果的服务单元。

每个智能体（Agent）：

- 继承 `BaseAgent`，实现 `stream(...)` 方法（处理用户主动请求）和可选的 `notify(...)` 方法（主动推送通知）
- 通过 `create_wrapped_agent(...)` 被包装成 A2A 服务
- 拥有独立的 Agent Card（服务发现）和 Agent YAML（运行配置）
- 监听独立的端口，作为独立服务运行

### 6.2 编排器（Orchestrator）

后端核心调度中心，负责：

1. 接收用户输入
2. 调用超级智能体（Super Agent）做初步判断
3. 决定是直接回答，还是进入规划流程
4. 调用 Task Executor 去和远程智能体（Agent）交互
5. 处理流式事件并返回给前端

关键设计：**后台 producer 模式**——即便前端连接断开，后台任务仍可继续执行。

### 6.3 超级智能体（Super Agent）

系统的第一层决策器。它不做业务，只做判断：

- **直接回答**：简单 Q&A 或检索式请求
- **转交规划器（Planner）**：需要复杂编排的请求，附带增强后的查询

### 6.4 规划器（Planner）

把自然语言请求转换成可执行计划。如果信息不足或操作风险较高，会进入**人机确认（HITL）**请求用户确认。

存在两种执行模式：

- **标准规划模式**：完整的意图分析 → 计划生成 → 执行
- **直通模式**（planner passthrough）：跳过规划，直接生成单任务计划

### 6.5 人机确认（Human-in-the-Loop, HITL）

系统在关键节点请求用户参与确认的机制，主要出现在规划阶段：

- 缺少必要参数
- 用户意图不明确
- 存在风险操作需要确认

### 6.6 流式响应（Streaming Response）

后端不会等任务全部结束再一次性返回，而是持续输出结构化事件。事件类型在 `valuecell/core/types.py` 中按语义分为五大类：

| 事件大类 | 说明 | 典型事件 |
|----------|------|----------|
| StreamResponseEvent | Agent 流式处理过程中的增量信息 | `message_chunk`、`tool_call_started`、`reasoning` 等 |
| CommonResponseEvent | 跨响应类型共享的富 UI 组件事件 | `component_generator` |
| SystemResponseEvent | 编排层产出的系统级信号 | `conversation_started`、`done` 等 |
| TaskStatusEvent | 单个任务的生命周期状态 | `task_started`、`task_completed` 等 |
| NotifyResponseEvent | Agent 主动推送的通知 | `message` |

前端根据这些事件逐步渲染，用户能看到完整的过程。完整的事件分类、字段说明和触发场景见 [streaming-deep-dive.md §2](./streaming-deep-dive.md)。

### 6.7 A2A 协议（Agent-to-Agent）

ValueCell 通过 A2A 协议把智能体（Agent）看成远程服务，而不是编排器（Orchestrator）内部硬编码函数。这意味着：

- Agent 可以独立服务化、独立部署
- 编排层与 Agent 逻辑完全解耦
- 更容易替换、扩展或远程部署

这种独立服务化的架构在运维上有明显的实际优势：升级或重启某个 Agent 时不会影响其他 Agent 的运行，不同 Agent 也可以根据负载情况独立进行水平扩缩容。对于需要 7x24 小时运行的金融场景（例如新闻监控 Agent 持续运行的同时升级研究 Agent），这种隔离性尤为关键。

### 6.8 数据适配器（Data Adapter）

ValueCell 内置三个免费的金融市场数据适配器，在 `python/valuecell/server/api/app.py` 启动时自动初始化：

| 适配器 | 数据来源 | 需要注册 |
|--------|----------|----------|
| **Yahoo Finance** | `yfinance` 库 | 不需要 |
| **AKShare** | 东方财富等国内数据源 | 不需要 |
| **BaoStock** | 证券宝 | 不需要 |

这三个适配器为 Agent 提供实时和历史行情数据，覆盖美股、港股、A 股和加密货币市场。

### 6.9 Agno 框架

ValueCell 使用 [Agno](https://docs.agno.com)（版本 >=2.0,<3.0）作为 Agent 构建框架。Agno 提供了：

- `Agent` 类：封装 LLM 调用、工具注册、知识库集成
- 流式推理接口：`agent.arun(query, stream=True)` 返回事件流
- 工具定义：普通 Python 函数加上 docstring 即可注册为工具

ValueCell 选择 Agno 的核心原因是其原生支持异步流式输出——`agent.arun(query, stream=True)` 返回的事件流可以自然地映射到 ValueCell 的统一事件格式，省去了额外的流式适配层。相比之下，许多其他 Agent 框架主要围绕请求-响应模式设计，要实现同等水平的流式体验需要大量额外封装。

在 ValueCell 中，每个 Agent 通常创建一个内部的 `agno.Agent` 实例，然后在 `stream()` 方法中将其事件流转换为 ValueCell 的统一事件格式。

### 6.10 组件类型（ComponentType）

`component_generator` 事件可以携带不同类型的 UI 组件，定义在 `valuecell.core.types.ComponentType` 枚举中：

| 组件类型 | 说明 |
|----------|------|
| `report` | 研究报告，支持 Markdown 格式内容 |
| `profile` | 公司或股票档案 |
| `filtered_line_chart` | 可交互的折线图 |
| `filtered_card_push_notification` | 带筛选选项的通知卡片 |
| `scheduled_task_controller` | 定时任务控制面板 |
| `scheduled_task_result` | 定时任务结果展示 |
| `subagent_conversation` | 子 Agent 会话边界标记（`start` / `end` 阶段） |

### 6.11 前端路由结构

前端路由定义在 `frontend/src/routes.ts` 中，主要路由如下：

| 路径 | 页面 | 说明 |
|------|------|------|
| `/home` | 首页 | 系统入口，展示投资组合概览 |
| `/home/stock/:stockId` | 股票详情 | 个股深度分析页面 |
| `/market` | 市场 | 市场数据与 Agent 列表 |
| `/agent/:agentName` | Agent 对话 | 与具体 Agent 交互 |
| `/agent/:agentName/config` | Agent 配置 | Agent 参数设置 |
| `/setting` | 设置 | 设置页面布局 |
| `/setting/models` | 模型设置 | 模型提供商管理 |
| `/setting/general` | 通用设置 | 基本配置 |
| `/setting/memory` | 记忆管理 | 对话历史管理 |

### 6.12 API 路由

后端 API 路由统一前缀为 `/api/v1`，在 `python/valuecell/server/api/app.py` 中注册：

| 路由 | 说明 |
|------|------|
| `/api/v1/healthz` | 健康检查 |
| `/api/v1/agent_stream` | Agent 流式对话（SSE） |
| `/api/v1/conversation` | 会话管理 |
| `/api/v1/agent` | Agent 管理与发现 |
| `/api/v1/task` | 任务管理 |
| `/api/v1/models` | 模型管理 |
| `/api/v1/watchlist` | 自选股管理 |
| `/api/v1/user_profile` | 用户配置 |
| `/api/v1/strategies` | 策略管理 |
| `/api/v1/system` | 系统信息 |
| `/api/v1/i18n` | 国际化资源 |

---

## 7. 产品结构总览

```
┌─────────────────────────────────────────────┐
│               前端层 (frontend/)              │
│  React Router v7 · TanStack Query · i18next   │
│  SSE Client · Zustand · Tailwind CSS          │
│  Tauri v2 桌面打包 · 自动更新 · 健康检查       │
├─────────────────────────────────────────────┤
│              API 层 (server/)                 │
│  FastAPI · uvicorn · .env 加载 · 路由注册     │
│  数据适配器初始化（Yahoo/AKShare/BaoStock）    │
├─────────────────────────────────────────────┤
│           运行时层 (core/)                    │
│  Orchestrator · SuperAgent · Planner          │
│  TaskExecutor · EventRouter · ResponseBuffer  │
│  ConversationStore · ItemStore                │
├─────────────────────────────────────────────┤
│           Agent 层 (agents/)                  │
│  ResearchAgent · NewsAgent                    │
│  PromptBasedStrategyAgent · GridStrategyAgent │
│  common/ · sources/ · utils/                  │
└─────────────────────────────────────────────┘
```

---

## 8. 典型使用场景

### 8.1 面向普通用户

- 下载桌面应用或在线访问
- 选择 Agent 开始对话
- 查看研究分析、新闻追踪、策略运行结果

### 8.2 面向金融研究用户

> **操作入口**：在 Web UI 的 `/market` 页面选择 ResearchAgent，或在地址栏直接访问 `/agent/ResearchAgent` 进入对话。

- 让 Research Agent 提取财报关键信息（营收、EPS、现金流等）
- 分析机构 13F 持仓变动
- 结合知识库做多源信息汇总

### 8.3 面向交易用户

> **操作入口**：策略类 Agent 默认在前端隐藏（`metadata.hidden: true`），需通过 Planner 直通模式触发，或在 API 层直接调用对应 Agent 的流式接口。

- 配置模型与交易所 API
- 选择或构造交易策略
- 在模拟盘验证后切换实盘

### 8.4 面向开发者

- 本地运行整套系统
- 阅读并理解核心编排逻辑
- 开发新的 Agent 扩展能力
- 接入新的模型提供商或数据源

---

## 9. 新手最容易踩的几个认知误区

### 9.1 以为 `.env` 在仓库根目录

当前后端使用的是**系统应用目录**中的 `.env`，不是仓库根目录。首次启动时，系统会将 `.env.example` 复制到系统目录。详见 [configuration.md](./configuration.md)。

### 9.2 以为前端直接连数据库

前端通过后端 API 和流式接口获取所有数据，不直接访问数据库。

### 9.3 以为所有 Agent 都写死在后端

Agent 通过 A2A 协议独立运行，通过 Agent Card 被发现与调用。编排层和 Agent 实现层完全解耦。

### 9.4 以为流式输出只是"打字机效果"

流式输出承载了文本片段、工具调用、推理过程、富 UI 组件、任务状态等结构化信息。前端有 8 个不同的渲染器来处理这些事件类型。

### 9.5 把路线图当作已实现功能

README 中的 Roadmap 描述的是未来计划（如欧股、商品、外汇、期权等），当前仅支持美股研究、加密货币交易和新闻追踪。阅读文档时，务必区分"已实现""部分测试"和"规划中"。

---

## 10. 概念速查：容易混淆的术语辨析

### 10.1 Agent vs Agno Agent

| 概念 | 位置 | 说明 |
|------|------|------|
| **ValueCell Agent** | `python/valuecell/agents/` | 继承 `BaseAgent`，实现 `stream()`，负责 A2A 协议层面的通信 |
| **Agno Agent** | Agent 内部的 LLM 调用封装实例（`agno.Agent`） | Agent 内部的 LLM 调用封装，负责模型选择、工具注册、知识库集成 |

关系：一个 ValueCell Agent 内部通常持有一个 Agno Agent 实例。ValueCell Agent 对外暴露流式事件接口，Agno Agent 对内处理 LLM 交互细节。

### 10.2 Agent Card vs Agent YAML vs Agent 代码

| 组成部分 | 格式 | 职责 |
|----------|------|------|
| **Agent Card**（JSON） | `configs/agent_cards/*.json` | 服务发现、能力声明、前端展示配置 |
| **Agent YAML** | `configs/agents/*.yaml` | 模型选择、参数覆盖、环境变量映射 |
| **Agent 代码**（Python） | `agents/<name>/` | 业务逻辑、流式响应产出 |

三者缺一不可：Card 让系统发现 Agent，YAML 让 Agent 知道用什么模型，代码让 Agent 具备实际能力。

### 10.3 stream() vs notify()

| 方法 | 触发方式 | 返回类型 | 典型场景 |
|------|----------|----------|----------|
| `stream()` | 用户主动发起请求 | `AsyncGenerator[StreamResponse, None]` | 对话回复、研究分析 |
| `notify()` | Agent 主动推送 | `AsyncGenerator[NotifyResponse, None]` | 定时新闻推送、文件监控通知 |

### 10.4 标准规划模式 vs 直通模式

| 模式 | 触发条件 | 流程 | 适用场景 |
|------|----------|------|----------|
| **标准规划** | 默认 | Super Agent → Planner 分析 → 生成多步计划 → 执行 | ResearchAgent、NewsAgent |
| **直通模式** | `metadata.planner_passthrough: true` | Super Agent → 跳过规划 → 直接生成单任务 → 执行 | 策略类 Agent |

### 10.5 流式事件 vs 一次性响应

| 特性 | 流式事件 | 一次性响应 |
|------|----------|-----------|
| 用户体验 | 实时看到过程 | 等待后一次性看到结果 |
| 技术实现 | SSE + `AsyncGenerator` | 同步函数 `return` |
| 事件类型 | 文本、推理、工具调用、组件等 | 单一结果 |
| 恢复能力 | 断线可恢复 | 不可恢复 |

ValueCell 全面采用流式事件架构，不使用一次性响应模式。

---

## 11. 常见问题

### Q1：ValueCell 支持哪些语言模型？能否同时使用多个提供商？

支持。ValueCell 当前已配置 9 个 LLM 提供商（OpenRouter、SiliconFlow、Google、OpenAI、Azure、DeepSeek、DashScope、Ollama、OpenAI-Compatible），系统支持自动检测和降级链机制——主提供商不可用时会自动切换到备选。此外，每个 Agent 可以通过 Agent YAML 独立配置模型，覆盖提供商级别的默认值。

### Q2：ValueCell 支持股票交易吗？

当前不支持股票交易。ValueCell 的交易能力仅限于加密货币合约交易，支持 Binance、Hyperliquid、OKX 等交易所（详见第 5 节）。股票研究方面，Research Agent 可以分析 SEC 财报、机构持仓等数据，但不会执行股票买卖操作。

### Q3：为什么策略类 Agent 在前端看不到？

策略类 Agent（PromptBasedStrategyAgent、GridStrategyAgent）的 Agent Card 中设置了 `metadata.hidden: true`，因此不会出现在前端 Agent 列表中。这类 Agent 采用直通模式（planner passthrough），通常由 Planner 在处理交易相关请求时自动调用，而非用户直接对话。如果需要直接交互，可通过 API 层调用对应 Agent 的流式接口。

### Q4：数据会保存在哪里？安全吗？

敏感数据（包括 `.env` 配置文件、SQLite 数据库、LanceDB 向量库和知识库数据）保存在操作系统级别的应用目录中，而不是仓库目录。前端不直接访问数据库，所有数据交互通过后端 API 完成。本地部署模式下，数据完全在用户本机，不会上传到外部服务器。

### Q5：流式输出和普通的"打字机效果"有什么区别？

流式输出不仅仅是逐字显示文本。它承载了 5 大类结构化事件（StreamResponseEvent、CommonResponseEvent、SystemResponseEvent、TaskStatusEvent、NotifyResponseEvent），包括文本片段、工具调用过程、推理链、富 UI 组件（图表、报告、通知卡片等）和任务状态变更。前端有 8 个不同的渲染器来处理这些事件类型，使得用户可以实时看到 Agent 的完整工作过程，而不仅仅是最终结果。

---

## 12. 自测检查清单

阅读完本文后，可以用以下清单检验自己的理解程度。如果每一条都能给出大致准确的回答，说明你已经掌握了 ValueCell 的核心概念：

- [ ] 能说出 ValueCell 的产品定位（面向金融场景的多智能体平台）和 5 个核心目标
- [ ] 能区分"已实现"和"规划中"的能力（例如哪些交易所已测试、哪些是部分测试）
- [ ] 能列出当前 4 个 Agent 的名称、端口和主要职责
- [ ] 能解释编排器（Orchestrator）、超级智能体（Super Agent）和规划器（Planner）三者之间的协作关系
- [ ] 能说明标准规划模式和直通模式的区别，以及各自适用的 Agent 类型
- [ ] 能描述 A2A 协议的核心思想（Agent 作为独立服务，与编排层解耦）及其运维优势
- [ ] 能解释 ValueCell Agent 和 Agno Agent 的关系（前者负责 A2A 协议通信，后者封装 LLM 调用）
- [ ] 能说出流式输出承载的 5 大类事件及其含义
- [ ] 能区分 Agent Card（服务发现）、Agent YAML（模型配置）和 Agent 代码（业务逻辑）三者的职责
- [ ] 能指出新手常见的认知误区（例如 `.env` 的实际位置、前端与数据库的关系）

---

## 13. 术语速查表

以下是本文涉及的关键术语的快速参考，方便随时查阅：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 智能体 | Agent | 能够接收用户查询、执行逻辑并以流式事件产出结果的独立服务单元 |
| 编排器 | Orchestrator | 后端核心调度中心，负责接收输入、路由决策和协调各 Agent 执行 |
| 超级智能体 | Super Agent | 系统的第一层决策器，判断是直接回答还是转交规划器 |
| 规划器 | Planner | 将自然语言请求转换为可执行的多步计划，信息不足时触发人机确认 |
| 人机确认 | Human-in-the-Loop (HITL) | 在关键节点（缺少参数、意图不明、高风险操作）请求用户确认的机制 |
| 流式响应 | Streaming Response | 持续输出结构化事件（文本、工具调用、推理、组件等）的实时通信方式 |
| A2A 协议 | Agent-to-Agent | 将 Agent 视为独立远程服务的通信协议，实现编排层与 Agent 的完全解耦 |
| Agno 框架 | Agno | ValueCell 使用的 Agent 构建框架，提供 LLM 调用封装和原生异步流式支持 |
| Agent Card | Agent Card | JSON 格式的服务发现文件，声明 Agent 的能力、端口和前端展示配置 |
| Agent YAML | Agent YAML | YAML 格式的运行配置文件，定义模型选择、参数覆盖和环境变量映射 |
| 直通模式 | Planner Passthrough | 跳过规划阶段，直接将请求转为单任务执行的简化模式，适用于策略类 Agent |
| 数据适配器 | Data Adapter | 内置的金融市场数据接口（Yahoo Finance、AKShare、BaoStock），提供实时和历史行情 |
| 组件类型 | ComponentType | 流式事件中携带的 UI 组件类型枚举（报告、图表、通知卡片等） |
| SSE | Server-Sent Events | 前端与后端之间实现流式通信的技术方式 |
| 降级链 | Fallback Chain | 主 LLM 提供商失败时自动切换到备选提供商的容错机制 |

---

## 14. 下一步该看什么

| 你想做什么 | 推荐阅读 |
|-----------|----------|
| 马上把系统跑起来 | [快速上手](./getting-started.md) |
| 搞清楚配置系统 | [配置说明](./configuration.md) |
| 理解系统如何运转 | [系统架构详解](./architecture.md) |
| 从代码开始深入 | [源码导读](./source-guide.md) |
| 自己开发 Agent | [Agent 开发与扩展](./agent-extension.md) |
| 看看实际使用场景 | [使用场景与实战指南](./use-cases.md) |
| 理解设计决策动机 | [核心设计原理](./principles.md) |
| 系统化学习路径 | [学习路径](./learning-path.md) |
