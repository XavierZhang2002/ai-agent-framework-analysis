# AI Agent 框架的"特性分化"：从代码审计看学术与工业的特性需求差异

**[English Version](BLOG.en.md)** | **中文**

---

> ** TL;DR **: 我审计了 9 个主流 Agent 框架的源码（[OpenAI Agents SDK](deep-dive/OpenAI-Agents-SDK-DEEP-DIVE.md)、[Claude Agent SDK](deep-dive/Claude-Agent-SDK-Python-DEEP-DIVE.md)、[Codex CLI](deep-dive/Codex-CLI-DEEP-DIVE.md)、[OpenCode](deep-dive/OpenCode-DEEP-DIVE.md)、[Kimi CLI](deep-dive/Kimi-CLI-DEEP-DIVE.md)、[Gemini CLI](deep-dive/Gemini-CLI-DEEP-DIVE.md)、[Qwen Code](deep-dive/Qwen-Code-DEEP-DIVE.md)、[SWE-agent](deep-dive/SWE-agent-DEEP-DIVE.md)、[OpenManus](deep-dive/OpenManus-DEEP-DIVE.md)），发现一个反直觉的事实：** 各框架的核心循环（Agent Loop）逻辑高度相似 **。真正的差异不在于"用了什么算法"，而在于** 工程特性的组合选择 **——状态管理、安全控制、协议支持等 30+ 个特性的不同实现。这种"特性分化"导致学术界难以使用最先进的 Agent 能力，形成了一道隐形的鸿沟。

> **阅读导航**：
> - 第 1-3 节：代码审计观察（事实层）
> - 第 4 节：学术与工业特性需求差异的具体表现（事实 + 推断）
> - 第 5 节：未来收敛趋势（作者判断）
> - 第 6 节：选型与实践建议（行动层）
> - [详细框架对比矩阵](COMPARISON.md)
> - [各框架深度源码分析](deep-dive/)

---

## 一、破除迷思：Agent 的核心循环高度稳定

> **重要限定**：本文聚焦**单 Agent 工具调用范式**（Single-Agent Tool-Calling Pattern）。这是当前 9 个开源框架的主流架构。Multi-Agent 编排（如 Planning、Agent 间通信）、Planner-Executor 分离等范式不在本文讨论范围。

### 1.1 9 个框架，同一个循环

让我先展示一个令人惊讶的事实。以下是 9 个框架的核心循环伪代码：

**OpenAI Agents SDK**（Python，生产级 SDK）：
```python
# src/agents/run.py:396-1329
while True:
    response = await model.get_response(messages, tools)
    
    if response.has_tool_calls():
        for tool_call in response.tool_calls:
            result = await execute_tool(tool_call)
            messages.append(tool_result_message(result))
    else:
        return RunResult(final_output=response.content)
```

**Claude Agent SDK**（Python，SDK + 闭源 CLI）：
```python
# _internal/query.py:172-232
while not done:
    response = await cli.communicate(messages)
    
    if response.type == "tool_use":
        # 通过 Hook 系统处理
        result = await handle_hooks("PreToolUse", response.tool)
        observation = await execute_tool(result)
        messages.append(observation)
    elif response.type == "stop":
        done = True
```

**Codex CLI**（Rust，企业级安全）：
```rust
// codex-rs/core/src/codex.rs:500-800
loop {
    let response = self.llm.generate(&context).await?;
    
    match response.finish_reason {
        FinishReason::ToolCalls(calls) => {
            // 安全检查 + 审批
            self.approval_manager.evaluate(&calls).await?;
            let observations = self.execute_tools(calls).await?;
            context.extend(observations);
        }
        FinishReason::Stop => break,
    }
}
```

**SWE-agent**（Python，研究框架）：
```python
# sweagent/agent/agents.py:1265
while not step_output.done:
    step_output = self.step()
    # step() 内部：
    #   thought, action = parse_react_output(llm_response)
    #   observation = execute_action(action)
```

**OpenManus**（Python，快速实验）：
```python
# app/agent/react.py:11-38
while self.current_step < self.max_steps:
    step_result = await self.step()  # think() -> act()
    if self.is_stuck():  # 检测重复响应
        self.handle_stuck_state()
```

**OpenCode**（TypeScript，100% 开源）：
```typescript
// src/session/prompt.ts:274-724
while (true) {
    const result = await processor.process({ messages, tools });
    
    if (result === "stop") break;
    if (result === "compact") await compactContext();  // 特性：自动压缩
    
    // 处理 subtasks、compaction、overflow 等特性
}
```

**看到了吗？** 所有框架都是同一个模式：

```
输入 → 构建上下文 → 调用 LLM → 解析输出 → 
如果有工具调用 → 执行工具 → 添加观察 → 重复
如果没有工具调用 → 完成 → 返回结果
```

这就是 **Agent Loop**，自 2022 年 ReAct 论文发表以来，** 本质逻辑保持高度稳定 **。

### 1.2 那么，差异到底在哪里？

如果核心循环相同，为什么这些框架在实际使用中感觉如此不同？

答案是：** 特性（Capabilities）的组合选择 **。

就像一个操作系统，内核（Loop）大家都用 Linux，但发行版（框架）的差异在于：
- 用什么包管理器？（工具系统）
- 用什么桌面环境？（UI/交互方式）
- 用什么安全机制？（权限控制）
- 用什么文件系统？（状态管理）

Agent 框架也是如此。核心循环是"内核"，而** 特性 **是"发行版定制"。

---

## 二、特性的四层分化

基于代码审计，我将 Agent 框架的差异归纳为** 四层特性体系 **：

### Layer 1：核心循环（Core Loop）- 所有框架相同

这是** 不可见的基础设施 **。无论用哪个框架，你都在用同样的逻辑：
- 维护消息历史（Message History）
- 调用 LLM（LLM API Call）
- 解析输出（Output Parsing）
- 执行工具（Tool Execution）
- 重复直到完成（Iteration）

** 重要结论 **：选择框架时，** 不要被"用了什么算法"迷惑 **。所有框架都支持 ReAct、Function Calling、甚至 Tree-of-Thoughts（只是 prompt 不同）。

### Layer 2：工程特性（Engineering Capabilities）- 差异化竞争

这是** 真正影响使用体验 **的层面。我将 30+ 个特性归纳为 5 大类：

#### 2.1 状态管理特性（State Management）

** 特性：会话持久化（Session Persistence）**

| 框架 | 实现方式 | 代码位置 |
|------|---------|---------|
| **OpenAI Agents SDK** | SQLite / Redis / 自定义（Protocol-based） | `src/agents/memory/session.py:14-55` |
| **OpenCode** | SQLite（Drizzle ORM） | `src/storage/database.ts` |
| **Codex CLI** | JSONL 文件 + SQLite 元数据 | `codex-rs/core/src/state_db.rs` |
| **Kimi CLI** | JSONL 文件 | `src/session.py:context_file` |
| **SWE-agent** | 仅 Trajectory 文件（研究用） | `trajectory.jsonl` |
| **OpenManus** | ❌ 无（In-memory only） | - |

** 为什么重要 **：生产环境必须支持对话历史持久化。如果 Agent 崩溃了，用户期望能恢复对话，而不是从头开始。

** 学术困境 **：SWE-agent 和 OpenManus 缺少通用的生产级 Session 管理（SWE-agent 以 trajectory 持久化为主），导致研究代码难以直接用于生产。

---

** 特性：状态序列化与 HITL（Human-in-the-Loop）**

这是** 生产级框架的标志 **。

**OpenAI Agents SDK** 的实现（最完整）：
```python
# src/agents/run_state.py:2384 lines
class RunState(Generic[TContext]):
    """可序列化的运行状态，支持中断恢复"""
    input: list[TResponseInputItem]
    output: list[RunItem]
    _current_step: NextStep  # 当前执行步骤
    _last_processed_response: ProcessedResponse
    
    def approve(self, interruption: Interruption) -> None:
        """人工审批后继续"""
        
    def to_state_dict(self) -> dict:
        """序列化为 JSON，可存储/恢复"""
```

** 使用场景 **：
1. Agent 提议执行 `rm -rf /`，系统暂停等待人工确认
2. 用户审查后，选择"拒绝"或"修改后执行"
3. 从暂停点恢复，无需重新运行

** 采用率 **：目前仅 OpenAI SDK 提供完整的内置支持。Claude SDK 通过 Hooks 可部分实现，其他框架多需自行补齐。

---

** 特性：上下文压缩（Context Compaction）**

**OpenCode** 的代表性实现：
```typescript
// src/session/prompt.ts:274-724
if (await SessionCompaction.isOverflow(sessionID)) {
    // 自动触发压缩
    await SessionCompaction.create({
        sessionID,
        reason: "token_limit_approaching"
    });
    continue;  // 重新循环，使用压缩后的上下文
}
```

** 为什么重要 **：Long-context LLM（如 Gemini 1M tokens）虽然出现，但 cost 仍是问题。OpenCode 的 Smart Compaction 在长会话场景下有助于降低 token 消耗（具体收益取决于任务与配置）。

** 学术困境 **：学术界通常使用短对话（< 10 轮），对 compaction 需求较低；但生产环境的长对话（> 50 轮）往往更需要此特性。

---

#### 2.2 安全控制特性（Security Controls）

这是** 企业级框架的核心差异点 **。5 种安全模型并存：

** 模型 1：无安全（No Security）**
- **OpenManus**：依赖运行环境安全
- **适用场景**：受控的学术研究环境

** 模型 2：Guardrails（护栏）**
- **OpenAI Agents SDK**：
```python
@input_guardrail
async def check_pii(context, input_items):
    """检查是否包含个人信息"""
    
@output_guardrail  
async def check_sensitive_output(context, output):
    """检查输出是否敏感"""

@tool_guardrail
def dangerous_tool_guardrail(context, tool_call):
    """检查工具调用是否危险"""
```

- ** 特点 **：三层防护（输入/输出/工具），Python 装饰器实现
- ** 局限 **：纯软件层，无法阻止底层系统调用

---

** 模型 3：Hooks（钩子）**
- **Claude Agent SDK**（代表性实现）：
```python
# 10+ 个钩子事件
hooks = {
    "PreToolUse": [check_safety],           # 工具执行前
    "PostToolUse": [log_result],            # 工具执行后
    "UserPromptSubmit": [check_prompt],     # 用户提交时
    "SubagentStart": [init_subagent],       # 子代理启动
    "SubagentStop": [cleanup_subagent],     # 子代理停止
    # ... 更多
}
```

- ** 特点 **：极致灵活，可在任何步骤介入
- ** 代价 **：需要开发者深度参与，学习成本高

---

** 模型 4：策略引擎（Policy Engine）**
- **OpenCode**（Wildcard Pattern）：
```json
{
  "permission": {
    "bash": {
      "rm *": "deny",
      "rm -rf /": "deny",
      "*": "ask"
    }
  }
}
```

- **Gemini CLI**（TOML Policy）：
```toml
[policy]
approval_mode = "manual"
trusted_folders = ["/home/user/safe"]

[[policy.rules]]
tool = "shell"
pattern = "sudo *"
action = "deny"
```

- ** 特点 **：声明式配置，Last Match Wins
- ** 优势 **：非开发者也能理解和修改

---

** 模型 5：沙箱隔离（Sandbox Isolation）**
- **Codex CLI**（代表性实现）：
```rust
// codex-rs/core/src/seatbelt.rs (macOS)
// codex-rs/core/src/landlock.rs (Linux)  
// codex-rs/core/src/windows_sandbox.rs (Windows)

pub struct SandboxPolicy {
    pub policy_type: PolicyType,
    pub writable_roots: Vec<PathBuf>,  // 可写目录白名单
    pub network_access: bool,          // 网络访问控制
}

// 三层安全
1. Platform Sandbox（OS-level 进程隔离）
2. Approval Policy（.rules 文件，命令级控制）
3. Network Control（协议/主机/端口级控制）
```

- ** 特点 **：在本次调研框架中提供最完整的 OS-level 隔离能力
- ** 代价 **：Rust 实现，配置复杂，性能开销

---

** 安全特性总结 **：

| 安全级别 | 代表框架 | 适用场景 | 侵入性 |
|---------|---------|---------|-------|
| Level 0 | OpenManus | 学术研究 | 无 |
| Level 1 | OpenAI SDK | 一般应用 | 低（装饰器） |
| Level 2 | Claude SDK | 需要灵活控制 | 中（Hooks） |
| Level 3 | OpenCode, Gemini | 策略可配置 | 低（配置文件） |
| Level 4 | Codex CLI | 企业/金融/医疗 | 高（Sandbox） |

** 学术困境 **：学术界通常使用 Level 0-1，但工业界（尤其金融、医疗）需要 Level 3-4。这导致学术代码难以直接部署。

---

#### 2.3 工具系统特性（Tooling System）

** 特性：MCP（Model Context Protocol）支持深度 **

| 深度级别 | 框架 | 实现 |
|---------|------|------|
| **Level 0** | SWE-agent | ❌ 不支持 |
| **Level 1** | OpenAI SDK | ⚠️ 可通过 tools 接入外部 MCP |
| **Level 2** | OpenCode, Kimi, Gemini, Qwen | ✅ MCP Client（连接外部服务器） |
| **Level 3** | OpenManus | ✅ MCP Client + Server（FastMCP） |
| **Level 4** | Claude SDK | ✅✅✅ SDK MCP（In-Process，零开销） |
| **Level 5** | Codex CLI | ✅✅✅✅ 双角色（Client + Server） |

**Claude SDK 的 SDK MCP**（创新）：
```python
# 在 Python 进程中运行 MCP 服务器，无需子进程
@tool("fibonacci", "Calculate fibonacci", {"n": int})
async def fibonacci(args):
    return {"content": [{"type": "text", "text": f"fib({args['n']})"}]}

server = create_sdk_mcp_server("math-tools", tools=[fibonacci])
```

** 为什么重要 **：传统 MCP 常需启动子进程（npx, python 等），会带来额外启动与通信开销。Claude 的 SDK MCP 直接在进程中运行，可显著降低这部分开销。

---

** 特性：并行工具执行（Parallel Tool Execution）**

**OpenAI Agents SDK**：
```python
# 自动并行（asyncio.gather）
await asyncio.gather(
    *[execute_tool(tc) for tc in response.tool_calls]
)
```

**SWE-agent/OpenManus**：
```python
# 串行执行
for tool_call in response.tool_calls:  # 一个一个执行
    result = execute_tool(tool_call)
```

** 性能差异 **：如果 Agent 需要同时"查询天气、查日历、搜索邮件"，并行执行快 2-3 倍。

** 学术困境 **：学术研究通常用串行（易于分析执行顺序），但生产环境往往更需要并行。

---

#### 2.4 可观测性特性（Observability）

** 特性：结构化追踪（Structured Tracing）**

**OpenAI Agents SDK**（最完整）：
```python
# 6 种 Span 类型
AgentSpan        # Agent 执行
GenerationSpan   # LLM 调用
FunctionSpan     # 工具调用
HandoffSpan      # Agent 切换
GuardrailSpan    # 安全检查
CustomSpan       # 自定义

# 导出到 6+ 平台
set_trace_processors([
    LogfireProcessor(),      # Logfire
    AgentOpsProcessor(),     # AgentOps
    BraintrustProcessor(),   # Braintrust
    # ...
])
```

** 为什么重要 **：You can't improve what you can't measure。生产环境通常需要知道：
- 每个 LLM 调用花了多少钱？
- 哪个工具最慢？
- 为什么 Agent 陷入循环？

** 采用率 **：仅 OpenAI SDK 提供较完整的内置结构化 Tracing；其他框架多为实验性 telemetry、hooks 或基础日志。

** 学术困境 **：学术界用 Trajectory 文件（SWE-agent），但那是给人类读的，不是给机器分析的。

---

#### 2.5 性能优化特性（Performance Optimization）

** 特性：Streaming（流式响应）**

** 普遍采用 **：7/9 框架支持。

** 重要性 **：用户体验。如果 Agent 思考 10 秒才一次性输出，用户会觉得"卡死了"。流式输出让用户看到"正在思考..."

---

** 特性：智能缓存（Smart Caching）**

** 现状 **：在本次调研范围内，** 尚未看到成熟的内置实现 **。

** 为什么需要 **：
```python
# 场景：Agent 多次调用相同工具
get_weather(city="Beijing")  # 第 1 次，调用 API
# ... 10 轮对话后 ...
get_weather(city="Beijing")  # 第 2 次，应该直接用缓存，但无框架支持
```

** 学术机会 **：这是一个** 尚未被满足的需求 **。实现 Tool Result Cache（带 TTL 和失效策略）是有价值的贡献。

---

### Layer 3：协议支持（Protocol Support）- 生态兼容性

这一层不是单个框架能决定的，而是** 行业标准 **。

#### 3.1 MCP（Model Context Protocol）

** 现状 **：8/9 框架支持，但深度不同（见 2.3 节）。

** 为什么重要 **：MCP 正在成为 Agent 生态的"USB 接口"。一旦工具以 MCP 服务器发布，所有支持 MCP 的框架都能使用。

** 学术机会 **：将学术 benchmark（如 SWE-Bench）的评估工具 MCP 化，让工业界框架也能公平对比。

---

#### 3.2 ACP（Agent Client Protocol）

** 现状 **：Kimi CLI 的 ACP 支持最完整；OpenCode 也有 ACP 支持，但生态与成熟度仍在演进。

** 用途 **：IDE 集成。让 VS Code、Zed、JetBrains 能统一接入不同 Agent。

** 前景 **：可能成为 IDE 集成的标准，但目前生态早期。

---

### Layer 4：交互方式（Interaction Mode）- 最表层

这是** 用户直接感知 **的差异：

| 模式 | 框架 | 适用场景 |
|------|------|---------|
| **Python SDK** | OpenAI SDK, Claude SDK | 嵌入应用、Jupyter Notebook |
| **CLI TUI** | OpenCode, Codex, Kimi | 终端重度用户 |
| **Web UI** | Kimi, Qwen | 桌面用户 |
| **IDE Plugin** | Gemini, Qwen | 开发者工作流 |

** 注意 **：这只是"前端"，背后的 Layer 2-3 特性才是核心。

---

## 三、特性的"不统一性"分析

### 3.1 没有"标准特性集"

以下是** 各项目特性的采用热力图 **（✅ = 完整支持，⚠️ = 部分支持，❌ = 不支持）：

| 特性 | OpenAI SDK | Claude SDK | OpenCode | Codex | Kimi | Gemini | Qwen | SWE-agent | OpenManus |
|------|-----------|-----------|---------|-------|------|--------|------|-----------|-----------|
| **Session Persistence** | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ❌ |
| **HITL State** | ✅ | ⚠️ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Context Compaction** | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **OS Sandbox** | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ⚠️ | ✅ | ❌ |
| **Policy Engine** | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **10+ Hooks** | ❌ | ✅ | ❌ | ⚠️ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **MCP Client** | ⚠️ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| **MCP Server** | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **SDK MCP** | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Parallel Tools** | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Tracing** | ✅ | ❌ | ⚠️ | ⚠️ | ❌ | ⚠️ | ❌ | ❌ | ❌ |
| **Smart Caching** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

** 观察 **：
1. ** 各框架的特性组合呈现高度差异化 **
2. ** 每个框架都有相对独特的侧重特性 **：
   - OpenAI SDK: HITL State, 完整 Tracing
   - Claude SDK: SDK MCP, 10+ Hooks
   - OpenCode: Context Compaction, Permission Ruleset
   - Codex: OS Sandbox, Approval Policy
   - Kimi: ACP Server
   - Qwen: Context Variable Templating
   - SWE-agent: Trajectory Recording
   - OpenManus: MCP 双角色（Client+Server）

### 3.2 低采用率特性（机会点）

以下特性在** 本次调研范围内尚未看到成熟内置实现 **，但生产环境可能需要：

1. **Circuit Breaker（熔断器）**：防止级联故障
2. **Tool Result Cache**：避免重复调用
3. **Semantic Memory**：长期记忆（不是简单的消息历史）
4. **Multi-Modal Tool**：统一处理文本/图像/音频工具

** 学术机会 **：这些是有价值的研究/工程方向。

---

## 四、学术与工业的特性需求差异：基于审计的观察

> 本节基于代码审计与对比表的事实观察，分析学术场景与工业场景对 Agent 框架特性的不同需求模式。这种差异可能导致学术成果向工业迁移时的摩擦。

### 4.1 学术界的"最小可行实现"

学术界倾向于选择** 特性较少 **的框架：

**SWE-agent**（研究首选）：
- ✅ ReAct Loop（易于分析）
- ✅ Trajectory Recording（论文素材）
- ❌ Session（不需要持久化）
- ❌ Security（受控环境）
- ❌ MCP（自成体系）
- ❌ Parallel Tools（串行易于分析）

**OpenManus**（快速实验）：
- ✅ 简单 ReAct 链
- ✅ MCP 双角色（探索性）
- ❌ 其他大多数特性

** 为什么？** 
- 论文关注** 算法创新 **（新的 Planning 策略）
- 不需要** 工程特性 **（Session, Security, Tracing）
- 简化实验环境（易于复现）

### 4.2 工业界的"生产级要求"

工业界需要** 完整特性栈 **：

**金融/医疗行业**（选择 Codex CLI）：
- ✅ OS Sandbox（数据隔离）
- ✅ Approval Policy（人工审批）
- ✅ Network Control（防止数据泄露）
- ✅ Audit Trail（可审计）

**SaaS 产品**（选择 OpenAI Agents SDK）：
- ✅ Session（用户对话历史）
- ✅ HITL（人工介入）
- ✅ Tracing（监控成本）
- ✅ Provider-agnostic（避免 lock-in）

** 为什么？**
- 生产环境不能崩溃、不能泄露数据、不能失控
- 需要可观测性（知道发生了什么）
- 需要成本控制（Token = 钱）

### 4.3 差异可能导致的问题

基于上述特性对比，可以推断以下潜在问题：

**场景 1：论文复现的摩擦**
- 论文用 SWE-agent 实现了一个新算法
- 工业界想用 OpenAI SDK 复现，发现：
  - 没有 Trajectory Recording（无法直接对比）
  - 需要添加 Session、Security、Tracing（额外工程工作）
- ** 可能结果 **：算法创新好，但迁移到生产环境需要额外工程投入

**场景 2：Benchmark 可比性挑战**
- SWE-Bench 使用 SWE-agent 的工具（`str_replace_editor`）
- 其他框架（OpenAI SDK, Codex）难以在相同工具条件下运行
- ** 可能结果 **：论文对比的可比性受限，"我的框架比你强"这类结论的可信度可能不足

**场景 3：先进能力的可用性差异**
- Claude Code（闭源）有强大的 Coding Agent 能力
- 学术界无法使用（不可复现）
- Claude Agent SDK（开源）有 SDK MCP、10+ Hooks
- 但依赖闭源 CLI，学术界可能犹豫
- ** 可能结果 **：学术界可能倾向于使用特性较简单的框架（SWE-agent, OpenManus），而非工程特性最完善的框架

---

## 五、未来预测：特性会收敛吗？（作者判断）

> 本节为趋势判断，基于前文事实推演，不等同于已被数据验证的结论。

### 5.1 Layer 2（工程特性）：短期内难以收敛

** 原因 **：
1. ** 差异化竞争 **：框架通过独特特性建立护城河
   - Codex: "我们最安全的 Agent"
   - Claude SDK: "最强大的 Hooks 系统"
   - OpenCode: "100% 开源 + 最强权限控制"

2. ** 场景分化 **：不同场景需要不同特性组合
   - 研究：简单 + 可解释
   - SaaS：成本 + 可观测性
   - 企业：安全 + 合规

3. ** 技术栈绑定 **：语言选择限制了特性实现
   - Rust（Codex）：适合 Sandbox，但不适合快速迭代
   - Python（OpenAI SDK）：适合 AI 生态，但性能受限
   - TypeScript（OpenCode）：适合 Web，但 ML 库少

### 5.2 Layer 3（协议支持）：更可能收敛

**MCP 正在成为事实标准**：
- 8/9 框架支持（仅 SWE-agent 不支持）
- Anthropic 推动，OpenAI 也接受
- 一旦工具生态 MCP 化，不支持 MCP 的框架竞争力将明显受限

** 预测 **：
- 2 年内，MCP Client 可能成为基础能力
- 3 年内，MCP Server 成为差异化竞争力（类似现在）

### 5.3 未来关键特性预测

**Tier 1：即将成为高优先级能力（未来 1-2 年）**
1. **Context Compaction**：Long-context LLM 虽强，但 cost 敏感
2. **Structured Tracing**：可观测性成为生产高优先级能力
3. **HITL State Management**：人工介入是安全底线

**Tier 2：企业级高优先级（未来 2-3 年）**
4. **Smart Caching**：Tool Result Cache（尚未实现）
5. **Multi-Modal Tool**：统一处理文本/图像/音频
6. **Semantic Memory**：长期记忆（非简单历史）

**Tier 3：前沿探索（长期）**
7. **Self-Reflection**：自动改进（类似 Reflexion，但生产级）
8. **Multi-Agent Protocol**：标准化 Agent 间协作

---

## 六、结论与行动建议：评估 Agent 框架的新范式

### 6.0 Meta 结论：Agent 框架竞争的本质

基于对 9 个框架的代码审计，本文的核心洞察是：

> **Agent 框架的竞争，本质上是工程系统的竞争，而不是推理算法的竞争。**

在 2022-2026 年间，Agent 领域经历了从"算法创新驱动"到"系统工程驱动"的转变：

- **算法创新**（2022-2023）：ReAct、Reflexion、Tree of Thoughts 等新推理范式
- **工程系统**（2024-2026）：Session、Tracing、Sandbox、MCP 等基础能力建设

这一转变的标志性事件是 **OpenAI Agents SDK** 的发布（2025）：它没有提出新的推理算法，而是将 Swarm 的工程实践产品化，提供标准化的 Session、Tracing、Guardrails 能力。

**对学术界和工业界的启示**：
- 学术界如果仍停留在"提出新推理算法"，可能低估工程特性的重要性
- 工业界如果忽视工程系统建设，再好的算法也难以落地生产
- 未来的突破可能来自**算法与工程的协同创新**（如 Learned Context Compaction、Adaptive Tool Selection 等）

### 6.1 先避免这三类低信息量问题

❌ "用没用 Function Calling？" → 只是解析方式，不是架构差异
❌ "支持不支持 MCP？" → 在本次调研中 8/9 支持，深度不同才是关键
❌ "Stars 多少？" → 人气不等于适合你的场景

### 6.2 再用这四组问题做场景化选型

✅ **"Layer 2 特性组合是否符合我的场景？"**
- 做研究？→ 选 Stateless + Trajectory（SWE-agent）
- 做 SaaS？→ 选 Session + Tracing（OpenAI SDK；若是长会话可补充 compaction 能力）
- 做企业？→ 选 Sandbox + Policy（Codex）

✅ **"是否需要 HITL？"**
- 如果涉及危险操作（删数据、转账），应优先选择支持 HITL 状态管理的框架（如 OpenAI SDK）

✅ **"安全需求级别？"**
- Level 1-2：OpenAI SDK（Guardrails）
- Level 3：OpenCode/Gemini（Policy Engine）
- Level 4：Codex（Sandbox）

✅ **"Provider Lock-in 容忍度？"**
- 零容忍 → OpenAI SDK（Provider-agnostic）
- 只用 Claude → Claude SDK

### 6.3 给学术界的建议（可执行）

1. **选择 OpenAI Agents SDK 作为基准**：
   - Provider-agnostic（可对比 GPT-4/Claude/Gemini）
   - 完整特性（Session, Tracing, Guardrails）
   - 生产就绪（易于迁移到工业界）

2. **将评估工具 MCP 化**：
   - 把 `str_replace_editor` 等工具发布为 MCP 服务器
   - 让所有框架能在相同条件下对比
   - 提高论文的可比性和影响力

3. **关注未实现的特性**：
   - Tool Result Cache（Caching）
   - Circuit Breaker（Fault Tolerance）
   - Semantic Memory（Long-term Memory）
   - 这些是有价值的工程/研究贡献

### 6.4 给工业界的建议（可执行）

1. **不要重新造轮子**：
   - 核心循环都一样，直接用成熟框架（OpenAI SDK, Codex）
   - 把精力放在业务逻辑，而非基础设施

2. **通过 MCP 集成专有工具**：
   - 不要 fork 框架代码
   - 用 MCP Server 暴露内部 API

3. **投资可观测性**：
   - 从 Day 1 就接入 Tracing（OpenAI SDK 的 Logfire/AgentOps）
   - 你不知道自己会踩什么坑，直到监控告诉你

---

## Related Work（已发表相关文章）

> 本节用于定位本文与已有公开文章的关系：哪些观点已有讨论、哪些是本文的增量。

### A. 框架选型与工程能力对比（高相关）

1. **Choosing the Right AI Framework**（Enhancial, 2025-10）  
    https://enhancial.substack.com/p/choosing-the-right-ai-framework-a  
    - 覆盖 OpenAI Agents SDK、Claude Agent SDK、LangGraph、MCP 的选型框架。  
    - 与本文相似点：强调工程能力（session、guardrails、tracing）比“模型名气”更重要。

2. **How to build AI agents with MCP: 12 framework comparison**（ClickHouse, 2025-10）  
    https://clickhouse.com/blog/how-to-build-ai-agents-mcp-12-frameworks  
    - 以 MCP 为主线横向比较 12 个框架，提供可运行示例。  
    - 与本文相似点：讨论 MCP 深度差异与安全/工具治理。

3. **OpenAI AgentKit vs Claude Agents SDK**（Bind AI, 2025-10）  
    https://blog.getbind.co/openai-agentkit-vs-claude-agents-sdk-which-is-better/  
    - 聚焦平台化（AgentKit）与 SDK 化（Claude + MCP）的治理差异。  
    - 与本文相似点：将差异落在权限、可观测性、控制面，而非“谁更聪明”。

### B. CLI Agent 生态横评（中高相关）

4. **EVERY CLI CODING AGENT, COMPARED**（Michael Livshits, 2026-02）  
    https://michaellivs.com/blog/cli-coding-agents-compared/  
    - 对 Codex/Gemini/Qwen/Kimi/OpenCode/SWE-agent 等进行全景比较。  
    - 与本文相似点：重视 sandbox、hooks、memory files、MCP 等工程特性。

5. **The 2026 Guide to Coding CLI Tools: 15 AI Agents Compared**（Tembo, 2026-02）  
    https://www.tembo.io/blog/coding-cli-tools-comparison  
    - 从“供应商绑定、自治度、成本、模型灵活性”维度做系统分类。  
    - 与本文相似点：强调工具选型要看约束条件与场景，而非单一指标。

6. **The Ultimate Comparison of Claude Code Alternatives**（Kevnu, 2025-11）  
    https://www.kevnu.com/en/posts/the-ultimate-comparison-of-claude-code-alternatives-a-complete-analysis-of-the-10-strongest-cli-ai-programming-tools  
    - 覆盖 OpenCode、Qwen、Kimi、Codex、Gemini 等 CLI 工具的能力矩阵。  
    - 与本文相似点：提供多工具横向能力对照。

### C. 官方方法论与单框架深挖（中相关）

7. **Building agents with the Claude Agent SDK**（Anthropic, 2025-09）  
    https://claude.com/blog/building-agents-with-the-claude-agent-sdk  
    - 提出 gather → act → verify → repeat 的 agent loop 与 compaction/subagent/MCP 实践。  
    - 与本文相似点：支持“核心循环高度稳定、差异在工程实现”的观察。

8. **I tested 5 AI CLI tools**（LogRocket, 2025-12）  
    https://blog.logrocket.com/tested-5-ai-cli-tools/  
    - 以统一任务对多个 CLI 进行可用性和质量对比。  
    - 与本文相似点：强调工程可用性与真实工作流表现。

### 本文的增量定位

- 与上述文章相比，本文的主要增量是：
  1) 用统一的 Layer 1-4 框架整理 9 个项目；  
  2) 将差异系统化到 session/security/policy/MCP/tracing/compaction 等工程维度；  
  3) 明确提出“学术实现到工业落地”之间的特性迁移成本问题。

---

## 附录：9 个框架的核心代码位置

| 框架 | 核心循环文件 | 特性文件 | 总代码规模 |
|------|------------|---------|-----------|
| **OpenAI Agents SDK** | `src/agents/run.py:396-1329` | `src/agents/handoffs/`, `src/agents/tracing/`, `src/agents/memory/` | ~15k lines |
| **Claude Agent SDK** | `_internal/query.py` | `_internal/query.py:286-299` (Hooks) | SDK ~3k lines |
| **OpenCode** | `src/session/prompt.ts:274-724` | `src/permission/next.ts`, `src/session/compaction.ts` | ~47k lines |
| **Codex CLI** | `codex-rs/core/src/codex.rs` | `codex-rs/core/src/exec_policy.rs`, `codex-rs/core/src/seatbelt.rs` | Core ~10k lines |
| **Kimi CLI** | `src/kimi_cli/soul/kimisoul.py` | `src/kimi_cli/acp/`, `src/kimi_cli/session.py` | 未统计 |
| **Gemini CLI** | `packages/core/src/core/client.ts` | `packages/core/src/policy/`, `packages/core/src/tools/mcp-client.ts` | ~866 files |
| **Qwen Code** | `packages/core/src/core/client.ts` | `packages/core/src/subagents/`, `packages/core/src/skills/` | 未统计 |
| **SWE-agent** | `sweagent/agent/agents.py:1265` | `sweagent/tools/`, `sweagent/agent/` | Agent ~1.3k lines |
| **OpenManus** | `app/agent/react.py:11-38` | `app/mcp/`, `app/flow/` | 未统计 |

---

## 参考

- **深度调研文档**：`agent_deep_dive/*-DEEP-DIVE.md`（~300 KB 代码审计）
- **GitHub 仓库**：见各框架链接
- **MCP 规范**：https://modelcontextprotocol.io/

---

## 作者与致谢

**研究团队**：深度技术调研组

**AI 辅助**：本研究及博客撰写过程中使用了 [Opencode](https://opencode.ai) 进行代码审计、架构分析和内容撰写。Opencode 是一个 AI 驱动的软件工程助手，帮助完成了约 300KB 源代码的分析和文档生成。

**研究方法**：所有技术细节均基于实际源代码审计，具体文件路径和行号可在 [deep-dive/](deep-dive/) 目录下的各框架深度调研报告中查证。

**反馈与贡献**：如发现错误或希望补充其他框架分析，欢迎提交 Issue 或 PR。

**版本**：2026-02-27

**许可证**：[MIT](LICENSE)
