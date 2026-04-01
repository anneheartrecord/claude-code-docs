[中文](./05-消息压缩系统.md)

# 05 Compaction System

## Why Compaction Is Needed

The longer an Agent's conversation goes, the larger the message history becomes. A complex task may involve dozens of tool calls, and each FileRead could return thousands of tokens of file content. Without compaction, the context window would be exhausted within just a few turns.

Claude Code's compaction system is located in `src/services/compact/`, containing 26 files. It is one of the most well-engineered modules in the entire codebase.

## Three-Layer Compaction Architecture

```
Before each API call
  ↓
Layer 1: Microcompact
  - Clean up old tool call results
  - Two modes: time-triggered / cached edits
  ↓
Check if token usage > context window - 13K?
  ↓ Exceeded
Layer 2: Session Memory Compact (tried first)
  - Persist key information to memory files
  - Delete old messages that have been persisted
  ↓ Not enough
Layer 3: Full Compact
  - Fork an independent Agent for full summarization
  - Nine-section structured template
  - Replace entire history with a single summary message
```

## Layer 1: Microcompact

### Trigger Conditions

Automatically executed before each API call, no threshold required.

### Scope

Only compacts output from specific tools — those that produce large but low-timeliness results:
- FileRead, FileEdit, FileWrite
- Bash (all shell tools)
- Grep, Glob
- WebSearch, WebFetch

User messages, assistant text, and thinking blocks are never touched.

### Two Implementation Approaches

**Time-based MC:**

If the time since the last assistant message exceeds the configured time threshold, the server-side prompt cache has expired. At this point, old tool_result contents are replaced with `[Old tool result content cleared]`, while the most recent N tool results are kept intact.

```
Trigger: minutes since last assistant message > gapThresholdMinutes
Retention: keep the most recent keepRecent items (minimum 1)
Action: tool_result.content = "[Old tool result content cleared]"
```

**Cached MC (feature flag: CACHED_MICROCOMPACT):**

Instead of modifying local message content, this approach uses the API layer's `cache_edits` mechanism to tell the server to delete the cache for specific tool_results. This preserves the existing prompt cache, resulting in better performance.

```
Trigger: number of registered tool_results > triggerThreshold
Retention: keep the most recent keepRecent items
Action: generate cache_edits blocks sent to the API, local messages unchanged
```

These two approaches are mutually exclusive. Cached MC takes priority; if unsupported (model doesn't support it or external build), it falls back to time-based MC.

## Layer 2: Session Memory Compact

### Trigger Conditions

Tried first when auto-compaction triggers. If Session Memory can free enough space, Full Compact is not needed.

### What It Does

Extracts key information from the conversation and writes it to a persistent session memory file, then deletes old messages that have already been recorded from the message history. Essentially, it "persists" message history to disk, freeing up context space.

## Layer 3: Full Compact

### Trigger Conditions

Token usage exceeds the context window minus a 13,000-token buffer, and Session Memory compaction is insufficient.

### Key Constants

```
AUTOCOMPACT_BUFFER_TOKENS = 13,000    // Trigger buffer
WARNING_THRESHOLD_BUFFER_TOKENS = 20,000  // Warning threshold
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000    // Maximum summary output
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3   // Circuit breaker threshold
```

### Execution Process

A separate Agent is forked to handle compaction. It's not done in the main loop to avoid blocking.

The compaction Agent receives the entire conversation history and generates a summary using a nine-section template:

```
1. User's main requests and intentions
2. Key technical concepts
3. Files and code snippets involved (with complete code)
4. Errors encountered and how they were fixed
5. Problem-solving process
6. All original user messages (non-tool-result ones)
7. Pending tasks
8. Work currently in progress
9. Next steps planned
```

### Two-Phase Generation: analysis + summary

The compaction prompt requires the LLM to first analyze and think within `<analysis>` tags, then write the formal summary within `<summary>` tags. Only the summary portion is injected into the context; the analysis is discarded.

This is a prompt technique to improve summary quality. It lets the LLM think things through before summarizing, without wasting context space.

### Two Compaction Directions

- **Full compaction (BASE):** Summarizes the entire conversation
- **Partial compaction (PARTIAL):** Only summarizes recent messages, preserving earlier already-summarized content. Split into `from` (starting from a certain message) and `up_to` (up to a certain message) directions

### Post-Compaction Processing

1. All old messages are deleted, replaced with a single user message containing the summary
2. The complete conversation is written to a transcript file as backup
3. Microcompact state, Session Memory pointers, and context collapse state are reset
4. Prompt cache detection is notified: the drop in cache hit rate was caused by compaction — don't raise false alarms

## Engineering Details

### Circuit Breaker

If auto-compaction fails 3 consecutive times, retries are stopped.

Code comment (BQ 2026-03-10): 1,279 sessions experienced 50+ consecutive compaction failures, with the highest being 3,272, wasting approximately 250,000 API calls per day. After adding the circuit breaker, this problem disappeared.

### Recursion Protection

Compaction itself is performed via LLM calls. What if the compaction Agent's context also overflows? The code includes explicit recursion guards:

```typescript
if (querySource === 'session_memory' || querySource === 'compact') {
  return false  // Do not trigger auto-compaction
}
```

Similarly, auto-compaction is not triggered in Context Collapse mode, as the two would compete with each other.

### Prompt Cache Awareness

Compaction destroys existing prompt cache. The code includes a dedicated `notifyCompaction` mechanism to inform the cache break detection module: the drop in cache hit rate was caused by compaction.

Production lesson (BQ 2026-03-01): After Session Memory compaction, the missing cache baseline reset caused 20% of cache break events to be false positives.
