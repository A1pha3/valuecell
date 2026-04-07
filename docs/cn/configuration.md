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

假设 `openrouter.yaml` 中默认模型是 `qwen/qwen3-max`：

```yaml
# python/configs/providers/openrouter.yaml
default_model: "qwen/qwen3-max"
```

你在 `.env` 中设置了：

```bash
RESEARCH_AGENT_MODEL_ID=google/gemini-2.5-flash
```

那么研究 Agent（Research Agent）最终使用的模型是 `google/gemini-2.5-flash`，因为 `.env` 的优先级高于 YAML 默认值。

如果你进一步在终端执行：

```bash
export RESEARCH_AGENT_MODEL_ID=anthropic/claude-sonnet-4.5
```

那么运行时使用的是 `anthropic/claude-sonnet-4.5`，因为环境变量优先级最高。

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
default_model: "qwen/qwen3-max"

# 模型参数默认值
defaults:
  temperature: 0.5
  max_tokens: 4096

# 嵌入模型默认值
embedding:
  default_model: "qwen/qwen3-embedding-4b"

# 可用模型列表
models:
  - id: "qwen/qwen3-max"
    name: "Qwen3 Max"
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

> **最佳实践**：如果只打算配置一个 API Key，推荐选择 OpenRouter。`OPENROUTER_API_KEY` 是通用性最强的选项——OpenRouter 聚合了 OpenAI、Anthropic、Google、Meta 等多家模型提供商，一个 Key 即可访问上百种模型。这意味着你可以通过切换模型 ID（如 `anthropic/claude-sonnet-4.5`、`google/gemini-2.5-flash`）来使用不同厂商的模型，而无需分别注册和配置多个提供商。

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

> **最佳实践**：`env_overrides` 的设计使得"零文件变更切换模型"成为可能。在团队协作或 CI/CD 场景中，你可以保留 YAML 文件不变，仅通过环境变量注入来控制不同环境使用的模型。例如：开发环境使用 `google/gemini-2.5-flash`（成本低），生产环境使用 `anthropic/claude-sonnet-4.5`（质量高），两者共享同一份代码和配置文件。

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

> **最佳实践**：在对 `.env` 文件做重大修改之前（如批量更换 API Key、切换提供商配置），建议先备份当前文件。这样可以快速回滚到已验证可用的配置状态：
>
> ```bash
> # macOS
> cp ~/Library/Application\ Support/ValueCell/.env ~/Library/Application\ Support/ValueCell/.env.backup
>
> # Linux
> cp ~/.config/valuecell/.env ~/.config/valuecell/.env.backup
> ```
>
> 如需恢复，将 `.env.backup` 重命名回 `.env` 即可。

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
| `AZURE_OPENAI_API_VERSION` | Azure OpenAI | API 版本（默认 `2024-10-21`） |
| `DEEPSEEK_API_KEY` | DeepSeek | [platform.deepseek.com](https://platform.deepseek.com/) |
| `DASHSCOPE_API_KEY` | 阿里云百炼 | [bailian.console.aliyun.com](https://bailian.console.aliyun.com/) |
| `OPENAI_COMPATIBLE_API_KEY` | 兼容接口 | 自定义 |
| `OPENAI_COMPATIBLE_BASE_URL` | 兼容接口 | 自定义 API 地址 |

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

### 7.4 特殊用途配置

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `SEC_EMAIL` | — | Research Agent 访问 SEC EDGAR 数据时使用的邮箱标识（User-Agent header） |
| `LANG` | `en` | 界面语言（`en`、`zh_CN`、`zh_TW`、`ja`），主要用于前端 |
| `TIMEZONE` | `America/New_York` | 时区设置 |
| `API_HOST` | `localhost` | API 服务绑定地址 |
| `API_PORT` | `8000` | API 服务端口 |

### 7.5 交易所配置（OKX）

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `OKX_NETWORK` | `paper` | `paper` 为模拟盘，`mainnet` 为实盘 |
| `OKX_API_KEY` | — | OKX API Key |
| `OKX_API_SECRET` | — | OKX API Secret |
| `OKX_API_PASSPHRASE` | — | API 密码 |
| `OKX_ALLOW_LIVE_TRADING` | `false` | 必须设为 `true` 才能路由到实盘 |
| `OKX_MARGIN_MODE` | `cash` | 交易模式：`cash`、`cross`、`isolated` |
| `OKX_USE_SERVER_TIME` | `false` | 启用后使用 OKX 服务器时间戳订单 |

> **重要**：在策略通过模拟盘验证之前，请保持 `OKX_ALLOW_LIVE_TRADING=false`。API 密钥属于生产凭据，务必妥善保管。

---

## 8. 配置加载的内部机制

### 8.1 三层加载器

配置系统内部由三层组成：

1. **Loader 层**（`valuecell/config/loader.py`）：读取 YAML 文件，解析 `${VAR}` 占位符，应用环境变量覆盖，实现缓存
2. **Manager 层**（`valuecell/config/manager.py`）：高层配置访问接口，提供商验证，模型工厂集成，降级链管理
3. **Factory 层**（`valuecell/adapters/models/factory.py`）：创建实际模型实例，提供商特定实现，参数合并，错误处理与降级

### 8.2 提供商验证逻辑

`ConfigManager.validate_provider()` 按以下顺序执行 4 层校验：

| 检查序 | 条件 | 失败消息 |
|------|------|----------|
| 1. 提供商是否存在 | `get_provider_config(name)` 返回非 None | `"Provider '{name}' not found in configuration"` |
| 2. 提供商是否启用 | `provider_config.enabled == True` | `"Provider '{name}' is disabled in config"` |
| 3. API Key 是否配置 | `provider_config.api_key` 为真值（**Ollama 豁免此项检查**） | `"API key not found. Please set {env_var} in .env"` |
| 4. Azure Endpoint 是否配置 | 仅 Azure 提供商：`provider_config.base_url` 为真值 | `"Azure endpoint not configured. Please set AZURE_OPENAI_ENDPOINT"` |

> **Ollama 特殊处理**：由于 Ollama 是本地模型部署，不需要 API Key。`get_enabled_providers()` 方法在判断提供商是否可用时，对 Ollama 跳过 API Key 检查——只要 YAML 中 `enabled: true`，即视为可用。

### 8.3 配置解析流程

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

#### 模型提供商返回 401 错误

当后端日志出现 `401 Unauthorized` 或 `Invalid API key` 时，说明 API Key 无效或已过期。按以下步骤排查：

1. 确认 Key 格式正确——不同提供商的 Key 前缀不同（如 OpenRouter 以 `sk-or-v1-` 开头，SiliconFlow 以 `sk` 开头）
2. 检查 Key 是否被意外截断（复制时可能遗漏末尾字符）
3. 前往对应提供商的控制台确认 Key 是否仍处于有效状态，部分平台支持查看 Key 的最后使用时间
4. 如果 Key 已过期或被撤销，重新生成并替换系统目录 `.env` 中的对应值，然后重启后端

常见误区：在 `.env` 中给 Key 值加了引号。大多数情况下 `OPENROUTER_API_KEY=sk-or-v1-xxxxx` 不需要引号；如果 Key 中包含特殊字符（如 `#`），则需要用引号包裹。

#### 切换提供商后 Agent 行为未变化

修改 `.env` 中的 `PRIMARY_PROVIDER` 或 API Key 后，如果没有重启后端服务，更改不会生效。排查步骤：

1. 确认已完全停止并重启后端（不是热重载）
2. 检查是否有运行时环境变量覆盖了 `.env` 中的设置——运行时环境变量的优先级更高，会掩盖 `.env` 的修改：

```bash
# 检查当前终端是否存在覆盖
env | grep PRIMARY_PROVIDER
env | grep OPENROUTER_API_KEY
```

3. 如果使用了 Agent 级别的 `env_overrides`（如 `RESEARCH_AGENT_PROVIDER`），即使切换了全局提供商，该 Agent 仍然使用覆盖值。需要同步修改或清除对应的 Agent 级环境变量
4. 浏览器前端可能缓存了旧的模型信息，尝试清除浏览器缓存或使用无痕模式访问

#### YAML 修改后不生效

直接编辑 `python/configs/` 下的 YAML 文件后没有效果，通常是因为三层优先级中有更高层覆盖了你的修改。按以下步骤确认：

1. **检查 `.env` 是否存在同名配置**：`.env` 的优先级高于 YAML。例如 YAML 中设置了 `default_model: "qwen/qwen3-max"`，但 `.env` 中有 `RESEARCH_AGENT_MODEL_ID=google/gemini-2.5-flash`，最终生效的是 `.env` 中的值
2. **检查运行时环境变量**：通过 `export` 设置的环境变量优先级最高，会同时覆盖 YAML 和 `.env`
3. **确认修改的是正确的文件**：系统从 `python/configs/` 目录加载 YAML，而不是其他位置的同名文件

理解三层优先级（环境变量 > `.env` > YAML）是解决此类问题的关键。当你修改 YAML 但不生效时，逐层向上排查是否存在更高优先级的覆盖。

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

## 11. 自测检查清单

完成本文阅读后，确认以下要点：

- [ ] 我能说出三层配置优先级（环境变量 > `.env` > YAML）及其设计意图
- [ ] 我知道 `.env` 文件在系统目录，不在仓库根目录
- [ ] 我成功配置了至少一个模型提供商的 API Key
- [ ] 我能区分 Agent Card（JSON）和 Agent YAML 的职责差异
- [ ] 我知道如何通过环境变量临时覆盖 Agent 的模型配置
- [ ] 我理解提供商降级机制的工作方式
- [ ] 我知道数据库兼容性问题时该如何清理重建

### 练习

1. **切换提供商**：在 `.env` 中将主提供商从 OpenRouter 切换到 SiliconFlow，重启后端，观察 Agent 行为差异
2. **环境变量覆盖**：通过 `export RESEARCH_AGENT_MODEL_ID=google/gemini-2.5-flash` 临时切换研究 Agent 的模型，不修改任何文件
3. **清理重建**：备份后删除系统目录中的 `valuecell.db`，重启系统，观察数据库自动重建过程
4. **配置降级并测试降级行为**：在 `.env` 中配置至少两个提供商的 API Key（如 OpenRouter 和 SiliconFlow），然后在研究 Agent 的 YAML 中配置 `provider_models` 降级映射。测试方法：临时将主提供商的 API Key 设为无效值（如 `OPENROUTER_API_KEY=invalid`），重启后端并执行一次研究任务，观察后端日志中是否触发了降级流程、是否成功切换到备用提供商完成请求

---

## 12. 配置速查卡

以下是最常见的配置操作快速索引，帮助你快速定位需要阅读的章节和执行的操作。

| 我想...... | 操作 | 参考章节 |
|------------|------|----------|
| 首次填入 API Key | 编辑系统目录 `.env`，添加 `OPENROUTER_API_KEY=sk-or-v1-xxxxx`，重启后端 | 第 4.3 节 |
| 切换默认模型提供商 | 在 `.env` 中设置 `PRIMARY_PROVIDER=siliconflow`，重启后端 | 第 4.4 节 |
| 临时测试另一个模型（不改文件） | 终端执行 `export RESEARCH_AGENT_MODEL_ID=google/gemini-2.5-flash`，然后启动后端 | 第 5.2 节 |
| 给 Agent 配置降级模型 | 编辑 Agent YAML，在 `models.primary.provider_models` 中添加备选提供商和模型 | 第 4.5 节 |
| 找到我的 `.env` 文件 | macOS：`~/Library/Application Support/ValueCell/.env`；Linux：`~/.config/valuecell/.env` | 第 2.2 节 |
| 新增一个自定义 Agent | 同时创建 Agent YAML（`python/configs/agents/`）和 Agent Card（`python/configs/agent_cards/`），确保 `name` 字段与类名一致 | 第 5.1 节 |
| 排查"配置没生效" | 按三层优先级排查：先查运行时环境变量，再查 `.env`，最后看 YAML | 第 9.1 节 |
| 连接本地 Ollama 模型 | 确保 Ollama 服务已启动，`python/configs/providers/ollama.yaml` 中 `enabled: true`，无需 API Key | 第 4.1 节 |

---

## 13. 继续阅读

- 想理解配置加载后如何进入运行时：看 [architecture.md](./architecture.md)
- 想知道源码入口和配置相关模块在哪：看 [source-guide.md](./source-guide.md)
- 想新增自定义 Agent：看 [agent-extension.md](./agent-extension.md)
- 想先把系统跑起来：看 [getting-started.md](./getting-started.md)
- 想按阶段系统学习：看 [learning-path.md](./learning-path.md)
