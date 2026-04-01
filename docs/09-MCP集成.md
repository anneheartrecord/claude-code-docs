[English](./09-MCP-Integration.md)

# 09 MCP 集成

## MCP 是什么

Model Context Protocol 是 Anthropic 提出的标准化工具调用协议。Agent 不需要为每个外部服务写一套对接逻辑，只要服务实现了 MCP 协议，Agent 就能直接调用。

Claude Code 的 MCP 实现位于 `src/services/mcp/`，包含 24 个文件、12,000+ 行代码，是代码量最大的子系统之一。

## 七种配置作用域

MCP Server 可以在不同层级配置：

| 作用域 | 位置 | 场景 |
|--------|------|------|
| local | 项目目录下 | 项目特有的工具 |
| user | ~/.claude/ | 个人全局工具 |
| project | .claude/ | 项目共享，checkin 到 git |
| dynamic | 运行时注入 | 插件动态注册 |
| enterprise | 管理员配置 | 企业统一工具 |
| claudeai | Anthropic 官方 | 官方提供的服务 |
| managed | 远程管理 | 集中管控 |

## 六种传输协议

| 协议 | 说明 |
|------|------|
| stdio | 标准输入输出，最常用 |
| sse | Server-Sent Events |
| sse-ide | IDE 专用 SSE |
| http | HTTP REST |
| ws | WebSocket |
| sdk | 进程内 SDK 调用 |

## OAuth 支持

MCP Server 可以配置 OAuth 认证：

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

还支持 XAA（Cross-App Access），允许多个 MCP Server 共享同一个 IdP 的认证。

## 权限管理

MCP 工具和内置工具一样受权限系统管控。`channelPermissions.ts` 对每个 MCP Server 的每个工具做独立的权限控制。

## 为什么 MCP 模块这么大

12,000 行做 MCP 集成，复杂度明显过高。分析原因：

1. 要支持 6 种传输协议，每种都有自己的连接管理、重试逻辑、错误处理
2. OAuth 流程本身就很复杂
3. 7 种配置作用域的合并和优先级处理
4. 大量的防御性代码，处理各种边界情况

这是 AI 生成代码的典型特征：面对一个不够理解的复杂协议，用大量代码做覆盖式处理，而人写的话会先把协议吃透再用更精练的方式实现。
