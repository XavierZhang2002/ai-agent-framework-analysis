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

### 3.1 Core Agent Loop (Verified)

**文件**: `packages/core/src/core/client.ts` (Lines 261-384)

**主循环方法**: `sendMessageStream()` - 协调整个对话流转的核心引擎

```typescript
async *sendMessageStream(
  request: PartListUnion,
  signal: AbortSignal,
  prompt_id: string,
  options?: { isContinuation: boolean },
  turns: number = MAX_TURNS,
): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
  
  // 1. 初始化 Loop Detector
  if (!options?.isContinuation) {
    this.loopDetector.reset(prompt_id);
  }
  
  // 2. 检查会话轮次限制
  this.sessionTurnCount++;
  if (this.sessionTurnCount > this.config.getMaxSessionTurns()) {
    yield { type: GeminiEventType.MaxSessionTurns };
    return turn;
  }
  
  // 3. 聊天历史压缩 (Context Management)
  const compressed = await this.tryCompressChat(prompt_id, false);
  
  // 4. 注入 IDE 上下文 (仅 IDE 模式)
  if (this.config.getIdeMode()) {
    const { contextParts } = this.getIdeContextParts(...);
    this.getChat().addHistory({ role: 'user', parts: contextParts });
  }
  
  // 5. 动态系统提醒注入
  const systemReminders = [];
  if (hasTaskTool && subagents.length > 0) {
    systemReminders.push(getSubagentSystemReminder(subagents));
  }
  if (this.config.getApprovalMode() === ApprovalMode.PLAN) {
    systemReminders.push(getPlanModeSystemReminder(...));
  }
  
  // 6. 执行当前 Turn
  const turn = new Turn(this.getChat(), prompt_id);
  const resultStream = turn.run(this.config.getModel(), requestToSent, signal);
  
  // 7. 流式处理响应 + Loop 检测
  for await (const event of resultStream) {
    if (this.loopDetector.addAndCheck(event)) {
      yield { type: GeminiEventType.LoopDetected };
      return turn;
    }
    yield event;
  }
  
  // 8. Next Speaker 检查 (自动继续机制)
  if (!turn.pendingToolCalls.length && signal && !signal.aborted) {
    const nextSpeakerCheck = await checkNextSpeaker(...);
    if (nextSpeakerCheck?.next_speaker === 'model') {
      yield* this.sendMessageStream([{ text: 'Please continue.' }], ...);
    }
  }
  
  return turn;
}
```

**关键设计**:
- **Loop Detection**: 防止重复模式导致的无限循环
- **Context Compression**: 自动压缩聊天历史保持上下文窗口
- **Multi-turn**: 支持最多 100 轮连续对话
- **Next Speaker**: 智能判断是否需要模型继续输出

### 3.2 Skills 系统 (Verified)

**文件**: `packages/core/src/skills/skill-manager.ts` (661 lines)

#### 文件结构
- **位置**: `.qwen/skills/<skill-name>/SKILL.md`
- **格式**: Markdown + YAML Frontmatter
- **热重载**: 基于 `chokidar` 的文件系统监听

**Skill 格式示例**:
```yaml
---
name: terminal-capture
description: Automates terminal UI screenshot testing
---

# Instructions...
```

#### 三级缓存架构

```typescript
// Lines 82-130: Skill 发现with缓存
async listSkills(options: ListSkillsOptions = {}): Promise<SkillConfig[]> {
  const levelsToCheck = options.level 
    ? [options.level] 
    : ['project', 'user', 'extension'];
  
  // 优先级: project > user > extension
  // project: 当前项目 .qwen/skills/
  // user:    ~/.qwen/skills/
  // extension: VS Code 扩展自带
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

#### 热重载机制

```typescript
// 使用 chokidar 监听文件变化
private watchSkillsDirectory(level: SkillLevel, skillsDir: string): void {
  const watcher = chokidar.watch(skillsDir, {
    persistent: true,
    ignoreInitial: true,
    depth: 2,  // 只监听 SKILL.md 文件
  });
  
  watcher.on('change', async (filePath) => {
    if (filePath.endsWith('SKILL.md')) {
      await this.reloadSkill(filePath, level);
    }
  });
}
```

**关键设计**:
- **三级覆盖**: project 配置覆盖 user，user 覆盖 extension
- **热重载**: 开发时无需重启即可更新 Skill
- **YAML 元数据**: 支持 name, description, triggers 等配置

### 3.3 SubAgents 实现 (Verified)

**文件**: `packages/core/src/subagents/subagent.ts` (1004 lines)

#### Context Variable Templating

**模板函数** (lines 29-98):
```typescript
function templateString(template: string, context: ContextState): string {
  const placeholderRegex = /\$\{(\w+)\}/g;
  return template.replace(placeholderRegex, (_match, key) =
    String(context.get(key)),
  );
}

export class ContextState {
  private variables = new Map<string, string>();
  
  // 内置变量
  this.variables.set('project_name', this.getProjectName());
  this.variables.set('current_directory', process.cwd());
  this.variables.set('timestamp', new Date().toISOString());
  
  // 模板解析: ${variable_name}
  interpolate(template: string): string {
    return template.replace(/\$\{(\w+)\}/g, (_, key) =
      this.variables.get(key) || '';
    );
  }
}
```

#### SubAgent 执行循环

**非交互模式** (`runNonInteractive`):
```typescript
async runNonInteractive(context: ContextState): Promise<void> {
  while (true) {
    // 1. 检查终止条件
    if (turnCounter >= max_turns) break;
    if (time_exceeded) break;
    
    // 2. 调用 LLM
    const responseStream = await chat.sendMessageStream(model, messages, promptId);
    
    // 3. 流式处理响应
    for await (const streamEvent of responseStream) {
      if (streamEvent.type === 'chunk') {
        if (resp.functionCalls) functionCalls.push(...resp.functionCalls);
      }
    }
    
    // 4. 处理工具调用或终止
    if (functionCalls.length > 0) {
      currentMessages = await this.processFunctionCalls(functionCalls);
    } else {
      // 无工具调用 - 生成最终答案
      this.finalText = roundText.trim();
      break;
    }
  }
}
```

#### SubAgentScope 主执行类

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

**关键设计**:
- **隔离执行**: 每个 SubAgent 有自己的工具注册表
- **变量模板**: `${project_name}`, `${current_directory}` 等上下文变量
- **流式处理**: 支持 chunked response 处理
- **终止条件**: max_turns + 超时双重保护

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

## 4. 关键源文件索引

| 文件路径 | 职责 | 代码行数 |
|---------|------|---------|
| `packages/core/src/core/client.ts` | 主 Agent 循环 (`sendMessageStream`) | 678 lines |
| `packages/core/src/subagents/subagent.ts` | SubAgent 执行引擎 + 变量模板 | 1004 lines |
| `packages/core/src/subagents/subagent-manager.ts` | SubAgent CRUD 管理 | ~300 lines |
| `packages/core/src/skills/skill-manager.ts` | Skills 加载 + 热重载 + 三级缓存 | 661 lines |
| `packages/core/src/tools/task.ts` | TaskTool - 子代理委托入口 | ~200 lines |
| `packages/core/src/ide/ide-client.ts` | IDE MCP 集成 + Diff View | 860 lines |
| `packages/core/src/qwen/qwenOAuth2.ts` | Qwen OAuth Device Flow | 1017 lines |
| `packages/core/src/config/` | 配置管理 (1735 lines) | 多文件 |

---

## 5. 扩展开发指南

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

## 6. 评估

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

**文档版本**: 1.1  
**最后更新**: 2026-02-27

### 更新记录
- v1.1: 添加 `sendMessageStream()` 核心循环实现，SubAgent `runNonInteractive()` 详细逻辑，Skills 热重载机制，关键源文件索引
- v1.0: 初始版本
