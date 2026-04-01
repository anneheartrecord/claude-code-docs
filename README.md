[English](./README_EN.md)

# Claude Code 源码解剖

![License](https://img.shields.io/badge/license-MIT-green)
![Docs](https://img.shields.io/badge/docs-13%20articles-blue)
![Language](https://img.shields.io/badge/language-中文%20%7C%20English-orange)
![Lines](https://img.shields.io/badge/analyzed-515K%20lines-red)
![AI](https://img.shields.io/badge/AI%20Powered-Claude%20Code-purple)

> 📖 **在线阅读：** https://anneheartrecord.github.io/claude-code-docs/

2026 年 3 月 31日，有人发现 Anthropic 发布在 npm 上的 Claude Code 客户端包里带了 sourcemap 文件。

Sourcemap 是前端构建工具生成的调试辅助文件，记录了编译后代码和原始源码之间的映射关系。正常发布时应该把它排除掉，但 Anthropic 的构建流程里漏了这一步。

于是完整的 TypeScript 源码被反向还原了出来。51.5 万行代码，2,766 个文件。

**需要明确的是：泄露的只是客户端侧的代码。** Claude Code 是典型的客户端-服务端分离架构。客户端跑在你的终端里，负责用户交互、工具执行、权限管理、上下文组装。服务端是 Anthropic 的 API，负责模型推理。模型本身和服务端逻辑没有泄露。

但客户端的这部分代码已经足够有价值了。因为 Agent 产品的核心竞争力不在模型调用本身，在于**怎么把模型的能力安全、高效、稳定地释放出来**。这正是 Harness Engineering 的范畴，客户端代码就是 Harness 的完整实现。

## 这个仓库做了什么

我用 Claude Code 对这份源码做了系统化的技术分析，逐模块拆解，写了 13 篇技术文档，中英双语。

从架构设计到 Agent 循环的六阶段实现，从三层消息压缩体系到 6,300 行的权限系统，从五层记忆加载到 40+ 工具的执行流水线。同时从 82 个 feature flag 里挖出了 Anthropic 还没发布的未来功能蓝图：Kairos 自主运行模式、Context Collapse 上下文折叠、Voice Mode 语音交互。

最后几篇是我自己的思考：这份代码到底值不值钱、AI Coding 时代工程师怎么做 Code Review、以及从源码里能看到 Claude 封号机制的哪些线索。

**目标读者：** 正在做 Agent 产品的工程师、对 AI Agent 架构感兴趣的开发者、想了解顶级 Agent 系统如何工程化落地的技术人。非技术背景的同学也能从中了解到 Agent 产品的运作方式和行业趋势。

## 文档目录

### 概览篇

| 文档 | 内容 |
|------|------|
| [01-架构总览](./docs/01-架构总览.md) | 整体架构、技术栈、核心文件、一次请求的完整旅程 |
| [02-源码泄露的价值之争](./docs/02-源码泄露的价值之争.md) | 产物 vs Harness 能力，两派观点分析，代码是快照能力是动态的 |

### 核心模块篇

| 文档 | 内容 |
|------|------|
| [03-Agent 循环](./docs/03-Agent循环.md) | 六阶段 ReAct 循环、AsyncGenerator 设计、状态管理、思维链保留 |
| [04-上下文工程](./docs/04-上下文工程.md) | System Prompt 构建、CLAUDE.md 加载、分层优先级、预取缓存、Prompt Cache 优化 |
| [05-消息压缩系统](./docs/05-消息压缩系统.md) | 三层压缩：微压缩、Session Memory、Full Compact，熔断器、递归保护 |
| [06-权限系统](./docs/06-权限系统.md) | 三模式权限、YOLO 分类器、42 条拦截规则、文件沙箱、Dangerous Rule Stripping |
| [07-记忆管理](./docs/07-记忆管理.md) | 五层记忆加载、@include 指令、MEMORY.md 管理、Session Memory |
| [08-工具与 Skill 系统](./docs/08-工具与Skill系统.md) | 40+ 工具注册、执行流水线、Pre/Post Hook、Skill fork 机制 |
| [09-MCP 集成](./docs/09-MCP集成.md) | 六种传输协议、OAuth、七种配置作用域 |

### 前瞻篇

| 文档 | 内容 |
|------|------|
| [10-未来功能蓝图](./docs/10-未来功能蓝图.md) | 82 个 feature flag 解析、Kairos 自主模式、Context Collapse、语音模式 |
| [11-AI Coding 时代的 Code Review](./docs/11-AI-Coding时代的Code-Review.md) | 个人/团队/CICD 三层 Review 范式、Review Agent 设想 |
| [12-从权限系统学 Agent 安全设计](./docs/12-从Claude%20Code权限系统学Agent安全设计.md) | 三层防御体系拆解、L0-L4 安全成熟度模型、落地建议 |
| [13-啃完源码之后的一些发现](./docs/13-啃完源码之后的一些发现.md) | AI 工程化短板、生产事故、Claude 封号机制分析 |


## 关键数据

| 指标 | 数据 |
|------|------|
| 代码总量 | 515,498 行 TypeScript/TSX |
| 文件数 | 2,766 |
| 构建产物 | 25.89 MB，5,344 模块 |
| 内置工具 | 40+ |
| Feature Flag | 82 个 |
| 权限拦截规则 | 42 条硬编码危险模式 |
| 消息压缩阈值 | 上下文窗口 - 13,000 token |
| npm 依赖 | 583 个包 |
| 是否 AI 生成 | 是，确信度 95%+ |

## License

本仓库为技术分析文档，不包含 Claude Code 源码本身。分析内容基于公开可获取的 sourcemap 还原代码。

MIT
