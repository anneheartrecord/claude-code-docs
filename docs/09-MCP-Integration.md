[中文](./09-MCP集成.md)

# 09 MCP Integration

## What is MCP

Model Context Protocol is a standardized tool-calling protocol proposed by Anthropic. Instead of writing custom integration logic for each external service, an Agent can directly invoke any service that implements the MCP protocol.

Claude Code's MCP implementation is located in `src/services/mcp/`, comprising 24 files and 12,000+ lines of code, making it one of the largest subsystems by code volume.

## Seven Configuration Scopes

MCP Servers can be configured at different levels:

| Scope | Location | Use Case |
|-------|----------|----------|
| local | Project directory | Project-specific tools |
| user | ~/.claude/ | Personal global tools |
| project | .claude/ | Shared across project, checked into git |
| dynamic | Injected at runtime | Dynamically registered by plugins |
| enterprise | Admin-configured | Enterprise-wide unified tools |
| claudeai | Anthropic official | Services provided by Anthropic |
| managed | Remotely managed | Centralized management |

## Six Transport Protocols

| Protocol | Description |
|----------|-------------|
| stdio | Standard input/output, most commonly used |
| sse | Server-Sent Events |
| sse-ide | IDE-specific SSE |
| http | HTTP REST |
| ws | WebSocket |
| sdk | In-process SDK calls |

## OAuth Support

MCP Servers can be configured with OAuth authentication:

```json
{
  "mcpServers": {
    "my-server": {
      "transport": "http",
      "url": "https://api.example.com/mcp",
      "oauth": {
        "clientId": "xxx",
        "callbackPort": 8080,
        "authServerMetadataUrl": "https://auth.example.com/.well-known"
      }
    }
  }
}
```

It also supports XAA (Cross-App Access), allowing multiple MCP Servers to share authentication from a single IdP.

## Permission Management

MCP tools are governed by the same permission system as built-in tools. `channelPermissions.ts` provides independent permission control for each tool on each MCP Server.

## Why Is the MCP Module So Large

12,000 lines for MCP integration — the complexity is clearly excessive. Analysis of the reasons:

1. Supporting 6 transport protocols, each with its own connection management, retry logic, and error handling
2. The OAuth flow itself is inherently complex
3. Merging and priority handling across 7 configuration scopes
4. A large volume of defensive code handling various edge cases

This is a typical characteristic of AI-generated code: when facing a complex protocol that isn't fully understood, the approach is to write massive amounts of code for blanket coverage. A human developer would first thoroughly understand the protocol and then implement it in a more concise way.
