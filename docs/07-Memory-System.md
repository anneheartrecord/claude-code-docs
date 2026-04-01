[中文](./07-记忆管理.md)

# 07 Memory System

## Two Types of Memory

Claude Code's memory is divided into two types: **Instruction Memory** and **Conversation Memory**.

**Instruction Memory (CLAUDE.md system):** Tells the Agent "how you should behave." Persists across sessions, manually maintained by users.

**Conversation Memory (MEMORY.md + Session Memory):** Records "what the Agent has done before." Automatically accumulated within a session, can persist across sessions.

## CLAUDE.md Five-Layer Loading

```
Priority from low to high:

1. Global Admin Level  /etc/claude-code/CLAUDE.md
   → Ops/admin configuration, shared by all users

2. User Level          ~/.claude/CLAUDE.md
   → Personal global preferences, effective across projects

3. Project Level       CLAUDE.md / .claude/CLAUDE.md / .claude/rules/*.md
   → Project conventions, checked into git, shared by team

4. Local Level         CLAUDE.local.md
   → Personal project config, gitignored, not shared

5. Auto Memory         feature flag: TEAMMEM
   → Team auto-synced memory (not yet released)
```

Higher-priority content overrides lower-priority content. Within the same level, multiple files under the rules directory are merged in alphabetical order.

## @include Directive

CLAUDE.md supports referencing external files via @include:

```markdown
# Project Standards
@./docs/coding-standards.md
@./docs/api-conventions.md

# Team Conventions
@~/.claude/team-rules.md
```

**Syntax rules:**
- `@path` — path relative to the current file
- `@./relative` — explicit relative path
- `@~/home` — user home directory
- `@/absolute` — absolute path

**Security restrictions:**
- Only parsed in leaf text nodes; `@` inside code blocks does not trigger
- Circular reference detection via a Set tracking processed files
- Supports 200+ file extensions, automatically blocks binary files
- Total character limit of 40,000

## MEMORY.md Management

MEMORY.md is the Agent's long-term factual memory. Unlike CLAUDE.md, it is written by the Agent itself.

**Key constraints:**
- Maximum 200 lines
- Maximum 25,000 bytes
- When exceeded, truncation is applied first by lines, then by bytes (cutting at the last newline)

**Truncation tracking:** The system records a `contentDiffersFromDisk` flag for cache deduplication. If MEMORY.md was truncated but the original on disk hasn't changed, it won't be written again.

**Related features:**
- `memoryAge.ts`: Tracks the age of memory entries
- `memoryScan.ts`: Scanning and pattern matching
- `EXTRACT_MEMORIES` feature flag: Automatically extracts key memories from conversations (not released)
- `MEMORY_SHAPE_TELEMETRY` feature flag: Memory shape telemetry (not released)

## Session Memory

Session-level short-term memory, used in conjunction with the compaction system.

Before Full Compact triggers, the system first attempts Session Memory Compact: persisting key information from the conversation to a session memory file, then deleting old messages that have been recorded from the message history. This frees up context space without losing key information.

## Memory Injection into System Prompt

```
system prompt =
  Built-in instructions
  + Global Admin Level CLAUDE.md
  + User Level CLAUDE.md
  + Project Level CLAUDE.md + rules/*.md
  + Local Level CLAUDE.local.md
  + MEMORY.md content
  + Current day's session memory
  + Skill list
  + Permission rules
```

An emphasis directive is attached during injection: **These instructions OVERRIDE default behavior**, ensuring that user-configured settings take priority over the model's default behavior.
