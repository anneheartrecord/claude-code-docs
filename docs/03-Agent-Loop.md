[中文](./03-Agent循环.md)

# 03 The Agent Loop

## 1️⃣ What Is an Agent Loop, and Why Does It Matter So Much

Before diving into Claude Code's Agent loop, we need to understand a fundamental concept: **large language models are inherently stateless.** You send in some text, it returns some text, and the interaction is over. It does not decide what to do next on its own. It does not proactively read files. It does not execute commands by itself.

So here is the question: how does Claude Code manage to receive a task and then independently read code, write files, run tests, discover errors, fix them, and potentially loop through dozens of iterations before completing the job?

The answer is the **Agent loop**. It is a piece of program logic that runs outside the language model, responsible for continuously asking the model questions, parsing its answers, executing the operations the model requests, feeding the results back, and then letting the model decide what to do next. This loop keeps running until the model determines the task is complete.

If you think of the large language model as a brilliant brain locked in a room, the Agent loop is the **body that carries information and executes actions for it**. The brain says "I want to see this file," and the body goes and fetches the file contents. The brain says "change this line of code to that," and the body goes and makes the edit.

This concept has a formal academic name: **ReAct**, short for Reasoning + Acting. The model reasons first, then takes action, observes the result, reasons about the next step, and so on in a loop. Nearly all modern Agent systems are built on this paradigm. The differences lie only in how refined the implementation is.

## 2️⃣ Claude Code's Core Design Choice: AsyncGenerator

Claude Code's Agent loop is implemented in `src/query.ts`, roughly **1,700 lines of code**. The entire loop skeleton is built using JavaScript's **AsyncGenerator**.

This technology choice is worth examining in detail because it directly shapes the system's architecture.

In JavaScript, an AsyncGenerator is a special kind of function. A normal function runs to completion and returns a result when called. An AsyncGenerator can **pause mid-execution, hand over an intermediate result, wait for the caller to process it, and then resume**. This mechanism is called **yield**.

This is a perfect match for an Agent loop. Consider the Agent's workflow: the model is generating a response, and each token needs to be displayed on the terminal in real time; the model requests a dangerous operation and the system needs to pause for user confirmation; after a tool finishes executing, the result needs to be sent back to the model to continue generation. Every one of these scenarios requires the ability to **pause, yield control, wait, and resume**, and AsyncGenerator supports all of this natively.

For comparison, implementing the same thing with traditional callback functions would result in deeply nested callback hell. Using a Promise chain would make real-time rendering of streaming output awkward. AsyncGenerator lets Claude Code's main loop read like synchronous code while retaining full asynchronous capability.

Other Agent frameworks in the market have made different choices. Early versions of LangChain used a **callback chain pattern**, registering callback functions for each step: flexible but hard to debug. AutoGPT used a simple **while loop with synchronous calls**: clear but incapable of streaming output. Cursor's internal implementation reportedly uses an **event-driven architecture**: fast response times but complex state management. Claude Code's choice of AsyncGenerator strikes an excellent balance between clarity, streaming capability, and state management.

## 3️⃣ The Six-Stage Loop: The Agent's Heartbeat

Each iteration of Claude Code's loop is divided into six stages. Think of it as the Agent's heartbeat: each beat completes a full **think-act-observe** cycle.

```
┌──────────────────────────────────────────────────────────┐
│                      query loop                           │
│                                                          │
│  ┌────────────────┐                                      │
│  │ 1. Prefetch    │ Memory files, Skill list (async)     │
│  └────┬───────────┘                                      │
│       ↓                                                  │
│  ┌────────────────┐                                      │
│  │ 2. Build       │ system prompt + messages + tools      │
│  └────┬───────────┘                                      │
│       ↓                                                  │
│  ┌────────────────┐                                      │
│  │ 3. API Call    │ Streaming request, yield per token    │
│  └────┬───────────┘                                      │
│       ↓                                                  │
│  ┌────────────────┐                                      │
│  │ 4. Tool Exec   │ Permission check → parallel exec → Hook │
│  └────┬───────────┘                                      │
│       ↓                                                  │
│  ┌────────────────┐                                      │
│  │ 5. Compaction  │ Over threshold? Micro/SM/Full Compact │
│  └────┬───────────┘                                      │
│       ↓                                                  │
│  ┌────────────────┐                                      │
│  │ 6. Continue?   │ tool_use → loop | end_turn → finish  │
│  └────────────────┘                                      │
└──────────────────────────────────────────────────────────┘
```

### Stage 1: Prefetch

Before making the actual API call, the system **asynchronously loads** several types of information in parallel:

- **Memory files**: User preferences and project rules from CLAUDE.md and MEMORY.md
- **Skill list**: Available skills discovered in the background
- **Git status**: Current branch, file changes, recent commits

Why have a separate prefetch stage? Because these operations involve reading from the filesystem or spawning subprocesses. These are **I/O operations**, meaning they read from disk or execute external commands. I/O operations are slow but do not consume CPU. If these reads were executed sequentially within the main loop, each iteration would waste dozens of milliseconds waiting. By initiating them asynchronously before the loop starts, the data is ready by the time it is actually needed.

Another key design is **session-level caching**. This information rarely changes within a single session, so it only needs to be fetched once. Subsequent iterations use the cached data directly, skipping the I/O entirely. This optimization seems simple, but in an Agent task that might loop twenty or thirty times, the cumulative time savings are substantial.

### Stage 2: Build Context

This stage assembles the complete request to send to the API. It is the **most information-dense** part of the entire loop. The request contains four core components:

**system prompt** provides the model with foundational instructions. It is dynamically assembled from multiple sources: built-in Agent behavior rules, user memory from CLAUDE.md, current Git repository status, and permission mode descriptions. This component's design is extremely refined. See [04-Context Engineering](./04-Context-Engineering.md) for details.

**messages** is the complete conversation history. This includes messages sent by the user, the model's previous replies, and results from all prior tool calls. This is the model's **primary source of information** for understanding the current task's progress.

**tools** is the list of tools the model can call. Claude Code registers **over 40 tools**, with feature flags controlling which ones are available. Each tool definition includes a name, description, and parameter schema. The model uses this information to decide which tool to invoke.

**thinking** contains the Extended Thinking configuration parameters. When enabled, the model performs an internal reasoning pass before generating its final response, and this reasoning is included in the return.

### Stage 3: Stream API Call

A streaming request is sent via the Anthropic SDK. **Streaming** means the model does not wait until the entire response is generated before returning it. Instead, **each token is returned the moment it is generated**. The Agent loop uses yield to pass each token one by one to the terminal UI above, so users see text appearing character by character rather than staring at a blank screen.

There is also an important fault-tolerance mechanism here: the **model fallback chain**. The API client has built-in retry logic. If the primary model call fails, it tries Opus, Sonnet, and Small in sequence. This ensures the Agent can continue working even if a specific model experiences a temporary outage.

This stage may seem technically simple, but it is critical to user experience. The **sense of immediate feedback** provided by streaming output is an important psychological factor in making users believe the Agent is working. Even if the results were identical, having to wait 10 seconds for a complete response would significantly degrade the experience.

### Stage 4: Tool Execution

When the model's response contains a **tool_use block**, it means the model has decided to call one or more tools. This is the critical turning point where the Agent transitions from "thinking" to "acting."

The tool execution flow is carefully designed with six steps:

```
tool_use blocks (tool call requests from model output)
  ↓
Pre-Hook (logging, parameter format validation)
  ↓
Permission check (based on default/auto/plan modes, determine if user confirmation is needed)
  ↓
Tool lookup (match by name in the tool registry)
  ↓
Parallel execution (Promise.all, multiple tools run simultaneously)
  ↓
Post-Hook (result sanitization, audit logging)
  ↓
tool_result appended to message history
```

**Parallel execution** is an important performance optimization. If the model requests reading three files simultaneously, all three read operations proceed in parallel rather than queuing up one after another. For complex tasks, this can significantly reduce total wait time.

**Permission checking** is the core safety mechanism. Based on the current permission mode and the specific tool type, the system decides whether to execute directly, auto-approve, or show a confirmation dialog. See [06-Permission System](./06-Permission-System.md) for details.

**Pre-Hook and Post-Hook** are interception points before and after tool execution. Pre-hooks can validate parameters and log operations; post-hooks can sanitize tool output and write audit logs. This **aspect-oriented** design separates the core tool execution logic from peripheral concerns, making the code easier to maintain.

### Stage 5: Compaction Check

After each tool execution completes, the system checks whether the current **token consumption** is approaching the context window limit. If the threshold is exceeded, a compaction operation is triggered.

This stage addresses a practical problem: when an Agent handles complex tasks, it may loop dozens of times, appending new content to the message history each iteration. Without any cleanup, the message history continuously grows until it exceeds the model's context window limit, causing the Agent to crash.

Claude Code designed a three-tier progressive compaction strategy: **micro-compaction** clears detailed output from earlier tool calls, **Session Memory** persists key information to durable storage, and **Full Compact** compresses the entire conversation history into a summary. See [05-Compaction System](./05-Compaction-System.md) for specifics.

### Stage 6: Continue Decision

Finally, the system decides what to do next based on the **stop_reason** field returned by the API:

- **tool_use**: The model needs to call more tools. Return to Stage 1 and continue the loop.
- **end_turn**: The model considers the task complete. Exit the loop.
- **max_tokens**: The model's single output hit the length limit. It may have more to say and needs to continue.

This decision mechanism may seem simple, but it gives the model the **ability to autonomously judge whether a task is complete**. The model does not follow preset steps. It dynamically decides whether to continue based on the result of each step. This is the fundamental difference between an Agent and a traditional automation script.

## 4️⃣ State Management: Immutable Updates

The Agent loop needs to track a large amount of state information during operation. Claude Code maintains **7 core state variables**:

```typescript
let state = {
  messages,                  // Complete message history
  toolUseContext,           // Tool execution context
  maxOutputTokensOverride,  // Dynamic output token limit
  autoCompactTracking,      // Compaction state tracking
  // ... 3 more
}
```

The key design decision is adopting an **immutable update pattern**. Instead of directly modifying the existing state object each iteration, a new copy is created:

```typescript
state = { ...state, messages: newMessages }
```

This pattern is already a best practice in frontend development. React and Redux use it extensively. But in an Agent loop, it provides an additional important benefit: **traceability**. When bugs occur, you can trace back through complete state snapshots for each loop iteration, precisely identifying which iteration and which field change caused the anomaly. For a complex Agent that may loop dozens of times, this debugging capability is essential.

## 5️⃣ Extended Thinking Preservation Rules

Extended Thinking is a special capability of the Claude model: it performs deep reasoning before generating the final response. This reasoning appears as a **thinking block** in the API return.

The code contains extensive comments explaining **three rules** that thinking blocks must follow:

1. Thinking blocks may only exist in requests where `max_thinking_length > 0`
2. A thinking block cannot be the last element in a block sequence
3. Thinking blocks must be **persistently preserved** throughout the entire assistant trajectory and never discarded

The third rule is the most critical and the easiest to get wrong. In an Agent loop, when the model calls tools, tool_use and tool_result entries alternate in the conversation history. If previous thinking blocks are discarded during this process, **the model's reasoning chain breaks**. It loses the context of its earlier thought process, and the quality of subsequent decisions degrades significantly.

This is a classic example of an engineering detail that **seems trivial but has massive impact**. Many Agent frameworks simplistically clean up historical messages during tool calls to save tokens, inadvertently deleting thinking blocks in the process. Claude Code has dedicated preservation logic ensuring thinking blocks persist intact throughout the entire conversation lifecycle.

## 6️⃣ The Agent Fork Mechanism

Claude Code supports **forking child Agents** from the main Agent during execution to handle specific tasks:

```
Main Agent
  ├── fork → Skill Agent (executes specific skills, independent token budget and message history)
  ├── fork → Compact Agent (generates compaction summaries)
  └── fork → Session Memory Agent (distills memories)
```

You can think of forking as creating a clone. When the main Agent encounters a sub-task that will take a long time, or a scenario requiring context isolation, it creates a child Agent to handle it. The child Agent has its own independent token budget and message history and will not pollute the main Agent's context.

There is a particularly elegant design here called **CacheSafeParams**. Anthropic's API has a **Prompt Cache** mechanism: if two requests share the same system prompt prefix, the second request can reuse the first request's cache, saving substantial computation costs. CacheSafeParams ensures the child Agent's system prompt and context parameters remain consistent with the parent Agent's, allowing the child Agent's API calls to hit the Prompt Cache already established by the parent.

This means forking child Agents is economically efficient. You do not need to pay the system prompt token cost again for each child Agent. This design is crucial for scenarios that require frequent child Agent forking.

## 7️⃣ Comparison with Other Agent Implementations

Comparing Claude Code's Agent loop with other well-known implementations in the industry reveals its design characteristics more clearly:

**LangChain AgentExecutor** is currently the most widely used open-source Agent framework. Its loop structure is a synchronous while loop that processes intermediate events through callback chains. The advantage is a rich ecosystem and low barrier to entry. The disadvantage is that callback chains become difficult to debug in complex scenarios. It has no built-in compaction or caching mechanism, making it easy to overflow tokens in long conversations.

**AutoGPT** was one of the first projects to ignite the Agent hype. Its loop is extremely simple and direct: call the model, parse the command, execute the command, feed the result back. The advantage is ease of understanding. The disadvantage is no streaming output, no permission system, no state management, making it very difficult to use in actual production.

**OpenAI Assistants API** takes a server-managed Run approach. Developers create a Run, and the server handles the loop execution until completion. The advantage is that developers do not need to manage the loop themselves. The disadvantage is high opacity, limited customization space, and inability to fine-tune for specific scenarios.

**Claude Code's Agent loop** sits at the **highest level of refinement** among these implementations. The clear six-stage division, AsyncGenerator streaming capability, immutable state management, three-tier compaction, and Prompt Cache-aware fork mechanism have all been deeply polished. The trade-off is that **complexity is also the highest**: 1,700 lines of main loop code far exceeds other implementations.

## 8️⃣ Summary

At its core, Claude Code's Agent loop does one thing: **it turns a stateless language model into an autonomous agent with memory, the ability to act, and the capacity to judge.** It is the heart of the entire system, and each beat passes through the six stages of prefetch, build, call, execute, compact, and decide.

This loop's design reveals a core reality of current Agent engineering: **model capability is the foundation, but the engineering framework that converts model capability into a reliable product experience is equally critical.** The same underlying model, wrapped in Agent loops of different quality, can produce dramatically different end products.

---

> Next: [04-Context Engineering](./04-Context-Engineering.md)
