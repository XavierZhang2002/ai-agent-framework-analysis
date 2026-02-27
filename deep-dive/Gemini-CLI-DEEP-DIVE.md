# Gemini CLI - 深度技术调研

**调研日期**: 2026-02-26  
**仓库**: https://github.com/google-gemini/gemini-cli  
**许可证**: Apache-2.0  
**主要语言**: TypeScript  
**定位**: Terminal-first 代理循环 + 内建工具，让 Gemini 能力直接在终端可用  

---

## 1. 项目概述

### 核心特点

- ✅ **Google 生态深度集成**: OAuth2, Code Assist, Google Search grounding
- ✅ **MCP 原生支持**: Transport (Stdio/SSE/HTTP)
- ✅ **多层安全**: Policy engine, approval modes, trusted folders
- ✅ **Monorepo 架构**: CLI + Core + SDK + VS Code extension + A2A server
- ✅ **免费额度**: 1,000 requests/day (Google OAuth 认证)
- ✅ **分层记忆系统**: GEMINI.md (Global/Project/JIT 三级)

---

## 2. 代码架构

### 目录结构

```
gemini-cli/
├── packages/
│   ├── cli/              # Frontend terminal UI (React/Ink)
│   ├── core/             # Core engine (866 files)
│   ├── sdk/              # Public SDK
│   ├── devtools/         # Developer tools
│   ├── a2a-server/       # Agent-to-Agent server
│   └── vscode-ide-companion/
├── docs/
├── evals/
└── integration-tests/
```

### 核心模块 (packages/core/src/)

```
core/src/
├── agents/           # Agent 执行系统
├── code_assist/      # OAuth2 & Code Assist 集成
├── config/           # 配置管理 (2905 lines in config.ts)
├── core/             # LLM client & chat (1120 lines in client.ts)
├── mcp/              # MCP 实现
├── policy/           # 安全策略引擎 (792 lines)
├── scheduler/        # 工具执行调度器 (758 lines)
├── tools/            # 30+ 工具
│   ├── mcp-client.ts # 2073 lines - MCP 发现层
│   ├── mcp-tool.ts   # MCP 工具包装
│   ├── web-search.ts # 246 lines - Google Search grounding
│   └── shell.ts      # 524 lines - Shell 执行
└── services/         # 核心服务
```

---

## 3. 底层实现

### 3.1 Terminal Agent 架构 (Core Agent Loop)

```
GeminiClient (Orchestrator)
  ↓
processTurn() - Single turn processing (Lines 248-358)
  ↓
Turn.run() - Stream from Gemini API
  ↓
Scheduler - Tool execution with policy checks
```

**核心入口** (`packages/cli/src/gemini.tsx` lines 1-906):
```typescript
1. initializeApp() - 认证 & 设置
2. render(<AppContainer />) - 启动 React UI
3. GeminiClient.sendMessageStream() - 流式 LLM 响应
4. Scheduler.schedule() - 并行执行工具
5. Turn management - 跟踪对话状态
```

**GeminiClient** (`/packages/core/src/core/client.ts` lines 82-1120):

核心方法:
- `sendMessageStream()` - Lines 360-450 - 主消息流处理
- `processTurn()` - Lines 248-358 - 单轮次处理

```typescript
class GeminiClient {
  async *sendMessageStream(
    request: PartListUnion,
    signal: AbortSignal,
    prompt_id: string,
    turns: number = MAX_TURNS,
  ): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
    
    // 1. 检查会话轮次限制
    this.sessionTurnCount++;
    if (this.sessionTurnCount > this.config.getMaxSessionTurns()) {
      yield { type: GeminiEventType.MaxSessionTurns };
      return turn;
    }
    
    // 2. 尝试对话压缩 (Chat Compression)
    const compressed = await this.tryCompressChat(prompt_id, false);
    
    // 3. 循环检测 (Loop Detection)
    if (this.loopDetector.addAndCheck(event)) {
      yield { type: GeminiEventType.LoopDetected };
      return turn;
    }
    
    // 4. 执行单轮对话
    const turn = new Turn(this.getChat(), prompt_id);
    const resultStream = turn.run(modelConfigKey, requestToSent, signal);
    
    // 5. 处理流式响应
    for await (const event of resultStream) {
      yield event;
    }
    
    // 6. 下一位说话者检查 - 必要时自动继续
    if (!turn.pendingToolCalls.length) {
      const nextSpeakerCheck = await checkNextSpeaker(...);
      if (nextSpeakerCheck?.next_speaker === 'model') {
        const nextRequest = [{ text: 'Please continue.' }];
        yield* this.sendMessageStream(nextRequest, signal, prompt_id, boundedTurns - 1);
      }
    }
  }
}
```

### 3.2 MCP 集成 (Full MCP Support with OAuth)

**MCP Client** (`/packages/core/src/tools/mcp-client.ts` - 2073 lines):

```typescript
class McpClient {
  async connect(): Promise<void>
  async discover(cliConfig: Config): Promise<void>
  async discoverTools(): Promise<void>
  async discoverResources(): Promise<void>
  async discoverPrompts(): Promise<void>
}
```

**Transport 层**:
- **StdioClientTransport**: 生成子进程 (Stdio)
- **SSEClientTransport**: Server-Sent Events
- **StreamableHTTPClientTransport**: HTTP streaming

**认证** (`/packages/core/src/mcp/`):
- **自动 OAuth 发现**: Automatic OAuth discovery
- OAuth provider (`oauth-provider.ts`)
- Google auth (`google-auth-provider.ts`)
- Service account impersonation (`sa-impersonation-provider.ts`)
- Token 存储（keychain/file hybrid）

### 3.3 Google Search Grounding

**实现** (`/packages/core/src/tools/web-search.ts` lines 1-246):

```typescript
class WebSearchToolInvocation extends BaseToolInvocation {
  async execute(signal: AbortSignal): Promise<WebSearchToolResult> {
    // 调用特殊 "web-search" model
    const response = await geminiClient.generateContent(
      { model: 'web-search' },
      [{ role: 'user', parts: [{ text: this.params.query }] }],
      signal,
      LlmRole.UTILITY_TOOL
    );
    
    // 提取 grounding metadata
    const groundingMetadata = response.candidates?.[0]?.groundingMetadata;
    const sources = groundingMetadata?.groundingChunks;
    
    // 使用 UTF-8 字节位置插入内联引用
    // 返回格式化响应with [1], [2] citations
  }
}
```

### 3.4 GEMINI.md - 分层记忆系统

**实现** (`/packages/core/src/utils/memoryDiscovery.ts`):

```
GEMINI.md 层级结构 (Hierarchical Memory):

Global Level    →  ~/.gemini/GEMINI.md
                    (用户级全局配置)
                       ↓
Project Level   →  <workspace>/GEMINI.md
                    (项目级上下文)
                       ↓
JIT Level       →  工具访问文件时动态扫描
                    (即时上下文注入)
```

**特点**:
- 自动发现机制
- 支持 Markdown 格式
- 文件变更自动重载
- 与工具执行上下文集成

### 3.5 工具执行流程

**Scheduler** (`/packages/core/src/scheduler/scheduler.ts` lines 1-758):

```typescript
class Scheduler {
  async schedule(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    signal: AbortSignal
  ): Promise<CompletedToolCall[]>
}
```

**执行管道**:
1. **Validation** - 工具参数根据 schema 验证
2. **Policy Check** - PolicyEngine 评估安全规则
3. **Confirmation** - 如果需要，用户审批
4. **Execution** - 工具调用with abort signal
5. **Post-processing** - 格式化结果给 LLM

**Policy Engine** (`/packages/core/src/policy/policy-engine.ts` lines 1-792):

```typescript
class PolicyEngine {
  evaluateToolCall(
    toolCall: FunctionCall,
    approvalMode: ApprovalMode,
    serverName?: string,
    toolAnnotations?: Record<string, unknown>
  ): Promise<PolicyDecision>
}
```

**分层策略系统 (Tiered Policy Engine)**:
```
优先级 (从低到高)     规则来源
    ↓
Tier 1              →  Default (系统默认)
    ↓
Tier 2              →  Extension (扩展定义)
    ↓
Tier 3              →  Workspace (工作空间级)
    ↓
Tier 4              →  User (用户级)
    ↓
Tier 5              →  Admin (管理员级)
    ↓
              【Last Match Wins - 最后匹配的规则生效】
```

**Policy Decisions**:
- `ALLOW` - 立即执行
- `DENY` - 阻止执行
- `ASK_USER` - 请求确认

### 3.6 高级功能

**循环检测 (Loop Detection)**:
- 防止无限循环执行
- 通过 `loopDetector.addAndCheck(event)` 检测模式
- 触发时返回 `GeminiEventType.LoopDetected`

**对话压缩 (Chat Compression)**:
- XML 结构化的摘要机制
- 通过 `tryCompressChat(prompt_id, force)` 触发
- 减少上下文窗口占用
- 保持关键对话历史

---

## 4. 扩展开发指南

### 4.1 配置 MCP Servers

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "-e", "GITHUB_PAT", 
               "ghcr.io/modelcontextprotocol/servers/github:latest"],
      "env": {
        "GITHUB_PAT": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      },
      "timeout": 60000,
      "trust": false
    },
    "remote-service": {
      "httpUrl": "https://api.example.com/mcp",
      "authProviderType": "oauth",
      "oauth": {
        "authorizationUrl": "https://auth.example.com/oauth/authorize",
        "tokenUrl": "https://auth.example.com/oauth/token",
        "clientId": "${CLIENT_ID}",
        "scopes": ["read", "write"]
      }
    }
  }
}
```

### 4.2 添加自定义工具

```typescript
import { BaseDeclarativeTool, BaseToolInvocation } from './tools.js';

export class MyTool extends BaseDeclarativeTool<MyParams, ToolResult> {
  name = 'my_custom_tool';
  kind = Kind.Other;
  
  declaration = {
    name: 'my_custom_tool',
    description: 'Does something custom',
    parameters: {
      type: Type.OBJECT,
      properties: {
        input: { type: Type.STRING }
      },
      required: ['input']
    }
  };
  
  build(params: MyParams): MyToolInvocation {
    return new MyToolInvocation(params);
  }
}
```

---

## 5. 评估

### 优势

1. ✅ **Google 生态**: Search grounding, Code Assist
2. ✅ **企业就绪**: OAuth, tiered policy engine
3. ✅ **丰富工具**: 30+ 内置工具 + MCP (Stdio/SSE/HTTP)
4. ✅ **免费额度**: 1,000 requests/day (Google OAuth)
5. ✅ **智能机制**: Loop detection, chat compression, auto-continue
6. ✅ **分层记忆**: GEMINI.md (Global/Project/JIT 三级)

### 限制

1. ⚠️ **Gemini 绑定**: 优化for Gemini
2. ⚠️ **复杂配置**: 选项众多

### 适用场景

- ✅ Google 生态协同
- ✅ 代码与自动化任务
- ✅ 企业级安全需求

---

## 6. 关键源文件索引

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| **核心 Agent 循环** | `packages/core/src/core/client.ts` | 主循环 (sendMessageStream: 360-450, processTurn: 248-358) |
| **单轮次执行** | `packages/core/src/core/turn.ts` | Turn.run() 实现 |
| **策略引擎** | `packages/core/src/policy/policy-engine.ts` | Tiered policy 系统 (792 lines) |
| **MCP 客户端** | `packages/core/src/tools/mcp-client.ts` | MCP 发现层 (2073 lines) |
| **GEMINI.md** | `packages/core/src/utils/memoryDiscovery.ts` | 分层记忆系统 |
| **调度器** | `packages/core/src/scheduler/scheduler.ts` | 工具执行调度 (758 lines) |
| **循环检测** | `packages/core/src/core/client.ts` | loopDetector.addAndCheck() |
| **对话压缩** | `packages/core/src/core/client.ts` | tryCompressChat() |

---

**文档版本**: 1.1  
**最后更新**: 2026-02-27
