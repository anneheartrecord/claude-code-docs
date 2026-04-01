[中文](05-消息压缩系统.md)

# 05 Compaction System

## Why Every AI Agent Hits a Compression Wall

Every large language model has a hard limit: the **context window**. Think of it as the model's "working memory capacity," similar to how much information a human brain can hold simultaneously while solving a problem. The total amount of text a model can process in a single conversation is capped. Claude's context window is 200K tokens, GPT-4o's is 128K tokens. These numbers sound large, but in Agent scenarios they get consumed remarkably fast.

Why so fast? Because Agents are fundamentally different from regular chatbots. In a normal chat, a human and a model exchange messages of a few dozen to a few hundred tokens each. But an Agent needs to read files, execute commands, and search codebases. Each tool call can return thousands of tokens. A single `FileRead` on a 500-line source file easily eats up 2,000-3,000 tokens. A complex programming task might involve **dozens of tool calls**, each stuffing large amounts of content into the context. At this rate, a dozen rounds of interaction can fill up that 200K window.

Other Agent projects in the industry face exactly the same problem. **Cursor** splits long conversations into multiple independent requests, sending only the most relevant context fragments each time. **Aider** uses a repo map approach, maintaining a structured summary of the codebase to avoid stuffing complete file contents into context. Early versions of **Devin** simply truncated old messages, causing the Agent to frequently "forget" what it had done previously.

Claude Code's solution is the most sophisticated in the industry to date: **a three-tier progressive compaction system**. Its code lives in `src/services/compact/`, spanning 26 files, making it one of the most heavily engineered modules in the entire Claude Code codebase.

## 1️⃣ The Three-Tier Compaction Architecture

Before diving into each tier, here's the big picture:

```
Before every API call
  ↓
Tier 1: Microcompact
  - Clean up stale tool call results
  - Runs every time, no threshold needed
  ↓
Check if token usage exceeds safety line
  ↓ Exceeded
Tier 2: Session Memory Compact
  - Persist key information to memory files
  - Delete old messages that have been persisted
  ↓ Still not enough space
Tier 3: Full Compact
  - Spawn independent Agent for full summarization
  - Nine-section structured template
  - Replace entire history with one summary message
```

The design philosophy across these three tiers is **progressive degradation**. Tier 1 has the lowest cost and runs every time. Tier 2 has moderate cost and only kicks in when approaching the limit. Tier 3 has the highest cost and serves as the last resort. This layered strategy avoids the extreme of "either no compression at all or compress everything at once," striking a balance between information retention and space reclamation.

## 2️⃣ Tier 1: Microcompact

### Core Idea

Microcompact has a simple goal: **clean up tool call results that are no longer useful**.

Imagine you ask the Agent to read a file, then modify it, then read the modified version. The result from the first read becomes stale after the modification. Keeping it in context is pure waste. Microcompact automatically cleans up this kind of stale content.

### Scope

Microcompact only processes output from specific tools that share one trait: **high output volume with low temporal relevance**:

- **FileRead, FileEdit, FileWrite**: File operations whose content may have been overwritten by subsequent edits
- **Bash and all shell tools**: Command execution results that are typically only relevant at execution time
- **Grep, Glob**: Search results that may become inaccurate as files change
- **WebSearch, WebFetch**: Web content that has already been digested by the Agent

Several content types are **never touched** by microcompact: user messages, assistant text responses, and thinking blocks. These represent either the user's original intent or the model's reasoning process. Losing them would be irreversible.

### Two Implementation Approaches

Claude Code implements two microcompact mechanisms. They solve the same problem through different technical paths.

**Time-Based Microcompact**

This is the more straightforward approach. When the time since the last assistant message exceeds a configured threshold, it means the Anthropic API's server-side **prompt cache** has expired. Prompt cache is an API-layer performance optimization: if you send multiple consecutive requests with largely identical prefix content, the server caches that prefix to avoid redundant processing, reducing latency and cost. But this cache has a TTL. Once expired, that old content will need reprocessing anyway, so it might as well be cleaned up.

The implementation replaces old tool_result content with `[Old tool result content cleared]`, keeping the most recent N tool results untouched. This way the model knows "there was a tool call here but its content has been cleared," avoiding confusion from suddenly missing messages.

```
Trigger: minutes since last assistant message > gapThresholdMinutes
Retention: keep the most recent keepRecent results, minimum 1
Action: tool_result.content = "[Old tool result content cleared]"
```

**Cached Microcompact**

This is a more elegant approach, controlled by the feature flag `CACHED_MICROCOMPACT`. Instead of modifying local message content, it uses the API's `cache_edits` mechanism to tell the server "please delete the cache for these specific tool_results."

The advantage is that it **preserves existing prompt cache**. Time-based microcompact modifies message content, which changes the prompt's hash and invalidates all cache. Cached microcompact only tells the server "drop this piece," while cache for everything else remains valid. Better performance overall.

```
Trigger: registered tool_result count > triggerThreshold
Retention: keep the most recent keepRecent
Action: generate cache_edits block for API, local messages unchanged
```

The two approaches are **mutually exclusive**, with cached microcompact taking priority. If the model doesn't support cache_edits, or if it's an external build, the system falls back to time-based microcompact automatically.

## 3️⃣ Tier 2: Session Memory Compact

### Motivation

When microcompact isn't enough to control context growth, the system needs more aggressive measures. But jumping straight to full summarization is expensive: it requires a separate LLM call, and the summarization process itself loses detail. Is there a middle ground?

Session Memory Compact is that middle ground. The idea is similar to how humans take notes: **write down the important stuff in a notebook, then clear the raw information from your head**.

### Workflow

When auto-compaction triggers, the system **tries Session Memory Compact first**:

1. Scan message history to identify key information: important decisions, discovered issues, user preferences, file modification records, etc.
2. Write this information to a persistent session memory file
3. Delete old messages from history that have already been recorded
4. Check if the freed space is sufficient

If Session Memory frees enough space, compaction stops here. No need to enter the more expensive Full Compact.

### How It Differs from Full Compact

Session Memory Compact preserves **structured key information**, while Full Compact generates **narrative summary text**. The former is more precise but limited in coverage; the latter covers more ground but loses detail. Together they form a strategy of "precise persistence first, comprehensive summarization if needed."

## 4️⃣ Tier 3: Full Compact

### Trigger Conditions

Full Compact activates when token usage exceeds the **context window minus a 13,000-token buffer**, and Session Memory Compact hasn't freed enough space.

Key constants:

```
AUTOCOMPACT_BUFFER_TOKENS = 13,000       // trigger buffer
WARNING_THRESHOLD_BUFFER_TOKENS = 20,000 // warning threshold
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000   // max summary output
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3 // circuit breaker threshold
```

The 13,000-token buffer ensures the model has enough **generation space**. If compaction only triggers when the context is completely full, the model wouldn't have room to generate a response. The 20,000-token warning threshold is an earlier alert, signaling "getting close to full."

### Independent Agent Execution

A critical design decision: **compaction is performed by an independently forked Agent**, not in the main loop.

Why? Because compaction itself is an LLM call. Running it in the main loop would make users wait for compaction to finish before they can continue interacting. By forking an independent Agent, the main loop can immediately feedback "compacting context" to the user while asynchronously waiting for results.

This independent compaction Agent receives the entire conversation history and generates a structured summary using a **nine-section template**:

```
1. User's primary requests and intentions
2. Key technical concepts
3. Files and code snippets involved, with complete code
4. Errors encountered and how they were fixed
5. Problem-solving process
6. All original user messages, excluding tool results
7. Pending tasks
8. Current work in progress
9. Next steps planned
```

This template is carefully designed. Sections 1-5 are **retrospective**, recording "what happened." Section 6 **preserves user verbatim messages**, which is crucial because users' original phrasing often contains subtle intentions that summaries can't capture. Sections 7-9 are **forward-looking**, ensuring the Agent knows what to do next after compaction.

Many other Agents' compression strategies only do retrospective summarization. After compression, the Agent "forgets" what it was doing and needs users to remind it. Claude Code's nine-section template solves this by explicitly including "current work" and "next steps."

### analysis + summary Two-Phase Generation

The compaction prompt requires the LLM to first analyze and think inside `<analysis>` tags, then write the formal summary inside `<summary>` tags. Only the summary portion is kept in context; the analysis is discarded.

This is a classic **Chain-of-Thought** application. The model first "thinks through" which information matters and which can be discarded, then writes the summary. The analysis phase may produce thousands of tokens of reasoning, but none of it needs to be retained in the final context. It's purely an intermediate step to improve summary quality.

### Two Compaction Directions

- **Full compaction BASE**: Summarizes the entire conversation history, generating a single new summary message to replace all old messages
- **Partial compaction PARTIAL**: Summarizes only a portion of messages. Split into `from` direction and `up_to` direction: the former means "summarize from a certain message onward," the latter means "summarize from the beginning up to a certain message." Partial compaction preserves unsummarized messages verbatim

A typical partial compaction scenario: the first half of the conversation was already compacted once, and the second half is new content. Only the second half needs summarizing, then it gets concatenated with the first half's summary.

### Post-Compaction Processing

After compaction completes, there's a series of cleanup tasks:

1. **Message replacement**: All old messages are deleted and replaced with a single user message containing the summary
2. **Transcript backup**: The complete original conversation is written to a transcript file, serving as a safety net for this irreversible operation
3. **State reset**: Microcompact state, Session Memory pointers, and context collapse state are all reset
4. **Cache notification**: The prompt cache detection module is notified that "this cache hit rate drop was caused by compaction, don't raise a false alarm"

## 5️⃣ Engineering Details

### Circuit Breaker

If auto-compaction fails 3 consecutive times, it **stops retrying**. Behind this seemingly simple mechanism lies a real production incident:

Code comments record a problem discovered on March 10, 2026: 1,279 sessions experienced 50+ consecutive compaction failures, with the worst case reaching **3,272 failures**. Each failure triggered an API call for retry, costing the system roughly **250,000 wasted API calls per day**. After adding the circuit breaker, this problem vanished completely.

The circuit breaker is a classic pattern from microservices architecture. Netflix open-sourced the Hystrix circuit breaker library in 2012, and the pattern has since been widely adopted in distributed systems. Seeing it applied in LLM Agent context management signals that this field's engineering maturity is beginning to match traditional backend systems.

### Recursion Protection

Compaction itself is performed via an LLM call. What if the compaction Agent's own context overflows? Would it trigger compaction of the compaction Agent, creating infinite recursion?

The code includes explicit recursion guards:

```typescript
if (querySource === 'session_memory' || querySource === 'compact') {
  return false  // do not trigger auto-compaction
}
```

Similarly, auto-compaction is disabled in **Context Collapse** mode. Context Collapse is another context management strategy that would compete with auto-compaction for resources. Running both simultaneously leads to unpredictable behavior.

### Prompt Cache Awareness

Compaction destroys existing prompt cache because message content changes, naturally invalidating cached hashes. The code includes a dedicated `notifyCompaction` mechanism that notifies the cache break detection module when compaction occurs.

This mechanism addresses an **observability problem**. Anthropic's internal monitoring systems track prompt cache hit rates. A sudden drop usually indicates a bug somewhere. But compaction-induced drops are normal and expected. Without distinguishing the two, operations teams would be overwhelmed by false alarms.

Code comments record a production lesson from March 1, 2026: a missing cache baseline reset after Session Memory Compact caused **20% of cache break events to be false positives**.

### Comparison with Industry Approaches

| Project | Compaction Strategy | Information Retention | Engineering Complexity |
|---------|-------------------|----------------------|----------------------|
| **Claude Code** | Three-tier progressive: Microcompact + Session Memory + Full Compact | High, nine-section template preserves key info | 26 files, heavily engineered |
| **Cursor** | Conversation splitting, sends only relevant fragments | Medium, depends on relevance judgment | Moderate |
| **Aider** | Repo Map structured summaries | Medium, only preserves structural info | Lower |
| **OpenAI Codex CLI** | Simple truncation + summarization | Low, early messages are lost | Lower |

Claude Code's approach ranks highest in both information retention and engineering complexity, consistent with its positioning as a long-running autonomous Agent. An Agent that needs to complete complex programming tasks within a single session demands far more from context management than a tool that only does code completion.

---

Next: [06-Permission-System](06-Permission-System.md)
