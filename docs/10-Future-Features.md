[中文](./10-未来功能蓝图.md)

# 10 Future Features Roadmap

## 1️⃣ The Technical Roadmap Behind 82 Feature Flags

In software development, a **Feature Flag** is a common release strategy. Developers write the code for a new feature but keep it deactivated, controlling whether it takes effect through a boolean switch. This lets teams safely merge incomplete features into the main branch and turn them on when the time is right.

Claude Code's source contains **82 Feature Flags**, controlled via the `feature('FLAG_NAME')` function. In the publicly available external build, this function always returns false, meaning all flagged features are disabled. But the code itself remains intact and fully readable.

This means we can read these disabled code paths to **glimpse Anthropic's technical roadmap**. These features may still be in testing, may already be in use internally, or may ultimately be abandoned. But they at minimum represent directions that Anthropic's engineering team has seriously considered.

## 2️⃣ Kairos: From Q&A Tool to Autonomous Agent

**Related Flags:** KAIROS, KAIROS_BRIEF, KAIROS_CHANNELS, KAIROS_DREAM, KAIROS_GITHUB_WEBHOOKS, KAIROS_PUSH_NOTIFICATION

Kairos is a comprehensive autonomous execution framework composed of 6 Feature Flags. Among all 82 flags, this group has the broadest scope and best reflects Anthropic's ambition.

**KAIROS** is the core flag. When enabled, the Agent can break free from step-by-step human instruction and enter an autonomous loop execution mode. Today's Claude Code operates on a "you give an instruction, it executes once" model. Kairos aims for "you give a goal, it works continuously until completion."

**KAIROS_BRIEF** enables briefing mode. While working autonomously, the Agent periodically reports progress to the user. This addresses a critical trust problem: if an Agent works silently in the background for half an hour, the user has no idea what it's doing, which creates anxiety. Periodic briefings keep users informed without interrupting the Agent's workflow.

**KAIROS_CHANNELS** supports multi-channel integration. Based on the flag name and code references, this likely means the Agent can interact through Slack, Discord, Teams, and other messaging platforms beyond just the terminal. Imagine sending Claude Code a message in Slack: "Fix this bug for me." It automatically pulls the code, analyzes the issue, commits a fix, opens a PR, and notifies you of the result in Slack.

**KAIROS_DREAM** is perhaps the most imaginative feature. Based on the name, this lets the Agent do background thinking and planning during idle time. Similar to how humans think about tomorrow's tasks even when not working, the Agent can use idle time for codebase analysis, context preloading, and pre-planning potential tasks.

**KAIROS_GITHUB_WEBHOOKS** lets the Agent listen to GitHub events and automatically trigger tasks. For example, when someone files an Issue, the Agent automatically starts analyzing and fixing it. When someone opens a PR, the Agent automatically conducts a Code Review. This transforms the Agent into a genuine team member rather than a passive tool.

**KAIROS_PUSH_NOTIFICATION** enables push notifications, alerting users on mobile devices or desktops about the Agent's work progress.

**If Kairos ships in full, Claude Code's positioning will fundamentally change.** It will transform from an interactive "you ask, it answers" tool into an autonomous "it works even when you don't ask" Agent. This aligns with the broader AI Agent industry direction: Devin, launched in 2024 and touted as "the first AI software engineer," emphasized exactly this kind of autonomous capability. But judging from Claude Code's code maturity, Anthropic's implementation may be considerably more systematic.

## 3️⃣ Context Collapse: Next-Generation Context Management

**Related Flag:** CONTEXT_COLLAPSE

Currently, Claude Code uses the autocompact mechanism for context management: when context approaches capacity limits, compression is triggered to summarize conversation history into a shorter version. This approach is simple and effective but has clear limitations. Compression is a lossy operation — critical details may be lost during the process — and compression timing depends entirely on a single metric: context utilization rate.

**Context Collapse is an entirely new context lifecycle management system** with a design philosophy fundamentally different from autocompact. Inferred from code comments:

- At **90% context usage**, triggers a commit: saves a snapshot of the current working state, similar to a Git commit, creating a recoverable checkpoint
- At **95% context usage**, triggers a blocking-spawn: blocks current work, starts a fresh context to continue execution, while maintaining a reference to the previous context snapshot
- **Mutually exclusive** with autocompact — they cannot be enabled simultaneously

The elegance of this design lies in transforming context management from "reactive compression" to "proactive management." Autocompact is like stuffing things into a closet when the room is full. Context Collapse is like moving to a new room while keeping the key to the old one.

**For complex tasks requiring extended runtime, Context Collapse could be a quantum leap.** Currently, when using Claude Code for large-scale refactoring, the biggest pain point is losing critical details after context compression. Context Collapse preserves complete historical state through its snapshot mechanism — even while working in a new context, you can trace back when needed.

## 4️⃣ Coordinator Mode: Multi-Agent Orchestration

**Related Flag:** COORDINATOR_MODE

When a task is sufficiently complex, a single Agent may struggle. Coordinator Mode lets Claude Code act as a coordinator for multiple Agents, decomposing large tasks into subtasks, distributing them to different sub-Agents for parallel execution, and consolidating results.

How does this differ from the existing AgentTool sub-Agent mechanism? AgentTool has the main Agent manually fork sub-Agents, delegating one small specific task at a time. Coordinator Mode operates at a higher level: understanding the global structure of the entire task, planning dependencies between subtasks, and managing parallel execution and result merging.

Similar approaches already exist in the industry. Microsoft's AutoGen framework supports multi-Agent conversations. CrewAI lets multiple Agents collaborate as a team. OpenAI's Swarm framework provides lightweight multi-Agent orchestration. But Claude Code's Coordinator Mode is integrated directly into the Agent's QueryEngine, deeply coupled with the tool system, permission system, and context management — potentially more efficient than standalone multi-Agent frameworks.

## 5️⃣ Voice Mode: Voice Interaction

**Related Flag:** VOICE_MODE

Voice Mode adds voice input and output support to Claude Code. The Agent is no longer limited to terminal text interaction — it can communicate with users through voice.

This might seem like a nice-to-have feature, but combined with Kairos autonomous mode, its significance grows substantially. **When an Agent is working autonomously in the background, voice is a very natural notification and interaction method.** You're drinking coffee, and the Agent tells you via voice: "PR has been submitted, all tests passed, awaiting your review." You reply by voice: "Change the log level to debug and run it again." This is far more natural than opening a terminal and typing.

## 6️⃣ Verification Agent: Built-in Quality Gatekeeper

**Related Flag:** VERIFICATION_AGENT

After completing a task, an independent verification Agent automatically launches to check whether results are correct. This is essentially an **Agent-to-Agent Review mechanism** implemented within the Agent itself.

Why does a separate Agent need to verify? Because the same Agent checking its own output suffers from "self-consistency bias" — it tends to believe its own work is correct. Using an independent Agent with a fresh context that examines results from scratch makes problems easier to spot.

This is fundamentally the same as Code Review in human engineering teams: the person who writes the code should not be the only person reviewing it.

## 7️⃣ Workflow Scripts: Predefined Automation

**Related Flag:** WORKFLOW_SCRIPTS

Lets users define automated workflow scripts. The Agent follows predefined steps to execute tasks rather than relying on the LLM to plan on the fly each time.

This solves a real problem: **LLM execution paths may vary each time.** For the same task, it might read files then search today, but search then read files tomorrow. For tasks requiring strict reproducibility — such as release processes, security audits, and compliance checks — this unpredictability is unacceptable.

Workflow Scripts let users codify proven workflows: what to do in step one, step two, which tools to use at each step, and how to handle failures. The Agent still handles the concrete execution of each step, but a human determines the orchestration.

## 8️⃣ Other Notable Flags

**PROACTIVE:** The Agent initiates tasks on its own. In the post-compaction recovery logic, there's a dedicated check: if in Proactive mode, the Agent continues working after compaction without greeting or asking the user. This hints at an entirely new usage pattern: the Agent proactively takes action without the user's knowledge, such as automatically fixing newly discovered lint errors or updating outdated dependencies.

**ULTRATHINK:** Super thinking mode. Based on the name, this likely gives the model more "thinking time" or "thinking tokens," enabling deeper reasoning on complex problems. This aligns with OpenAI's o1/o3 model chain-of-thought approach.

**TEAMMATES:** Team collaboration memory. Possibly allows multiple developers to share the Agent's memories and experiences — for example, operational techniques one team member teaches the Agent could benefit other members too.

**FAST_MODE:** Fast mode, likely using a lower-latency but slightly less capable model for simple tasks, trading quality for speed.

**CONTEXT_1M:** Million-token context support. This would dramatically improve the Agent's ability to handle large codebases. The current context limit is one of the biggest bottlenecks when an Agent works on complex projects.

## 9️⃣ Complete Feature Flag Category Table

### Agent & Workflow: 15

| Flag | Speculated Function |
|------|-------------------|
| PROACTIVE | Proactive execution mode |
| KAIROS | Autonomous execution framework |
| KAIROS_BRIEF | Periodic briefings |
| KAIROS_CHANNELS | Multi-channel access |
| KAIROS_DREAM | Background thinking during idle |
| KAIROS_GITHUB_WEBHOOKS | GitHub event triggers |
| KAIROS_PUSH_NOTIFICATION | Push notifications |
| AGENT_TRIGGERS | Agent triggers |
| AGENT_TRIGGERS_REMOTE | Remote triggers |
| WORKFLOW_SCRIPTS | Workflow scripts |
| FORK_SUBAGENT | Sub-Agent forking |
| COORDINATOR_MODE | Coordinator mode |
| VERIFICATION_AGENT | Verification Agent |
| BUILTIN_EXPLORE_PLAN_AGENTS | Built-in explore/plan Agents |
| AGENT_MEMORY_SNAPSHOT | Agent memory snapshots |

### Permissions & Security: 3

| Flag | Speculated Function |
|------|-------------------|
| TRANSCRIPT_CLASSIFIER | Transcript-based permission classification |
| BASH_CLASSIFIER | Bash command safety classifier |
| POWERSHELL_AUTO_MODE | PowerShell auto mode |

### Context & Memory: 5

| Flag | Speculated Function |
|------|-------------------|
| CONTEXT_COLLAPSE | Context collapse |
| EXTRACT_MEMORIES | Automatic memory extraction |
| MEMORY_SHAPE_TELEMETRY | Memory shape telemetry |
| TEAMMATES | Team collaboration memory |
| ULTRATHINK | Ultra-think mode |

### API & Performance: 13

| Flag | Speculated Function |
|------|-------------------|
| PROMPT_CACHE_BREAK_DETECTION | Cache hit rate monitoring |
| FAST_MODE | Fast mode with low-latency model |
| AFK_MODE | Away-from-keyboard mode |
| TOKEN_BUDGET | Token budget management |
| UNATTENDED_RETRY | Unattended retry |
| REACTIVE_COMPACT | Reactive compaction |
| CACHED_MICROCOMPACT | Cached edit-based micro-compaction |
| EFFORT | Effort level control |
| CONTEXT_1M | Million-token context |
| STRUCTURED_OUTPUTS | Structured outputs |
| TASK_BUDGETS | Task budgets |
| REDACT_THINKING | Redact thinking process |
| CONTEXT_MANAGEMENT | Context management |

### UI & Output: 9

| Flag | Speculated Function |
|------|-------------------|
| STREAMLINED_OUTPUT | Streamlined output mode |
| HISTORY_SNIP | History snipping |
| HISTORY_PICKER | History picker |
| REVIEW_ARTIFACT | Review artifact |
| AUTO_THEME | Auto theme |
| TERMINAL_PANEL | Terminal panel |
| MESSAGE_ACTIONS | Message actions |
| NATIVE_CLIPBOARD_IMAGE | Native clipboard image |
| WEB_BROWSER_TOOL | Web browser tool |

### Infrastructure: 10

| Flag | Speculated Function |
|------|-------------------|
| BRIDGE_MODE | Bridge mode |
| DAEMON | Daemon process |
| CCR_REMOTE_SETUP | Remote setup |
| CCR_AUTO_CONNECT | Auto connect |
| CCR_MIRROR | Mirror |
| VOICE_MODE | Voice mode |
| MCP_SKILLS | MCP Skill integration |
| CHICAGO_MCP | Chicago MCP service |
| UDS_INBOX | Unix Domain Socket inbox |
| BUDDY | Buddy mode |

## 🔟 Production Lesson Annotations: Engineering Wisdom in the Code

The code is scattered with production lesson annotations prefixed by **BQ**. These annotations document real production incidents — each one is engineering wisdom purchased with real costs.

| Date | Issue | Impact | Lesson |
|------|-------|--------|--------|
| BQ 2026-03-22 | Tool descriptions embedding dynamic lists caused cache breaks | **77% of tool cache breaks originated here** | Any dynamic content in the system prompt will break prompt cache. Tool descriptions must be static |
| BQ 2026-03-12 | Bridge connection failure patterns | **147K 404 errors + 22K WebSocket closures per week** | Network layer failures need graceful degradation strategies; failed connections can't keep retrying and wasting resources |
| BQ 2026-03-10 | Auto-compaction infinite loop | **1,279 sessions failed 3,272 times consecutively, wasting 250K API calls per day** | Any automated process needs a circuit breaker to prevent infinite failure loops |
| BQ 2026-03-01 | Cache baseline not reset after compaction | **20% of cache break events were false positives** | State resets must be synchronized with state changes; missing any reset point causes cascading errors |

**These annotations are themselves extremely valuable engineering wisdom.** They reveal which seemingly minor design details can cause severe production issues in large-scale AI Agent systems. The auto-compaction infinite loop incident on 2026-03-10 is particularly striking — wasting 250K API calls per day. At Claude API pricing, that translates to thousands of dollars per day in wasted spend.

For engineers building AI Agent systems, **these production lessons are more practically valuable than any design patterns textbook.**

Next: [11-AI Code Review](./11-AI-Coding时代的Code-Review.md)
