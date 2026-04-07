# ValueCell 系统架构详解

## 学习目标

阅读本文后，你将能够：

- 画出一次请求从前端到 Agent 再回到前端的完整链路
- 解释 Orchestrator、Super Agent、Planner、TaskExecutor 各自的职责边界
- 理解流式事件的产生、路由、缓冲和持久化机制
- 知道某类需求应该改哪一层

---

## 1. 系统全局视图

ValueCell 是一条从用户请求到多智能体执行再到前端实时展示的链路：

```
用户请求 → 前端 → 后端 API → Orchestrator → Super Agent（分流）
                                            ↓
                                    ┌── 直接回答
                                    └── Planner（规划）
                                            ↓
                                    HITL 确认（如需）
                                            ↓
                                    TaskExecutor（执行）
                                            ↓
                                    远程 Agent（A2A）
                                            ↓
                                    事件路由 → 缓冲 → 持久化
                                            ↓
                                    前端按事件逐步渲染
```

---

## 2. 前端层

前端位于 `frontend/`，核心依赖：

| 技术 | 用途 |
|------|------|
| React Router v7 | 路由和页面框架 |
| TanStack Query | 服务端状态管理 |
| Zustand | 本地持久化状态 |
| i18next | 多语言国际化 |
| 自定义 SSE Client | 流式事件消费 |

### 2.1 路由结构

当前主要路由（来自 `frontend/src/routes.ts`）：

| 路径 | 页面 |
|------|------|
| `/home` | 首页，展示投资组合概览 |
| `/home/stock/:stockId` | 股票详情页，个股深度分析 |
| `/market` | 市场数据与 Agent 列表 |
| `/agent/:agentName` | Agent 对话 |
| `/agent/:agentName/config` | Agent 配置 |
| `/setting` | 设置总览 |
| `/setting/models` | 模型提供商管理 |
| `/setting/general` | 通用设置 |
| `/setting/memory` | 记忆管理 |

### 2.2 根布局注入

`frontend/src/root.tsx` 在根层统一注入：

- **QueryClient**：所有服务端数据的缓存和失效
- **ThemeProvider**：暗色/亮色主题
- **Sidebar**：全局导航
- **BackendHealthCheck**：后端健康探测
- **AutoUpdateCheck**：桌面应用自动更新
- **TrackerProvider**：使用数据追踪

### 2.3 SSE 客户端

`frontend/src/lib/sse-client.ts` **不是**原生 `EventSource`，而是基于 `fetch + ReadableStream` 的自定义实现：

- 使用 **POST** 请求（可携带 body 和自定义 header）
- 自己拆分 `data:` 块并解析 JSON
- 支持超时和重连

> **为什么不直接用原生 EventSource？** 原生 `EventSource` 只支持 GET 请求，不能发送自定义 body。ValueCell 需要在请求体中传递对话消息，因此必须自行实现。

### 2.4 渲染器

`frontend/src/components/valuecell/renderer/index.tsx` 定义了 **8 个渲染器**，每种处理不同的事件类型：

| 渲染器 | 用途 |
|--------|------|
| ChatConversationRenderer | 对话容器 |
| MarkdownRenderer | 普通文本消息 |
| ReasoningRenderer | Agent 推理过程 |
| ReportRenderer | 研究报告 |
| ScheduledTaskControllerRenderer | 定时任务控制面板 |
| ScheduledTaskRenderer | 定时任务执行结果 |
| ToolCallRenderer | 工具调用展示 |
| UnknownRenderer | 未知事件兜底 |

---

## 3. API 层

后端 API 位于 `python/valuecell/server/`，由 FastAPI 暴露。

### 3.1 启动职责

`server/api/app.py` 的 `create_app()` 在启动时执行：

1. 确保系统 `.env` 存在并加载（`_ensure_system_env_and_load()`）
2. 创建 FastAPI app 实例
3. 初始化数据库（SQLite）
4. 初始化市场数据适配器（Yahoo Finance、AKShare、BaoStock）
5. 注册 API 路由（统一前缀 `/api/v1`）

### 3.2 API 路由清单

后端 API 路由统一前缀为 `/api/v1`，当前注册的路由如下：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/healthz` | 健康检查 |
| POST | `/api/v1/agent_stream` | Agent 流式对话（SSE） |
| GET/POST/PUT/DELETE | `/api/v1/conversation` | 会话管理 |
| GET | `/api/v1/agent` | Agent 发现与列表 |
| GET/POST/PUT/DELETE | `/api/v1/task` | 任务管理 |
| GET/POST | `/api/v1/models` | 模型管理 |
| GET/POST/PUT/DELETE | `/api/v1/watchlist` | 自选股管理 |
| GET/PUT | `/api/v1/user_profile` | 用户配置 |
| GET/POST | `/api/v1/strategies` | 策略管理 |
| GET | `/api/v1/system` | 系统信息 |
| GET | `/api/v1/i18n` | 国际化资源 |

#### 市场数据适配器初始化

在 lifespan 阶段，系统会依次初始化三个免费数据适配器：

```python
# server/api/app.py 中的适配器配置（简化）
manager.configure_yfinance()    # Yahoo Finance — 美股、港股
manager.configure_akshare()     # AKShare — A 股、国内市场
manager.configure_baostock()    # BaoStock — A 股历史数据
```

三个适配器均无需 API Key，启动即用。它们为 Agent 提供实时和历史行情数据。

### 3.3 API 层不是"业务逻辑终点"

分清两层职责：

- `server/`：服务化、路由化、启动与初始化
- `core/`：对话编排、任务执行、事件持久化

API 层是入口，真正的运行时在 `core/`。

---

## 4. 编排运行时层

`python/valuecell/core/` 是系统真正的核心。它是一个**多智能体运行时**，不是简单的工具函数集合。

### 4.1 Orchestrator：总调度中心

**核心文件**：`python/valuecell/core/coordinate/orchestrator.py`

**关键常量**：

| 常量 | 值 | 说明 |
|------|-----|------|
| `DEFAULT_CONTEXT_TIMEOUT_SECONDS` | `3600`（1 小时） | 执行上下文过期时间 |
| `ASYNC_SLEEP_INTERVAL` | `0.1`（100ms） | Planner 轮询间隔 |

**执行上下文（`ExecutionContext`）**：Orchestrator 为每个等待用户输入（HITL）的会话维护一个 `ExecutionContext`，存储在内存字典 `_execution_contexts` 中。上下文包含：

- `stage`：当前阶段（目前仅支持 `"planning"`）
- `conversation_id`：关联的会话 ID
- `user_id`：发起用户
- `created_at`：创建时间（用于过期检查）
- `metadata`：附加数据（如原始用户输入、Planning Task 引用、回调函数）

上下文有过期检查机制：`is_expired(max_age_seconds=3600)`，超时后的 HITL 恢复请求会被拒绝。系统会定期调用 `_cleanup_expired_contexts()` 清理过期上下文。

`AgentOrchestrator.process_user_input(...)` 是主入口，职责：

1. 接收用户输入
2. 确保会话存在（或恢复已有会话）
3. 判断当前会话是否在等待用户输入（HITL 恢复场景）
4. 启动后台 producer task 处理请求
5. 通过队列持续产出流式响应

#### 关键设计：后台 producer 模式

```
                     ┌─────────────────────────┐
                     │     Orchestrator          │
                     │                          │
  前端 SSE 连接 ────→│  Consumer ← Queue        │
  (可断开重连)       │       ↑                  │
                     │  Producer (后台 task)     │
                     │   ↓                      │
                     │  Super Agent → Planner   │
                     │   → TaskExecutor → Agent  │
                     └─────────────────────────┘
```

**核心设计意图**：

- 请求进来后，不是同步执行完再返回，而是启动后台 producer task
- 用 `asyncio.Queue` 把响应持续推给 consumer
- **即便前端 SSE 连接断开，后台长任务仍可继续执行**
- 前端重连后，从持久化的 Store 中恢复状态

> **这是理解 ValueCell 流式架构的关键点。** 如果你只能读一个后端文件，就先读 `orchestrator.py`。

### 4.2 Super Agent：先分流，再决定是否规划

`super_agent/` 目录结构：

| 文件 | 职责 |
|------|------|
| `core.py` | `SuperAgent` 主逻辑：决策 ANSWER 或 HANDOFF_TO_PLANNER |
| `service.py` | 薄封装层，供 Orchestrator 调用 |
| `prompts.py` | 指令模板 |

Super Agent 不做业务，只做第一层判断。它的内部实现细节：

**类名常量**：`SuperAgent.name = "ValueCellAgent"`——这个名字也是前端发送请求时的默认 `target_agent_name`。Orchestrator 在 `_handle_new_request` 中检查 `user_input.target_agent_name == self.super_agent_service.name`，只有匹配时才执行 Super Agent 分流逻辑。

**模型初始化**：`_get_or_init_agent()` 使用延迟初始化模式——第一次调用时才创建 Agno Agent 实例。它从 `get_model_for_agent("super_agent")` 获取模型配置。**注意**：`super_agent.yaml` 中的 `models` 段是全部注释掉的，实际模型来自系统默认提供商的默认模型。注释掉的 models 段意味着 Super Agent 使用系统默认提供商的默认模型，这是一种设计选择——让 Super Agent 的模型跟随全局配置而非单独指定。

**热重载**：`_get_or_init_agent()` 会检测 model_id 和 provider 的变化，如果检测到变化，自动重建 Agent 实例（透明切换模型，无需重启）。

**JSON 模式**：Super Agent 使用 `output_schema=SuperAgentOutcome` 强制 LLM 输出 JSON 格式的决策结果。对于 DashScope 等不支持 JSON 模式的提供商，会自动降级。

**失败降级**：如果 Super Agent 初始化失败（如模型不可用）或执行出错，`run()` 方法会自动降级为 `HANDOFF_TO_PLANNER`，将所有请求交给 Planner 处理。

**历史上下文**：Super Agent 保留了最近 5 轮对话历史（`num_history_runs=5`），并启用了会话摘要功能（`enable_session_summaries=True`），以便在多轮对话中保持上下文连贯。

决策结果（`SuperAgentOutcome`）：

```
用户请求 → Super Agent → ANSWER（直接回答）
                      → HANDOFF_TO_PLANNER（转交规划器，附带增强后的查询）
```

它相当于系统的**第一层路由器**，价值在于快速分流，而不是承担所有逻辑。

### 4.3 Planner：把自然语言变成可执行步骤

`plan/` 目录结构：

| 文件 | 职责 |
|------|------|
| `service.py` | PlanService：管理规划器生命周期和 HITL 注册表 |
| `planner.py` | ExecutionPlanner：核心决策逻辑 |
| `models.py` | 计划/步骤数据模型 |
| `prompts.py` | Prompt 模板（LLM 规划时使用） |

Planner 负责：

- 识别用户目标
- 发现缺少的信息
- 生成可执行计划

#### HITL（Human-in-the-loop）

当 Planner 判断信息不足或操作风险较高时：

1. 发出 `PLAN_REQUIRE_USER_INPUT` 事件
2. Orchestrator 将确认请求推送给前端
3. 用户回复后，Orchestrator 恢复规划流程
4. Planner 基于补充信息继续生成计划

#### 两种执行模式

**标准规划模式**：

```
请求 → Super Agent → Planner 分析 → 生成多步计划 → 逐步执行
```

**直通模式**（planner passthrough）：

```
请求 → Super Agent → 跳过规划 → 直接生成单任务计划 → 立即执行
```

直通模式由 Agent Card 中的 `metadata.planner_passthrough: true` 触发。`PromptBasedStrategyAgent` 和 `GridStrategyAgent` 使用这种模式。

> **调试时注意**：如果你在调试某个策略 Agent，不要惊讶于没有进入完整的 Planner 推理流程——它可能走的是直通模式。

### 4.4 TaskExecutor：执行计划

`task/` 目录结构：

| 文件 | 职责 |
|------|------|
| `executor.py` | 执行引擎：驱动任务、处理事件 |
| `service.py` | 任务持久化和状态转换 |
| `models.py` | 任务数据模型 |

TaskExecutor 负责**真正去做**，而不是决定做什么。它承担六件事：

1. 遍历计划中的任务
2. 为 handoff 场景发出子 Agent 会话组件
3. 启动任务状态跟踪
4. 调用底层 Agent 连接执行任务
5. 处理定时任务的多次执行
6. 在完成/失败/取消等节点更新状态并发出事件

#### handoff 机制

当任务从 Super Agent handoff 给子 Agent 时，执行器会额外生成：

- 子 Agent **会话开始**组件
- 子 Agent **会话结束**组件

这意味着前端看到的不只是"文本输出"，还可能看到代表子 Agent 对话窗口边界的结构化组件。

#### 定时任务

TaskExecutor 还包含定时任务逻辑：

- 定时任务可以多次执行
- 运行中累积输出
- 最终汇总为 scheduled task result component

> **TaskExecutor 不是"只处理一次短调用"的简单封装**，而是带生命周期管理的任务运行器。

---

## 5. Agent 层

Agent 实现位于 `python/valuecell/agents/`，每个 Agent 通常有三部分：

### 5.1 Agent 代码实现

```
python/valuecell/agents/<agent_name>/
├── __init__.py
├── __main__.py          # 独立运行入口
└── core.py              # 核心逻辑（继承 BaseAgent，实现 stream()）
```

当前 Agent：

| Agent | 类名 | Card 文件名 | 端口 |
|-------|------|------|------|
| 研究 Agent | `ResearchAgent` | `investment_research_agent.json` | 10004 |
| 新闻 Agent | `NewsAgent` | `news_agent.json` | 10005 |
| 提示策略 Agent | `PromptBasedStrategyAgent` | `prompt_strategy_agent.json` | 10006 |
| 网格策略 Agent | `GridStrategyAgent` | `grid_agent.json` | 10007 |

### 5.2 Agent Card

位于 `python/configs/agent_cards/*.json`，决定：

- 系统如何发现这个 Agent
- 它监听哪个 URL
- 它具备哪些能力（skills）
- 前端如何展示它

以 `investment_research_agent.json` 为例，关键字段：

```json
{
  "name": "ResearchAgent",          // 必须与类名一致
  "display_name": "Research Agent", // 前端显示名
  "url": "http://localhost:10004/", // 服务地址
  "skills": [...],                  // 能力声明
  "enabled": true,
  "metadata": {
    "local_agent_class": "valuecell.agents.research_agent.core:ResearchAgent"
  }
}
```

### 5.3 Agent YAML 配置

位于 `python/configs/agents/*.yaml`，决定：

- 使用哪个模型
- 模型参数（温度、最大 Token 等）
- 环境变量覆盖映射

### 5.4 A2A 协议的意义

ValueCell 通过 **Agent-to-Agent（A2A）协议**把 Agent 看成远程服务：

```
编排层 (Orchestrator) ←──A2A协议──→ Agent (独立服务)
```

好处：

- Agent 可以独立服务化、独立部署
- 编排层与 Agent 逻辑完全解耦
- 更容易替换、扩展或远程部署
- Agent 可以用不同的框架实现（当前使用 Agno）

### 5.5 Agno 框架的角色

ValueCell 使用 [Agno](https://docs.agno.com)（v2.x）作为 Agent 构建框架。在架构中，Agno 承担：

- **LLM 调用封装**：`agno.Agent` 封装了模型选择、工具注册、知识库集成
- **流式推理**：`agent.arun(query, stream=True)` 返回事件流
- **工具管理**：普通 Python 函数 + docstring 即可注册为 LLM 可调用的工具

典型 Agent 的内部结构：

```
ValueCell Agent (继承 BaseAgent)
 └── agno.Agent (内部实例)
      ├── model (来自 get_model_for_agent())
      ├── tools[] (注册的工具函数)
      └── knowledge (可选的知识库)
```

Agno 事件到 ValueCell 事件的映射：

| Agno 事件 | ValueCell 事件 |
|-----------|---------------|
| `RunContent` | `streaming.message_chunk()` |
| `ToolCallStarted` | `streaming.tool_call_started()` |
| `ToolCallCompleted` | `streaming.tool_call_completed()` |

**组件类型（ComponentType 枚举）**：

| 组件类型 | 说明 |
|----------|------|
| `report` | 研究报告 |
| `profile` | 公司/股票档案 |
| `subagent_conversation` | 子 Agent 会话边界标记 |
| `filtered_line_chart` | 可交互折线图 |
| `filtered_card_push_notification` | 带筛选的通知卡片 |
| `scheduled_task_controller` | 定时任务控制面板 |
| `scheduled_task_result` | 定时任务结果展示 |

> **注意**：Agno 是 Agent 内部使用的框架，与 A2A 协议是不同层面的概念。Agno 解决"如何构建 Agent"，A2A 解决"如何发现和通信"。

---

## 6. 事件处理链路

这是理解 ValueCell 最关键的部分。

### 6.1 流式事件类型

在 `valuecell.core.types` 中定义的事件类型按语义分为五大类：

| 事件大类 | 说明 | 典型事件 |
|----------|------|----------|
| StreamResponseEvent | Agent 流式处理过程中的增量信息 | `message_chunk`、`tool_call_started`、`reasoning` 等 |
| CommonResponseEvent | 跨响应类型共享的富 UI 组件事件 | `component_generator` |
| SystemResponseEvent | 编排层产出的系统级信号 | `conversation_started`、`done` 等 |
| TaskStatusEvent | 单个任务的生命周期状态 | `task_started`、`task_completed` 等 |
| NotifyResponseEvent | Agent 主动推送的通知 | `message` |

这些事件不是装饰性的——它们**决定前端渲染什么**。完整的事件分类、字段说明和触发场景见 [streaming-deep-dive.md §2](./streaming-deep-dive.md)。

### 6.2 事件处理三层架构

```
Agent 产出事件
     ↓
ResponseRouter（router.py）── 语义分类：工具调用 / 推理 / 组件 / 消息 / 失败
     ↓
ResponseBuffer（buffer.py）── 分配稳定 item_id + 聚合分片消息
     ↓
EventResponseService（service.py）── 持久化到 Store
     ↓
前端消费并渲染
```

### 6.3 ResponseRouter 的实际职责

`core/event/router.py` 把底层 `TaskStatusUpdateEvent` 转成统一响应：

- 按语义分类：工具调用事件、推理事件、组件生成事件、一般消息事件、失败事件
- 失败时可以返回 side effect（如要求标记任务为失败）

### 6.4 ResponseBuffer 的实际职责

`core/event/buffer.py` 解决"流式消息为什么不会把界面刷成碎片"：

1. 为流式段落分配**稳定的 `item_id`**
2. 把可缓冲事件按上下文聚合
3. 在"立即事件"到来时触发 flush

所以，前端能看到一条消息"逐渐长出来"，不是前端猜的，而是后端显式维护了流式段落的身份和边界。

### 6.5 EventResponseService 的位置

`core/event/service.py` 是统一门面：

```
创建响应 → 调用缓冲器处理 → 持久化到会话存储
```

它把"响应构造"和"持久化细节"从 Orchestrator / TaskExecutor 中抽走了。

---

## 7. 对话与持久化

ValueCell 不仅"显示结果"，还会把结果保存下来。

### 7.1 持久化的内容

- **会话**（Conversation）：一次完整交互的上下文
- **线程**（Thread）：会话内的消息序列
- **任务**（Task）：执行计划中的单个步骤
- **消息项**（Item）：流式消息的原子单位

### 7.2 存储后端

两个核心存储文件位于 `python/valuecell/core/conversation/`：

| 文件 | 职责 |
|------|------|
| `conversation_store.py` | 会话的创建、查询、状态管理 |
| `item_store.py` | 消息项的存储、查询、按时间排序 |

存储后端支持：

- **SQLite**（默认）：数据持久化到系统目录中的 `valuecell.db`
- **内存**：开发调试用的轻量后端
- 过滤和分页支持
- 快速"恢复到上次"的能力（获取最新 item）

### 7.3 为什么需要持久化

持久化支撑了三个关键能力：

1. **历史记录**：用户可以回看过去的对话
2. **会话恢复**：前端断开重连后，从 Store 中恢复状态
3. **HITL 恢复**：用户确认后，Orchestrator 能从断点继续

---

## 8. 一条请求的完整旅程

把一次典型请求理解为以下步骤，附带各阶段的参考耗时：

```
1. 用户在前端输入查询
     ↓                                              ── 即时
2. 前端通过 SSE Client 发送 POST 请求
     ↓                                              ── 网络延迟（本地 ~5ms）
3. API 层接收，转发给 Orchestrator
     ↓                                              ── FastAPI 路由 ~1ms
4. Orchestrator 确保会话存在，启动后台 producer
     ↓                                              ── 会话查询 + 初始化 ~10ms
5. Super Agent 分析，决定：
   ├─ ANSWER → 直接产出消息 → 事件处理 → 前端渲染 → done
   └─ HANDOFF → Planner 分析
     ↓                                              ── Super Agent 分流 1-3 秒
                                                     （取决于 LLM 响应速度）
6. Planner 生成计划（如需确认，进入 HITL）
     ↓                                              ── Planner 推理 2-5 秒
                                                     （HITL 场景需等待用户响应）
7. TaskExecutor 按计划调用远程 Agent（A2A）
     ↓                                              ── Agent 初始化 + 工具准备 ~0.5-1 秒
8. Agent 产出流式事件
     ↓                                              ── 持续流式输出，首个 token 0.5-2 秒
9. ResponseRouter 语义分类 → ResponseBuffer 分配 ID/聚合
     ↓                                              ── 内存操作，<1ms
10. EventResponseService 持久化到 Store
     ↓                                              ── SQLite 写入 ~1-5ms
11. 前端收到事件 → 8 个渲染器按类型渲染
     ↓                                              ── React 渲染周期 ~16ms（一帧）
12. done 事件 → 前端标记完成
```

**整体耗时参考**：

- **简单问答（Super Agent 直接回答）**：约 2-5 秒，用户可看到逐字输出
- **单 Agent 任务（Planner + 执行）**：约 5-15 秒，取决于任务复杂度和 Agent 工具调用次数
- **多步计划任务**：约 15-60 秒或更长，每步顺序执行，事件实时推送
- **定时任务**：首次创建约 5-10 秒，后续执行按配置周期自动触发

> **注意**：上述耗时基于本地部署、默认模型配置的典型值。使用不同 LLM 提供商、不同模型规格（如更大的参数量）或网络部署时，耗时会有显著差异。LLM 推理（步骤 5、6、8）是主要的时间消耗环节。

---

## 9. 为什么这个架构适合多智能体

### 9.1 Agent 与编排完全解耦

编排层不需要知道每个 Agent 的实现细节，只需遵循 A2A 协议和事件契约。Agent 可以独立开发、独立部署、独立扩展。

### 9.2 流式体验

用户不必等待长任务全部结束，能够实时看到：

- 文本消息逐渐生成
- 工具调用的开始和结果
- 推理过程的展开
- 富 UI 组件的替换

### 9.3 可恢复

后台 producer + 会话持久化 + HITL 机制，使系统更适合真实任务流程：

- 前端断开后任务不丢失
- 用户确认后能从断点恢复
- 所有结果都被持久化以供回看

### 9.4 内部状态模型

ValueCell 的运行时使用两套状态枚举，分别管理"事件"和"内部状态"。

**TaskStatusEvent（事件层，对外暴露）**——定义在 `core/types.py`：

| 事件 | 触发时机 |
|------|----------|
| `TASK_STARTED` | TaskExecutor 开始执行任务时发出 |
| `TASK_COMPLETED` | Agent 调用 `streaming.done()` 或任务正常完成时发出 |
| `TASK_FAILED` | Agent 调用 `streaming.failed()` 或执行出错时发出 |
| `TASK_CANCELLED` | 任务被用户或系统取消时发出 |

**TaskStatus（内部状态，不对外暴露）**——定义在 `core/task/models.py`：

| 状态 | 说明 |
|------|------|
| `PENDING` | 任务已创建，等待执行 |
| `RUNNING` | 任务正在执行中 |
| `WAITING_INPUT` | 任务等待输入（代码中已定义但当前未被使用）。这个状态是为未来的执行阶段 HITL 恢复预留的，当前仅在规划阶段实现了 HITL。 |
| `COMPLETED` | 任务成功完成 |
| `FAILED` | 任务执行失败 |
| `CANCELLED` | 任务被取消 |

**ConversationStatus（会话状态）**——定义在 `core/conversation/models.py`：

| 状态 | 说明 |
|------|------|
| `ACTIVE` | 会话活跃，可正常交互 |
| `INACTIVE` | 会话不活跃 |
| `REQUIRE_USER_INPUT` | 会话暂停，等待用户输入（HITL 状态） |

### 9.5 Orchestrator 韧性设计细节

以下是与多智能体架构直接相关的韧性设计细节——正是这些机制保证了多个 Agent 长时间协作时系统的稳定性。

**前端断开时的行为**：Orchestrator 的 `process_user_input` 在 `asyncio.CancelledError`（消费者断开）时，仅将 `active` 标志设为 `False`，**不会取消后台 producer task**。这意味着：

- 长时间运行的策略任务不会因用户关闭浏览器而中断
- 后台任务产出的所有事件仍会被持久化到 Store
- 用户重新连接后可从 Store 恢复完整状态

**执行上下文过期**：Orchestrator 为每个 HITL 会话维护一个 `ExecutionContext`，存储在内存字典 `_execution_contexts` 中。上下文有 1 小时（`DEFAULT_CONTEXT_TIMEOUT_SECONDS = 3600`）的过期时间。超时后的恢复请求会被拒绝。系统会定期清理过期的上下文。

**HITL 恢复限制**：当前 HITL 恢复仅支持 `"planning"` 阶段。如果执行上下文的 `stage` 不是 `"planning"`，Orchestrator 会发出 `system_failed("Resuming execution stage is not yet supported.")` 错误。这意味着用户只能在规划阶段被中断后恢复，执行阶段的恢复尚不支持。

---

## 10. 责任边界：改哪一层

读懂架构后，一个更实际的问题是：**某类需求到底该改哪一层**。

| 需求类型 | 改哪一层 | 关键文件 |
|----------|----------|----------|
| 页面布局、路由、展示 | 前端 | `routes.ts`、`root.tsx`、`app/`、`renderer/` |
| 请求进不去、初始化异常 | API 层 | `server/main.py`、`server/api/app.py` |
| 对话逻辑、HITL 行为异常 | 编排层 | `python/valuecell/core/coordinate/orchestrator.py`、`python/valuecell/core/plan/service.py` |
| 消息碎片化、组件显示异常 | 事件层 | `core/event/router.py`、`core/event/buffer.py` |
| Agent 不工作、能力不对 | Agent 层 | `agents/<agent>/`、`configs/agent_cards/*.json` |
| 配置加载问题 | 配置层 | `utils/env.py`、`config/loader.py` |

---

## 11. 架构阅读建议

推荐阅读顺序：

```
1. frontend/src/lib/sse-client.ts       → 理解流式通信基础
2. python/valuecell/server/api/app.py   → 理解后端初始化
3. python/valuecell/core/types.py       → 理解事件类型契约
4. python/valuecell/core/coordinate/orchestrator.py  → 理解编排核心
5. python/valuecell/core/super_agent/   → 理解分流逻辑
6. python/valuecell/core/plan/          → 理解规划系统
7. python/valuecell/core/task/          → 理解任务执行
8. python/valuecell/core/event/         → 理解事件处理
9. python/valuecell/agents/             → 理解 Agent 实现
```

---

## 12. 架构决策记录

本节记录影响系统行为的几个关键架构决策及其影响。

### 12.1 为什么 Agent 监听独立端口

每个 Agent 作为独立 HTTP 服务运行在独立端口上（如 ResearchAgent 在 10004、NewsAgent 在 10005）。这种设计的理由：

- **进程隔离**：单个 Agent 崩溃不会影响其他 Agent
- **独立部署**：Agent 可以部署在不同机器上
- **独立重启**：更新某个 Agent 只需重启该进程

代价是增加了端口管理复杂度和进程间通信开销。

### 12.2 为什么使用 `asyncio.Queue` 而不是直接 SSE

Orchestrator 使用 `asyncio.Queue` 作为 Producer 和 Consumer 之间的桥梁，而不是让前端直接消费 Agent 事件流。原因：

1. **解耦生命周期**：前端断开后，后台任务不受影响
2. **中间处理**：事件在推送前经过 Router → Buffer → Service 三层处理
3. **持久化窗口**：事件进入 Queue 前已持久化，前端可以随时恢复

### 12.3 为什么数据库使用 SQLite 而非 PostgreSQL

SQLite 的选择基于以下考量：

- **桌面应用友好**：无需额外安装和配置数据库服务
- **零配置**：单个文件即可使用，符合"开箱即用"的设计目标
- **性能足够**：桌面应用场景下并发量有限，SQLite 完全胜任
- **数据可移植**：数据库文件可以直接复制备份

> **注意**：SQLite 的写操作是串行的（同一时间只允许一个写入者），在桌面应用场景下这不是问题，但如果未来需要多用户并发写入，需要迁移到支持并发写入的数据库。

如果未来需要支持多用户在线部署，可能需要迁移到 PostgreSQL。

### 12.4 为什么 Super Agent 不直接执行任务

Super Agent 只做 ANSWER 或 HANDOFF_TO_PLANNER 两种决策，不执行具体任务。这避免了：

- Super Agent 成为一个"上帝 Agent"，职责膨胀难以维护
- 分流逻辑与业务逻辑耦合，导致修改一处影响全局
- Super Agent 的 LLM 调用成为系统的性能瓶颈

### 12.5 架构决策速查表

下表汇总了影响系统行为的关键架构决策及其取舍，便于快速查阅。

| 决策点 | 选择 | 核心收益 | 付出的代价 | 影响范围 |
|--------|------|----------|-----------|----------|
| Agent 运行方式 | 独立端口、独立进程 | 进程隔离、独立部署重启 | 端口管理复杂度、IPC 开销 | Agent 层 |
| 流式通信机制 | `asyncio.Queue` + 后台 producer | 前端断开后任务继续、事件可持久化 | 实现复杂度高于直连 SSE | 编排层 |
| 持久化存储 | SQLite（单文件） | 零配置、桌面友好、数据可移植 | 写操作串行，不支持多用户并发写入 | 全局 |
| Super Agent 定位 | 仅做 ANSWER/HANDOFF 分流 | 职责清晰、避免耦合膨胀 | 多一次 LLM 调用（1-3 秒） | 编排层 |
| Planner 执行模式 | 标准规划 + 直通模式双轨 | 简单请求跳过规划降低延迟 | 需维护两套逻辑路径 | 编排层 |
| 事件处理 | Router → Buffer → Service 三层 | 职责分离、消息聚合、持久化窗口 | 链路较长、调试需跨三层追踪 | 事件层 |
| Agent 通信协议 | A2A（基于 HTTP） | Agent 可远程部署、框架无关 | HTTP 开销、需端口管理 | Agent 层 |
| Agent 构建框架 | Agno（v2.x） | LLM 封装、工具注册、流式推理开箱即用 | 框架依赖、版本锁定 | Agent 层 |
| 会话状态管理 | 内存 + SQLite 双层 | 快速访问 + 持久化恢复 | 需同步两层状态 | 编排层 |
| HITL 超时策略 | 1 小时硬过期 | 防止上下文无限堆积 | 超时后用户需重新发起请求 | 编排层 |

> **如何使用此表**：当你需要修改某层的行为时，先在此表中找到相关的决策点，理解当初选择的原因和代价，再决定是调整现有设计还是引入新机制。

---

## 13. 常见问题

### Q1：单个 Agent 崩溃会影响其他 Agent 吗？

不会。每个 Agent 运行在独立进程中，监听独立端口。ResearchAgent（端口 10004）崩溃不会影响 NewsAgent（端口 10005）或其他 Agent 的正常运行。

崩溃发生时，TaskExecutor 在调用该 Agent 时会捕获异常，将对应任务标记为 `FAILED`，并通过事件链路通知前端。同一计划中的其他任务不受影响。用户可以在 Agent 恢复后重新发起请求。

需要人工干预的场景是：如果崩溃的 Agent 是某个计划中顺序执行的步骤之一，后续步骤会因前置依赖失败而无法继续。此时 TaskExecutor 会在任务失败后停止该计划的执行。

### Q2：可以添加超过 4 个 Agent 吗？

可以。系统对 Agent 数量没有硬性上限。添加新 Agent 的步骤：

1. 在 `python/valuecell/agents/` 下实现 Agent 类（继承 `BaseAgent`，实现 `stream()` 方法）
2. 在 `python/configs/agent_cards/` 下创建对应的 Agent Card JSON 文件（设置 `enabled: true`、指定端口和 `local_agent_class`）
3. 在 `python/configs/agents/` 下创建 YAML 配置文件（指定模型和参数）
4. 系统启动时 `get_local_agent_cards()` 会自动扫描 Agent Cards 目录，发现并注册新 Agent

唯一需要注意的是端口分配：每个 Agent 需要独立的端口号，避免冲突。

### Q3：系统如何处理并发请求？

当前架构基于 `asyncio` 异步模型，每个用户请求由独立的 producer task 处理，通过各自的 `asyncio.Queue` 推送事件。不同请求之间不会互相阻塞。

但有两个限制需要了解：

- **SQLite 写入串行**：所有持久化操作共享同一个 SQLite 数据库文件，同一时刻只允许一个写入者。在桌面应用场景下这通常不构成瓶颈，但如果未来需要多用户在线部署，需要迁移到支持并发写入的数据库（如 PostgreSQL）。
- **Agent 单实例**：每个 Agent 当前以单进程运行，多个并发请求会排队等待同一 Agent 的响应。如果某个 Agent 是性能瓶颈，需要考虑多实例部署方案。

### Q4：TaskStatus（内部）和 TaskStatusEvent（外部）有什么关系和区别？

这是两个不同层面的概念，分别服务于不同的消费者：

- **TaskStatus**（定义在 `core/task/models.py`）：系统内部的任务状态枚举，包含 `PENDING`、`RUNNING`、`WAITING_INPUT`、`COMPLETED`、`FAILED`、`CANCELLED` 六个值。它由 TaskService 维护，用于持久化到数据库，记录任务的完整生命周期。前端不会直接看到这些状态值。
- **TaskStatusEvent**（定义在 `core/types.py`）：对外暴露的事件类型，包含 `TASK_STARTED`、`TASK_COMPLETED`、`TASK_FAILED`、`TASK_CANCELLED` 四个值。它作为流式事件通过 SSE 推送给前端，驱动 UI 更新（如任务进度指示器）。

两者的映射关系是：当内部 TaskStatus 发生状态转换时（例如从 `PENDING` 变为 `RUNNING`），系统会发出对应的 TaskStatusEvent（`TASK_STARTED`）。`WAITING_INPUT` 状态当前在 TaskStatus 中已定义但未使用，因此没有对应的 TaskStatusEvent。

简言之：**TaskStatus 是内部账本，TaskStatusEvent 是对外通知**。

### Q5：前端 SSE 连接断开后，正在执行的任务会怎样？

任务不会中断。Orchestrator 的设计明确区分了 consumer（前端 SSE 连接）和 producer（后台执行任务）的生命周期：

- 当 SSE 连接断开时（`asyncio.CancelledError`），Orchestrator 仅将 `active` 标志设为 `False`，**不会取消后台 producer task**。
- 后台任务继续执行，所有产出的事件仍会被持久化到 SQLite Store。
- 用户重新连接后，前端从 Store 中恢复完整状态，包括断线期间产生的所有消息和组件。

这一设计对于长时间运行的策略任务尤为重要——用户不会因为关闭浏览器窗口而丢失正在执行的量化策略。

---

## 14. 自测检查清单

阅读完本文后，用以下问题检验你的理解程度。每个问题对应文中的一个核心概念，如果你能不回看原文就回答大部分问题，说明你已经掌握了 ValueCell 的架构全貌。

1. **请求链路**：请默写出一条用户请求从前端到 Agent 再回到前端所经过的每一个组件，按顺序列出。你能说出每个组件的"输入是什么、输出是什么"吗？

2. **Orchestrator 的 producer 模式**：为什么 Orchestrator 使用后台 producer task 加 `asyncio.Queue` 的模式，而不是同步执行后直接返回？如果前端 SSE 连接在任务执行中途断开，后台任务会怎样？

3. **Super Agent 的职责边界**：Super Agent 只做哪两件事？为什么不直接让 Super Agent 执行具体任务？如果 Super Agent 初始化失败，系统会怎样？

4. **Planner 的两种模式**：标准规划模式和直通模式有什么区别？直通模式由什么条件触发？如果你在调试某个策略 Agent 时发现没有进入完整的 Planner 推理流程，可能的原因是什么？

5. **事件处理三层架构**：ResponseRouter、ResponseBuffer、EventResponseService 各自解决什么问题？为什么需要 ResponseBuffer 来分配稳定的 `item_id`，而不是直接转发原始事件？

6. **TaskStatus 与 TaskStatusEvent 的区别**：哪个是"内部账本"，哪个是"对外通知"？`WAITING_INPUT` 状态属于哪个？为什么它当前没有被使用？

7. **持久化的意义**：系统持久化了哪些数据（说出四种）？持久化支撑了哪三个关键能力？如果去掉持久化，系统的哪些行为会不可用？

8. **Agent 发现与注册**：当你添加一个新的 Agent 时，需要创建哪三个文件？系统是如何自动发现新 Agent 的？端口号的作用是什么？

9. **SQLite 的取舍**：为什么 ValueCell 选择 SQLite 而不是 PostgreSQL？在什么场景下这个选择会成为瓶颈？

10. **改哪一层**：如果用户反馈"工具调用的结果没有在界面上正确显示"，你应该优先检查哪个目录下的文件？如果反馈"页面布局和导航有问题"呢？

> **评分参考**：能回答 8 个以上——你已经具备了独立阅读和修改 ValueCell 源码的架构认知。能回答 5-7 个——建议回顾对应的章节和关键文件。少于 5 个——建议从 §11 的阅读顺序开始，逐个文件精读。

---

## 15. 继续阅读

- 想知道从目录与文件入口怎么读：看 [source-guide.md](./source-guide.md)
- 想自己新增 Agent：看 [agent-extension.md](./agent-extension.md)
- 想先把系统跑起来：看 [getting-started.md](./getting-started.md)
- 想深入配置系统：看 [configuration.md](./configuration.md)
