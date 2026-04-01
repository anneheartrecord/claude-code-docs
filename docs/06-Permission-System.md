[中文](06-权限系统.md)

# 06 Permission System

## The Security Dilemma of AI Agents

Since 2024, the AI Agent space has seen several alarming security incidents. Researchers have demonstrated how carefully crafted prompt injections can trick Agents into executing dangerous commands that users never requested. Malicious comments have been planted in public code repositories so that when an Agent reads those files, it gets manipulated into running `rm -rf /` or uploading sensitive files to external servers. These incidents reveal a fundamental problem: **when you give AI the ability to perform real actions, you simultaneously give it the ability to cause real damage**.

This is exactly the problem that the permission system solves. It needs to balance two contradictory goals: on one hand, making the Agent autonomous enough to efficiently complete tasks; on the other hand, constraining its behavioral boundaries to prevent irreversible dangerous operations.

Claude Code's permission system is the **single largest module** in the entire codebase, with over 6,300 lines of code spread across 25 files. This level of investment alone speaks to how seriously Anthropic takes Agent security. By comparison, many open-source Agent frameworks have permission mechanisms that amount to a simple whitelist/blacklist, or no permission controls at all.

## 1️⃣ Three Permission Modes

Claude Code offers three permission modes to accommodate different usage scenarios and risk appetites:

| Mode | How It Works | Best For |
|------|-------------|----------|
| **default** | Prompts user confirmation for every tool call | Maximum safety, ideal when working with sensitive code or production environments |
| **auto** | ML classifier pre-assesses risk; only high-risk actions prompt the user | Day-to-day development, requires Opus 4.6 or higher |
| **plan** | Model plans actions first with confidence scores; only low-confidence actions prompt | Complex multi-step tasks |

### Default Mode: Full Human Confirmation

This is the most conservative mode. Every time the Agent wants to execute a tool call, the flow is:

1. Display the tool name, parameters, and affected file paths to the user
2. User chooses: **Allow** to proceed / **Deny** to reject / **Allow-All-For-This-Tool** to permit all future calls of this tool
3. After choosing Allow-All, that tool won't prompt again for the rest of the session

The strength of Default mode is maximum safety since all operations go through human review. The weakness is low efficiency. A complex task might involve dozens of tool calls, and confirming each one severely disrupts workflow. This is why Claude Code provides two additional modes.

### Auto Mode: ML Classifier-Assisted Judgment

Auto mode introduces an **ML classifier** to automate safety judgments, dramatically reducing the number of prompts requiring human confirmation:

```
Tool call request
  ↓
Hardcoded dangerous pattern check → Match → Auto-reject
  ↓ No match
YOLO classifier pre-assessment
  ↓
High confidence safe → Auto-allow
Low confidence or medium risk → Prompt user for confirmation
```

The **YOLO classifier** has an interesting name. YOLO in internet culture stands for "You Only Live Once," implying "just go for it." In Claude Code's context, it refers to a classifier that makes a side-query call to the Claude model to evaluate command safety. Its implementation lives in `yoloClassifier.ts`, and it works by sending the bash command about to be executed, the command pattern, and current environment information to Claude, which then judges whether the command is safe and returns a match result with a confidence score.

Auto mode requires Opus 4.6 or higher because the classifier's accuracy directly depends on the model's judgment capability. Running the classifier with a weaker model would lead to higher error rates: either letting dangerous commands through, or generating frequent false positives that interrupt users.

### Plan Mode: Planning Before Execution

Plan mode's approach is to have the model plan before executing:

1. The model outputs an execution plan listing every step it intends to take
2. Each step carries a **confidence score** indicating how certain the model is about that operation's safety
3. High-confidence steps execute automatically
4. Low-confidence steps pause for user confirmation

This mode is especially well-suited for complex multi-step tasks. Users can see the complete plan first and intervene on questionable steps, without needing to make decisions at every single step. It works with the `/plan` command, allowing users to proactively trigger planning mode.

## 2️⃣ 42 Dangerous Patterns: The Hardcoded Safety Floor

Regardless of which permission mode is active, the code hardcodes **42 auto-reject bash patterns**. This is an unbypassable safety floor. Even if a user has configured allow rules, these patterns will still be intercepted.

The 42 patterns fall into three categories:

**Cross-platform code execution, 16 patterns**

```
python, node, deno, tsx, ruby, perl, php, lua,
npx, bunx, npm run, yarn run, pnpm run, bun run,
bash, sh
```

**Unix-specific, 5 patterns**

```
zsh, fish, eval, exec, env/xargs/sudo
```

**Internal-only, 8 patterns, only for Anthropic internal users**

```
fa run, coo, gh/gh api, curl, wget,
git, kubectl, aws/gcloud/gsutil
```

Why are these commands intercepted? The core logic is: **any command capable of launching an interpreter or executing arbitrary code must be intercepted**. The reason is straightforward: once the Agent enters a Python interpreter or Node.js runtime, it can execute any operation inside that interpreter, completely bypassing all other permission checks in Claude Code.

For example, suppose the permission system forbids directly executing `rm -rf /`. If the Agent can freely run `python -c "import os; os.system('rm -rf /')"`, then the restriction on `rm` is meaningless. Intercepting interpreter entry points closes this bypass path.

This is consistent with the **principle of least privilege** in operating system security design. Linux's SELinux and AppArmor achieve security isolation by restricting what system calls a process can make. Claude Code's dangerous pattern checking is analogous to a syscall filter at the Agent level.

## 3️⃣ Permission Rule Configuration

Users can customize permission rules in `~/.claude/settings.json` for more granular control:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run build)",
      "Bash(git log *)",
      "Read",
      "Edit"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ]
  }
}
```

Rules support **wildcard matching**. `Bash(git log *)` matches all commands starting with `git log`. The processing order is:

1. Check deny rules; match triggers auto-reject
2. Check allow rules; match triggers auto-allow
3. No match on either falls through to the current permission mode's default flow

This design lets users customize based on their work context. For instance, a frontend developer might allow `npm run build` and `npm run test` because these commands are safe in their project. But the same commands might behave completely differently in another project, making this configuration inherently project-specific.

## 4️⃣ Permission Mode State Machine

Switching between the three modes follows strict state machine rules:

```
default ──→ bypass, when user explicitly enables dangerouslySkipPermissions
bypass  ──→ default, when user disables or after N operations, auto-reverts
default ──→ auto, when classifier is enabled and user gives explicit consent
```

There's a critical safety mechanism here: **Dangerous Rule Stripping**.

When a user switches from default mode to auto mode, the system automatically inspects the user's previously configured allow rules and **proactively removes** any that involve dangerous patterns.

Why is this needed? Consider this scenario: a user adds a `Bash(python *)` allow rule while in default mode. Since every call still gets a confirmation prompt in default mode, this rule just makes confirmation one step faster. But if the user switches to auto mode, the classifier might auto-allow all python commands based on that rule. This would bypass the hardcoded protection of the 42 dangerous patterns.

Dangerous Rule Stripping prevents this vulnerability by proactively cleaning up potentially dangerous allow rules during mode transitions. This is a textbook case of **defensive programming**: never assuming that user configuration is safe in all scenarios, instead performing additional safety checks at critical transition points.

## 5️⃣ File System Sandbox

Beyond command-level permission controls, Claude Code also implements isolation at the file system level. `pathValidation.ts` implements path validation logic:

- **Working directory restriction**: All file operations are confined to the current working directory and its subdirectories. The Agent cannot read or write files outside the working directory
- **Path traversal prevention**: Using `..` for path traversal is prohibited. Even if the Agent constructs a path like `../../etc/passwd`, the validation will catch and block it
- **Sensitive directory protection**: Writes to `.git`, `.claude`, and similar directories are intercepted. These directories contain version control information and Agent configuration; tampering with them could have serious consequences
- **Additional working directories**: Extra permitted directories can be added via configuration, accommodating scenarios like monorepos that require cross-directory access

This sandbox design draws from the **namespace isolation** concept in container technology. Docker uses mount namespaces to limit the filesystem scope a container can see. Claude Code uses path validation to limit the file scope an Agent can operate on. The underlying thinking is identical.

## 6️⃣ Behavioral Learning: Denial Tracking

`denialTracking.ts` implements a noteworthy feature: **user denial behavior tracking**.

The system records the count and patterns of user tool call denials. If the same type of operation is repeatedly denied, the system adjusts subsequent permission policies, trending more conservative for that category.

This design reflects an important product philosophy: permission systems should be **adaptive** over time. If a user consistently denies a certain operation, it indicates they consider that operation inappropriate in the current context. The system should learn this preference rather than repeatedly asking the same question.

This kind of behavioral learning has precedents in traditional security systems. Adaptive firewall rules, heuristic antivirus detection, and mobile app permission recommendations are all examples of adjusting security policies based on user behavior patterns.

## 7️⃣ Comparison with Industry Approaches

| Project | Permission Mechanism | Security Depth | Adaptability |
|---------|---------------------|---------------|-------------|
| **Claude Code** | 3 modes + 42 hardcoded rules + ML classifier + file sandbox + behavioral learning | Multi-layered defense in depth | High: denial tracking + rule stripping |
| **Cursor** | User confirmation prompts | Single layer | None |
| **Aider** | Whitelisted commands | Single layer | None |
| **OpenAI Codex CLI** | Sandbox mode + confirmation prompts | Two layers | None |
| **Devin** | Restricted environment + human review | Two layers | Limited |

Claude Code's permission system surpasses competitors in both depth and breadth. This investment makes sense: an Agent that can autonomously execute shell commands and read/write arbitrary files carries far greater potential risk than a tool that only makes code suggestions. **The permission system is the core of the Agent Harness.** No matter how capable the model is, without a reliable permission system constraining its behavioral boundaries, it's just a dangerous toy. Claude Code invested the most engineering resources in the entire codebase into permissions. This prioritization is worth studying for every team building Agents.

---

Next: [07-Memory-System](07-Memory-System.md)
