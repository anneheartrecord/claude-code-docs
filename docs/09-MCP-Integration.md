[中文](09-MCP集成.md)

# 09 MCP Integration

## 1️⃣ What Is MCP, and Why Does It Matter

To understand MCP, let's first revisit an industry pain point.

Before 2024, every AI Agent integrated with external services **in its own bespoke way**. Want your Agent to call Jira? Write a Jira adapter. Call Slack? Write another adapter. Connect to a database? Yet another one. Every Agent framework was reinventing the wheel, and every external service had to provide different integration paths for different Agents. It was like the early days of the internet, when every website had to build different pages for different browsers.

**Model Context Protocol, or MCP, is a standardized tool-calling protocol proposed by Anthropic in late 2024.** Its core idea is simple: define a universal interface specification so that any AI Agent can call any external service that implements the MCP protocol. As long as the server exposes functionality per the MCP standard and the client calls it per the MCP standard, the two sides work seamlessly together.

This follows the same logic as USB. Before USB, every peripheral had its own connector standard. USB unified the interface, making any device plug-and-play. MCP aims to be the USB of the AI Agent ecosystem.

Claude Code's MCP implementation runs deep. The entire MCP subsystem lives in `src/services/mcp/`, comprising **24 files and 12,000+ lines of code**, making it one of the largest subsystems in Claude Code. This scale shows that Anthropic treats MCP as Claude Code's most critical extension mechanism, investing substantial engineering resources.

## 2️⃣ Seven Configuration Scopes: From Individual to Enterprise

MCP Server configurations support seven different scope levels. This design serves use cases ranging from individual developers to large enterprises.

| Scope | Configuration Location | Typical Use Case | Priority |
|-------|----------------------|------------------|----------|
| **local** | Config file in the project directory | Tools used only in the current project, like a project-specific database client | Highest |
| **user** | Config file under ~/.claude/ | Personal global tools, like your own knowledge base MCP | High |
| **project** | .claude/ directory, checked into git | Team-shared project tools, like a team's API testing tool | Medium |
| **dynamic** | Injected via code at runtime | Tools dynamically registered by plugins or IDEs | Medium |
| **enterprise** | Admin-unified configuration | Enterprise-mandated security audit tools | Medium-low |
| **claudeai** | Anthropic official configuration | MCP services provided by Anthropic | Low |
| **managed** | Remote management platform | Centrally governed enterprise toolsets | Lowest |

**Why so many levels?** Consider a real scenario: you're a developer at a company. The IT department uses the enterprise scope to mandate security scanning MCP tools. Your team shares CI/CD-related MCP tools via the project scope. You personally configure your favorite note-taking tool through the user scope. The current project has a project-specific database management tool in the local scope. All of these are **active simultaneously** in Claude Code, with higher-priority configs overriding same-named lower-priority ones.

This multi-layered configuration design is common in traditional software — Git has system/global/local configs, NPM has multi-level .npmrc files. But MCP extends to seven layers, reflecting that the AI Agent tool ecosystem is far more complex than traditional development tooling.

## 3️⃣ Six Transport Protocols: Adapting to Every Runtime Environment

MCP defines the standard interface, but how data actually travels between Agent and Server requires concrete transport protocols. Claude Code supports six:

| Protocol | Technical Mechanism | Best For |
|----------|-------------------|----------|
| **stdio** | Communicates via standard input/output. Agent launches the Server process and exchanges JSON messages through stdin/stdout | Local tools — simplest and most reliable |
| **sse** | Server-Sent Events, HTTP-based unidirectional streaming. Agent sends HTTP requests, Server pushes results via SSE stream | Remote tools with real-time push support |
| **sse-ide** | IDE-specific SSE variant, with connections managed by VS Code or similar IDEs | IDE integration environments |
| **http** | Standard HTTP REST request-response | Remote API services |
| **ws** | WebSocket bidirectional communication | Scenarios requiring two-way real-time communication |
| **sdk** | In-process SDK call, no network involved, direct function call within the same process | Built-in tools with extreme performance requirements |

**stdio is the most commonly used protocol** because most MCP Servers are small local programs. The Agent starts them and communicates through pipes — simple and direct. But as the MCP ecosystem grows, more and more Servers are remote services, making HTTP, SSE, and WebSocket increasingly important.

**Each protocol requires independent connection management, retry logic, and error handling code.** This is one of the main reasons the MCP module is so large. A broken stdio connection might mean the process crashed and needs restarting. A failed HTTP connection might be a network issue requiring exponential backoff retries. A dropped WebSocket connection might need re-handshake authentication. Each protocol's failure modes are entirely different and require specialized handling.

## 4️⃣ OAuth Authentication: Connecting MCP to Enterprise Services

Many valuable external services require authentication. MCP includes full OAuth 2.0 support, allowing the Agent to securely call protected APIs.

Configuration example:

```json
{
  "mcpServers": {
    "my-server": {
      "transport": "http",
      "url": "https://api.example.com/mcp",
      "oauth": {
        "clientId": "your-client-id",
        "callbackPort": 8080,
        "authServerMetadataUrl": "https://auth.example.com/.well-known"
      }
    }
  }
}
```

How the OAuth flow works within MCP:

1. Agent attempts to call the MCP Server
2. Server returns 401 Unauthorized
3. Agent automatically initiates the OAuth authorization flow, opening a browser for user consent
4. After user authorization, the Agent obtains an access token
5. Subsequent calls automatically include the token
6. Token is automatically refreshed upon expiry

**Claude Code also supports XAA, or Cross-App Access.** This is a more advanced authentication mechanism that allows multiple MCP Servers to share authentication from a single identity provider. For example, if your company has a unified SSO system, all internal MCP Servers can use the same authentication through XAA. Users only need to log in once to access all tools.

## 5️⃣ Permission Management: MCP Tools Under Security Governance

Once MCP tools are connected, they cannot be called freely. They are subject to the same comprehensive permission system as Claude Code's built-in tools.

`channelPermissions.ts` provides **independent permission control** for each tool on each MCP Server. This means:

- You can auto-approve a certain MCP Server's read tools while requiring manual confirmation for write tools
- You can completely block specific tools from a certain MCP Server
- Enterprise administrators can set unified MCP tool permission policies that developers cannot bypass

This design ensures a critical security property: **a third-party MCP Server does not automatically gain full access to your system just because the Agent connects to it.** Even if an MCP Server's code has security vulnerabilities, the permission system limits its blast radius.

## 6️⃣ Why the MCP Module Has 12,000 Lines of Code

12,000 lines for a protocol integration is substantial by any standard. Here's where the complexity comes from:

**First, full implementation of six transport protocols.** Each protocol has its own connection establishment, message serialization, error recovery, and timeout handling code. The transport layer alone likely accounts for 3,000-4,000 lines.

**Second, complete OAuth flow implementation.** OAuth 2.0 is inherently complex, involving authorization code flows, token refresh, PKCE security extensions, and error handling. Add XAA cross-application authentication support, and the auth module likely accounts for 2,000-3,000 lines.

**Third, merge logic for seven configuration scopes.** Different levels need merging, overriding, and conflict resolution. Each scope has different loading mechanisms and storage locations. Configuration management likely accounts for 1,500-2,000 lines.

**Fourth, extensive defensive code.** MCP Servers are third-party code. They might return malformed data, time out, crash, or return oversized responses. Claude Code must handle all these edge cases to ensure a buggy MCP Server doesn't bring down the entire Agent.

**Fifth, tool discovery and dynamic registration.** MCP Servers can dynamically register and unregister tools at runtime. The Agent needs to detect these changes in real time and update its tool list accordingly.

It's worth noting that this code volume inflation is partly a **characteristic of AI-assisted coding**. When facing a complex protocol, AI tends toward blanket coverage implementations, handling each possible exception individually. An engineer with deep MCP protocol understanding might use more refined abstractions to reduce code volume. On the other hand, this "coverage-first" approach also means better robustness and fewer unhandled edge cases.

## 7️⃣ The MCP Ecosystem: Present and Future

As of early 2026, the MCP ecosystem has reached meaningful scale. Major MCP Servers include:

- **GitHub MCP Server**: Directly manipulate GitHub repos, PRs, Issues
- **Slack MCP Server**: Read and write Slack messages and channels
- **PostgreSQL MCP Server**: Query and manage databases
- **Playwright MCP Server**: Browser automation testing
- **Filesystem MCP Server**: Controlled file system access

MCP's industry influence is expanding. Beyond Claude Code, VS Code's GitHub Copilot has also started supporting MCP, and AI coding tools like Cursor and Windsurf are following suit. **MCP is becoming the de facto standard for AI Agent tool invocation.**

But MCP also faces challenges. The protocol itself is still evolving rapidly, and version compatibility is a concern. The existence of six transport protocols suggests the community hasn't reached consensus on the optimal transport mechanism. Additionally, MCP Server quality varies widely, and there's no unified security audit standard.

For developers, investing time in learning and using MCP now is worthwhile. **It represents the direction of AI tool ecosystems: standardized, composable, and securely controllable.**

Next: [10-Future Features](10-未来功能蓝图.md)
