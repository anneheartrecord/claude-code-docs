[中文](./11-AI-Coding时代的Code-Review.md)

# 11 Code Review in the AI Coding Era

## Starting from the Sourcemap Leak

The Claude Code source leak is itself a perfect case study. A 515,000-line project written by AI forgot to exclude sourcemap files during the build, exposing the complete source code.

AI wrote the features flawlessly with zero type errors, but left a fatal vulnerability in a release engineering detail like build security.

**This is precisely why the focus of Code Review needs to fundamentally shift in the AI Coding era.**

## The Review Paradigm Across Three Levels

### 1️⃣ Individual Level: From Granular CR to Reviewing Only Critical Paths

Last year, I was still doing fairly detailed CR on every line of AI-generated code, ensuring readability, naming conventions, and design patterns. But as model capabilities have improved, and since the whole point of using AI is speed, spending too much time on CR defeats most of the tool's value.

**This year, I've essentially stopped doing traditional CR.** I might glance at the correctness of core interface logic, but everything else gets a pass.

The underlying logic of this shift: AI is already good enough at the "form" aspects — code style, naming conventions, design patterns — that humans don't need to nitpick them. Human attention should focus on where AI falls short:

- **Business logic correctness:** AI doesn't understand your business — it just writes code based on the prompt. Only you know whether the logic is correct
- **Security:** AI doesn't proactively consider injection, privilege escalation, or information leakage. You have to watch for these yourself
- **Architectural decisions:** Whether to split this module, whether to introduce this dependency, whether to expose this API. AI provides locally optimal solutions — global judgment depends on humans

Most engineers should adapt to this new CR paradigm. **The purpose of CR has shifted from "finding code problems" to "verifying whether AI's decisions match expectations."**

### 2️⃣ Team Level: Shared System Prompt Baseline

A development team should maintain a shared baseline system prompt that precisely defines the project's technical standards:

- Project tech stack and architecture conventions
- Code style and naming conventions
- Commit conventions and branching strategy
- Testing requirements
- Security red lines

This system prompt is the CLAUDE.md or equivalent configuration file. Its purpose is to ensure that AI Coding output from every team member doesn't deviate too far.

**Without a shared prompt, the code style, architecture style, and testing style from each person's AI will all be different.** Merging them together creates a chaotic mess. With a shared prompt, at least a baseline is guaranteed.

This is the same principle as a traditional team's coding standard document — except the audience has changed from humans to AI.

### 3️⃣ CICD Infrastructure Level: A New Paradigm for the AI Era

This is the layer that most needs to be redefined.

**Traditional CICD pass criteria:** Compiles, runs, tests pass.

**Standards that CICD should add for the AI era:**

**Security scanning.** Scan AI-generated code for hardcoded API keys, internal URLs, and debug configurations. The Claude Code sourcemap leak happened precisely because the release pipeline lacked this step.

**Privacy checks.** AI sometimes leaks prompt information into code comments or logs. CI should scan for sensitive information appearing where it shouldn't.

**Sourcemap / debug file detection.** Check whether build artifacts contain .map files, debug symbol tables, or internal documents that shouldn't be there.

**Dependency security audits.** AI loves pulling in third-party libraries to solve problems — every new dependency is a supply chain attack surface. Running npm audit / snyk in CI is essential.

**Code Review Agent.** This is the most interesting direction. Introduce a Review Agent that performs a comprehensive review on every commit.

Specifically: commit the requirement prompt alongside the code implementation. The Review Agent reads the prompt and checks whether the code implementation matches expectations. Using AI to review AI.

```
Commit contents:
├── prompt.md          # Original requirement prompt
├── src/feature.ts     # AI-generated code
└── tests/feature.test.ts  # AI-generated tests

Review Agent checklist:
1. Does the code implementation match the requirements described in prompt.md?
2. Are there any security vulnerabilities? (injection, privilege escalation, information leakage)
3. Is there any hardcoded sensitive information?
4. What scenarios do the tests cover? What edge cases are missing?
5. Are there any unnecessary dependencies introduced?
```

**A new CICD paradigm for the AI era should be established as part of Harness Engineering — perhaps even the most important part.** Because when the producer of code shifts from humans to AI, traditional manual review is no longer sufficient. You need infrastructure to guarantee quality.

## An Actionable AI Code Review Workflow

```
1. AI generates code (with original prompt attached)
     ↓
2. Automated checks (CI pipeline)
   - lint / type check / build
   - Sensitive information scanning
   - Sourcemap / debug file detection
   - Dependency security audit
     ↓
3. Review Agent (AI reviews AI)
   - Requirement conformance check
   - Security audit
   - Edge case coverage
     ↓
4. Human review (critical paths only)
   - Business logic correctness
   - Architectural decision soundness
   - Security attack surface
     ↓
5. Merge and release
```

## Review Lessons Learned from Claude Code's Source

| AI Code Characteristic | What to Focus On |
|----------------------|-----------------|
| Every async has a try/catch | Does the catch just log and swallow the error? |
| Extremely thorough type annotations | Are there `as any` casts bypassing type checks? |
| All branches are handled | Are else branches just placeholders? |
| Error messages are comprehensive | Do they leak internal implementation details? |
| Large codebase with neat structure | Is there massive duplicated code that could be abstracted? |
| Many dependencies | Are there unnecessary libraries introduced? |
| Build passes | Are there files in the build artifacts that shouldn't be there? |

**Core principle: AI's efficiency advantage in writing code lies in generation speed. The irreplaceable value of human involvement lies in security judgment and engineering details. CICD infrastructure is the Harness that ties both together.**
