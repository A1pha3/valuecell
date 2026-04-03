# ValueCell 中文文档中心

本目录面向中文读者整理 ValueCell 的使用、配置、架构、源码与扩展文档。

## 文档原则

1. **以当前仓库代码为准**：如果旧文档与现有实现不一致，以当前源码行为为准
2. **按读者阶段分层**：新手先看"概览"和"快速上手"，开发者继续看"架构""源码导读""Agent 扩展"
3. **严格尊重事实**：不编造未实现的功能，区分"已实现""部分测试"和"规划中"

## 文档导航

| 文档 | 适合谁 | 重点内容 |
| --- | --- | --- |
| [overview.md](./overview.md) | 所有人 | 产品定位、核心能力、Agent 列表、模型提供商、核心概念 |
| [getting-started.md](./getting-started.md) | 新用户 / 初次运行者 | 安装前提、API Key 配置、一键启动、常见问题 |
| [configuration.md](./configuration.md) | 使用者 / 开发者 | 三层配置系统、提供商配置、环境变量参考、排错指南 |
| [architecture.md](./architecture.md) | 后端 / 前端 / 架构阅读者 | 编排链路、流式事件机制、Agent A2A 协议、持久化 |
| [source-guide.md](./source-guide.md) | 开发者 | 前后端入口、按角色/问题/链路三种阅读路线 |
| [agent-extension.md](./agent-extension.md) | Agent 开发者 | 最小接入路径、事件系统、工具集成、调试方法 |
| [learning-path.md](./learning-path.md) | 系统学习者 | 6 阶段学习路径、自测清单、实践练习 |

## 推荐阅读路线

### 新手入门

```
overview.md → getting-started.md → configuration.md
```

### 理解原理

```
overview.md → architecture.md → source-guide.md
```

### 开发扩展

```
architecture.md → source-guide.md → agent-extension.md
```

### 系统学习

```
learning-path.md（按阶段指引阅读其他文档）
```

## 当前文档覆盖范围

覆盖内容：

- ValueCell 前端、后端与多智能体运行时
- 本地运行与调试路径
- 三层配置系统（环境变量 > `.env` > YAML）
- Agent Card + YAML + 代码实现三部分接入方式
- 9 个 LLM 提供商的配置方式
- 交易所支持状态与安全注意事项
- 6 阶段从新手到专家的学习路径

不在覆盖范围内：

- 第三方服务的完整使用教程
- 未来规划功能的实现细节
- 未在当前仓库中出现的内部部署方案

## 对照英文资料

- [`README.md`](../../README.md)
- [`docs/CONFIGURATION_GUIDE.md`](../CONFIGURATION_GUIDE.md)
- [`docs/CORE_ARCHITECTURE.md`](../CORE_ARCHITECTURE.md)
- [`docs/CONTRIBUTE_AN_AGENT.md`](../CONTRIBUTE_AN_AGENT.md)
