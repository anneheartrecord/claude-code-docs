[中文](docs/07-记忆管理.md)

# 07 Memory System

## Why AI Agents Need Memory

Human programmers rely on two kinds of knowledge at work. The first is long-accumulated experience and standards: "the team style guide requires 4-space indentation," "commit messages must be in English," "test coverage can't drop below 80%." The second is working notes from the current task: "which file was just modified," "what was the root cause of the bug found during the last debugging session," "what steps remain to be completed."

AI Agents need both kinds of knowledge too. But they face a problem that humans never encounter: **every time a conversation starts, their memory is blank**. Large language models have no persistent memory across sessions. All information accumulated during the previous conversation vanishes once the session ends. Without external memory system support, an Agent is like a newly hired employee every single time, needing to learn project standards and background from scratch.

Different Agent products in the industry handle the memory problem very differently. **GitHub Copilot** has long relied on the current file and recently opened tabs as context, with no persistent memory mechanism, only recently introducing instructions file capabilities. **Cursor** offers a `.cursorrules` file to carry project standards, but it's a single layer with no hierarchical override mechanism. **Aider** uses `.aider.conf.yml` for configuration, but this is more of a tool config than knowledge memory. **Devin** has a built-in knowledge base feature, but its structure is relatively opaque.

Claude Code's memory system achieves the most sophisticated design in the industry across hierarchy, controllability, and automation.

## 1️⃣ The Two Categories of Memory

Claude Code's memory divides into two major categories, each serving different responsibilities:

**Instruction Memory: The CLAUDE.md System**

This tells the Agent "how you should behave." This category of memory persists across sessions and is manually written and maintained by users. Its contents include code style standards, project architecture descriptions, team conventions, and personal preferences. Think of it as the onboarding guide and standards manual you'd give a new team member.

**Conversation Memory: MEMORY.md + Session Memory**

This records "what the Agent has done before." This category accumulates automatically within a session, and parts of it can persist across sessions. Its contents include completed operations, discovered issues, decisions made, and pending tasks. Think of it as a work log and meeting minutes.

The separation of these two categories is a significant design decision. Instruction memory is **declarative**, describing "what things should be like." Conversation memory is **procedural**, recording "what happened." Mixing them together leads to information management chaos: today's debugging notes shouldn't live in the same place as permanent code style standards.

## 2️⃣ CLAUDE.md Five-Layer Loading System

CLAUDE.md is the core carrier of Claude Code's instruction memory. Its loading system is divided into five tiers, from lowest to highest priority:

```
Priority from low to high:

1. Global Admin Level    /etc/claude-code/CLAUDE.md
   → Configured by ops/admins, shared across all users
   → Typical content: company-level security policies, compliance requirements

2. User Level            ~/.claude/CLAUDE.md
   → Personal global preferences, effective across all projects
   → Typical content: preferred programming languages, general coding habits

3. Project Level         CLAUDE.md / .claude/CLAUDE.md / .claude/rules/*.md
   → Project conventions, checked into git, shared by the team
   → Typical content: project architecture docs, API design standards, testing strategies

4. Local Level           CLAUDE.local.md
   → Personal project config, added to gitignore, not shared
   → Typical content: local environment-specific settings, personal dev preferences

5. Auto Memory           feature flag: TEAMMEM
   → Team auto-synced memory, not yet officially released
   → Design goal: let team members' Agents automatically share valuable discoveries
```

Higher-priority content overrides **conflicting** lower-priority content, while non-conflicting content from all levels is preserved. In practice, content from all tiers is concatenated together and injected into the system prompt, loaded in order from lowest to highest priority. When instructions from two tiers contradict each other, the higher-priority instruction appearing later wins, because LLMs have stronger attention weight on instructions positioned later in the context. Non-conflicting instructions from any tier all remain effective simultaneously. Within the same tier, multiple files under the rules directory are merged in alphabetical order.

The inspiration for this five-layer design is clear: **it is isomorphic to configuration management hierarchies in software engineering**. Consider CSS cascading rules, Kubernetes ConfigMap override chains, or Git's configuration levels. They all follow the same pattern: more specific configuration overrides more general configuration, local overrides global. Claude Code applies this well-proven pattern to Agent instruction memory management.

This hierarchical design solves a very common real-world need: **different projects within the same company require different Agent behaviors, yet share some company-wide universal standards**. Global admin level holds universal standards, project level holds project-specific standards, local level holds personal preferences, all without interference.

## 3️⃣ The @include Directive: Modular Memory Organization

When project standards are complex, stuffing everything into a single CLAUDE.md file makes it bloated and hard to maintain. The @include directive solves this by allowing CLAUDE.md to reference external files:

```markdown
# Project Standards
@./docs/coding-standards.md
@./docs/api-conventions.md

# Team Conventions
@~/.claude/team-rules.md
```

**Path syntax**:
- `@path` path relative to the current file
- `@./relative` explicit relative path
- `@~/home` user home directory
- `@/absolute` absolute path

This design lets teams split standards across different domains into independent files, each maintained by the respective domain owner. Frontend standards, backend standards, database standards, and CI/CD standards can each evolve independently, ultimately converging into CLAUDE.md through @include.

### Security Safeguards for @include

Introducing a file inclusion mechanism also introduces security risks. Claude Code has built comprehensive protections:

- **Code block immunity**: The @ symbol is only parsed in leaf text nodes; @ inside markdown code blocks does not trigger file inclusion. This prevents @ in code examples from being misinterpreted
- **Circular reference detection**: A Set data structure tracks already-processed file paths. If A includes B and B includes A, the detection mechanism stops on the second encounter with A, preventing infinite recursion
- **File type filtering**: Supports 200+ text file extensions, automatically blocking binary files. This prevents the Agent from accidentally loading compiled artifacts or image files as instructions
- **Total size limit**: All content brought in by @include has a total character cap of 40,000. This prevents a malicious or mistaken configuration from injecting massive text into the system prompt, which would both waste tokens and potentially override important instructions

## 4️⃣ MEMORY.md: Long-Term Memory Written by the Agent Itself

MEMORY.md has a fundamental difference from CLAUDE.md: **CLAUDE.md is written by humans for the Agent to read; MEMORY.md is written by the Agent for itself to read**.

When the Agent discovers information worth remembering during its work, such as user preferences, special project conventions, or recurring problem patterns, it proactively writes this information to MEMORY.md. At the start of the next session, this information gets loaded into the system prompt, allowing the Agent to "remember" what it learned previously.

### Capacity Limits

MEMORY.md has strict capacity limits:

- **Maximum 200 lines**
- **Maximum 25,000 bytes**
- When exceeded, truncation is applied first by lines, then by bytes, cutting at the last newline

Why the limits? Because MEMORY.md content gets injected into the system prompt of every conversation. Without limits, MEMORY.md would grow larger over time, consuming more and more context window space, eventually impacting the Agent's normal operation. 200 lines and 25,000 bytes represent a carefully balanced ceiling: enough to store dozens of valuable memories without excessively encroaching on context space.

### Truncation Tracking

The system records a `contentDiffersFromDisk` flag for cache deduplication. If MEMORY.md has been truncated in memory but the original file on disk hasn't changed, the system won't redundantly write to disk. This avoids unnecessary disk I/O from truncation operations and prevents file watchers from being falsely triggered.

### Evolving Features

Claude Code's source also contains several memory-related feature modules that reveal the system's evolutionary direction:

- **memoryAge.ts**: Tracks the age of each memory entry. Old entries may be outdated; tracking age lays the foundation for future "memory eviction" strategies
- **memoryScan.ts**: Scans memory content with pattern matching to identify duplicate, conflicting, or outdated entries
- **EXTRACT_MEMORIES feature flag**: Automatically extracts key memories from conversations, not yet officially released. Once enabled, MEMORY.md will evolve from "Agent proactively writes" to "system automatically extracts," dramatically reducing the mental overhead of memory management
- **MEMORY_SHAPE_TELEMETRY feature flag**: Memory shape telemetry that collects statistics on user memory usage to guide future optimizations

## 5️⃣ Session Memory: Session-Level Short-Term Memory

Session Memory is session-level short-term memory, deeply integrated with the compaction system.

In article 05, we covered the three-tier compaction architecture. Session Memory is the core mechanism of the second compaction tier: when context is about to overflow, the system first attempts to persist key information from the conversation into a session memory file, then deletes old messages that have been recorded.

The distinction between Session Memory and MEMORY.md lies in **lifecycle**:

- **MEMORY.md** is cross-session long-term memory; its content needs to remain valuable in future sessions
- **Session Memory** is intra-session short-term memory; it's only valid during the current conversation and can be cleaned up after the session ends

This separation ensures that long-term memory won't be polluted by short-term context compaction needs. A temporary variable value discovered during one debugging session shouldn't live alongside a long-term standard like "the project uses 4-space indentation."

## 6️⃣ Memory Injection into the System Prompt

All types of memory ultimately converge in the system prompt, forming the Agent's "initial knowledge" for each conversation:

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

A critical emphasis directive is attached during injection: **These instructions OVERRIDE default behavior**.

Why is this emphasis needed? Because large language models have their own default behavior patterns, derived from vast amounts of data during pre-training and RLHF stages. If a user configures instructions via CLAUDE.md that conflict with default behavior, the model might choose to ignore the user's instructions and follow its own defaults. The explicit OVERRIDE declaration is a **prompt-level priority marker**, telling the model "when instructions conflict, defer to these configurations."

This bears similarity to the **priority inversion** problem in operating systems. In real-time operating systems, resources held by low-priority tasks can block the execution of high-priority tasks. The solution is to use priority inheritance or priority ceiling protocols to ensure high-priority task instructions get executed. Claude Code's OVERRIDE declaration is essentially doing the same thing: ensuring user-configured priorities are higher than the model's built-in default behaviors.

## 7️⃣ Comparison with Industry Approaches

| Project | Instruction Memory | Conversation Memory | Hierarchy | Automation Level |
|---------|-------------------|--------------------|-----------|-----------------|
| **Claude Code** | CLAUDE.md five-layer system + @include | MEMORY.md + Session Memory | Five-layer override | Medium-high, Session Memory auto-persists |
| **Cursor** | .cursorrules single file | No persistence | Single layer | Low |
| **GitHub Copilot** | .github/copilot-instructions.md | None | Single layer | Low |
| **Aider** | .aider.conf.yml config | None | Single layer | Low |
| **Devin** | Built-in knowledge base | Limited session memory | Opaque | Medium |

Claude Code clearly leads the industry in memory system design. The five-layer loading system provides complete coverage from global to local. @include supports modular organization. The separation of MEMORY.md and Session Memory ensures long-term and short-term memory don't interfere with each other. Behind these design decisions lies a clear understanding: **an Agent's capability ceiling depends on how much correct context information it can access**. The memory system is the source of that context information, and its quality directly determines Agent performance.

---

Next: [08-Tools-and-Skills](docs/08-Tools-and-Skills.md)
