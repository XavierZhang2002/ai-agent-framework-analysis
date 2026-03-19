# OpenManus - 深度技术调研

!!! info "项目概览"
    **调研日期**: 2026-02-26  
    **仓库**: https://github.com/FoundationAgents/OpenManus  
    **许可证**: MIT  
    **主要语言**: Python  
    **定位**: 快速复现通用 Manus 风格 Agent，强调可用性和社区共建  

---

## 1. 项目概述

### 1.1 核心定位

OpenManus 是一个 **Python 主体的可运行框架**，旨在快速复现 Manus 风格的通用 Agent，特点：
- ✅ 基于 ReAct 模式的 Agent 实现
- ✅ MCP 客户端与服务器双角色
- ✅ 多 Agent 流程系统（实验性）
- ✅ 丰富的工具集成
- ✅ 浏览器自动化支持

### 1.2 三种运行模式

1. **main.py**: 基础 Manus agent
2. **run_mcp.py**: MCP-enabled agent（支持 stdio/SSE）
3. **run_flow.py**: Multi-agent flow 系统（标注为不稳定）

---

## 2. 代码架构深度分析

### 2.1 目录结构

```
OpenManus/
├── main.py                    # 基础 agent 启动器
├── run_mcp.py                 # MCP agent 启动器 (116 lines)
├── run_flow.py                # Multi-agent flow 启动器 (52 lines)
├── run_mcp_server.py          # MCP server 启动器
├── app/
│   ├── agent/                 # Agent 实现
│   │   ├── base.py           # BaseAgent 抽象类 (196 lines)
│   │   ├── react.py          # ReAct agent (38 lines)
│   │   ├── toolcall.py       # Tool-calling agent (250 lines)
│   │   ├── manus.py          # 主 Manus agent (165 lines)
│   │   ├── mcp.py            # MCP-specific agent (185 lines)
│   │   ├── browser.py        # Browser context helper
│   │   ├── data_analysis.py  # Data analysis agent
│   │   └── swe.py            # Software engineering agent
│   ├── flow/                  # Multi-agent flow 系统
│   │   ├── base.py           # BaseFlow 抽象 (57 lines)
│   │   ├── flow_factory.py   # Flow creation factory (30 lines)
│   │   └── planning.py       # Planning flow (442 lines)
│   ├── tool/                  # 工具实现
│   │   ├── base.py           # BaseTool 抽象类 (181 lines)
│   │   ├── tool_collection.py # 工具集合管理器 (71 lines)
│   │   ├── mcp.py            # MCP client tools (194 lines)
│   │   ├── python_execute.py # Python 执行
│   │   ├── str_replace_editor.py # 文件编辑 (432 lines)
│   │   ├── planning.py       # 规划工具 (363 lines)
│   │   ├── bash.py           # Bash 执行
│   │   ├── browser_use_tool.py # 浏览器自动化
│   │   └── search/           # Web 搜索工具
│   ├── mcp/                   # MCP server 实现
│   │   └── server.py         # FastMCP server (180 lines)
│   ├── prompt/                # Agent prompts
│   │   ├── manus.py
│   │   ├── mcp.py
│   │   ├── planning.py
│   │   └── toolcall.py
│   ├── sandbox/               # Sandbox 执行
│   ├── llm.py                 # LLM 接口 (766 lines)
│   ├── config.py              # 配置管理 (372 lines)
│   ├── schema.py              # 数据模型 (187 lines)
│   └── logger.py              # 日志工具
├── config/
│   ├── config.example.toml    # 主配置 (113 lines)
│   └── mcp.example.json       # MCP 服务器配置 (8 lines)
└── protocol/a2a/              # Agent-to-Agent 协议
```

### 2.2 Agent 继承链

```
BaseAgent (base.py)
    ↓
ReActAgent (react.py)
    ↓
ToolCallAgent (toolcall.py)
    ↓
Manus (manus.py)
```

**BaseAgent** (`app/agent/base.py:13-196`):
- **核心属性**: name, description, llm, memory, state
- **状态管理**: IDLE → RUNNING → FINISHED → ERROR
- **主循环**: `run()` 方法 (Lines 116-154)
- **抽象方法**: `step()` 必须由子类实现

**关键代码** (`app/agent/base.py:116-154`):
```python
async def run(self, request: Optional[str] = None) -> str:
    """Execute the agent's main loop asynchronously."""
    if self.state != AgentState.IDLE:
        raise RuntimeError(f"Cannot run agent from state: {self.state}")

    if request:
        self.update_memory("user", request)

    results: List[str] = []
    async with self.state_context(AgentState.RUNNING):
        while (
            self.current_step < self.max_steps and self.state != AgentState.FINISHED
        ):
            self.current_step += 1
            logger.info(f"Executing step {self.current_step}/{self.max_steps}")
            step_result = await self.step()

            # Check for stuck state
            if self.is_stuck():
                self.handle_stuck_state()

            results.append(f"Step {self.current_step}: {step_result}")

        if self.current_step >= self.max_steps:
            self.current_step = 0
            self.state = AgentState.IDLE
            results.append(f"Terminated: Reached max steps ({self.max_steps})")
    await SANDBOX_CLIENT.cleanup()
    return "\n".join(results) if results else "No steps executed"
```

**ReActAgent** (`app/agent/react.py`):
```python
class ReActAgent(BaseAgent, ABC):
    @abstractmethod
    async def think(self) -> bool:
        """Process current state and decide next action"""

    @abstractmethod
    async def act(self) -> str:
        """Execute decided actions"""

    async def step(self) -> str:
        """Execute a single step: think and act."""
        should_act = await self.think()
        if not should_act:
            return "Thinking complete - no action needed"
        return await self.act()
```

**ToolCallAgent** (`app/agent/toolcall.py:18-250`):
- **工具执行引擎**
- **关键方法:**
  - `think()`: 使用 LLM 决定调用哪些工具 (Lines 39-129)
  - `act()`: 执行工具调用并处理结果 (Lines 131-164)
  - `execute_tool()`: 处理单个工具执行 (Lines 166-208)

**关键代码 - 工具执行** (`app/agent/toolcall.py`):
```python
async def think(self) -> bool:
    """Process current state and decide next actions using tools"""
    response = await self.llm.ask_tool(
        messages=self.messages,
        tools=self.available_tools.to_params(),
        tool_choice=self.tool_choices,
    )
    self.tool_calls = response.tool_calls if response else []
    return bool(self.tool_calls)

async def act(self) -> str:
    """Execute tool calls and handle their results"""
    results = []
    for command in self.tool_calls:
        result = await self.execute_tool(command)
        tool_msg = Message.tool_message(
            content=result,
            tool_call_id=command.id,
            name=command.function.name,
        )
        self.memory.add_message(tool_msg)
        results.append(result)
    return "\n\n".join(results)
```

---

## 3. 底层实现原理

### 3.1 Manus Agent - 主实现

**文件**: `app/agent/manus.py:18-165`

**架构**:
```python
class Manus(ToolCallAgent):
    # 内置工具集合
    available_tools: ToolCollection = Field(
        default_factory=lambda: ToolCollection(
            PythonExecute(),
            BrowserUseTool(),
            StrReplaceEditor(),
            AskHuman(),
            Terminate(),  # Lines 34-42
        )
    )
    
    # MCP 客户端用于远程工具
    mcp_clients: MCPClients = Field(default_factory=MCPClients)  # Line 31
    
    # 浏览器上下文助手
    browser_context_helper: Optional[BrowserContextHelper] = None
```

**MCP Server 初始化** (Lines 67-89):
```python
async def initialize_mcp_servers(self) -> None:
    for server_id, server_config in config.mcp_config.servers.items():
        if server_config.type == "sse":
            await self.connect_mcp_server(server_config.url, server_id)
        elif server_config.type == "stdio":
            await self.connect_mcp_server(
                server_config.command,
                server_id,
                use_stdio=True,
                stdio_args=server_config.args,  # Lines 78-84
            )
```

**动态工具管理** (Lines 91-112):
```python
async def connect_mcp_server(self, server_url: str, server_id: str):
    if use_stdio:
        await self.mcp_clients.connect_stdio(
            server_url, stdio_args or [], server_id  # Lines 100-102
        )
    else:
        await self.mcp_clients.connect_sse(server_url, server_id)
    
    # 添加新 MCP 工具到可用工具
    new_tools = [
        tool for tool in self.mcp_clients.tools 
        if tool.server_id == server_id
    ]
    self.available_tools.add_tools(*new_tools)  # Lines 109-112
```

### 3.2 MCP 实现 - Client & Server

#### **MCP Client** (`app/tool/mcp.py`)

**MCPClients 类架构**:
```python
class MCPClients(ToolCollection):
    sessions: Dict[str, ClientSession] = {}
    
    async def connect_sse(self, server_url: str, server_id: str = "") -> None:
        exit_stack = AsyncExitStack()
        streams = await exit_stack.enter_async_context(sse_client(url=server_url))
        session = await exit_stack.enter_async_context(ClientSession(*streams))
        self.sessions[server_id] = session
        
    async def connect_stdio(self, command: str, args: List[str], server_id: str = "") -> None:
        server_params = StdioServerParameters(command=command, args=args)
        transport = await exit_stack.enter_async_context(stdio_client(server_params))
        session = await exit_stack.enter_async_context(ClientSession(*transport))
        self.sessions[server_id] = session
```

**MCPClientTool** - 远程工具代理:
```python
class MCPClientTool(BaseTool):
    session: Optional[ClientSession] = None
    server_id: str = ""
    original_name: str = ""
    
    async def execute(self, **kwargs) -> ToolResult:
        result = await self.session.call_tool(self.original_name, kwargs)
        content_str = ", ".join(
            item.text for item in result.content 
            if isinstance(item, TextContent)
        )
        return ToolResult(output=content_str or "No output returned.")
```

#### **MCP Server** (`app/mcp/server.py`)

**服务器架构**:
```python
class MCPServer:
    def __init__(self, name: str = "openmanus"):
        self.server = FastMCP(name)
        self.tools["bash"] = Bash()
        self.tools["browser"] = BrowserUseTool()
        self.tools["editor"] = StrReplaceEditor()
        self.tools["terminate"] = Terminate()
```

**工具注册**:
```python
def register_tool(self, tool: BaseTool, method_name: Optional[str] = None):
    tool_name = method_name or tool.name
    tool_param = tool.to_param()
    tool_function = tool_param["function"]
    
    # 创建异步包装器
    async def tool_method(**kwargs):
        result = await tool.execute(**kwargs)
        
        if hasattr(result, "model_dump"):
            return json.dumps(result.model_dump())
        elif isinstance(result, dict):
            return json.dumps(result)
        return result
    
    # 从工具参数构建函数签名
    tool_method.__name__ = tool_name
    tool_method.__doc__ = self._build_docstring(tool_function)
    tool_method.__signature__ = self._build_signature(tool_function)
    
    self.server.tool()(tool_method)  # 注册到 FastMCP
```

### 3.3 Flow 系统 - Multi-Agent 编排

#### **BaseFlow** (`app/flow/base.py:9-57`)

```python
class BaseFlow(BaseModel, ABC):
    agents: Dict[str, BaseAgent]
    tools: Optional[List] = None
    primary_agent_key: Optional[str] = None
    
    @property
    def primary_agent(self) -> Optional[BaseAgent]:
        return self.agents.get(self.primary_agent_key)
    
    @abstractmethod
    async def execute(self, input_text: str) -> str:
        """Execute the flow with given input"""
```

#### **PlanningFlow** (`app/flow/planning.py`)

**核心架构**:
```python
class PlanningFlow(BaseFlow):
    llm: LLM = Field(default_factory=lambda: LLM())
    planning_tool: PlanningTool = Field(default_factory=PlanningTool)
    executor_keys: List[str] = Field(default_factory=list)
    active_plan_id: str = Field(default_factory=lambda: f"plan_{int(time.time())}")
    current_step_index: Optional[int] = None
```

**执行流程**:
```python
async def execute(self, input_text: str) -> str:
    if input_text:
        await self._create_initial_plan(input_text)
        
    while True:
        self.current_step_index, step_info = await self._get_current_step_info()
        if self.current_step_index is None:
            result += await self._finalize_plan()
            break
            
        step_type = step_info.get("type") if step_info else None
        executor = self.get_executor(step_type)
        step_result = await self._execute_step(executor, step_info)
        result += step_result + "\n"
```

**Agent 选择** (Lines 77-92):
```python
def get_executor(self, step_type: Optional[str] = None) -> BaseAgent:
    # 如果步骤类型匹配 agent key，使用该 agent
    if step_type and step_type in self.agents:
        return self.agents[step_type]
    
    # 否则使用第一个可用的 executor
    for key in self.executor_keys:
        if key in self.agents:
            return self.agents[key]
    
    # 回退到 primary agent
    return self.primary_agent
```

**使用 LLM 创建计划** (Lines 136-211):
```python
async def _create_initial_plan(self, request: str):
    # 使用 agent 描述构建系统消息
    agents_description = [
        {"name": key.upper(), "description": self.agents[key].description}
        for key in self.executor_keys if key in self.agents
    ]
    
    system_message_content = (
        "You are a planning assistant. Create a concise, actionable plan..."
    )
    
    if len(agents_description) > 1:
        system_message_content += (
            f"\nNow we have {agents_description} agents. "
            "When creating steps, specify agent names using '[agent_name]'."
        )
    
    # 使用 PlanningTool 调用 LLM
    response = await self.llm.ask_tool(
        messages=[user_message],
        system_msgs=[system_message],
        tools=[self.planning_tool.to_param()],
        tool_choice=ToolChoice.AUTO,  # Lines 170-176
    )
```

### 3.4 工具系统架构

#### **BaseTool** (`app/tool/base.py:78-175`)

```python
class BaseTool(ABC, BaseModel):
    name: str
    description: str
    parameters: Optional[dict] = None
    
    @abstractmethod
    async def execute(self, **kwargs) -> Any:
        """使用给定参数执行工具"""
    
    def to_param(self) -> Dict:
        """转换为 OpenAI function calling 格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }
```

**ToolResult 模型** (Lines 38-76):
```python
class ToolResult(BaseModel):
    output: Any = Field(default=None)
    error: Optional[str] = Field(default=None)
    base64_image: Optional[str] = Field(default=None)
    system: Optional[str] = Field(default=None)
    
    def __str__(self):
        return f"Error: {self.error}" if self.error else self.output
```

#### **ToolCollection** (`app/tool/tool_collection.py:9-71`)

```python
class ToolCollection:
    def __init__(self, *tools: BaseTool):
        self.tools = tools
        self.tool_map = {tool.name: tool for tool in tools}
    
    def to_params(self) -> List[Dict[str, Any]]:
        return [tool.to_param() for tool in self.tools]
    
    async def execute(self, *, name: str, tool_input: Dict[str, Any] = None):
        tool = self.tool_map.get(name)
        if not tool:
            return ToolFailure(error=f"Tool {name} is invalid")
        
        result = await tool(**tool_input)
        return result
    
    def add_tools(self, *tools: BaseTool):
        for tool in tools:
            if tool.name in self.tool_map:
                logger.warning(f"Tool {tool.name} already exists, skipping")
                continue
            
            self.tools += (tool,)
            self.tool_map[tool.name] = tool
```

### 3.5 LLM 模块

**文件**: `app/llm.py:174-766`

**Singleton 模式**:
```python
class LLM:
    _instances: Dict[str, "LLM"] = {}
    
    def __new__(cls, config_name: str = "default", llm_config: Optional[LLMSettings] = None):
        if config_name not in cls._instances:
            instance = super().__new__(cls)
            instance.__init__(config_name, llm_config)
            cls._instances[config_name] = instance
        return cls._instances[config_name]
```

**Token 管理** (Lines 238-264):
```python
def update_token_count(self, input_tokens: int, completion_tokens: int = 0):
    self.total_input_tokens += input_tokens
    self.total_completion_tokens += completion_tokens
    logger.info(
        f"Token usage: Input={input_tokens}, Completion={completion_tokens}, "
        f"Cumulative Total={self.total_input_tokens + self.total_completion_tokens}"
    )

def check_token_limit(self, input_tokens: int) -> bool:
    if self.max_input_tokens is not None:
        return (self.total_input_tokens + input_tokens) <= self.max_input_tokens
    return True
```

**工具调用** (Lines 644-766):
```python
async def ask_tool(
    self,
    messages: List[Union[dict, Message]],
    system_msgs: Optional[List[Union[dict, Message]]] = None,
    tools: Optional[List[dict]] = None,
    tool_choice: TOOL_CHOICE_TYPE = ToolChoice.AUTO,
    **kwargs,
) -> ChatCompletionMessage | None:
    # 格式化消息（支持图片）
    # 计算 tokens
    # 检查 token 限制
    # 设置参数
    # 调用 OpenAI API
    
    response: ChatCompletion = await self.client.chat.completions.create(**params)
    
    return response.choices[0].message
```

---

## 4. 扩展开发指南

### 4.1 添加自定义工具

**方法 1: 创建 BaseTool 子类**

```python
# app/tool/custom_tool.py
from app.tool.base import BaseTool, ToolResult

class CustomTool(BaseTool):
    name: str = "custom_tool"
    description: str = "Tool description"
    parameters: dict = {
        "type": "object",
        "properties": {
            "param1": {
                "type": "string",
                "description": "Parameter description",
            },
        },
        "required": ["param1"],
    }
    
    async def execute(self, param1: str, **kwargs) -> ToolResult:
        result = f"Processed: {param1}"
        return ToolResult(output=result)

# 添加到 Manus agent
from app.tool.custom_tool import CustomTool

available_tools: ToolCollection = Field(
    default_factory=lambda: ToolCollection(
        PythonExecute(),
        CustomTool(),  # 在这里添加工具
        Terminate(),
    )
)
```

### 4.2 配置 MCP Servers

**编辑 `config/mcp.json`**:

```json
{
    "mcpServers": {
        "filesystem": {
            "type": "stdio",
            "command": "python",
            "args": ["-m", "mcp_server_filesystem"]
        },
        "web_server": {
            "type": "sse",
            "url": "http://localhost:8000/sse"
        }
    }
}
```

### 4.3 定义 Multi-Agent Flows

**创建自定义 Flow**:

```python
# app/flow/custom_flow.py
from app.flow.base import BaseFlow

class CustomFlow(BaseFlow):
    async def execute(self, input_text: str) -> str:
        # 步骤 1: 使用 primary agent 进行初始分析
        analysis = await self.primary_agent.run(
            f"Analyze this request: {input_text}"
        )
        
        # 步骤 2: 路由到专门的 agent
        if "data" in input_text.lower():
            specialist = self.get_agent("data_analysis")
            result = await specialist.run(analysis)
        else:
            specialist = self.get_agent("manus")
            result = await specialist.run(analysis)
        
        return result

# 在 FlowFactory 中注册
class FlowType(str, Enum):
    PLANNING = "planning"
    CUSTOM = "custom"

flows = {
    FlowType.PLANNING: PlanningFlow,
    FlowType.CUSTOM: CustomFlow,
}
```

---

## 5. 评估与建议

!!! success "核心优势"
    1. ✅ **简单易用**: Python + 清晰的继承链
    2. ✅ **MCP 双角色**: 既是客户端也是服务器
    3. ✅ **多 Agent 支持**: Flow 系统（虽然不稳定）
    4. ✅ **工具丰富**: 内置多种工具 + MCP 扩展

!!! warning "局限性"
    1. ⚠️ **Flow 系统不稳定**: `run_flow.py` 标注为不稳定
    2. ❌ **无内置 Session 管理**: 需手动管理历史
    3. ❌ **无 Tracing**: 需自行实现
    4. ⚠️ **文档不完整**: 部分功能缺少文档

!!! tip "推荐场景"
    - ✅ 快速搭建通用任务 Agent
    - ✅ MCP 工具实验
    - ✅ 多 Agent 研究（接受不稳定）

!!! danger "不推荐场景"
    - ❌ 生产级应用（Flow 系统不稳定）
    - ❌ 需要企业级特性（session、tracing）

---

## 6. 关键文件清单

### Core Implementation Files

| 文件路径 | 描述 |
|---------|------|
| `app/agent/base.py` | 核心 Agent 循环，BaseAgent 抽象类 |
| `app/agent/react.py` | ReAct 模式实现 |
| `app/agent/toolcall.py` | 工具调用 Agent 实现 |
| `app/mcp/server.py` | MCP 服务器实现 |
| `app/tool/mcp.py` | MCP 客户端实现 |
| `app/flow/planning.py` | Flow 编排系统 |
| `app/tool/tool_collection.py` | 工具集合管理 |

### Agent 继承链

```
BaseAgent (app/agent/base.py)
    ↓
ReActAgent (app/agent/react.py)
    ↓
ToolCallAgent (app/agent/toolcall.py)
    ↓
Manus (app/agent/manus.py)
```

---

**文档版本**: 1.1  
**最后更新**: 2026-02-27
