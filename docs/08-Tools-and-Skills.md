[中文](/docs/08-工具与Skill系统.md)

# 08 Tools and Skills

If the Agent loop is the heart of Claude Code, then **the tool system is its hands**. No matter how intelligent an AI Agent is, without the ability to read files, write code, and execute commands, it's just a chatbot. The tool system gives the Agent the power to interact with the real world, and the Skill system lets users freely extend that power.

Behind this design lies a broader industry trend: **AI Agents are evolving from "conversational" to "operational."** In 2023, ChatGPT could only generate text. In 2024, Agents started calling tools. By 2025, Claude Code can independently complete an entire workflow from reading requirements to writing code to running tests. The quality of a tool system directly determines an Agent's real-world productivity.

## 1️⃣ Tool Registration: A Panorama of 40+ Built-in Tools

Claude Code registers over 40 built-in tools, placing it in the top tier of the industry. For comparison, GitHub Copilot Agent has roughly 10 tools, Cursor has around 15, and the open-source Aider has only two broad categories: file editing and command execution.

**What does a high tool count mean?** It means the Agent can make more granular operational choices. For example, many Agents have only a single generic "execute command" tool, requiring file searches via `find` and content searches via `grep`. Claude Code breaks these into dedicated **GlobTool** and **GrepTool**, each with carefully designed parameters and return formats. This makes it easier for the model to pick the right tool, reducing errors and improving speed.

### Always-Available Core Tools

| Tool | Function | Why It Matters |
|------|----------|---------------|
| **BashTool** | Execute shell commands | The Agent's master key for any system operation |
| **GlobTool** | File pattern matching | Locates files faster and more AI-friendly than `find` |
| **GrepTool** | Content search | Searches codebases using ripgrep under the hood |
| **FileReadTool** | Read files | Supports code, images, PDFs, Jupyter Notebooks |
| **FileEditTool** | Edit files | Precise string replacement, avoids rewriting entire files |
| **FileWriteTool** | Write files | Creates new files or performs complete rewrites |
| **WebFetchTool** | Fetch web content | Retrieves documentation, API references, online resources |
| **WebSearchTool** | Web search | Used when the latest information is needed |
| **AgentTool** | Spawn sub-Agents | Delegates complex tasks to independent sub-processes |
| **SkillTool** | Invoke Skills | Executes user-defined Skills |
| **TaskOutputTool** | Get background task output | Reads results from previously backgrounded tasks |
| **EnterPlanModeTool** | Enter plan mode | Switches to read-only planning mode |
| **ExitPlanModeTool** | Exit plan mode | Returns to executable normal mode |
| **AskUserQuestionTool** | Ask the user | Proactively asks when information is insufficient |
| **NotebookEditTool** | Edit Jupyter Notebooks | Manipulates individual cells in .ipynb files |

### Feature Flag-Controlled Tools

Some tools are disabled in the external build and only available internally at Anthropic or in specific environments:

| Tool | Feature Flag | Function |
|------|-------------|----------|
| **SleepTool** | Undisclosed | Lets the Agent wait for a specified duration |
| **CronTools** | Undisclosed | Scheduled task management |
| **MonitorTool** | Undisclosed | Monitoring and observability |
| **WebBrowserTool** | WEB_BROWSER_TOOL | Full browser automation interaction |
| **WorkflowTool** | WORKFLOW_SCRIPTS | Executes predefined workflow scripts |

**The separation of FileEditTool and FileWriteTool is a notable design choice.** Most Agents have a single file-writing tool that overwrites the entire file at once. Claude Code separates "editing" into its own tool that uses precise string matching and replacement. This means modifying 3 lines in a 1,000-line file requires the Agent to send only those 3 lines of changes, not the entire file. **This design saves both token consumption and reduces error probability** — a very pragmatic engineering decision.

## 2️⃣ Tool Execution Pipeline: From API Response to Real Action

When the Claude model decides to call a tool, the request doesn't execute directly. It passes through a **carefully designed pipeline**, with each step serving specific security and engineering purposes.

```
API returns tool_use blocks
  ↓
┌──────────────────┐
│  Pre-Hook         │  Logging, parameter validation, input sanitization
│                   │  Ensures tool call parameters are valid
└────────┬─────────┘
         ↓
┌──────────────────┐
│  Permission Check │  Three permission modes:
│                   │    default → judge by built-in rules
│                   │    auto → auto-approve trusted tools
│                   │    plan → only allow read-only operations
│                   │
│                   │  Checking flow:
│                   │    1. Hardcoded danger patterns → reject
│                   │    2. User-defined rules → allow/deny
│                   │    3. No match → fall through to defaults
└────────┬─────────┘
         ↓
┌──────────────────┐
│  Tool Lookup      │  Match by name in the registry
│                   │  Not found → return error to model
│                   │  Model adjusts its strategy accordingly
└────────┬─────────┘
         ↓
┌──────────────────┐
│  Parallel Exec    │  Concurrent via Promise.all
│                   │  Multiple tool calls run simultaneously
│                   │  Each has independent try/catch
│                   │  Single failure doesn't affect others
└────────┬─────────┘
         ↓
┌──────────────────┐
│  Post-Hook        │  Result sanitization, length truncation
│                   │  Audit logging
│                   │  Sensitive information filtering
└────────┬─────────┘
         ↓
tool_result appended to message history
Model decides next action based on results
```

**Parallel execution is the most critical design in this pipeline.** Traditional Agent implementations typically call tools serially: call A, wait for A's response, call B, wait for B's response. Claude Code's model can return multiple tool_use blocks in a single response, and the pipeline executes them simultaneously via `Promise.all`.

The effect in practice is dramatic. For instance, if the model needs to read three files to understand a feature's implementation, serial execution requires three IO waits, while parallel execution needs only one. In large projects, this parallelism can reduce tool execution time by over 50%.

**Error isolation is equally important.** Each tool call is wrapped in its own `try/catch`, so one tool's failure doesn't cascade to other parallel tools. Failed results are returned to the model as error messages, allowing it to take corrective action. This design gives the Agent **graceful degradation** capability.

## 3️⃣ The Skill System: Teaching the Agent New Abilities

### What Is a Skill?

**A Skill is an instruction manual written for an AI Agent.** It's a `SKILL.md` file that describes how to accomplish a specific task using natural language. When the Agent reads this file, it follows the instructions to perform the operation.

This concept has many parallels in the industry. OpenAI's GPTs let users customize behavior through Instructions. LangChain has Tool and Chain concepts. AutoGPT has a Plugin system. But Claude Code's Skill system makes a unique design choice: **Skills are natural language instructions, not code plugins.**

Why natural language instead of code? Code plugins must conform to specific API specifications, have high development barriers, and risk breaking every time the underlying framework upgrades. Natural language instructions are completely decoupled: as long as the Agent understands human language, a SKILL.md will always work. This lowers the barrier to creating Skills — anyone who can write documentation can create a Skill.

### Skill Discovery

```
Automatic scan at startup:
~/.claude/skills/*/SKILL.md  → User's global Skills
```

SKILL.md uses YAML frontmatter plus Markdown body:

```yaml
---
name: my-skill
description: Brief description of what this Skill does
allowed-tools:    # Optional, restricts which tools this Skill can use
  - Bash
  - Read
---
# Skill Instructions

## When to Use
Use this Skill when the user asks for ...

## Execution Steps
1. First, read ...
2. Then, execute ...
3. Finally, output ...
```

**The description field is the most critical metadata in the Skill system.** When the Agent chooses which Skill to use, it only sees a list of all Skill names and descriptions — it does not read the full SKILL.md content. Only after deciding to use a specific Skill does it read the complete instructions. This **two-phase loading** design prevents stuffing all Skill contents into the system prompt, which would cause context explosion.

### Skill Execution: Sub-Agent Isolation

When a Skill is invoked, Claude Code doesn't execute it directly in the main Agent loop. Instead, it **forks an independent sub-Agent**. This design mirrors the process isolation philosophy in operating systems.

```
User types /my-skill or Agent selects autonomously
  ↓
SkillTool takes control
  ↓
Creates an isolated sub-Agent
  - Independent token budget: won't exhaust the main Agent's quota
  - Independent message history: won't pollute the main conversation context
  - Restricted tool set: only safe tool subsets available
  ↓
Sub-Agent reads the full SKILL.md content
  ↓
Executes the task per instructions, possibly multiple tool call rounds
  ↓
Results returned to the main Agent
Main Agent continues subsequent work
```

**The sub-Agent's tool restrictions are the key to security.** Sub-Agents can only use Bash, File-related tools, Web-related tools, SkillTool, and AgentTool. They **cannot access permission management, MCP configuration, authentication**, or other sensitive tools. This means even if a malicious SKILL.md tries to make the Agent perform dangerous operations, the sub-Agent simply lacks the necessary tools.

**Recursive invocation:** Skills can call other Skills through SkillTool, enabling composition and reuse. For example, a "full-stack feature development" Skill could sequentially invoke a "backend API development" Skill and a "frontend page development" Skill.

### Skills in the System Prompt

All discovered Skills are injected into the system prompt as a summary list:

```
System prompt content:
...
Available skills:
- my-skill: What this Skill does
- review: Code review tool
- translate: Translation tool
...

Agent behavior rule: After selecting the best-matching Skill,
read its full SKILL.md first, then execute per instructions.
```

### Capacity Limits

The Skill system implements multiple capacity limits to prevent performance issues from an uncontrolled number of Skills:

| Limit | Default | Design Rationale |
|-------|---------|-----------------|
| Max scan candidates per source | 300 | Prevents filesystem scan from taking too long |
| Max loaded per source | 200 | Prevents excessive memory usage |
| Max listed in system prompt | 150 | Prevents Skill list from consuming too much context |
| Total Skill list character budget | 30,000 | Roughly 10% of the system prompt |
| Max single SKILL.md size | 256 KB | Prevents overly verbose Skill instructions |

## 4️⃣ Design Philosophy of the Tool System

Looking back at Claude Code's tool and Skill system, several core design principles emerge:

**Granularity over generality.** Search is split into Glob and Grep, file operations into Read, Edit, and Write — each tool with a single responsibility. This makes it easier for the model to make correct tool selections.

**Parallelism over serialization.** Tool execution supports concurrency, and the model can request multiple tool calls at once. This noticeably improves interaction speed in daily use.

**Isolation over sharing.** Skills execute via sub-Agents with independent budgets, histories, and tool sets. A problem in one Skill doesn't affect the main Agent.

**Natural language over code APIs.** Skills are described in Markdown files, not code plugins. This keeps the barrier to creating and maintaining Skills extremely low.

All these design choices point in the same direction: **making the Agent more reliable, more secure, and more extensible.** In the early days of AI Agents, reliability matters more than feature richness.

## 5️⃣ Horizontal Comparison with Other Agent Tool Systems

| Feature | Claude Code | GitHub Copilot | Cursor | Aider |
|---------|------------|---------------|--------|-------|
| Built-in tool count | 40+ | ~10 | ~15 | ~5 |
| Parallel tool calls | Yes | No | Partial | No |
| User-defined extensions | Skill system | None | None | None |
| External tool protocol | MCP | None | None | None |
| File editing approach | Precise replacement | Full file rewrite | Diff mode | Diff mode |
| Sub-Agent isolation | Yes | No | No | No |

Claude Code clearly leads in tool system completeness. However, more tools also mean a higher probability of the model picking the wrong one. Anthropic mitigates this through carefully crafted tool descriptions and usage guidelines in the system prompt, but this remains an area of ongoing optimization.

Next: [09-MCP Integration](/docs/09-MCP集成.md)
