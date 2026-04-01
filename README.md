[English](./README_EN.md)

# Claude Code 源码解剖

基于 Anthropic Claude Code 泄露源码的深度技术分析。51.5 万行 TypeScript，2,766 个文件，完整拆解每一个核心模块。

## 这个仓库是什么

2026 年 3 月，Claude Code 的完整源码通过 sourcemap 泄露。这是目前市面上最成熟的 AI Agent 产品之一，也是第一个被完整逆向的商业级 Agent 系统。

本仓库对这份源码进行了系统化的技术分析，拆解了架构设计、上下文工程、消息压缩、权限系统、记忆管理等核心模块，同时挖掘了 82 个 feature flag 背后隐藏的未来功能蓝图。

**目标读者：** 正在做 Agent 产品的工程师、对 AI Agent 架构感兴趣的开发者、想了解顶级 Agent 系统如何工程化落地的技术人。

## 文档目录

### 概览篇

| 文档 | 内容 |
|------|------|
| [01-架构总览](./docs/01-架构总览.md) | 整体架构、技术栈、核心文件、数据流 |
| [02-源码泄露的价值之争](./docs/02-源码泄露的价值之争.md) | 产物 vs Harness 能力，两派观点分析 |

### 核心模块篇

| 文档 | 内容 |
|------|------|
| [03-Agent 循环](./docs/03-Agent循环.md) | 六阶段 ReAct 循环、AsyncGenerator 设计、状态机 |
| [04-上下文工程](./docs/04-上下文工程.md) | 系统 prompt 构建、CLAUDE.md 加载、分层优先级、预取缓存 |
| [05-消息压缩系统](./docs/05-消息压缩系统.md) | 三层压缩体系：微压缩、Session Memory、Full Compact |
| [06-权限系统](./docs/06-权限系统.md) | 三模式权限、YOLO 分类器、危险模式、42 条拦截规则 |
| [07-记忆管理](./docs/07-记忆管理.md) | 五层记忆加载、@include 指令、MEMORY.md 管理 |
| [08-工具与 Skill 系统](./docs/08-工具与Skill系统.md) | 40+ 工具注册、执行流水线、Skill fork 机制 |
| [09-MCP 集成](./docs/09-MCP集成.md) | 多传输协议、OAuth、七种配置作用域 |

### 前瞻篇

| 文档 | 内容 |
|------|------|
| [10-未来功能蓝图](./docs/10-未来功能蓝图.md) | 82 个 feature flag 解析、Kairos 自主模式、Context Collapse、语音模式 |
| [11-AI Coding 时代的 Code Review](./docs/11-AI-Coding时代的Code-Review.md) | 个人/团队/CICD 三层 Review 范式、Review Agent 设想 |

## 快速开始

```bash
# 克隆源码仓库（fork 自泄露的 sourcemap 还原版）
git clone https://github.com/anthropics/claude-code.git
cd claude-code

# 安装依赖
bun install

# 构建
bun run build
# ✓ Bundled 5344 modules in 554ms
#   cli.js  25.89 MB

# 运行
bun run dev --version
# 2.1.888 (Claude Code)
```

## 关键数据

| 指标 | 数据 |
|------|------|
| 代码规模 | 515,498 行 TypeScript/TSX |
| 文件数 | 2,766 |
| 构建产物 | 25.89 MB，5,344 模块 |
| 工具数 | 40+ |
| Feature Flag | 82 个 |
| 权限规则 | 42 条危险模式 |
| 压缩阈值 | 上下文窗口 - 13K token |
| 依赖 | 583 个包 |

## License

本仓库为技术分析文档，不包含 Claude Code 源码本身。分析内容基于公开可获取的 sourcemap 还原代码。

MIT
