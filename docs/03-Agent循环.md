[English](./03-Agent-Loop.md)

# 03 Agent 循环

## 核心设计：AsyncGenerator + ReAct

Claude Code 的 Agent 循环实现在 `src/query.ts`（1,700 行），核心是一个基于 AsyncGenerator 的 ReAct 循环。

用 AsyncGenerator 做主循环是一个精巧的选择。它天然支持流式输出，调用方可以逐事件消费也可以一次性 collect。Generator 的暂停/恢复语义天然适配 Agent 的等待工具结果、等待用户确认这些场景。

## 六阶段循环

```
┌──────────────────────────────────────────────────┐
│                  query loop                       │
│                                                  │
│  ┌──────────┐                                    │
│  │ 1. 预取   │ 记忆文件、Skill 列表（异步）       │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 2. 构建   │ system prompt + messages + tools   │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 3. API   │ 流式调用，逐 token yield            │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 4. 工具   │ 权限检查 → 并行执行 → Hook         │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 5. 压缩   │ 超阈值？微压缩/SM/Full Compact     │
│  └────┬─────┘                                    │
│       ↓                                          │
│  ┌──────────┐                                    │
│  │ 6. 继续？ │ tool_use → 循环 | end_turn → 结束  │
│  └──────────┘                                    │
└──────────────────────────────────────────────────┘
```

### 阶段一：预取

在调 API 之前，异步加载几类信息：
- 记忆文件（CLAUDE.md、MEMORY.md）
- Skill 列表（后台发现）
- Git 状态（有 memoize 缓存）

预取的好处是避免了主循环中的 IO 等待。这些信息大部分在 session 内不变，缓存命中后直接跳过。

### 阶段二：构建上下文

组装发送给 API 的完整请求：
- system prompt = 内置指令 + CLAUDE.md 记忆 + Git 状态 + 权限规则
- messages = 用户消息 + 历史对话 + 工具结果
- tools = 已注册工具列表（40+，feature flag 控制）
- thinking = Extended Thinking 配置

### 阶段三：流式调用 API

通过 Anthropic SDK 发起流式请求，逐 token yield 给上层。上层（终端 UI）收到 token 后实时渲染。

API 客户端有重试逻辑和模型 fallback 链：Opus → Sonnet → Small。

### 阶段四：工具执行

API 返回 tool_use block 后进入工具执行：

```
tool_use blocks
  ↓
Pre-Hook（前置钩子：日志、参数校验）
  ↓
权限检查（default/auto/plan 三种模式）
  ↓
查找工具（按 name 在注册表中匹配）
  ↓
并行执行（Promise.all，多个工具可同时跑）
  ↓
Post-Hook（后置钩子：结果清洗、审计日志）
  ↓
tool_result 追加到消息历史
```

### 阶段五：压缩判断

检查当前 token 用量是否超过阈值。详见 [05-消息压缩系统](./05-compaction-system.md)。

### 阶段六：继续决策

根据 API 返回的 `stop_reason` 决定：
- `tool_use`：还有工具调用要执行，继续循环
- `end_turn`：模型认为任务完成，结束
- `max_tokens`：输出达到上限，可能需要继续

## 状态管理

循环内维护 7 个状态变量，每次继续时创建新的 state 拷贝：

```typescript
let state = {
  messages,                  // 消息历史
  toolUseContext,           // 工具执行上下文
  maxOutputTokensOverride,  // 输出 token 限制
  autoCompactTracking,      // 压缩追踪状态
  // ... 3 more
}

// 每次循环创建新拷贝，不修改原对象
state = { ...state, messages: newMessages }
```

## Extended Thinking 保留

代码中有大段注释解释 thinking block 的三条规则：

1. thinking block 必须存在于 max_thinking_length > 0 的请求中
2. thinking block 不能是一个 block 序列的最后一个元素
3. thinking block 必须在整个 assistant 轨迹中持久化保留

这意味着当工具调用发生时，thinking block 不能被丢弃，必须在 tool_use 和 tool_result 之间正确保留，否则模型的推理链会断裂。

## Agent Fork 机制

Claude Code 支持 fork 子 Agent 来执行 Skill 或并行任务：

```
主 Agent
  ├── fork → Skill Agent（独立 token 预算、独立消息历史）
  ├── fork → Compact Agent（做压缩摘要）
  └── fork → Session Memory Agent（做记忆沉淀）
```

fork 的关键设计是 **CacheSafeParams**：子 Agent 的 system prompt 和上下文必须和父 Agent 匹配，这样才能命中 prompt cache，避免重复计费。
