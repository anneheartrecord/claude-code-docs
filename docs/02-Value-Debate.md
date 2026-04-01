[中文](/docs/02-源码泄露的价值之争.md)

# 02 The Value Debate: What Is Leaked Source Code Really Worth?

## 1️⃣ Background: An Accidental Open-Sourcing

In March 2026, Anthropic's AI programming tool **Claude Code** was found to be fully recoverable through deobfuscation of its shipped JavaScript bundles. The result: **515,000 lines of production-grade TypeScript code** covering Agent loops, tool orchestration, permission management, context engineering, message compaction, memory systems, and virtually every other core module in the AI Agent domain.

The timing made this especially explosive. Between 2025 and 2026, the tech industry was in the **critical commercialization window for AI Agents**. OpenAI, Google, Microsoft, and countless startups were all pouring resources into Agent product development. Everyone was wrestling with the same fundamental question: **how do you turn a large language model from a chatbot into an autonomous agent that can complete complex tasks?**

Against this backdrop, one of the world's top AI companies had the complete implementation of its flagship Agent product exposed to the public. It was, in effect, an **involuntary open-sourcing**. The community quickly split into two camps, debating the real value of this code.

## 2️⃣ Camp One: The Code Itself Is Extremely Valuable

People in this camp saw the leak as a **textbook-grade Agent engineering reference implementation**.

To understand their excitement, you need to know the state of the industry. In early 2026, Agent frameworks on the market fell roughly into three categories.

**The first category was academic frameworks.** Tools like LangChain and early versions of AutoGPT. They provided basic proof-of-concept for Agent architectures but were far from production-ready. Error handling was rough, performance optimization was absent, and permission management was practically nonexistent.

**The second category was cloud vendor Agent platforms.** AWS Bedrock Agents, Azure AI Agent Service, and similar offerings. They targeted enterprise markets with full infrastructure support, but exposed only highly abstracted APIs. Developers could not see internal implementation details.

**The third category was startup Agent products.** AI coding tools like Devin, Cursor, and Windsurf. They went deep in specific scenarios but were entirely closed-source.

This is what made the Claude Code leak so significant: **it was the first time a complete, battle-tested, production-grade Agent system's internals had been fully exposed.**

Specifically, Camp One considered the following modules most valuable:

**The six-stage loop design in query.ts** is the core engine of the entire system. Prefetch, context construction, streaming API calls, tool execution, compaction, and continuation decisions form six tightly interconnected stages. The boundary condition handling at each stage represents hard-won lessons from real user scenarios. For example, the auto-compaction trigger threshold is set at **context window minus 13K tokens**, and there is a rule that **thinking blocks must be persisted between tool_use and tool_result**. Discovering these details independently could take weeks or months.

**The three-mode permission system** is another highlight. The default, auto, and plan modes work with a YOLO classifier, Bash pattern matching, and path sandboxing in a layered defense. Most open-source Agent frameworks either have no permission system at all or just a simple allowlist. Claude Code's permission system is **at least two orders of magnitude more sophisticated**.

**The three-tier message compaction mechanism** solves a core pain point for long-conversation Agents. Micro-compaction, Session Memory, and Full Compact work in a progressive hierarchy, combined with circuit breakers, recursion protection, and Prompt Cache awareness. How to compress historical information without losing critical context is an engineering problem that every long-context Agent must solve, yet the industry rarely discusses it openly.

## 3️⃣ Camp Two: Harness Engineering Capability Is What Matters

Camp Two's core argument can be summed up in one sentence: **you got the blueprint for a sports car's parts, but you have no idea how the factory that built it operates.**

This requires explaining a concept. **Harness**, in the AI Agent domain, refers to the entire engineering framework built around a large language model. This includes prompt design, tool orchestration, context management, error recovery, user interaction, and all other non-model code. The Harness **harnesses** a raw language model into a usable product.

Camp Two argued that code is merely a snapshot of Harness engineering at a single point in time. You can copy the snapshot, but you cannot copy the engineering capability that produced it.

Their arguments were compelling.

**Code tells you what was done, but not why.** Why is the auto-compaction threshold exactly context window minus 13K? What happens if you choose minus 5K or minus 20K? Why does the permission system have three modes instead of two or four? The reasoning behind these design decisions, including what experiments were run, what alternatives were rejected, and what data-driven trade-offs were made, is completely absent from the code.

**Many designs are environment-specific.** Claude Code is a terminal tool for individual developers. Its permission system has an implicit assumption: a human is sitting at the terminal, ready to click confirm or reject at any time. This assumption is perfectly reasonable for the product's form factor. But if you try to port this code into an enterprise multi-Agent orchestration system, the assumption completely breaks down. In unattended automation scenarios, you need an entirely different permission model. Directly copying Claude Code's implementation would only create more problems.

**Artifacts become obsolete; capabilities do not.** Large language models evolve extremely fast. Context windows have grown from 100K to 200K and may reach 1M in the future. Tool use reliability is continuously improving. Model reasoning capabilities make significant leaps with each generation. Many of the carefully designed engineering patches in today's code, such as complex message compaction strategies and various fallback and retry logic, may become unnecessary with the next generation of models. But the **ability to design these patches**, meaning the engineering capability to identify bottlenecks, design solutions, rapidly validate, and continuously iterate, will always be valuable.

**The real moat is iteration speed.** Anthropic's competitive advantage is not in the 515,000 lines of code themselves, but in their organizational ability to **rapidly adjust Harness strategies** after each model upgrade. They have the deepest model understanding, the richest user feedback data, and the shortest **hypothesis-experiment-validation** cycles. You may have copied today's code, but most companies cannot keep pace with their iteration rhythm next month.

## 4️⃣ My Assessment: Both Are Right, but Camp Two Gets Closer to the Truth

The two camps' views are not contradictory. The code's reference value is real, especially for teams building Agent systems from scratch. But if you have to rank them, Camp Two's insight is more profound.

A financial industry analogy works well here. Imagine someone open-sources the complete backtesting code for a quantitative trading strategy. Is the code valuable? Of course. You can learn how to build a backtesting framework, how to design risk control modules, and how to implement an order management system. But what actually makes money in the market is the **market insight** and **continuous iteration capability** behind the strategy. Market conditions change, and strategies must change with them. A static code snapshot can never keep up with a dynamically evolving team.

The Agent domain is the same. **Code is a snapshot; capability is dynamic.**

## 5️⃣ The Overlooked Third Layer of Value: The Technical Roadmap

Beyond architectural reference and engineering methodology, this code hides a layer of information that is easy to overlook but may hold the greatest strategic value.

The codebase contains **82 feature flags** that reveal Anthropic's complete thinking about the future of AI Agents. Most of these feature flags are in a disabled state, representing features under development but not yet released.

Several are particularly noteworthy:

- **Kairos Autonomous Mode**: Letting the Agent run autonomously in the background for extended periods, evolving from a human-supervised assistant to an independently operating agent
- **Context Collapse**: An entirely new context management strategy that could fundamentally change how long-conversation Agents are implemented
- **Voice Mode**: Expanding the Agent from a text terminal to a voice interface
- **Workflow Scripts**: Letting users define complex multi-step workflows via scripts
- **Coordinator Mode**: One Agent directing multiple sub-Agents to work collaboratively

These feature flags paint a clear picture: Anthropic is evolving Claude Code from a programming assistant into a **general-purpose autonomous Agent platform**.

**The future technical roadmap of one of the world's top AI Agent companies may be more valuable than the existing architecture itself.** See [10-Future Features](/docs/10-Future-Features.md) for details.

## 6️⃣ Summary

The answer to this value debate is actually simple: **the code is worth reading, worth studying, but should not be overestimated.**

For practitioners in the Agent space, this code's value exists on three levels:

- **Tactical level**: Specific engineering implementations can be directly referenced, saving weeks of trial and error
- **Strategic level**: The methodology and design philosophy of Harness engineering deserve deep study
- **Intelligence level**: The future technical directions revealed by feature flags are worth continuous tracking

But ultimately, **what is truly valuable is the Harness Engineering capability that produced this code**. Code can be copied; capability cannot.

---

> Next: [03-Agent Loop](/docs/03-Agent-Loop.md)
