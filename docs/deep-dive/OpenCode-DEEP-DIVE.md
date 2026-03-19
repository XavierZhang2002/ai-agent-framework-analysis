# OpenCode - 深度技术调研

!!! info "项目概览"
    **调研日期**: 2026-02-26 (Updated: 2026-02-27)  
    **仓库**: https://github.com/anomalyco/opencode  
    **许可证**: MIT  
    **主要语言**: TypeScript  
    **运行时**: Bun  

---

## 1. 项目概述

### 1.1 定位

OpenCode 是一个 **100% 开源**、**provider-agnostic** 的终端 coding agent，强调：
- ✅ TUI (Terminal User Interface) 优先体验
- ✅ 跨模型供应商（OpenAI、Anthropic、Google、Qwen 等）
- ✅ 完全可审计的编排逻辑
- ✅ 强大的权限控制系统
- ✅ 插件与配置驱动的扩展性

### 1.2 核心架构

```
用户终端
    ↓
OpenCode TUI/CLI (Bun + TypeScript)
    ↓
Provider Abstraction Layer (Vercel AI SDK)
    ↓
LLM APIs (OpenAI/Anthropic/Google/Qwen/...)
```

**与竞品的差异**:
- 无依赖闭源 CLI（vs. Claude Agent SDK）
- 纯客户端运行（vs. 托管服务）
- 完全开源的编排逻辑

---

## 2. 代码架构深度分析

### 2.1 目录结构（39 个顶级模块，~47,324 行 TypeScript）

```
packages/opencode/src/
├── acp/              # Agent Client Protocol 支持
├── agent/            # Agent profile 系统
├── auth/             # 认证提供商
├── bus/              # 事件总线系统
├── cli/              # 命令行接口
├── command/          # 自定义命令系统
├── config/           # 配置管理
├── file/             # 文件操作
├── permission/       # 权限引擎 ★★★
├── provider/         # LLM provider 抽象 ★★★
├── session/          # Session 管理 ★★★
├── tool/             # 工具实现 ★★★
├── server/           # HTTP 服务器 (Hono)
├── storage/          # 数据库 (SQLite/Drizzle)
├── mcp/              # Model Context Protocol
├── skill/            # Skills 系统
├── plugin/           # 插件系统
└── [24+ more modules]
```

### 2.2 核心类关系

```
Config (配置中心)
    ↓
AgentProfile (build/plan/general/explore)
    ↓
Session (会话管理)
    ↓
SessionPrompt.loop() (主执行循环) ← 核心文件: packages/opencode/src/session/prompt.ts:347-738
    ↓
Provider (LLM 调用)
    ↓
Tool Registry (工具执行)
    ↓
Permission Engine (权限检查)
```

### 2.3 系统架构

```
TUI (SolidJS) → SDK (@opencode-ai/sdk) → HTTP API → Core Loop (prompt.ts)
                                    ↓
                              Event Bus (SSE)
```

### 2.4 关键设计模式

1. **Event-Driven**: 中央事件总线用于 pub/sub 消息传递
2. **Instance-Scoped State**: `Instance.state()` 模式用于per-project状态隔离
3. **Functional API**: `fn()` wrapper使用Zod schemas进行输入验证
4. **Plugin-Based**: 基于 hook 的插件系统，带生命周期回调

### 2.5 核心模块清单

| 模块 | 文件路径 | 功能描述 |
|------|----------|----------|
| **Core Loop** | `packages/opencode/src/session/prompt.ts` | 主执行循环 (lines 347-738) |
| **Context Compaction** | `packages/opencode/src/session/compaction.ts` | 上下文压缩与Token管理 |
| **LLM Processor** | `packages/opencode/src/session/processor.ts` | LLM请求处理与流式响应 |
| **Permission Engine** | `packages/opencode/src/permission/next.ts` | 权限规则评估引擎 |

---

## 3. 底层实现原理

### 3.1 Run 命令工作流程

**文件**: `src/cli/cmd/run.ts:221-623`

```typescript
// 1. 参数解析 (lines 221-300)
- message: 文本/文件发送
- --agent: agent profile
- --model: provider/model 覆盖
- --session/-s: session ID 继续
- --continue/-c: 继续上次 session
- --fork: fork 现有 session

// 2. 权限设置 (lines 352-368)
const rules: PermissionNext.Ruleset = [
  { permission: "question", action: "deny", pattern: "*" },
  { permission: "plan_enter", action: "deny", pattern: "*" },
  { permission: "plan_exit", action: "deny", pattern: "*" },
]

// 3. Session 管理 (lines 376-389)
- 创建新 session 或继续现有
- 处理 forking: sdk.session.fork()
- 设置自定义标题

// 4. 执行循环 (lines 406-608)
- 订阅 SSE 事件流
- 处理事件: message.updated, message.part.updated, session.status, permission.asked
- 内联渲染工具输出 (bash, edit, read, task 等)
- 流式传输文本响应到 stdout

// 5. Bootstrap 或 Attach (lines 611-623)
- 如果 --attach: 连接到远程服务器
- 否则: 通过 bootstrap() 启动本地服务器
```

**工具渲染调度** (lines 406-426):
```typescript
function tool(part: ToolPart) {
  if (part.tool === "bash") return bash(props<typeof BashTool>(part))
  if (part.tool === "glob") return glob(props<typeof GlobTool>(part))
  if (part.tool === "edit") return edit(props<typeof EditTool>(part))
  if (part.tool === "read") return read(props<typeof ReadTool>(part))
  // ... 分发到工具特定的格式化器
}
```

### 3.2 Agent Profiles 系统

**文件**: `src/agent/agent.ts:76-203`

**内置 Agents**:

1. **build** (primary): 默认执行 agent
   - 完整工具访问
   - 允许 question 和 plan_enter
   - 权限从 defaults + user config 合并

2. **plan** (primary): 仅规划模式
   - 只读工具（deny 所有编辑，除了 plan 文件）
   - 特殊目录白名单：`.opencode/plans/`
   - 工作流定义在 `session/prompt/plan.txt`

3. **general** (subagent): 多任务执行
   - Denies todoread/todowrite
   - 用于并行任务执行

4. **explore** (subagent): 代码库探索
   - 限制为: grep, glob, list, bash, webfetch, read
   - 快速搜索，无写入
   - 系统提示: `agent/prompt/explore.txt`

5. **compaction**, **title**, **summary** (hidden): 内部 agents

**Agent 配置** (lines 205-232):
```typescript
// 用户配置覆盖原生 agents
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) { delete result[key]; continue }
  
  item.model = value.model ? Provider.parseModel(value.model) : undefined
  item.variant = value.variant
  item.prompt = value.prompt
  item.temperature = value.temperature
  item.permission = PermissionNext.merge(
    item.permission, 
    PermissionNext.fromConfig(value.permission)
  )
}
```

### 3.3 权限决策引擎

**文件**: `src/permission/next.ts`

**核心评估逻辑** (lines 236-243):
```typescript
export function evaluate(
  permission: string, 
  pattern: string, 
  ...rulesets: Ruleset[]
): Rule {
  const merged = merge(...rulesets)
  const match = merged.findLast(
    (rule) => Wildcard.match(permission, rule.permission) && 
              Wildcard.match(pattern, rule.pattern)
  )
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

**权限请求流程** (lines 131-161):
```typescript
export const ask = fn(
  Request.partial({ id: true }).extend({ ruleset: Ruleset }),
  async (input) => {
    for (const pattern of request.patterns ?? []) {
      const rule = evaluate(request.permission, pattern, ruleset, s.approved)
      
      if (rule.action === "deny") 
        throw new DeniedError(ruleset.filter(...))
      
      if (rule.action === "ask") {
        return new Promise<void>((resolve, reject) => {
          s.pending[id] = { info, resolve, reject }
          Bus.publish(Event.Asked, info)  // 触发 UI 提示
        })
      }
      
      if (rule.action === "allow") continue
    }
  }
)
```

**回复处理** (lines 163-234):
```typescript
export const reply = fn(..., async (input) => {
  const existing = s.pending[input.requestID]
  delete s.pending[input.requestID]
  
  if (input.reply === "reject") {
    existing.reject(new RejectedError())
  }
  
  if (input.reply === "always") {
    // 添加到批准列表
    s.approved.push({ permission, pattern, action: "allow" })
    // 自动批准现在匹配的待处理请求
  }
})
```

**Ruleset 来源**（优先级顺序）:
1. Agent 默认值（硬编码在 `agent/agent.ts`）
2. 用户配置（`~/.config/opencode/opencode.json`）
3. Session 特定规则（传递给 `Session.create()`）

### 3.4 Task Delegation（Sub-agents）实现

**文件**: `src/tool/task.ts:45-163`

**工具定义** (lines 14-25):
```typescript
const parameters = z.object({
  description: z.string().describe("A short (3-5 words) description"),
  prompt: z.string().describe("The task for the agent to perform"),
  subagent_type: z.string().describe("Type of specialized agent"),
  task_id: z.string().optional().describe("Resume previous task"),
  command: z.string().optional().describe("Command that triggered this"),
})
```

**执行流程**:

1. **权限检查** (lines 49-59):
   ```typescript
   if (!ctx.extra?.bypassAgentCheck) {
     await ctx.ask({
       permission: "task",
       patterns: [params.subagent_type],
       always: ["*"],
     })
   }
   ```

2. **Session 创建** (lines 66-102):
   ```typescript
   const session = await iife(async () => {
     if (params.task_id) {
       return await Session.get(params.task_id)  // 恢复
     }
     
     return await Session.create({
       parentID: ctx.sessionID,  // 创建子 session
       title: params.description + ` (@${agent.name} subagent)`,
       permission: [
         { permission: "todowrite", action: "deny" },
         { permission: "todoread", action: "deny" },
         // 防止递归 task 调用，除非 agent 允许
         ...(hasTaskPermission ? [] : [{ permission: "task", action: "deny" }])
       ]
     })
   })
   ```

3. **模型选择** (lines 106-109):
   ```typescript
   const model = agent.model ?? {
     modelID: msg.info.modelID,      // 从 parent 继承
     providerID: msg.info.providerID,
   }
   ```

4. **提示执行** (lines 126-143):
   ```typescript
   const result = await SessionPrompt.prompt({
     messageID,
     sessionID: session.id,
     model,
     agent: agent.name,
     tools: {
       todowrite: false,
       todoread: false,
       ...(hasTaskPermission ? {} : { task: false })
     },
     parts: await SessionPrompt.resolvePromptParts(params.prompt)
   })
   ```

5. **输出格式** (lines 145-153):
   ```typescript
   const output = [
     `task_id: ${session.id} (for resuming to continue this task if needed)`,
     "",
     "<task_result>",
     text,
     "</task_result>",
   ].join("\n")
   ```

**关键设计模式**:
- 分层 sessions（parent/child 关系）
- 权限继承与限制
- 模型从 parent 消息继承
- 通过 `ctx.abort` 传播取消

### 3.5 Session 管理与事件流式传输

**Session 生命周期** (`src/session/index.ts:287-326`):

```typescript
export async function createNext(input) {
  const result: Info = {
    id: Identifier.descending("session", input.id),  // ULID
    slug: Slug.create(),
    projectID: Instance.project.id,
    directory: input.directory,
    permission: input.permission,
    time: { created: Date.now(), updated: Date.now() }
  }
  
  Database.use((db) => {
    db.insert(SessionTable).values(toRow(result)).run()
    Database.effect(() => Bus.publish(Event.Created, { info: result }))
  })
}
```

**消息结构** (`src/session/message-v2.ts`):
- User messages: text, file, agent parts
- Assistant messages: text, reasoning, tool, step-start/finish parts
- Parts 单独存储，支持 delta streaming

**事件流式传输** (`src/server/routes/session.ts`):
```typescript
streamSSE(c, async (stream) => {
  const unsub = Bus.subscribe((event) => {
    stream.writeSSE({
      data: JSON.stringify(event),
      event: event.type,
      id: String(Date.now())
    })
  })
})
```

### 3.5 核心 Agent 循环 (SessionPrompt.loop)

**文件**: `packages/opencode/src/session/prompt.ts`  
**函数**: `SessionPrompt.loop()`  
**代码行**: 347-738

这是 OpenCode 的核心执行引擎，负责协调 LLM 调用、工具执行、上下文管理和子任务调度。

#### 3.5.1 核心循环实现

```typescript
export async function loop(sessionID: string, options?: LoopOptions): Promise<MessageV2.WithParts> {
  let step = 0
  const session = await Session.get(sessionID)
  
  while (true) {
    SessionStatus.set(sessionID, { type: "busy" })
    
    // 1. 过滤已压缩的消息，获取当前有效消息列表
    let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))
    
    // 2. 反向遍历查找最后的关键消息
    let lastUser, lastAssistant, lastFinished
    for (let i = msgs.length - 1; i >= 0; i--) {
      const msg = msgs[i]
      if (!lastUser && msg.info.role === "user") lastUser = msg.info
      if (!lastAssistant && msg.info.role === "assistant") lastAssistant = msg.info
      if (!lastFinished && msg.info.role === "assistant" && msg.info.finish)
        lastFinished = msg.info
    }
    
    // 3. 退出条件：Assistant 已完成响应且晚于最后一条用户消息
    if (lastAssistant?.finish && lastUser.id < lastAssistant.id) {
      break
    }
    
    step++
    
    // 4. 处理子任务（Subtask）- 并行Agent调用
    const task = tasks.pop()
    if (task?.type === "subtask") {
      // 执行 task 工具，创建子Session执行并行任务
      // 结果通过消息系统返回给父Session
      continue
    }
    
    // 5. 处理上下文压缩任务
    if (task?.type === "compaction") {
      const result = await SessionCompaction.process({...})
      if (result === "stop") break  // 压缩失败或用户取消
      continue
    }
    
    // 6. 检查上下文Token溢出
    if (await SessionCompaction.isOverflow({ tokens: lastFinished.tokens, model })) {
      await SessionCompaction.create({ sessionID, auto: true })
      continue  // 触发压缩后重新循环
    }
    
    // 7. 正常 LLM 处理流程
    const processor = SessionProcessor.create({...})
    const tools = await resolveTools({...})
    
    const result = await processor.process({
      user: lastUser,
      tools,
      model,
    })
    
    // 8. 处理结果状态
    if (result === "stop") break
    if (result === "compact") {
      await SessionCompaction.create({...})
    }
  }
}
```

#### 3.5.2 循环阶段详解

| 阶段 | 代码位置 | 功能描述 |
|------|----------|----------|
| **消息过滤** | Lines 360-365 | 过滤已压缩消息，维护有效上下文 |
| **消息扫描** | Lines 368-378 | 反向遍历找 lastUser/lastAssistant/lastFinished |
| **退出检测** | Lines 381-385 | 判断 Assistant 是否已完成对用户请求的响应 |
| **子任务处理** | Lines 390-396 | 执行 Task 工具，创建子Agent并行执行 |
| **Compaction** | Lines 399-406 | 处理上下文压缩队列任务 |
| **溢出检查** | Lines 409-413 | 检测 Token 是否超过模型限制 |
| **LLM 调用** | Lines 416-426 | 创建 Processor，解析工具，发起 LLM 请求 |
| **结果处理** | Lines 429-435 | 处理 stop/compact 等返回状态 |

#### 3.5.3 核心特性详解

**1. 上下文压缩（Context Compaction）**

- **触发条件**: Token 数超过模型限制的阈值（通常 20k-40k tokens）
- **实现文件**: `packages/opencode/src/session/compaction.ts`
- **压缩策略**:
  - **Pruning**: 移除旧的工具输出和中间结果
  - **Summarization**: 创建合成继续提示，保留关键上下文
- **自动触发**: 每次循环检查 `SessionCompaction.isOverflow()`

**2. 子任务系统（Subtask System）**

- **实现方式**: Task 工具 (`src/tool/task.ts`)
- **工作机制**:
  - 通过 `task` 工具创建子 Session
  - 子 Agent 独立执行，结果通过消息系统返回
  - 支持任务恢复（通过 `task_id`）
- **并发控制**: 通过 `tasks` 队列管理待执行的子任务

**3. 权限系统（Permission System）**

- **实现文件**: `packages/opencode/src/permission/next.ts`
- **规则引擎**: 基于通配符的模式匹配（Wildcard matching）
- **评估流程**:
  1. Agent 默认规则 → 2. 用户配置 → 3. Session 特定规则
  4. 对于每个工具调用，评估 action: `allow`/`deny`/`ask`
- **用户交互**: 通过 SSE 事件流触发 UI 权限询问

**4. 多 Agent 支持（Multi-Agent）**

- **Agent 类型**: Primary (用户可选择) / Subagent (通过 task 调用)
- **Agent Profile**: 每个 Agent 可配置独立的模型、提示词、权限
- **Agent 切换**: 通过 `SessionPrompt.loop()` 的 `agent` 参数指定

**5. Doom Loop 检测**

- **检测机制**: 跟踪最近 3+ 次相同的工具调用
- **处理策略**: 检测到循环时触发上下文压缩或终止执行
- **防护目的**: 防止 Agent 在错误的代码/配置中无限循环

---

## 4. 与竞品对比

### 4.1 OpenCode vs. OpenAI Agents SDK

| 维度 | OpenCode | OpenAI Agents SDK |
|------|---------|-------------------|
| **架构** | TUI + HTTP Server | 纯 Python SDK |
| **语言** | TypeScript (Bun) | Python |
| **Provider** | ✅ Provider-agnostic | ✅ Provider-agnostic (100+ LLMs) |
| **Session** | ✅ SQLite (built-in) | ✅ SQLite/Redis/自定义 |
| **Tracing** | ⚠️ OpenTelemetry (experimental) | ✅ 内置（多后端） |
| **Handoff** | ✅ task 工具 + sub-agents | ✅ Handoff 类（first-class） |
| **Permissions** | ✅ ✅ ✅ 高级（ruleset + wildcards） | ✅ Guardrails |
| **TUI** | ✅ ✅ ✅ Solid.js + Tauri | ❌ CLI-only |
| **Plugin System** | ✅ Native plugins | ⚠️ 通过 Protocol 扩展 |
| **MCP** | ✅ Client 模式 | ⚠️ 通过 tools |
| **使用场景** | 终端重度用户 | Python 应用嵌入 |

### 4.2 OpenCode vs. Codex CLI

| 维度 | OpenCode | Codex CLI |
|------|---------|-----------|
| **语言** | TypeScript (Bun) | Rust + TypeScript |
| **Sandbox** | ❌ 无内置 | ✅ Seatbelt/Landlock/Windows |
| **Approval** | Permission ruleset | .rules 文件 |
| **MCP** | ✅ Client | ✅ Client + Server（双角色） |
| **Provider** | ✅ Provider-agnostic | ⚠️ OpenAI 为主 |
| **TUI** | ✅ Solid.js | ✅ ratatui (Rust) |
| **成熟度** | 社区驱动 | OpenAI 官方 |

---

## 5. 扩展开发指南

### 5.1 创建自定义 Agent Profiles

**通过配置文件** (`~/.config/opencode/opencode.json` 或 `.opencode/opencode.json`):

```jsonc
{
  "agent": {
    "myagent": {
      "name": "myagent",
      "description": "Custom agent for X",
      "mode": "primary",  // 或 "subagent" 或 "all"
      "model": "anthropic/claude-3-5-sonnet-20241022",
      "variant": "high",  // reasoning effort
      "temperature": 0.7,
      "top_p": 0.9,
      "color": "#ff5733",
      "hidden": false,
      "steps": 50,  // max LLM turns
      "prompt": "You are a specialized agent for...",
      "permission": {
        "edit": "allow",
        "bash": "ask",
        "external_directory": {
          "/tmp/*": "allow",
          "*": "deny"
        }
      },
      "options": {
        "custom_key": "custom_value"
      }
    }
  }
}
```

**Agent 模式**:
- `primary`: 在 agent 切换器中显示，用户可选择
- `subagent`: 仅通过 task 工具可调用
- `all`: 既是 primary 也是 subagent

### 5.2 创建自定义工具

**1. 文件型工具** (`.opencode/tool/mytool.ts` 或 `.opencode/tools/mytool.ts`):

```typescript
import type { ToolDefinition } from "@opencode-ai/plugin"

export default {
  description: "Does something useful",
  args: {
    input: { type: "string", description: "Input parameter" },
    count: { type: "number", description: "How many times" },
  },
  async execute(args, ctx) {
    // ctx.sessionID, ctx.messageID, ctx.abort, ctx.directory, ctx.worktree
    
    // 执行工作
    const result = doSomething(args.input, args.count)
    
    return `Result: ${result}`
  }
} satisfies ToolDefinition
```

**2. 高级工具（Tool.define）** (for native integration):

```typescript
// packages/opencode/src/tool/mytool.ts
import { Tool } from "./tool"
import z from "zod"

const parameters = z.object({
  path: z.string(),
  options: z.record(z.string(), z.any()).optional(),
})

export const MyTool = Tool.define("mytool", async (ctx) => {
  // ctx.agent 在这里可用，用于 agent 特定行为
  
  return {
    description: "Advanced tool with validation",
    parameters,
    async execute(params, ctx) {
      // 权限检查
      await ctx.ask({
        permission: "mytool",
        patterns: [params.path],
        always: [params.path],
        metadata: { options: params.options }
      })
      
      // Abort 支持
      ctx.abort.throwIfAborted()
      
      // 执行期间更新元数据
      await ctx.metadata({
        title: "Processing...",
        metadata: { progress: 0.5 }
      })
      
      // 返回结构化结果
      return {
        title: "Success",
        metadata: { processed: true },
        output: "Tool output here",
        attachments: [
          {
            type: "file",
            url: "file:///path/to/file",
            filename: "result.txt",
            mime: "text/plain"
          }
        ]
      }
    },
    formatValidationError(error: z.ZodError) {
      return `Custom error: ${error.errors[0].message}`
    }
  }
})
```

**3. 注册到 Tool Registry**:

```typescript
// packages/opencode/src/tool/registry.ts
import { MyTool } from "./mytool"

// 添加到 all() 函数中的 tools 数组 (line 103)
return [
  // ... 现有工具
  MyTool,
]
```

### 5.3 扩展权限策略

**1. 基于配置**（最常见）:

```jsonc
{
  "permission": {
    // 全局 deny/allow/ask
    "bash": "ask",
    "edit": "allow",
    
    // 基于模式的规则
    "read": {
      "*": "allow",
      "*.env": "ask",              // 读取 .env 文件前询问
      "*.env.*": "ask",
      "*.env.example": "allow"     // 覆盖示例
    },
    
    "external_directory": {
      "/tmp/*": "allow",
      "/home/user/safe/*": "allow",
      "*": "ask"
    },
    
    // 自定义工具权限
    "mytool": {
      "*.txt": "allow",
      "*.md": "allow",
      "*": "deny"
    }
  }
}
```

**2. Agent 特定覆盖**:

```jsonc
{
  "agent": {
    "restrictive": {
      "name": "restrictive",
      "mode": "primary",
      "permission": {
        "bash": "deny",      // 覆盖全局
        "edit": "ask",
        "read": {
          "/etc/*": "deny"   // 与全局 read 规则合并
        }
      }
    }
  }
}
```

**3. Session 特定**（编程方式）:

```typescript
// 创建 session 时
await sdk.session.create({
  title: "Restricted Session",
  permission: [
    { permission: "bash", pattern: "*", action: "deny" },
    { permission: "edit", pattern: "*.md", action: "allow" },
    { permission: "edit", pattern: "*", action: "deny" }
  ]
})
```

### 5.4 配置系统

**配置文件位置**（优先级: low → high）:

1. Remote `.well-known/opencode` (enterprise)
2. `~/.config/opencode/opencode.json{,c}` (global)
3. `$OPENCODE_CONFIG` (custom path)
4. `opencode.json{,c}` (project root)
5. `.opencode/opencode.json{,c}` (project .opencode dir)
6. `$OPENCODE_CONFIG_CONTENT` (inline JSON)
7. `/etc/opencode/` or `C:\ProgramData\opencode\` (managed, enterprise)

**完整配置 Schema** (`src/config/config.ts`):

```typescript
{
  "$schema": "https://opencode.ai/config.json",
  
  "username": "string",
  "default_agent": "build",
  
  "agent": {
    "agentname": {
      "name": "string",
      "description": "string",
      "mode": "primary" | "subagent" | "all",
      "model": "provider/model",
      "variant": "high" | "max" | "minimal",
      "temperature": 0.7,
      "prompt": "string",
      "permission": { /* ... */ },
      "disable": false
    }
  },
  
  "command": {
    "cmdname": {
      "description": "string",
      "agent": "agentname",
      "template": "Prompt with $1 $2 $ARGUMENTS",
      "subtask": true
    }
  },
  
  "permission": {
    "tool": "allow" | "deny" | "ask",
    "tool": { "pattern": "action" }
  },
  
  "plugin": ["package@version", "file:///path"],
  "instructions": ["Custom instruction 1"],
  
  "experimental": {
    "openTelemetry": false,
    "batch_tool": false
  }
}
```

---

## 6. 生产部署考量

### 6.1 性能优化

- **事件总线优化**: 使用 `Bus.publish()` 而非直接调用
- **数据库连接池**: SQLite 使用 better-sqlite3 的 connection pooling
- **流式传输**: 所有 LLM 响应都是流式的
- **增量 Delta**: IDE 上下文只发送变更

### 6.2 监控

```typescript
// 通过 experimental.openTelemetry
{
  "experimental": {
    "openTelemetry": true
  }
}

// 环境变量
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

---

## 7. 评估与建议

!!! success "核心优势"
    1. ✅ **100% 开源**: 完全可审计的代码
    2. ✅ **Provider-agnostic**: 支持所有主流 LLM
    3. ✅ **强大的 TUI**: Solid.js + Tauri 桌面应用
    4. ✅ **高级权限系统**: 细粒度控制
    5. ✅ **插件生态**: 丰富的扩展机制

!!! warning "局限性"
    1. ⚠️ **Bun 依赖**: 需要 Bun 运行时
    2. ❌ **无内置 Tracing**: OpenTelemetry 是实验性的
    3. ⚠️ **社区维护**: 非大厂官方维护

!!! tip "推荐场景"
    - ✅ 终端重度用户
    - ✅ 需要完全开源方案
    - ✅ 需要跨 LLM provider
    - ✅ 需要强大权限控制

---

**文档版本**: 1.1  
**最后更新**: 2026-02-27
