[中文](docs/04-上下文工程.md)

# 04 Context Engineering

## 1️⃣ What Is Context Engineering, and Why Is It the Achilles' Heel of Agents

In 2025, Shopify CEO Tobi Lütke posted an opinion on social media: we should stop saying **prompt engineering** and start saying **context engineering**. The idea spread rapidly through the tech community because it captured a core challenge in AI application development that had been chronically underestimated.

To understand context engineering, you first need to understand a fundamental limitation of large language models: **the context window is finite.**

The context window can be thought of as the model's **working memory**. When a human thinks about a complex problem, the amount of information the brain can actively hold in focus is limited. Large language models are the same. Claude's context window is approximately 200K tokens. That sounds large, but in a complex Agent task, this space gets consumed surprisingly fast.

Consider what Claude Code needs to fit into the context when handling a large code refactoring task: system instructions, the user's personalized configuration, project rules, the current Git repository status, the complete history of dozens of conversation turns, the results of every tool call, and definition documents for over 40 tools. All of this information competes for the same finite space.

The core problem that context engineering solves is: **how to fit the most valuable information into a limited space while ensuring the model's attention is not diluted by irrelevant content.**

This shares a surprising similarity with library management. A good librarian does not pile every book onto the reader's desk. Instead, based on the reader's current needs, they precisely retrieve the most relevant few books and arrange them in a helpful order. Context engineering does exactly this: it helps the model **filter information, prioritize, and dynamically adjust**.

In Claude Code's source code, context engineering is spread across multiple files, but the core philosophy can be distilled into a comprehensive system of **layered priorities + prefetch caching + dynamic injection**.

## 2️⃣ System Prompt Construction: The Model's Operating Manual

The system prompt is carried with every API call. Think of it as the **model's operating manual**. Claude Code's system prompt is not a static block of text. It is dynamically constructed through two phases.

### Static Parts: The Foundational Rules

The first phase loads foundational information that does not change across sessions:

- **Built-in Agent behavior rules**: Telling the model it is a programming assistant, how it should interact with users, what operations require permission, and when it should proactively provide more information
- **Available slash command list**: The model needs to know what shortcut commands users can use
- **Model configuration and thinking parameters**: Whether Extended Thinking is enabled, reasoning depth settings
- **Permission mode descriptions**: Which permission mode is currently active, which operations require user confirmation

### Dynamic Parts: Real-Time Intelligence

The second phase loads information that changes based on the current environment. This is the most refined part of context engineering.

**Git status injection** is implemented in the context.ts file. The system executes 4 git commands in parallel to gather information: the current branch, `git status --short` showing file changes, the most recent commit, and the current username. The output from these four commands is concatenated into structured text and injected into the system prompt.

Several engineering details here are worth noting. First, Git status output is **strictly capped at 2,000 characters**. If a large project has hundreds of changed files, the status output could run to thousands of characters. Cramming all of it into the system prompt would be wasteful. Excess content is truncated with a warning telling the model the information is incomplete. Second, Git status is fetched only once per session, with subsequent uses going through a **memoize cache**. Finally, in remote mode or when the user has explicitly disabled it, this step is skipped entirely.

**The CLAUDE.md memory system** is the most complex module in Claude Code's context engineering. The implementation lives in claudemd.ts and spans **1,400 lines**.

CLAUDE.md is a mechanism that lets users tell Claude about project rules and personal preferences. But Claude Code's design goes far beyond a simple config file. It establishes a **five-layer priority loading system**, ranked from low to high:

The first layer is **global admin level**, at the path `/etc/claude-code/CLAUDE.md`. This location is typically configured by enterprise IT administrators to enforce uniform rules across an entire organization. For example, "all code changes must include tests" or "direct operations on production databases are prohibited."

The second layer is **user level**, at `~/.claude/CLAUDE.md`. This is the user's personal global configuration, applying to all projects. For example, "I prefer TypeScript" or "write commit messages in Chinese."

The third layer is **project level**, including `CLAUDE.md` in the project root, `.claude/CLAUDE.md`, and all `.md` files in the `.claude/rules/` directory. This layer defines project-specific rules and is typically committed to Git for team sharing.

The fourth layer is **local level**, with the filename `CLAUDE.local.md`. This file is gitignored and dedicated to personal private configuration. For example, local API key paths or special development environment settings.

The fifth layer is **auto-memory**, controlled by the TEAMMEM feature flag. This is an experimental feature where the system automatically extracts valuable information from conversations and persists it.

The loading order of these five layers means higher-priority configurations override lower-priority ones. A project-level rule can override a user-level default, and a local-level configuration can override a project-level rule.

CLAUDE.md also supports a powerful **@include directive system** that allows one CLAUDE.md file to reference the contents of other files. The syntax supports four path formats: `@path` for relative paths, `@./relative` for explicit relative paths, `@~/home` for user directory paths, and `@/absolute` for absolute paths. The system only parses @include directives in **leaf text nodes** and never triggers them inside code blocks. It includes **circular reference detection** using a Set data structure to track already-processed file paths, preventing infinite recursion from A referencing B which references A. The system supports over 200 file extensions and automatically blocks binary files.

The entire memory system has several **key constants** governing its boundaries:

- `MAX_MEMORY_CHARACTER_COUNT = 40000`: Total character limit after merging all CLAUDE.md content
- `MAX_ENTRYPOINT_LINES = 200`: Maximum lines for MEMORY.md files
- `MAX_ENTRYPOINT_BYTES = 25000`: Maximum bytes for MEMORY.md files

These limits ensure the memory system does not bloat excessively and consume too much context window space.

**Permission rules** are loaded from two locations: `~/.claude/settings.json` for user-level settings and `.claude-permissions.json` for project-level settings. They are merged and parsed into a **ToolPermissionContext** data structure containing allow, deny, and ask rules. These rules are injected into the system prompt so the model knows which operations can be executed directly, which require user confirmation, and which are explicitly prohibited.

## 3️⃣ Layered Priority: When Space Runs Out, Who Goes First

One of the most critical design decisions in context engineering is **when space is insufficient, what to keep and what to discard**. Claude Code establishes a clear six-tier priority system:

```
Priority from high to low:

1. System instructions          ← Always retained, never compacted
2. CLAUDE.md user memory        ← Always retained, never compacted
3. Permission rules             ← Always retained, never compacted
4. Recent conversation          ← Compacted last
5. Historical tool call results ← First to be micro-compacted
6. Early conversation history   ← Replaced by summaries during Full Compact
```

There is deep product logic behind this priority design.

**The top three tiers are never compacted.** This means that no matter how many iterations the Agent has looped through and no matter how many times the context has been compressed, the user's personalized configuration, project rules, and permission settings remain fully intact. Imagine if permission rules were accidentally dropped during a compaction. The Agent might execute dangerous operations that should have been forbidden. Or imagine if the "write commit messages in Chinese" rule from CLAUDE.md were dropped. The Agent would suddenly switch to English commits. These behavioral inconsistencies would seriously damage user trust.

**The fourth tier is the recent conversation.** This is the model's direct basis for understanding the current task. Compacting it would cause the model to lose awareness of current work progress. So it is the last part to be touched.

**The fifth and sixth tiers are cleared first.** Earlier tool call results and conversation history have diminishing information value over time. File contents read an hour ago are likely no longer relevant. Replacing them with concise summaries frees substantial space for more valuable new information.

This priority system has an elegant property: **users benefit from it without needing to know it exists**. Users do not need to understand context windows, token limits, or compaction strategies. They just use Claude Code normally, and the system manages everything in the background. But when a user discovers that the Agent still remembers their project specifications from CLAUDE.md at the 30th loop iteration, that feeling of "it really understands me" is the direct result of this context engineering.

## 4️⃣ Prefetch and Caching: The Battle for Speed

Each iteration of the Agent loop needs to read context information. If every iteration started from scratch with I/O operations, performance would be terrible. Claude Code addresses this with a **prefetch + cache** system:

| Information Type | Prefetch Strategy | Cache Strategy |
|---------|---------|---------|
| Git status | Async prefetch at query loop start | Memoize within session, fetched once |
| CLAUDE.md | Async load at application startup | Cached within session, no re-reads |
| Skill list | Background discovery process | File watcher, hot-reload on change |
| Tool definitions | One-time registration at startup | Feature flag controlled, immutable at runtime |

The core idea of **prefetching** is to **let I/O happen before the demand arises**. Before the query loop actually needs Git status information, the prefetch operation has already been launched in the background. By the time the context-building stage needs this information, the data is ready with zero wait time.

The core idea of **caching** is to **avoid repeated work**. CLAUDE.md content rarely changes within a single session, so reading it once is sufficient. Git status is also stable in most scenarios. Through memoize caching, subsequent iterations directly use the previous result.

The Skill list caching strategy is unique. It uses a **file watcher** instead of simple memoize. Users may add new Skill files while the Agent is running, and the system needs to detect this change and hot-reload. But it does not rescan the filesystem every iteration. It only triggers an update when the watcher detects a change event.

The numerical impact of this prefetch and cache system is significant. In a 20-iteration Agent loop, if every iteration performed full I/O operations, assuming each read of CLAUDE.md, Git status, and Skill list takes 30 milliseconds, 20 iterations would mean 1.8 seconds of pure wait time. Through prefetching and caching, this time drops to near zero.

## 5️⃣ Dynamic Context Injection: The Right Information at the Right Time

Another core principle of context engineering is **do not dump all information into the model at once**.

This principle runs counter to many people's intuition. People often assume more information is always better for the model. But both research and practice show that excessive context information **dilutes the model's attention**, degrading its processing capability on critical information. Academics call this phenomenon **Lost in the Middle**, meaning that when context is very long, the model's attention to information in the middle portion drops significantly.

Claude Code's solution is to **dynamically inject different context based on the Agent's current stage**:

**During the Planning stage**, a global view is needed. The system injects the complete Skill list and project structure information so the model can formulate a comprehensive execution plan.

**During the Execution stage**, focus on the current task is needed. The system injects only file contents directly relevant to the task being processed, avoiding wasting precious context space on irrelevant information.

**During the Post-compaction Recovery stage**, rapid context reconstruction is needed. The system injects previously generated summaries along with paths to complete transcript files. If the model needs a specific detail that was compacted away, it can proactively call the file read tool to fetch it on demand, rather than having all details crammed in from the start.

The elegance of this design lies in how it **transforms context management into a continuous decision-making process**. At each loop iteration, the system re-evaluates based on the current state: what information is most important right now? What can be set aside for the moment? What should be removed entirely?

## 6️⃣ Prompt Cache Optimization: Saving Real Money

Anthropic's API features a **Prompt Cache** mechanism. If two API calls share the same request prefix, the second call can reuse the first call's computation cache, significantly reducing latency and cost. For an Agent that makes frequent API calls, Prompt Cache hit rate directly impacts operational costs.

Claude Code manages Prompt Cache at two levels.

**The first level is CacheSafeParams.** This is a set of parameters shared between parent and child Agents, including four fields: systemPrompt, userContext, systemContext, and toolUseContext. The system ensures these four fields remain perfectly consistent between the main Agent and all forked child Agents. This way, child Agent API calls can hit the Prompt Cache already established by the parent Agent.

The economic value of this design is very intuitive. Suppose the system prompt is 10,000 tokens. Without CacheSafeParams, every child Agent fork would need to reprocess those 10,000 tokens. In a complex task that might fork a dozen child Agents, that is 150,000 tokens of additional overhead. With CacheSafeParams, all of these can hit the cache, with savings potentially reaching several dollars per task.

**The second level is cache break detection.** The system has a dedicated monitoring module that continuously tracks Prompt Cache hit rates. If the hit rate drops suddenly, the system analyzes the cause: is it a genuine cache break from a code bug, or an **expected drop** from compaction or micro-compaction operations?

There is a production lesson from inside Anthropic worth sharing. A post-mortem analysis on March 22, 2026 found that **77% of tool-related cache breaks** were caused by a single issue: the description fields of AgentTool and SkillTool had **dynamic lists** embedded in them. Every time the available Agent or Skill list changed, the tool definition text changed, causing the system prompt hash to change and the Prompt Cache to invalidate.

This case vividly illustrates the difficulty of context engineering. A seemingly reasonable design choice, listing all available sub-tools in the tool description, turns out to be a performance killer from the Prompt Cache perspective. The fix was to remove the dynamic lists from tool descriptions and place them in session-level context instead, keeping the tool definitions themselves stable.

## 7️⃣ Comparison with Other Systems' Context Management

Different Agent systems invest vastly different amounts of effort in context management.

**LangChain's Memory module** provides several basic context management strategies: ConversationBufferMemory stores complete history, ConversationSummaryMemory retains summaries, and ConversationBufferWindowMemory keeps only the most recent N turns. These strategies are simple and intuitive but are all **one-size-fits-all** approaches. There is no layered priority, no prefetch caching, no dynamic injection. When context approaches the limit, the choice is either to keep everything or to truncate bluntly.

**OpenAI Assistants API** manages conversation history through Threads and provides some degree of automatic context management. But it is a black box. Developers cannot control compaction strategies, set priorities, or perform fine-grained cache optimization.

**Cursor** is reported to have invested heavily in context management, particularly in precise retrieval of code context. It uses **vector retrieval** technology to find code snippets most relevant to the current task. This is complementary to Claude Code's approach: one focuses on **retrieval precision**, the other on **full lifecycle management**.

**Claude Code's** context engineering holds the position of **most systematic** in the industry. The five-layer priority memory loading, six-tier compaction priority, prefetch caching system, dynamic injection strategy, and fine-grained Prompt Cache management form a complete end-to-end solution. The level of investment is reflected in the 1,400 lines of CLAUDE.md loading code and in the engineering discipline of building a dedicated detection module for cache breaks.

## 8️⃣ Summary

Context engineering is the **invisible pillar** of Agent product user experience. Users cannot see it, but it determines whether the Agent can still remember project rules at the 30th loop iteration, whether it can still understand the current task after compressing historical information, and whether it can maintain cost efficiency when forking child Agents.

Claude Code's intensity of investment in this area, in terms of both code volume and design refinement, is at the **top tier** of the industry. It demonstrates a key insight: in the AI Agent domain, **managing the information the model can see is just as important as the model's own capabilities.**

---

> Next: [05-Compaction System](docs/05-Compaction-System.md)
