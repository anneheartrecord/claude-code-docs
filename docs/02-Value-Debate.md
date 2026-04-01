[中文](./02-源码泄露的价值之争.md)

# 02 The Value Debate Over the Source Code Leak

## Two Schools of Thought

After the source code leaked, the community split into two camps.

**Camp One: The code itself is extremely valuable.** This is Anthropic's real internal Agent implementation — 510,000 lines of production-grade code covering a complete Agent architecture including ReAct loop, tool orchestration, permission system, context engineering, and message compaction. Every team building Agents can directly reference it and save months of trial and error.

**Camp Two: The code is just the output of Harness Engineering; the output doesn't matter, the Harness Engineering itself does.** You've got the blueprint for a sports car's parts, but you don't know how the factory that built the car operates. You can copy blueprints, but you can't copy the ability to create them.

## My Take: Both Are Right, but Camp Two Is Closer to the Truth

### The Code Does Have Reference Value

**The six-stage loop design in query.ts.** Prefetch → context build → streaming API call → tool execution → compaction → continue decision. The boundary condition handling at every stage represents battle-tested optimal solutions. For example, the auto-compaction trigger threshold is set at context window - 13K tokens; the persistence rules for thinking blocks between tool_use/tool_result — figuring out these details on your own could take weeks.

**The three-mode permission system.** Three modes — default/auto/plan — with layered defense through YOLO classifier + Bash pattern matching + path sandboxing. Most open-source Agent frameworks' permission systems are two orders of magnitude behind this.

**The message compaction mechanism.** A three-layer compaction system: micro-compact / Session Memory / Full Compact, plus circuit breaker, recursion protection, and Prompt Cache awareness. These are engineering problems that long-context Agents must solve, yet few discuss publicly.

### But Camp Two Is Closer to the Truth

**The code tells you what, not why.** Why is the auto-compaction threshold set at context window - 13K instead of - 5K or - 20K? Why does the permission system have three modes? The reasoning behind these decisions is invisible in the code.

**Many designs are environment-specific.** Claude Code is a terminal tool for individual developers. Its permission system assumes a human is sitting in front of the terminal ready to click confirm at any time. Move it to an enterprise multi-Agent orchestration system, and that assumption falls apart.

**Outputs become obsolete; capabilities don't.** When models get upgraded, context windows grow larger, and tool use reliability improves, many engineering patches are no longer needed. But the ability to design those patches retains its value forever.

**The real moat is the iteration speed of Harness design.** Anthropic's competitive advantage isn't in these 515,000 lines of code themselves — it's in their ability to rapidly adjust Harness strategies after every model upgrade. You can copy today's code, but most companies can't keep up with their iteration next month.

## Another Layer of Value in the Code

Beyond architectural reference, this codebase has another layer of value that's easy to overlook:

**It exposes a large number of in-development plans and upcoming features.** Behind the 82 feature flags lies Anthropic's complete thinking about the future of Agents: Kairos autonomous mode, Context Collapse, Voice Mode, Workflow Scripts, and Coordinator Mode.

**A top-tier Agent company's future technology roadmap may be more valuable than the existing architecture.** See [10-Future Feature Roadmap](./10-future-features.md) for details.

## An Analogy

Imagine someone open-sources the backtesting code for a trading strategy. Is the code valuable? Yes — you can learn how to build a backtesting framework and design risk control modules. But what actually makes money is the market insight behind the strategy and the ability to iterate continuously. Code is a snapshot; capability is dynamic.

**The code is worth reading and learning from, but don't overestimate it. What's truly valuable is the Harness Engineering capability that produced this code.**
