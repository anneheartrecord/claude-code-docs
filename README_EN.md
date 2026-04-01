[中文](./README.md)

# Claude Code Source Anatomy

Deep technical analysis based on the leaked Anthropic Claude Code source. 515K lines of TypeScript, 2,766 files, every core module dissected.

## What Is This

In March 2026, Claude Code's complete source was leaked via sourcemaps. This is one of the most mature AI Agent products on the market, and the first commercial-grade Agent system to be fully reverse-engineered.

This repository provides a systematic technical analysis, breaking down the architecture, context engineering, message compaction, permission system, memory management, and more. It also uncovers the future feature roadmap hidden behind 82 feature flags.

**Target Audience:** Engineers building Agent products, developers interested in AI Agent architecture, anyone wanting to understand how a top-tier Agent system is engineered for production.

## Documentation

### Overview

| Document | Content |
|----------|---------|
| [01-Architecture Overview](./docs/01-Architecture-Overview.md) | Overall architecture, tech stack, core files, data flow |
| [02-Value Debate](./docs/02-Value-Debate.md) | Artifact vs Harness capability, two perspectives analyzed |

### Core Modules

| Document | Content |
|----------|---------|
| [03-Agent Loop](./docs/03-Agent-Loop.md) | Six-phase ReAct loop, AsyncGenerator design, state machine |
| [04-Context Engineering](./docs/04-Context-Engineering.md) | System prompt construction, CLAUDE.md loading, layered priorities, prefetch caching |
| [05-Compaction System](./docs/05-Compaction-System.md) | Three-tier compaction: microcompact, session memory, full compact |
| [06-Permission System](./docs/06-Permission-System.md) | Three-mode permissions, YOLO classifier, dangerous patterns, 42 interception rules |
| [07-Memory System](./docs/07-Memory-System.md) | Five-layer memory loading, @include directives, MEMORY.md management |
| [08-Tools & Skills](./docs/08-Tools-and-Skills.md) | 40+ tool registry, execution pipeline, skill fork mechanism |
| [09-MCP Integration](./docs/09-MCP-Integration.md) | Multi-transport protocol, OAuth, seven config scopes |

### Forward-Looking

| Document | Content |
|----------|---------|
| [10-Future Features](./docs/10-Future-Features.md) | 82 feature flags decoded, Kairos autonomous mode, Context Collapse, voice mode |
| [11-AI Code Review](./docs/11-AI-Code-Review.md) | Individual/team/CICD three-layer review paradigm, Review Agent concept |

## Quick Start

```bash
git clone https://github.com/anthropics/claude-code.git
cd claude-code
bun install
bun run build
bun run dev --version
# 2.1.888 (Claude Code)
```

## Key Stats

| Metric | Data |
|--------|------|
| Codebase | 515,498 lines TypeScript/TSX |
| Files | 2,766 |
| Build Output | 25.89 MB, 5,344 modules |
| Tools | 40+ |
| Feature Flags | 82 |
| Permission Rules | 42 dangerous patterns |
| Compaction Threshold | Context window - 13K tokens |
| Dependencies | 583 packages |

## License

This repository contains technical analysis documentation only, not Claude Code source code itself. Analysis is based on publicly accessible sourcemap-reconstructed code.

MIT
