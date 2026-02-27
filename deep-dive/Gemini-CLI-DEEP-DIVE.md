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

### 3.1 Terminal Agent 架构

```typescript
// packages/cli/src/gemini.tsx (lines 1-906)
1. initializeApp() - 认证 & 设置
2. render(<AppContainer />) - 启动 React UI
3. GeminiClient.sendMessageStream() - 流式 LLM 响应
4. Scheduler.schedule() - 并行执行工具
5. Turn management - 跟踪对话状态
```

**GeminiClient** (`/packages/core/src/core/client.ts` lines 82-1120):
```typescript
class GeminiClient {
  async *sendMessageStream(
    request: PartListUnion,
    tools: Tool[],
    signal: AbortSignal,
    routingContext?: RoutingContext
  ): AsyncGenerator<ServerGeminiStreamEvent>
}
```

### 3.2 MCP 集成

**发现层** (`/packages/core/src/tools/mcp-client.ts` - 2073 lines):

```typescript
class McpClient {
  async connect(): Promise<void>
  async discover(cliConfig: Config): Promise<void>
  async discoverTools(): Promise<void>
  async discoverResources(): Promise<void>
  async discoverPrompts(): Promise<void>
}
```

**Transport 层** (lines 158-195):
- **Stdio**: 生成子进程
- **SSE**: Server-Sent Events
- **Streamable HTTP**: HTTP streaming

**认证** (`/packages/core/src/mcp/`):
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

### 3.4 工具执行流程

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

**Policy Decisions**:
- `ALLOW` - 立即执行
- `DENY` - 阻止执行
- `ASK_USER` - 请求确认

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
2. ✅ **企业就绪**: OAuth, policy engine
3. ✅ **丰富工具**: 30+ 内置工具 + MCP

### 限制

1. ⚠️ **Gemini 绑定**: 优化for Gemini
2. ⚠️ **复杂配置**: 选项众多

### 适用场景

- ✅ Google 生态协同
- ✅ 代码与自动化任务
- ✅ 企业级安全需求

---

**文档版本**: 1.0  
**最后更新**: 2026-02-26
