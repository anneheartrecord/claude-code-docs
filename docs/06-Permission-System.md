[中文](/docs/06-权限系统.md)

# 06 Permission System

## Overview

The permission system is the single largest module in Claude Code by code volume, with 6,300+ lines across 25 files. The core problem it solves is: **how to let an Agent autonomously execute tasks while preventing it from performing dangerous operations.**

## Three Permission Modes

| Mode | How It Works | Use Case |
|------|-------------|----------|
| **default** | Pops up a confirmation dialog for every tool call | Safest, suitable for sensitive operations |
| **auto** | ML classifier pre-assesses risk; only high-risk actions trigger a dialog | Daily use, requires Opus 4.6+ |
| **plan** | Model first plans actions with confidence scores; only low-confidence actions trigger a dialog | Complex tasks |

### Default Mode

The simplest and safest. Every time the Agent wants to execute a tool:
1. Display tool name, parameters, and affected file paths
2. User chooses: Allow / Deny / Allow-All-For-This-Tool
3. After choosing Allow-All, that tool no longer prompts

### Auto Mode

Introduces an ML classifier for automated safety assessment:

```
Tool call request
  ↓
Hardcoded dangerous pattern check → Match → Auto-reject
  ↓ No match
YOLO classifier pre-assessment
  ↓
High confidence safe → Auto-allow
Low confidence / Medium risk → Prompt user for confirmation
```

The YOLO classifier (yoloClassifier.ts) makes a side-query call to Claude, passing in the bash command/pattern/environment info, and returns the match result and confidence level.

### Plan Mode

The model first outputs an execution plan with confidence scores for each step:
- High-confidence actions are executed automatically
- Low-confidence actions pause for manual confirmation
- Used in conjunction with the `/plan` command

## 42 Dangerous Patterns

The code hardcodes 42 auto-reject bash patterns in three categories:

**Cross-platform code execution (16 patterns):**
```
python, node, deno, tsx, ruby, perl, php, lua,
npx, bunx, npm run, yarn run, pnpm run, bun run,
bash, sh
```

**Unix-specific (5 patterns):**
```
zsh, fish, eval, exec, env/xargs/sudo
```

**Internal-only (8 patterns, ANT users only):**
```
fa run, coo, gh/gh api, curl, wget,
git, kubectl, aws/gcloud/gsutil
```

Core logic: **Any command capable of launching an interpreter to execute arbitrary code is intercepted.** Because once inside an interpreter, the Agent can execute any operation through it, bypassing all other permission checks.

## Permission Rule Configuration

Users can customize rules in `~/.claude/settings.json`:

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

Rules support wildcard matching. Allow rules auto-permit, deny rules auto-reject, and anything unmatched goes through the default permission mode.

## Permission Mode State Machine

```
default ──→ bypass (user explicitly enables dangerouslySkipPermissions)
bypass  ──→ default (user disables or after N operations)
default ──→ auto (classifier enabled + user consent)
```

When entering auto mode, a **Dangerous Rule Stripping** pass is performed: allow rules involving dangerous patterns that the user previously configured are automatically removed. This prevents a situation where a user allowed `python*` in default mode, and after switching to auto mode, the classifier still auto-permits based on that rule.

## File System Sandbox

`pathValidation.ts` performs file path validation:
- File operations are restricted to the working directory and its subdirectories
- `..` path traversal is prohibited
- Writing to sensitive directories like `.git` and `.claude` is prohibited
- Additional working directories can be added via configuration

## Production Lessons

**Denial Tracking (denialTracking.ts):** Records the number of times users deny tool calls. If the same type of operation is repeatedly denied, the system adjusts subsequent permission policies.

**The permission system is the core of the Harness.** No matter how powerful the model is, without a reliable permission system to constrain its behavioral boundaries, it's just a dangerous toy. Claude Code invested the most engineering resources in the entire codebase into permissions — a prioritization worth referencing for all teams building Agents.
