[中文](docs/13-啃完源码之后的一些发现.md)

# 13 Findings from Devouring 510K Lines of Source Code, and Claude's Ban Mechanism

## Beyond the Serious Analysis

The previous 12 articles were all serious teardowns of architecture and modules. This one is a bit different — it's about the things hidden in the corners of the code that truly shocked me or made me laugh. It also discusses Claude's ban mechanism based on what the source code reveals.

## 1️⃣ About the Source of This Code

An important clarification first. The GitHub repository you see is not Anthropic's original repository. It's a fork created by a developer who reconstructed the code from sourcemaps.

The reconstructed code lacked the build scaffolding and couldn't run directly. The developer did extensive repair work to get it to compile: filling in missing type definitions, fixing compilation errors, cleaning up `any` annotations, and more. The commits with `Co-Authored-By: Claude Opus 4.6` indicate that Claude was used to assist with the repair process.

**These commit records reflect the repair work, not Anthropic's original development history.** We cannot determine Anthropic's internal development process, commit conventions, or whether they used AI-assisted coding from this repository alone.

## 2️⃣ The Source Leak Itself Is Evidence of AI's Engineering Shortcomings

The entire source code leaked because sourcemap files weren't excluded when publishing to npm.

AI wrote the functionality flawlessly — zero type annotation errors, airtight error handling. But it stumbled on an engineering detail like publish configuration. It wouldn't proactively think: Are there files in the build artifacts that shouldn't be there? Is the `.npmignore` configured correctly? Should we do an artifact audit before publishing?

**AI is an engineer with exceptional ability but terrible discipline.** The ceiling is very high, but the floor is very low. It only does what you ask it to do — it won't proactively do things you didn't ask for but should have.

## 3️⃣ main.tsx — A Single File with 5,000 Lines

It crams CLI bootstrapping, argument parsing, permission initialization, REPL startup, print mode, and various subcommands all into one file. Normal engineering practice would split this into five or six modules.

AI writes code like writing an essay — one continuous flow from start to finish. Humans write code like building with LEGO — planning modules first, then assembling. AI doesn't proactively split files, because splitting means maintaining import relationships across files, which is a harder task for AI.

## 4️⃣ Auto-compression Once Burned 250K API Calls Per Day

There's a production lesson in the code comments, dated BQ 2026-03-10:

1,279 sessions experienced 50+ consecutive compression failures, with the worst session hitting **3,272 consecutive failures**. Each failure triggered a retry, wasting approximately 250,000 API calls per day globally.

The fix was three lines of code: stop retrying after 3 consecutive failures.

**An automated process without a retry limit is a ticking time bomb.** This lesson holds true in any distributed system.

## 5️⃣ 82 Feature Flags All Hardcoded to false

```typescript
const feature = (_name: string) => false;
```

One line of code turns off all unreleased features. Bun's bundler performs dead code elimination, stripping all code wrapped in flags from the build output.

**But the code is still there in the source.** You can see every feature Anthropic is developing: Kairos autonomous mode, Context Collapse, Voice Mode for voice interaction, Verification Agent for automated verification. The technical roadmap of a top-tier Agent company, written in plain text in the code.

## 6️⃣ Compression Summaries Use an Elegant Prompt Technique

During compression, the model is asked to first analyze in `<analysis>` tags, then write the summary in `<summary>` tags. Only the summary is kept; the analysis is discarded.

Let the model think clearly before summarizing, without wasting context space. The thinking process is scratch paper — used once, then thrown away. This technique is simple but effective, and worth learning for anyone building Agents.

## 7️⃣ 583 Dependencies

A CLI tool with 583 npm dependencies. When AI encounters a problem while writing code, it `npm install`s first — if a third-party library can solve it, it will never hand-write the solution. A human would weigh whether a feature that only takes 20 lines of code really justifies pulling in a library with tens of thousands of lines. AI doesn't make this trade-off.

**Every additional dependency is another supply chain attack surface.**

## 8️⃣ Zero Tests

515,000 lines of code with essentially zero test coverage. The test framework is configured but there are no actual tests. When AI is asked to write features, it writes features — if you don't explicitly ask for tests, it absolutely will not volunteer them.

## 9️⃣ Claude's Ban Mechanism — What the Source Code Reveals

This is a question many users (especially those in China) care about. Why is it easy to get banned using Claude? The source code offers some clues.

**The client reports extensive telemetry data.** The code contains numerous `logEvent` calls that report information including but not limited to:

- Token consumption, latency, and model selection for every API call
- Tool call types, frequency, and success rates
- Permission mode usage patterns
- Compression trigger counts and reasons
- Session duration and interaction patterns
- Error types and frequency

This telemetry data is sent to Anthropic's backend. While model inference happens server-side, client-side behavioral pattern data is also being collected.

**Speculations directly related to banning:**

**IP and geolocation detection.** Claude Code checks the network environment at startup. The code includes health checks for API endpoints. If you access via a proxy, the proxy's stability, IP geolocation, and request latency patterns may all be logged. Frequently switching IPs across different countries is the behavior most likely to trigger risk controls.

**Usage pattern anomaly detection.** Normal users have predictable usage patterns: active during working hours, with thinking pauses between interactions, diverse tool call types. If your usage pattern resembles an automated script — 24/7 non-stop calls, precisely uniform intervals between calls, using only one type of tool — the backend risk control system may flag you.

**Token consumption rate.** The code contains `TOKEN_BUDGET` and `TASK_BUDGETS` feature flags, indicating Anthropic is implementing usage-level budget management. If your token consumption rate significantly exceeds the distribution of normal users, that's another risk control signal.

**Why are users in China more likely to get banned?**

1. **Proxy IP issues.** Users in China must access through proxies. Most proxies use datacenter IPs, where a single IP might have dozens or hundreds of users behind it. When Anthropic's risk control system sees a massive volume of requests from one IP, it flags that IP as suspicious
2. **Frequent IP switching.** Many proxy tools automatically switch nodes — you might be in Japan five minutes ago and the US five minutes later. Normal users don't behave this way, triggering risk controls immediately
3. **Payment info and registration info mismatch.** Registering with a virtual credit card whose billing address is in the US, but actually using it from an IP in Southeast Asia — this inconsistency gets flagged
4. **Shared accounts.** Multiple people sharing one account, with usage spanning 24 hours non-stop and wildly inconsistent usage patterns

**Practical tips to reduce ban risk:**

- Use a fixed residential IP proxy; don't frequently switch nodes
- Maintain geographic consistency between your IP and registration/payment information
- Don't share accounts
- Keep your usage patterns similar to a normal user: work and rest, don't go 24/7 non-stop

## The Future Direction of Agents

After thoroughly analyzing this source code, and combining it with my own experience building an internal Agent platform based on OpenClaw, I have some predictions about where Agents are heading.

**Claude Code is evolving toward a platform, and its evolutionary path overlaps heavily with the design philosophy of Agent platforms like OpenClaw.**

The most obvious signal: cross-session memory. The MEMORY.md mechanism in the source code has already implemented a basic version of cross-session memory persistence. The EXTRACT_MEMORIES and TEAMMEM feature flags indicate Anthropic is working on two things — automatic memory extraction and team memory sharing. This aligns perfectly with OpenClaw's design philosophy: an Agent shouldn't start as a blank slate every time it launches. It needs to remember who the user is, what they prefer, and where things left off last time. OpenClaw achieved instance-level state persistence through COS storage early on. Claude Code is now heading in the same direction, just via a different path — one uses the file system, the other uses object storage.

The second signal is native channel integration. On March 20, 2025, Claude gained the ability to connect to Telegram through MCP plugins. This means the Agent is no longer confined to a terminal waiting for users to come to it — it can proactively exist within the communication tools users already use daily. OpenClaw was designed around channels from day one — WeChat, WeCom, Slack, Discord. Where the Agent lives depends on where the user is. Claude Code is now beginning to understand this. It's foreseeable that Claude Code will natively support more channels in the future, rather than just bridging through MCP. WhatsApp, Line, Lark — these are all just a matter of time.

The third signal is the continuous lowering of the usage barrier. Looking at Claude Code's product iteration trajectory: from manually configuring API keys to one-click login, from pure CLI to VS Code integration, from manually writing CLAUDE.md to automatic memory extraction. Every step lowers the bar. The ultimate goal is clear: **let people who don't understand technology get things done with an Agent.** OpenClaw's Skills marketplace is the ultimate expression of this philosophy — users don't need to understand prompt engineering; they just pick a ready-made Skill and use it. Claude Code's Skills system is already doing the same thing.

But lowering the barrier doesn't mean sacrificing the ceiling. This is the fourth prediction: **for advanced users, Agent platforms will provide increasingly more professional modes.**

The 82 feature flags in Claude Code's source code are evidence. Default configuration is enough for ordinary users, but capable users can unlock entirely different experiences through flag combinations: custom compression strategies, permission mode adjustments, MCP toolchain configuration, Hooks for workflow automation. This layered design — simple to start, deep to master — is a common trait of all successful developer tools. Git is like this, Vim is like this, Kubernetes is like this too.

OpenClaw's Planner-Executor architecture and Claude Code's ReAct loop are essentially solving the same problem: how to keep an Agent controllable during complex tasks. OpenClaw uses DAGs for task orchestration; Claude Code uses AsyncGenerator for tool loops. Different implementations, but the design philosophy is the same — **give the Agent autonomy, but always preserve human intervention points.**

My prediction is: within the next 12 months, Claude Code and platforms like OpenClaw will rapidly converge in functionality. Claude Code will become more platform-like (multi-channel, persistent memory, team collaboration), and OpenClaw will become more intelligent (stronger models, better reasoning capabilities). The ultimate competition won't be about who has the stronger model or more features, but about who can find the best balance between **lowering the usage barrier** and **raising the capability ceiling**.

## Final Thoughts

These findings gave me a more concrete understanding of AI Coding.

AI can write 510,000 lines of code with zero compilation errors but forget to exclude sourcemaps. It can build an elegant three-layer compression system but not write a single test. It can design a sophisticated feature flag mechanism but stuff 5,000 lines into one file without seeing a problem.

**This precisely defines the value of humans in the AI Coding era: security judgment, engineering discipline, and holistic perspective.** These are things AI does poorly, and there's no sign of improvement in the near term.

Those who think AI will replace engineers should feel reassured after seeing this source code. AI-written code is indeed impressive, but between that and being able to ship to production independently — there's still a gap that requires someone who understands engineering.
