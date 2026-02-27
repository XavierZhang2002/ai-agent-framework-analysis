# Qwen Code - 深度技术调研

**调研日期**: 2026-02-26  
**仓库**: https://github.com/QwenLM/qwen-code  
**许可证**: Apache-2.0  
**主要语言**: TypeScript  
**定位**: 终端优先 + Skills/SubAgents，围绕 Qwen Coder 协同演进  

---

## 1. 项目概述

Qwen Code 是阿里云通义千问团队的终端 AI coding assistant，特点：

- ✅ **Skills 系统**: Markdown-based 专业知识包
- ✅ **SubAgents**: 专门化子代理（general-purpose 等）
- ✅ **IDE 深度集成**: VS Code, Zed, JetBrains via MCP
- ✅ **Qwen OAuth**: Device flow with PKCE
- ✅ **Variable 模板**: `${project_name}`, `${current_directory}` 等
- ✅ **Extension 框架**: 支持 Claude, Gemini, native Qwen extensions

---

## 2. 代码架构

### 目录结构

```
qwen-code/
├── packages/
│   ├── cli/          # CLI (React+Ink)
│   ├── core/         # Backend engine
│   │   └── src/
│   │       ├── core/             # Client orchestration
│   │       ├── tools/            # 30+ 工具
│   │       ├── subagents/        # SubAgent 系统 (1004 lines in subagent.ts)
│   │       ├── skills/           # Skills 管理 (661 lines in skill-manager.ts)
│   │       ├── qwen/             # Qwen OAuth (1017 lines in qwenOAuth2.ts)
│   │       ├── ide/              # IDE 集成 (860 lines in ide-client.ts)
│   │       └── config/           # 配置 (1735 lines)
│   ├── sdk-typescript/
│   ├── sdk-java/
│   └── vscode-ide-companion/
└── .qwen/skills/
```

---

## 3. 底层实现

### 3.1 Agentic Workflow

**文件**: `packages/core/src/core/client.ts` (678 lines)

```typescript
class GeminiClient {
  async chat(userMessage: string) {
    // 1. 添加 IDE 上下文
    const ideContext = this.getIdeContextParts(forceFullContext);
    
    // 2. 构建系统提示with tools
    const systemInstruction = getCoreSystemPrompt(userMemory, model);
    
    // 3. 发送到模型with tools
    const response = await this.geminiChat.sendMessage(...);
    
    // 4. 执行工具调用（如果有）
    await this.coreToolScheduler.executeToolCalls(toolCalls);
    
    // 5. 循环直到模型停止请求工具 (MAX_TURNS = 100)
  }
}
```

### 3.2 Skills 系统

**实现** (`packages/core/src/skills/skill-manager.ts`, 661 lines):

```typescript
// Lines 82-130: Skill 发现with缓存
async listSkills(options: ListSkillsOptions = {}): Promise<SkillConfig[]> {
  const levelsToCheck = options.level 
    ? [options.level] 
    : ['project', 'user', 'extension'];
  
  // 优先级: project > user > extension
  for (const level of levelsToCheck) {
    const levelSkills = this.skillsCache?.get(level) || [];
    for (const skill of levelSkills) {
      if (!seenNames.has(skill.name)) {
        skills.push(skill);
        seenNames.add(skill.name);
      }
    }
  }
}
```

**Skill 格式** (`.qwen/skills/*/SKILL.md`):
```yaml
---
name: terminal-capture
description: Automates terminal UI screenshot testing
---

# Instructions...
```

### 3.3 SubAgents 实现

**文件**: `packages/core/src/subagents/subagent.ts` (1004 lines)

**Context State with variable templating** (lines 29-98):
```typescript
export class ContextState {
  private variables = new Map<string, string>();
  
  // 内置变量
  this.variables.set('project_name', this.getProjectName());
  this.variables.set('current_directory', process.cwd());
  this.variables.set('timestamp', new Date().toISOString());
  
  // 模板解析: ${variable_name}
  interpolate(template: string): string {
    return template.replace(/\$\{(\w+)\}/g, (_, key) => {
      return this.variables.get(key) || '';
    });
  }
}
```

**SubAgentScope** - 主执行类 (lines 183-272):
```typescript
export class SubAgentScope {
  async run(abortSignal: AbortSignal): Promise<SubAgentResult> {
    // 1. 初始化 context
    const contextState = new ContextState(this.config);
    
    // 2. 创建隔离的工具注册表
    const toolRegistry = this.createToolRegistry();
    
    // 3. 应用系统提示with variable 插值
    const systemPrompt = contextState.interpolate(
      this.promptConfig.systemPrompt
    );
    
    // 4. 使用超时 & max turns 约束执行
    for (let turn = 0; turn < maxTurns; turn++) {
      const response = await this.chat.sendMessage(...);
      
      if (this.shouldTerminate(response)) {
        return { status: 'GOAL', output: response };
      }
    }
  }
}
```

### 3.4 Qwen OAuth

**文件**: `packages/core/src/qwen/qwenOAuth2.ts` (1017 lines)

**Device Flow** (lines 250-350):
```typescript
async requestDeviceAuthorization(options): Promise<DeviceAuthorizationResponse> {
  const response = await fetch(QWEN_OAUTH_DEVICE_CODE_ENDPOINT, {
    method: 'POST',
    body: objectToUrlEncoded({
      client_id: QWEN_OAUTH_CLIENT_ID,
      scope: options.scope,
      code_challenge: options.code_challenge,
      code_challenge_method: options.code_challenge_method,
    }),
  });
  return response.json();
}
```

**Token 管理** (`qwen/sharedTokenManager.ts`, 27KB):
- 多进程 token 共享with file locking
- 原子 token 刷新操作

### 3.5 IDE 集成

**文件**: `packages/core/src/ide/ide-client.ts` (860 lines)

**连接管理** (lines 77-206):
```typescript
export class IdeClient {
  async connect(): Promise<void> {
    // 1. 检测 IDE (VS Code, Zed, JetBrains)
    this.currentIde = detectIde(this.ideProcessInfo, ...);
    
    // 2. 验证 workspace 路径
    const { isValid, error } = IdeClient.validateWorkspacePath(...);
    
    // 3. 尝试连接方法（优先级顺序）
    // a. HTTP 连接
    if (this.connectionConfig?.port) {
      await this.establishHttpConnection(this.connectionConfig.port);
    }
    
    // b. Stdio 连接
    if (this.connectionConfig?.stdio) {
      await this.establishStdioConnection(this.connectionConfig.stdio);
    }
  }
}
```

**Diff View with Mutex** (lines 226-280):
```typescript
async openDiff(filePath: string, newContent: string): Promise<DiffUpdateResult> {
  const release = await this.acquireMutex();  // 一次只有一个 diff
  
  try {
    return await new Promise<DiffUpdateResult>((resolve) => {
      this.diffResponses.set(filePath, resolve);
      
      this.client.request({
        method: 'tools/call',
        params: {
          name: 'openDiff',
          arguments: { filePath, newContent },
        },
      }, CallToolResultSchema, { timeout: IDE_REQUEST_TIMEOUT_MS });
    });
  } finally {
    release();
  }
}
```

---

## 4. 扩展开发指南

### 4.1 添加 Skills

```yaml
# .qwen/skills/my-skill/SKILL.md
---
name: my-skill
description: Brief description with trigger keywords
---

# Instructions
```

### 4.2 创建 SubAgents

```yaml
# .qwen/agents/my-agent.md
---
name: my-agent
description: When to use this agent
tools:
  - read_file
  - bash
modelConfig:
  model: qwen3-coder-plus
runConfig:
  max_time_minutes: 15
---

System prompt with ${variable} templating...
```

---

## 5. 评估

### 优势

1. ✅ **Qwen 生态集成**: OAuth, API, models
2. ✅ **Skills + SubAgents**: 强大的模块化
3. ✅ **IDE 集成**: 深度 MCP 集成

### 限制

1. ⚠️ **Qwen 倾向**: 优化for Qwen生态
2. ⚠️ **中文文档**: 部分文档中文

### 适用场景

- ✅ Qwen 生态集成
- ✅ 代码理解与改造
- ✅ 终端自动化

---

**文档版本**: 1.0  
**最后更新**: 2026-02-26
