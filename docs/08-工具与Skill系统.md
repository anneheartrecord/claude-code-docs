[English](docs/08-Tools-and-Skills.md)

# 08 工具与 Skill 系统

如果说 Agent 循环是 Claude Code 的心脏，那么**工具系统就是它的双手**。一个 AI Agent 再聪明，如果不能读文件、写代码、执行命令，它就只是一个聊天机器人。工具系统赋予了 Agent 与真实世界交互的能力，而 Skill 系统则让这种能力可以被用户自由扩展。

这套设计背后有一个更大的行业趋势：**AI Agent 正在从对话式走向操作式**。2023 年的 ChatGPT 只能生成文字，2024 年的 Agent 开始调用工具，2025 年的 Claude Code 已经能独立完成从读需求到写代码到跑测试的完整流程。工具系统的设计质量，直接决定了一个 Agent 的实际生产力。

## 1️⃣ 工具注册：40+ 个内置工具的全景图

![40+工具全景图](../imgs/08-tools-panorama.png)


Claude Code 内置注册了超过 40 个工具。这个数量在行业中属于第一梯队。作为对比，GitHub Copilot Agent 大约有 10 个工具，Cursor 有 15 个左右，而开源的 Aider 只有文件编辑和命令执行两大类。

**工具数量多意味着什么？** 意味着 Agent 可以做更精细的操作选择。举个例子，很多 Agent 只有一个通用的"执行命令"工具，搜索文件需要通过 `find` 命令，搜索内容需要通过 `grep` 命令。而 Claude Code 把这些拆成了独立的 **GlobTool** 和 **GrepTool**，每个工具都有精心设计的参数和返回格式。这样做的好处是模型更容易选对工具，出错率更低，速度也更快。

### 始终可用的核心工具

| 工具 | 功能 | 为什么重要 |
|------|------|-----------|
| **BashTool** | 执行 shell 命令 | Agent 的万能钥匙，能做任何系统操作 |
| **GlobTool** | 文件模式匹配搜索 | 快速定位文件，比 `find` 命令更适合 AI 使用 |
| **GrepTool** | 内容搜索 | 在代码库中搜索特定内容，底层基于 ripgrep |
| **FileReadTool** | 读取文件 | 支持代码、图片、PDF、Jupyter Notebook |
| **FileEditTool** | 编辑文件 | 精确字符串替换，避免重写整个文件 |
| **FileWriteTool** | 写入文件 | 创建新文件或完整重写 |
| **WebFetchTool** | 抓取网页内容 | 获取文档、API 参考等在线资源 |
| **WebSearchTool** | 搜索网页 | 当需要最新信息时使用 |
| **AgentTool** | 启动子 Agent | 将复杂任务委派给独立的子进程 |
| **SkillTool** | 调用 Skill | 执行用户自定义的 Skill |
| **TaskOutputTool** | 获取后台任务输出 | 读取之前在后台运行的任务的结果 |
| **EnterPlanModeTool** | 进入计划模式 | 切换到只读的规划模式 |
| **ExitPlanModeTool** | 退出计划模式 | 回到可执行的正常模式 |
| **AskUserQuestionTool** | 向用户提问 | 当信息不足时主动询问 |
| **NotebookEditTool** | 编辑 Jupyter Notebook | 操作 .ipynb 文件的单元格 |

### 通过 Feature Flag 控制的工具

部分工具在外部构建中被禁用，只在 Anthropic 内部或特定环境中可用：

| 工具 | Feature Flag | 功能 |
|------|-------------|------|
| **SleepTool** | 未公开 | 让 Agent 等待指定时间 |
| **CronTools** | 未公开 | 定时任务管理 |
| **MonitorTool** | 未公开 | 监控和观测 |
| **WebBrowserTool** | WEB_BROWSER_TOOL | 完整的浏览器自动化交互 |
| **WorkflowTool** | WORKFLOW_SCRIPTS | 执行预定义的工作流脚本 |

**值得注意的是 FileEditTool 和 FileWriteTool 的分离设计。** 大多数 Agent 只有一个写文件的工具，整个文件一次性覆盖。Claude Code 把"编辑"独立出来，使用精确的字符串匹配和替换。这意味着修改一个 1000 行文件中的 3 行代码时，Agent 只需要发送那 3 行的变更，而非整个文件。**这个设计同时节省了 token 消耗和出错概率**，是一个非常务实的工程选择。

## 2️⃣ 工具执行流水线：从 API 响应到实际操作

![工具执行流水线](../imgs/08-tools-pipeline.png)


当 Claude 模型决定调用一个工具时，这个请求不是直接执行的。它要经过一条**精心设计的流水线**，每一步都有特定的安全和工程目的。

```
API 返回 tool_use blocks
  ↓
┌──────────────────┐
│  Pre-Hook         │  日志记录、参数校验、输入清洗
│                   │  确保工具调用的参数合法
└────────┬─────────┘
         ↓
┌──────────────────┐
│  权限检查         │  三种权限模式：
│                   │    default → 按内置规则判断
│                   │    auto → 自动放行受信工具
│                   │    plan → 只允许只读操作
│                   │
│                   │  检查流程：
│                   │    1. 硬编码危险模式 → 直接拒绝
│                   │    2. 用户自定义规则 → allow/deny
│                   │    3. 都不匹配 → 走默认权限流程
└────────┬─────────┘
         ↓
┌──────────────────┐
│  工具查找         │  按 name 在注册表中匹配
│                   │  找不到 → 返回错误信息给模型
│                   │  模型会据此调整策略
└────────┬─────────┘
         ↓
┌──────────────────┐
│  并行执行         │  使用 Promise.all 并发
│                   │  多个工具调用可以同时运行
│                   │  每个工具有独立的 try/catch
│                   │  单个失败不影响其他工具
└────────┬─────────┘
         ↓
┌──────────────────┐
│  Post-Hook        │  结果清洗、长度截断
│                   │  审计日志记录
│                   │  敏感信息过滤
└────────┬─────────┘
         ↓
tool_result 追加到消息历史
模型基于结果决定下一步动作
```

**并行执行是这条流水线中最关键的设计。** 传统的 Agent 实现通常是串行调用工具的：先调用 A，等 A 返回，再调用 B，等 B 返回。而 Claude Code 的模型可以在一次响应中返回多个 tool_use block，流水线会用 `Promise.all` 同时执行它们。

![并行执行对比](../imgs/08-tools-parallel.png)

这在实际使用中的效果非常明显。比如模型需要同时读取三个文件来理解一个功能的实现，串行执行需要三次 IO 等待，并行执行只需要一次。在大型项目中，这种并行能力可以将工具执行时间缩短 50% 以上。

**错误隔离同样重要。** 每个工具调用都被包裹在独立的 `try/catch` 中，一个工具的失败不会导致其他并行工具也失败。失败的结果会以错误信息的形式返回给模型，模型可以据此采取补救措施。这种设计让 Agent 具备了**优雅降级**的能力。

## 3️⃣ Skill 系统：让 Agent 学会新能力

### Skill 的本质

**Skill 是一种给 AI Agent 写的说明书。** 它是一个 `SKILL.md` 文件，用自然语言描述如何完成某种特定任务。Agent 读到这个文件后，按照说明执行操作。

这个概念在行业中有很多类似的实现。OpenAI 的 GPTs 允许用户通过 Instructions 定制行为，LangChain 有 Tool 和 Chain 的概念，AutoGPT 有 Plugin 系统。但 Claude Code 的 Skill 系统有一个独特的设计选择：**Skill 不是代码插件，而是自然语言说明**。

为什么用自然语言而非代码？因为代码插件需要遵循特定的 API 规范，开发门槛高，而且每次底层框架升级都可能导致插件不兼容。自然语言说明则完全解耦了：只要 Agent 能理解人话，SKILL.md 就永远能工作。这降低了创建 Skill 的门槛，任何能写文档的人都可以创建 Skill。

### Skill 的发现机制

![Skill发现与执行流程](../imgs/08-skill-discovery.png)


```
启动时自动扫描：
~/.claude/skills/*/SKILL.md  → 用户全局 Skill
```

SKILL.md 的格式使用 YAML frontmatter 加 Markdown 正文：

```yaml
---
name: my-skill
description: 这个 Skill 的功能简述
allowed-tools:    # 可选，限定这个 Skill 可以使用哪些工具
  - Bash
  - Read
---
# Skill 使用说明

## 什么时候使用
当用户要求 ... 时使用这个 Skill。

## 执行步骤
1. 首先读取 ...
2. 然后执行 ...
3. 最后输出 ...
```

**description 字段是 Skill 系统最关键的元数据。** Agent 在选择使用哪个 Skill 时，看到的只是所有 Skill 的名称和描述列表，并不会读取完整的 SKILL.md 内容。只有在确定要使用某个 Skill 之后，才会读取完整说明。这种**两阶段加载**的设计避免了把所有 Skill 的完整内容塞进 system prompt 导致上下文爆炸。

### Skill 的执行：子 Agent 隔离

调用 Skill 时，Claude Code 不是在主 Agent 循环中直接执行，而是 **fork 出一个独立的子 Agent**。这个设计和操作系统中的进程隔离思路一致。

```
用户输入 /my-skill 或 Agent 自主选择
  ↓
SkillTool 接管控制
  ↓
创建隔离的子 Agent
  - 独立的 token 预算：不会耗尽主 Agent 的配额
  - 独立的消息历史：不污染主对话的上下文
  - 受限的工具集：只能用安全的工具子集
  ↓
子 Agent 读取 SKILL.md 完整内容
  ↓
按说明执行任务，可能涉及多轮工具调用
  ↓
执行结果返回给主 Agent
主 Agent 继续后续工作
```

**子 Agent 的工具限制是安全的关键。** 子 Agent 只能使用 Bash、File 系列工具、Web 系列工具、SkillTool 和 AgentTool。它**不能访问权限管理、MCP 配置管理、认证**等敏感工具。这意味着即使一个恶意的 SKILL.md 试图让 Agent 做危险操作，子 Agent 也没有对应的工具可用。

**递归调用：** Skill 内部可以通过 SkillTool 调用其他 Skill，实现 Skill 的组合和复用。比如一个"全栈功能开发"的 Skill 可以依次调用"后端 API 开发"Skill 和"前端页面开发"Skill。

### Skill 在 System Prompt 中的呈现

所有已发现的 Skill 会以摘要列表的形式注入到 system prompt 中：

```
system prompt 内容：
...
Available skills:
- my-skill: 这个 Skill 做什么
- review: 代码审查工具
- translate: 翻译工具
...

Agent 的行为规则：选择一个最匹配的 Skill 后，
先读取它的完整 SKILL.md，然后按说明执行。
```

### 容量限制

Skill 系统设计了多层容量限制，防止 Skill 数量失控导致性能问题：

| 限制项 | 默认值 | 设计原因 |
|--------|--------|---------|
| 每个来源最多扫描候选数 | 300 | 防止文件系统扫描耗时过长 |
| 每个来源最多加载数 | 200 | 防止内存占用过高 |
| system prompt 里最多列出数 | 150 | 防止 Skill 列表占用过多上下文 |
| Skill 列表总字符预算 | 30,000 | 约占 system prompt 的 10% |
| 单个 SKILL.md 最大 | 256 KB | 防止单个 Skill 说明过于冗长 |

## 4️⃣ 工具系统的设计哲学

![工具系统设计哲学](../imgs/08-skill-philosophy.png)


回顾 Claude Code 的工具和 Skill 系统，可以提炼出几个核心的设计原则：

**精细化优于通用化。** 把搜索拆成 Glob 和 Grep，把文件操作拆成 Read、Edit、Write，每个工具职责单一。这让模型更容易做出正确的工具选择。

**并行优于串行。** 工具执行支持并发，模型可以一次请求多个工具调用。这在日常使用中显著提升了交互速度。

**隔离优于共享。** Skill 通过子 Agent 执行，有独立的预算、历史和工具集。一个 Skill 的问题不会影响主 Agent。

**自然语言优于代码 API。** Skill 用 Markdown 文件描述，而非代码插件。这让创建和维护 Skill 的门槛极低。

这些设计选择的共同指向是：**让 Agent 更可靠、更安全、更易扩展**。在 AI Agent 还处于早期阶段的今天，可靠性比功能丰富度更重要。

## 5️⃣ 与其他 Agent 工具系统的横向对比

| 特性 | Claude Code | GitHub Copilot | Cursor | Aider |
|------|------------|---------------|--------|-------|
| 内置工具数量 | 40+ | 约 10 | 约 15 | 约 5 |
| 并行工具调用 | 支持 | 不支持 | 部分支持 | 不支持 |
| 用户自定义扩展 | Skill 系统 | 无 | 无 | 无 |
| 外部工具协议 | MCP | 无 | 无 | 无 |
| 文件编辑方式 | 精确替换 | 整文件重写 | diff 模式 | diff 模式 |
| 子 Agent 隔离 | 有 | 无 | 无 | 无 |

Claude Code 在工具系统的完备性上明显领先。但也要看到，工具越多，模型选错工具的概率也越高。Anthropic 通过精心设计的工具描述和 system prompt 中的使用指南来缓解这个问题，但这依然是一个持续优化的方向。

下一篇：[09-MCP 集成](docs/09-MCP集成.md)
