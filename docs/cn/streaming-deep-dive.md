# ValueCell 流式事件系统深度解析

> **前置知识**：本文假设你已阅读 [architecture.md](./architecture.md) 的 §6 节，对事件处理链路有基本了解。本文是事件系统的权威参考，其他文档中的事件相关内容均引用自本文。

## 学习目标

阅读本文后，你将能够：

- 解释为什么 ValueCell 选择流式事件架构而非"请求-响应"模式
- 列出全部 5 类事件及其所属的枚举类型和触发场景
- 画出从 Agent 产出事件到前端渲染的完整处理管线
- 理解 `ResponseBuffer` 如何通过稳定 `item_id` 实现消息的"逐步生长"
- 区分"立即事件"和"可缓冲事件"并说明各自的持久化策略
- 描述前端自定义 SSE Client 的设计决策与实现细节
- 说出 8 个渲染器的名称、对应事件类型和触发场景
- 追踪 Agno 框架事件到 ValueCell 事件的转换链路
- 列出全部 7 种 ComponentType 及其用途
- 定位并调试常见的流式事件问题

---

## 1. 为什么需要流式事件

### 1.1 用户体感差异

传统"请求-响应"模式下，用户发出查询后必须等待 Agent 完成所有推理、工具调用和结果汇总，才能一次性看到最终答案。对于金融研究这类复杂任务，单次请求可能需要 30 秒甚至更长时间，期间用户看到的是一个空白界面。

流式事件架构改变了这个体验：

```
传统模式：
用户发送 ──────────────────────────────────── 收到响应（30 秒后）
         └──── 空白等待 ────┘

流式模式：
用户发送 ── 推理中 ── 工具调用 ── 推理中 ── 文本生成 ── 完成
          (1s)     (5s)      (8s)     (15s)     (30s)
```

### 1.2 不仅仅是"打字机效果"

ValueCell 的流式系统承载的不只是逐字输出。它同时传递：

- **文本消息**（message_chunk）：LLM 生成的回复片段
- **推理过程**（reasoning）：Agent 的思考链，让用户理解"为什么这么做"
- **工具调用**（tool_call_started / tool_call_completed）：正在查询哪些数据源
- **富 UI 组件**（component_generator）：研究报告、折线图、定时任务面板等
- **任务状态**（task_started / task_completed 等）：多步骤计划的执行进度
- **系统事件**（conversation_started / done 等）：会话生命周期信号

> **核心认知**：流式事件是 ValueCell 前后端通信的统一协议，而不是一个展示层的装饰。

### 1.3 支持可恢复性

流式事件与持久化结合，使系统具备"断线不丢任务"的能力。后台 producer 模式意味着即使前端 SSE 连接断开，后台任务继续执行，所有事件被持久化到 Store。前端重连后可以从 Store 恢复完整状态。

---

## 2. 完整事件分类

所有事件类型定义在 `python/valuecell/core/types.py` 中，按语义分为 5 个枚举类。

### 2.1 StreamResponseEvent — 流式响应事件

这是最核心的事件类别，承载 Agent 在流式处理过程中产出的各类增量信息。

| 事件常量 | 字符串值 | 说明 |
|----------|----------|------|
| `MESSAGE_CHUNK` | `"message_chunk"` | 文本消息片段，LLM 逐步生成的回复内容 |
| `TOOL_CALL_STARTED` | `"tool_call_started"` | 工具调用开始，携带 `tool_call_id` 和 `tool_name` |
| `TOOL_CALL_COMPLETED` | `"tool_call_completed"` | 工具调用完成，额外携带 `tool_result` |
| `REASONING_STARTED` | `"reasoning_started"` | 推理过程开始标记 |
| `REASONING` | `"reasoning"` | 推理内容片段 |
| `REASONING_COMPLETED` | `"reasoning_completed"` | 推理过程结束标记 |

触发场景示例：

```
用户: "分析特斯拉最新财报"
  → REASONING_STARTED          "开始分析..."
  → REASONING                  "需要先获取 10-K 报告..."
  → TOOL_CALL_STARTED          sec_filing_search(TSLA, 10-K)
  → TOOL_CALL_COMPLETED        [搜索结果...]
  → MESSAGE_CHUNK              "根据最新 10-K 报告..."
  → MESSAGE_CHUNK              "特斯拉 2024 年营收..."
  → MESSAGE_CHUNK              "同比增长 18%..."
```

### 2.2 CommonResponseEvent — 通用响应事件

跨响应类型共享的事件，当前只有一种。

| 事件常量 | 字符串值 | 说明 |
|----------|----------|------|
| `COMPONENT_GENERATOR` | `"component_generator"` | 触发前端渲染一个富 UI 组件 |

`COMPONENT_GENERATOR` 既出现在 `StreamResponse` 中（用户主动请求的流式输出），也出现在通知管道中（Agent 主动推送）。组件类型通过 metadata 中的 `component_type` 字段指定，见第 8 节。

### 2.3 SystemResponseEvent — 系统响应事件

由编排层（Orchestrator）产出的系统级信号，不涉及具体内容，主要用于标记会话生命周期。

| 事件常量 | 字符串值 | 说明 |
|----------|----------|------|
| `CONVERSATION_STARTED` | `"conversation_started"` | 新会话已建立 |
| `THREAD_STARTED` | `"thread_started"` | 新消息线程已开始，携带用户查询内容 |
| `PLAN_REQUIRE_USER_INPUT` | `"plan_require_user_input"` | 规划器需要用户确认（HITL 触发） |
| `PLAN_FAILED` | `"plan_failed"` | 规划失败 |
| `SYSTEM_FAILED` | `"system_failed"` | 系统级错误 |
| `DONE` | `"done"` | 当前线程/会话响应结束 |

时序示例：

```
CONVERSATION_STARTED     ← 会话建立
THREAD_STARTED           ← 线程开始，携带用户原始查询
  TASK_STARTED           ← 任务开始
    MESSAGE_CHUNK...     ← Agent 产出内容
  TASK_COMPLETED         ← 任务完成
DONE                     ← 线程结束
```

### 2.4 TaskStatusEvent — 任务状态事件

与 Planner 产出的任务执行步骤关联，反映单个任务的生命周期。

| 事件常量 | 字符串值 | 说明 |
|----------|----------|------|
| `TASK_STARTED` | `"task_started"` | 任务已开始执行 |
| `TASK_COMPLETED` | `"task_completed"` | 任务成功完成 |
| `TASK_FAILED` | `"task_failed"` | 任务执行失败 |
| `TASK_CANCELLED` | `"task_cancelled"` | 任务被取消 |

### 2.5 NotifyResponseEvent — 通知响应事件

由 Agent 的 `notify()` 方法产出，用于 Agent 主动向用户推送消息。

| 事件常量 | 字符串值 | 说明 |
|----------|----------|------|
| `MESSAGE` | `"message"` | Agent 主动推送的通知消息 |

通知事件与流式事件的关键区别：流式事件由 `stream()` 方法产出（响应用户请求），通知事件由 `notify()` 方法产出（Agent 主动推送）。两者共享 `component_generator` 能力。

### 2.6 事件与响应类型的绑定关系

每种事件类型只能出现在特定的响应模型中：

```
StreamResponse.event  ← StreamResponseEvent | TaskStatusEvent | CommonResponseEvent
NotifyResponse.event  ← NotifyResponseEvent | TaskStatusEvent | CommonResponseEvent
```

系统级事件（`SystemResponseEvent`）由编排层直接构造为 `BaseResponse` 子类（如 `ConversationStartedResponse`、`ThreadStartedResponse`、`DoneResponse` 等），不经过 Agent 的 `stream()` / `notify()` 管道。

---

## 3. 事件处理管线

从 Agent 产出原始事件到前端消费，事件经过三层处理。

### 3.1 管线全景

```
Agent stream() / notify()
     │
     │  StreamResponse / NotifyResponse（原始事件）
     ↓
┌──────────────────────────────────────────────────┐
│  ResponseRouter (core/event/router.py)            │
│  ── 语义分类：工具调用 / 推理 / 组件 / 消息 / 失败  │
│  ── 将底层 TaskStatusUpdateEvent 转为 BaseResponse │
│  ── 失败时产出 SideEffect（如标记任务失败）          │
└──────────────────────────────────────────────────┘
     │
     │  RouteResult { responses, done, side_effects }
     ↓
┌──────────────────────────────────────────────────┐
│  EventResponseService (core/event/service.py)     │
│  ── 统一门面：annotate → ingest → persist          │
│  ── 调用 ResponseFactory 构造响应                   │
│  ── 调用 ResponseBuffer 分配 ID / 聚合分片          │
│  ── 调用 ConversationService 持久化到 Store        │
└──────────────────────────────────────────────────┘
     │
     │  BaseResponse（带稳定 item_id 的最终响应）
     ↓
  持久化到 SQLite ──→ 推入 asyncio.Queue ──→ 前端 SSE 消费
```

### 3.2 ResponseRouter：语义分类器

**文件**：`python/valuecell/core/event/router.py`

`handle_status_update()` 函数接收底层 A2A 的 `TaskStatusUpdateEvent`，根据 `event.metadata` 中的 `response_event` 字段进行语义分流：

```
TaskStatusUpdateEvent
  │
  ├── state == failed
  │     → 产出 task_failed 响应
  │     → 附带 SideEffect(FAIL_TASK)
  │
  ├── response_event ∈ { TOOL_CALL_STARTED, TOOL_CALL_COMPLETED }
  │     → 调用 factory.tool_call()
  │
  ├── response_event ∈ { REASONING_STARTED, REASONING, REASONING_COMPLETED }
  │     → 调用 factory.reasoning()
  │
  ├── response_event == COMPONENT_GENERATOR
  │     → 调用 factory.component_generator()
  │
  ├── response_event ∈ { MESSAGE_CHUNK, MESSAGE }
  │     → 调用 factory.message_response_general()
  │
  └── 其他
        → 静默忽略（如 submitted / completed 状态）
```

`RouteResult` 数据结构：

```python
@dataclass
class RouteResult:
    responses: List[BaseResponse]    # 要发出的响应列表
    done: bool = False               # 是否结束处理
    side_effects: List[SideEffect]   # 要求编排层执行的副作用
```

`SideEffect` 当前只有一种：`FAIL_TASK`（标记任务失败）。这允许路由层在检测到不可恢复错误时，请求编排层执行状态变更。

### 3.3 EventResponseService：统一门面

**文件**：`python/valuecell/core/event/service.py`

`EventResponseService` 是事件处理的唯一入口，组合了三个组件：

```python
class EventResponseService:
    _conversation_service    # 持久化层
    _factory                 # ResponseFactory — 构造响应
    _buffer                  # ResponseBuffer — 分配 ID / 聚合
```

核心方法调用链：

```
emit(response)
  ├── _buffer.annotate(response)   # 为可缓冲事件分配稳定 item_id
  ├── _buffer.ingest(response)     # 聚合或直接产出 SaveItem 列表
  └── _persist_items(items)        # 逐条调用 ConversationService.add_item()
```

`emit_many()` 是批量版本，按顺序对每个响应调用 `emit()`。`flush_task_response()` 在任务结束时强制刷出该任务上下文的所有缓冲区。

---

## 4. 稳定 ID 与消息聚合

### 4.1 问题：为什么需要稳定 ID

流式场景下，一条完整的消息被拆成多个 `message_chunk` 事件。如果每个 chunk 使用不同的 ID，前端无法将它们组装为一条消息。`ResponseBuffer` 通过为属于同一段落（paragraph）的所有 chunk 分配相同的 `item_id` 来解决这个问题。

### 4.2 BufferKey 与 BufferEntry

缓冲区使用四元组作为键：

```python
BufferKey = (conversation_id, thread_id, task_id, event)
```

同一个键下的 chunk 被聚合到同一个 `BufferEntry` 中：

```python
class BufferEntry:
    parts: List[str]        # 累积的文本片段
    item_id: str            # 稳定段落 ID（一旦分配不再改变）
    last_updated: float     # 最后更新时间
    role: Optional[Role]    # 消息角色
    agent_name: Optional[str]  # Agent 名称
```

### 4.3 两类事件：立即事件 vs 可缓冲事件

**分类判断标准**：如果一个事件携带的是**自包含的完整信息**（如工具调用结果、UI 组件），它就是立即事件——可以直接持久化，不需要与其他事件聚合。如果一个事件携带的是**增量文本片段**（如 LLM 逐字输出的内容），它就是可缓冲事件——需要与同一上下文中的其他片段聚合为一整条消息。

注意：`TOOL_CALL_STARTED` 不在立即事件列表中，因为它与 `TOOL_CALL_COMPLETED` 属于同一次工具调用的两个阶段——只有 `TOOL_CALL_COMPLETED` 携带完整结果，需要立即持久化并触发 flush。`TOOL_CALL_STARTED` 作为一般消息事件处理。

**立即事件**（`_immediate_events`）：

```python
{
    StreamResponseEvent.TOOL_CALL_COMPLETED,
    CommonResponseEvent.COMPONENT_GENERATOR,
    NotifyResponseEvent.MESSAGE,
    SystemResponseEvent.PLAN_REQUIRE_USER_INPUT,
    SystemResponseEvent.THREAD_STARTED,
}
```

立即事件直接持久化，同时触发同一上下文中所有缓冲区的 flush。

**可缓冲事件**（`_buffered_events`）：

```python
{
    StreamResponseEvent.MESSAGE_CHUNK,
    StreamResponseEvent.REASONING,
}
```

可缓冲事件在 `BufferEntry` 中累积文本，每次 chunk 到达时产出当前聚合快照的 `SaveItem`（upsert 语义）。

### 4.4 处理流程

```
                        ┌─────────────────┐
                        │ ResponseBuffer  │
                        └────────┬────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          ↓                      ↓                      ↓
    立即事件到达          可缓冲事件到达          其他事件
          │                      │                  （忽略）
          ↓                      ↓
  1. flush 同上下文      1. 查找/创建 BufferEntry
     的缓冲区            2. append(text)
  2. 持久化立即事件      3. snapshot_payload()
                          4. 产出 upsert SaveItem
```

### 4.5 annotate 与 ingest 的两阶段设计

`annotate()` 在事件进入缓冲区之前调用，为可缓冲事件的 `response.data.item_id` 赋值。这确保了：

1. 前端收到的每个 chunk 都带有相同的 `item_id`
2. 持久化层的 `item_id` 与前端看到的 `item_id` 一致

`ingest()` 执行实际的聚合和持久化逻辑。

### 4.6 完整示例

以 Agent 产出 3 个 `message_chunk` 后跟一个 `component_generator` 为例：

```
时间线：
  t1: MESSAGE_CHUNK "根据分析"
  t2: MESSAGE_CHUNK "特斯拉营收"
  t3: MESSAGE_CHUNK "同比增长18%"
  t4: COMPONENT_GENERATOR (report 组件)

缓冲区状态：
  t1 后: BufferEntry(item_id="item-abc", parts=["根据分析"])
         → 持久化 SaveItem(item_id="item-abc", content="根据分析")

  t2 后: BufferEntry(item_id="item-abc", parts=["根据分析", "特斯拉营收"])
         → 持久化 SaveItem(item_id="item-abc", content="根据分析特斯拉营收")

  t3 后: BufferEntry(item_id="item-abc", parts=["根据分析", "特斯拉营收", "同比增长18%"])
         → 持久化 SaveItem(item_id="item-abc", content="根据分析特斯拉营收同比增长18%")

  t4: COMPONENT_GENERATOR 是立即事件
      → 先 flush 缓冲区（最终版本已包含）
      → 再持久化组件事件

前端效果：
  "根据分析" → "根据分析特斯拉营收" → "根据分析特斯拉营收同比增长18%" → [报告组件出现]
  （同一条消息在同一个 item_id 下"逐步生长"）
```

---

## 5. 前端 SSE 实现

### 5.1 设计决策：为什么不用原生 EventSource

原生 `EventSource` 有两个关键限制：

1. **只支持 GET 请求**：无法在请求体中传递对话内容
2. **不支持自定义 Header**：无法携带认证信息

ValueCell 需要在请求体中传递当前对话的消息序列，因此必须自行实现 SSE 客户端。

### 5.2 实现架构

**文件**：`frontend/src/lib/sse-client.ts`

```
SSEClient
  │
  ├── connect(body)          ← 建立 SSE 连接，body 为 JSON 字符串
  │     └── startConnection()
  │           ├── fetch(url, { method: "POST", body, ... })
  │           └── readStream(response.body)
  │                 ├── TextDecoder 解码字节流
  │                 ├── 按 "\n\n" 拆分事件块
  │                 └── processEvent(block)
  │                       ├── 提取 "data:" 行
  │                       └── best-effort-json-parser 解析 JSON
  │
  ├── close()                ← 主动关闭连接
  ├── destroy()              ← 关闭连接并清理 handler
  └── state                  ← 当前连接状态 (CONNECTING / OPEN / CLOSED)
```

### 5.3 SSE 事件解析

后端返回的 SSE 格式遵循标准规范：

```
data: {"event":"message_chunk","data":{"conversation_id":"conv-xxx",...}}

data: {"event":"tool_call_started","data":{"conversation_id":"conv-xxx",...}}

```

解析步骤：

1. `readStream()` 将字节流追加到 `buffer` 字符串
2. 按 `\n\n` 分割取出完整事件块
3. `processEvent()` 按行拆分，提取 `data:` 开头的行
4. 使用 `best-effort-json-parser`（而非原生 `JSON.parse`）解析，以处理不完整 JSON
5. 调用 `onData` 回调传递解析后的对象

### 5.4 超时与状态管理

```typescript
enum SSEReadyState {
  CONNECTING = 0,   // 正在建立连接
  OPEN = 1,         // 连接已建立，正在读取流
  CLOSED = 2,       // 连接已关闭
}
```

- 默认握手超时：**30 秒**（可通过 `timeout` 选项覆盖）
- 超时后通过 `AbortController` 取消请求
- 连接状态变化时触发 `onStateChange` 回调
- 防止重复连接：`CONNECTING` 或 `OPEN` 状态下调用 `connect()` 无效

### 5.5 错误处理策略

```
错误类型              处理方式
───────────────────────────────────────────
HTTP 错误             触发 onError，关闭连接
握手超时              触发 onError（带 "Handshake timeout" 消息）
Abort（被取代）       触发 onClose（视为正常关闭）
JSON 解析失败         console.warn 记录，不触发连接级错误
网络错误              触发 onError，关闭连接
```

> **设计亮点**：JSON 解析失败不会导致连接断开。这是因为流式场景下 LLM 输出可能跨越多个 TCP 包，单个事件块可能不完整。

---

## 6. 渲染器系统

### 6.1 渲染器注册

**文件**：`frontend/src/components/valuecell/renderer/index.tsx`

```typescript
export { default as ChatConversationRenderer } from "./chat-conversation-renderer";
export { default as MarkdownRenderer } from "./markdown-renderer";
export { default as ReasoningRenderer } from "./reasoning-renderer";
export { default as ReportRenderer } from "./report-renderer";
export { default as ScheduledTaskControllerRenderer } from "./scheduled-task-controller-renderer";
export { default as ScheduledTaskRenderer } from "./scheduled-task-renderer";
export { default as ToolCallRenderer } from "./tool-call-renderer";
export { default as UnknownRenderer } from "./unknown-renderer";
```

### 6.2 8 个渲染器详解

| 渲染器 | 文件 | 处理的事件 | 说明 |
|--------|------|-----------|------|
| `ChatConversationRenderer` | `chat-conversation-renderer.tsx` | 全部事件 | 对话容器，管理消息列表和滚动行为 |
| `MarkdownRenderer` | `markdown-renderer.tsx` | `message_chunk` | 渲染 Markdown 格式的文本消息 |
| `ReasoningRenderer` | `reasoning-renderer.tsx` | `reasoning_started` / `reasoning` / `reasoning_completed` | 展示 Agent 推理过程，通常可折叠 |
| `ReportRenderer` | `report-renderer.tsx` | `component_generator`（`report` 类型） | 渲染研究报告，支持标题、Markdown 内容和外部链接 |
| `ScheduledTaskControllerRenderer` | `scheduled-task-controller-renderer.tsx` | `component_generator`（`scheduled_task_controller` 类型） | 定时任务控制面板（启动/停止） |
| `ScheduledTaskRenderer` | `scheduled-task-renderer.tsx` | `component_generator`（`scheduled_task_result` 类型） | 定时任务执行结果展示 |
| `ToolCallRenderer` | `tool-call-renderer.tsx` | `tool_call_started` / `tool_call_completed` | 展示工具调用名称、参数和返回结果 |
| `UnknownRenderer` | `unknown-renderer.tsx` | 未识别的事件类型 | 兜底渲染器，防止未知事件导致界面空白 |

### 6.3 渲染器选择流程

```
SSE 事件到达
  │
  ├── event == "message_chunk"
  │     → MarkdownRenderer
  │
  ├── event == "tool_call_started" || "tool_call_completed"
  │     → ToolCallRenderer
  │
  ├── event == "reasoning_started" || "reasoning" || "reasoning_completed"
  │     → ReasoningRenderer
  │
  ├── event == "component_generator"
  │     ├── component_type == "report"
  │     │     → ReportRenderer
  │     ├── component_type == "scheduled_task_controller"
  │     │     → ScheduledTaskControllerRenderer
  │     ├── component_type == "scheduled_task_result"
  │     │     → ScheduledTaskRenderer
  │     └── 其他
  │           → UnknownRenderer（或对应的专用渲染器）
  │
  └── 其他
        → UnknownRenderer
```

所有渲染器被包裹在 `ChatConversationRenderer` 中，后者负责管理消息列表的渲染顺序、自动滚动和新消息动画。

### 6.4 组件替换行为

`component_generator` 事件支持通过 `component_id` 实现组件替换。当 metadata 中包含 `component_id` 时，该值会覆盖自动生成的 `item_id`，前端可以用相同的 `component_id` 替换已有组件而不是新增一个。

这个机制在定时任务场景中特别有用：任务控制面板可以原地更新为任务结果。

---

## 7. Agno 事件转换

### 7.1 Agno 框架的角色

ValueCell 使用 [Agno](https://docs.agno.com)（v2.x）作为 Agent 构建框架。Agno 的 `Agent.arun(query, stream=True)` 返回一个异步事件流，其中包含 Agno 自有的事件类型。ValueCell Agent 需要将这些事件转换为 ValueCell 统一的 `StreamResponse`。

### 7.2 转换映射

当前有 3 种 Agno 事件被显式处理：

| Agno 事件 | ValueCell 转换 | 说明 |
|-----------|---------------|------|
| `RunContent` | `streaming.message_chunk(event.content)` | 文本内容片段 |
| `ToolCallStarted` | `streaming.tool_call_started(tool_call_id, tool_name)` | 工具调用开始 |
| `ToolCallCompleted` | `streaming.tool_call_completed(tool_result, tool_call_id, tool_name)` | 工具调用完成 |

### 7.3 实际代码示例

以 `NewsAgent` 为例（`python/valuecell/agents/news_agent/core.py`）：

```python
async def stream(self, query, conversation_id, task_id, dependencies=None):
    response_stream = self.knowledge_news_agent.arun(
        query,
        stream=True,
        stream_intermediate_steps=True,    # 启用工具调用事件
        session_id=conversation_id,
    )
    async for event in response_stream:
        if event.event == "RunContent":
            yield streaming.message_chunk(event.content)
        elif event.event == "ToolCallStarted":
            yield streaming.tool_call_started(
                event.tool.tool_call_id, event.tool.tool_name
            )
        elif event.event == "ToolCallCompleted":
            yield streaming.tool_call_completed(
                event.tool.result, event.tool.tool_call_id, event.tool.tool_name
            )
    yield streaming.done()
```

### 7.4 streaming 命名空间

`streaming` 是 `_StreamResponseNamespace` 的单例实例（定义在 `core/agent/responses.py`），提供以下工厂方法：

| 方法 | 返回类型 | 事件枚举 | 说明 |
|------|---------|---------|------|
| `message_chunk(content: str)` | `StreamResponse` | `StreamResponseEvent.MESSAGE_CHUNK` | 文本消息片段 |
| `tool_call_started(tool_call_id: str, tool_name: str)` | `StreamResponse` | `StreamResponseEvent.TOOL_CALL_STARTED` | 工具调用开始 |
| `tool_call_completed(tool_result: str, tool_call_id: str, tool_name: str)` | `StreamResponse` | `StreamResponseEvent.TOOL_CALL_COMPLETED` | 工具调用完成 |
| `component_generator(content: str, component_type: str, component_id: Optional[str] = None)` | `StreamResponse` | `CommonResponseEvent.COMPONENT_GENERATOR` | 富 UI 组件 |
| `done(content: Optional[str] = None)` | `StreamResponse` | `TaskStatusEvent.TASK_COMPLETED` | 任务成功完成 |
| `failed(content: Optional[str] = None)` | `StreamResponse` | `TaskStatusEvent.TASK_FAILED` | 任务失败 |

> **注意**：`streaming.done()` 创建的是 `TaskStatusEvent.TASK_COMPLETED`（任务级完成），而 `SystemResponseEvent.DONE`（会话级结束）由编排层在所有任务完成后自动发出。Agent 只需调用 `streaming.done()` 标记自己的任务完成即可。

对应的 `notification` 命名空间（`_NotifyResponseNamespace` 实例）提供通知场景的工厂方法：

| 方法 | 返回类型 | 事件枚举 | 说明 |
|------|---------|---------|------|
| `message(content: str)` | `NotifyResponse` | `NotifyResponseEvent.MESSAGE` | 文本通知 |
| `component_generator(content: str, component_type: str, component_id: Optional[str] = None)` | `StreamResponse` | `CommonResponseEvent.COMPONENT_GENERATOR` | 组件通知 |
| `done(content: Optional[str] = None)` | `NotifyResponse` | `TaskStatusEvent.TASK_COMPLETED` | 通知完成 |
| `failed(content: Optional[str] = None)` | `NotifyResponse` | `TaskStatusEvent.TASK_FAILED` | 通知失败 |

> **为什么 `notification.component_generator()` 返回 `StreamResponse` 而非 `NotifyResponse`？** 因为 `component_generator` 事件（`CommonResponseEvent.COMPONENT_GENERATOR`）是一个跨管道共享的通用事件——它既出现在 `stream()` 管道中也出现在 `notify()` 管道中。底层事件模型只有一种 `COMPONENT_GENERATOR`，因此在 `notification` 命名空间中，这个方法返回的是 `StreamResponse`（包含通用事件字段），而不是 `NotifyResponse`。这是一个有意的设计选择，使得组件事件的处理逻辑在两层保持一致。

此外，`EventPredicates` 类提供事件类型判断的静态方法：`is_task_completed()`、`is_task_failed()`、`is_tool_call()`、`is_reasoning()`、`is_message()`。

### 7.5 转换链路全图

```
Agno Agent.arun(stream=True)
     │
     │  Agno 事件: RunContent / ToolCallStarted / ToolCallCompleted
     ↓
ValueCell Agent.stream()
  └── streaming.message_chunk() / streaming.tool_call_started() / ...
     │
     │  StreamResponse / NotifyResponse
     ↓
AgentWrapper (A2A 协议层)
  └── 包装为 TaskStatusUpdateEvent，事件信息存入 metadata
     │
     │  TaskStatusUpdateEvent
     ↓
ResponseRouter.handle_status_update()
  └── 语义分类，调用 ResponseFactory 构造 BaseResponse
     │
     │  BaseResponse
     ↓
EventResponseService.emit()
  └── annotate → ingest → persist
     │
     │  最终 BaseResponse（带 item_id）
     ↓
前端 SSE 消费 → 渲染器渲染
```

---

## 8. 组件类型系统

### 8.1 ComponentType 枚举

**定义**：`python/valuecell/core/types.py` 中的 `ComponentType` 枚举。

| 组件类型 | 字符串值 | 说明 | 前端渲染器 |
|----------|----------|------|-----------|
| `REPORT` | `"report"` | 研究报告，携带标题、Markdown 内容、可选 URL 和创建时间 | `ReportRenderer` |
| `PROFILE` | `"profile"` | 公司或股票档案 | 对应的专用渲染 |
| `SUBAGENT_CONVERSATION` | `"subagent_conversation"` | 子 Agent 会话边界标记（`start` / `end` 阶段） | 会话容器内嵌展示 |
| `SCHEDULED_TASK_CONTROLLER` | `"scheduled_task_controller"` | 定时任务控制面板，展示任务 ID、标题和当前状态 | `ScheduledTaskControllerRenderer` |
| `SCHEDULED_TASK_RESULT` | `"scheduled_task_result"` | 定时任务执行结果，携带任务信息和执行输出 | `ScheduledTaskRenderer` |
| `FILTERED_LINE_CHART` | `"filtered_line_chart"` | 可交互的折线图，携带表格化数据 | 图表渲染组件 |
| `FILTERED_CARD_PUSH_NOTIFICATION` | `"filtered_card_push_notification"` | 带筛选选项的通知卡片 | 卡片通知组件 |

### 8.2 组件数据的 Payload 结构

每种组件类型有对应的 Payload 模型：

**ReportComponentData**：

```python
class ReportComponentData(BaseModel):
    title: str          # 报告标题
    data: str           # 报告内容（Markdown）
    url: Optional[str]  # 外部链接
    create_time: str    # 创建时间（UTC, YYYY-MM-DD HH:MM:SS）
```

**FilteredLineChartComponentData**：

```python
class FilteredLineChartComponentData(BaseModel):
    title: str       # 图表标题
    data: str        # 表格数据，格式: [['x_axis_name', 'v1', 'v2'], ['t1', v1, v2], ...]
    create_time: str  # 创建时间
```

**FilteredCardPushNotificationComponentData**：

```python
class FilteredCardPushNotificationComponentData(BaseModel):
    title: str        # 通知标题
    data: str         # 通知数据
    filters: List[str] # 筛选条件列表
    table_title: str   # 表格标题
    create_time: str   # 创建时间
```

**ScheduledTaskComponentContent**：

```python
class ScheduledTaskComponentContent(BaseModel):
    task_id: Optional[str]      # 定时任务 ID
    task_title: Optional[str]   # 任务标题
    task_status: Optional[str]  # 任务状态
    result: Optional[str]       # 执行结果
    create_time: Optional[str]  # 创建时间
```

### 8.3 组件的构造

`ResponseFactory` 提供了便捷方法来构造组件事件：

```python
# 通用组件
factory.component_generator(
    conversation_id, thread_id, task_id,
    content, component_type, component_id
)

# 定时任务控制面板
factory.schedule_task_controller_component(conversation_id, thread_id, task)

# 定时任务结果
factory.schedule_task_result_component(task, content)
```

定时任务相关方法内部会自动构造 `ScheduledTaskComponentContent` 并序列化为 JSON。

---

## 9. 通知系统

### 9.1 推送通知模式

除了用户主动发起的 `stream()` 请求，ValueCell 还支持 Agent 主动推送通知。这是通过 `BaseAgent.notify()` 方法实现的：

```python
async def notify(
    self,
    query: str,
    conversation_id: str,
    task_id: str,
    dependencies: Optional[Dict] = None,
) -> AsyncGenerator[NotifyResponse, None]:
```

### 9.2 stream() 与 notify() 的区别

| 维度 | stream() | notify() |
|------|----------|----------|
| 触发方式 | 用户主动发起请求 | Agent 主动推送 |
| 返回类型 | `AsyncGenerator[StreamResponse, None]` | `AsyncGenerator[NotifyResponse, None]` |
| 事件类型 | `StreamResponseEvent` | `NotifyResponseEvent` |
| 典型场景 | 对话回复 | 定时新闻推送、文件监控通知 |

### 9.3 notification 命名空间

`core/agent/responses.py` 中的 `notification` 命名空间提供通知场景的工厂方法：

```python
notification.message(content)                        # 文本通知
notification.component_generator(content, component_type, component_id)  # 组件通知
notification.done(content)                           # 完成信号
notification.failed(content)                         # 失败信号
```

### 9.4 通知的数据流

```
定时触发 / 文件监控事件
     │
     ↓
Agent.notify()
  └── notification.message("新消息内容")
     │
     │  NotifyResponse
     ↓
编排层处理
  └── ResponseRouter 分类 → ResponseBuffer 聚合 → 持久化
     │
     ↓
推送到已订阅的前端 SSE 连接
```

---

## 10. 调试流式事件问题

### 10.1 常见问题与排查路径

| 症状 | 可能原因 | 排查文件 |
|------|----------|----------|
| 前端无输出 | SSE 连接未建立 | `sse-client.ts` → 浏览器 Network 面板 |
| 消息碎片化 | `BufferKey` 不一致（通常是 `task_id` 变化）导致同一消息被拆分为多个 `item_id` | `buffer.py` 的 `annotate()` + 检查 `BufferKey` 四元组是否稳定 |
| 消息丢失 | 立即事件到达时缓冲区未被正确 flush | `buffer.py` 的 `ingest()` |
| 组件不显示 | `component_type` 值与前端渲染器不匹配 | `renderer/index.tsx` + `types.py` |
| 重复消息 | `item_id` 不稳定导致前端认为是新消息 | `buffer.py` 的 `BufferEntry` |
| 工具调用不显示 | Agent 未设置 `stream_intermediate_steps=True` | Agent 的 `core.py` |
| 连接超时 | 后端未响应或超时设置过短 | `sse-client.ts` 的 `timeout` |

### 10.2 关键日志位置

```python
# 路由层 — 查看事件分类是否正确
router.py: logger.info(f"Task {task.task_id} status update: {state}")

# Agent 层 — 查看 Agno 事件是否被正确捕获
agents/*/core.py: logger.info("... processed successfully")
```

### 10.3 前端调试技巧

1. **浏览器 Network 面板**：查看 SSE 请求的 Response，确认事件流正在到达
2. **Console 输出**：SSE 解析失败会输出 `console.warn("Failed to parse SSE message:", data, error)`
3. **React DevTools**：检查渲染器组件的 props，确认 `item_id` 和 `event` 值正确

### 10.4 后端调试技巧

1. **开启调试模式**：设置 `AGENT_DEBUG_MODE=true` 获取详细日志
2. **检查 BufferEntry 状态**：在 `buffer.py` 的关键分支点添加日志，观察键的创建和 flush
3. **验证 metadata 传递**：Agno 事件转 StreamResponse 后，metadata 中的 `response_event` 是否正确决定了路由层的分类

### 10.5 测试覆盖

事件系统有完整的单元测试，位于 `python/valuecell/core/event/tests/`：

| 测试文件 | 覆盖内容 |
|----------|----------|
| `test_response_router.py` | ResponseRouter 的分类逻辑 |
| `test_response_buffer.py` | BufferEntry 的聚合和 flush 行为 |
| `test_event_response_service.py` | EventResponseService 的完整管线 |
| `test_response_factory.py` | ResponseFactory 的各工厂方法 |
| `test_component_id.py` | 组件 ID 替换行为 |

运行测试：

```bash
cd python && uv run pytest valuecell/core/event/tests/
```

---

## 11. 常见问题

### Q1：前端收到的消息顺序和后端发出的一致吗？

是的。后端使用 `asyncio.Queue` 作为事件传递通道，Queue 是 FIFO（先进先出）数据结构。`EventResponseService.emit_many()` 按顺序对每个响应调用 `emit()`，持久化后的事件以相同的顺序推入 Queue。前端 `SSEClient` 的 `readStream()` 按字节流顺序解析事件块，因此前后端的顺序是一致的。

### Q2：如果 Agent 长时间不产出事件会怎样？

SSE 连接有 **30 秒握手超时**（可在构造 `SSEClient` 时通过 `timeout` 选项覆盖）。如果后端在 30 秒内未返回任何数据，前端会通过 `AbortController` 取消请求并触发 `onError` 回调，界面进入超时状态。但后端的 producer 模式仍在继续运行——Agent 的 `stream()` 方法不会因为前端断开而中断，所有事件继续被持久化到 Store。前端重连后可以从 Store 恢复完整状态，不会丢失数据。

### Q3：如何新增一个自定义组件类型？

需要修改三个地方：

1. **后端枚举**：在 `python/valuecell/core/types.py` 的 `ComponentType` 枚举中添加新成员，例如 `MY_COMPONENT = "my_component"`
2. **Payload 模型**：在 `types.py` 中创建对应的 Pydantic 模型（如 `MyComponentData`），定义该组件需要的数据字段
3. **前端渲染器**：在 `frontend/src/components/valuecell/renderer/` 目录下创建新的渲染器组件，并在 `renderer/index.tsx` 中导出；然后在渲染器选择流程中添加 `component_type == "my_component"` 的匹配分支

如果组件支持替换行为（定时任务场景），还需要在构造时传入 `component_id` 参数。

### Q4：message_chunk 的大小是多少？

`message_chunk` 的大小取决于底层 LLM 的输出行为，ValueCell 不做人为切分。每个 chunk 通常包含 **1-5 个 token** 的文本。以中文为例，一个 chunk 大约包含 2-8 个汉字；以英文为例，大约包含 5-15 个单词。LLM 模型内部会根据生成策略决定每次 yield 的文本量，Agno 框架将 LLM 的流式输出直接映射为 `RunContent` 事件，ValueCell 再将其包装为 `message_chunk`。

### Q5：为什么有些事件被静默忽略？

`ResponseRouter` 在 `handle_status_update()` 中对 `state` 为 `submitted` 或 `completed` 的 `TaskStatusUpdateEvent` 采取静默忽略策略。原因是这些底层 A2A 协议状态不携带对前端有用的语义信息——`submitted` 仅表示任务已提交给 Agent（尚未开始产出内容），`completed` 已通过 `TaskStatusEvent.TASK_COMPLETED` 事件在更高层处理。路由层只关注携带实际内容的 `active` 状态事件。

### Q6：如何区分同一条消息的多个 chunk？

通过 `item_id`。`ResponseBuffer` 使用四元组 `BufferKey = (conversation_id, thread_id, task_id, event)` 作为缓冲区键。属于同一个 `BufferKey` 的所有 `message_chunk` 被聚合到同一个 `BufferEntry` 中，共享同一个 `item_id`。前端只需按 `item_id` 分组，就能将多个 chunk 组装为一条完整的消息。`annotate()` 阶段为可缓冲事件分配 `item_id`，确保前端收到的每个 chunk 都带有相同的稳定标识。

---

## 12. 自测检查清单

阅读完本文后，用以下检查清单验证你的理解程度。每一条都应该能够不看原文直接回答。

### 基础认知

- [ ] **事件分类**：能否列出全部 5 类事件（StreamResponseEvent、CommonResponseEvent、SystemResponseEvent、TaskStatusEvent、NotifyResponseEvent）以及每类的典型事件？
- [ ] **ResponseBuffer 的存在意义**：能否解释为什么需要 ResponseBuffer？（核心原因：为属于同一段落的 chunk 分配稳定的 `item_id`，使前端能够将多个 `message_chunk` 组装为一条逐步生长的消息）
- [ ] **立即事件与可缓冲事件**：能否描述两者的区别？（立即事件携带自包含的完整信息，直接持久化并触发 flush；可缓冲事件携带增量文本片段，需要在 BufferEntry 中聚合）

### 管线与转换

- [ ] **事件处理管线**：能否画出从 Agent 产出事件到前端渲染的三个处理层？（ResponseRouter 语义分类 → EventResponseService 统一门面 → 前端 SSE 消费）
- [ ] **Agno 事件转换**：能否追踪从 Agno 事件到 ValueCell 事件的完整链路？（Agno `RunEvent` → Agent `stream()` 中的 `streaming.*` 调用 → AgentWrapper 包装为 `TaskStatusUpdateEvent` → ResponseRouter 分类 → EventResponseService 持久化）
- [ ] **自定义 SSE Client**：能否解释为什么不用原生 EventSource？（两个原因：只支持 GET 请求，不支持自定义 Header；ValueCell 需要在 POST body 中传递对话消息并携带认证信息）

### 前端与组件

- [ ] **8 个渲染器**：能否列出全部 8 个渲染器（ChatConversationRenderer、MarkdownRenderer、ReasoningRenderer、ReportRenderer、ScheduledTaskControllerRenderer、ScheduledTaskRenderer、ToolCallRenderer、UnknownRenderer）及其对应的事件类型？
- [ ] **组件替换机制**：能否描述 `component_id` 如何实现组件替换？（`component_generator` 事件的 metadata 中包含 `component_id` 时，该值会覆盖自动生成的 `item_id`，前端用相同的 `component_id` 替换已有组件而非新增）
- [ ] **BufferKey 四元组**：能否说出 BufferKey 的组成（conversation_id、thread_id、task_id、event）并解释为什么 task_id 变化会导致消息碎片化？

### 系统集成

- [ ] **stream() 与 notify() 的区别**：能否从触发方式、返回类型、事件类型和典型场景四个维度说明两者的差异？
- [ ] **streaming 与 notification 命名空间**：能否列出两个命名空间各自提供的工厂方法，并解释为什么 `notification.component_generator()` 返回 `StreamResponse` 而非 `NotifyResponse`？

---

## 13. 调试速查

当流式事件出现问题时，使用下表快速定位原因和排查方向。

| 症状 | 可能原因 | 首先检查的文件 | 排查思路 |
|------|----------|---------------|----------|
| 前端完全无输出 | SSE 连接未建立或请求失败 | `frontend/src/lib/sse-client.ts` | 浏览器 Network 面板查看请求状态码和响应头，确认 `Content-Type: text/event-stream` |
| 消息碎片化（应是一条消息却显示为多条） | `BufferKey` 不一致，通常是 `task_id` 在中途变化 | `python/valuecell/core/event/buffer.py` 的 `annotate()` | 检查同一会话中 chunk 的 BufferKey 四元组是否稳定，重点关注 `task_id` 是否在 Agent 产出 chunk 的过程中发生了变化 |
| 消息内容丢失 | 立即事件到达时缓冲区未被正确 flush | `python/valuecell/core/event/buffer.py` 的 `ingest()` | 在 `ingest()` 的 flush 分支添加日志，观察立即事件到达时是否触发了同上下文缓冲区的刷出 |
| 组件不显示或显示为 Unknown | `component_type` 字符串值与前端渲染器匹配逻辑不一致 | `python/valuecell/core/types.py` + `frontend/src/components/valuecell/renderer/index.tsx` | 对比后端枚举值和前端匹配条件，确认大小写和拼写完全一致 |
| 消息重复出现 | `item_id` 不稳定导致前端认为是新消息 | `python/valuecell/core/event/buffer.py` 的 `BufferEntry` | 检查 `annotate()` 是否每次都分配了新的 `item_id` 而非复用已有的 |
| 工具调用不显示 | Agent 未开启工具调用事件流 | Agent 的 `core.py`（如 `python/valuecell/agents/*/core.py`） | 确认 `arun()` 调用中包含 `stream_intermediate_steps=True` 参数 |
| 连接超时断开 | 后端响应延迟超过 30 秒 | `frontend/src/lib/sse-client.ts` 的 `timeout` 配置 | 检查 Agent 是否在长耗时操作前产出了 keep-alive 事件，或适当增大 `timeout` 值 |
| JSON 解析失败但连接未断（预期行为） | LLM 输出跨越多个 TCP 包，单个事件块不完整 | `frontend/src/lib/sse-client.ts` 的 `best-effort-json-parser` | 查看控制台 `console.warn` 输出，确认后续包到达后事件被正确重组 |
| 推理过程不显示 | Agno 事件中 `RunReasoning` 未被 Agent 处理 | Agent 的 `core.py` 事件循环 | 确认 Agent 的 `stream()` 方法中是否处理了 `RunReasoning` 事件（注意：当前默认模板可能只处理 `RunContent` 和工具调用事件） |
| 定时任务组件未替换 | `component_id` 未正确传递 | `python/valuecell/core/event/factory.py` | 确认 `schedule_task_result_component()` 调用时是否传入了与控制面板相同的 `component_id` |
| 路由分类错误 | `metadata` 中的 `response_event` 字段缺失或拼写错误 | `python/valuecell/core/event/router.py` 的 `handle_status_update()` | 检查 AgentWrapper 包装事件时是否正确设置了 `metadata["response_event"]` |

> **快速定位原则**：问题出现在"前端能看到但显示不对"时，优先检查渲染器和 Buffer；问题出现在"前端完全看不到"时，优先检查 SSE 连接和路由层。

---

## 14. 继续阅读

| 你想深入了解什么 | 推荐阅读 |
|----------------|----------|
| 系统架构全貌 | [系统架构详解](./architecture.md) |
| 从源码入口开始阅读 | [源码导读](./source-guide.md) |
| 开发新 Agent | [Agent 开发与扩展](./agent-extension.md) |
| 配置系统细节 | [配置说明](./configuration.md) |
| 系统化学习路径 | [学习路径](./learning-path.md) |
