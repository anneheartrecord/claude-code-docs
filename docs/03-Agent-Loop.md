[中文](./03-Agent循环.md)

# 03 Agent Loop

## Core Design: AsyncGenerator + ReAct

Claude Code's Agent loop is implemented in `src/query.ts` (1,700 lines). At its core is a ReAct loop based on AsyncGenerator.

Using AsyncGenerator for the main loop is an elegant choice. It natively supports streaming output — callers can consume events one by one or collect them all at once. The pause/resume semantics of generators naturally fit Agent scenarios like waiting for tool results or waiting for user confirmation.

## Six-Stage Loop

```
┌──────────────────────────────────────────────────┐
│                  query loop                       │
│                                                  │
│  ┌──────────┐                                    │
│  │ 1. Prefetch │ Memory files, Skill list (async) │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 2. Build  │ system prompt + messages + tools   │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 3. API    │ Streaming call, yield per token    │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 4. Tools  │ Permission check → parallel exec → Hook │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 5. Compact│ Over threshold? Micro/SM/Full Compact │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 6. Continue? │ tool_use → loop | end_turn → finish │
│  └──────────┘                                    │
└──────────────────────────────────────────────────┘
```

### Stage 1: Prefetch

Before calling the API, several types of information are loaded asynchronously:
- Memory files (CLAUDE.md, MEMORY.md)
- Skill list (background discovery)
- Git status (with memoize cache)

The benefit of prefetching is avoiding IO waits in the main loop. Most of this information doesn't change within a session, so once cached, it's skipped entirely.

### Stage 2: Build Context

Assemble the complete request to send to the API:
- system prompt = built-in instructions + CLAUDE.md memory + Git status + permission rules
- messages = user messages + conversation history + tool results
- tools = registered tool list (40+, controlled by feature flags)
- thinking = Extended Thinking configuration

### Stage 3: Stream API Call

Initiate a streaming request via the Anthropic SDK, yielding tokens one by one to the upper layer. The upper layer (terminal UI) renders tokens in real time as they arrive.

The API client has retry logic and a model fallback chain: Opus → Sonnet → Small.

### Stage 4: Tool Execution

After the API returns tool_use blocks, tool execution begins:

```
tool_use blocks
  ↓
Pre-Hook (logging, parameter validation)
  ↓
Permission check (default/auto/plan — three modes)
  ↓
Tool lookup (match by name in registry)
  ↓
Parallel execution (Promise.all, multiple tools can run simultaneously)
  ↓
Post-Hook (result sanitization, audit logging)
  ↓
Append tool_result to message history
```

### Stage 5: Compaction Check

Check whether the current token usage exceeds the threshold. See [05-Compaction System](./05-compaction-system.md) for details.

### Stage 6: Continue Decision

Based on the `stop_reason` returned by the API:
- `tool_use`: More tool calls to execute — continue looping
- `end_turn`: The model considers the task complete — finish
- `max_tokens`: Output hit the limit — may need to continue

## State Management

The loop maintains 7 state variables, creating a new state copy on each iteration:

```typescript
let state = {
  messages,                  // Message history
  toolUseContext,           // Tool execution context
  maxOutputTokensOverride,  // Output token limit
  autoCompactTracking,      // Compaction tracking state
  // ... 3 more
}

// Create new copy each iteration, never mutate the original
state = { ...state, messages: newMessages }
```

## Extended Thinking Preservation

The code contains extensive comments explaining three rules for thinking blocks:

1. Thinking blocks must exist in requests where max_thinking_length > 0
2. A thinking block cannot be the last element in a block sequence
3. Thinking blocks must be persistently preserved throughout the entire assistant trajectory

This means that when tool calls occur, thinking blocks cannot be discarded — they must be correctly preserved between tool_use and tool_result. Otherwise, the model's reasoning chain will break.

## Agent Fork Mechanism

Claude Code supports forking child Agents to execute Skills or parallel tasks:

```
Main Agent
  ├── fork → Skill Agent (independent token budget, independent message history)
  ├── fork → Compact Agent (generates compaction summaries)
  └── fork → Session Memory Agent (distills memories)
```

The key design in forking is **CacheSafeParams**: the child Agent's system prompt and context must match the parent Agent's. This ensures prompt cache hits and avoids redundant billing.
