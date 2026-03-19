# Kimi CLI - 深度技术调研

!!! info "项目概览"
    **调研日期**: 2026-02-26  
    **仓库**: https://github.com/MoonshotAI/kimi-cli  
    **许可证**: Apache-2.0  
    **主要语言**: Python + TypeScript  
    **定位**: 终端 Agent + shell 一体化，强调 IDE/ACP 接入  

---

## 1. 项目概述

Kimi CLI 是 Moonshot AI 的终端 AI 助手，核心特点：

- ✅ **ACP Server**: 完整的 Agent Client Protocol 服务器实现
- ✅ **Shell 模式**: Interactive TUI with prompt_toolkit
- ✅ **Wire Protocol**: JSON-RPC for IDE 集成
- ✅ **Skills 系统**: Markdown-based skill + flow workflows
- ✅ **OAuth 集成**: Device flow with PKCE，multi-process token 协调
- ✅ **Web UI**: FastAPI backend + React frontend

---

## 2. 代码架构

### 2.1 目录结构

```
src/kimi_cli/
├── cli/              # CLI 入口 (776 lines in __init__.py)
├── app.py            # KimiCLI 主类 (364 lines)
├── acp/              # ACP Server 实现
│   ├── server.py     # 351 lines - 多 session 管理
│   ├── session.py    # 457 lines - ACPSession
│   ├── convert.py    # ACP ↔ Wire 消息转换
│   ├── tools.py      # 工具替换
│   └── kaos.py       # ACPKaos 文件系统桥
├── session.py        # Session 生命周期 (274 lines)
├── soul/             # 核心 agent 运行时
│   ├── kimisoul.py   # KimiSoul 主循环 (核心)
│   ├── denwarenji.py # DenwaRenji - 时间旅行管理
│   ├── context.py    # 上下文管理与 checkpoint/revert
│   ├── agent.py      # 323 lines - Runtime, Agent, LaborMarket
│   ├── toolset.py    # 466 lines - KimiToolset, MCP 集成
│   ├── approval.py   # 审批系统
│   ├── compaction.py # 上下文压缩
│   └── slash.py      # Slash 命令注册表
├── tools/            # 工具实现
│   ├── shell/        # Shell 工具 (128 lines)
│   ├── file/         # 文件操作（read, write, replace, glob, grep）
│   ├── web/          # Web 工具（search, fetch）
│   ├── multiagent/   # 多 agent（task, create subagent）
│   ├── ask_user/     # AskUserQuestion
│   └── todo/         # SetTodoList
├── wire/             # Wire Protocol
│   ├── server.py     # 732 lines - JSON-RPC over stdio
│   ├── file.py       # WireFile 用于 session replay
│   └── types.py      # Wire 消息类型定义
├── auth/             # OAuth 2.0
│   ├── oauth.py      # 790 lines - OAuthManager, device flow
│   └── platforms.py  # 平台特定 OAuth 配置
├── skill/            # Skills 系统
│   ├── __init__.py   # 304 lines - 发现, 加载
│   └── flow/         # Flow 执行引擎
├── web/              # Web UI
│   ├── app.py        # 540 lines - FastAPI factory
│   └── runner/       # Worker subprocess manager
└── ui/               # UI 实现
    ├── shell/        # Interactive shell TUI
    ├── print/        # Non-interactive print mode
    └── acp/          # ACP UI wrapper
```

**关键文件说明**:

| 文件路径 | 功能描述 | 核心方法/类 |
|---------|---------|------------|
| `src/kimi_cli/soul/kimisoul.py` | 核心 Agent 循环 | `_agent_loop()`, `_step()` |
| `src/kimi_cli/soul/denwarenji.py` | 时间旅行管理器 | `DenwaRenji`, `D-Mail` |
| `src/kimi_cli/soul/context.py` | 上下文快照/回滚 | `revert_to()`, `checkpoint()` |
| `src/kimi_cli/acp/server.py` | ACP 协议服务器 | `ACPServer`, `new_session()` |
| `src/kimi_cli/wire/types.py` | Wire 消息类型 | `StepBegin`, `ToolResult` |

---

## 3. 底层实现原理

### 3.1 Core Agent Loop (核心 Agent 循环)

**文件**: `src/kimi_cli/soul/kimisoul.py`

这是 Kimi CLI 最核心的 agent 执行循环，实现了独特的「时间旅行」机制。

#### _agent_loop() - 主循环 (Lines 206-275)

```python
async def _agent_loop(self) -> TurnOutcome:
    """The main agent loop for one run."""
    step_no = 0
    while True:
        step_no += 1
        if step_no > self._loop_control.max_steps_per_turn:
            raise MaxStepsReached(...)
            
        wire_send(StepBegin(n=step_no))
        
        # Context compaction check
        if self._context.token_count + reserved >= self._runtime.llm.max_context_size:
            await self.compact_context()
            
        # Create checkpoint for time travel
        await self._checkpoint()
        
        try:
            step_outcome = await self._step()
        except BackToTheFuture as e:
            # Time travel!
            await self._context.revert_to(e.checkpoint_id)
            await self._context.append_message(e.messages)
            continue
            
        if step_outcome is not None:
            return TurnOutcome(...)
```

#### _step() - 单步执行 (Lines 277-348)

```python
async def _step(self) -> StepOutcome | None:
    """Run a single step."""
    result = await kosong.step(
        chat_provider,
        self._agent.system_prompt,
        self._agent.toolset,
        self._context.history,
        on_message_part=wire_send,
        on_tool_result=wire_send,
    )
    
    # Wait for tool results
    results = await result.tool_results()
    await self._grow_context(result, results)
    
    # Handle D-Mail (time travel)
    if dmail := self._denwa_renji.fetch_pending_dmail():
        raise BackToTheFuture(dmail.checkpoint_id, [...])
        
    if result.tool_calls:
        return None  # Continue loop
    return StepOutcome(stop_reason="no_tool_calls", ...)
```

#### 循环控制流程

```
_agent_loop()
  ├── StepBegin (发送 wire 事件)
  ├── 检查上下文长度 → compact_context()
  ├── 创建 checkpoint (时间旅行锚点)
  └── _step()
        ├── kosong.step() (LLM 调用)
        ├── 等待 tool 执行结果
        ├── _grow_context() (更新历史)
        ├── 检查 D-Mail (时间旅行信号)
        └── 返回 StepOutcome (结束) 或 None (继续)
  
  ← 捕获 BackToTheFuture 异常
    ├── revert_to(checkpoint_id) (回滚)
    ├── 注入新消息
    └── continue (重新开始)
```

---

### 3.3 CLI 入口流程

**文件**: `src/kimi_cli/cli/__init__.py` (776 lines)

```python
# 入口层次结构:
kimi (CLI 命令)
  ↓
cli() -> typer.Typer callback
  ↓
kimi() function (lines 54-625)
  ↓
_run() async function (lines 461-530)
  ↓
KimiCLI.create() (in app.py)
  ↓
instance.run_shell() / run_print() / run_acp() / run_wire_stdio()
```

**关键 CLI 模式**:
- `--print`: 非交互模式 (`ui = "print"`)
- `--acp`: ACP server 模式 (`ui = "acp"`)
- `--wire`: Wire protocol over stdio (`ui = "wire"`)
- 默认: Interactive shell (`ui = "shell"`)

**关键代码**:
```python
# Line 488-500: KimiCLI 实例创建
instance = await KimiCLI.create(
    session,
    config=config,
    model_name=model_name,
    thinking=thinking,
    yolo=yolo or (ui == "print"),  # print 模式隐含 yolo
    agent_file=agent_file,
    mcp_configs=mcp_configs,
    skills_dir=skills_dir,
    max_steps_per_turn=max_steps_per_turn,
    max_retries_per_step=max_retries_per_step,
    max_ralph_iterations=max_ralph_iterations,
)
```

---

### 3.4 Unique Features (独特机制)

#### 3.4.1 D-Mail Time Travel (时间旅行系统)

**灵感来源**: 《命运石之门》(Steins;Gate) 中的 D-Mail (DeLorean Mail)

**核心组件**:
- **DenwaRenji** (`src/kimi_cli/soul/denwarenji.py`): 「电话微波炉」管理时间旅行
- **Checkpoint/Revert** (`src/kimi_cli/soul/context.py`): 上下文快照与回滚
- **BackToTheFuture** 异常: 触发时间旅行的信号

**工作原理**:
```
1. 每次 step 前创建 checkpoint (上下文快照)
2. 工具执行时可发送 D-Mail 到过去的 checkpoint
3. _agent_loop 捕获 BackToTheFuture 异常
4. 调用 revert_to(checkpoint_id) 回滚状态
5. 注入新消息，重新开始执行
```

**典型使用场景**:
- 用户中途纠正 agent 的行为
- 工具执行失败后的策略调整
- 多分支探索 (尝试 A 路径失败，回滚尝试 B)

#### 3.4.2 Steer Injection (实时转向)

**机制**: 用户可在 agent 执行过程中发送纠正指令

**实现**:
- 存储于 `asyncio.Queue` 中
- 转换为 synthetic tool call + result 形式
- 在下一个 step 前注入到上下文

#### 3.4.3 Context Compaction (上下文压缩)

**触发条件**: `token_count + reserved >= max_context_size`

**实现** (`src/kimi_cli/soul/compaction.py`):
- 对旧消息进行摘要总结
- 保留 system prompt 和最近对话
- 将摘要作为新 context 的一部分

#### 3.4.4 Tenacity Retry (指数退避重试)

**用途**: LLM API 调用失败时的自动重试

**策略**:
- 指数退避 (exponential backoff)
- 最大重试次数可配置 (`max_retries_per_step`)
- 区分可重试错误 (网络) 和不可重试错误 (无效参数)

---

### 3.5 ACP Server 实现

**文件**: `src/kimi_cli/acp/server.py` (351 lines)

**核心功能**:
- **多 Session 支持**: 同时管理多个独立的 agent sessions
- **MCP 服务器转换**: 将 ACP MCP servers 转换为内部配置格式
- **Session 生命周期**: create, load, list 操作

**架构**:
- **入口点**: `kimi acp` 命令 → `acp_main()` in `__init__.py`
- **Server 类**: `ACPServer` 管理多个 sessions
- **Session 类**: `ACPSession` 处理单个 agent sessions

**关键方法**:

```python
# Line 107-166: new_session() - 创建新 ACP session
async def new_session(
    self, cwd: str, mcp_servers: list[MCPServer], **kwargs: Any
) -> acp.NewSessionResponse:
    session = await Session.create(KaosPath.unsafe_from_local_path(Path(cwd)))
    mcp_config = acp_mcp_servers_to_mcp_config(mcp_servers)
    cli_instance = await KimiCLI.create(session, mcp_configs=[mcp_config])
    
    acp_kaos = ACPKaos(self.conn, session.id, self.client_capabilities)
    acp_session = ACPSession(session.id, cli_instance, self.conn, kaos=acp_kaos)
    self.sessions[session.id] = (acp_session, model_id_conv)
```

**工具调用流式传输** (in `acp/session.py` lines 72-104):

```python
class _ToolCallState:
    """管理流式工具调用更新到 ACP 客户端"""
    
    def __init__(self, tool_call: ToolCall):
        self.tool_call = tool_call
        self.args = tool_call.function.arguments or ""
        self.lexer = streamingjson.Lexer()  # 增量解析 JSON
    
    @property
    def acp_tool_call_id(self) -> str:
        # 每个 turn 的唯一 ID 以避免冲突
        turn_id = _current_turn_id.get()
        return f"{turn_id}/{self.tool_call.id}"
```

### 3.6 Session 管理

**文件**: `src/kimi_cli/session.py` (274 lines)

**Session 结构**:
```python
@dataclass(slots=True, kw_only=True)
class Session:
    id: str                        # UUID
    work_dir: KaosPath             # 规范工作目录
    work_dir_meta: WorkDirMeta     # 工作目录元数据
    context_file: Path             # JSONL 文件存储消息历史
    wire_file: WireFile            # Wire protocol 消息日志
    state: SessionState            # 审批设置，动态 subagents
    title: str                     # 从第一条用户消息派生
    updated_at: float              # 最后修改时间戳
```

**存储布局**:
```
~/.kimi/
├── config.toml                    # 全局配置
├── mcp.json                       # MCP 服务器配置
├── metadata.json                  # Workspace → session 映射
├── logs/kimi.log                  # 应用日志
└── sessions/
    └── <work_dir_hash>/
        └── <session_id>/
            ├── context.jsonl      # 对话历史
            ├── wire.jsonl         # Wire protocol 事件
            └── state.json         # Session 状态
```

### 3.7 工具执行流程

**文件**: `src/kimi_cli/soul/toolset.py` (466 lines)

**工具加载** (lines 152-200):
```python
def load_tools(self, tool_paths: list[str], dependencies: dict[type[Any], Any]):
    """从路径加载工具，如 'kimi_cli.tools.shell:Shell'"""
    for tool_path in tool_paths:
        module_name, class_name = tool_path.rsplit(":", 1)
        module = importlib.import_module(module_name)
        tool_cls = getattr(module, class_name)
        
        # 通过构造函数检查注入依赖
        args = []
        for param in inspect.signature(tool_cls).parameters.values():
            if param.annotation in dependencies:
                args.append(dependencies[param.annotation])
        
        self.add(tool_cls(*args))
```

**工具执行** (lines 97-124):
```python
def handle(self, tool_call: ToolCall) -> HandleResult:
    token = current_tool_call.set(tool_call)  # 设置 context variable
    try:
        tool = self._tool_dict[tool_call.function.name]
        arguments = json.loads(tool_call.function.arguments or "{}")
        
        async def _call():
            ret = await tool.call(arguments)
            return ToolResult(tool_call_id=tool_call.id, return_value=ret)
        
        return asyncio.create_task(_call())  # 异步执行
    finally:
        current_tool_call.reset(token)
```

**MCP 工具集成** (lines 202-466):
- MCP servers 通过 `fastmcp` 库加载
- Servers 按需启动with connection pooling
- Tools 转换为 kosong Tool 格式
- Timeout 处理（默认 60s）
- OAuth 流程用于认证 MCP servers

### 3.8 OAuth/认证

**文件**: `src/kimi_cli/auth/oauth.py` (790 lines)

**OAuth 流程**:

```python
# CLI 登录的 Device Code Flow
async def login_kimi_code(config: Config) -> AsyncIterator[OAuthEvent]:
    # 1. 请求 device code
    device_auth = await _request_device_authorization(client_id, scope)
    
    # 2. 提示用户访问 URL
    yield OAuthEvent(
        type="verification_url",
        message=f"Visit: {device_auth.verification_uri_complete}",
    )
    
    # 3. 轮询 token
    while True:
        token = await _poll_token(device_auth.device_code)
        if token:
            await _save_token(token, KIMI_CODE_OAUTH_KEY)
            break
        await asyncio.sleep(device_auth.interval)
    
    # 4. 获取用户模型并更新配置
    models = await _fetch_user_models(token)
    _update_config_with_models(config, models)
```

**Token 存储**:
- **Keyring**（首选）: 系统凭据存储
- **文件回退**: `~/.kimi/oauth/<key>.json` with mode 0600

**自动刷新** (lines 433-510):
```python
@asynccontextmanager
async def refreshing(self, runtime: Runtime):
    """自动刷新 OAuth tokens 的 context manager"""
    refresh_task = asyncio.create_task(self._refresh_loop(runtime))
    try:
        yield
    finally:
        refresh_task.cancel()

async def _refresh_loop(self, runtime: Runtime):
    while True:
        await asyncio.sleep(REFRESH_INTERVAL_SECONDS)  # 60s
        if token.expires_at - time.time() < REFRESH_THRESHOLD_SECONDS:  # 300s
            new_token = await self._refresh_token(token)
            await self._save_token(new_token)
```

---

## 4. 扩展开发指南

### 4.1 添加自定义工具

```python
# my_tools/custom.py
from kosong.tooling import CallableTool2, ToolReturnValue
from pydantic import BaseModel, Field

class MyToolParams(BaseModel):
    query: str = Field(description="User query")

class MyTool(CallableTool2[MyToolParams]):
    name: str = "MyTool"
    params: type[MyToolParams] = MyToolParams
    
    def __init__(self, approval: Approval):
        super().__init__(description="My custom tool")
        self._approval = approval
    
    async def __call__(self, params: MyToolParams) -> ToolReturnValue:
        if not await self._approval.request(self.name, "perform action", f"Execute: {params.query}"):
            return ToolRejectedError()
        
        result = await do_something(params.query)
        
        builder = ToolResultBuilder()
        builder.write(result)
        return builder.ok("Success!")
```

### 4.2 配置 Agents

```yaml
# my_agent.yaml
version: 1
agent:
  extend: default
  name: "My Custom Agent"
  system_prompt_path: ./system.md
  tools:
    - "my_tools.custom:MyTool"
```

### 4.3 IDE 集成

**配置 (Zed/JetBrains)**:
```json
{
  "agent_servers": {
    "Kimi Code CLI": {
      "command": "kimi",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

---

## 5. 评估

!!! success "核心优势"
    1. ✅ **完整的 ACP 支持**: IDE 集成一流
    2. ✅ **Skills 系统**: 可扩展的专业知识包
    3. ✅ **Web UI**: 桌面友好的 React UI
    4. ✅ **OAuth 集成**: 企业级认证

!!! warning "局限性"
    1. ⚠️ **Kimi 倾向**: 优化for Kimi模型
    2. ⚠️ **文档**: 中文文档为主

!!! tip "推荐场景"
    - ✅ IDE Agent Server 接入
    - ✅ 终端开发
    - ✅ 命令行增强

---

**文档版本**: 1.1  
**最后更新**: 2026-02-27  
**更新内容**: 添加 Core Agent Loop 详细分析、D-Mail 时间旅行机制、关键代码片段
