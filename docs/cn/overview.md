# ValueCell 项目概览

## 学习目标

阅读本文后，你将能够：

- 准确描述 ValueCell 的产品定位和核心价值
- 区分"已实现"和"规划中"的能力
- 解释系统各层的职责和协作方式
- 理解 Agent、Orchestrator、Planner 等核心概念
- 避免新手常见的认知误区

---

## 1. 项目是什么

ValueCell 是一个**社区驱动的、面向金融场景的多智能体（Multi-Agent）平台**。它的目标不是做一个聊天机器人，而是构建一个由多个专业 AI Agent 协同工作的金融助手团队。

从仓库当前实现来看，ValueCell 由以下部分组成：

| 层级 | 职责 | 技术 |
|------|------|------|
| **前端层** | 用户界面、流式渲染、状态管理 | React Router v7、TypeScript、Vite |
| **API 层** | 服务入口、路由、初始化 | FastAPI、uvicorn |
| **运行时层** | 编排、规划、执行、事件处理 | Python asyncio |
| **Agent 层** | 具体金融能力实现 | Agno 框架、A2A 协议 |

### 1.1 核心目标

1. **统一入口**：用户通过一个界面与不同类型的 Agent 交互
2. **Agent 协作**：支持研究、新闻、策略等不同能力的 Agent 协同工作
3. **数据本地化**：配置、数据库、知识库等敏感数据保存在本地系统目录
4. **实时反馈**：通过流式响应持续展示推理过程、工具调用和任务结果

### 1.2 在线版本

ValueCell 已推出在线服务，提供 A 股深度研究和市场分析功能，无需本地部署，直接访问 [valuecell.ai](https://valuecell.ai) 即可使用。

---

## 2. 它不是什么

为了避免误解，明确以下几点：

- 它**不是**单一模型提供商的固定应用——仓库支持 **9 个** LLM 提供商接入
- 它**不是**只做前端展示——核心价值在于后端的编排、规划、任务执行和 Agent 协议集成
- 它**不是**纯命令式任务脚本——带有规划、流式事件、对话持久化和 HITL（Human-in-the-loop，人机确认）机制
- 它**不是**把所有状态写在仓库根目录——使用系统级应用目录保存 `.env`、SQLite、LanceDB 和知识数据

---

## 3. 当前已实现的 Agent

根据仓库中的 Agent Card 配置，当前有以下 Agent：

### 3.1 ResearchAgent（研究 Agent）

**文件**：`investment_research_agent.json` → `python/valuecell/agents/research_agent/`

分析 SEC 财报（10-K、10-Q、13F、8-K）和内部知识库，生成可溯源的摘要和数据提取。

| 能力 | 说明 |
|------|------|
| 财务数据提取 | 从财报中提取营收、净收入、EPS、现金流等指标 |
| 13F 持仓分析 | 解析机构持仓、仓位变动 |
| 知识库综合 | 结合知识库搜索结果提供解读和上下文 |
| 文件监控与通知 | 跟踪公司文件提交并发送摘要 |

### 3.2 NewsAgent（新闻 Agent）

**文件**：`news_agent.json` → `python/valuecell/agents/news_agent/`

实时新闻检索与分析，支持个性化定时推送。

| 能力 | 说明 |
|------|------|
| 实时新闻搜索 | 通过网络搜索获取当前新闻 |
| 突发新闻监控 | 监控并获取紧急新闻更新 |
| 金融市场新闻 | 综合性金融市场和商业新闻 |

### 3.3 PromptBasedStrategyAgent（提示策略 Agent）

**文件**：`prompt_strategy_agent.json` → `python/valuecell/agents/prompt_strategy_agent/`

LLM 驱动的策略编排器，将市场特征转化为标准化交易指令。

- 使用 `planner_passthrough` 模式（跳过规划器，直接执行）
- 前端默认隐藏（`metadata.hidden: true`）

### 3.4 GridStrategyAgent（网格策略 Agent）

**文件**：`grid_agent.json` → `python/valuecell/agents/grid_agent/`

网格交易策略 Agent，同样使用 `planner_passthrough` 模式。

> **注意**：策略类 Agent 涉及实盘交易，当前状态因交易所而异（见第 5 节），请谨慎使用。

---

## 4. 模型提供商支持

ValueCell 通过统一的配置接口支持多个 LLM 提供商，当前已配置 **9 个**提供商：

| 提供商 | 配置文件 | 特点 |
|--------|----------|------|
| **OpenRouter** | `providers/openrouter.yaml` | 聚合多模型，模型最全 |
| **SiliconFlow** | `providers/siliconflow.yaml` | 性价比高，支持中文模型 |
| **Google** | `providers/google.yaml` | Gemini 系列 |
| **OpenAI** | `providers/openai.yaml` | GPT 系列 |
| **Azure** | `providers/azure.yaml` | 企业级部署 |
| **DeepSeek** | `providers/deepseek.yaml` | DeepSeek 模型 |
| **DashScope** | `providers/dashscope.yaml` | 阿里云百炼（Qwen 系列） |
| **Ollama** | `providers/ollama.yaml` | 本地模型部署 |
| **OpenAI-Compatible** | `providers/openai-compatible.yaml` | 兼容接口的自定义提供商 |

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

### 6.1 Agent（智能体）

在 ValueCell 中，Agent 是一个能够接收用户查询、执行内部逻辑、并以**流式事件**产出结果的服务单元。

每个 Agent：

- 继承 `BaseAgent`，实现 `stream(...)` 方法
- 通过 `create_wrapped_agent(...)` 被包装成 A2A 服务
- 拥有独立的 Agent Card（服务发现）和 Agent YAML（运行配置）
- 监听独立的端口，作为独立服务运行

### 6.2 Orchestrator（编排器）

后端核心调度中心，负责：

1. 接收用户输入
2. 调用 Super Agent 做初步判断
3. 决定是直接回答，还是进入规划流程
4. 调用 Task Executor 去和远程 Agent 交互
5. 处理流式事件并返回给前端

关键设计：**后台 producer 模式**——即便前端连接断开，后台任务仍可继续执行。

### 6.3 Super Agent（超级智能体）

系统的第一层决策器。它不做业务，只做判断：

- **直接回答**：简单 Q&A 或检索式请求
- **转交规划器**：需要复杂编排的请求，附带增强后的查询

### 6.4 Planner（规划器）

把自然语言请求转换成可执行计划。如果信息不足或操作风险较高，会进入 **HITL** 请求用户确认。

存在两种执行模式：

- **标准规划模式**：完整的意图分析 → 计划生成 → 执行
- **直通模式**（planner passthrough）：跳过规划，直接生成单任务计划

### 6.5 HITL（Human-in-the-loop）

系统在关键节点请求用户参与确认的机制，主要出现在规划阶段：

- 缺少必要参数
- 用户意图不明确
- 存在风险操作需要确认

### 6.6 流式响应（Streaming Response）

后端不会等任务全部结束再一次性返回，而是持续输出结构化事件：

| 事件类型 | 含义 |
|----------|------|
| `message_chunk` | 文本片段 |
| `tool_call_started` | 工具调用开始 |
| `tool_call_completed` | 工具调用完成 |
| `reasoning_started/reasoning/reasoning_completed` | 推理过程 |
| `component_generator` | 富 UI 组件（图表、报告等） |
| `done` | 响应结束 |

前端根据这些事件逐步渲染，用户能看到完整的过程。

### 6.7 A2A 协议（Agent-to-Agent）

ValueCell 通过 A2A 协议把 Agent 看成远程服务，而不是编排器内部硬编码函数。这意味着：

- Agent 可以独立服务化、独立部署
- 编排层与 Agent 逻辑完全解耦
- 更容易替换、扩展或远程部署

---

## 7. 产品结构总览

```
┌─────────────────────────────────────────────┐
│               前端层 (frontend/)              │
│  React Router v7 · TanStack Query · i18next   │
│  SSE Client · Zustand · Tailwind CSS          │
├─────────────────────────────────────────────┤
│              API 层 (server/)                 │
│  FastAPI · uvicorn · .env 加载 · 路由注册     │
├─────────────────────────────────────────────┤
│           运行时层 (core/)                    │
│  Orchestrator · SuperAgent · Planner          │
│  TaskExecutor · EventRouter · ResponseBuffer  │
├─────────────────────────────────────────────┤
│           Agent 层 (agents/)                  │
│  ResearchAgent · NewsAgent                    │
│  PromptBasedStrategyAgent · GridStrategyAgent │
└─────────────────────────────────────────────┘
```

---

## 8. 典型使用场景

### 8.1 面向普通用户

- 下载桌面应用或在线访问
- 选择 Agent 开始对话
- 查看研究分析、新闻追踪、策略运行结果

### 8.2 面向金融研究用户

- 让 Research Agent 提取财报关键信息（营收、EPS、现金流等）
- 分析机构 13F 持仓变动
- 结合知识库做多源信息汇总

### 8.3 面向交易用户

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

## 10. 下一步该看什么

| 你想做什么 | 推荐阅读 |
|-----------|----------|
| 马上把系统跑起来 | [快速上手](./getting-started.md) |
| 搞清楚配置系统 | [配置说明](./configuration.md) |
| 理解系统如何运转 | [系统架构详解](./architecture.md) |
| 从代码开始深入 | [源码导读](./source-guide.md) |
| 自己开发 Agent | [Agent 开发与扩展](./agent-extension.md) |
| 系统化学习路径 | [学习路径](./learning-path.md) |
