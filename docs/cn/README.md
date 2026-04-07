# ValueCell 中文文档中心

ValueCell 是一个**社区驱动的、面向金融场景的多智能体（Multi-Agent）平台**——由多个专业 AI Agent 协同工作，提供金融研究、新闻追踪、交易策略等能力。支持 9 个 LLM 提供商、4 个内置 Agent、3 个免费市场数据适配器，可打包为 macOS / Windows 桌面应用。

本目录面向中文读者整理 ValueCell 的使用、配置、架构、源码与扩展文档。

## 文档原则

1. **以当前仓库代码为准**：如果旧文档与现有实现不一致，以当前源码行为为准
2. **按读者阶段分层**：新手先看"概览"和"快速上手"，开发者继续看"架构""源码导读""Agent 扩展"
3. **严格尊重事实**：不编造未实现的功能，区分"已实现""部分测试"和"规划中"
4. **术语首次标注原文**：核心术语首次出现时使用"中文（English）"格式；后续使用中文名或英文缩写（以文档内首次标注的译法为准）
5. **中英文混排规范**：中英文、中文数字之间保持空格，中文语境使用全角标点

## 如何使用本文档

### 如果你是新手

从未接触过 ValueCell 的读者，推荐按以下顺序阅读：

1. 先读 [项目概览](./overview.md)，建立全局印象（约 10 分钟）
2. 再读 [快速上手](./getting-started.md)，把系统跑起来（约 30 分钟）
3. 然后按 [学习路径](./learning-path.md) 系统学习

### 如果你是开发者

想基于 ValueCell 进行二次开发的读者，推荐路径：

1. 快速浏览 [项目概览](./overview.md)，了解整体结构
2. 精读 [系统架构详解](./architecture.md) 和 [核心设计原理](./principles.md)
3. 按角色阅读 [源码导读](./source-guide.md) 中对应的路线
4. 实践 [Agent 开发与扩展](./agent-extension.md)

### 如果你想快速查找信息

直接在下面的导航表中找到对应的文档。

## 文档导航

| 文档 | 适合谁 | 核心内容 | 预计阅读时间 |
| --- | --- | --- | --- |
| [overview.md](./overview.md) | 所有人 | 产品定位、核心能力、Agent 列表、模型提供商、核心概念速查表 | 15 分钟 |
| [getting-started.md](./getting-started.md) | 新用户 | 环境预检、API Key 配置、一键启动、分步验证、常见问题 | 25 分钟 |
| [use-cases.md](./use-cases.md) | 使用者 / 开发者 | 金融研究、新闻追踪、交易策略、场景决策树、最佳实践 | 20 分钟 |
| [configuration.md](./configuration.md) | 使用者 / 开发者 | 三层配置系统、提供商降级、配置速查卡、排错指南 | 20 分钟 |
| [architecture.md](./architecture.md) | 架构阅读者 | 编排链路、流式事件机制、Agent A2A 协议、架构决策速查 | 30 分钟 |
| [principles.md](./principles.md) | 高级开发者 | 核心设计原理、架构决策动机、现实类比、原则对比表 | 25 分钟 |
| [streaming-deep-dive.md](./streaming-deep-dive.md) | 后端 / 前端开发者 | 事件分类体系、处理管线、调试速查表、渲染器映射 | 30 分钟 |
| [frontend-architecture.md](./frontend-architecture.md) | 前端开发者 | React Router v7、状态流图、SSE 通信、Tauri v2 桌面集成 | 25 分钟 |
| [data-adapter.md](./data-adapter.md) | 后端 / 数据层开发者 | 适配器选择指南、Ticker 转换速查、路由与 Failover | 25 分钟 |
| [source-guide.md](./source-guide.md) | 开发者 | 源码导航地图、按角色 / 问题 / 链路三种阅读路线 | 20 分钟 |
| [agent-extension.md](./agent-extension.md) | Agent 开发者 | 最小接入路径、事件系统、扩展路线图、工具集成 | 25 分钟 |
| [learning-path.md](./learning-path.md) | 系统学习者 | 6 阶段路径、易错点提醒、阶段衔接指南、评估标准 | 10 分钟 |

## 推荐阅读路线

### 路线一：新手入门（总计约 1 小时）

```text
overview.md → getting-started.md → use-cases.md → configuration.md
```

目标：理解 ValueCell 是什么、能跑起来、会基本配置。

### 路线二：理解原理（总计约 1.5 小时）

```text
overview.md → architecture.md → principles.md → streaming-deep-dive.md
```

目标：理解系统为什么这样设计，核心模块之间如何协作。

### 路线三：开发扩展（总计约 1.5 小时）

```text
architecture.md → frontend-architecture.md → data-adapter.md → source-guide.md → agent-extension.md
```

目标：掌握代码结构，能独立开发新 Agent 或修改现有模块。

### 路线四：系统学习（按阶段推进）

```text
learning-path.md（按 6 个阶段逐步推进，每阶段有自测和练习）
```

目标：从新手到专家的完整学习路径。

## 当前文档覆盖范围

### 已覆盖内容

- ValueCell 前端、后端与多智能体运行时的完整架构解析
- 本地运行与调试路径（一键启动、分模块启动、调试模式）
- 三层配置系统（环境变量 > `.env` > YAML）的原理与操作
- Agent Card + YAML + 代码实现三部分接入方式
- 9 个 LLM 提供商的配置方式和降级机制
- 交易所支持状态与安全注意事项
- 金融研究、新闻追踪、交易策略等实战场景操作指南
- 核心设计原理与架构决策动机的深度解析
- 流式事件系统完整分类与处理管线详解
- 前端架构（React Router v7、状态管理、SSE 通信、Tauri v2 桌面集成）
- 数据适配器架构与三大适配器（Yahoo Finance / AKShare / BaoStock）详解
- 6 阶段从新手到专家的学习路径

### 文档学习辅助

每篇文档均包含以下学习辅助内容（随文档深度递增）：

| 辅助元素 | 说明 | 出现位置 |
|----------|------|----------|
| **学习目标** | 阅读后应获得的能力清单 | 所有文档开头 |
| **"为什么"解释** | 核心概念的存在原因、适用边界和代价 | 所有文档正文 |
| **常见问题** | 该主题下最常见的 4-6 个问题与解答 | 所有文档末尾 |
| **自测检查清单** | 不看原文即可回答的验证问题 | 所有文档末尾 |
| **练习** | 动手实践验证理解的练习题 | getting-started、configuration、learning-path 等 |
| **速查卡/决策表** | 紧凑的操作参考卡片 | configuration、data-adapter、streaming-deep-dive 等 |
| **术语速查表** | 关键术语的一行解释参考表 | overview、learning-path |

### 不在覆盖范围内

- 第三方服务的完整使用教程（如 SEC EDGAR、交易所 API 的详细使用说明）
- 未来规划功能的实现细节（如欧股、商品、外汇、期权等）
- 未在当前仓库中出现的内部部署方案
- 第三方库（Agno、yfinance、AKShare 等）的 API 文档（请查阅各自官方文档）

## 文档约定

| 约定 | 说明 |
|------|------|
| 代码块语言 | 所有代码块均指定语言（`bash`、`python`、`json`、`yaml`、`typescript` 等） |
| 状态标记 | ✅ 已实现 / ✅ 已测试 / 🟡 部分测试 / 📋 规划中 |
| 文件路径 | 使用仓库根目录的相对路径（如 `python/valuecell/core/types.py`） |
| 术语格式 | 首次出现使用"中文（English）"，后续统一使用中文 |

## 对照英文资料

中文文档基于英文文档扩展编写，内容更详细但以当前代码为准：

- [`README.md`](../../README.md) — 项目主文档
- [`docs/CONFIGURATION_GUIDE.md`](../CONFIGURATION_GUIDE.md) — 配置指南
- [`docs/CORE_ARCHITECTURE.md`](../CORE_ARCHITECTURE.md) — 核心架构
- [`docs/CONTRIBUTE_AN_AGENT.md`](../CONTRIBUTE_AN_AGENT.md) — Agent 贡献指南

## 文档更新说明

- 中文文档随仓库代码同步更新
- 当中文文档与英文文档或源码不一致时，以**当前源码**为准
- 如发现文档错误，欢迎通过 GitHub Issue 或 PR 反馈
