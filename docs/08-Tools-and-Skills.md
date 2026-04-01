[中文](./08-工具与Skill系统.md)

# 08 Tools and Skills

## Tool Registration

Claude Code registers 40+ tools, with feature flags controlling which ones are available in external builds.

### Always Available

| Tool | Function |
|------|----------|
| BashTool | Execute shell commands |
| GlobTool | File pattern matching search |
| GrepTool | Content search |
| FileReadTool | Read files |
| FileEditTool | Edit files (exact replacement) |
| FileWriteTool | Write files |
| WebFetchTool | Fetch web page content |
| WebSearchTool | Search the web |
| AgentTool | Launch sub-Agent |
| SkillTool | Invoke Skills |
| TaskOutputTool | Get background task output |
| EnterPlanModeTool | Enter plan mode |
| ExitPlanModeTool | Exit plan mode |
| AskUserQuestionTool | Ask user a question |
| NotebookEditTool | Edit Jupyter Notebooks |

### Feature Flag Controlled (Disabled in External Builds)

| Tool | Feature Flag | Function |
|------|-------------|----------|
| SleepTool | Unknown | Sleep |
| CronTools | Unknown | Scheduled tasks |
| MonitorTool | Unknown | Monitoring |
| WebBrowserTool | WEB_BROWSER_TOOL | Browser interaction |
| WorkflowTool | WORKFLOW_SCRIPTS | Workflow execution |

## Tool Execution Pipeline

```
API returns tool_use blocks
  ↓
┌─────────────────┐
│  Pre-Hook        │ Logging, parameter validation
└────────┬────────┘
         ↓
┌─────────────────┐
│  Permission      │ Three modes: default/auto/plan
│  Check           │ Hardcoded dangerous patterns → Auto-reject
│                  │ User rule matching → allow/deny
│                  │ No match → Default permission flow
└────────┬────────┘
         ↓
┌─────────────────┐
│  Tool Lookup     │ Match by name in the registry
│                  │ Not found → Return error
└────────┬────────┘
         ↓
┌─────────────────┐
│  Parallel        │ Promise.all
│  Execution       │ Multiple tool calls can run simultaneously
│                  │ Each tool has its own try/catch
└────────┬────────┘
         ↓
┌─────────────────┐
│  Post-Hook       │ Result sanitization, audit logging
└────────┬────────┘
         ↓
tool_result appended to message history
```

## Skill System

### What Is a Skill

A Skill is a SKILL.md file that describes how to use a specific capability. When the Agent reads a SKILL.md, it executes according to the instructions.

### Skill Discovery

```
Scanned at startup:
~/.claude/skills/*/SKILL.md  → Auto-discovered
```

SKILL.md format:
```yaml
---
name: my-skill
description: What this skill does
allowed-tools:    # Optional
  - Bash
  - Read
---
# Skill usage instructions (for the Agent)
...
```

### Skill Execution: Forking a Sub-Agent

When a Skill is invoked, it's not executed in the main loop — instead, an independent sub-Agent is forked:

```
User inputs /my-skill
  ↓
SkillTool takes over
  ↓
Creates an isolated sub-Agent
  - Independent token budget
  - Independent message history
  - Restricted tool set
  ↓
Sub-Agent reads SKILL.md
  ↓
Executes task according to instructions
  ↓
Results returned to main Agent
```

**Sub-Agent tool restrictions:** Can only use Bash, File series, Web series, SkillTool, and Agent. Cannot access permission-sensitive tools (Auth, MCP management, etc.).

**Recursion support:** Skills can call SkillTool, meaning Skills can nest and invoke other Skills internally.

### Skill Loading into System Prompt

The name + description list of all discovered Skills is injected into the system prompt. The Agent selects the best-matching Skill from the list, then reads the complete SKILL.md to execute.

```
In the system prompt:
...
Available skills:
- my-skill: What this skill does
- review: Code review tool
- translate: Translation tool
...

"Read exactly one skill after selecting"
```

### Key Limits

| Limit | Default Value |
|-------|---------------|
| Max candidates scanned per source | 300 |
| Max loaded per source | 200 |
| Max listed in system prompt | 150 |
| Skill list total character budget | 30,000 |
| Max size of a single SKILL.md | 256 KB |

## MCP Tools

In addition to built-in tools and Skills, Claude Code also integrates external tools via the MCP protocol. See [09-MCP Integration](./09-mcp-integration.md) for details.
