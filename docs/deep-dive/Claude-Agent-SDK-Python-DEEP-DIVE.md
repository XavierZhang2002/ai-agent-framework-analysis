# Claude Agent SDK (Python) - 深度技术调研

!!! info "项目概览"
    **调研日期**: 2026-02-26  
    **仓库**: https://github.com/anthropics/claude-agent-sdk-python  
    **许可证**: MIT (repository) + Anthropic Commercial Terms  
    **版本**: v0.1.44  
    **Stars**: 5k | **Forks**: 667 | **Contributors**: 47  

---

## 目录

1. [项目概述](#1-项目概述)
2. [代码架构深度分析](#2-代码架构深度分析)
3. [底层实现原理](#3-底层实现原理)
4. [与竞品对比](#4-与竞品对比)
5. [扩展开发指南](#5-扩展开发指南)
6. [生产部署考量](#6-生产部署考量)
7. [评估与建议](#7-评估与建议)

---

## 1. 项目概述

### 1.1 定位与特点

Claude Agent SDK (Python) 是 Anthropic 官方提供的 Python SDK，用于与 **Claude Code CLI** 进行交互，构建基于 Claude 的 agent 应用。

**核心特点**:
- ✅ **SDK + CLI 架构**: Python SDK 作为接口层，Claude Code CLI 作为执行引擎
- ✅ **Hooks 系统**: 在 agent 生命周期关键点提供可编程控制
- ✅ **SDK MCP Servers**: 将 Python 函数作为 in-process MCP 服务器运行（无需子进程）
- ✅ **双向通信**: ClaudeSDKClient 支持交互式对话
- ✅ **自动 CLI 捆绑**: 安装时自动下载并打包 Claude Code CLI

**架构定位**:
```
用户 Python 应用
    ↓
Claude Agent SDK (Python)
    ↓
Claude Code CLI (捆绑的二进制文件)
    ↓
Anthropic API 服务
```

**重要说明**: 这是一个 **混合开源架构**：
- SDK 层代码：MIT 开源
- Claude Code CLI：闭源（bundled binary）
- 服务使用：受 Anthropic Commercial Terms 约束

### 1.2 核心组件

**两种 API 模式**:

1. **`query()`**: 单次查询（类似函数调用）
   ```python
   async for message in query(prompt="Hello"):
       print(message)
   ```

2. **`ClaudeSDKClient`**: 交互式会话（双向通信）
   ```python
   async with ClaudeSDKClient(options) as client:
       await client.query("First message")
       async for msg in client.receive_response():
           print(msg)
       
       await client.query("Follow-up question")
   ```

**三大扩展机制**:

1. **Custom Tools (SDK MCP Servers)**: 
   - 在 Python 进程内运行的 MCP 服务器
   - 无需启动子进程
   - 直接调用 Python 函数

2. **Hooks**: 
   - 在特定时机执行自定义逻辑
   - 10+ 种 hook 事件（PreToolUse, PostToolUse 等）
   - 可修改工具输入、拒绝执行、添加上下文

3. **Permission Callbacks**: 
   - 细粒度工具权限控制
   - 动态批准/拒绝工具调用

---

## 2. 代码架构深度分析

### 2.1 目录结构与模块职责

```
claude-agent-sdk-python/
├── src/claude_agent_sdk/          # 主 SDK 包
│   ├── __init__.py                # 公开 API 导出
│   ├── query.py                   # query() 函数 (126 lines)
│   ├── client.py                  # ClaudeSDKClient 类 (414 lines)
│   ├── types.py                   # 类型定义 (859 lines)
│   ├── _errors.py                 # 异常类
│   ├── _version.py                # SDK 版本
│   ├── _cli_version.py            # 捆绑的 CLI 版本
│   ├── _bundled/                  # 捆绑的 Claude Code CLI 二进制
│   └── _internal/                 # 内部实现
│       ├── client.py              # 内部客户端逻辑
│       ├── query.py               # Query 控制协议处理器 (634 lines)
│       ├── message_parser.py      # 消息解析
│       └── transport/             # 传输层
│           ├── __init__.py        # Transport 抽象类
│           └── subprocess_cli.py  # CLI 子进程传输 (629 lines)
├── examples/                      # 使用示例
│   ├── quick_start.py
│   ├── streaming_mode.py
│   ├── hooks.py
│   └── mcp_calculator.py
├── tests/                         # 单元测试
├── e2e-tests/                     # 端到端测试
└── scripts/                       # 构建与维护脚本
    └── build_wheel.py             # 打包脚本（下载 CLI）
```

### 2.2 核心类关系图

```
┌─────────────────────────────────────────────────────────────┐
│                      Public API Layer                        │
├─────────────────────────────────────────────────────────────┤
│  query() ──────────┐                                         │
│                    │                                         │
│  ClaudeSDKClient ──┼───────────────────────────────────────┐ │
│                    │                                       │ │
└────────────────────┼───────────────────────────────────────┼─┘
                     │                                       │
                     ▼                                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   Internal Layer                             │
├─────────────────────────────────────────────────────────────┤
│  InternalClient.process_query()                              │
│         │                                                    │
│         ├──► Query (Control Protocol Handler)                │
│         │      - Hook callbacks                              │
│         │      - Tool permissions                            │
│         │      - SDK MCP server routing                      │
│         │      - Bidirectional control messages              │
│         │                                                    │
│         └──► Transport (I/O Abstraction)                     │
│                  │                                           │
│                  └──► SubprocessCLITransport                 │
│                        - CLI subprocess management           │
│                        - stdin/stdout/stderr streams         │
│                        - Command building                    │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│               Claude Code CLI Process                        │
│  (Bundled in _bundled/ or system-installed)                 │
│  - Agent loop execution                                      │
│  - Tool orchestration                                        │
│  - MCP server management                                     │
│  - Message streaming (JSON lines)                            │
└─────────────────────────────────────────────────────────────┘
```

**关键关系**:
1. **query() → InternalClient**: 委托模式，简化 API
2. **ClaudeSDKClient → Query**: 组合模式，管理 Query 实例生命周期
3. **Query ↔ Transport**: 桥接模式，抽象 I/O 层
4. **SubprocessCLITransport → CLI**: 进程管理，标准输入输出通信

### 2.3 关键设计模式

| 模式 | 应用场景 | 代码位置 |
|------|---------|---------|
| **Abstract Factory** | Transport 抽象允许自定义传输实现 | `_internal/transport/__init__.py` |
| **Strategy** | `can_use_tool` 和 `hooks` 回调提供可定制行为 | `types.py`, `query.py` |
| **Bridge** | Query 桥接 Transport 和控制协议 | `_internal/query.py` |
| **Observer** | AsyncIterator 消息流传播事件 | `query.py`, `client.py` |
| **Command** | Control requests 封装操作 | `_internal/query.py:233-342` |
| **Async Context Manager** | `ClaudeSDKClient.__aenter__/__aexit__` | `client.py:87-177` |
| **Adapter** | SDK MCP servers 将 Python 函数适配为 MCP 协议 | `__init__.py:157-319` |

---

## 3. 底层实现原理

### 3.1 query() 工作流程

**文件**: `src/claude_agent_sdk/query.py:12-126`

```python
async def query(
    *,
    prompt: str | AsyncIterable[dict[str, Any]],
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None,
) -> AsyncIterator[Message]:
```

**执行流程**:

1. **环境设置** (Line 119):
   ```python
   os.environ["CLAUDE_CODE_ENTRYPOINT"] = "sdk-py"
   ```

2. **客户端创建** (Line 121):
   ```python
   client = InternalClient()
   ```

3. **委托处理** (Lines 123-125):
   ```python
   async for message in client.process_query(
       prompt=prompt, options=options, transport=transport
   ):
       yield message
   ```

**内部处理** (`_internal/client.py:43-146`):

1. **权限验证** (Lines 52-69):
   ```python
   if configured_options.can_use_tool is not None:
       if not isinstance(prompt, AsyncIterable):
           raise ValueError("can_use_tool requires AsyncIterable prompt")
       if configured_options.permission_prompt_tool_name is not None:
           raise ValueError("cannot set both can_use_tool and permission_prompt_tool_name")
       configured_options.permission_prompt_tool_name = "stdio"
   ```

2. **Transport 选择** (Lines 72-78):
   ```python
   chosen_transport = transport if transport else SubprocessCLITransport(
       prompt=prompt,
       options=configured_options,
   )
   ```

3. **连接 Transport** (Line 81):
   ```python
   await chosen_transport.connect()  # 启动 CLI 子进程
   ```

4. **提取 SDK MCP Servers** (Lines 84-90):
   ```python
   sdk_mcp_servers = {}
   for name, config in configured_options.mcp_servers.items():
       if config.get("type") == "sdk":
           sdk_mcp_servers[name] = config["instance"]
   ```

5. **创建 Query** (Lines 103-112):
   ```python
   query = Query(
       transport=chosen_transport,
       is_streaming_mode=True,  # 内部总是流式！
       can_use_tool=configured_options.can_use_tool,
       hooks=self._convert_hooks_to_internal_format(...),
       sdk_mcp_servers=sdk_mcp_servers,
       agents=agents_dict,
   )
   ```

6. **初始化控制协议** (Lines 116-119):
   ```python
   await query.start()      # 启动后台消息读取器
   await query.initialize()  # 发送 initialize 控制请求
   ```

7. **输入流式传输** (Lines 122-137):
   - **字符串提示**: 写入 JSON 用户消息到 stdin，关闭 stdin
   - **AsyncIterable 提示**: 在后台任务中流式传输消息

8. **消息产出** (Lines 140-143):
   ```python
   async for data in query.receive_messages():
       message = parse_message(data)
       if message is not None:
           yield message
   ```

### 3.2 ClaudeSDKClient 双向通信管理

**文件**: `src/claude_agent_sdk/client.py:14-414`

#### 连接建立 (`connect()` 方法, lines 87-177):

1. **空流创建** (Lines 96-101):
   ```python
   async def _empty_stream() -> AsyncIterator[dict[str, Any]]:
       return  # 永不产出，但保持连接开放
       yield {}  # 不可达但使其成为 async generator
   ```

2. **权限配置** (Lines 106-124):
   ```python
   if self.options.can_use_tool is not None:
       if self.options.permission_prompt_tool_name is not None:
           raise ValueError(...)
       self.options.permission_prompt_tool_name = "stdio"
   ```

3. **Transport 初始化** (Lines 127-134):
   ```python
   if self._custom_transport:
       self._transport = self._custom_transport
   else:
       self._transport = SubprocessCLITransport(
           prompt=actual_prompt,
           options=options,
       )
   await self._transport.connect()
   ```

4. **Query 创建** (Lines 159-169):
   ```python
   self._query = Query(
       transport=self._transport,
       is_streaming_mode=True,
       can_use_tool=self.options.can_use_tool,
       hooks=self._convert_hooks_to_internal_format(self.options.hooks),
       sdk_mcp_servers=sdk_mcp_servers,
       initialize_timeout=initialize_timeout,
       agents=agents_dict,
   )
   ```

5. **后台消息读取器** (Lines 172-173):
   ```python
   await self._query.start()      # 创建 anyio.TaskGroup
   await self._query.initialize()  # 发送 initialize 请求
   ```

6. **输入流式传输** (Lines 176-177):
   ```python
   if prompt is not None and isinstance(prompt, AsyncIterable):
       self._query._tg.start_soon(self._query.stream_input, prompt)
   ```

#### 消息流架构:

```
┌────────────────────────────────────────────────────┐
│         ClaudeSDKClient (Public API)                │
├────────────────────────────────────────────────────┤
│  query(prompt) ──► Transport.write(json + "\n")    │
│                                                     │
│  receive_messages() ──► Query.receive_messages()   │
│         ▲                      │                    │
│         │                      │                    │
│         │                      ▼                    │
│         │         ┌────────────────────────┐        │
│         │         │   Memory Stream        │        │
│         │         │   (anyio.MemoryStream) │        │
│         │         └────────────────────────┘        │
│         │                      ▲                    │
│         │                      │                    │
└─────────┼──────────────────────┼────────────────────┘
          │                      │
          │         ┌────────────┴────────────┐
          │         │  Query._read_messages() │
          │         │  (Background Task)       │
          │         └─────────────────────────┘
          │                      ▲
          │                      │
          ▼                      │
┌────────────────────────────────────────────────────┐
│             Transport Layer                         │
├────────────────────────────────────────────────────┤
│  write() ──► CLI stdin                             │
│  read_messages() ──► CLI stdout (JSON lines)       │
└────────────────────────────────────────────────────┘
```

#### 控制协议消息路由 (`_internal/query.py:172-232`):

```python
async def _read_messages(self) -> None:
    async for message in self.transport.read_messages():
        msg_type = message.get("type")
        
        # 路由控制消息
        if msg_type == "control_response":
            # 匹配待处理请求，设置事件
            request_id = message.get("response", {}).get("request_id")
            event = self.pending_control_responses[request_id]
            self.pending_control_results[request_id] = response
            event.set()  # 唤醒等待的 _send_control_request()
            
        elif msg_type == "control_request":
            # 处理传入控制请求（hook、权限、MCP）
            self._tg.start_soon(self._handle_control_request, request)
            
        # 常规 SDK 消息进入流
        await self._message_send.send(message)
```

### 3.3 Claude Code CLI 集成（子进程管理）

**文件**: `src/claude_agent_sdk/_internal/transport/subprocess_cli.py:335-410`

#### 子进程生命周期:

**1. CLI 发现** (`_find_cli()`, lines 64-95):
```python
def _find_cli(self) -> str:
    # 1. 检查捆绑的 CLI
    bundled_cli = self._find_bundled_cli()  # Lines 97-110
    if bundled_cli:
        return bundled_cli
    
    # 2. 检查系统 PATH
    if cli := shutil.which("claude"):
        return cli
    
    # 3. 检查常见位置
    locations = [
        Path.home() / ".npm-global/bin/claude",
        Path("/usr/local/bin/claude"),
        # ... 更多位置
    ]
    
    # 4. 如果未找到则抛出错误
    raise CLINotFoundError(...)
```

**2. 命令构建** (`_build_command()`, lines 166-333):

关键标志:
- `--output-format stream-json`: JSON 流式输出
- `--verbose`: 详细日志
- `--input-format stream-json`: 在 stdin 上接受 JSON 消息
- `--system-prompt`, `--tools`, `--allowed-tools` 等
- `--mcp-config`: MCP 服务器配置（过滤 SDK 服务器）
- `--permission-mode`, `--permission-prompt-tool`: 权限设置
- `--max-turns`, `--max-budget-usd`: 执行限制

**3. 进程启动** (`connect()`, lines 335-410):
```python
async def connect(self) -> None:
    # 版本检查
    await self._check_claude_version()  # Lines 587-625
    
    cmd = self._build_command()
    
    # 环境设置
    process_env = {
        **os.environ,
        **self._options.env,
        "CLAUDE_CODE_ENTRYPOINT": "sdk-py",
        "CLAUDE_AGENT_SDK_VERSION": __version__,
    }
    
    if self._options.enable_file_checkpointing:
        process_env["CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING"] = "true"
    
    # 使用 anyio 启动进程
    self._process = await anyio.open_process(
        cmd,
        stdin=PIPE,
        stdout=PIPE,
        stderr=PIPE if should_pipe_stderr else None,
        cwd=self._cwd,
        env=process_env,
    )
    
    # 包装流
    self._stdout_stream = TextReceiveStream(self._process.stdout)
    self._stdin_stream = TextSendStream(self._process.stdin)
    
    # 在后台启动 stderr 处理器
    if should_pipe_stderr:
        self._stderr_task_group.start_soon(self._handle_stderr)
```

**4. 消息读取** (`_read_messages_impl()`, lines 519-586):

```python
async def _read_messages_impl(self) -> AsyncIterator[dict[str, Any]]:
    json_buffer = ""
    
    async for line in self._stdout_stream:
        # 累积部分 JSON
        for json_line in line.strip().split("\n"):
            json_buffer += json_line
            
            # 检查缓冲区大小限制
            if len(json_buffer) > self._max_buffer_size:
                raise SDKJSONDecodeError(...)
            
            # 尝试解析
            try:
                data = json.loads(json_buffer)
                json_buffer = ""
                yield data
            except json.JSONDecodeError:
                continue  # 继续累积
    
    # 检查退出码
    returncode = await self._process.wait()
    if returncode != 0:
        raise ProcessError(...)
```

**5. 进程清理** (`close()`, lines 440-479):
```python
async def close(self) -> None:
    # 关闭 stderr 任务
    if self._stderr_task_group:
        self._stderr_task_group.cancel_scope.cancel()
        await self._stderr_task_group.__aexit__(None, None, None)
    
    # 关闭 stdin（带锁）
    async with self._write_lock:
        if self._stdin_stream:
            await self._stdin_stream.aclose()
    
    # 终止进程
    if self._process.returncode is None:
        self._process.terminate()
        await self._process.wait()
```

### 3.4 SDK MCP Servers 工作原理（In-Process）

**定义** (`__init__.py:157-319`):

#### Tool 装饰器:
```python
@tool("add", "Add two numbers", {"a": float, "b": float})
async def add_numbers(args: dict[str, Any]) -> dict[str, Any]:
    result = args["a"] + args["b"]
    return {
        "content": [{"type": "text", "text": f"Result: {result}"}]
    }
```

#### 服务器创建:
```python
def create_sdk_mcp_server(
    name: str, 
    version: str = "1.0.0", 
    tools: list[SdkMcpTool[Any]] | None = None
) -> McpSdkServerConfig:
    from mcp.server import Server
    
    server = Server(name, version=version)
    
    if tools:
        tool_map = {tool_def.name: tool_def for tool_def in tools}
        
        @server.list_tools()
        async def list_tools() -> list[Tool]:
            # 将工具定义转换为 MCP Tool 格式
            # 处理 schema 转换: dict → JSON Schema
            ...
        
        @server.call_tool()
        async def call_tool(name: str, arguments: dict) -> Any:
            tool_def = tool_map[name]
            result = await tool_def.handler(arguments)
            # 将结果转换为 MCP 内容格式
            ...
    
    return McpSdkServerConfig(type="sdk", name=name, instance=server)
```

#### JSONRPC 路由 (`_internal/query.py:391-527`):

当 CLI 发送 MCP 请求时:
```python
async def _handle_sdk_mcp_request(
    self, server_name: str, message: dict[str, Any]
) -> dict[str, Any]:
    server = self.sdk_mcp_servers[server_name]
    method = message.get("method")
    
    if method == "initialize":
        return {
            "jsonrpc": "2.0",
            "id": message.get("id"),
            "result": {
                "protocolVersion": "2024-11-05",
                "capabilities": {"tools": {}},
                "serverInfo": {"name": server.name, ...}
            }
        }
    
    elif method == "tools/list":
        request = ListToolsRequest(method=method)
        handler = server.request_handlers.get(ListToolsRequest)
        result = await handler(request)
        # 转换 MCP 结果为 JSONRPC
        ...
    
    elif method == "tools/call":
        call_request = CallToolRequest(...)
        handler = server.request_handlers.get(CallToolRequest)
        result = await handler(call_request)
        # 转换为 JSONRPC 响应
        ...
```

**关键洞察**: SDK 使用手动 JSONRPC 路由，因为 Python MCP SDK 缺少 TypeScript 版本的 `Transport` 抽象。见 lines 422-427 的注释。

### 3.5 Hooks 实现与调用

**Hook 配置** (`client.py:69-85`):

```python
def _convert_hooks_to_internal_format(
    self, hooks: dict[HookEvent, list[HookMatcher]]
) -> dict[str, list[dict[str, Any]]]:
    internal_hooks: dict[str, list[dict[str, Any]]] = {}
    for event, matchers in hooks.items():
        internal_hooks[event] = []
        for matcher in matchers:
            internal_matcher = {
                "matcher": matcher.matcher,
                "hooks": matcher.hooks,  # 回调函数列表
            }
            if matcher.timeout is not None:
                internal_matcher["timeout"] = matcher.timeout
            internal_hooks[event].append(internal_matcher)
    return internal_hooks
```

**Hook 注册** (`_internal/query.py:119-163`):

```python
async def initialize(self) -> dict[str, Any] | None:
    hooks_config: dict[str, Any] = {}
    if self.hooks:
        for event, matchers in self.hooks.items():
            hooks_config[event] = []
            for matcher in matchers:
                callback_ids = []
                for callback in matcher.get("hooks", []):
                    # 为每个回调分配唯一 ID
                    callback_id = f"hook_{self.next_callback_id}"
                    self.next_callback_id += 1
                    self.hook_callbacks[callback_id] = callback
                    callback_ids.append(callback_id)
                
                hook_matcher_config = {
                    "matcher": matcher.get("matcher"),
                    "hookCallbackIds": callback_ids,
                }
                if matcher.get("timeout") is not None:
                    hook_matcher_config["timeout"] = matcher.get("timeout")
                hooks_config[event].append(hook_matcher_config)
    
    # 发送带 hook 配置的 initialize 请求
    request = {
        "subtype": "initialize",
        "hooks": hooks_config if hooks_config else None,
    }
    response = await self._send_control_request(request, timeout=60.0)
```

**Hook 调用** (`_internal/query.py:286-299`):

```python
async def _handle_control_request(self, request: SDKControlRequest) -> None:
    request_data = request["request"]
    
    if subtype == "hook_callback":
        callback_id = request_data["callback_id"]
        callback = self.hook_callbacks.get(callback_id)
        
        # 调用用户的 hook 函数
        hook_output = await callback(
            request_data.get("input"),      # HookInput
            request_data.get("tool_use_id"),
            {"signal": None},               # HookContext
        )
        
        # 转换 Python 安全名称（async_, continue_）为 CLI 名称（async, continue）
        response_data = _convert_hook_output_for_cli(hook_output)
        
        # 将响应发回 CLI
        await self.transport.write(json.dumps({
            "type": "control_response",
            "response": {
                "subtype": "success",
                "request_id": request_id,
                "response": response_data,
            }
        }) + "\n")
```

**Python 字段名转换** (`_internal/query.py:34-50`):

```python
def _convert_hook_output_for_cli(hook_output: dict[str, Any]) -> dict[str, Any]:
    """将 Python 安全字段名转换为 CLI 期望的字段名"""
    converted = {}
    for key, value in hook_output.items():
        if key == "async_":
            converted["async"] = value  # Python 关键字 → JS 名称
        elif key == "continue_":
            converted["continue"] = value  # Python 关键字 → JS 名称
        else:
            converted[key] = value
    return converted
```

### 3.6 工具执行流程

**完整流程**:

1. **用户查询**: `client.query("Run bash command: ls")`

2. **CLI 决定使用工具**: Claude Code 决定调用 "Bash" 工具

3. **权限请求**（如果配置了 `can_use_tool` 回调）:
   ```
   CLI → SDK: control_request
   {
     "type": "control_request",
     "request_id": "req_123",
     "request": {
       "subtype": "can_use_tool",
       "tool_name": "Bash",
       "input": {"command": "ls"},
       "permission_suggestions": [...]
     }
   }
   
   SDK → 用户回调: can_use_tool("Bash", {"command": "ls"}, context)
   用户回调 → SDK: PermissionResultAllow() 或 PermissionResultDeny(...)
   
   SDK → CLI: control_response
   {
     "type": "control_response",
     "response": {
       "subtype": "success",
       "request_id": "req_123",
       "response": {
         "behavior": "allow",
         "updatedInput": {...}
       }
     }
   }
   ```

4. **PreToolUse Hook**（如果配置）:
   ```
   CLI → SDK: control_request with subtype="hook_callback"
   SDK → 用户 Hook: PreToolUseHookInput
   用户 Hook → SDK: HookJSONOutput (permissionDecision, additionalContext 等)
   SDK → CLI: control_response with hook output
   ```

5. **工具执行**:
   - 对于 SDK MCP 工具: `_handle_sdk_mcp_request()` 调用进程内处理器
   - 对于外部工具: CLI 处理执行

6. **PostToolUse Hook**（如果配置）:
   ```
   CLI → SDK: control_request with tool_response data
   SDK → 用户 Hook: PostToolUseHookInput
   用户 Hook → SDK: HookJSONOutput
   ```

7. **消息流式传输**:
   ```
   CLI → SDK: {"type": "assistant", "message": {...}}
   SDK → 用户: AssistantMessage with ToolUseBlock
   
   CLI → SDK: {"type": "user", "message": {"content": [ToolResultBlock]}}
   SDK → 用户: UserMessage with ToolResultBlock
   ```

---

## 4. 与竞品对比

### 4.1 Claude Agent SDK vs. OpenAI Agents SDK

| 维度 | Claude Agent SDK | OpenAI Agents SDK |
|------|-----------------|-------------------|
| **架构模式** | SDK + 闭源 CLI | 纯 Python SDK |
| **Provider 绑定** | ❌ Claude-only | ✅ Provider-agnostic (100+ LLMs) |
| **Session 管理** | ❌ 无内置 | ✅ 内置（SQLite/Redis） |
| **Tracing** | ❌ 无内置 | ✅ 内置（多后端支持） |
| **Handoff 机制** | ✅ 通过 subagents（文档化） | ✅ 一等公民（Handoff 类） |
| **HITL 支持** | ✅ Hooks + permission_mode | ✅ RunState + interruptions |
| **工具定义** | `@tool` + SDK MCP Server | `@function_tool` |
| **MCP 集成** | ✅ 原生（in-process SDK MCP） | ❌ 无（但支持外部 MCP via tools） |
| **CLI 依赖** | ✅ 依赖 Claude Code CLI（bundled） | ❌ 无 |
| **License** | MIT + Anthropic ToS | MIT（纯开源） |
| **架构透明度** | ⚠️ SDK 开源，CLI 闭源 | ✅ 完全开源 |
| **Stars** | 5k | 19.2k |

**代码对比 - 基础 Agent**:

```python
# ═══════════════════════════════════════════════════════════
# Claude Agent SDK
# ═══════════════════════════════════════════════════════════
from claude_agent_sdk import query, ClaudeAgentOptions

options = ClaudeAgentOptions(
    system_prompt="You are helpful",
    max_turns=1
)

async for message in query(prompt="Hello", options=options):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)

# ═══════════════════════════════════════════════════════════
# OpenAI Agents SDK
# ═══════════════════════════════════════════════════════════
from agents import Agent, Runner

agent = Agent(
    name="Assistant",
    instructions="You are helpful",
    model="gpt-4o"  # 或 "claude-3-5-sonnet", "gemini-pro" 等
)

result = await Runner.run(agent, "Hello")
print(result.final_output)
```

### 4.2 Feature Matrix

| Feature | Claude Agent SDK | OpenAI Agents SDK | LangChain |
|---------|-----------------|-------------------|-----------|
| **Multi-agent** | ✅ (via subagents) | ✅ ✅ ✅ (first-class) | ✅ ✅ |
| **Hooks** | ✅ ✅ ✅ (10+ events) | ⚠️ (仅基础 hooks) | ✅ (Callbacks) |
| **Sessions** | ❌ | ✅ ✅ ✅ | ✅ ✅ |
| **Tracing** | ❌ | ✅ ✅ ✅ | ✅ (LangSmith) |
| **MCP Support** | ✅ ✅ ✅ (native in-process) | ⚠️ (via tools) | ❌ |
| **Streaming** | ✅ ✅ | ✅ ✅ | ✅ |
| **Provider-agnostic** | ❌ | ✅ ✅ ✅ | ✅ ✅ |
| **RAG** | ❌ | ❌ | ✅ ✅ ✅ |
| **License** | MIT + ToS | MIT | MIT |

---

## 5. 扩展开发指南

### 5.1 创建自定义工具（SDK MCP Servers）

**Step 1: 定义工具**

```python
from claude_agent_sdk import tool
from typing import Any

@tool(
    name="fibonacci",
    description="Calculate fibonacci number at position n",
    input_schema={"n": int}
)
async def fibonacci(args: dict[str, Any]) -> dict[str, Any]:
    n = args["n"]
    
    if n < 0:
        return {
            "content": [{"type": "text", "text": "Error: n must be non-negative"}],
            "is_error": True
        }
    
    def fib(x: int) -> int:
        if x <= 1:
            return x
        return fib(x - 1) + fib(x - 2)
    
    result = fib(n)
    return {
        "content": [
            {"type": "text", "text": f"fibonacci({n}) = {result}"}
        ]
    }
```

**Step 2: 创建服务器**

```python
from claude_agent_sdk import create_sdk_mcp_server

server = create_sdk_mcp_server(
    name="math-tools",
    version="1.0.0",
    tools=[fibonacci]
)
```

**Step 3: 使用**

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

options = ClaudeAgentOptions(
    mcp_servers={"math": server},
    allowed_tools=["mcp__math__fibonacci"]
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Calculate fibonacci(10)")
    async for msg in client.receive_response():
        print(msg)
```

### 5.2 实现 Hooks

**PreToolUse Hook（阻止危险命令）**:

```python
from claude_agent_sdk import HookMatcher, ClaudeAgentOptions
from claude_agent_sdk.types import HookInput, HookContext, HookJSONOutput

async def check_bash_safety(
    input_data: HookInput,
    tool_use_id: str | None,
    context: HookContext
) -> HookJSONOutput:
    if input_data["hook_event_name"] != "PreToolUse":
        return {}
    
    tool_name = input_data["tool_name"]
    tool_input = input_data["tool_input"]
    
    if tool_name != "Bash":
        return {}
    
    command = tool_input.get("command", "")
    
    dangerous = ["rm -rf /", "sudo rm", "mkfs", "dd if="]
    for pattern in dangerous:
        if pattern in command:
            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "deny",
                    "permissionDecisionReason": f"Blocked: {pattern}",
                },
                "systemMessage": f"🚫 Dangerous command blocked",
                "reason": f"Command contains: {pattern}"
            }
    
    return {}

options = ClaudeAgentOptions(
    allowed_tools=["Bash"],
    hooks={
        "PreToolUse": [
            HookMatcher(
                matcher="Bash",
                hooks=[check_bash_safety],
                timeout=5.0
            )
        ]
    }
)
```

**PostToolUse Hook（添加上下文）**:

```python
async def review_bash_output(
    input_data: HookInput,
    tool_use_id: str | None,
    context: HookContext
) -> HookJSONOutput:
    if input_data["hook_event_name"] != "PostToolUse":
        return {}
    
    tool_response = input_data.get("tool_response", "")
    
    if "error" in str(tool_response).lower():
        return {
            "hookSpecificOutput": {
                "hookEventName": "PostToolUse",
                "additionalContext": "The command failed. Consider checking syntax."
            },
            "systemMessage": "⚠️ Command error",
        }
    
    return {}
```

### 5.3 权限模式

**配置**:

```python
from claude_agent_sdk import ClaudeAgentOptions

options = ClaudeAgentOptions(
    permission_mode="acceptEdits",  # 自动接受编辑
    allowed_tools=["Write", "Edit", "MultiEdit"]
)
```

**动态更改**:

```python
async with ClaudeSDKClient(options) as client:
    # 默认权限
    await client.query("Analyze codebase")
    async for msg in client.receive_response():
        print(msg)
    
    # 切换到自动接受
    await client.set_permission_mode("acceptEdits")
    await client.query("Implement changes")
```

### 5.4 自定义 Transport

```python
from claude_agent_sdk import Transport
import websockets

class RemoteClaudeTransport(Transport):
    def __init__(self, url: str, api_key: str):
        self.url = url
        self.api_key = api_key
        self.websocket = None
    
    async def connect(self) -> None:
        self.websocket = await websockets.connect(
            self.url,
            extra_headers={"Authorization": f"Bearer {self.api_key}"}
        )
    
    async def write(self, data: str) -> None:
        await self.websocket.send(data)
    
    async def read_messages(self):
        async for message in self.websocket:
            yield json.loads(message)
    
    async def close(self) -> None:
        if self.websocket:
            await self.websocket.close()
```

---

## 6. 生产部署考量

### 6.1 性能优化

**并发控制**:

```python
import asyncio

semaphore = asyncio.Semaphore(10)

async def run_with_limit(prompt):
    async with semaphore:
        async for msg in query(prompt=prompt):
            yield msg
```

**连接池**（对于 ClaudeSDKClient）:

```python
class ClientPool:
    def __init__(self, max_size: int = 10):
        self.pool = []
        self.max_size = max_size
        self.lock = asyncio.Lock()
    
    async def acquire(self):
        async with self.lock:
            if self.pool:
                return self.pool.pop()
            return await ClaudeSDKClient(options)
    
    async def release(self, client):
        async with self.lock:
            if len(self.pool) < self.max_size:
                self.pool.append(client)
            else:
                await client.close()
```

### 6.2 错误处理

```python
from claude_agent_sdk import (
    ClaudeSDKError,
    CLINotFoundError,
    ProcessError,
)
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
)
async def query_with_retry(prompt):
    try:
        async for msg in query(prompt=prompt):
            yield msg
    except ProcessError as e:
        logger.error(f"Process failed: {e.exit_code}")
        raise
    except CLINotFoundError:
        logger.error("Claude Code CLI not found")
        raise
```

### 6.3 监控

```python
from prometheus_client import Counter, Histogram
import time

query_count = Counter("claude_queries_total", "Total queries")
query_latency = Histogram("claude_query_latency_seconds", "Query latency")

async def query_with_metrics(prompt):
    start = time.time()
    query_count.inc()
    
    try:
        async for msg in query(prompt=prompt):
            yield msg
    finally:
        query_latency.observe(time.time() - start)
```

---

## 7. 评估与建议

!!! success "核心优势"
    1. ✅ **强大的 Hook 系统**: 10+ 事件类型，细粒度控制
    2. ✅ **SDK MCP Servers**: 无需子进程的进程内工具
    3. ✅ **官方支持**: Anthropic 官方维护
    4. ✅ **自动 CLI 捆绑**: 零配置安装

!!! warning "局限性"
    1. ❌ **Claude-only**: 无法使用其他 LLM
    2. ❌ **闭源 CLI**: Claude Code CLI 不开源
    3. ❌ **无 Session 管理**: 需手动管理历史
    4. ❌ **无 Tracing**: 需自行实现可观测性
    5. ⚠️ **商业条款**: 受 Anthropic ToS 约束

!!! tip "推荐场景"
    - ✅ 深度集成 Claude 生态
    - ✅ 需要强大的 Hook 系统
    - ✅ 需要 in-process MCP 工具
    - ✅ Python 应用嵌入 Claude 能力

!!! danger "不推荐场景"
    - ❌ 需要跨 LLM provider
    - ❌ 需要完全开源方案
    - ❌ 需要内置 session/tracing

### 7.4 最佳实践

1. 使用 Hooks 实现细粒度控制
2. 利用 SDK MCP Servers 避免子进程开销
3. 实现自定义错误处理和重试
4. 监控 Claude Code CLI 进程健康
5. 定期更新到最新版本（CLI 版本跟踪）

---

## 附录

### A. 关键文件速查

| 文件 | 行数 | 关键内容 |
|------|-----|---------|
| `query.py` | 126 | `query()` 函数 |
| `client.py` | 414 | `ClaudeSDKClient` 类 |
| `_internal/query.py` | 634 | 控制协议处理 |
| `_internal/transport/subprocess_cli.py` | 629 | CLI 子进程管理 |
| `types.py` | 859 | 类型定义 |
| `__init__.py` | 319 | `@tool`, `create_sdk_mcp_server` |

### B. Hook 事件类型

1. `PreToolUse` - 工具执行前
2. `PostToolUse` - 工具执行后（成功）
3. `PostToolUseFailure` - 工具执行后（失败）
4. `UserPromptSubmit` - 用户提交提示时
5. `Stop` - 会话停止
6. `SubagentStop` - 子代理停止
7. `PreCompact` - 上下文压缩前
8. `Notification` - 通知事件
9. `SubagentStart` - 子代理启动
10. `PermissionRequest` - 权限请求

### C. 参考资源

- **GitHub**: https://github.com/anthropics/claude-agent-sdk-python
- **Examples**: https://github.com/anthropics/claude-agent-sdk-python/tree/main/examples
- **Releases**: https://github.com/anthropics/claude-agent-sdk-python/releases

---

**文档版本**: 1.0  
**最后更新**: 2026-02-26
