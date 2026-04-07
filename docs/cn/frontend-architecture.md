# ValueCell 前端架构深度解析

## 学习目标

阅读本文后，你将能够：

- 解释前端技术栈中每一项技术的选型理由和实际使用方式
- 描述从根布局到页面组件的 Provider 注入链
- 画出 SSE 流式通信的完整数据流（从 `useSSE` Hook 到渲染器）
- 区分 Zustand 和 TanStack Query 各自管理的状态边界
- 理解 8 个渲染器如何将后端事件映射为不同的 UI 组件
- 知道前端修改应该从哪个文件开始

---

## 1. 前端技术选型与理由

ValueCell 前端位于 `frontend/` 目录，核心依赖如下：

| 技术 | 版本 | 用途 | 选型理由 |
|------|------|------|----------|
| React Router v7 | ^7.10.1 | 路由与页面框架（原 Remix） | 文件系统路由 + SPA 模式，适合桌面应用 |
| TypeScript | ^5.9.3 | 类型安全 | 前后端契约严格、事件类型多 |
| Vite | ^7.1.12（rolldown-vite） | 构建工具 | Tauri 集成友好、开发体验好 |
| Bun | 1.3.3 | 包管理器 | 安装速度快、兼容 npm 生态 |
| Radix UI + Tailwind CSS | — | UI 组件与样式 | 无障碍 + 定制灵活 + Tailwind v4 原子化 |
| Zustand | ^5.0.9 | 本地持久化状态 | 轻量、支持 `persist` 中间件 |
| TanStack Query | ^5.90.12 | 服务端状态管理 | 缓存、失效、自动重试 |
| ECharts | ^6.0.0 | 金融图表 | K 线图、折线图等金融可视化 |
| i18next | ^25.7.2 | 国际化 | React 绑定成熟、支持 4 种语言 |
| next-themes | ^0.4.6 | 主题切换 | 简洁的 dark/light/system 切换 |
| Tauri v2 | ^2.9.1 | 桌面打包 | Rust 后端、体积小、安全 |

> **版本说明**：上表中的版本号为编写文档时的快照，实际版本以 `frontend/package.json` 为准。

### 1.1 SPA 模式

前端采用 **SPA（单页应用）模式**，而非 SSR。在 `frontend/react-router.config.ts` 中明确配置：

```typescript
export default {
  ssr: false,       // 不启用服务端渲染
  appDirectory: "src",
} satisfies Config;
```

这一选择与桌面应用场景匹配——Tauri 的 WebView 加载本地静态资源，不需要服务端渲染。

### 1.2 Vite 配置要点

`frontend/vite.config.ts` 中有几个针对 Tauri 的特殊配置：

- **固定端口**：开发服务器监听 `1420` 端口（`strictPort: true`），Tauri 窗口指向此地址
- **HMR 配置**：支持通过 `TAURI_DEV_HOST` 环境变量设置 WebSocket 主机
- **忽略 `src-tauri/`**：避免 Rust 编译产物触发前端热更新
- **Tailwind v4 集成**：通过 `@tailwindcss/vite` 插件集成，无需 PostCSS
- **SVG Sprite**：`vite-plugin-svg-sprite` 自动将 `assets/svg/` 下的图标打包为 SVG sprite

---

## 2. 项目结构概览

```
frontend/
├── src/
│   ├── root.tsx                    # 根布局，全局 Provider 注入
│   ├── routes.ts                   # 路由定义
│   ├── global.css                  # 全局样式（Tailwind 指令）
│   ├── vite-env.d.ts              # Vite 环境类型声明
│   │
│   ├── app/                        # 页面组件（按路由组织）
│   │   ├── redirect-to-home.tsx    # / → /home 重定向
│   │   ├── home/                   # 首页与股票详情
│   │   ├── agent/                  # Agent 对话与配置页面
│   │   ├── market/                 # 市场与 Agent 列表
│   │   ├── setting/                # 设置页面
│   │   └── test.tsx                # 开发测试页面
│   │
│   ├── api/                        # API 调用封装（TanStack Query hooks）
│   │   ├── agent.ts                # Agent 列表、信息
│   │   ├── conversation.ts         # 会话管理
│   │   ├── setting.ts              # 设置相关
│   │   ├── stock.ts                # 股票数据
│   │   ├── strategy.ts             # 策略管理
│   │   └── system.ts               # 系统信息
│   │
│   ├── components/
│   │   ├── ui/                     # 基础 UI 组件（Radix UI + Tailwind）
│   │   ├── valuecell/              # 业务组件
│   │   │   ├── app/                # 全局组件（侧边栏、健康检查、自动更新）
│   │   │   ├── charts/             # ECharts 金融图表
│   │   │   ├── renderer/           # 8 个事件渲染器
│   │   │   ├── form/               # 表单组件
│   │   │   ├── button/             # 按钮组件
│   │   │   └── icon/               # 图标组件
│   │   └── tradingview/            # TradingView 嵌入组件
│   │
│   ├── hooks/                      # 自定义 React Hooks
│   │   ├── use-sse.ts              # SSE 连接 Hook
│   │   ├── use-chart-resize.ts     # 图表自适应
│   │   ├── use-debounce.ts         # 防抖
│   │   ├── use-mobile.ts           # 移动端检测
│   │   ├── use-tauri-info.ts       # Tauri 环境信息
│   │   ├── use-update-toast.tsx    # 更新提示
│   │   └── use-form.tsx            # 表单 Hook
│   │
│   ├── lib/                        # 工具库
│   │   ├── api-client.ts           # 统一 HTTP 客户端
│   │   ├── sse-client.ts           # SSE 客户端
│   │   ├── agent-store.ts          # Agent 会话状态处理逻辑
│   │   ├── tracker.ts              # 使用追踪
│   │   ├── time.ts                 # 时间工具
│   │   └── utils.ts                # 通用工具函数
│   │
│   ├── store/                      # Zustand 状态管理
│   │   ├── agent-store.ts          # Agent 会话状态
│   │   ├── settings-store.ts       # 用户设置（语言、颜色模式）
│   │   ├── system-store.ts         # 系统信息（认证、用户数据）
│   │   └── plugin/
│   │       └── tauri-store-state.ts # Tauri 原生存储适配器
│   │
│   ├── types/                      # TypeScript 类型定义
│   │   ├── agent.ts                # Agent 相关类型
│   │   ├── conversation.ts         # 会话类型
│   │   ├── renderer.ts             # 渲染器类型
│   │   ├── stock.ts                # 股票类型
│   │   ├── strategy.ts             # 策略类型
│   │   ├── setting.ts              # 设置类型
│   │   ├── system.ts               # 系统类型
│   │   └── chart.ts                # 图表类型
│   │
│   ├── constants/                  # 常量定义
│   │   ├── api.ts                  # API Query Keys 与 URL
│   │   ├── agent.ts                # Agent 常量
│   │   ├── stock.ts                # 股票颜色常量
│   │   ├── schema.ts               # 表单 Schema
│   │   └── icons.ts                # 图标常量
│   │
│   ├── i18n/                       # 国际化
│   │   ├── index.ts                # i18next 初始化
│   │   └── locales/                # 翻译文件
│   │       ├── en.json
│   │       ├── zh_CN.json
│   │       ├── zh_TW.json
│   │       └── ja.json
│   │
│   ├── provider/                   # Context Provider
│   │   ├── tracker-provider.tsx    # 全局点击追踪
│   │   └── multi-section-provider.tsx # 多分区上下文
│   │
│   ├── assets/                     # 静态资源
│   │   ├── svg/                    # SVG 图标
│   │   └── png/                    # 图片资源
│   │
│   └── mock/                       # Mock 数据
│
├── src-tauri/                      # Tauri 桌面应用配置
│   ├── Cargo.toml                  # Rust 依赖
│   ├── tauri.conf.json             # Tauri 应用配置
│   └── capabilities/               # 权限声明
│
├── package.json                    # 依赖与脚本
├── react-router.config.ts          # React Router 配置
└── vite.config.ts                  # Vite 配置
```

---

## 3. 根布局与全局 Provider

`frontend/src/root.tsx` 是前端架构的起点。它分为两个核心部分：

### 3.1 Layout 组件

`Layout` 函数负责 HTML 文档结构，根据当前语言设置 `html` 元素的 `lang` 属性：

```
支持的语言映射：
en    → "en"
zh_CN → "zh-CN"
zh_TW → "zh-TW"
ja    → "ja"
```

### 3.2 Root 组件的 Provider 注入链

`Root` 组件按以下顺序嵌套 Provider：

```
QueryClientProvider          ← TanStack Query 缓存层
  └── ThemeProvider          ← next-themes 暗色/亮色主题
        └── BackendHealthCheck  ← 后端健康探测（不可用时显示加载画面）
              ├── TrackerProvider     ← 全局点击事件追踪
              │     └── SidebarProvider    ← 侧边栏状态
              │           └── <div>         ← 页面布局容器
              │                 ├── AppSidebar      ← 全局导航侧边栏
              │                 ├── <Outlet />      ← 页面内容（路由出口）
              │                 └── <Toaster />     ← 全局消息提示
              └── AutoUpdateCheck        ← 桌面应用自动更新检测
```

> **源码注意**：`AutoUpdateCheck` 和 `TrackerProvider` 在 `BackendHealthCheck` 内部是**兄弟节点**（平级关系），不是嵌套关系。页面布局容器使用 `fixed flex size-full overflow-hidden` 样式，将侧边栏、主内容区和消息提示横向排列。

每一层的职责：

| Provider | 来源 | 职责 |
|----------|------|------|
| `QueryClientProvider` | TanStack Query | 为所有 API 请求提供缓存、失效、重试机制 |
| `ThemeProvider` | next-themes | 管理 `valuecell-theme` 存储 Key，支持 system/dark/light |
| `BackendHealthCheck` | 自定义组件 | 通过 TanStack Query 轮询 `/api/v1/healthz`，不可用时展示全屏加载 |
| `TrackerProvider` | 自定义组件 | 全局 `click` 事件委托，捕获 `data-track` 属性并发送追踪事件 |
| `SidebarProvider` | 基于 Radix UI 封装 | 管理侧边栏展开/折叠状态（`components/ui/sidebar`） |
| `AutoUpdateCheck` | 自定义组件 | 仅在 Tauri 环境生效，每 60 分钟检查一次更新 |

### 3.3 QueryClient 配置

`QueryClient` 实例在 `root.tsx` 中创建并配置全局默认值：

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,      // 5 分钟内认为数据新鲜
      gcTime: 30 * 60 * 1000,         // 30 分钟垃圾回收
      refetchOnWindowFocus: false,     // 窗口聚焦时不自动重取
      retry: 1,                        // 失败重试 1 次
    },
    mutations: {
      retry: 1,                        // 变更操作重试 1 次
    },
  },
});
```

---

## 4. 路由系统

### 4.1 路由定义

路由在 `frontend/src/routes.ts` 中定义，使用 React Router v7 的文件系统路由 API：

| 路径 | 组件文件 | 说明 |
|------|----------|------|
| `/` | `app/redirect-to-home.tsx` | 重定向到 `/home` |
| `/home` | `app/home/_layout.tsx` → `app/home/home.tsx` | 首页布局，展示投资组合 |
| `/home/stock/:stockId` | `app/home/_layout.tsx` → `app/home/stock.tsx` | 股票详情页 |
| `/market` | `app/market/agents.tsx` | 市场页面，Agent 列表 |
| `/agent/:agentName` | `app/agent/chat.tsx` | Agent 对话页面 |
| `/agent/:agentName/config` | `app/agent/config.tsx` | Agent 配置页面 |
| `/setting` | `app/setting/_layout.tsx` → `app/setting/models.tsx` | 设置布局（默认进入模型设置） |
| `/setting/general` | `app/setting/_layout.tsx` → `app/setting/general.tsx` | 通用设置 |
| `/setting/memory` | `app/setting/_layout.tsx` → `app/setting/memory.tsx` | 记忆管理 |
| `/test` | `app/test.tsx` | 开发测试页面，仅用于本地调试 |

### 4.2 路由组织方式

路由使用四个核心 API 组织：

- **`index()`**：根路径重定向
- **`layout()`**：共享布局的嵌套路由（如 `/home` 和 `/setting` 都有各自的布局组件）
- **`prefix()`**：路由前缀分组
- **`route()`**：独立路由定义

嵌套结构意味着 `/home` 和 `/home/stock/:stockId` 共享 `home/_layout.tsx` 的布局；`/setting`、`/setting/general`、`/setting/memory` 共享 `setting/_layout.tsx` 的布局。

### 4.3 侧边栏导航

`AppSidebar` 组件（`frontend/src/components/valuecell/app/app-sidebar.tsx`）通过 `useGetAgentList` 动态获取已启用的 Agent 列表，生成导航按钮。导航结构分为三个区域：

```
顶部区域（固定导航）：
  - 首页 (/home)
  - 策略 Agent (/agent/StrategyAgent)
  - 市场 (/market)
  - 会话列表（弹出面板）

中部区域（动态 Agent 列表）：
  - 从后端 API 获取所有 enabled Agent
  - 每个 Agent 显示头像 + Tooltip

底部区域（固定导航）：
  - 设置 (/setting)
```

---

## 5. 状态管理架构

前端采用 **Zustand + TanStack Query** 双轨状态管理策略：

```
┌─────────────────────────────────────────────────────┐
│                    前端状态管理                       │
├─────────────────────┬───────────────────────────────┤
│   Zustand（本地）    │    TanStack Query（服务端）     │
│                     │                               │
│ • 用户设置          │ • API 缓存                     │
│   - 语言偏好        │   - Agent 列表                 │
│   - 涨跌颜色模式    │   - 股票数据                   │
│ • 系统信息          │   - 会话历史                   │
│   - 认证 Token      │   - 策略列表                   │
│   - 用户资料        │   - 模型提供商                 │
│ • Agent 会话状态    │                               │
│   - 实时对话数据    │ • 缓存失效与重取               │
│   - SSE 事件累积    │ • 请求去重                     │
│                     │ • 后台更新                     │
└─────────────────────┴───────────────────────────────┘
```

### 5.1 Zustand Stores

#### SettingsStore（`store/settings-store.ts`）

管理用户偏好设置，使用 `persist` 中间件持久化到 `localStorage`（key: `valuecell-settings`）：

| 状态 | 类型 | 说明 |
|------|------|------|
| `stockColorMode` | `"GREEN_UP_RED_DOWN"` 或 `"RED_UP_GREEN_DOWN"` | 涨跌颜色模式（适配中国/国际习惯） |
| `language` | `"en"` / `"zh_CN"` / `"zh_TW"` / `"ja"` | 界面语言 |

还派生了多个便捷 Hook：`useStockColors()`、`useStockGradientColors()`、`useStockBadgeColors()` 等。

#### SystemStore（`store/system-store.ts`）

管理认证和用户信息。使用 `persist` 中间件 + **TauriStoreState** 自定义存储适配器：

- 在 Tauri 环境中使用 `@tauri-apps/plugin-store` 原生存储
- 在浏览器环境中自动降级到 `localStorage`
- 存储包含 `access_token`、`refresh_token`、用户资料
- 使用 `devtools` 中间件，仅在开发环境启用 Redux DevTools

#### AgentStore（`store/agent-store.ts`）

管理 Agent 对话的实时状态，**不使用 persist**（对话历史通过 API 加载）：

| 状态 | 说明 |
|------|------|
| `agentStore` | 以 `conversation_id` 为 key 的嵌套结构 |
| `curConversationId` | 当前活跃的对话 ID |

核心方法：

- `dispatchAgentStore(action)` — 处理单个 SSE 事件，更新对话状态
- `dispatchAgentStoreHistory(conversationId, history)` — 批量加载历史记录

### 5.2 TanStack Query 使用模式

API 调用封装在 `frontend/src/api/` 目录下，每个文件导出基于 TanStack Query 的自定义 Hook：

| 文件 | 导出 Hook 示例 | 说明 |
|------|---------------|------|
| `agent.ts` | `useGetAgentList()` | 获取 Agent 列表 |
| `conversation.ts` | — | 会话管理 |
| `stock.ts` | — | 股票数据查询 |
| `strategy.ts` | — | 策略管理 |
| `setting.ts` | — | 设置 API |
| `system.ts` | `useBackendHealth()` | 后端健康检查 |

Query Keys 在 `constants/api.ts` 中集中管理，按领域分组：

```typescript
export const API_QUERY_KEYS = {
  STOCK: { watchlist, stockList, stockDetail, ... },
  AGENT: { agentList, agentInfo, ... },
  CONVERSATION: { conversationList, conversationHistory, ... },
  SETTING: { memoryList, modelProviders, ... },
  STRATEGY: { strategyList, strategyApiKey, ... },
  SYSTEM: { ... },
} as const;
```

集中管理的好处是缓存的 **失效（invalidation）** 可以精确控制——当某个 mutation 成功后，只失效相关域的缓存。

### 5.3 Agent 会话状态处理

`frontend/src/lib/agent-store.ts` 是理解前端实时对话体验的关键文件。它使用 `mutative` 库进行不可变更新，核心处理逻辑：

```
SSE 事件 → processSSEEvent() → 按事件类型分发
  ├─ message_chunk    → append 文本内容到 markdown 类型
  ├─ reasoning        → append-reasoning（JSON 内追加 content 字段）
  ├─ reasoning_started → 创建初始 reasoning 项
  ├─ reasoning_completed → 标记 isComplete
  ├─ tool_call_started/completed → replace（整体替换）
  ├─ component_generator → 按子类型分发到对应渲染器
  └─ 其他事件 → markdown 类型兜底
```

每个对话的数据结构为三层嵌套：

```
ConversationView
  ├── threads: Record<thread_id, ThreadView>
  │     └── tasks: Record<task_id, TaskView>
  │           └── items: ChatItem[]  ← 实际的对话消息列表
  └── sections: Record<component_type, ThreadView>
        └── （与 threads 相同结构，用于独立分区的组件类型）
```

### 5.4 状态流图

以下示意图展示了 SSE 事件从后端到达前端渲染器的完整数据流：

```
                         后端 Python
                             │
                   POST /api/v1/agent_stream
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                        SSEClient                                 │
│                  (lib/sse-client.ts)                             │
│                                                                  │
│  fetch(POST) → ReadableStream → TextDecoder → 按 "\n\n" 拆分   │
│                             │                                    │
│                    best-effort JSON 解析                         │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    onData 回调（原始事件）
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      useSSE Hook                                 │
│                   (hooks/use-sse.ts)                             │
│                                                                  │
│  单例 SSEClient（useRef）                                        │
│  handlersRef 避免闭包陈旧 → 调用最新 onChunk handler             │
│  isStreaming 状态同步到 React                                    │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    onChunk(message)
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│               agent-store.ts（状态处理层）                        │
│             (lib/agent-store.ts + store/agent-store.ts)          │
│                                                                  │
│  processSSEEvent(message, conversationId)                        │
│       │                                                          │
│       ├─ message_chunk      → append 到 markdown 类型的 item    │
│       ├─ reasoning_started  → 创建初始 reasoning 项             │
│       ├─ reasoning          → 追加 reasoning content            │
│       ├─ reasoning_completed→ 标记 isComplete                   │
│       ├─ tool_call_started  → replace（整体替换）                │
│       ├─ tool_call_completed→ replace（整体替换）                │
│       ├─ component_generator→ 按 component_type 分发            │
│       └─ 其他事件           → markdown 类型兜底                  │
│                                                                  │
│  mutative 库执行不可变更新                                       │
│  dispatchAgentStore(action) → Zustand set() → 触发 React 重渲染 │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                  Zustand state 变更
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                 renderer/index.tsx（渲染分发）                    │
│                                                                  │
│  根据 ChatItem 的 component_type 查找 RendererPropsMap          │
│       │                                                          │
│       ├─ "markdown"            → MarkdownRenderer               │
│       ├─ "reasoning"           → ReasoningRenderer              │
│       ├─ "tool_call"           → ToolCallRenderer               │
│       ├─ "report"              → ReportRenderer                 │
│       ├─ "scheduled_task_result"                                   │
│       │                       → ScheduledTaskRenderer           │
│       ├─ "scheduled_task_controller"                                │
│       │                       → ScheduledTaskControllerRenderer  │
│       ├─ "subagent_conversation"                                    │
│       │                       → ChatConversationRenderer        │
│       └─ 未知类型             → UnknownRenderer（兜底）          │
└──────────────────────────────────────────────────────────────────┘
```

数据流核心路径：**后端 SSE 端点 → SSEClient（fetch 流式读取）→ useSSE Hook（React 桥接）→ agent-store.ts（事件处理 + 状态更新）→ Zustand store → React 重渲染 → renderer/index.tsx（组件分发）→ 各渲染器 UI 组件**。

---

## 6. SSE 通信层

### 6.1 为什么不用原生 EventSource

原生 `EventSource` 只支持 GET 请求且不支持自定义 Header，而 ValueCell 的 SSE 端点 `POST /api/v1/agent_stream` 需要在请求体中传递对话上下文。完整的设计决策分析见 [streaming-deep-dive.md §5](./streaming-deep-dive.md)。

### 6.2 SSEClient 实现

`frontend/src/lib/sse-client.ts` 是基于 `fetch + ReadableStream` 的自定义 SSE 客户端：

```
┌──────────────────────────────────────────────┐
│               SSEClient                       │
│                                              │
│  connect(body)                               │
│    └── startConnection()                     │
│          ├── fetch(url, { method: "POST" })  │
│          ├── 超时检测（AbortController）       │
│          └── readStream(response.body)       │
│                ├── TextDecoder 解码           │
│                ├── 按 "\n\n" 拆分事件块       │
│                └── processEvent()            │
│                      ├── 提取 "data:" 行     │
│                      └── best-effort JSON 解析│
│                                              │
│  状态：CONNECTING → OPEN → CLOSED            │
│  事件：onData / onOpen / onError / onClose   │
└──────────────────────────────────────────────┘
```

关键实现细节：

- **POST 方法**：`method: "POST"`，携带 `Content-Type: application/json`
- **超时机制**：默认 30 秒握手超时，通过 `AbortController` 实现
- **流式读取**：使用 `ReadableStream.getReader()` 逐块读取
- **事件解析**：以 `\n\n` 为分隔符拆分 SSE block，提取 `data:` 前缀的行
- **容错解析**：使用 `best-effort-json-parser` 库处理不完整的 JSON（流式场景下 JSON 可能被截断）
- **防重连**：`connect()` 在已连接状态下拒绝重复调用

### 6.3 useSSE Hook

`frontend/src/hooks/use-sse.ts` 将 `SSEClient` 封装为 React Hook：

```typescript
interface UseSSEReturn {
  isStreaming: boolean;         // 当前是否在流式传输中
  connect: (body?) => Promise<void>;  // 建立连接
  close: () => void;            // 关闭连接
}
```

设计要点：

- **单例客户端**：通过 `useRef` 确保组件生命周期内只创建一个 `SSEClient` 实例
- **Handler 引用更新**：使用 `handlersRef` 避免闭包陈旧问题——Handler 可以在组件中动态更新，SSEClient 始终调用最新版本
- **状态同步**：SSE 客户端的连接状态通过 `handleStateChange` 回调同步到 React 状态 `isStreaming`

---

## 7. API 客户端层

### 7.1 ApiClient 类

`frontend/src/lib/api-client.ts` 实现了统一的 HTTP 请求封装：

```
┌─────────────────────────────────────────────┐
│              ApiClient                       │
│                                             │
│  request<T>(method, endpoint, data, config) │
│    ├── getServerUrl(endpoint)  → URL 拼接   │
│    ├── 注入 Authorization Header（如需）      │
│    ├── fetch(url, requestConfig)             │
│    └── handleResponse<T>(response)          │
│          ├── 401 → Token 刷新 → 重试        │
│          ├── !ok → 抛出 ApiError            │
│          └── JSON / Text 解析               │
│                                             │
│  方法：get / post / put / patch / delete    │
│       / upload（FormData）                   │
└─────────────────────────────────────────────┘
```

### 7.2 后端信封结构

后端 API 返回统一的信封格式：

```typescript
interface ApiResponse<T> {
  code: number;
  data: T;
  msg: string;
}
```

### 7.3 Token 刷新机制

当收到 `401` 状态码时，`ApiClient` 自动执行 Token 刷新流程：

1. 使用 `refresh_token` 调用 `POST /api/v1/refresh`
2. 获取新的 `access_token` 和 `refresh_token`
3. 获取用户信息并更新 `SystemStore`
4. 刷新失败则清除用户信息（退出登录状态）

### 7.4 服务器 URL 解析

`getServerUrl()` 函数处理两种情况：

- 以 `http` 开头的 URL 直接使用（用于在线服务 `https://backend.valuecell.ai/api/v1`）
- 相对路径拼接 `VITE_API_BASE_URL` 环境变量（默认 `http://localhost:8000/api/v1`）

---

## 8. 渲染器系统

### 8.1 渲染器架构

`frontend/src/components/valuecell/renderer/` 目录包含 **8 个渲染器**，每个处理一种后端事件类型对应的 UI 展示：

```
后端 SSE 事件
      ↓
renderer/index.tsx  ──→  根据 component_type 选择渲染器
      ↓
┌──────────────────────────────────────────────────────┐
│  MarkdownRenderer          ← message_chunk、普通消息  │
│  ReasoningRenderer         ← reasoning 推理过程      │
│  ToolCallRenderer          ← tool_call 工具调用      │
│  ReportRenderer            ← report 研究报告         │
│  ScheduledTaskRenderer     ← scheduled_task_result   │
│  ScheduledTaskControllerRenderer ← 任务控制面板       │
│  ChatConversationRenderer  ← subagent_conversation   │
│  UnknownRenderer           ← 未知类型兜底            │
└──────────────────────────────────────────────────────┘
```

### 8.2 类型安全的 Props 映射

`frontend/src/types/renderer.ts` 定义了渲染器的类型系统：

```typescript
type RendererPropsMap = {
  scheduled_task_result: ScheduledTaskRendererProps;
  scheduled_task_controller: ScheduledTaskControllerRendererProps;
  report: ReportRendererProps;
  reasoning: ReasoningRendererProps;
  markdown: MarkdownRendererProps;
  tool_call: ToolCallRendererProps;
  subagent_conversation: ChatConversationRendererProps;
};
```

每个渲染器共享 `BaseRendererProps`（`content` + `className` + `onOpen`），部分渲染器有额外的 Props：

- `ReportRenderer`：`isActive`（当前是否活跃）
- `ReasoningRenderer`：`isComplete`（推理是否完成）

> **前后端组件类型不完全对齐**：后端 `ComponentType` 枚举定义了 7 种组件类型（含 `filtered_line_chart`、`filtered_card_push_notification`、`profile`），但前端当前只为其中 7 种提供了专用渲染器。后端产出未在 `RendererPropsMap` 中映射的组件类型时，前端使用 `UnknownRenderer` 兜底渲染，不会导致界面空白。

### 8.3 渲染器与后端事件的对应关系

| 后端事件 | component_type | 渲染器 | 展示效果 |
|----------|----------------|--------|----------|
| `message_chunk` | `markdown` | MarkdownRenderer | 逐字流式输出的文本 |
| `reasoning` / `reasoning_started` | `reasoning` | ReasoningRenderer | 可折叠的推理过程 |
| `tool_call_started` / `tool_call_completed` | `tool_call` | ToolCallRenderer | 工具调用的参数和结果 |
| `component_generator(report)` | `report` | ReportRenderer | Markdown 格式的研究报告 |
| `component_generator(scheduled_task_result)` | `scheduled_task_result` | ScheduledTaskRenderer | 定时任务执行结果 |
| `component_generator(scheduled_task_controller)` | `scheduled_task_controller` | ScheduledTaskControllerRenderer | 定时任务控制面板 |
| `component_generator(subagent_conversation)` | `subagent_conversation` | ChatConversationRenderer | 子 Agent 会话边界 |

---

## 9. 国际化（i18next）

### 9.1 配置

`frontend/src/i18n/index.ts` 初始化 i18next：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 语言资源 | `en`、`zh_CN`、`zh_TW`、`ja` | 4 种语言 |
| 默认语言 | 浏览器语言自动检测 | 中文系统 → `zh_CN`，日文 → `ja`，其他 → `en` |
| Fallback | `"en"` | 缺失翻译时使用英文 |
| 开发调试 | `import.meta.env.DEV` | 开发环境启用 debug 和缺失 key 日志 |

### 9.2 语言检测逻辑

语言检测在 `store/settings-store.ts` 的 `getLanguage()` 函数中：

```typescript
const map: Record<string, string> = {
  "zh-Hans": "zh_CN",
  "zh-Hant": "zh_TW",
  "ja-JP": "ja",
};
return map[navigator.language] ?? DEFAULT_LANGUAGE;  // 默认 "en"
```

### 9.3 使用方式

组件中通过 `react-i18next` 的 `useTranslation` Hook 使用：

```typescript
const { t } = useTranslation();
// 在 JSX 中
<p>{t("common.settingUpEnvironment")}</p>
```

侧边栏导航等文本全部通过 `t()` 函数获取，支持运行时切换语言（`setLanguage()` 会同步调用 `i18n.changeLanguage()`）。

---

## 10. Tauri v2 桌面集成

### 10.1 Tauri 配置

`frontend/src-tauri/tauri.conf.json` 定义了桌面应用的核心配置：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| productName | `"ValueCell"` | 应用名称 |
| identifier | `"com.valuecell.valuecellapp"` | 应用唯一标识 |
| 窗口最小尺寸 | 1300 x 780 | 固定的最小窗口大小 |
| hiddenTitle | `true` | 隐藏标题栏（自定义窗口） |
| devUrl | `http://localhost:1420` | 开发模式 URL |
| frontendDist | `../build/client` | 生产构建输出目录 |

### 10.2 Tauri 插件

`Cargo.toml` 中启用了以下 Tauri 插件：

| 插件 | 用途 |
|------|------|
| `tauri-plugin-opener` | 打开外部链接 |
| `tauri-plugin-log` | 原生日志 |
| `tauri-plugin-shell` | 调用系统命令 |
| `tauri-plugin-store` | 原生键值存储 |
| `tauri-plugin-deep-link` | URL Scheme 注册（`valuecell://`） |
| `tauri-plugin-os` | 操作系统信息 |
| `tauri-plugin-dialog` | 原生对话框 |
| `tauri-plugin-fs` | 文件系统访问 |
| `tauri-plugin-process` | 进程管理（仅桌面端） |
| `tauri-plugin-single-instance` | 单实例锁（仅桌面端） |
| `tauri-plugin-updater` | 自动更新（仅桌面端） |

### 10.3 TauriStoreState 适配器

`frontend/src/store/plugin/tauri-store-state.ts` 实现了 Zustand `persist` 中间件的 `StateStorage` 接口：

- 运行在 Tauri 环境时，使用 `@tauri-apps/plugin-store` 进行原生存储
- 运行在浏览器时，自动降级（返回 `null`，由 Zustand 使用默认 localStorage）
- 写操作通过 **1 秒防抖** 批量保存，避免频繁 I/O

### 10.4 环境感知

前端通过 `isTauri()` 函数（来自 `@tauri-apps/api/core`）判断运行环境：

- **Tauri 环境**：启用自动更新检查、原生存储、使用追踪、deep-link
- **浏览器环境**：跳过所有 Tauri 特有功能

`AutoUpdateCheck` 组件在 Tauri 环境中每 60 分钟检查一次更新，首次延迟 5 秒：

```typescript
const CHECK_INTERVAL_MS = 60 * 60 * 1000;  // 1 小时
const INITIAL_DELAY_MS = 5 * 1000;          // 5 秒
```

### 10.5 打包资源

Tauri 打包时会将以下资源包含在安装包中：

- Python 后端（`../../python` → `backend`）
- 环境变量模板（`../../.env.example`）
- `uv` 包管理器二进制文件（`binaries/uv`）

这意味着桌面应用包含完整的后端，无需用户单独安装 Python。

### 10.6 Python 后端的运行方式

Tauri 应用启动时，Rust 侧通过 `tauri-plugin-shell` 将 `uv` 和 Python 后端作为子进程启动：

1. **内嵌 `uv` 二进制**：`binaries/uv` 随安装包分发，用于在用户机器上管理 Python 环境
2. **Python 源码打包**：`../../python` 目录被复制到安装包的 `backend/` 目录下
3. **运行时启动**：应用启动时通过 Tauri 的 sidecar 机制运行 `uv run python -m valuecell.server.main`，Python 进程独立于 Tauri WebView 进程
4. **进程生命周期**：Tauri 窗口关闭时，Python 后端进程也随之终止

> **关键理解**：桌面应用内嵌的是 Python 源码和 `uv` 包管理器，**不是**编译后的二进制文件。首次启动时 `uv` 会自动安装 Python 解释器和依赖。

---

## 11. 主题系统

### 11.1 ThemeProvider

使用 `next-themes` 库管理主题切换：

```typescript
<ThemeProvider
  attribute="class"           // 通过 CSS class 切换
  defaultTheme="system"       // 默认跟随系统
  enableSystem                // 启用系统主题检测
  enableColorScheme           // 同步修改 CSS color-scheme
  storageKey="valuecell-theme" // localStorage 存储键
/>
```

### 11.2 实现方式

- 通过 `attribute="class"` 在 `<html>` 元素上添加 `dark` 或 `light` class
- Tailwind CSS 的 `dark:` 变体自动生效
- 用户偏好持久化到 `localStorage`（key: `valuecell-theme`）

### 11.3 股票颜色模式

独立的颜色模式设置（`settings-store.ts`），支持两种模式：

- **国际模式**（`GREEN_UP_RED_DOWN`）：涨绿跌红
- **中国模式**（`RED_UP_GREEN_DOWN`）：涨红跌绿

通过 `useStockColors()`、`useStockGradientColors()`、`useStockBadgeColors()` 等 Hook 在图表和列表中统一使用。

---

## 12. 开发命令参考

### 12.1 日常开发

```bash
cd frontend

bun install           # 安装依赖
bun run dev           # 启动开发服务器（端口 1420）
bun run dev:tauri     # Tauri 开发模式（启用文件轮询）
```

### 12.2 代码质量

```bash
bun run typecheck     # TypeScript 类型检查（先执行 react-router typegen）
bun run lint          # Biome lint 检查（error 级别）
bun run lint:fix      # Biome lint 自动修复
bun run format        # Biome 格式检查
bun run format:fix    # Biome 格式化
bun run check         # Biome 综合检查
bun run check:fix     # Biome 综合修复
```

### 12.3 构建与 Tauri

```bash
bun run build         # 生产构建
bun run start         # 预览生产构建
bun run tauri         # Tauri CLI 代理
```

### 12.4 注意事项

- 代码检查工具使用 **Biome**（非 ESLint），配置在 `biome.json` 中
- `typecheck` 会先执行 `react-router typegen` 自动生成路由类型
- `package.json` 中 `overrides` 将 `vite` 替换为 `rolldown-vite`（Rust 实现的 Vite，构建更快）
- 前端当前没有自动化测试脚本，校验主要依赖 `build`、`typecheck`、`lint`、`check`

---

## 13. 常见问题

### 前端如何判断运行在 Tauri 还是浏览器环境？

前端通过 `@tauri-apps/api/core` 提供的 `isTauri()` 函数判断当前运行环境。该函数检测 `window.__TAURI_INTERNALS__` 是否存在，如果存在则说明代码运行在 Tauri WebView 中。在整个前端代码中，`isTauri()` 的返回值控制着以下功能的开关：自动更新检查（`AutoUpdateCheck`）、原生存储适配器（`TauriStoreState`）、使用追踪上报、以及 deep-link 处理。浏览器环境下这些功能会被自动跳过。

### 为什么用 Zustand 而不是 Redux？

ValueCell 选择 Zustand 主要基于以下考量：第一，Zustand 的打包体积远小于 Redux（约 1KB vs 约 7KB），对于桌面应用而言轻量级的状态管理已经足够；第二，Zustand 原生支持 `persist` 中间件，可以零配置将用户设置等状态持久化到 `localStorage` 或 Tauri 原生存储，无需额外引入 `redux-persist`；第三，ValueCell 的服务端状态由 TanStack Query 管理，Zustand 只负责本地持久化状态（用户设置、认证信息、实时对话），职责范围有限，Redux 的中间件生态和严格的单向数据流模式在这个规模下并非必需。

### 为什么用 Biome 而不是 ESLint？

Biome 是一个基于 Rust 编写的前端工具链，相比 ESLint 有两个核心优势：第一，速度显著更快，由于使用 Rust 实现，lint 和格式化的执行速度比基于 JavaScript 的 ESLint + Prettier 快一个数量级，在大型项目中体感尤为明显；第二，Biome 将代码检查（lint）和格式化（format）统一为一个工具，避免了 ESLint 和 Prettier 之间规则冲突的问题。ValueCell 的 `biome.json` 配置同时覆盖了 lint 规则和格式化规则，开发者只需执行 `bun run check:fix` 即可一次性完成所有代码质量修复。

### 前端如何处理 SSE 连接断开？

SSE 连接断开的处理由 `useSSE` Hook 统一管理。当 `SSEClient` 检测到连接错误或流式读取异常时，会触发 `onError` 回调，`useSSE` 随后将 `isStreaming` 状态设为 `false`，前端界面会相应地显示连接已结束的状态。`SSEClient` 本身不内置自动重连逻辑，这是有意为之的设计决策：因为每次 Agent 对话都是一个独立的 POST 请求，断开后用户可以通过界面上的发送按钮重新发起对话，避免在未确认的情况下重复发送请求。对话的历史记录通过 TanStack Query 的 API 调用加载，即使 SSE 中断也不会丢失已有的对话内容。

### 如何添加新的国际化语言？

添加新语言需要以下步骤：首先，在 `frontend/src/i18n/locales/` 目录下创建新的翻译文件（例如 `ko.json`），可以复制 `en.json` 作为模板进行翻译；其次，在 `frontend/src/i18n/index.ts` 中导入新语言资源并添加到 i18next 的 `resources` 配置中；最后，在 `frontend/src/store/settings-store.ts` 中更新 `language` 的类型定义（在 `"en" | "zh_CN" | "zh_TW" | "ja"` 中追加新语言标识），并在 `getLanguage()` 函数的映射表中添加对应的浏览器语言映射。完成这三步后，设置页面中的语言选择器就会自动出现新语言选项。

---

## 14. 自测检查清单

阅读完本文后，可以用以下检查清单验证自己的理解程度。如果每一条都能清晰回答，说明你已经掌握了 ValueCell 前端架构的核心要点。

1. **全局 Provider 注入顺序**：能否按顺序列出 `root.tsx` 中所有全局 Provider（QueryClientProvider → ThemeProvider → BackendHealthCheck → TrackerProvider → SidebarProvider → AutoUpdateCheck），并说明每一层的职责和为什么是这个顺序？

2. **Zustand 与 TanStack Query 的状态边界**：能否清晰解释哪些状态由 Zustand 管理（用户设置、认证信息、实时对话数据），哪些由 TanStack Query 管理（API 缓存、服务端数据），以及为什么不全部用同一个方案？

3. **SSE 客户端实现细节**：能否说明为什么 ValueCell 不使用浏览器原生的 `EventSource`（只支持 GET、不支持自定义 Header），转而使用基于 `fetch + ReadableStream` 的自定义 `SSEClient`，以及 `best-effort JSON 解析` 解决了什么问题？

4. **8 个渲染器**：能否列出全部 8 个渲染器（MarkdownRenderer、ReasoningRenderer、ToolCallRenderer、ReportRenderer、ScheduledTaskRenderer、ScheduledTaskControllerRenderer、ChatConversationRenderer、UnknownRenderer），并说明每个渲染器对应的后端事件类型？

5. **TauriStoreState 双模适配**：能否描述 `tauri-store-state.ts` 如何在 Tauri 环境下使用 `@tauri-apps/plugin-store` 进行原生存储，在浏览器环境下自动降级为 `localStorage`，以及写操作为什么需要 1 秒防抖？

6. **QueryClient 配置依据**：能否解释 `staleTime: 5 分钟`（平衡数据新鲜度与请求频率）、`gcTime: 30 分钟`（给用户足够时间回到之前页面而不丢失缓存）、`refetchOnWindowFocus: false`（桌面应用不需要窗口聚焦重取）、`retry: 1`（桌面应用场景下减少用户等待）这几项配置的实际考量？

7. **agent-store.ts 的 SSE 事件处理**：能否说明 `processSSEEvent()` 如何根据事件类型选择不同的更新策略——`message_chunk` 用 append（追加文本），`tool_call_started/completed` 用 replace（整体替换），`component_generator` 按子类型分发——以及 `mutative` 库在其中扮演的角色？

8. **i18n 语言检测流程**：能否描述从 `navigator.language` 到最终显示语言的完整路径：`getLanguage()` 读取浏览器语言 → 通过映射表转换为应用语言标识（如 `zh-Hans` → `zh_CN`）→ 无匹配则使用默认 `en` → `setLanguage()` 调用 `i18n.changeLanguage()` 切换 → React 组件通过 `useTranslation()` 的 `t()` 函数获取翻译文本？

---

## 15. 继续阅读

| 你想做什么 | 推荐阅读 |
|-----------|----------|
| 理解后端如何配合前端流式通信 | [系统架构详解](./architecture.md) |
| 从源码层面理解前端入口文件 | [源码导读](./source-guide.md) |
| 理解后端事件如何被生产和路由 | [系统架构详解 - 事件处理链路](./architecture.md) |
| 自己开发新的 Agent | [Agent 开发与扩展](./agent-extension.md) |
| 了解配置系统 | [配置说明](./configuration.md) |
| 系统化学习路径 | [学习路径](./learning-path.md) |
