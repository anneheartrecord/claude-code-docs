[中文](./01-架构总览.md)

# 01 Architecture Overview

## One-Line Summary

Claude Code is a ReAct Agent based on AsyncGenerator that interacts with users through a terminal UI. Its core loop is "build context → call API → execute tools → decide whether to continue."

## Tech Stack

| Layer | Technology |
|---|------|
| Runtime | Bun 1.3.11+ |
| Language | TypeScript 6.0.2 |
| UI | React 19.2.4 + ink (terminal rendering) |
| CLI | Commander.js 14.0.0 |
| API | Anthropic SDK + AWS Bedrock + Azure + Vertex |
| Lint | Biome 2.4.10 |
| Bundler | Bun native bundler |

## Overall Architecture

```
┌─────────────────────────────────────────────┐
│          CLI Entry (cli.tsx)                 │
│      Fast path: --version etc. return early  │
└────────────────┬────────────────────────────┘
                 │
        ┌────────▼─────────┐
        │  Init (init.ts)   │
        │ - Bootstrap        │
        │ - Load config      │
        │ - Prefetch context │
        └────────┬─────────┘
                 │
     ┌───────────▼──────────────┐
     │  Commander CLI (main.tsx) │
     │  - Auth: login/logout     │
     │  - Plugins: install/list  │
     │  - MCP: add/remove        │
     │  - Main logic: REPL/print │
     └───────────┬──────────────┘
                 │
    ┌────────────▼─────────────────┐
    │   QueryEngine                │
    │  - Session state machine      │
    │  - Message history management │
    │  - Auto-compaction scheduling │
    │  - 20+ config parameters     │
    └────────────┬─────────────────┘
                 │
    ┌────────────▼─────────────────┐
    │   Query Loop (query.ts)      │
    │  ┌──────────────────────┐    │
    │  │ 1. Prefetch context   │    │
    │  │ 2. Build system prompt│    │
    │  │ 3. Stream API call    │    │
    │  │ 4. Parse tool calls   │    │
    │  │ 5. Permission + exec  │    │
    │  │ 6. Compact + continue?│    │
    │  └──────────────────────┘    │
    └────────────┬─────────────────┘
                 │
    ┌────────────▼──────────────────┐
    │  Tool Execution Layer (40+ tools) │
    │  Bash / File series / Web series  │
    │  Agent / Skill / MCP / Task       │
    └──────────────────────────────┘
```

## Core File Reference

| File | Lines | Responsibility |
|------|------|------|
| `src/query.ts` | 1,700 | Main query loop, streaming orchestration |
| `src/QueryEngine.ts` | 1,300 | Session state machine, 20+ config parameters |
| `src/main.tsx` | 5,000+ | CLI bootstrap, permission initialization |
| `src/Tool.ts` | 400+ | Tool interface definition |
| `src/tools.ts` | 300+ | Tool registry, feature flag control |
| `src/context.ts` | 200+ | Git status, context assembly |
| `src/services/api/claude.ts` | 800+ | API client, retry, model fallback |
| `src/utils/permissions/` | 6,300+ | Three-mode permission system |
| `src/services/compact/` | 26 files | Three-layer message compaction |
| `src/services/mcp/` | 12,000+ | MCP protocol integration |
| `src/utils/claudemd.ts` | 1,400+ | CLAUDE.md five-layer loading |

## Data Flow

A complete user request follows this path:

```
User input
  ↓
REPL capture → Determine if it's a slash command or natural language
  ↓
QueryEngine.query() → Assemble system prompt + message history
  ↓
Query loop begins
  ↓
Prefetch: memory files, Skill list, Git status (skip if cache hit)
  ↓
Build API request: system prompt + messages + tools + thinking config
  ↓
Stream call to Anthropic API → Render tokens to terminal in real time
  ↓
Parse response: plain text → output directly | tool_use → enter tool execution
  ↓
Tool execution: Pre-Hook → Permission check → Parallel execution → Post-Hook
  ↓
Append tool_result to message history
  ↓
Compaction check: token usage over threshold? → Micro-compact / Session Memory / Full Compact
  ↓
Continue check: stop_reason === 'tool_use' → continue loop | 'end_turn' → finish
  ↓
Output final result
```

## Was It Written by AI?

Yes. All 20 commits come from claude-code-best, with three commits bearing `Co-Authored-By: Claude Opus 4.6`. 515,000 lines of code compiled with zero errors on the first try. See [02-Value-Debate](./02-Value-Debate.md) for details.
