[中文](./10-未来功能蓝图.md)

# 10 Future Features Roadmap

## 82 Feature Flags

The Claude Code source contains 82 feature flags, controlled via `feature('FLAG_NAME')`. In the external build, `feature()` always returns false, so all features are disabled. But the code is still there, giving us a glimpse into Anthropic's technical roadmap.

## Most Noteworthy Future Features

### Kairos Autonomous Mode

**Related flags:** KAIROS, KAIROS_BRIEF, KAIROS_CHANNELS, KAIROS_DREAM, KAIROS_GITHUB_WEBHOOKS, KAIROS_PUSH_NOTIFICATION

This is a complete autonomous execution framework. Inferred from flag names and code references:

- **KAIROS:** The Agent no longer needs to be driven step-by-step by humans — it can autonomously loop and execute tasks
- **KAIROS_BRIEF:** Briefing mode — the Agent periodically reports progress to users
- **KAIROS_CHANNELS:** Multi-channel access (possibly supporting Slack, Discord, etc.)
- **KAIROS_DREAM:** Background thinking/planning during idle time
- **KAIROS_GITHUB_WEBHOOKS:** Listening to GitHub events to auto-trigger tasks
- **KAIROS_PUSH_NOTIFICATION:** Push notifications

If this feature set ships, Claude Code transforms from a "you ask, it answers" tool into an "it works even when you don't ask" autonomous Agent.

### Context Collapse

**Related flag:** CONTEXT_COLLAPSE

Inferred from code comments, this is a new context management system designed to replace autocompact:

- Triggers a commit (saves current state) at 90% context usage
- Triggers a blocking-spawn (blocks and starts a new context) at 95%
- Mutually exclusive with autocompact — they cannot be enabled simultaneously

This is more granular than the current "compress-and-summarize" approach. Instead of waiting until context is nearly full before compressing, it proactively manages the context lifecycle.

### Coordinator Mode

**Related flag:** COORDINATOR_MODE

Enables Claude Code to act as a coordinator for multiple Agents. Similar in concept to our openclaw-orchestrator, but implemented within Claude Code's own QueryEngine.

### Voice Mode

**Related flag:** VOICE_MODE

Voice input and output. The Agent becomes more than a terminal tool — it can interact via voice.

### Verification Agent

**Related flag:** VERIFICATION_AGENT

Automatically launches a verification Agent after task completion to check whether the results are correct. Essentially a built-in Agent-to-Agent review.

### Workflow Scripts

**Related flag:** WORKFLOW_SCRIPTS

Allows users to define automated workflows. The Agent executes according to predefined script steps rather than relying on the LLM to plan on the fly each time.

### Proactive Mode

**Related flag:** PROACTIVE

The Agent initiates tasks proactively instead of waiting for user instructions. In the post-compaction recovery logic, there's a dedicated check: if in Proactive mode, the Agent continues working after compaction without greeting or asking the user.

## Complete Feature Flag Categories

### Agent / Workflow (15)

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

### Permissions / Security (3)

| Flag | Speculated Function |
|------|-------------------|
| TRANSCRIPT_CLASSIFIER | Transcript-based permission classification |
| BASH_CLASSIFIER | Bash command classifier |
| POWERSHELL_AUTO_MODE | PowerShell auto mode |

### Context / Memory (5)

| Flag | Speculated Function |
|------|-------------------|
| CONTEXT_COLLAPSE | Context collapse |
| EXTRACT_MEMORIES | Automatic memory extraction |
| MEMORY_SHAPE_TELEMETRY | Memory shape telemetry |
| TEAMMATES | Team collaboration memory |
| ULTRATHINK | Ultra-think mode |

### API / Performance (13)

| Flag | Speculated Function |
|------|-------------------|
| PROMPT_CACHE_BREAK_DETECTION | Cache hit rate monitoring |
| FAST_MODE | Fast mode (low-latency model) |
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

### UI / Output (9)

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

### Infrastructure (10)

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

## Production Lesson Annotations

The code is scattered with production lesson annotations prefixed with "BQ", documenting real production incidents:

| Date | Issue | Impact |
|------|-------|--------|
| BQ 2026-03-22 | Tool descriptions embedding dynamic lists caused cache breaks | 77% of tool cache breaks originated here |
| BQ 2026-03-12 | Bridge connection failure patterns | 147K 404s + 22K ws closures per week |
| BQ 2026-03-10 | Auto-compaction infinite loop | 1,279 sessions failed 3,272 times consecutively, wasting 250K API calls per day |
| BQ 2026-03-01 | Cache baseline not reset after compaction | 20% of cache break events were false positives |

These annotations are themselves extremely valuable engineering lessons.
