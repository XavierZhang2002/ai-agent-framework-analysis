# Agent 开源与编排对比调研（截至 2026-02-26）

**基于深度代码分析的全面对比报告**

> **调研方法**：  
> 本报告基于对 9 个主流 Agent 框架的深度代码分析，包括源码审计、架构分析、扩展点研究。  
> 每个项目的详细技术调研文档见 `agent_deep_dive/*-DEEP-DIVE.md`。
> 
> **判定口径**：  
> 1) "仓库开源" = 代码仓库公开 + 有开源许可证；  
> 2) "核心逻辑可看可改" = Agent 编排/工具调用主逻辑是否在仓库可审计并可改；  
> 3) "黑盒" 主要指运行时依赖的闭源服务（模型推理、托管工具、商业条款约束组件）。

---

## 1) 总览表（按 Stars 排序）

| 项目 | Stars | 仓库/许可证 | 核心逻辑透明度 | 主要语言 | Python SDK | 代码规模 | 设计动机 | 适用场景 | 主要黑盒点 |
|---|---|---|---|---|---|---|---|---|---|
| **OpenAI Agents SDK** | 19.2k | `openai/openai-agents-python` / MIT | ✅ ✅ ✅ 完全开源 | Python | ✅ 核心定位 | ~15k lines | Swarm 生产演进版，轻量级多 Agent 编排，provider-agnostic | Python 应用多 Agent 工作流、生产级编排、跨 LLM | 所选模型服务（100+ LLMs） |
| **OpenCode** | 未公开 | `anomalyco/opencode` / MIT | ✅ ✅ ✅ 100% 开源 | TypeScript (Bun) | ❌ CLI/TUI 为主 | ~47k lines | Provider-agnostic 终端编码 Agent，强 TUI 与开放生态 | 日常本地编码、跨模型、终端重度用户 | 所选模型服务 |
| **Claude Agent SDK** | 5k | `anthropics/claude-agent-sdk-python` / MIT + ToS | ⚠️ SDK 开源，CLI 闭源 | Python | ✅ 核心定位 | SDK ~3k lines + 闭源 CLI | 以 Python 编排 Claude Code 能力，强 Hooks | Python 应用接入 Claude、工具与 hooks | Claude Code CLI（bundled）+ Anthropic ToS |
| **Codex CLI** | 未公开 | `openai/codex` / Apache-2.0 | ✅ ✅ 开源 | Rust + TypeScript | ❌ 有 TS SDK | Core ~10k lines (codex.rs) | 本地优先终端 agent，审批/沙箱/MCP 双角色 | 本地工程、受控执行、可审计工作流 | OpenAI 模型服务（主） |
| **Kimi CLI** | 未公开 | `MoonshotAI/kimi-cli` / Apache-2.0 | ✅ ✅ 开源 | Python + TypeScript | ⚠️ CLI 可 pip 安装 | 未统计 | 终端 + shell 一体化，强 IDE/ACP 接入 | 终端开发、IDE Agent Server、命令行增强 | Kimi/外部模型服务 |
| **Gemini CLI** | 未公开 | `google-gemini/gemini-cli` / Apache-2.0 | ✅ ✅ 开源 | TypeScript | ❌ Node CLI | Core ~866 files | Gemini 能力终端化，工具与 MCP 扩展 | 代码与自动化、Google 生态协同 | Gemini/Google 云侧能力 |
| **Qwen Code** | 未公开 | `QwenLM/qwen-code` / Apache-2.0 | ✅ ✅ 开源 | TypeScript | ❌ Node CLI | 未统计 | 终端优先 + Skills/SubAgents，Qwen 协同演进 | 代码理解与改造、终端自动化、Qwen 集成 | Qwen OAuth/API 服务 |
| **SWE-agent** | 未公开 | `SWE-agent/SWE-agent` / MIT | ✅ ✅ ✅ 研究导向 | Python | ❌ 框架本体 | Agent ~1.3k lines | 自动修 issue/基准评测的研究型 Agent | SWE-bench、自动修复、研究 | 所接入 LLM 服务 |
| **OpenManus** | 未公开 | `FoundationAgents/OpenManus` / MIT | ✅ ✅ Python 开源 | Python | ❌ 可运行框架 | 未统计 | 快速复现 Manus 风格 Agent，社区共建 | 通用任务 Agent、MCP 实验、多 Agent 研究 | 所连接 LLM/API 服务 |
| **Aider** | 40.3k | `Aider-AI/aider` / Apache-2.0 | ✅ ✅ ✅ 完全开源 | Python | ❌ CLI 工具 | ~8k lines | Git-native 编码助手，首创 Repo Map | 大型代码库、多文件编辑、Git 工作流 | 所选模型服务 |
| **Goose** | 29.9k | `block/goose` / Apache-2.0 | ✅ ✅ ✅ 完全开源 | Rust | ❌ CLI 工具 | Core ~3k lines | MCP-Native 架构，Block 企业级 | MCP 生态、企业部署、工具多样化 | 所选模型服务 |
| **OpenHands** | 67.5k | `All-Hands-AI/OpenHands` / MIT | ✅ ✅ ✅ 完全开源 | Python | ❌ 平台化 | ~15k lines | 完整 Agent 平台，Docker 沙箱 | 学术研究、团队协作、SWE-bench | 所选模型服务 |

**注**: 
- **"Big Four" 第一梯队**（按 Stars）：OpenHands (67.5k) > Aider (40.3k) > Goose (29.9k) > OpenAI Agents SDK (19.2k)
- 市场格局显示开源社区项目（Aider、Goose、OpenHands）的 Stars 超越多数实验室项目

---

## 2) 技术架构深度对比

### 2.1 核心编排实现对比

| 项目 | 核心编排文件 | 编排模式 | 关键实现细节 |
|------|------------|---------|------------|
| **OpenAI Agents SDK** | `src/agents/run.py:396-1329` (1623 lines) | Agent loop with NextStep state machine | • RunState 可序列化 HITL<br>• Handoff 作为 first-class primitive<br>• Parallel guardrails |
| **Claude Agent SDK** | `_internal/query.py` (634 lines) | Control protocol over subprocess | • Bidirectional JSON-RPC<br>• Hook callbacks at 10+ events<br>• SDK MCP in-process routing |
| **OpenCode** | `src/session/prompt.ts:274-724` | Event-driven loop with SSE streaming | • Instance-scoped state<br>• Permission ruleset engine<br>• Subtask 同步执行 |
| **Codex CLI** | `codex-rs/core/src/codex.rs` (9,712 lines) | Rust event stream architecture | • Arc-based shared state<br>• Thread-safe message passing<br>• Sandbox abstraction layer |
| **Kimi CLI** | `src/kimi_cli/soul/kimisoul.py` | KimiSoul main loop | • Wire protocol JSON-RPC<br>• Multi-process token coordination<br>• ACP server multi-session |
| **Gemini CLI** | `packages/core/src/core/client.ts` (1120 lines) | GeminiClient streaming loop | • React/Ink UI<br>• Tool scheduler (758 lines)<br>• Policy engine (792 lines) |
| **Qwen Code** | `packages/core/src/core/client.ts` (678 lines) | GeminiClient with IDE context | • Context variable templating<br>• SubAgent isolation<br>• File-lock token manager |
| **SWE-agent** | `sweagent/agent/agents.py:1265` | DefaultAgent.run() | • YAML-driven config<br>• Trajectory recording<br>• Tool bundle system |
| **OpenManus** | `app/agent/base.py:116-154` | BaseAgent.run() template method | • ReAct inheritance chain<br>• ToolCollection composition<br>• Singleton LLM |
| **Aider** | `aider/coders/base_coder.py:800+` | Coder.step() | • Git-native workflow<br>• Repo Map (tree-sitter)<br>• Multi-file editing |
| **Goose** | `crates/goose-core/src/agent.rs:800+` | Agent loop with MCP routing | • MCP-Native architecture<br>• Process isolation<br>• Rust performance |
| **OpenHands** | `openhands/controller/agent_controller.py:900+` | AgentController.step() | • Micro-agent orchestration<br>• Docker sandbox<br>• Web UI + API |

### 2.2 工具/协议扩展能力对比

| 项目 | MCP 支持 | 工具定义方式 | 扩展机制 | 工具数量 |
|------|---------|------------|---------|---------|
| **OpenAI Agents SDK** | ⚠️ 可通过 tools 接入外部 MCP | `@function_tool` decorator | Protocol-based (Session/Model/Tracing) | 无内置（需自定义） |
| **Claude Agent SDK** | ✅ ✅ ✅ Native in-process SDK MCP | `@tool` decorator + `create_sdk_mcp_server()` | Custom Transport, Hooks (10+ events) | 依赖 Claude Code CLI 内置工具 |
| **OpenCode** | ✅ MCP client（连接外部服务器） | `Tool.define()` + 文件型工具 (`.opencode/tool/*.ts`) | Plugin system, custom agents via config | 39 个核心模块 |
| **Codex CLI** | ✅ ✅ ✅ MCP Client + Server（双角色） | Rust `ToolHandler` trait + MCP servers | Custom tool handlers (Rust), MCP servers | 30+ 内置 handlers |
| **Kimi CLI** | ✅ MCP client（via fastmcp） | `CallableTool2` + import paths | Agent spec (YAML), custom tools, skills | 15+ 内置工具 |
| **Gemini CLI** | ✅ ✅ MCP (Stdio/SSE/HTTP) + OAuth | `BaseDeclarativeTool` + `ToolInvocation` | Extension framework, MCP servers | 30+ 内置工具 |
| **Qwen Code** | ✅ ✅ MCP + IDE integration | `BaseDeclarativeTool` + dynamic schema | Extensions (agents/skills/tools), MCP | 30+ 内置工具 |
| **SWE-agent** | ❌ 无 MCP | Tool bundles (YAML + shell scripts in `tools/`) | Custom agents, tool bundles, parsers | 15+ tool bundles |
| **OpenManus** | ✅ ✅ MCP Client + Server（双角色） | `BaseTool` + Pydantic | Subclass BaseTool, MCP servers, Flow system | 8 个内置工具 |

### 2.3 权限与安全控制对比

| 项目 | 安全机制 | 配置方式 | 细粒度 | 沙箱 |
|------|---------|---------|-------|-----|
| **OpenAI Agents SDK** | Guardrails (input/output/tool) | Python decorators (`@input_guardrail`) | ✅ ✅ 函数级 | ❌ |
| **Claude Agent SDK** | Hooks (10+ events) + `permission_mode` | Python callbacks, `allowed_tools` list | ✅ ✅ ✅ Hook 事件级 | ❌ |
| **OpenCode** | Permission ruleset engine | JSON config + wildcards | ✅ ✅ ✅ Pattern-based | ❌ |
| **Codex CLI** | Approval policy + Sandbox + Network control | `.rules` files + config.toml | ✅ ✅ ✅ ✅ Command prefix + network | ✅ ✅ ✅ Platform-specific |
| **Kimi CLI** | Approval system + yolo mode | Session state, agent spec | ✅ ✅ Action-level | ❌ |
| **Gemini CLI** | Policy engine + approval modes | TOML policies + trusted folders | ✅ ✅ ✅ Policy rules | ❌ |
| **Qwen Code** | Approval mode + tool config | JSON settings, allowed/exclude tools | ✅ ✅ Tool-level | ⚠️ 可配置 Docker sandbox |
| **SWE-agent** | Command blocklists | YAML config, tool bundles | ⚠️ 基础 | ✅ Docker/Modal 容器 |
| **OpenManus** | 运行环境约束 | 配置文件 | ⚠️ 基础 | ❌ |

### 2.4 Multi-Agent / 子代理能力对比

| 项目 | 实现方式 | 关键特性 | 代码位置 |
|------|---------|---------|---------|
| **OpenAI Agents SDK** | Handoff 类（first-class） | • Input filter<br>• History nesting<br>• 自动 tool schema 生成 | `src/agents/handoffs/__init__.py:93-321` |
| **Claude Agent SDK** | Subagents（文档化） + Hooks | • Session forking<br>• SubagentStart/Stop hooks | 通过 Hooks + Claude Code CLI |
| **OpenCode** | Task 工具 + Agent profiles | • Parent/child sessions<br>• Permission 继承<br>• Model 继承 | `src/tool/task.ts:45-163` |
| **Codex CLI** | Multi-agent tool handler | • Thread hierarchy<br>• Approval 链 | `tools/handlers/multi_agents.rs` |
| **Kimi CLI** | Task + CreateSubagent 工具 | • Dynamic subagent creation<br>• Event emitter communication<br>• Labor market（subagent pool） | `tools/multiagent/task.py`<br>`soul/agent.py:183-207` |
| **Gemini CLI** | Agent registry + remote agents | • Local/remote agents<br>• A2A protocol | `agents/agent-scheduler.ts`<br>`a2a-server/` package |
| **Qwen Code** | SubAgent system + Skills | • Context variable templating<br>• Precedence: session>project>user<br>• ContextState isolation | `subagents/subagent.ts:183-272`<br>`subagents/subagent-manager.ts:147-195` |
| **SWE-agent** | 单 agent 为主 | • RetryAgent（multi-attempt）<br>• Reviewer/Chooser 机制 | `sweagent/agent/agents.py:257-441` |
| **OpenManus** | PlanningFlow（实验性） | • Agent 选择器<br>• LLM-driven planning | `app/flow/planning.py:77-92` |

### 2.5 Session/会话管理对比

| 项目 | Session 支持 | 存储后端 | 持久化格式 | 压缩/Compaction |
|------|------------|---------|-----------|---------------|
| **OpenAI Agents SDK** | ✅ ✅ ✅ 内置 | SQLite, Redis, 自定义（Protocol） | JSON (SQLite) / List (Redis) | ❌ 无（需自实现） |
| **Claude Agent SDK** | ❌ 需手动管理 | 无 | 无 | ❌ |
| **OpenCode** | ✅ ✅ 内置 | SQLite (Drizzle ORM) | MessageV2 + Parts 结构 | ✅ SessionCompaction |
| **Codex CLI** | ✅ ✅ Rollout files | JSONL files + SQLite metadata | JSONL (每行一个事件) | ⚠️ 未明确 |
| **Kimi CLI** | ✅ ✅ 内置 | JSONL files + state.json | `context.jsonl` + `wire.jsonl` | ✅ Compaction service |
| **Gemini CLI** | ✅ ChatRecordingService | JSON files | Conversation history JSON | ⚠️ 未明确 |
| **Qwen Code** | ✅ Session system | 未详细说明 | 未详细说明 | ⚠️ 未明确 |
| **SWE-agent** | ✅ Trajectory files | JSON/JSONL | Trajectory steps | ❌ |
| **OpenManus** | ❌ Memory in-memory | In-memory list | 无持久化 | ❌ |

### 2.6 Tracing/可观测性对比

| 项目 | Tracing 支持 | 实现方式 | 外部集成 | Span 类型 |
|------|------------|---------|---------|---------|
| **OpenAI Agents SDK** | ✅ ✅ ✅ 内置 | TracingProcessor Protocol | Logfire, AgentOps, Braintrust, Scorecard, Keywords AI | Agent, Generation, Function, Handoff, Guardrail, Custom |
| **Claude Agent SDK** | ❌ 无内置 | 需自实现 | 无 | 无 |
| **OpenCode** | ⚠️ OpenTelemetry（实验） | 实验性 OTEL 集成 | OTEL endpoints | 未详细说明 |
| **Codex CLI** | ⚠️ Telemetry hooks | Hook system | 自定义 hooks | 未详细说明 |
| **Kimi CLI** | ⚠️ 基础日志 | Structlog | 无 | 无 |
| **Gemini CLI** | ⚠️ Telemetry service | 内置 telemetry | Google telemetry, OTLP | 未详细说明 |
| **Qwen Code** | ⚠️ 基础日志 | 日志系统 | 无 | 无 |
| **SWE-agent** | ❌ 无（仅 trajectory） | Trajectory recording | 无 | 无 |
| **OpenManus** | ❌ 无 | 仅日志 | 无 | 无 |

---

## 3) 深度技术对比

### 3.1 架构风格分类

#### **A) 纯 SDK 层（Python 应用嵌入）**

**OpenAI Agents SDK**:
- **架构**: 纯 Python SDK，无 CLI 依赖
- **核心抽象**: Agent, Handoff, Session, Guardrail, Tracing
- **执行模式**: `Runner.run()` → AgentRunner → run_internal/*
- **状态管理**: RunState (2384 lines) 支持 HITL
- **扩展性**: Protocol-based (Session/Model/TracingProcessor)
- **代码规模**: ~15,000 lines core + 12,775 lines examples
- **适用**: Python 应用中嵌入多 agent 工作流

**Claude Agent SDK**:
- **架构**: Python SDK + 闭源 Claude Code CLI（bundled）
- **核心抽象**: query(), ClaudeSDKClient, Hooks, SDK MCP Server
- **执行模式**: SDK → Subprocess → Claude Code CLI → Anthropic API
- **状态管理**: 无（需手动管理历史）
- **扩展性**: Hooks (10+ events), Custom Transport, SDK MCP Servers
- **代码规模**: SDK ~3,000 lines + 闭源 CLI
- **适用**: Python 应用深度集成 Claude 能力

**对比总结**:
- OpenAI Agents SDK: 完全开源，provider-agnostic，更轻量
- Claude Agent SDK: 强大的 Hooks，in-process MCP，但依赖闭源 CLI

#### **B) 终端 CLI 型（终端重度用户）**

**OpenCode** (TypeScript/Bun):
- **架构**: 事件驱动 + Instance-scoped state
- **核心模块**: 39 个顶级模块，~47k lines
- **Agent 系统**: Profile-based (build/plan/general/explore)
- **权限引擎**: Wildcard pattern matching, Last match wins
- **TUI**: Solid.js + Tauri（桌面应用）
- **特色**: 100% 开源，强大的 permission system

**Codex CLI** (Rust + TypeScript):
- **架构**: Rust 核心 (9,712 lines in codex.rs) + TS SDK
- **安全**: 三层（Platform Sandbox + Approval Policy + Network Proxy）
- **MCP**: 双角色（Client 接入工具 + Server 暴露 codex_tool_call）
- **配置**: 100+ config fields, profile system, constraints
- **特色**: 企业级安全，高性能 Rust 核心

**Kimi CLI** (Python + TypeScript):
- **架构**: Python backend + React web UI
- **ACP**: 完整 Agent Client Protocol 服务器
- **Wire Protocol**: JSON-RPC for IDE integration
- **Skills**: Markdown + Flow workflows (Mermaid/D2)
- **特色**: IDE 深度集成，多 UI 模式（shell/web/ACP）

**Gemini CLI** (TypeScript):
- **架构**: Monorepo (CLI + Core + SDK + VS Code ext)
- **工具**: 30+ 内置工具 + MCP integration
- **Policy**: TOML-based security policies
- **Google 生态**: OAuth2, Code Assist, Search grounding
- **特色**: React/Ink UI，Google 服务深度集成

**Qwen Code** (TypeScript):
- **架构**: Monorepo with SDK (TS + Java)
- **SubAgents**: Context variable templating, precedence system
- **Skills**: Auto-discovery with file watching
- **IDE**: Deep MCP integration (VS Code, Zed, JetBrains)
- **特色**: Qwen OAuth with PKCE，variable templating

**对比总结**:
- **最强安全**: Codex CLI（Sandbox + Approval + Network）
- **最开放**: OpenCode（100% 开源 + provider-agnostic）
- **最强 IDE**: Kimi CLI（ACP + Wire）+ Qwen Code（MCP IDE）
- **最佳 Google**: Gemini CLI（Search grounding + Code Assist）

#### **C) 研究/实验框架**

**SWE-agent** (Python):
- **架构**: YAML-driven configuration
- **核心**: DefaultAgent (1294 lines), RetryAgent
- **工具**: Bundle system (YAML + shell scripts)
- **评测**: SWE-Bench 集成，批量执行
- **Trajectory**: 完整执行轨迹记录
- **适用**: 研究、benchmark、自动修复

**OpenManus** (Python):
- **架构**: ReAct 继承链（BaseAgent → ReActAgent → ToolCallAgent → Manus）
- **Flow**: PlanningFlow (442 lines, 标注为不稳定)
- **MCP**: 双角色（Client + Server via FastMCP）
- **工具**: 8 个内置工具 + MCP tools
- **适用**: 快速实验、MCP 探索、社区共建

**对比总结**:
- **SWE-agent**: 专注 benchmark，YAML 配置化，trajectory 分析
- **OpenManus**: 通用实验框架，MCP 双角色，Flow 系统不稳定

---

## 4) 关键差异总结（基于代码审计）

### 4.1 开源透明度分级

**A 级（完全开源，核心逻辑可审计）**:
1. **OpenAI Agents SDK**: MIT，完整 run loop 实现可见（`run.py:396-1329`）
2. **OpenCode**: MIT，所有 39 个模块开源（~47k lines）
3. **SWE-agent**: MIT，研究导向，完全透明
4. **OpenManus**: MIT，Python 主体开源
5. **Qwen Code**: Apache-2.0，TypeScript 开源
6. **Gemini CLI**: Apache-2.0，TypeScript 开源（866 files）
7. **Codex CLI**: Apache-2.0，Rust core 开源（9,712 lines in codex.rs）
8. **Kimi CLI**: Apache-2.0，Python + TS 开源

**B 级（SDK 开源但执行链路含闭源组件）**:
9. **Claude Agent SDK**: MIT + Anthropic ToS，SDK 开源但依赖闭源 Claude Code CLI

### 4.2 Python 生态可嵌入性

**Tier 1: 纯 Python SDK 产品**
- **OpenAI Agents SDK**: 生产就绪，`pip install openai-agents`
- **Claude Agent SDK**: 生产就绪，`pip install claude-agent-sdk`

**Tier 2: Python 主体框架**
- **OpenManus**: 可运行框架，ReAct 继承链
- **SWE-agent**: 研究框架，YAML 驱动
- **Kimi CLI**: CLI 本体，`pip install kimi-cli`

**Tier 3: 非 Python（但有 SDK）**
- **Codex CLI**: Rust CLI + TypeScript SDK
- **Gemini CLI**: TypeScript CLI（Node.js）
- **Qwen Code**: TypeScript CLI + TS/Java SDK
- **OpenCode**: TypeScript CLI（Bun）

### 4.3 Provider-agnostic（跨模型）能力分级

**Level 1: 明确支持多供应商**
- **OpenAI Agents SDK**: ✅ ✅ ✅ 100+ LLMs via LiteLLM，无绑定
- **OpenCode**: ✅ ✅ ✅ Provider-agnostic 设计，Vercel AI SDK

**Level 2: 可配置但倾向特定供应商**
- **SWE-agent**: ✅ ✅ LiteLLM 集成，完全可配（研究用）
- **OpenManus**: ✅ ✅ LiteLLM 集成，可配置
- **Kimi CLI**: ✅ kosong 框架支持多供应商，Kimi 为主
- **Codex CLI**: ✅ 支持多 LLMs，OpenAI 为主
- **Gemini CLI**: ✅ 支持 OpenAI/Anthropic，Gemini 为主
- **Qwen Code**: ✅ 支持 OpenAI/Anthropic/Gemini，Qwen 优化

**Level 3: 严格绑定供应商**
- **Claude Agent SDK**: ❌ Claude-only，无法使用其他 LLMs

### 4.4 架构成熟度与生产就绪度

**生产就绪（Production-Ready）**:
1. **OpenAI Agents SDK**: ✅ ✅ ✅ (1,135 commits, 219 contributors, v0.10.2)
2. **Claude Agent SDK**: ✅ ✅ (333 commits, 47 contributors, v0.1.44, 官方维护)
3. **Codex CLI**: ✅ ✅ (OpenAI 官方，Rust 高性能)
4. **Gemini CLI**: ✅ ✅ (Google 官方，monorepo)
5. **Kimi CLI**: ✅ (Moonshot AI 官方，PyPI 发布)
6. **Qwen Code**: ✅ (阿里云通义千问团队)

**实验性/社区驱动**:
7. **OpenCode**: ⚠️ 社区驱动（非大厂官方）
8. **OpenManus**: ⚠️ 社区实验（Flow 系统标注为不稳定）
9. **SWE-agent**: ⚠️ 研究框架（非生产目的）

---

## 5) 技术亮点与独特优势

### 5.1 OpenAI Agents SDK 的独特优势

**核心亮点**:
1. **Protocol-based 扩展**: Session/Model/TracingProcessor 使用 Protocol（duck typing）
2. **RunState for HITL**: 2,384 lines 的可序列化状态，支持中断恢复
3. **Handoff as First-Class**: 专门的 Handoff 类with input_filter 和 history nesting
4. **内置 Tracing**: 完整的 Trace → Span 架构，支持 6+ 外部平台
5. **Streaming 优先**: `run_streamed()` 一等公民
6. **社区活跃**: 219 contributors，19.2k stars

**代码证据**:
- `src/agents/run.py:396-1329` - 主 agent loop 实现
- `src/agents/run_state.py` - 2,384 lines RunState
- `src/agents/handoffs/__init__.py:93-321` - Handoff 实现
- `src/agents/tracing/processor_interface.py` - TracingProcessor Protocol

### 5.2 Claude Agent SDK 的独特优势

**核心亮点**:
1. **10+ Hook Events**: PreToolUse, PostToolUse, UserPromptSubmit, SubagentStart/Stop 等
2. **SDK MCP Servers**: In-process MCP 服务器，无子进程开销
3. **Bidirectional Control**: 双向 JSON-RPC control protocol
4. **Auto-bundled CLI**: 自动下载并打包 Claude Code CLI
5. **Python Keyword Safe**: `async_`, `continue_` 字段名自动转换

**代码证据**:
- `_internal/query.py:119-163` - Hook 注册
- `_internal/query.py:286-299` - Hook 调用
- `_internal/query.py:391-527` - SDK MCP JSONRPC 路由
- `__init__.py:157-319` - `create_sdk_mcp_server()`

### 5.3 OpenCode 的独特优势

**核心亮点**:
1. **Permission Ruleset Engine**: Wildcard matching, last match wins, 3 层 merge
2. **Instance-Scoped State**: Per-project 状态隔离
3. **Event Bus Architecture**: 中央 pub/sub 消息系统
4. **Agent Profiles**: 4 个内置（build/plan/general/explore）+ 用户自定义
5. **Plugin System**: Native plugins with lifecycle hooks
6. **TUI**: Solid.js + Tauri 桌面应用

**代码证据**:
- `src/permission/next.ts:236-243` - Permission evaluation
- `src/agent/agent.ts:76-203` - Agent profiles
- `src/tool/task.ts:66-102` - Hierarchical sessions
- `src/session/prompt.ts:274-724` - Main execution loop

### 5.4 Codex CLI 的独特优势

**核心亮点**:
1. **Platform Sandboxes**: Seatbelt (macOS) / Landlock (Linux) / Windows Job Objects
2. **MCP 双角色**: 既是 client 也是 server，唯一完整实现
3. **Approval Policy**: `.rules` 文件with prefix/network rules
4. **Rust 核心**: 高性能，type-safe，9,712 lines in codex.rs
5. **67 Crates**: 模块化设计，职责清晰
6. **App Server**: JSON-RPC protocol for IDE integration

**代码证据**:
- `codex-rs/core/src/codex.rs` - 9,712 lines 核心编排
- `codex-rs/core/src/exec_policy.rs` - 2,149 lines 审批引擎
- `codex-rs/mcp-server/src/message_processor.rs` - 603 lines MCP server
- `codex-rs/core/src/seatbelt.rs` - macOS sandbox

### 5.5 其他项目的独特优势

**Kimi CLI**:
- ✅ **完整 ACP Server**: 351 lines in `acp/server.py`
- ✅ **Multi-process OAuth**: File-lock based token coordination
- ✅ **Skills + Flow**: Markdown skills + Mermaid/D2 workflows
- ✅ **Triple UI**: Shell TUI + Web UI + ACP

**Gemini CLI**:
- ✅ **Google Search Grounding**: Native web-search model with citations
- ✅ **Monorepo**: CLI + Core + SDK + VS Code ext + A2A server
- ✅ **Policy Engine**: TOML-based, 792 lines
- ✅ **Service Account**: Google SA impersonation for enterprise

**Qwen Code**:
- ✅ **Context Variable Templating**: `${project_name}`, `${current_directory}` in prompts
- ✅ **SubAgent Precedence**: session > project > user > extension > builtin
- ✅ **IDE Context Injection**: Full & delta modes
- ✅ **Multi-SDK**: TypeScript + Java SDK

**SWE-agent**:
- ✅ **YAML-Driven**: 完全配置化 agent 行为
- ✅ **Tool Bundles**: Shell scripts + YAML definitions
- ✅ **Trajectory Recording**: 完整执行路径
- ✅ **SWE-Bench**: 自动 benchmark 集成

**OpenManus**:
- ✅ **ReAct 继承链**: BaseAgent → ReActAgent → ToolCallAgent → Manus
- ✅ **PlanningFlow**: LLM-driven multi-agent planning (442 lines)
- ✅ **MCP 双角色**: Client + Server (FastMCP)
- ✅ **Browser Automation**: Browser context helper

---

## 6) 选择决策树（基于深度分析）

### 6.1 按需求选择

**需求: 构建生产级 Python 多 agent 应用**
→ **OpenAI Agents SDK**
- 理由: 完整的 Session/Tracing/Guardrails，provider-agnostic，社区活跃
- 代码: `src/agents/run.py`, `src/agents/handoffs/`, `src/agents/tracing/`

**需求: 深度集成 Claude，需要强大的 Hooks**
→ **Claude Agent SDK**
- 理由: 10+ hook events，SDK MCP servers，官方支持
- 代码: `_internal/query.py:286-299` (hook invocation)
- 权衡: 接受闭源 CLI 和 Anthropic ToS

**需求: 终端编码，需要完全开源 + 强权限控制**
→ **OpenCode**
- 理由: 100% 开源，强大的 permission engine，provider-agnostic
- 代码: `src/permission/next.ts:236-243` (permission evaluation)

**需求: 企业级安全，需要 Sandbox + 审批策略**
→ **Codex CLI**
- 理由: 唯一提供平台 sandbox，审批策略完善，MCP 双角色
- 代码: `codex-rs/core/src/exec_policy.rs` (2,149 lines)
- 权衡: Rust 学习曲线，配置复杂

**需求: IDE 集成，需要 ACP Server**
→ **Kimi CLI**或**Qwen Code**
- Kimi: 完整 ACP 实现 (`acp/server.py:107-166`)
- Qwen: 深度 IDE MCP 集成 (`ide/ide-client.ts:77-206`)

**需求: Google 生态，需要 Search grounding**
→ **Gemini CLI**
- 理由: Native Google Search，Code Assist，OAuth2
- 代码: `tools/web-search.ts:82-200` (grounding implementation)

**需求: 学术研究，需要 benchmark SWE-Bench**
→ **SWE-agent**
- 理由: YAML-driven，trajectory recording，SWE-Bench 集成
- 代码: `sweagent/agent/agents.py:1265` (run loop)

**需求: 快速实验，需要 MCP 双角色**
→ **OpenManus**
- 理由: 简单 ReAct 链，MCP client+server，易上手
- 代码: `app/agent/manus.py`, `app/mcp/server.py`
- 权衡: Flow 系统不稳定

### 6.2 按技术栈选择

**Python 应用嵌入**:
1. **首选**: OpenAI Agents SDK（完整特性，provider-agnostic）
2. **次选**: Claude Agent SDK（强 Hooks，但 Claude-only）
3. **备选**: Kimi CLI/OpenManus（CLI 本体，非纯 SDK）

**终端 CLI 工具**:
1. **开源 + 跨模型**: OpenCode（Bun/TS，permission engine）
2. **企业安全**: Codex CLI（Rust，sandbox + approval）
3. **IDE 集成**: Kimi CLI（ACP）或 Qwen Code（MCP IDE）
4. **Google 生态**: Gemini CLI（Search grounding）

**研究/实验**:
1. **Benchmark**: SWE-agent（YAML-driven, SWE-Bench）
2. **通用实验**: OpenManus（ReAct 链，MCP 双角色）

### 6.3 按安全需求选择

**最高安全（Sandbox + Approval + Network）**:
→ **Codex CLI**（唯一提供 OS-level sandbox）

**细粒度权限控制**:
1. **OpenCode**: Permission ruleset with wildcards
2. **Claude Agent SDK**: 10+ hook events
3. **OpenAI Agents SDK**: Guardrails (input/output/tool)
4. **Gemini CLI**: TOML policy engine

**基础审批**:
- Kimi CLI, Qwen Code, SWE-agent, OpenManus

---

## 7) 技术债务与风险评估

| 项目 | 技术债务 | 风险点 | 缓解措施 |
|------|---------|-------|---------|
| **OpenAI Agents SDK** | ⚠️ 缺少 RAG/Vector stores | Provider API 依赖 | 使用外部 RAG 库集成 |
| **Claude Agent SDK** | ❌ 依赖闭源 CLI | Anthropic ToS 约束 | 无法缓解（架构限制） |
| **OpenCode** | ⚠️ Bun 运行时依赖 | 社区维护风险 | Bun 兼容性良好 |
| **Codex CLI** | ⚠️ Rust 学习曲线高 | 配置复杂度 | 详细文档 + TypeScript SDK |
| **Kimi CLI** | ⚠️ 文档主要为中文 | Kimi 服务依赖 | 支持其他 LLMs |
| **Gemini CLI** | ⚠️ Google 生态绑定 | Gemini 服务依赖 | 支持 OpenAI/Anthropic |
| **Qwen Code** | ⚠️ 文档部分中文 | Qwen 服务依赖 | 支持其他 LLMs |
| **SWE-agent** | ❌ 非生产目的 | 研究框架，API 不稳定 | 仅用于研究 |
| **OpenManus** | ❌ Flow 系统不稳定 | 社区维护 | 使用 main.py（稳定） |

---

## 8) 功能矩阵（完整对比）

| Feature | OpenAI SDK | Claude SDK | OpenCode | Codex | Kimi | Gemini | Qwen | SWE-agent | OpenManus |
|---------|-----------|-----------|---------|-------|------|--------|------|-----------|-----------|
| **Multi-agent** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ | ⚠️ | ⚠️ |
| **Handoffs** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ | ✅ ✅ | ✅ | ✅ ✅ | ❌ | ⚠️ |
| **Sessions** | ✅ ✅ ✅ | ❌ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ | ✅ | ❌ |
| **Tracing** | ✅ ✅ ✅ | ❌ | ⚠️ OTEL | ⚠️ Hooks | ⚠️ Logs | ⚠️ Telem | ⚠️ Logs | ❌ | ❌ |
| **HITL** | ✅ ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ⚠️ | ⚠️ |
| **Streaming** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ | ⚠️ | ⚠️ |
| **Guardrails** | ✅ ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ | ⚠️ | ⚠️ |
| **MCP Client** | ⚠️ | ❌ | ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ | ✅ ✅ ✅ | ❌ | ✅ ✅ ✅ |
| **MCP Server** | ❌ | ✅ ✅ ✅ | ❌ | ✅ ✅ ✅ | ❌ | ❌ | ❌ | ❌ | ✅ ✅ |
| **Sandbox** | ❌ | ❌ | ❌ | ✅ ✅ ✅ | ❌ | ❌ | ⚠️ Docker | ✅ Docker | ❌ |
| **IDE 集成** | ❌ | ❌ | ⚠️ ACP | ✅ App Server | ✅ ✅ ✅ ACP | ✅ VS Code | ✅ ✅ ✅ MCP | ❌ | ❌ |
| **Skills/Flows** | ❌ | ❌ | ✅ Skills | ❌ | ✅ ✅ Skills+Flow | ❌ | ✅ ✅ ✅ Skills | ❌ | ⚠️ Flow不稳定 |
| **Provider-agnostic** | ✅ ✅ ✅ | ❌ | ✅ ✅ ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ ✅ | ✅ ✅ |
| **License** | MIT | MIT+ToS | MIT | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 | MIT | MIT |
| **生产就绪** | ✅ ✅ ✅ | ✅ ✅ | ⚠️ | ✅ ✅ | ✅ | ✅ ✅ | ✅ | ❌ | ⚠️ |

**图例**: ✅ ✅ ✅ = 优秀 | ✅ ✅ = 良好 | ✅ = 基础支持 | ⚠️ = 部分/实验性 | ❌ = 不支持

---

## 9) 代码规模与复杂度对比

| 项目 | 核心代码行数 | 语言 | 关键文件行数 | 复杂度评级 |
|------|------------|------|------------|----------|
| **OpenAI Agents SDK** | ~15,000 lines | Python | `run.py` 1,623<br>`run_state.py` 2,384<br>`tool_execution.py` ~1,400 | ⭐⭐⭐ 中等 |
| **Claude Agent SDK** | SDK ~3,000 lines + 闭源 CLI | Python | `query.py` 634<br>`subprocess_cli.py` 629<br>`types.py` 859 | ⭐⭐ 简单（SDK层） |
| **OpenCode** | ~47,324 lines | TypeScript | `prompt.ts` 1,451+<br>`config.ts` ~2,000 | ⭐⭐⭐⭐ 高 |
| **Codex CLI** | Core ~9,712 lines (codex.rs) | Rust + TS | `codex.rs` 9,712<br>`exec_policy.rs` 2,149 | ⭐⭐⭐⭐⭐ 很高 |
| **Kimi CLI** | 未统计 | Python + TS | `toolset.py` 466<br>`oauth.py` 790<br>`wire/server.py` 732 | ⭐⭐⭐ 中等 |
| **Gemini CLI** | Core ~866 files | TypeScript | `client.ts` 1,120<br>`coreToolScheduler.ts` 1,354<br>`mcp-client.ts` 2,073 | ⭐⭐⭐⭐ 高 |
| **Qwen Code** | 未统计 | TypeScript | `subagent.ts` 1,004<br>`subagent-manager.ts` 975<br>`skill-manager.ts` 661 | ⭐⭐⭐ 中等 |
| **SWE-agent** | Agent ~1,294 lines | Python | `agents.py` 1,294<br>`models.py` 903<br>`tools.py` 430 | ⭐⭐ 简单 |
| **OpenManus** | 未统计 | Python | `manus.py` 165<br>`planning.py` 442<br>`llm.py` 766 | ⭐⭐ 简单 |

---

## 10) 最终推荐（基于深度分析）

### 10.1 生产应用（按优先级）

**Python 应用嵌入**:
1. **OpenAI Agents SDK** - 如果需要 provider-agnostic + 完整特性（Session/Tracing/Guardrails）
2. **Claude Agent SDK** - 如果深度集成 Claude + 需要强大 Hooks

**终端 CLI 工具**:
1. **Codex CLI** - 如果需要最高安全（Sandbox + Approval）
2. **OpenCode** - 如果需要 100% 开源 + 强权限控制
3. **Kimi CLI** - 如果需要 IDE 集成（ACP Server）
4. **Gemini CLI** - 如果深度集成 Google 生态
5. **Qwen Code** - 如果使用 Qwen 生态 + 需要 SubAgents

### 10.2 研究/实验（按优先级）

1. **SWE-agent** - Benchmark, 自动修复研究
2. **OpenManus** - 快速实验，MCP 探索

### 10.3 不推荐的组合

❌ **OpenAI Agents SDK + 需要 RAG**: 无内置 RAG，选 LangChain  
❌ **Claude Agent SDK + 跨 LLM**: Claude-only，选 OpenAI SDK  
❌ **SWE-agent + 生产应用**: 研究框架，非生产目的  
❌ **OpenManus + 稳定 multi-agent**: Flow 系统标注为不稳定  

---

## 11) 参考资源

### 11.1 深度调研文档（本次调研产出）

所有项目的深度技术调研文档位于 `agent_deep_dive/` 目录：

- **OpenAI-Agents-SDK-DEEP-DIVE.md** (106 KB) - 最详细示范文档
- **Claude-Agent-SDK-Python-DEEP-DIVE.md** (40 KB)
- **OpenCode-DEEP-DIVE.md** (20 KB)
- **OpenManus-DEEP-DIVE.md** (21 KB)
- **Codex-CLI-DEEP-DIVE.md** (17 KB)
- **Kimi-CLI-DEEP-DIVE.md** (11 KB)
- **Gemini-CLI-DEEP-DIVE.md** (6.5 KB)
- **Qwen-Code-DEEP-DIVE.md** (7.3 KB)
- **SWE-agent-DEEP-DIVE.md** (7.7 KB)

每个文档包含：
- ✅ 代码架构深度分析（具体到文件路径和行号）
- ✅ 底层实现原理（带代码片段）
- ✅ 与竞品对比
- ✅ 扩展开发指南（完整代码示例）
- ✅ 评估与建议

### 11.2 GitHub 仓库链接

- **OpenManus**: <https://github.com/FoundationAgents/OpenManus>  
- **OpenCode**: <https://github.com/anomalyco/opencode>  
- **OpenAI Agents SDK**: <https://github.com/openai/openai-agents-python>  
- **Claude Agent SDK**: <https://github.com/anthropics/claude-agent-sdk-python>  
- **Codex CLI**: <https://github.com/openai/codex>  
- **Kimi CLI**: <https://github.com/MoonshotAI/kimi-cli>  
- **Gemini CLI**: <https://github.com/google-gemini/gemini-cli>  
- **Qwen Code**: <https://github.com/QwenLM/qwen-code>  
- **SWE-agent**: <https://github.com/SWE-agent/SWE-agent>  
- **Swarm** (deprecated): <https://github.com/openai/swarm>

### 11.3 关键代码位置速查

**OpenAI Agents SDK**:
- Agent loop: `src/agents/run.py:396-1329`
- Handoffs: `src/agents/handoffs/__init__.py:93-321`
- Tracing: `src/agents/tracing/processor_interface.py`

**Claude Agent SDK**:
- Query control: `_internal/query.py:119-163`
- Hook invocation: `_internal/query.py:286-299`
- SDK MCP: `_internal/query.py:391-527`

**OpenCode**:
- Run command: `src/cli/cmd/run.ts:221-623`
- Permission engine: `src/permission/next.ts:236-243`
- Task delegation: `src/tool/task.ts:45-163`

**Codex CLI**:
- Core orchestration: `codex-rs/core/src/codex.rs` (9,712 lines)
- Approval policy: `codex-rs/core/src/exec_policy.rs` (2,149 lines)
- MCP server: `codex-rs/mcp-server/src/message_processor.rs` (603 lines)

**其他项目**: 详见各自的 DEEP-DIVE 文档

---

## 12) 版本信息与更新日志

| 项目 | 最新版本 | 发布日期 | 更新频率 | 维护状态 |
|------|---------|---------|---------|---------|
| **OpenAI Agents SDK** | v0.10.2 | 2026-02-26 | 活跃（1,135 commits） | ✅ 官方维护 |
| **Claude Agent SDK** | v0.1.44 | 2026-02-26 | 活跃（333 commits） | ✅ 官方维护 |
| **OpenCode** | 未查询 | 持续更新 | 活跃 | ✅ 社区维护 |
| **Codex CLI** | 未公开 | 持续更新 | 活跃 | ✅ 官方维护 |
| **Kimi CLI** | 未查询 | 持续更新 | 活跃 | ✅ 官方维护 |
| **Gemini CLI** | 未查询 | 持续更新 | 活跃 | ✅ 官方维护 |
| **Qwen Code** | 未查询 | 持续更新 | 活跃 | ✅ 官方维护 |
| **SWE-agent** | 未查询 | 持续更新 | 活跃 | ✅ 社区维护 |
| **OpenManus** | 未查询 | 持续更新 | 中等 | ⚠️ 社区维护 |

---

## 总结

本调研基于 **2026-02-26**的深度代码分析，覆盖 9 个主流 Agent 框架，总计审查**~300 KB** 源码文档。

**核心发现**:

1. **Python SDK 双雄**: OpenAI Agents SDK（provider-agnostic，完整特性）vs. Claude Agent SDK（强 Hooks，Claude-only）
2. **终端 CLI 多样化**: OpenCode（100% 开源）、Codex（最强安全）、Kimi（ACP IDE）、Gemini（Google）、Qwen（Skills）各有千秋
3. **MCP 双角色**: 仅 Codex CLI 和 OpenManus 同时实现 MCP Client + Server
4. **Security 领袖**: Codex CLI 唯一提供 OS-level Sandbox（Seatbelt/Landlock/Windows）
5. **研究框架**: SWE-agent 专注 benchmark，OpenManus Flow 系统不稳定

**选择建议**: 根据 **需求（功能）**+**技术栈（语言）**+**安全级别** 三维度综合决策，详见第 6 节决策树。

---

**调研团队**: 深度技术调研组  
**调研日期**: 2026-02-26  
**文档版本**: 2.0（基于深度代码分析）  
**总计调研**: 9 个项目，~300 KB 深度文档，包含具体代码位置和架构分析
