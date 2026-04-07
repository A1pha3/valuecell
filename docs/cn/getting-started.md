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

### 3.2 验证环境就绪

在执行一键启动之前，建议先运行以下命令确认环境已准备就绪：

**检查 Python 版本（需要 3.12+）：**

```bash
python3 --version
# 预期输出类似：Python 3.12.x 或更高版本
```

如果版本低于 3.12，请先安装或升级 Python。推荐使用 `uv` 自动管理 Python 版本：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.12
```

**检查网络连通性（需要访问外部 LLM API 和包仓库）：**

```bash
# 测试 PyPI 连通性
curl -s -o /dev/null -w "%{http_code}" https://pypi.org/simple/

# 测试 npm registry 连通性
curl -s -o /dev/null -w "%{http_code}" https://registry.npmjs.org/
```

两条命令均应返回 `200`。如果无法连通，请检查代理或网络设置。

**检查磁盘空间（建议至少预留 2 GB）：**

```bash
# macOS / Linux
df -h .

# Windows PowerShell
Get-PSDrive -Name C
```

ValueCell 的依赖包和数据库文件合计约需 1-2 GB 空间，建议预留充足余量。

> **提示**：如果以上三项检查全部通过，你可以放心地继续下一步。如果某项不通过，请先解决对应问题再启动，避免后续排查困难。

### 3.3 一键启动

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

### 3.4 访问界面

启动完成后，在浏览器中打开：

```
http://localhost:1420
```

### 3.5 可选的启动参数

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
| **Azure** | 企业级部署 | Azure 门户 |
| **DeepSeek** | DeepSeek 系列 | [platform.deepseek.com](https://platform.deepseek.com/) |
| **DashScope** | 阿里云百炼，Qwen 系列 | [bailian.console.aliyun.com](https://bailian.console.aliyun.com/) |
| **Ollama** | 本地部署模型 | [ollama.com](https://ollama.com/) |
| **OpenAI-Compatible** | 兼容 OpenAI 接口的自定义服务 | 自定义 |

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

**练习：提供商配置实战**

1. **配置多个提供商并观察降级行为**：在 `.env` 中同时配置两个提供商的 API Key（例如 OpenRouter 和 SiliconFlow），然后故意将主提供商的 Key 改为无效值（如 `OPENROUTER_API_KEY=invalid-key`），设置 `FALLBACK_PROVIDERS=siliconflow`，启动后端并发起一次对话。观察日志中是否出现降级提示，确认系统是否成功切换到备用提供商。完成后记得将 Key 改回正确值。

2. **查看当前生效的提供商**：启动后端后，查看终端日志中包含 `provider` 关键字的行，确认系统实际使用的是哪个提供商。尝试在 `.env` 中设置 `PRIMARY_PROVIDER=google`，重启后端，再次检查日志中的提供商信息是否发生变化。

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

**练习：验证与排查**

1. **模拟后端断开场景**：在系统正常运行时，手动停止后端进程（在终端按 `Ctrl+C`），然后在前端发起一次对话。观察前端的表现（是否显示错误提示？网络请求返回什么状态码？）。随后重新启动后端，确认前端能否自动恢复连接。

2. **阅读启动日志并标注关键信息**：重新启动系统，将终端日志复制到文本编辑器中，标注以下关键信息出现的位置：数据库初始化、Agent 注册数量、服务监听端口、前端编译成功提示。这有助于你在后续遇到问题时快速定位相关日志。

---

## 8. 验证后的深入探索

完成基本验证后，如果你想更深入地了解系统工作方式，可以按以下步骤逐一操作：

### 8.1 开启调试模式并观察完整日志

调试模式会输出请求的完整 Prompt、工具调用细节和模型返回的元数据，是理解系统内部行为的第一步。

**操作步骤：**

```bash
# 1. 编辑系统目录中的 .env 文件（以 macOS 为例）
nano ~/Library/Application\ Support/ValueCell/.env

# 2. 添加或修改以下行
AGENT_DEBUG_MODE=true

# 3. 保存后重启后端
# 先停止正在运行的后端进程（Ctrl+C），然后重新启动
bash start.sh
```

**观察要点：**

- 终端日志中会出现以 `[DEBUG]` 开头的行，记录发送给 LLM 的完整 Prompt 内容
- 每次工具调用（如搜索、代码执行）都会输出调用参数和返回结果
- 流式响应的每个 chunk 都会被记录，你可以看到响应逐步生成的过程
- 日志中会显示使用的模型名称和提供商信息

完成观察后，建议将 `AGENT_DEBUG_MODE` 改回 `false`，避免日志过于冗长影响日常使用。

### 8.2 追踪一条完整的请求链路

通过一次实际对话，追踪请求从接入到返回的全过程。

**操作步骤：**

```bash
# 1. 确保后端已启动（调试模式建议开启）
# 2. 在浏览器中打开 http://localhost:1420
# 3. 选择一个 Agent（推荐 News Agent），发起一次简单提问，例如：
#    "最近有什么科技新闻？"
# 4. 同时观察终端中的日志输出
```

**在日志中依次寻找以下阶段：**

1. **请求接入**：日志中出现接收请求的记录，包含请求 ID 和时间戳
2. **Agent 分流**：日志显示请求被路由到哪个 Agent，注意观察 Agent 名称
3. **Prompt 组装**：调试模式下可看到组装后的完整 Prompt，包含系统提示、历史对话和当前用户输入
4. **工具调用**：如果 Agent 使用了工具（如搜索工具），日志会显示工具名称和参数
5. **流式响应**：日志逐行记录 LLM 返回的内容片段
6. **事件返回**：最终响应通过 Server-Sent Events (SSE) 推送到前端

**进阶提示：** 如果日志滚动太快，可以将输出重定向到文件：

```bash
bash start.sh 2>&1 | tee startup.log
# 发起对话后，用 grep 搜索关键内容
grep -i "agent\|route\|tool\|error" startup.log
```

### 8.3 阅读一个 Agent 的源码实现

通过阅读 News Agent 的代码，理解 Agent 的编写模式。

**操作步骤：**

```bash
# 查看 Agent 目录结构
ls python/valuecell/agents/

# 阅读 News Agent 的核心实现
cat python/valuecell/agents/news_agent/core.py
```

**阅读重点：**

- `stream()` 方法是 Agent 的核心入口，接收用户消息并返回流式响应
- 注意 `stream()` 方法中如何调用 LLM 和工具
- 观察返回值的格式，了解流式事件是如何组织的
- 对比其他 Agent（如 Research Agent）的实现，找出共性与差异：

```bash
# 对比不同 Agent 的实现结构
diff <(ls python/valuecell/agents/news_agent/) <(ls python/valuecell/agents/research_agent/)
```

### 8.4 检查 Agent 注册与发现机制

了解系统如何发现并注册所有 Agent。

**操作步骤：**

```bash
# 查看 Agent 配置文件
ls python/configs/agents/

# 查看某个 Agent 的配置
cat python/configs/agents/news_agent.yaml

# 查看 Agent Card（描述 Agent 能力的 JSON 文件）
ls python/configs/agent_cards/
cat python/configs/agent_cards/news_agent.json
```

**观察要点：**

- YAML 配置文件定义了 Agent 的名称、使用的模型、工具列表
- Agent Card 以 JSON 格式描述 Agent 的能力和接口，用于 A2A 协议通信
- 配置中的 `name` 字段必须与 Agent 类名一致，否则注册会失败

---

## 9. 常见问题

### 9.1 修改了仓库根目录的 `.env` 但没生效

当前后端使用的是**系统目录**中的 `.env`，不是仓库根目录。

```bash
# 确认你编辑的是正确的文件
# macOS
ls ~/Library/Application\ Support/ValueCell/.env

# Linux
ls ~/.config/valuecell/.env
```

### 9.2 数据库兼容性错误

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

### 9.3 前端启动了但看不到后端数据

按以下顺序排查：

1. 后端服务是否正在运行
2. 后端启动日志中是否有初始化错误
3. 前端是否能访问后端 API（检查浏览器控制台网络请求）
4. 是否已完成模型提供商配置（没有 API Key 时 Agent 无法工作）

### 9.4 Agent 不工作

最常见的原因是配置问题：

- Agent Card（`python/configs/agent_cards/` 下的 JSON）是否存在？
- Agent YAML（`python/configs/agents/` 下的 YAML）是否存在？
- Card 中的 `name` 是否与类名一致？
- Card 中的 `url` 端口是否与其他 Agent 冲突？

详见 [configuration.md](./configuration.md) 的排错章节。

### 9.5 启动脚本安装依赖失败

- 确认网络连接正常
- 如果 `uv` 或 `bun` 安装失败，尝试手动安装：
  - uv：`curl -LsSf https://astral.sh/uv/install.sh | sh`
  - bun：`curl -fsSL https://bun.sh/install | bash`

### 9.6 启动后端口被占用怎么办

ValueCell 默认使用端口 `8000`（后端）和 `1420`（前端）。如果这些端口已被其他程序占用，启动会失败。

**查找并释放占用端口的进程：**

```bash
# macOS / Linux：查看占用端口的进程
lsof -i :8000    # 后端默认端口
lsof -i :1420    # 前端默认端口

# 根据输出的 PID 终止进程
kill -9 <PID>
```

**或者通过环境变量指定不同端口：**

```bash
# 在 .env 中修改端口配置
BACKEND_PORT=8001
FRONTEND_PORT=1421
```

修改后重启系统即可。注意：如果修改了前端端口，浏览器中访问的地址也需要同步更改（如 `http://localhost:1421`）。

### 9.7 如何在 WSL2 中运行 ValueCell

ValueCell 可以在 WSL2（Windows Subsystem for Linux 2）中正常运行，但需要注意以下几点：

**安装前置依赖：**

```bash
# 确保 WSL2 已安装并更新
wsl --update

# 在 WSL2 内安装必要的系统包
sudo apt update
sudo apt install -y build-essential curl git
```

**处理文件路径差异：**

WSL2 中 ValueCell 的数据目录遵循 Linux 路径规则，位于：

```bash
~/.config/valuecell/
```

如果从 Windows 文件管理器访问，对应路径为：

```
\\wsl$\<发行版名称>\home\<用户名>\.config\valuecell\
```

**网络与端口注意事项：**

- WSL2 默认使用 NAT 网络模式，`localhost` 端口通常会自动映射到 Windows 主机
- 如果浏览器无法访问 `http://localhost:1420`，尝试在 WSL2 内用 `curl` 先验证：

```bash
curl -s http://localhost:1420 | head -5
```

- 如果 `curl` 能访问但 Windows 浏览器不能，可能需要手动配置端口转发或使用 WSL2 的镜像网络模式（在 `%USERPROFILE%\.wslconfig` 中设置 `networkingMode=mirrored`）

**GPU 加速（可选）：**

如果你的 Windows 主机有 NVIDIA GPU，WSL2 支持 CUDA 加速。这对于使用 Ollama 本地模型时会有显著性能提升。参考 [NVIDIA WSL2 CUDA 文档](https://docs.nvidia.com/cuda/wsl-user-guide/index.html) 进行配置。

### 9.8 后端启动后日志中出现 "No module found" 错误

这类错误通常表示 Python 依赖未正确安装或虚拟环境不一致。

**排查步骤：**

```bash
cd python

# 1. 确认 uv 已安装
uv --version

# 2. 重新同步依赖（包含开发依赖）
uv sync --group dev

# 3. 如果仍然报错，尝试清理并重建虚拟环境
rm -rf .venv
uv sync --group dev

# 4. 验证关键模块是否能正常导入
uv run python -c "import valuecell; print('模块导入成功')"
```

**常见原因及解决方案：**

| 原因 | 解决方案 |
|------|----------|
| 未使用 `uv sync` 安装依赖 | 运行 `uv sync --group dev` |
| 使用了系统 Python 而非虚拟环境 | 确保通过 `uv run` 命令启动，而非直接 `python` |
| 仓库更新后新增了依赖 | 重新运行 `uv sync --group dev` |
| Python 版本不兼容 | 运行 `uv python install 3.12` 安装正确版本 |

---

## 10. 面向不同角色的建议

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

## 11. 继续阅读

- 想深入理解配置系统：看 [configuration.md](./configuration.md)
- 想理解系统架构与运行原理：看 [architecture.md](./architecture.md)
- 想从源码层面理解：看 [source-guide.md](./source-guide.md)
- 想自己开发 Agent：看 [agent-extension.md](./agent-extension.md)
- 想系统化学习：看 [learning-path.md](./learning-path.md)
