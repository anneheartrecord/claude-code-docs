[English](docs/09-MCP-Integration.md)

# 09 MCP 集成

## 1️⃣ MCP 是什么，为什么它很重要

![MCP协议USB类比](../imgs/09-mcp-usb-analogy.png)


要理解 MCP，先回顾一个行业痛点。

2024 年之前，每个 AI Agent 接入外部服务的方式都是**各写各的**。你要让 Agent 调用 Jira？写一套 Jira 适配代码。要调用 Slack？再写一套。要调用数据库？又一套。每个 Agent 框架都在重复造轮子，每个外部服务都要为不同的 Agent 提供不同的对接方式。这就像互联网早期，每个网站都要为不同的浏览器写不同的页面。

**Model Context Protocol，简称 MCP，是 Anthropic 在 2024 年底提出的标准化工具调用协议。** 它的核心理念很简单：定义一套通用的接口规范，让任何 AI Agent 都能调用任何实现了 MCP 协议的外部服务。只要服务端按 MCP 标准暴露功能，客户端按 MCP 标准调用，两边就能无缝配合。

这和 USB 的思路一致。在 USB 之前，每个外设都有自己的接口标准。USB 统一了接口，任何设备都可以即插即用。MCP 要做的就是 AI Agent 生态的 USB。

Claude Code 对 MCP 的实现非常深入。整个 MCP 子系统位于 `src/services/mcp/` 目录，包含 **24 个文件、12,000+ 行代码**，是 Claude Code 代码量最大的子系统之一。这个规模说明 Anthropic 把 MCP 视为 Claude Code 最核心的扩展机制，投入了大量工程资源。

## 2️⃣ 七种配置作用域：从个人到企业的全覆盖

![七层配置作用域](../imgs/09-mcp-seven-scopes.png)


MCP Server 的配置支持七种不同的作用域层级。这个设计是为了满足从个人开发者到大型企业的各种使用场景。

| 作用域 | 配置位置 | 典型使用场景 | 优先级 |
|--------|---------|-------------|--------|
| **local** | 项目目录下的配置文件 | 只在当前项目中使用的工具，比如项目专用的数据库客户端 | 最高 |
| **user** | ~/.claude/ 下的配置文件 | 个人全局工具，比如你自己搭建的知识库 MCP | 高 |
| **project** | .claude/ 目录，会被 checkin 到 git | 团队共享的项目工具，比如团队公共的 API 测试工具 | 中 |
| **dynamic** | 运行时通过代码注入 | 插件或 IDE 动态注册的工具 | 中 |
| **enterprise** | 管理员统一配置 | 企业级强制要求的安全审计工具 | 中低 |
| **claudeai** | Anthropic 官方配置 | 由 Anthropic 官方提供的 MCP 服务 | 低 |
| **managed** | 远程管理平台 | 集中管控的企业工具集 | 最低 |

**为什么需要这么多层级？** 考虑一个真实场景：你是某家公司的开发者。公司 IT 部门通过 enterprise 作用域强制配置了安全扫描 MCP 工具。你的团队通过 project 作用域共享了 CI/CD 相关的 MCP 工具。你个人通过 user 作用域配置了自己喜欢的笔记工具。当前项目通过 local 作用域配置了项目专用的数据库管理工具。所有这些在 Claude Code 中**同时生效**，高优先级的配置覆盖低优先级的同名配置。

这种多层配置的设计在传统软件中很常见，比如 Git 的 system/global/local 三层配置，NPM 的 .npmrc 多层配置。但 MCP 把层级扩展到了七层，这反映了 AI Agent 工具生态的复杂性远超传统开发工具。

## 3️⃣ 六种传输协议：适配各种运行环境

![六种传输协议](../imgs/09-mcp-six-protocols.png)


MCP 定义了标准接口，但数据怎么在 Agent 和 Server 之间传输，需要具体的传输协议。Claude Code 支持六种：

| 协议 | 技术原理 | 适用场景 |
|------|---------|---------|
| **stdio** | 通过标准输入输出通信，Agent 启动 Server 进程，通过 stdin/stdout 交换 JSON 消息 | 本地工具，最简单最可靠 |
| **sse** | Server-Sent Events，基于 HTTP 的单向流式推送，Agent 发 HTTP 请求，Server 通过 SSE 流推送结果 | 远程工具，支持实时推送 |
| **sse-ide** | IDE 专用的 SSE 变体，由 VS Code 等 IDE 托管连接 | IDE 集成环境 |
| **http** | 标准 HTTP REST 请求响应 | 远程 API 服务 |
| **ws** | WebSocket 双向通信 | 需要双向实时通信的场景 |
| **sdk** | 进程内 SDK 调用，不走网络，直接在同一个进程内调用 | 性能要求极高的内置工具 |

**stdio 是最常用的协议**，因为大多数 MCP Server 就是一个本地运行的小程序。Agent 启动它，通过管道通信，简单直接。但随着 MCP 生态的发展，越来越多的 Server 是远程服务，这时候 HTTP、SSE、WebSocket 就变得重要了。

**每种协议都需要独立的连接管理、重试逻辑和错误处理代码。** 这是 MCP 模块代码量大的主要原因之一。一个 stdio 连接断了，可能是进程崩溃了，需要重启。一个 HTTP 连接失败了，可能是网络问题，需要退避重试。一个 WebSocket 连接断了，可能需要重新握手认证。每种协议的异常模式完全不同，都需要专门处理。

## 4️⃣ OAuth 认证：让 MCP 连接企业级服务

很多有价值的外部服务需要认证才能访问。MCP 内置了完整的 OAuth 2.0 支持，让 Agent 可以安全地调用受保护的 API。

配置示例：

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

OAuth 流程在 MCP 中的运作方式：

1. Agent 尝试调用 MCP Server
2. Server 返回 401 未认证
3. Agent 自动启动 OAuth 授权流程，打开浏览器让用户授权
4. 用户授权后，Agent 获取 access token
5. 后续调用自动携带 token
6. Token 过期时自动刷新

**Claude Code 还支持 XAA，即 Cross-App Access。** 这是一个更高级的认证机制，允许多个 MCP Server 共享同一个身份提供商的认证。比如你的公司有一个统一的 SSO 系统，所有内部 MCP Server 都可以通过 XAA 使用同一套认证，用户只需要登录一次就能访问所有工具。

## 5️⃣ 权限管理：MCP 工具也受安全管控

MCP 工具接入后，并非可以随意调用。它们和 Claude Code 的内置工具一样，受到权限系统的完整管控。

`channelPermissions.ts` 对每个 MCP Server 的每个工具做**独立的权限控制**。这意味着：

- 你可以让某个 MCP Server 的读取工具自动放行，但写入工具需要人工确认
- 你可以完全禁止某个 MCP Server 的特定工具
- 企业管理员可以统一设定 MCP 工具的权限策略，开发者无法绕过

这种设计确保了一个关键的安全属性：**第三方 MCP Server 不会因为被 Agent 接入而自动获得对你系统的完全访问权限**。即使一个 MCP Server 的代码有安全漏洞，权限系统也能限制它的影响范围。

## 6️⃣ 为什么 MCP 模块有 12,000 行代码

![12000行代码分布](../imgs/09-mcp-12000-lines.png)


12,000 行代码做一个协议集成，这个规模在任何标准下都不小。分析其复杂度来源：

**第一，六种传输协议的完整实现。** 每种协议都有自己的连接建立、消息序列化、错误恢复、超时处理代码。仅传输层就可能占了 3,000-4,000 行。

**第二，OAuth 流程的完整实现。** OAuth 2.0 本身就是一个复杂的协议，涉及授权码流程、token 刷新、PKCE 安全扩展、错误处理。加上 XAA 跨应用认证的支持，认证模块可能占了 2,000-3,000 行。

**第三，七种配置作用域的合并逻辑。** 不同层级的配置需要合并、覆盖、冲突解决。每种作用域有不同的加载方式和存储位置。配置管理可能占了 1,500-2,000 行。

**第四，大量的防御性代码。** MCP Server 是第三方代码，它可能返回格式错误的数据、超时、崩溃、返回超大响应。Claude Code 需要对所有这些异常情况做处理，确保一个有问题的 MCP Server 不会拖垮整个 Agent。

**第五，工具发现和动态注册。** MCP Server 可以在运行时动态注册和注销工具，Agent 需要实时感知这些变化并更新自己的工具列表。

值得注意的是，这种代码量膨胀在一定程度上是 **AI 辅助编码的典型特征**。面对一个复杂协议，AI 倾向于用大量代码做覆盖式实现，对每种可能的异常都单独处理。如果由一个对 MCP 协议理解很深的工程师来写，可能会用更精炼的抽象来减少代码量。但另一方面，这种"覆盖式"的写法也意味着更好的健壮性和更少的未处理边界情况。

## 7️⃣ MCP 生态的现状与未来

截至 2026 年初，MCP 生态已经初具规模。主要的 MCP Server 包括：

- **GitHub MCP Server**：直接操作 GitHub 仓库、PR、Issues
- **Slack MCP Server**：读写 Slack 消息和频道
- **PostgreSQL MCP Server**：查询和管理数据库
- **Playwright MCP Server**：浏览器自动化测试
- **文件系统 MCP Server**：受控的文件系统访问

MCP 的行业影响正在扩大。除了 Claude Code，VS Code 的 GitHub Copilot 也开始支持 MCP，Cursor、Windsurf 等 AI 编码工具也在跟进。**MCP 正在成为 AI Agent 工具调用的事实标准。**

但 MCP 也面临挑战。协议本身还在快速演进中，版本兼容性是个问题。六种传输协议的存在也说明社区对最佳传输方案还没有共识。此外，MCP Server 的质量参差不齐，缺乏统一的安全审计标准。

对于开发者来说，现在投入精力学习和使用 MCP 是值得的。**它代表了 AI 工具生态的发展方向：标准化、可组合、安全可控。**

下一篇：[10-未来功能蓝图](docs/10-未来功能蓝图.md)
