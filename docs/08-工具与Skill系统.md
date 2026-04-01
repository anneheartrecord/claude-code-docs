[English](./08-Tools-and-Skills.md)

# 08 工具与 Skill 系统

## 工具注册

Claude Code 注册了 40+ 个工具，通过 feature flag 控制哪些在外部构建中可用。

### 始终可用

| 工具 | 功能 |
|------|------|
| BashTool | 执行 shell 命令 |
| GlobTool | 文件模式匹配搜索 |
| GrepTool | 内容搜索 |
| FileReadTool | 读取文件 |
| FileEditTool | 编辑文件（精确替换） |
| FileWriteTool | 写入文件 |
| WebFetchTool | 抓取网页内容 |
| WebSearchTool | 搜索网页 |
| AgentTool | 启动子 Agent |
| SkillTool | 调用 Skill |
| TaskOutputTool | 获取后台任务输出 |
| EnterPlanModeTool | 进入计划模式 |
| ExitPlanModeTool | 退出计划模式 |
| AskUserQuestionTool | 向用户提问 |
| NotebookEditTool | 编辑 Jupyter Notebook |

### Feature Flag 控制（外部构建中禁用）

| 工具 | Feature Flag | 功能 |
|------|-------------|------|
| SleepTool | 未知 | 休眠 |
| CronTools | 未知 | 定时任务 |
| MonitorTool | 未知 | 监控 |
| WebBrowserTool | WEB_BROWSER_TOOL | 浏览器交互 |
| WorkflowTool | WORKFLOW_SCRIPTS | 工作流执行 |

## 工具执行流水线

```
API 返回 tool_use blocks
  ↓
┌─────────────────┐
│  Pre-Hook        │ 日志记录、参数校验
└────────┬────────┘
         ↓
┌─────────────────┐
│  权限检查        │ default/auto/plan 三种模式
│                  │ 硬编码危险模式 → 自动拒绝
│                  │ 用户规则匹配 → allow/deny
│                  │ 不匹配 → 走默认权限流程
└────────┬────────┘
         ↓
┌─────────────────┐
│  查找工具        │ 按 name 在注册表中匹配
│                  │ 找不到 → 返回 error
└────────┬────────┘
         ↓
┌─────────────────┐
│  并行执行        │ Promise.all
│                  │ 多个工具调用可同时运行
│                  │ 每个工具有独立的 try/catch
└────────┬────────┘
         ↓
┌─────────────────┐
│  Post-Hook       │ 结果清洗、审计日志
└────────┬────────┘
         ↓
tool_result 追加到消息历史
```

## Skill 系统

### Skill 是什么

Skill 是一个 SKILL.md 文件，描述一种特定能力的使用方法。Agent 读到 SKILL.md 后，按说明执行。

### Skill 发现

```
启动时扫描：
~/.claude/skills/*/SKILL.md  → 自动发现
```

SKILL.md 格式：
```yaml
---
name: my-skill
description: 这个 skill 做什么
allowed-tools:    # 可选
  - Bash
  - Read
---
# Skill 使用说明（给 Agent 看的）
...
```

### Skill 执行：Fork 子 Agent

调用 Skill 时不是在主循环里执行，而是 fork 一个独立的子 Agent：

```
用户输入 /my-skill
  ↓
SkillTool 接管
  ↓
创建隔离的子 Agent
  - 独立的 token 预算
  - 独立的消息历史
  - 受限的工具集
  ↓
子 Agent 读取 SKILL.md
  ↓
按说明执行任务
  ↓
结果返回给主 Agent
```

**子 Agent 工具限制：** 只能使用 Bash、File 系列、Web 系列、SkillTool、Agent。不能访问权限敏感的工具（Auth、MCP 管理等）。

**递归支持：** Skill 可以调用 SkillTool，也就是说 Skill 内部可以嵌套调用其他 Skill。

### Skill 加载到 System Prompt

所有已发现的 Skill 的 name + description 列表会注入到 system prompt 中。Agent 根据列表选择最匹配的 Skill，然后读取完整的 SKILL.md 执行。

```
system prompt 里：
...
Available skills:
- my-skill: 这个 skill 做什么
- review: 代码审查工具
- translate: 翻译工具
...

"Read exactly one skill after selecting"
```

### 关键限制

| 限制 | 默认值 |
|------|--------|
| 每个来源最多扫描候选数 | 300 |
| 每个来源最多加载数 | 200 |
| system prompt 里最多列出数 | 150 |
| Skill 列表总字符预算 | 30,000 |
| 单个 SKILL.md 最大 | 256 KB |

## MCP 工具

除了内置工具和 Skill，Claude Code 还通过 MCP 协议接入外部工具。详见 [09-MCP 集成](./09-mcp-integration.md)。
