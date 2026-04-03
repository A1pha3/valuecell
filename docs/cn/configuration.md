# ValueCell 配置系统详解

## 学习目标

阅读本文后，你将能够：

- 说出配置系统的三层优先级结构及其设计意图
- 区分系统目录 `.env`、YAML 配置、环境变量各自的适用场景
- 独立完成模型提供商的 API Key 配置
- 理解 Agent YAML 与 Agent Card 的区别与联系
- 排查常见的配置类问题

---

## 1. 配置系统概览

### 1.1 为什么需要三层配置

ValueCell 面向多种使用场景：本地开发、桌面客户端部署、团队共享。如果所有配置都写死在一个文件里，很难同时满足"开发者需要灵活切换参数"和"普通用户只想填一个 API Key"这两种需求。

因此，项目采用**三层优先级配置**，高优先级覆盖低优先级：

| 优先级 | 来源 | 适用场景 | 修改方式 |
|--------|------|----------|----------|
| **最高** | 运行时环境变量 | 临时覆盖、CI/CD、A/B 测试 | `export VAR=value` |
| **中** | 系统目录中的 `.env` 文件 | 用户级持久化配置（API Key 等） | 编辑系统目录下的 `.env` |
| **最低** | `python/configs/` 下的 YAML 文件 | 系统默认值、版本控制 | 编辑仓库中的 YAML |

这三层的关系可以理解为：**YAML 提供合理的默认值，`.env` 提供用户的个性化设置，环境变量提供临时覆盖能力。**

### 1.2 一个直观的例子

假设 `openrouter.yaml` 中默认模型是 `anthropic/claude-haiku-4.5`：

```yaml
# python/configs/providers/openrouter.yaml
default_model: "anthropic/claude-haiku-4.5"
```

你在 `.env` 中设置了：

```bash
RESEARCH_AGENT_MODEL_ID=google/gemini-2.5-flash
```

那么研究 Agent（Research Agent）最终使用的模型是 `google/gemini-2.5-flash`，因为 `.env` 的优先级高于 YAML 默认值。

如果你进一步在终端执行：

```bash
export RESEARCH_AGENT_MODEL_ID=anthropic/claude-3.5-sonnet
```

那么运行时使用的是 `anthropic/claude-3.5-sonnet`，因为环境变量优先级最高。

---

## 2. 最重要的事实：`.env` 不在仓库根目录

这是新手最容易踩的坑。

### 2.1 当前实现的行为

后端启动时，`python/valuecell/server/api/app.py` 中的 `_ensure_system_env_and_load()` 会执行以下逻辑：

1. 计算当前操作系统的系统应用目录路径
2. 检查该目录下是否存在 `.env`
3. 如果不存在，且仓库根目录存在 `.env.example`，则将 `.env.example` 复制到系统目录
4. 加载系统目录中的 `.env`
5. **不会创建或加载仓库根目录的 `.env`**

### 2.2 系统应用目录路径

路径规则由 `python/valuecell/utils/env.py` 定义：

| 操作系统 | 路径 |
|----------|------|
| macOS | `~/Library/Application Support/ValueCell/` |
| Linux | `~/.config/valuecell/` |
| Windows | `%APPDATA%\ValueCell\` |

### 2.3 系统目录里通常有什么

```
~/Library/Application Support/ValueCell/    # 以 macOS 为例
├── .env                  # 用户配置（API Key 等）
├── valuecell.db          # SQLite 数据库
├── lancedb/              # 向量数据库
└── .knowledge/           # 知识库数据
```

### 2.4 为什么要这样设计

- **代码与数据分离**：升级代码（`git pull`）时，本地数据不受影响
- **安全**：敏感信息（API Key、数据库）不在代码仓库中，不会被意外提交
- **桌面应用友好**：符合各操作系统对应用数据存储的惯例

### 2.5 `.env.example` 的角色

仓库根目录的 `.env.example` 是一个**模板文件**，列出所有可配置的环境变量及其说明。首次启动时，系统会把它复制到系统目录并重命名为 `.env`，供用户编辑。

---

## 3. 配置文件目录结构

配置相关的文件分布在两个位置：

### 3.1 仓库中的配置文件

```
python/configs/
├── config.yaml                         # 主配置文件（全局默认）
├── providers/                          # 模型提供商配置
│   ├── openrouter.yaml                 # OpenRouter
│   ├── siliconflow.yaml                # SiliconFlow
│   ├── google.yaml                     # Google
│   ├── openai.yaml                     # OpenAI
│   ├── azure.yaml                      # Azure OpenAI
│   ├── deepseek.yaml                   # DeepSeek
│   ├── dashscope.yaml                  # 阿里云百炼（DashScope）
│   ├── ollama.yaml                     # Ollama 本地模型
│   └── openai-compatible.yaml          # 兼容 OpenAI 接口的提供商
├── agents/                             # Agent 运行配置
│   ├── super_agent.yaml                # Super Agent
│   ├── research_agent.yaml             # 研究 Agent
│   └── news_agent.yaml                 # 新闻 Agent
├── agent_cards/                        # Agent 发现与展示配置
│   ├── investment_research_agent.json  # 研究 Agent 卡片
│   ├── news_agent.json                 # 新闻 Agent 卡片
│   ├── grid_agent.json                 # 网格策略 Agent 卡片
│   └── prompt_strategy_agent.json      # 提示策略 Agent 卡片
└── locales/                            # 国际化资源
```

### 3.2 系统目录中的配置文件

```
~/Library/Application Support/ValueCell/  # 以 macOS 为例
└── .env                                 # 用户级环境变量
```

---

## 4. 模型提供商配置

### 4.1 支持的提供商

ValueCell 当前支持 9 个模型提供商：

| 提供商 | 配置文件 | 说明 |
|--------|----------|------|
| OpenRouter | `openrouter.yaml` | 聚合多家模型，推荐首选 |
| SiliconFlow | `siliconflow.yaml` | 国内模型，性价比高 |
| Google | `google.yaml` | Gemini 系列模型 |
| OpenAI | `openai.yaml` | GPT 系列 |
| Azure | `azure.yaml` | Azure OpenAI |
| DeepSeek | `deepseek.yaml` | DeepSeek 系列 |
| DashScope | `dashscope.yaml` | 阿里云百炼（通义千问） |
| Ollama | `ollama.yaml` | 本地部署模型 |
| OpenAI-Compatible | `openai-compatible.yaml` | 兼容 OpenAI 接口的自定义服务 |

### 4.2 提供商配置文件结构

每个提供商的 YAML 文件包含以下关键字段：

```yaml
# python/configs/providers/openrouter.yaml（示例结构）
name: "OpenRouter"
provider_type: "openrouter"
enabled: true

# 连接信息
connection:
  base_url: "https://openrouter.ai/api/v1"
  api_key_env: "OPENROUTER_API_KEY"     # 指定读取哪个环境变量

# 默认模型
default_model: "anthropic/claude-haiku-4.5"

# 模型参数默认值
defaults:
  temperature: 0.5
  max_tokens: 4096

# 可用模型列表
models:
  - id: "anthropic/claude-haiku-4.5"
    name: "Claude Haiku 4.5"
    context_length: 200000
  # ...更多模型

# 提供商特有的请求头
extra_headers:
  HTTP-Referer: "https://valuecell.ai"
  X-Title: "ValueCell"
```

其中 `api_key_env` 字段指定了从哪个环境变量读取 API Key。系统启动时，会按以下顺序查找：

1. 运行时环境变量（`export OPENROUTER_API_KEY=...`）
2. 系统目录 `.env` 文件中的 `OPENROUTER_API_KEY=...`
3. 如果都找不到，该提供商不可用

### 4.3 首次配置：设置 API Key

首次使用时，至少需要配置一个提供商的 API Key。

**推荐方案**：选择 OpenRouter（模型最多）或 SiliconFlow（国内友好）。

操作步骤：

1. 找到系统目录下的 `.env` 文件（参见第 2.2 节路径）
2. 用文本编辑器打开，添加对应的 Key：

```bash
# 选择一个或多个提供商
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx
# SILICONFLOW_API_KEY=sk-xxxxxxxxxxxxx
# GOOGLE_API_KEY=AIzaSyDxxxxxxxxxxxxx
# DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxx
```

3. 保存文件，重启后端生效

### 4.4 自动检测与优先级

当多个提供商都配置了 API Key 时，系统会按以下顺序自动选择主提供商：

1. OpenRouter
2. SiliconFlow
3. Google
4. OpenAI
5. OpenAI-Compatible
6. Azure
7. 其他已配置的提供商

你也可以手动指定：

```bash
# 在 .env 中设置
PRIMARY_PROVIDER=siliconflow

# 或通过环境变量临时覆盖
export PRIMARY_PROVIDER=google
```

### 4.5 提供商降级（Fallback）机制

如果主提供商请求失败，系统会自动尝试其他已配置的提供商，直到成功为止。

你可以在 Agent 配置中为不同提供商指定不同的模型（因为不同提供商支持的模型不同）：

```yaml
# 在 agent 的 YAML 中
models:
  primary:
    model_id: "anthropic/claude-haiku-4.5"
    provider: "openrouter"
    provider_models:                    # 降级时的模型映射
      siliconflow: "Qwen/Qwen3-235B-A22B-Thinking-2507"
      google: "gemini-2.5-flash"
```

降级时的行为：

1. 先用 OpenRouter + `anthropic/claude-haiku-4.5` 尝试
2. 失败后，用 SiliconFlow + `Qwen/Qwen3-235B-A22B-Thinking-2507` 尝试
3. 再失败，用 Google + `gemini-2.5-flash` 尝试

---

## 5. Agent 配置

### 5.1 Agent YAML 与 Agent Card 的区别

这是两个非常容易混淆的概念，必须严格区分：

| | Agent YAML | Agent Card |
|---|---|---|
| **位置** | `python/configs/agents/` | `python/configs/agent_cards/` |
| **格式** | YAML | JSON |
| **用途** | 定义运行时参数（模型、温度等） | 定义服务发现与能力声明 |
| **影响** | Agent 用什么模型、什么参数 | 系统如何发现、展示、连接这个 Agent |
| **命名** | 文件名与 Agent 模块名一致 | `name` 字段必须与 Agent 类名一致 |

**如果只改了 YAML 没改 Card**：系统可能发现了 Agent，但连接参数不对。
**如果只改了 Card 没改 YAML**：系统可能根本发现不了这个 Agent。

### 5.2 Agent YAML 详解

以研究 Agent 为例（`python/configs/agents/research_agent.yaml`）：

```yaml
name: "Research Agent"
enabled: true

# 模型配置
models:
  # 主模型
  primary:
    model_id: "google/gemini-2.5-flash"       # 默认模型
    provider: "openrouter"                      # 提供商
    provider_models:                            # 各提供商的对应模型
      siliconflow: "Qwen/Qwen3-235B-A22B-Thinking-2507"
      google: "gemini-2.5-flash"
    parameters:                                 # 模型参数（覆盖提供商默认值）
      temperature: 0.7

# 环境变量映射：定义哪些环境变量可以覆盖哪些配置路径
env_overrides:
  RESEARCH_AGENT_MODEL_ID: "models.primary.model_id"
  RESEARCH_AGENT_PROVIDER: "models.primary.provider"
```

`env_overrides` 是一个关键机制——它定义了"哪个环境变量覆盖哪个配置路径"。这意味着你可以在不修改 YAML 文件的情况下，通过环境变量动态切换 Agent 使用的模型：

```bash
# 临时让研究 Agent 使用不同的模型
export RESEARCH_AGENT_MODEL_ID="anthropic/claude-3.5-sonnet"
export RESEARCH_AGENT_PROVIDER="openrouter"
```

### 5.3 Agent Card 详解

Agent Card（智能体卡片）是系统发现和展示 Agent 的依据。

当前仓库中的 Agent Card 文件：

| 文件名 | `name` 字段（类名） | `url` | 说明 |
|--------|---------------------|-------|------|
| `investment_research_agent.json` | `ResearchAgent` | `http://localhost:10004/` | SEC 文件分析与知识库研究 |
| `news_agent.json` | `NewsAgent` | `http://localhost:10005/` | 实时新闻检索与市场动态 |
| `grid_agent.json` | `GridStrategyAgent` | `http://localhost:10007/` | 网格交易策略 |
| `prompt_strategy_agent.json` | `PromptBasedStrategyAgent` | `http://localhost:10006/` | LLM 驱动的策略生成 |

Agent Card 的关键字段：

```json
{
  "name": "ResearchAgent",                    // 必须与 Agent 类名一致
  "display_name": "Research Agent",           // 前端展示名
  "url": "http://localhost:10004/",           // 服务监听地址与端口
  "description": "...",                        // Agent 描述
  "capabilities": { "streaming": true },       // 能力声明
  "skills": [                                  // 能力列表（影响 Planner 决策）
    {
      "id": "extract_financials",
      "name": "Extract financial line items",
      "description": "...",
      "examples": ["..."],
      "tags": ["10-K", "10-Q"]
    }
  ],
  "enabled": true,
  "metadata": {
    "version": "1.0.0",
    "local_agent_class": "valuecell.agents.research_agent.core:ResearchAgent"
  }
}
```

### 5.4 Agent Card 最容易出错的地方

#### `name` 字段

`name` 必须与 Python 类名完全一致。例如 `ResearchAgent` 对应 `research_agent/core.py` 中的 `class ResearchAgent`。如果不一致，系统将无法完成 Agent 的发现与绑定。

#### `url` 字段

`url` 决定 Agent 服务绑定的地址与端口。多个 Agent 同时运行时，端口不能冲突。从上表可以看到，当前端口分配为 10004-10007。

#### `metadata.planner_passthrough`

部分 Agent Card 的 `metadata` 中有 `planner_passthrough: true`（如策略类 Agent）。这表示 Planner 不会对这些 Agent 做完整的规划推理，而是直接生成一个单任务计划，把用户请求原样转发。这是一个重要的架构细节。

---

## 6. 全局配置文件

主配置文件 `python/configs/config.yaml` 定义了系统级默认值：

```yaml
# 模型相关
models:
  # 主提供商（当多个提供商都有 API Key 时的首选）
  primary_provider: "openrouter"

  # 全局默认参数（所有模型都适用，除非被覆盖）
  defaults:
    temperature: 0.5
    max_tokens: 4096

  # 提供商注册表
  providers:
    openrouter:
      config_file: "providers/openrouter.yaml"
      api_key_env: "OPENROUTER_API_KEY"
    siliconflow:
      config_file: "providers/siliconflow.yaml"
      api_key_env: "SILICONFLOW_API_KEY"
    google:
      config_file: "providers/google.yaml"
      api_key_env: "GOOGLE_API_KEY"
    # ...更多提供商

# Agent 注册表
agents:
  super_agent:
    config_file: "agents/super_agent.yaml"
  research_agent:
    config_file: "agents/research_agent.yaml"
```

---

## 7. 环境变量参考

### 7.1 全局配置

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `PRIMARY_PROVIDER` | 自动检测 | 主提供商名称 |
| `AUTO_DETECT_PROVIDER` | `true` | 是否自动检测可用的提供商 |
| `FALLBACK_PROVIDERS` | 自动填充 | 降级提供商列表（逗号分隔） |
| `APP_ENVIRONMENT` | — | 应用环境（`production` 等） |
| `AGENT_DEBUG_MODE` | `false` | 调试模式（开启后输出详细日志） |

### 7.2 提供商凭据

| 环境变量 | 提供商 | 注册地址 |
|----------|--------|----------|
| `OPENROUTER_API_KEY` | OpenRouter | [openrouter.ai](https://openrouter.ai/) |
| `SILICONFLOW_API_KEY` | SiliconFlow | [siliconflow.cn](https://www.siliconflow.cn/) |
| `GOOGLE_API_KEY` | Google | [ai.google.dev](https://ai.google.dev/) |
| `OPENAI_API_KEY` | OpenAI | [platform.openai.com](https://platform.openai.com/) |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI | Azure 门户 |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI | Azure 门户 |
| `DEEPSEEK_API_KEY` | DeepSeek | [platform.deepseek.com](https://platform.deepseek.com/) |
| `DASHSCOPE_API_KEY` | 阿里云百炼 | [bailian.console.aliyun.com](https://bailian.console.aliyun.com/) |

### 7.3 Agent 级配置

| 环境变量 | 影响范围 | 说明 |
|----------|----------|------|
| `RESEARCH_AGENT_MODEL_ID` | 研究 Agent | 覆盖默认模型 ID |
| `RESEARCH_AGENT_PROVIDER` | 研究 Agent | 覆盖默认提供商 |
| `RESEARCH_AGENT_TEMPERATURE` | 研究 Agent | 覆盖温度参数 |
| `RESEARCH_AGENT_MAX_TOKENS` | 研究 Agent | 覆盖最大 Token 数 |
| `SUPER_AGENT_MODEL_ID` | Super Agent | 覆盖默认模型 ID |
| `SUPER_AGENT_PROVIDER` | Super Agent | 覆盖默认提供商 |
| `NEWS_AGENT_MODEL_ID` | 新闻 Agent | 覆盖默认模型 ID |
| `NEWS_AGENT_PROVIDER` | 新闻 Agent | 覆盖默认提供商 |

### 7.4 交易所配置（OKX）

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `OKX_NETWORK` | `paper` | `paper` 为模拟盘，`mainnet` 为实盘 |
| `OKX_API_KEY` | — | OKX API Key |
| `OKX_API_SECRET` | — | OKX API Secret |
| `OKX_API_PASSPHRASE` | — | API 密码 |
| `OKX_ALLOW_LIVE_TRADING` | `false` | 必须设为 `true` 才能路由到实盘 |
| `OKX_MARGIN_MODE` | `cash` | 交易模式：`cash`、`cross`、`isolated` |

> **重要**：在策略通过模拟盘验证之前，请保持 `OKX_ALLOW_LIVE_TRADING=false`。API 密钥属于生产凭据，务必妥善保管。

---

## 8. 配置加载的内部机制

### 8.1 三层加载器

配置系统内部由三层组成：

1. **Loader 层**（`valuecell/config/loader.py`）：读取 YAML 文件，解析 `${VAR}` 占位符，应用环境变量覆盖，实现缓存
2. **Manager 层**（`valuecell/config/manager.py`）：高层配置访问接口，提供商验证，模型工厂集成，降级链管理
3. **Factory 层**（`valuecell/adapters/models/factory.py`）：创建实际模型实例，提供商特定实现，参数合并，错误处理与降级

### 8.2 配置解析流程

当系统需要一个模型时：

1. 读取提供商 YAML（如 `providers/openrouter.yaml`）
2. 解析其中的 `${VAR}` 占位符（替换为环境变量值）
3. 应用环境变量覆盖（如 `OPENROUTER_API_KEY` 覆盖 `connection.api_key`）
4. 返回解析后的 `ProviderConfig` 对象

当系统需要一个 Agent 的配置时：

1. 读取 Agent YAML（如 `agents/research_agent.yaml`）
2. 应用 `env_overrides` 中定义的环境变量映射
3. 与全局默认值（`config.yaml`）合并
4. 返回完整的 `AgentConfig` 对象

---

## 9. 配置排错指南

### 9.1 排错顺序

遇到配置问题时，按以下顺序排查：

**第一步：确认 `.env` 位置**

检查系统目录（不是仓库根目录）中的 `.env` 是否存在且包含你期望的内容：

```bash
# macOS
cat ~/Library/Application\ Support/ValueCell/.env

# Linux
cat ~/.config/valuecell/.env
```

**第二步：确认后端启动时加载成功**

`.env` 的加载发生在 FastAPI app 创建阶段（`app.py` 的 `create_app()`），所以应优先查看后端启动日志。

**第三步：确认 Agent Card 和 Agent YAML 同时存在**

很多"Agent 不工作"的问题不是因为代码有 bug，而是配置缺失：

- YAML 文件是否存在？
- Agent Card 文件是否存在？
- Card 中的 `name` 是否与类名一致？
- Card 中的 `url` 端口是否冲突？

**第四步：区分前端与后端问题**

前端"展示不到内容"和后端"没有产出内容"是两类不同问题，不要混在一起排查。

### 9.2 常见问题

#### 改了仓库根目录的 `.env` 没生效

当前后端使用的是**系统目录**中的 `.env`，不是仓库根目录。请修改系统目录中的文件。

#### 数据库兼容性错误

如果长时间未更新导致数据库格式不兼容，可以清理系统目录中的以下文件后重启：

- `lancedb/`
- `.knowledge/`
- `valuecell.db`

系统会在下次启动时自动重建。

#### 多个 Agent 端口冲突

检查各 Agent Card 中的 `url` 字段，确保端口不重复。当前默认分配：

- 10004：ResearchAgent
- 10005：NewsAgent
- 10006：PromptBasedStrategyAgent
- 10007：GridStrategyAgent

---

## 10. 配置模式参考

### 模式一：多提供商高可用

```bash
# .env
OPENROUTER_API_KEY=sk-or-v1-xxxxx        # 主提供商，模型最全
SILICONFLOW_API_KEY=sk-xxxxx             # 降级方案，性价比高
GOOGLE_API_KEY=AIzaSyD-xxxxx             # 第二降级
```

### 模式二：按 Agent 选择不同模型

```yaml
# research_agent.yaml
models:
  primary:
    provider: "openrouter"
    model_id: "anthropic/claude-3.5-sonnet"  # 研究用强模型
```

### 模式三：运行时临时切换

```bash
# A/B 测试不同模型，不修改任何文件
RESEARCH_AGENT_MODEL_ID="gpt-4o" bash start.sh
```

---

## 11. 继续阅读

- 想理解配置加载后如何进入运行时：看 [architecture.md](./architecture.md)
- 想知道源码入口和配置相关模块在哪：看 [source-guide.md](./source-guide.md)
- 想新增自定义 Agent：看 [agent-extension.md](./agent-extension.md)
- 想先把系统跑起来：看 [getting-started.md](./getting-started.md)
