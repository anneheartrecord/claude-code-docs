[中文](./04-上下文工程.md)

# 04 Context Engineering

## Core Problem

The context window is finite. You must fit the most valuable information into a limited space while ensuring the model's attention isn't diluted.

Claude Code addresses this with a complete system of layered priorities + prefetch caching + dynamic injection.

## System Prompt Construction

Built dynamically in two phases:

### Phase 1: Static Parts
- Built-in Agent behavior rules
- Available slash command list
- Model configuration, thinking parameters
- Permission mode descriptions

### Phase 2: Dynamic Parts

**Git Status (context.ts):**
- Executes 4 git commands in parallel: current branch, status --short, recent commit, username
- Results capped at 2,000 characters; truncated with a warning if exceeded
- Memoize cache within session — fetched only once
- Skipped in remote mode (CCR) or when disabled by user

**CLAUDE.md Memory (claudemd.ts, 1,400 lines):**
- Five-layer priority loading (low to high):
  1. Global admin level `/etc/claude-code/CLAUDE.md`
  2. User level `~/.claude/CLAUDE.md`
  3. Project level `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`
  4. Local level `CLAUDE.local.md` (gitignored, private)
  5. Auto-memory (feature flag: TEAMMEM)

- **@include Directive System:**
  - Syntax: `@path`, `@./relative`, `@~/home`, `@/absolute`
  - Only parsed in leaf text nodes — never triggered inside code blocks
  - Circular reference detection via Set tracking of processed files
  - Supports 200+ file extensions, automatically blocks binary files

- **Key Constants:**
  - `MAX_MEMORY_CHARACTER_COUNT = 40000` (total character limit)
  - `MAX_ENTRYPOINT_LINES = 200` (MEMORY.md line limit)
  - `MAX_ENTRYPOINT_BYTES = 25000` (MEMORY.md byte limit)

**Permission Rules:**
- Loaded from `~/.claude/settings.json` and `.claude-permissions.json`
- Parsed into ToolPermissionContext containing allow/deny/ask rules

## Layered Priority

When context approaches the limit, each part is retained by priority:

```
Priority from high to low:

1. System instructions (built-in behavior rules)    ← Never compacted
2. CLAUDE.md user memory                            ← Never compacted
3. Permission rules                                 ← Never compacted
4. Recent user messages and assistant replies        ← Compacted last
5. Historical tool call results                     ← First to be micro-compacted
6. Early conversation history                       ← Replaced by summaries during Full Compact
```

This priority system ensures that even when context is compacted, the user's personalized configuration and behavior rules are never lost.

## Prefetch and Caching

Avoiding IO on every loop iteration is key to performance:

| Information | Prefetch Strategy | Cache Strategy |
|------|---------|---------|
| Git status | Async prefetch at query loop start | Memoize within session, fetched once |
| CLAUDE.md | Async load at startup | Cached within session, no re-read |
| Skill list | Background discovery | File watcher, hot-reload on change |
| Tool definitions | Registered at startup | Feature flag controlled, immutable at runtime |

## Dynamic Context Injection

Different context is injected at different stages to avoid dumping everything in at once:

- **Planning stage:** Inject full Skill list and project structure
- **Execution stage:** Only inject file contents relevant to the current task
- **Post-compaction recovery stage:** Inject summaries + transcript file paths (read on demand)

## Prompt Cache Optimization

Claude Code manages prompt cache with fine granularity:

**CacheSafeParams:** Parent and child Agents share a set of parameters including systemPrompt, userContext, systemContext, and toolUseContext. API requests can only hit the prompt cache when all these parameters match exactly.

**Cache break detection:** A dedicated detection module monitors cache hit rates. If they drop suddenly, it distinguishes between genuine breaks and normal drops caused by compaction/micro-compaction.

Production lesson (BQ 2026-03-22): 77% of tool cache breaks were caused by dynamic lists embedded in the descriptions of AgentTool and SkillTool — every time the list changed, the cache was invalidated.
