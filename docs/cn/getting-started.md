# ValueCell 快速上手

## 学习目标

阅读本文后，你将能够：

- 在本地成功启动 ValueCell 完整系统
- 配置至少一个模型提供商
- 完成一次与 Agent 的对话交互
- 知道去哪里查看日志、排查问题

---

## 1. 两种使用方式

ValueCell 提供两种使用方式：

### 1.1 桌面应用（推荐新用户使用）

直接从 [GitHub Releases](https://github.com/ValueCell-ai/valuecell/releases) 下载 macOS 或 Windows 安装包。安装后在应用内配置模型提供商即可使用。

### 1.2 本地开发（面向开发者）

克隆仓库并在本地运行前后端。这也是本文重点介绍的方式。

---

## 2. 运行前先知道的事

### 2.1 技术栈概览

| 组件 | 技术 | 包管理器 |
|------|------|----------|
| 后端 | Python 3.12+、FastAPI、uvicorn | `uv` |
| 前端 | React Router v7、TypeScript、Vite | `bun` |
| 桌面打包 | **Tauri v2**（macOS / Windows） | Cargo |
| 数据存储 | SQLite、LanceDB | — |
| Agent 框架 | Agno（Agent 构建）、A2A 协议（通信） | — |

### 2.2 系统要求

- **Python**：3.12 或更高版本
- **Node.js**：不需要手动安装（`bun` 会处理）
- **操作系统**：macOS、Linux 或 Windows
- **网络**：需要访问外部 LLM API（如 OpenRouter、SiliconFlow 等）

### 2.3 关于数据存储位置

开始之前，你需要知道一个关键事实：**ValueCell 的配置和数据不在代码仓库中**。

它们存储在操作系统的应用目录中：

| 操作系统 | 路径 |
|----------|------|
| macOS | `~/Library/Application Support/ValueCell/` |
| Linux | `~/.config/valuecell/` |
| Windows | `%APPDATA%\ValueCell\` |

该目录包含：

- `.env` — 你的 API Key 等配置
- `valuecell.db` — SQLite 数据库
- `lancedb/` — 向量数据库
- `.knowledge/` — 知识库数据

> **为什么这样设计？** 将敏感数据（API Key）和用户数据与代码分离，避免在 `git pull` 时意外覆盖，也更符合桌面应用的存储惯例。详见 [configuration.md](./configuration.md)。

---

## 3. 快速启动

### 3.1 克隆仓库

```bash
git clone https://github.com/ValueCell-ai/valuecell.git
cd valuecell
```

### 3.2 一键启动

仓库根目录已提供启动脚本，会自动处理依赖安装、数据库初始化和服务启动：

**macOS / Linux：**

```bash
bash start.sh
```

**Windows PowerShell：**

```powershell
.\start.ps1
```

启动脚本会执行以下操作：

1. 检查并安装 `bun`（前端包管理器）和 `uv`（Python 包管理器）
2. 安装前后端依赖
3. 首次运行时，将 `.env.example` 复制到系统目录作为 `.env`
4. 初始化数据库
5. 启动后端服务（含所有 Agent）
6. 启动前端开发服务器

### 3.3 访问界面

启动完成后，在浏览器中打开：

```
http://localhost:1420
```

### 3.4 可选的启动参数

```bash
# 只启动后端（不启动前端）
bash start.sh --no-frontend

# 只启动前端（不启动后端）
bash start.sh --no-backend
```

---

## 4. 首次启动后的配置

首次启动后，**最关键的一步是配置模型提供商**。没有 API Key，Agent 无法工作。

### 4.1 获取 API Key

ValueCell 支持多个 LLM 提供商，选择至少一个：

| 提供商 | 特点 | 注册地址 |
|--------|------|----------|
| **OpenRouter** | 模型最全，推荐首选 | [openrouter.ai](https://openrouter.ai/) |
| **SiliconFlow** | 性价比高，适合中国用户 | [siliconflow.cn](https://www.siliconflow.cn/) |
| **Google** | Gemini 系列 | [ai.google.dev](https://ai.google.dev/) |
| **OpenAI** | GPT 系列 | [platform.openai.com](https://platform.openai.com/) |
| **DashScope** | 阿里云百炼，Qwen 系列 | [bailian.console.aliyun.com](https://bailian.console.aliyun.com/) |
| **DeepSeek** | DeepSeek 系列 | [platform.deepseek.com](https://platform.deepseek.com/) |

### 4.2 配置 API Key

有两种方式：

**方式一：通过 Web UI 配置（推荐）**

启动系统后，在界面的设置页面中直接填入 API Key。

**方式二：编辑 `.env` 文件**

编辑系统目录中的 `.env` 文件：

```bash
# macOS
nano ~/Library/Application\ Support/ValueCell/.env

# Linux
nano ~/.config/valuecell/.env
```

添加你的 API Key：

```bash
# 选择至少一个提供商
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx

# 或
SILICONFLOW_API_KEY=sk-xxxxxxxxxxxxx

# 或
GOOGLE_API_KEY=AIzaSyDxxxxxxxxxxxxx
```

> **注意**：编辑的是系统目录中的 `.env`，不是仓库根目录的。修改后需要重启后端生效。

### 4.3 提供商自动检测

系统会根据你配置的 API Key 自动选择提供商。检测优先级为：

1. OpenRouter
2. SiliconFlow
3. Google
4. OpenAI
5. OpenAI-Compatible
6. Azure
7. 其他已配置的提供商

你也可以手动指定：

```bash
# 在 .env 中添加
PRIMARY_PROVIDER=siliconflow
```

### 4.4 提供商降级机制

如果主提供商不可用，系统会自动尝试其他已配置 API Key 的提供商。你可以指定降级顺序：

```bash
FALLBACK_PROVIDERS=siliconflow,google
```

---

## 5. 分模块手动运行

如果你需要单独调试某个模块，可以手动启动。

### 5.1 后端

```bash
cd python
uv sync --group dev
uv run python -m valuecell.server.main
```

### 5.2 前端

```bash
cd frontend
bun install
bun run dev
```

### 5.3 单个 Agent

```bash
cd python
uv run python -m valuecell.agents.research_agent
```

---

## 6. 常用开发命令

### 6.1 Python / 后端

```bash
cd python

# 安装依赖（含开发依赖）
uv sync --group dev

# 运行全部测试
uv run pytest

# 运行单个测试文件
uv run pytest valuecell/core/plan/tests/test_planner.py

# 运行单个测试用例
uv run pytest valuecell/core/plan/tests/test_planner.py::test_name
```

根目录提供了快捷命令：

```bash
make format    # 格式化 Python 代码并执行 isort
make lint      # 运行 Ruff 检查
make test      # 运行 Python 测试
```

### 6.2 前端

```bash
cd frontend

bun install      # 安装依赖
bun run dev      # 启动开发服务器
bun run build    # 构建生产版本
bun run typecheck # 类型检查
bun run lint     # 代码检查
bun run format   # 格式化代码
bun run check    # 综合检查
```

> **注意**：当前前端没有自动化测试脚本，日常校验主要依赖 `build`、`typecheck`、`lint`、`check`。

### 6.3 调试模式

在 `.env` 中启用调试模式：

```bash
AGENT_DEBUG_MODE=true
```

调试模式会记录发送给模型的完整 Prompt、工具调用详情、中间步骤和提供商返回的元数据。

> **注意**：调试模式会记录敏感输入/输出并增加延迟，仅在本地开发环境使用。

---

## 7. 首次启动后的验证路径

启动并配置完成后，建议按以下最短路径验证系统是否正常工作：

### 7.1 检查终端日志

启动后先看终端输出，确认：

- 后端是否正常启动（无报错）
- 数据库是否初始化成功
- 各 Agent 是否被发现并注册
- 前端开发服务器是否正常运行

### 7.2 验证最短交互路径

1. 打开 `http://localhost:1420`
2. 进入某个 Agent 页面（如 News Agent 或 Research Agent）
3. 发起一次简单对话（如"今天有什么重要新闻？"）
4. 确认前端能持续收到流式响应
5. 确认对话结束后能看到完整内容

如果这五步都能完成，说明系统核心链路正常。

---

## 7.5 验证后的深入探索

完成基本验证后，如果你想更深入地了解系统工作方式，可以尝试以下操作：

1. **开启调试模式**：在 `.env` 中设置 `AGENT_DEBUG_MODE=true`，再发起一次对话，观察终端中的完整日志
2. **追踪一条请求**：观察后端日志，尝试对应"请求接入 → Super Agent 分流 → 规划/直通 → 执行 → 事件返回"这几个阶段
3. **阅读一个 Agent 实现**：打开 `python/valuecell/agents/news_agent/core.py`，从 `stream()` 方法开始读

---

## 8. 常见问题

### 8.1 修改了仓库根目录的 `.env` 但没生效

当前后端使用的是**系统目录**中的 `.env`，不是仓库根目录。

```bash
# 确认你编辑的是正确的文件
# macOS
ls ~/Library/Application\ Support/ValueCell/.env

# Linux
ls ~/.config/valuecell/.env
```

### 8.2 数据库兼容性错误

长时间未更新后可能出现数据库格式不兼容。解决方法：

```bash
# macOS
rm -rf ~/Library/Application\ Support/ValueCell/lancedb
rm -rf ~/Library/Application\ Support/ValueCell/.knowledge
rm ~/Library/Application\ Support/ValueCell/valuecell.db

# Linux
rm -rf ~/.config/valuecell/lancedb
rm -rf ~/.config/valuecell/.knowledge
rm ~/.config/valuecell/valuecell.db
```

重启系统后会自动重建。

### 8.3 前端启动了但看不到后端数据

按以下顺序排查：

1. 后端服务是否正在运行
2. 后端启动日志中是否有初始化错误
3. 前端是否能访问后端 API（检查浏览器控制台网络请求）
4. 是否已完成模型提供商配置（没有 API Key 时 Agent 无法工作）

### 8.4 Agent 不工作

最常见的原因是配置问题：

- Agent Card（`python/configs/agent_cards/` 下的 JSON）是否存在？
- Agent YAML（`python/configs/agents/` 下的 YAML）是否存在？
- Card 中的 `name` 是否与类名一致？
- Card 中的 `url` 端口是否与其他 Agent 冲突？

详见 [configuration.md](./configuration.md) 的排错章节。

### 8.5 启动脚本安装依赖失败

- 确认网络连接正常
- 如果 `uv` 或 `bun` 安装失败，尝试手动安装：
  - uv：`curl -LsSf https://astral.sh/uv/install.sh | sh`
  - bun：`curl -fsSL https://bun.sh/install | bash`

---

## 9. 面向不同角色的建议

### 如果你是新用户，只想体验

1. 从 [Releases](https://github.com/ValueCell-ai/valuecell/releases) 下载桌面应用
2. 在应用内配置 API Key
3. 开始使用

### 如果你是后端开发者

1. 跑通 `uv sync --group dev`
2. 运行 `uv run pytest` 确认测试通过
3. 重点阅读 `python/valuecell/server/` 和 `python/valuecell/core/`
4. 继续看 [architecture.md](./architecture.md)

### 如果你是前端开发者

1. 跑通 `bun install` 和 `bun run dev`
2. 运行 `bun run typecheck` 确认类型正确
3. 重点阅读 `frontend/src/` 下的 `api/`、`store/`、`app/`、`components/`
4. 继续看 [source-guide.md](./source-guide.md)

### 如果你要开发 Agent

1. 先完成基本启动验证
2. 阅读现有 Agent 实现（`python/valuecell/agents/`）
3. 继续看 [agent-extension.md](./agent-extension.md)

---

## 10. 继续阅读

- 想深入理解配置系统：看 [configuration.md](./configuration.md)
- 想理解系统架构与运行原理：看 [architecture.md](./architecture.md)
- 想从源码层面理解：看 [source-guide.md](./source-guide.md)
- 想自己开发 Agent：看 [agent-extension.md](./agent-extension.md)
- 想系统化学习：看 [learning-path.md](./learning-path.md)
