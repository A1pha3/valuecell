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
| `/home` | 首页 |
| `/market` | 市场数据 |
| `/agent/:agentName` | Agent 对话 |
| `/agent/:agentName/config` | Agent 配置 |
| `/setting` | 设置总览 |
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
3. 初始化数据库
4. 初始化市场数据适配器
5. 注册 API 路由（统一前缀 `/api/v1`）

### 3.2 API 层不是"业务逻辑终点"

分清两层职责：

- `server/`：服务化、路由化、启动与初始化
- `core/`：对话编排、任务执行、事件持久化

API 层是入口，真正的运行时在 `core/`。

---

## 4. 编排运行时层

`python/valuecell/core/` 是系统真正的核心。它是一个**多智能体运行时**，不是简单的工具函数集合。

### 4.1 Orchestrator：总调度中心

**核心文件**：`coordinate/orchestrator.py`

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

Super Agent 不做业务，只做第一层判断：

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

| Agent | 类名 | 端口 |
|-------|------|------|
| 研究 Agent | `ResearchAgent` | 10004 |
| 新闻 Agent | `NewsAgent` | 10005 |
| 提示策略 Agent | `PromptBasedStrategyAgent` | 10006 |
| 网格策略 Agent | `GridStrategyAgent` | 10007 |

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

---

## 6. 事件处理链路

这是理解 ValueCell 最关键的部分。

### 6.1 流式事件类型

在 `valuecell.core.types` 中定义的事件类型：

| 事件 | 含义 |
|------|------|
| `message_chunk` | 文本片段 |
| `tool_call_started` | 工具调用开始 |
| `tool_call_completed` | 工具调用完成 |
| `reasoning_started` | 推理开始 |
| `reasoning` | 推理内容 |
| `reasoning_completed` | 推理结束 |
| `component_generator` | 富 UI 组件（图表、报告等） |
| `task_started` | 任务开始 |
| `task_completed` | 任务完成 |
| `task_failed` | 任务失败 |
| `plan_require_user_input` | 需要用户确认 |
| `done` | 响应结束 |

这些事件不是装饰性的——它们**决定前端渲染什么**。

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

`conversation_store.py` 和 `item_store.py` 提供：

- 内存和 SQLite 两种后端
- 过滤和分页支持
- 快速"恢复到上次"的能力

### 7.3 为什么需要持久化

持久化支撑了三个关键能力：

1. **历史记录**：用户可以回看过去的对话
2. **会话恢复**：前端断开重连后，从 Store 中恢复状态
3. **HITL 恢复**：用户确认后，Orchestrator 能从断点继续

---

## 8. 一条请求的完整旅程

把一次典型请求理解为以下步骤：

```
1. 用户在前端输入查询
     ↓
2. 前端通过 SSE Client 发送 POST 请求
     ↓
3. API 层接收，转发给 Orchestrator
     ↓
4. Orchestrator 确保会话存在，启动后台 producer
     ↓
5. Super Agent 分析，决定：
   ├─ ANSWER → 直接产出消息 → 事件处理 → 前端渲染 → done
   └─ HANDOFF → Planner 分析
     ↓
6. Planner 生成计划（如需确认，进入 HITL）
     ↓
7. TaskExecutor 按计划调用远程 Agent（A2A）
     ↓
8. Agent 产出流式事件
     ↓
9. ResponseRouter 语义分类 → ResponseBuffer 分配 ID/聚合
     ↓
10. EventResponseService 持久化到 Store
     ↓
11. 前端收到事件 → 8 个渲染器按类型渲染
     ↓
12. done 事件 → 前端标记完成
```

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

---

## 10. 责任边界：改哪一层

读懂架构后，一个更实际的问题是：**某类需求到底该改哪一层**。

| 需求类型 | 改哪一层 | 关键文件 |
|----------|----------|----------|
| 页面布局、路由、展示 | 前端 | `routes.ts`、`root.tsx`、`app/`、`renderer/` |
| 请求进不去、初始化异常 | API 层 | `server/main.py`、`server/api/app.py` |
| 对话逻辑、HITL 行为异常 | 编排层 | `core/coordinate/orchestrator.py`、`core/plan/service.py` |
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

## 12. 继续阅读

- 想知道从目录与文件入口怎么读：看 [source-guide.md](./source-guide.md)
- 想自己新增 Agent：看 [agent-extension.md](./agent-extension.md)
- 想先把系统跑起来：看 [getting-started.md](./getting-started.md)
- 想深入配置系统：看 [configuration.md](./configuration.md)
