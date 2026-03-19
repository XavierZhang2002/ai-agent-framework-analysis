# Codex CLI - 深度技术调研

!!! info "项目概览"
    **调研日期**: 2026-02-26  
    **仓库**: https://github.com/openai/codex  
    **许可证**: Apache-2.0  
    **主要语言**: Rust + TypeScript  
    **定位**: 本地优先的终端 coding agent，带审批/沙箱/MCP  

---

## 1. 项目概述

### 1.1 核心特点

Codex CLI 是 OpenAI 官方的终端 coding agent，独特之处：

- ✅ **Rust 核心**: 高性能、类型安全的核心引擎
- ✅ **双 MCP 角色**: 既是 MCP 客户端，也是 MCP 服务器
- ✅ **多层安全**: 审批策略 + Sandbox + 网络控制
- ✅ **企业级**: 可控执行、可审计工作流
- ✅ **跨平台 Sandbox**: Seatbelt (macOS) / Landlock (Linux) / Windows Job Objects

### 1.2 架构概览

```
Codex CLI (Rust)
  ├─ MCP Client: 连接外部 MCP 服务器获取工具
  ├─ MCP Server: 暴露 codex_tool_call 给其他 agents
  ├─ Approval Policy: .rules 文件定义命令白名单/黑名单
  ├─ Sandbox: 平台特定的进程隔离
  └─ App Server: JSON-RPC 协议供 IDE 集成
```

---

## 2. 代码架构深度分析

### 2.1 目录结构（67 个 Rust crates）

```
codex-rs/
├── cli/               # 主 CLI 入口点
│   └── src/main.rs           # 1,469 lines - 子命令路由
├── core/              # 核心业务逻辑
│   └── src/
│       ├── codex.rs          # 9,712 lines - 核心 agent 编排
│       ├── thread_manager.rs # 745 lines - 对话管理
│       ├── codex_thread.rs   # 133 lines - Thread 抽象
│       ├── exec_policy.rs    # 2,149 lines - 审批策略引擎
│       ├── tools/            # 工具系统
│       ├── mcp/              # MCP 集成 (544 lines in mod.rs)
│       └── config/           # 配置系统
├── tui/               # Terminal UI (ratatui)
│   └── src/                  # 72 个源文件
├── mcp-server/        # MCP Server 模式
│   └── src/
│       ├── lib.rs            # 154 lines - Server 入口
│       ├── message_processor.rs # 603 lines - 请求处理
│       ├── codex_tool_runner.rs # Codex-as-tool 执行器
│       ├── exec_approval.rs  # 审批引发
│       └── patch_approval.rs # Patch 审批
├── rmcp-client/       # MCP Client
│   └── src/                  # MCP 客户端实现
├── execpolicy/        # 审批策略
│   └── src/
│       ├── policy.rs         # Policy 评估引擎
│       ├── rule.rs           # 规则匹配逻辑
│       ├── decision.rs       # 决策类型
│       └── parser.rs         # .rules 文件解析器
├── app-server/        # App Server for IDEs
│   └── src/                  # 16 个源文件
├── sandboxing/        # Sandbox 实现
│   └── src/
│       ├── seatbelt.rs       # macOS Seatbelt
│       ├── landlock.rs       # Linux Landlock
│       └── windows_sandbox.rs # Windows Job Objects
└── [58+ more crates]
```

### 2.2 核心类关系

```
ThreadManager (管理多个 threads)
    ↓
CodexThread (单个对话)
    ↓
Codex (核心 agent 实例)
    ↓
ToolRegistry (工具调度)
    ↓
ExecPolicyManager (审批策略)
    ↓
SandboxPolicy (沙箱执行)
```

### 2.3 Core Agent Loop - 三层架构 (Verified)

**文件**: `codex-rs/core/src/codex.rs` (9,712 lines)

Codex 的核心采用**Actor Model**实现三层消息循环架构，实现高并发、事件驱动的 agent 执行：

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 1: Submission Loop                 │
│                   (lines 3685-3855)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Actor Model Message Loop                           │   │
│  │  - Op::UserInput     - 用户消息提交                 │   │
│  │  - Op::ExecApproval  - 命令执行审批                 │   │
│  │  - Op::Interrupt     - 中断信号                     │   │
│  │  - Op::Shutdown      - 关闭信号                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                            ↓                                │
│                    Layer 2: Turn Execution                  │
│                   (lines 4837-5199)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Main Turn Processing Loop                          │   │
│  │  - 单 turn 内多 sampling 请求处理                   │   │
│  │  - Context Compaction (上下文压缩)                  │   │
│  │  - Follow-up 判断与执行                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                            ↓                                │
│                    Layer 3: Sampling Loop                   │
│                   (lines 6220-6554)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LLM Streaming Response Processing                  │   │
│  │  - Parallel Tool Execution (FuturesOrdered)         │   │
│  │  - Event-driven: OutputItemDone, Completed          │   │
│  │  - Real-time streaming to UI                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### Layer 1: Submission Loop (lines 3685-3855)

**Actor 模型的消息循环**，通过 channel 接收外部操作：

```rust
// codex-rs/core/src/codex.rs (lines 3685-3855)
async fn submission_loop(
    &self,
    mut rx_sub: mpsc::Receiver<Submission>,
    event_tx: mpsc::Sender<Event>,
    // ...
) {
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op {
            Op::UserInput { items, reply_to } => {
                handlers::user_input_or_turn(...).await
            }
            Op::ExecApproval { id, decision } => {
                handlers::exec_approval(...).await
            }
            Op::Interrupt { id } => {
                // 处理中断
            }
            Op::Shutdown => break,
        }
    }
}
```

**关键特性**:
- **Actor Model**: Session 作为 Actor，通过消息传递实现并发安全
- **结构化并发**: 每个 operation 有唯一 ID，支持追踪和取消
- **背压处理**: Channel-based 通信防止内存溢出

#### Layer 2: Turn Execution (lines 4837-5199)

**单 turn 内的主执行循环**，处理 LLM 请求和后续动作：

```rust
// codex-rs/core/src/codex.rs (lines 4837-5199)
pub(crate) async fn run_turn(
    &self,
    items_for_next_turn: Vec<ConvertibleItem>,
    // ...
) -> TurnResult {
    loop {
        // 构建 sampling 请求输入
        let sampling_request_input = build_sampling_request(...);
        
        // 执行 sampling 请求
        match run_sampling_request(...).await {
            Ok(result) => {
                // 判断是否需要进行 follow-up turn
                if !result.needs_follow_up { 
                    break; 
                }
                continue;
            }
            Err(e) => {
                // 错误处理和 context compaction
            }
        }
    }
}
```

**关键特性**:
- **多 Sampling 支持**: 单 turn 可包含多个独立的 LLM 调用
- **Context Compaction**: 自动上下文压缩，管理 token 限制
- **Follow-up 逻辑**: 智能判断是否需要继续执行

#### Layer 3: Sampling Loop (lines 6220-6554)

**LLM 流式响应处理**，支持并行工具执行：

```rust
// codex-rs/core/src/codex.rs (lines 6220-6554)
async fn try_run_sampling_request(
    &self,
    client_session: &mut ClientSession,
    prompt: impl Into<PromptInput>,
    // ...
) -> Result<SamplingResult, Error> {
    // 创建 LLM 流
    let mut stream = client_session.stream(prompt, ...).await?;
    
    // 并行工具执行队列 (FuturesOrdered 保持顺序)
    let mut in_flight: FuturesOrdered<ToolCallFuture> = FuturesOrdered::new();
    
    loop {
        match stream.next().await {
            ResponseEvent::OutputItemDone(item) => {
                // 工具调用入队
                if let Some(tool_future) = output_result.tool_future {
                    in_flight.push_back(tool_future);
                }
            }
            ResponseEvent::OutputTextDelta { delta, .. } => {
                // 实时流式输出到 UI
                event_tx.send(Event::OutputTextDelta { ... }).await?;
            }
            ResponseEvent::Completed { ... } => break,
            ResponseEvent::ErrorOccurred { error } => {
                return Err(error.into());
            }
        }
    }
}
```

**关键特性**:
- **Parallel Tool Execution**: 使用 `FuturesOrdered` 实现并发工具调用，同时保持输出顺序
- **Streaming Architecture**: LLM 流式响应实时转发到 UI，降低延迟
- **Event-driven**: 基于事件的架构，支持 OutputItemDone、Completed、OutputTextDelta 等事件

### 2.4 架构亮点总结

| 特性 | 实现 | 优势 |
|------|------|------|
| **Actor Model** | Session 作为 Actor，消息传递 | 并发安全、状态隔离 |
| **三层循环** | Submission → Turn → Sampling | 清晰的职责分离 |
| **并行工具执行** | `FuturesOrdered` | 并发性能 + 顺序保证 |
| **流式架构** | 实时 event streaming | 低延迟、实时反馈 |
| **审批系统** | 所有命令需 approval | 安全可控 |
| **三层沙箱** | OS-level + Policy + Network | 企业级安全 |

### 2.5 关键支持文件

| 文件 | 行数 | 功能 |
|------|------|------|
| `core/src/codex.rs` | 9,712 | 核心 agent 编排，三层循环架构 |
| `core/src/tools/router.rs` | ~300 | 工具路由调度 |
| `core/src/tools/parallel.rs` | ~150 | 并行工具执行支持 |
| `core/src/seatbelt.rs` | ~200 | macOS Seatbelt 沙箱 |
| `core/src/landlock.rs` | ~180 | Linux Landlock 沙箱 |
| `core/src/exec_policy.rs` | 2,149 | 审批策略引擎 |

---

## 3. 底层实现原理

### 3.1 Session 和对话管理

**文件**: `codex-rs/core/src/thread_manager.rs` (745 lines)

```rust
pub struct ThreadManager {
    threads: Arc<RwLock<HashMap<ThreadId, Arc<CodexThread>>>>,
    thread_created_tx: broadcast::Sender<ThreadId>,
    auth_manager: Arc<AuthManager>,
    models_manager: Arc<ModelsManager>,
    skills_manager: Arc<SkillsManager>,
    file_watcher: Arc<FileWatcher>,
    session_source: SessionSource,
}

// 关键方法 (lines 200-300):
pub async fn create_thread(...) -> CodexResult<NewThread>
pub async fn spawn_codex(...) -> CodexResult<CodexSpawnOk>
```

**Session 流程**:
1. `ThreadManager::create_thread()` 初始化新对话
2. 创建 `Codex` 实例with config snapshot
3. 为事件处理生成后台任务
4. 返回包装在 Arc 中的 `CodexThread` 用于并发访问

**文件**: `codex-rs/core/src/codex_thread.rs` (133 lines)

```rust
pub struct CodexThread {
    codex: Codex,
    rollout_path: Option<PathBuf>,
    _watch_registration: WatchRegistration,
}

// 关键方法:
pub async fn submit(&self, op: Op) -> CodexResult<String>
pub async fn steer_input(&self, input: Vec<UserInput>, expected_turn_id: Option<&str>)
pub async fn next_event(&self) -> CodexResult<Event>
```

**对话持久化**:
- Rollout 文件存储在 `~/.codex/sessions/` 或 `~/.codex/archived-sessions/`
- 格式: JSONL（每行一个事件）
- Thread 元数据在 SQLite 数据库: `codex-rs/core/src/state_db.rs`

### 3.2 MCP 双角色实现

#### **MCP Server 模式（Codex AS MCP Server）**

**文件**: `codex-rs/mcp-server/src/lib.rs` (154 lines)

```rust
pub async fn run_main(
    arg0_paths: Arg0DispatchPaths, 
    cli_config_overrides: CliConfigOverrides
)
```

**架构**:
- **3 个异步任务**: stdin reader, message processor, stdout writer
- **有界通道**: 128 消息容量用于背压
- **Transport**: stdio，换行符分隔的 JSON

**文件**: `codex-rs/mcp-server/src/message_processor.rs` (603 lines)

```rust
pub(crate) struct MessageProcessor {
    outgoing: Arc<OutgoingMessageSender>,
    initialized: bool,
    thread_manager: Arc<ThreadManager>,
    running_requests_id_to_codex_uuid: Arc<Mutex<HashMap<RequestId, ThreadId>>>,
}

// 关键处理器 (lines 81-154):
async fn handle_initialize(...)
async fn handle_list_tools(...)
async fn handle_call_tool(...)
```

**暴露的工具**:
- `codex_tool_call` - 将 Codex 作为工具运行
- `codex_tool_call_reply` - 向运行中的 Codex 发送反馈

**MCP 工具调用流程**:
1. Client 发送 `tools/call` 请求with `codex_tool_call`
2. `MessageProcessor` 通过 `ThreadManager` 创建新 thread
3. 提交用户消息并流式传输事件
4. 聚合 items 并发送 `tools/call` 响应

#### **MCP Client 模式（Codex USES MCP Servers）**

**文件**: `codex-rs/core/src/mcp_connection_manager.rs`

```rust
pub struct McpConnectionManager {
    connections: HashMap<String, McpConnection>,
    tools: HashMap<String, Tool>,
    resources: HashMap<String, Vec<Resource>>,
}

// 关键方法:
pub async fn initialize_servers(...)
pub async fn call_tool(...) -> CallToolResult
```

**MCP 集成文件**:
- `codex-rs/core/src/mcp/mod.rs` (544 lines) - MCP helpers
- `codex-rs/core/src/tools/handlers/mcp.rs` (68 lines) - MCP tool handler
- `codex-rs/rmcp-client/` - 低级 MCP 客户端

**MCP Server 配置**（from `config/types.rs` lines 62-227）:

```rust
pub struct McpServerConfig {
    pub transport: McpServerTransportConfig,
    pub enabled: bool,
    pub required: bool,
    pub startup_timeout_sec: Option<Duration>,
    pub tool_timeout_sec: Option<Duration>,
    pub enabled_tools: Option<Vec<String>>,
    pub disabled_tools: Option<Vec<String>>,
    pub scopes: Option<Vec<String>>,
}

pub enum McpServerTransportConfig {
    Stdio { command, args, env, env_vars, cwd },
    StreamableHttp { url, bearer_token_env_var, http_headers, env_http_headers },
}
```

### 3.3 审批策略系统

**文件**: `codex-rs/core/src/exec_policy.rs` (2,149 lines)

```rust
pub(crate) struct ExecPolicyManager {
    policy: ArcSwap<Policy>,
}

impl ExecPolicyManager {
    pub(crate) async fn create_exec_approval_requirement_for_command(
        &self,
        req: ExecApprovalRequest<'_>,
    ) -> Result<ExecApprovalRequirement, ExecPolicyError>
}
```

**策略文件格式** (`.rules`):

```
# 允许命令
allow prefix git
allow prefix ls

# 需要审批
prompt prefix npm install

# 禁止危险操作
forbidden prefix rm -rf /
```

**关键文件**:
- `codex-rs/execpolicy/src/policy.rs` - Policy 评估引擎
- `codex-rs/execpolicy/src/rule.rs` - 规则匹配逻辑
- `codex-rs/execpolicy/src/parser.rs` - `.rules` 文件解析器
- `codex-rs/execpolicy/src/amend.rs` - 动态规则修改

**审批流程** (lines 198-400):
1. Agent 提议命令
2. `ExecPolicyManager::create_exec_approval_requirement_for_command()` 评估
3. 检查 workspace 和 home 中的 `.rules` 文件
4. 返回 `Decision`: Allow/Prompt/Forbidden
5. 如果 Prompt，通过 TUI 或 MCP elicitation 提示用户

**规则文件位置**:
- Workspace: `./.codex/rules/default.rules`
- User home: `~/.codex/rules/default.rules`

### 3.4 Sandbox 模式实现

**平台特定 Sandboxes**:

**macOS (Seatbelt)**:
- 文件: `codex-rs/core/src/seatbelt.rs`
- 二进制: `/usr/bin/sandbox-exec`
- Profiles: `.sbpl` 文件（定义读/写权限）
- 关键文件:
  - `seatbelt_base_policy.sbpl` - 基础限制
  - `seatbelt_network_policy.sbpl` - 网络规则
  - `seatbelt_platform_defaults.sbpl` - 平台默认值

**Linux (Landlock + seccomp)**:
- 文件: `codex-rs/core/src/landlock.rs`
- Crate: `codex-linux-sandbox/`
- 使用 Landlock LSM 进行文件系统限制
- seccomp 用于系统调用过滤

**Windows**:
- 文件: `codex-rs/core/src/windows_sandbox.rs`
- 使用受限 tokens 和 job objects
- 模式: Elevated/Unelevated

**Sandbox 抽象** (from `sandboxing/mod.rs`):

```rust
pub enum SandboxPermissions {
    ReadOnly,
    WorkspaceWrite { 
        writable_roots: Vec<PathBuf>, 
        network_access: bool 
    },
}

pub struct SandboxPolicy {
    pub policy_type: PolicyType,
    pub writable_roots: Vec<PathBuf>,
    pub network_access: bool,
}
```

### 3.5 工具执行流程

**文件**: `codex-rs/core/src/tools/registry.rs` (436 lines)

```rust
pub struct ToolRegistry {
    handlers: HashMap<String, Arc<dyn ToolHandler>>,
}

pub trait ToolHandler: Send + Sync {
    fn kind(&self) -> ToolKind;
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool;
    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError>;
}

// 调度流程 (lines 79-250):
pub async fn dispatch(&self, invocation: ToolInvocation) -> Result<ResponseInputItem, FunctionCallError>
```

**工具执行管道**:
1. Agent 使用 `FunctionCall` 或 MCP `CallTool` 调用工具
2. `ToolRegistry::dispatch()` 查找处理器
3. 检查是否 mutating (line 157) - 如果是，等待 gate
4. 调用 `handler.handle(invocation)`
5. 发出 telemetry 和 hook 事件
6. 返回 `ResponseInputItem` 给模型

**内置工具处理器** (`codex-rs/core/src/tools/handlers/`):
- `shell.rs` (621 lines) - Shell 命令执行
- `mcp.rs` (68 lines) - MCP 工具调用
- `read_file.rs` - 读取文件内容
- `list_dir.rs` - 列出目录
- `grep_files.rs` - 搜索文件
- `apply_patch.rs` - 应用 diffs
- `request_user_input.rs` - 用户输入提示
- `multi_agents.rs` - Multi-agent 协调
- `plan.rs` - 规划模式

### 3.6 配置系统

**文件**: `codex-rs/core/src/config/mod.rs`

```rust
pub struct Config {
    pub codex_home: PathBuf,
    pub model: String,
    pub approval_policy: AskForApproval,
    pub sandbox_policy: SandboxPolicy,
    pub mcp_servers: Constrained<HashMap<String, McpServerConfig>>,
    pub features: Features,
    // ... 100+ 字段
}
```

**配置加载** (from `config_loader/`):
1. 读取 `~/.codex/config.toml`
2. 应用 profile 覆盖 (`[profile.dev]`)
3. 应用 CLI 覆盖 (`-c key=value`)
4. 应用 constraints from `requirements.toml` (企业)
5. 验证并冻结

**关键配置文件**:
- `codex-rs/core/src/config/types.rs` (1,167 lines) - 类型定义
- `codex-rs/core/src/config/edit.rs` - 编程方式编辑
- `codex-rs/config/src/` - 配置加载/合并/验证

**Schema 生成**:
- JSON Schema: `codex-rs/core/config.schema.json`
- 命令: `just write-config-schema`

---

## 4. 扩展开发指南

### 4.1 添加自定义工具

#### **选项 1: MCP Server（推荐）**

**步骤 1: 创建 MCP Server**

```typescript
// my-tool-server/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  { name: "my-tool-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "my_custom_tool",
      description: "Does something useful",
      inputSchema: {
        type: "object",
        properties: {
          param1: { type: "string" },
        },
        required: ["param1"],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "my_custom_tool") {
    return { content: [{ type: "text", text: "Result" }] };
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

**步骤 2: 配置在 `~/.codex/config.toml`**

```toml
[mcp_servers.my_tool]
command = "node"
args = ["/path/to/my-tool-server/index.js"]
enabled = true
```

#### **选项 2: Rust Tool Handler（高级）**

**在 `codex-rs/core/src/tools/handlers/` 创建新处理器**:

```rust
// my_tool.rs
use async_trait::async_trait;
use crate::tools::registry::{ToolHandler, ToolKind};
use crate::tools::context::{ToolInvocation, ToolOutput};

pub struct MyToolHandler;

#[async_trait]
impl ToolHandler for MyToolHandler {
    fn kind(&self) -> ToolKind { ToolKind::Function }

    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        // 解析参数
        // 执行逻辑
        // 返回输出
        Ok(ToolOutput::Function {
            body: FunctionCallOutputBody::Text { text: "Result".to_string() },
            success: true,
        })
    }
}
```

**注册在 `codex-rs/core/src/tools/orchestrator.rs`**:

```rust
handlers.insert("my_tool".to_string(), Arc::new(MyToolHandler));
```

### 4.2 配置审批策略

#### **`.rules` 文件语法**

**位置**: `./.codex/rules/default.rules` 或 `~/.codex/rules/default.rules`

```
# 注释以 # 开头

# 允许特定命令前缀，无需审批
allow prefix git status
allow prefix ls
allow prefix cat

# 运行前需要审批提示
prompt prefix npm install
prompt prefix cargo build

# 禁止危险命令
forbidden prefix rm -rf
forbidden prefix sudo rm

# 网络规则 (requires execpolicy v2)
allow network protocol:https host:api.github.com
prompt network protocol:tcp port:3000
forbidden network protocol:* host:internal.corp.net
```

**编程方式修改**（from `exec_policy.rs` lines 900-1000）:

```rust
// 运行时添加规则
let amendment = ExecPolicyAmendment {
    prefix: vec!["python".to_string(), "-c".to_string()],
};
exec_policy_manager.add_allow_prefix_rule(amendment).await?;
```

**审批策略模式** (config.toml):

```toml
approval_policy = "on-request"  # 仅在 .rules 要求时提示
# approval_policy = "never"     # 无提示（危险）
# approval_policy = "on-failure" # 失败后提示
# approval_policy = "unless-trusted" # 提示，除非 .rules 允许
```

### 4.3 扩展 MCP Servers

#### **添加 OAuth 支持**

**文件**: `codex-rs/core/src/mcp/auth.rs`

```toml
[mcp_servers.my_oauth_server]
command = "npx"
args = ["-y", "my-oauth-mcp-server"]
scopes = ["read:data", "write:data"]
```

**MCP Server 实现**:

```typescript
server.setRequestHandler(ListResourcesRequestSchema, async (request) => {
  // 使用 session 中的 OAuth token
  const token = await getOAuthToken();
  // 获取资源
});
```

---

## 5. 与竞品对比

### Codex CLI vs. OpenCode

| 维度 | Codex CLI | OpenCode |
|------|-----------|---------|
| **语言** | Rust + TypeScript | TypeScript (Bun) |
| **Sandbox** | ✅ ✅ ✅ (Seatbelt/Landlock/Windows) | ❌ 无内置 |
| **审批** | .rules 文件 | Permission ruleset |
| **MCP** | ✅ ✅ Client + Server（双角色） | ✅ Client |
| **Provider** | ⚠️ OpenAI 为主 | ✅ Provider-agnostic |
| **TUI** | ✅ ratatui (Rust) | ✅ Solid.js |
| **成熟度** | ✅ OpenAI 官方 | 社区驱动 |

### Codex CLI vs. Claude Agent SDK

| 维度 | Codex CLI | Claude Agent SDK |
|------|-----------|-----------------|
| **架构** | 独立 CLI (Rust) | SDK + CLI (Python + 闭源 CLI) |
| **Sandbox** | ✅ ✅ ✅ | ❌ |
| **MCP** | ✅ ✅ Client + Server | ✅ SDK MCP Servers |
| **Hooks** | ⚠️ 基础 | ✅ ✅ ✅ (10+ events) |
| **Provider** | ⚠️ OpenAI 为主 | ❌ Claude-only |

---

## 6. 评估与建议

!!! success "核心优势"
    1. ✅ **多层安全**: Sandbox + Approval + Network Control
    2. ✅ **高性能**: Rust 核心
    3. ✅ **双 MCP 角色**: 既可使用也可被使用
    4. ✅ **企业级**: 可控执行、可审计
    5. ✅ **官方支持**: OpenAI 维护

!!! warning "局限性"
    1. ⚠️ **OpenAI 倾向**: 虽支持其他 LLMs，但优化for OpenAI
    2. ❌ **复杂度高**: Rust + TypeScript 双语言栈
    3. ⚠️ **文档分散**: 部分功能需查看源码

!!! tip "推荐场景"
    - ✅ 需要企业级安全控制
    - ✅ 本地工程改码
    - ✅ 受控执行环境
    - ✅ MCP 生态集成

!!! danger "不推荐场景"
    - ❌ 快速原型（配置复杂）
    - ❌ 非技术用户（需理解 Rust/安全概念）

---

## 附录

### A. 关键文件速查

| 文件 | 行数 | 关键内容 | 关键代码行 |
|------|------|---------|-----------|
| `cli/src/main.rs` | 1,469 | CLI 入口和子命令 | - |
| `core/src/codex.rs` | 9,712 | 核心 agent 编排 | L3685-3855 Submission Loop<br>L4837-5199 Turn Execution<br>L6220-6554 Sampling Loop |
| `core/src/thread_manager.rs` | 745 | Thread 生命周期 | L200-300 |
| `core/src/exec_policy.rs` | 2,149 | 审批策略引擎 | L198-400 |
| `core/src/tools/router.rs` | ~300 | 工具路由调度 | - |
| `core/src/tools/parallel.rs` | ~150 | 并行工具执行 | FuturesOrdered 实现 |
| `core/src/seatbelt.rs` | ~200 | macOS Seatbelt 沙箱 | - |
| `core/src/landlock.rs` | ~180 | Linux Landlock 沙箱 | - |
| `mcp-server/src/message_processor.rs` | 603 | MCP 请求处理 | L81-154 |
| `tools/handlers/shell.rs` | 621 | Shell 执行 | - |

---

**文档版本**: 1.1  
**最后更新**: 2026-02-27
