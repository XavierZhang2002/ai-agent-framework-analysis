# OpenAI Agents SDK - 深度技术调研

!!! info "项目概览"
    **调研日期**: 2026-02-26  
    **仓库**: https://github.com/openai/openai-agents-python  
    **许可证**: MIT  
    **版本**: v0.10.2  
    **Stars**: 19.2k | **Forks**: 3.2k | **Contributors**: 219  

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

### 1.1 定位与演进

OpenAI Agents SDK 是 **Swarm** 框架的生产就绪演进版本：

- **Swarm** (已废弃): 实验性/教育性框架，用于探索轻量级多 agent 编排模式
- **Agents SDK** (当前): 生产级框架，由 OpenAI 团队积极维护，面向实际应用

**关键演进**:
```
Swarm (2024)                      Agents SDK (2025-2026)
├─ 教育性质                       ├─ 生产就绪
├─ 基础 handoff 机制               ├─ 完整 handoff + sessions + tracing
├─ 简单示例                        ├─ 12,775 行示例代码
├─ 无状态管理                      ├─ RunState + Session 持久化
└─ 社区实验项目                    └─ 官方维护 (1,135 commits)
```

### 1.2 核心特性

**5 大核心原语**:

1. **Agents** - 配置化的 LLM 实例（instructions + tools + handoffs + guardrails + model）
2. **Handoffs** - Agent 间控制权转移的专用机制（非简单函数调用）
3. **Guardrails** - 输入/输出验证与安全检查
4. **Sessions** - 自动会话历史管理（SQLite/Redis/自定义后端）
5. **Tracing** - 内置可观测性（支持 Logfire、AgentOps、Braintrust 等）

**设计哲学**:
- ✅ **Lightweight over feature-heavy**: 专注核心原语，避免过度抽象
- ✅ **Provider-agnostic**: 通过 LiteLLM 支持 100+ LLM（不绑定 OpenAI）
- ✅ **Protocol-based extensibility**: 使用 Protocol 而非继承，鼓励组合
- ✅ **Production-ready**: 完整的状态管理、错误处理、流式传输、HITL（Human-in-the-Loop）

---

## 2. 代码架构深度分析

### 2.1 目录结构与模块职责

```
src/agents/
├── __init__.py                      # 公开 API 导出 (469 lines)
├── agent.py                         # Agent 类定义 (890 lines)
├── run.py                           # Runner & AgentRunner (1623 lines) ★★★
├── run_state.py                     # HITL 可序列化状态 (2384 lines) ★★★
│
├── memory/                          # Session 后端 ★★
│   ├── session.py                   # Session Protocol (150 lines)
│   ├── sqlite_session.py            # SQLite 实现
│   ├── openai_*_session.py          # OpenAI 服务端 sessions
│   └── ...
│
├── models/                          # 模型抽象层 ★★
│   ├── interface.py                 # Model/ModelProvider Protocol (141 lines)
│   ├── openai_responses.py          # OpenAI Responses API
│   ├── openai_chatcompletions.py    # Chat Completions API
│   └── multi_provider.py            # 多供应商支持（LiteLLM）
│
├── tool.py                          # 工具定义 (1288 lines) ★★
├── handoffs/                        # Handoff 系统 ★★★
│   ├── __init__.py                  # Handoff 类 (334 lines)
│   └── history.py                   # 历史映射
│
├── tracing/                         # Tracing 基础设施 ★★★
│   ├── traces.py                    # Trace 抽象 (533 lines)
│   ├── spans.py                     # Span 抽象
│   ├── processor_interface.py       # TracingProcessor Protocol (142 lines)
│   └── processors.py                # 内置处理器
│
├── run_internal/                    # 核心运行循环 ★★★★★
│   ├── run_loop.py                  # 主编排逻辑 (1623+ lines) ★★★★★
│   ├── turn_resolution.py           # Turn 处理 (1623 lines)
│   ├── tool_execution.py            # 工具执行 (~50KB, 1400 lines) ★★★★
│   ├── tool_planning.py             # 工具规划/调度
│   ├── session_persistence.py       # Session 保存逻辑
│   └── streaming.py                 # 流式传输辅助
│
├── guardrail.py                     # Input/Output guardrails
├── tool_guardrails.py               # Tool-specific guardrails
│
└── extensions/                      # 扩展实现
    ├── memory/
    │   ├── redis_session.py         # Redis 后端
    │   ├── sqlalchemy_session.py    # SQLAlchemy 后端
    │   └── dapr_session.py          # Dapr 后端
    └── models/
        └── litellm_model.py         # LiteLLM 集成 ★★★
```

**★ 标注说明**:
- ★★★★★ = 核心中的核心，必须深入理解
- ★★★ = 核心机制
- ★★ = 重要扩展点
- 无标注 = 工具性/配置性代码

### 2.2 核心类关系图

```
┌────────────────────────────────────────────────────────────────┐
│                         Runner                                  │
│  静态入口点:                                                     │
│  • run() → AgentRunner.run()                                   │
│  • run_sync() → asyncio.run(run())                             │
│  • run_streamed() → AgentRunner.run_streamed()                │
└──────────────────────────┬─────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────┐
│                      AgentRunner                                │
│  核心编排器 (src/agents/run.py:396-1329):                       │
│  • run() - 主 async 循环                                        │
│  • run_streamed() - 流式传输变体                                │
│  • 职责: 管理 while loop，协调各子系统                          │
└──────────────────────────┬─────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┬───────────┐
        ▼                  ▼                  ▼           ▼
┌──────────────┐    ┌─────────────┐   ┌────────────┐  ┌──────────┐
│    Agent     │    │  RunState   │   │RunContext  │  │  Model   │
│              │    │             │   │  Wrapper   │  │          │
│ • name       │    │ HITL 关键   │   │            │  │ Protocol │
│ • instruct'ns│    │ • serialize │   │ • context  │  │ based    │
│ • tools      │    │ • resume    │   │ • usage    │  │          │
│ • handoffs   │    │ • approval  │   │ • approvals│  │          │
│ • model      │    │             │   │            │  │          │
│ • guardrails │    │             │   │            │  │          │
└──────┬───────┘    └─────────────┘   └────────────┘  └──────────┘
       │
       ├────────┬──────────┬──────────┬──────────┐
       ▼        ▼          ▼          ▼          ▼
   ┌──────┐ ┌────────┐ ┌──────────┐┌─────────┐┌────────┐
   │ Tool │ │Handoff │ │Guardrail ││Session  ││Tracing │
   │      │ │        │ │          ││         ││        │
   │func  │ │→Agent  │ │Input/Out ││Protocol ││Trace   │
   │tool  │ │        │ │Tool I/O  ││         ││  Span  │
   └──────┘ └────────┘ └──────────┘└─────────┘└────────┘
```

**关系说明**:
1. **Runner → AgentRunner**: 静态委托模式
2. **AgentRunner → RunState**: 创建/管理 HITL 状态
3. **RunState → Session**: 可选持久化后端
4. **Agent → Tool/Handoff/Guardrail**: 组合模式（一对多）
5. **Model ← ModelProvider**: 工厂模式

### 2.3 关键设计模式

| 模式 | 应用场景 | 代码位置 |
|------|---------|---------|
| **Protocol-based Design** | Session, Model, TracingProcessor | `memory/session.py`, `models/interface.py` |
| **Factory Pattern** | ModelProvider.get_model(), handoff() | `models/interface.py:50-80` |
| **Strategy Pattern** | 不同 Session/Model/Tracing 实现 | `memory/`, `models/`, `tracing/` |
| **Context Manager** | TraceCtxManager, agent_span() | `tracing/context.py` |
| **Observer Pattern** | Tracing processors | `tracing/processor_interface.py` |
| **State Machine** | NextStep variants | `run_internal/turn_resolution.py:20-60` |
| **Async Iterator** | Streaming responses | `run.py:1330+`, `run_loop.py:374-1042` |

---

## 3. 底层实现原理

### 3.1 Agent Loop 核心流程

**入口**: `src/agents/run.py:396-1329` (`AgentRunner.run()`)

```python
async def run(self, starting_agent, input, **kwargs) -> RunResult:
    # ═══════════════════════════════════════════════════════════
    # 第一阶段: 初始化 (lines 402-634)
    # ═══════════════════════════════════════════════════════════
    
    # 1.1 RunState 准备
    if is_resumed_state:
        run_state = cast(RunState[TContext], input)
        # 恢复会话设置
        conversation_id, previous_response_id, auto_previous_response_id = \
            apply_resumed_conversation_settings(run_state, ...)
        # 恢复 context
        context_wrapper = resolve_resumed_context(run_state, context)
        max_turns = run_state._max_turns
    else:
        run_state = RunState[TContext](...)
    
    # 1.2 Session 集成
    if session:
        input_items, new_items_for_save = await prepare_input_with_session(
            raw_input, session, callback, settings, ...
        )
    
    # 1.3 Tracing 上下文
    with TraceCtxManager(
        workflow_name=trace_workflow_name,
        trace_id=trace_id,
        group_id=trace_group_id,
        metadata=trace_metadata,
        tracing=trace_config,
        disabled=run_config.tracing_disabled,
    ):
    
        # ═══════════════════════════════════════════════════════════
        # 第二阶段: 主循环 (lines 637-1304)
        # ═══════════════════════════════════════════════════════════
        
        while True:
            # ───────────────────────────────────────────────────────
            # 2.1 处理中断恢复
            # ───────────────────────────────────────────────────────
            if run_state._current_step == NextStepInterruption:
                # 恢复被中断的 turn (lines 647-836)
                turn_result = await resolve_interrupted_turn(...)
                # 跳过后续的 model 调用
                skip_model_call = True
            
            # ───────────────────────────────────────────────────────
            # 2.2 工具准备
            # ───────────────────────────────────────────────────────
            all_tools = await get_all_tools(current_agent, context_wrapper)
            await initialize_computer_tools(...)
            
            # ───────────────────────────────────────────────────────
            # 2.3 创建 Agent Span (Tracing)
            # ───────────────────────────────────────────────────────
            current_span = agent_span(
                name=current_agent.name,
                agent=current_agent,
                ...
            )
            current_span.start(mark_as_current=True)
            
            # ───────────────────────────────────────────────────────
            # 2.4 Turn 计数与限制检查
            # ───────────────────────────────────────────────────────
            current_turn += 1
            if current_turn > max_turns:
                # 触发 max_turns 处理 (lines 864-951)
                # ... 可能创建 FinalOutput 或 Interruption
                break
            
            # ───────────────────────────────────────────────────────
            # 2.5 Guardrails 检查 (仅第一个 turn)
            # ───────────────────────────────────────────────────────
            if current_turn <= 1 and not skip_model_call:
                # 顺序执行的 guardrails
                sequential_results = await run_input_guardrails(
                    context_wrapper, input_items, 
                    guardrails=[g for g in input_guardrails if g.parallel==False]
                )
                
                # 并行 guardrails 与 model 调用
                parallel_results, turn_result = await asyncio.gather(
                    run_input_guardrails(
                        context_wrapper, input_items,
                        guardrails=[g for g in input_guardrails if g.parallel==True]
                    ),
                    run_single_turn(...)
                )
            else:
                # ───────────────────────────────────────────────────
                # 2.6 执行单个 Turn
                # ───────────────────────────────────────────────────
                turn_result = await run_single_turn(
                    agent=current_agent,
                    original_input=original_input,
                    input=current_input,
                    all_tools=all_tools,
                    handoffs=handoffs,
                    context_wrapper=context_wrapper,
                    run_state=run_state,
                    hooks=hooks,
                    ...
                )
            
            # ───────────────────────────────────────────────────────
            # 2.7 处理 Turn 结果
            # ───────────────────────────────────────────────────────
            generated_items = turn_result.pre_step_items + turn_result.new_step_items
            session_items.extend(...)
            
            # ───────────────────────────────────────────────────────
            # 2.8 NextStep 分发
            # ───────────────────────────────────────────────────────
            if isinstance(turn_result.next_step, NextStepFinalOutput):
                # ┌────────────────────────────────────────────────┐
                # │ 情况 A: 最终输出                                │
                # └────────────────────────────────────────────────┘
                
                # A.1 运行 Output Guardrails
                for guardrail in output_guardrails:
                    result = await guardrail.func(
                        context_wrapper,
                        turn_result.next_step.final_output
                    )
                    if result.output.tripwire_triggered:
                        # 处理 guardrail 违规
                        ...
                
                # A.2 运行 PostRun Hooks
                await run_hooks(hooks.get("PostRun", []), ...)
                
                # A.3 保存 Session (如果有)
                if session:
                    await save_result_to_session(
                        session, input_items, output_items, run_state, ...
                    )
                
                # A.4 返回 RunResult
                return RunResult(
                    final_output=turn_result.next_step.final_output,
                    run_context=context_wrapper,
                    model_usage=context_wrapper.usage,
                    ...
                )
            
            elif isinstance(turn_result.next_step, NextStepHandoff):
                # ┌────────────────────────────────────────────────┐
                # │ 情况 B: Handoff 到新 Agent                      │
                # └────────────────────────────────────────────────┘
                
                # B.1 切换 Agent
                current_agent = turn_result.next_step.new_agent
                
                # B.2 更新 RunState
                run_state._current_agent = current_agent.name
                
                # B.3 结束当前 Span
                current_span.finish(reset_current=True)
                
                # B.4 继续循环（with 新 Agent）
                continue
            
            elif isinstance(turn_result.next_step, NextStepInterruption):
                # ┌────────────────────────────────────────────────┐
                # │ 情况 C: 中断（需要人工干预）                     │
                # └────────────────────────────────────────────────┘
                
                # C.1 保存状态
                run_state._current_step = NextStepInterruption
                run_state._last_processed_response = turn_result.processed_response
                
                # C.2 构建 Interruption Result
                return build_interruption_result(
                    interruptions=turn_result.next_step.interruptions,
                    run_state=run_state,
                    ...
                )
            
            elif isinstance(turn_result.next_step, NextStepRunAgain):
                # ┌────────────────────────────────────────────────┐
                # │ 情况 D: 继续执行（工具调用完成，无 handoff）      │
                # └────────────────────────────────────────────────┘
                continue
```

**关键点**:
1. **无状态设计**: 每次 `run()` 调用都是独立的（除非通过 Session 或 RunState 恢复）
2. **事件驱动**: Tracing spans 自动记录每个操作
3. **并发优化**: Guardrails 与 Model 调用并行执行
4. **可中断性**: 通过 `NextStepInterruption` + `RunState` 实现 HITL

### 3.2 Single Turn 执行详解

**入口**: `src/agents/run_internal/run_loop.py:1044-1623+` (`run_single_turn()`)

```python
async def run_single_turn(...) -> SingleStepResult:
    # ═══════════════════════════════════════════════════════════
    # 阶段 1: 调用 Model
    # ═══════════════════════════════════════════════════════════
    new_response = await get_new_response(
        agent=agent,
        system_prompt=system_prompt,
        input=input,
        model_settings=model_settings,
        all_tools=all_tools,
        output_schema=output_schema,
        handoffs=handoffs,
        run_state=run_state,
        ...
    )
    # new_response: ModelResponse(output, usage, response_id)
    
    # ═══════════════════════════════════════════════════════════
    # 阶段 2: 处理 Model 响应
    # ═══════════════════════════════════════════════════════════
    processed_response = await process_model_response(
        agent=agent,
        new_response=new_response,
        all_tools=all_tools,
        handoffs=handoffs,
        context_wrapper=context_wrapper,
        run_state=run_state,
        ...
    )
    # processed_response: ProcessedResponse(
    #     run_tools=[ToolCallRun(tool, arguments), ...],
    #     run_handoffs=[HandoffRun(handoff, arguments), ...],
    #     final_output=<if no tools/handoffs>,
    #     ...
    # )
    
    # ═══════════════════════════════════════════════════════════
    # 阶段 3: 执行工具与副作用
    # ═══════════════════════════════════════════════════════════
    turn_result = await execute_tools_and_side_effects(
        agent=agent,
        original_input=original_input,
        new_response=new_response,
        processed_response=processed_response,
        hooks=hooks,
        context_wrapper=context_wrapper,
        ...
    )
    # turn_result: SingleStepResult(
    #     next_step=<NextStepFinalOutput | NextStepHandoff | NextStepInterruption | NextStepRunAgain>,
    #     pre_step_items=[...],
    #     new_step_items=[...],
    #     ...
    # )
    
    return turn_result
```

**阶段 1: `get_new_response()` 细节** (run_loop.py):
```python
async def get_new_response(...):
    # 1.1 准备系统提示
    system_prompt = await resolve_system_prompt(agent, context_wrapper)
    
    # 1.2 过滤输入（应用 handoff input_filter）
    filtered_input = await maybe_filter_model_input(input, ...)
    
    # 1.3 准备工具 schemas
    tool_schemas = [tool.to_schema() for tool in all_tools]
    
    # 1.4 调用 Model
    if streaming:
        response_stream = await model.stream_response(
            system=system_prompt,
            messages=filtered_input,
            tools=tool_schemas,
            ...
        )
        # 流式处理...
    else:
        response = await model.get_response(
            system=system_prompt,
            messages=filtered_input,
            tools=tool_schemas,
            output_schema=output_schema,  # 结构化输出
            ...
        )
    
    return ModelResponse(output=response, usage=usage, response_id=...)
```

**阶段 2: `process_model_response()` 细节** (turn_resolution.py):
```python
async def process_model_response(...):
    # 2.1 解析 Model 输出
    assistant_msg = new_response.output
    
    # 2.2 提取工具调用
    tool_use_blocks = [b for b in assistant_msg.content if isinstance(b, ToolUseBlock)]
    
    # 2.3 分类为 Tools vs. Handoffs
    run_tools = []
    run_handoffs = []
    
    for tool_use in tool_use_blocks:
        tool_name = tool_use.name
        
        # 检查是否是 Handoff
        handoff = next((h for h in handoffs if h.tool_name == tool_name), None)
        if handoff:
            run_handoffs.append(HandoffRun(handoff, tool_use.input))
        else:
            tool = next((t for t in all_tools if t.name == tool_name), None)
            if tool:
                run_tools.append(ToolCallRun(tool, tool_use.input, tool_use.id))
    
    # 2.4 判断 final_output
    if not run_tools and not run_handoffs:
        # 没有工具调用 → 这是最终输出
        final_output = extract_final_output(assistant_msg, output_schema)
    else:
        final_output = None
    
    return ProcessedResponse(
        run_tools=run_tools,
        run_handoffs=run_handoffs,
        final_output=final_output,
        ...
    )
```

**阶段 3: `execute_tools_and_side_effects()` 细节** (tool_execution.py):
```python
async def execute_tools_and_side_effects(...):
    # 3.1 处理 Handoff (优先级最高)
    if processed_response.run_handoffs:
        return await execute_handoffs(
            processed_response.run_handoffs,
            context_wrapper,
            ...
        )
    
    # 3.2 处理工具调用
    if processed_response.run_tools:
        # 3.2.1 运行 PreToolUse Hooks
        for tool_run in processed_response.run_tools:
            hook_results = await run_hooks(
                hooks.get("PreToolUse", []),
                tool_run,
                context_wrapper,
                ...
            )
            # 检查是否被 deny
            if hook_results.permission_decision == "deny":
                # 创建 denied tool result
                ...
        
        # 3.2.2 并行执行工具
        tool_results = await execute_function_tool_calls(
            processed_response.run_tools,
            context_wrapper,
            hooks,
            ...
        )
        
        # 3.2.3 运行 PostToolUse Hooks
        for tool_result in tool_results:
            await run_hooks(
                hooks.get("PostToolUse", []),
                tool_result,
                context_wrapper,
                ...
            )
        
        # 3.2.4 创建 Tool Result Items
        new_step_items.extend(
            [ToolResultItem(tool_use_id=..., content=...) for ... in tool_results]
        )
        
        return SingleStepResult(
            next_step=NextStepRunAgain(),  # 继续循环
            new_step_items=new_step_items,
            ...
        )
    
    # 3.3 没有工具/handoff → 最终输出
    if processed_response.final_output:
        return SingleStepResult(
            next_step=NextStepFinalOutput(final_output=processed_response.final_output),
            ...
        )
```

### 3.3 Handoff 机制深度解析

**核心思想**: Handoff 是一个特殊的 Tool，但不返回 Tool Result，而是返回新的 Agent

**实现位置**: `src/agents/handoffs/__init__.py:93-321`

```python
@dataclass
class Handoff(Generic[TContext, TAgent]):
    """Handoff 定义"""
    tool_name: str                              # e.g., "transfer_to_Spanish_agent"
    tool_description: str
    input_json_schema: dict                     # Tool input schema
    on_invoke_handoff: Callable[..., Awaitable[TAgent]]  # 返回目标 Agent
    agent_name: str                             # 目标 Agent 名称
    input_filter: HandoffInputFilter | None     # 可选：过滤传递给新 Agent 的历史
    
    def to_tool_schema(self) -> dict:
        """转换为 Tool Schema（供 LLM 使用）"""
        return {
            "type": "function",
            "function": {
                "name": self.tool_name,
                "description": self.tool_description,
                "parameters": self.input_json_schema,
            }
        }

# ═══════════════════════════════════════════════════════════
# Factory 函数: handoff()
# ═══════════════════════════════════════════════════════════
def handoff(
    agent: TAgent | str,
    *,
    tool_name: str | None = None,
    tool_description: str | None = None,
    input_filter: HandoffInputFilter | None = None,
) -> Handoff[Any, TAgent]:
    """创建 Handoff"""
    
    # 1. 生成默认 tool_name
    if tool_name is None:
        if isinstance(agent, str):
            tool_name = f"transfer_to_{agent}"
        else:
            tool_name = f"transfer_to_{agent.name}"
    
    # 2. 生成默认 description
    if tool_description is None:
        tool_description = f"Transfer to {agent_name} agent"
    
    # 3. 定义 invoke 函数
    async def on_invoke_handoff(
        context: RunContextWrapper[Any],
        arguments_json: str
    ) -> TAgent:
        # 直接返回 Agent（可以是动态的）
        if callable(agent):
            return await agent(context, arguments_json)
        else:
            return agent
    
    return Handoff(
        tool_name=tool_name,
        tool_description=tool_description,
        input_json_schema={"type": "object", "properties": {}, "additionalProperties": False},
        on_invoke_handoff=on_invoke_handoff,
        agent_name=agent_name,
        input_filter=input_filter,
    )

# ═══════════════════════════════════════════════════════════
# Handoff 执行: execute_handoffs()
# ═══════════════════════════════════════════════════════════
# 位置: src/agents/run_internal/turn_resolution.py:268-400+
async def execute_handoffs(...) -> SingleStepResult:
    # 1. 只处理第一个 Handoff（忽略多个）
    if len(run_handoffs) > 1:
        warnings.warn("Multiple handoffs detected, only first will be processed")
    
    handoff_run = run_handoffs[0]
    
    # 2. 调用 Handoff 函数
    new_agent = await handoff_run.handoff.on_invoke_handoff(
        context_wrapper,
        handoff_run.arguments_json
    )
    
    # 3. 应用 input_filter（如果有）
    if handoff_run.handoff.input_filter:
        handoff_input_data = HandoffInputData(
            input_history=original_input,
            pre_handoff_items=tuple(pre_step_items),
            new_items=tuple(new_step_items),
            run_context=context_wrapper,
        )
        
        filtered_data = await handoff_run.handoff.input_filter(handoff_input_data)
        
        # 应用过滤结果
        # filtered_data 包含: input_history, pre_handoff_items, new_items
        # 这些将传递给新 Agent
    
    # 4. 可选: 嵌套历史（summarize 以前的对话）
    if nest_handoff_history:
        # 使用 history.py 中的 mapper 来嵌套历史
        filtered_data = nest_handoff_history(filtered_data, mapper=...)
    
    # 5. 创建 Handoff Items
    new_step_items.append(
        HandoffCallItem(
            tool_use_id=handoff_run.tool_use_id,
            tool_name=handoff_run.handoff.tool_name,
            arguments_json=handoff_run.arguments_json,
        )
    )
    new_step_items.append(
        HandoffOutputItem(
            tool_use_id=handoff_run.tool_use_id,
            output=f"Transferred to {new_agent.name}",
        )
    )
    
    # 6. 返回 NextStepHandoff
    return SingleStepResult(
        next_step=NextStepHandoff(new_agent=new_agent),
        pre_step_items=filtered_data.pre_handoff_items,
        new_step_items=new_step_items,
        ...
    )
```

**Handoff 的特殊之处**:

| 维度 | 普通 Tool | Handoff Tool |
|------|----------|-------------|
| **返回值** | Tool Result (string/dict) | 新的 Agent 对象 |
| **执行后行为** | 添加 ToolResultItem，继续循环 | 切换 Agent，重置 Span，继续循环 |
| **历史处理** | 保留完整历史 | 可选 input_filter 过滤历史 |
| **调用次数** | 可并行多个 | 只处理第一个 |
| **NextStep** | NextStepRunAgain | NextStepHandoff |

**Input Filter 示例**:
```python
async def spanish_agent_filter(data: HandoffInputData) -> HandoffInputData:
    """只传递最后 3 条消息给 Spanish Agent"""
    return HandoffInputData(
        input_history=data.input_history[-3:],
        pre_handoff_items=data.pre_handoff_items,
        new_items=data.new_items,
        run_context=data.run_context,
    )

spanish_agent = Agent(...)
triage_agent = Agent(
    handoffs=[
        handoff(spanish_agent, input_filter=spanish_agent_filter)
    ]
)
```

### 3.4 Session 存储与检索机制

**Session Protocol** (`src/agents/memory/session.py:14-55`):
```python
@runtime_checkable
class Session(Protocol):
    """Session 后端必须实现的协议"""
    session_id: str
    session_settings: SessionSettings | None
    
    async def get_items(self, limit: int | None = None) -> list[TResponseInputItem]:
        """获取会话历史
        
        Args:
            limit: 最多返回多少条（None = 全部）
        
        Returns:
            历史消息列表（按时间顺序）
        """
        ...
    
    async def add_items(self, items: list[TResponseInputItem]) -> None:
        """添加新消息到会话"""
        ...
    
    async def pop_item(self) -> TResponseInputItem | None:
        """移除并返回最新的一条消息"""
        ...
    
    async def clear_session(self) -> None:
        """清空会话"""
        ...
```

**SQLite 实现** (`src/agents/memory/sqlite_session.py`):
```python
class SQLiteSession(SessionABC):
    def __init__(
        self,
        session_id: str,
        db_path: str = "conversations.db",
        *,
        session_settings: SessionSettings | None = None,
    ):
        self.session_id = session_id
        self.db_path = db_path
        self._lock = asyncio.Lock()
        self.session_settings = session_settings
        
        # 创建表
        self._create_tables()
    
    def _create_tables(self):
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS sessions (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    session_id TEXT NOT NULL,
                    item_json TEXT NOT NULL,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)
            conn.execute("""
                CREATE INDEX IF NOT EXISTS idx_session_id 
                ON sessions(session_id)
            """)
    
    async def get_items(self, limit: int | None = None) -> list[TResponseInputItem]:
        limit = resolve_session_limit(limit, self.session_settings)
        
        async with self._lock:
            with sqlite3.connect(self.db_path) as conn:
                if limit is None or limit <= 0:
                    query = """
                        SELECT item_json FROM sessions 
                        WHERE session_id = ? 
                        ORDER BY id ASC
                    """
                    rows = conn.execute(query, (self.session_id,)).fetchall()
                else:
                    query = """
                        SELECT item_json FROM sessions 
                        WHERE session_id = ? 
                        ORDER BY id DESC 
                        LIMIT ?
                    """
                    rows = conn.execute(query, (self.session_id, limit)).fetchall()
                    rows = list(reversed(rows))  # 恢复时间顺序
                
                return [json.loads(row[0]) for row in rows]
    
    async def add_items(self, items: list[TResponseInputItem]) -> None:
        async with self._lock:
            with sqlite3.connect(self.db_path) as conn:
                for item in items:
                    conn.execute(
                        """
                        INSERT INTO sessions (session_id, item_json) 
                        VALUES (?, ?)
                        """,
                        (self.session_id, json.dumps(item))
                    )
                conn.commit()
    
    async def pop_item(self) -> TResponseInputItem | None:
        async with self._lock:
            with sqlite3.connect(self.db_path) as conn:
                row = conn.execute(
                    """
                    SELECT id, item_json FROM sessions 
                    WHERE session_id = ? 
                    ORDER BY id DESC 
                    LIMIT 1
                    """,
                    (self.session_id,)
                ).fetchone()
                
                if row:
                    conn.execute("DELETE FROM sessions WHERE id = ?", (row[0],))
                    conn.commit()
                    return json.loads(row[1])
                return None
```

**Session 集成流程** (`src/agents/run_internal/session_persistence.py`):
```python
# ═══════════════════════════════════════════════════════════
# 阶段 1: 准备输入（合并历史）
# ═══════════════════════════════════════════════════════════
async def prepare_input_with_session(
    raw_input,
    session: Session | None,
    callback: Callable | None,
    settings: SessionSettings | None,
    ...
):
    if not session:
        return raw_input, None
    
    # 1. 获取历史
    history = await session.get_items()
    
    # 2. 合并历史 + 新输入
    merged_input = history + normalize_input(raw_input)
    
    # 3. 返回用于保存的新项
    new_items_for_save = normalize_input(raw_input)
    
    return merged_input, new_items_for_save

# ═══════════════════════════════════════════════════════════
# 阶段 2: 保存结果
# ═══════════════════════════════════════════════════════════
async def save_result_to_session(
    session: Session | None,
    input_items: list[TResponseInputItem],
    output_items: list[TResponseInputItem],
    run_state: RunState | None,
    ...
):
    if not session:
        return
    
    # 合并输入 + 输出
    all_items = input_items + output_items
    
    # 保存到 session
    await session.add_items(all_items)
```

**使用示例**:
```python
from agents import Agent, Runner, SQLiteSession

agent = Agent(name="Assistant")

# 创建 session
session = SQLiteSession("user_123", "conversations.db")

# 第一轮
result1 = await Runner.run(
    agent,
    "What city is the Golden Gate Bridge in?",
    session=session
)
print(result1.final_output)  # "San Francisco"

# 第二轮（自动加载历史）
result2 = await Runner.run(
    agent,
    "What state is it in?",
    session=session
)
print(result2.final_output)  # "California"

# Session 中的内容:
# [
#   {"role": "user", "content": "What city is the Golden Gate Bridge in?"},
#   {"role": "assistant", "content": "San Francisco"},
#   {"role": "user", "content": "What state is it in?"},
#   {"role": "assistant", "content": "California"},
# ]
```

### 3.5 Tracing 实现详解

**架构概览**:
```
Trace (workflow-level, 一次 run() 调用)
 ├─ metadata: {workflow_name, trace_id, group_id, ...}
 ├─ Span (operation-level)
 │   ├─ AgentSpan (agent execution)
 │   │   └─ span_data: AgentSpanData(agent_name, model, ...)
 │   ├─ GenerationSpan (LLM call)
 │   │   └─ span_data: GenerationSpanData(model, prompt_tokens, ...)
 │   ├─ FunctionSpan (tool call)
 │   │   └─ span_data: FunctionSpanData(tool_name, arguments, result, ...)
 │   ├─ HandoffSpan (handoff)
 │   │   └─ span_data: HandoffSpanData(from_agent, to_agent, ...)
 │   └─ GuardrailSpan (guardrail check)
 │       └─ span_data: GuardrailSpanData(guardrail_name, triggered, ...)
 └─ processors: [TracingProcessor, ...]
```

**TracingProcessor Protocol** (`src/agents/tracing/processor_interface.py:9-130`):
```python
class TracingProcessor(abc.ABC):
    """Tracing 事件处理器"""
    
    @abstractmethod
    def on_trace_start(self, trace: Trace) -> None:
        """Trace 开始时调用"""
        ...
    
    @abstractmethod
    def on_trace_end(self, trace: Trace) -> None:
        """Trace 结束时调用（可导出数据）"""
        ...
    
    @abstractmethod
    def on_span_start(self, span: Span[Any]) -> None:
        """Span 开始时调用"""
        ...
    
    @abstractmethod
    def on_span_end(self, span: Span[Any]) -> None:
        """Span 结束时调用"""
        ...
    
    @abstractmethod
    def shutdown(self) -> None:
        """清理资源"""
        ...
    
    @abstractmethod
    def force_flush(self) -> None:
        """强制刷新缓冲数据"""
        ...
```

**Trace 生命周期** (`src/agents/run.py:545-554`):
```python
# 在 AgentRunner.run() 中
with TraceCtxManager(
    workflow_name="customer_support",
    trace_id="trace_abc123",
    group_id="session_xyz",
    metadata={"user_id": "user_456"},
    tracing=trace_config,  # TracingConfig(processors=[...])
    disabled=False,
):
    # ───────────────────────────────────────────────────────
    # on_trace_start() 被调用
    # ───────────────────────────────────────────────────────
    
    while True:
        # 创建 Agent Span
        current_span = agent_span(
            name=current_agent.name,
            agent=current_agent,
            ...
        )
        current_span.start(mark_as_current=True)
        # ───────────────────────────────────────────────────
        # on_span_start(current_span) 被调用
        # ───────────────────────────────────────────────────
        
        # ... agent 执行 ...
        
        # 在 run_single_turn() 内部:
        with generation_span(model=..., input=...) as gen_span:
            # on_span_start(gen_span) 被调用
            response = await model.get_response(...)
            # on_span_end(gen_span) 被调用
        
        # 工具执行
        for tool_run in processed_response.run_tools:
            with function_span(tool_name=tool_run.tool.name, ...) as func_span:
                # on_span_start(func_span) 被调用
                result = await execute_tool(tool_run)
                # on_span_end(func_span) 被调用
        
        current_span.finish(reset_current=True)
        # ───────────────────────────────────────────────────
        # on_span_end(current_span) 被调用
        # ───────────────────────────────────────────────────
        
        # ... 循环或退出 ...
    
    # ───────────────────────────────────────────────────────
    # on_trace_end() 被调用（可导出完整 trace）
    # ───────────────────────────────────────────────────────
```

**Span 数据结构** (`src/agents/tracing/span_data.py`):
```python
@dataclass
class AgentSpanData:
    agent_name: str
    agent_model: str
    agent_instructions: str | None
    tools: list[str]
    handoffs: list[str]
    output_type: str | None

@dataclass
class GenerationSpanData:
    model: str
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    latency_ms: float
    system_prompt: str | None
    messages: list[dict]
    response: dict

@dataclass
class FunctionSpanData:
    tool_name: str
    arguments: dict
    result: str | None
    error: str | None
    latency_ms: float

@dataclass
class HandoffSpanData:
    from_agent: str
    to_agent: str
    input_filter_applied: bool

@dataclass
class GuardrailSpanData:
    guardrail_name: str
    guardrail_type: Literal["input", "output", "tool_input", "tool_output"]
    tripwire_triggered: bool
    reason: str | None
```

**自定义 Processor 示例**:
```python
from agents import TracingProcessor, Trace, Span
import json

class JSONFileTracingProcessor(TracingProcessor):
    def __init__(self, output_file: str):
        self.output_file = output_file
        self.traces = []
    
    def on_trace_start(self, trace: Trace) -> None:
        print(f"[TRACE START] {trace.name} ({trace.trace_id})")
    
    def on_trace_end(self, trace: Trace) -> None:
        # 导出完整 trace 数据
        data = trace.export()
        if data:
            self.traces.append(data)
            
            # 写入文件
            with open(self.output_file, 'w') as f:
                json.dump(self.traces, f, indent=2)
            
            print(f"[TRACE END] Exported to {self.output_file}")
    
    def on_span_start(self, span: Span) -> None:
        print(f"  [SPAN START] {span.span_data}")
    
    def on_span_end(self, span: Span) -> None:
        print(f"  [SPAN END] {span.span_id}")
    
    def shutdown(self) -> None:
        print(f"[SHUTDOWN] Total traces: {len(self.traces)}")
    
    def force_flush(self) -> None:
        # 强制写入
        with open(self.output_file, 'w') as f:
            json.dump(self.traces, f, indent=2)

# 使用
from agents import set_trace_processors

processor = JSONFileTracingProcessor("traces.json")
set_trace_processors([processor])

# 运行 agent（自动 trace）
result = await Runner.run(agent, "Hello")
```

---

## 4. 与竞品对比

### 4.1 OpenAI Agents SDK vs. Claude Agent SDK

| 维度 | OpenAI Agents SDK | Claude Agent SDK |
|------|------------------|-----------------|
| **Provider 绑定** | ✅ Provider-agnostic (100+ LLMs via LiteLLM) | ❌ Claude-only |
| **核心抽象** | Agent, Handoff, Session, Guardrail, Tracing | Query, ClaudeSDKClient, Hooks, SDK MCP Server |
| **Session 管理** | ✅ 内置（SQLite/Redis/自定义） | ❌ 无（需自行管理） |
| **Tracing** | ✅ 内置（多后端支持） | ❌ 无内置 tracing |
| **Handoff 机制** | ✅ 一等公民（Handoff 类 + input_filter） | ✅ 通过 subagents 支持（文档化） |
| **HITL 支持** | ✅ RunState + interruptions + approval | ✅ Hooks + permission_mode |
| **工具定义** | Python 函数 (`@function_tool`) | Python 函数 (`@tool` + SDK MCP Server) |
| **MCP 集成** | ❌ 无（但支持外部 MCP via tools） | ✅ 原生支持（in-process SDK MCP Server） |
| **CLI 依赖** | ❌ 无（纯 Python SDK） | ✅ 依赖 Claude Code CLI（bundled） |
| **License** | MIT (纯开源) | MIT + Anthropic 商业条款 |
| **架构透明度** | ✅ 完全开源（run loop 可审计） | ⚠️ SDK 开源，但依赖闭源 CLI |
| **Stars** | 19.2k | 5k |
| **生产成熟度** | ✅ 高（活跃维护，219 contributors） | ✅ 中（官方维护，47 contributors） |
| **适用场景** | 跨 LLM 多 agent 编排 | Claude 深度集成 |

**代码对比 - 基础 Agent**:

```python
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
```

**代码对比 - Handoff**:

```python
# ═══════════════════════════════════════════════════════════
# OpenAI Agents SDK
# ═══════════════════════════════════════════════════════════
from agents import Agent, Runner, handoff

spanish_agent = Agent(name="Spanish", instructions="Only speak Spanish")
english_agent = Agent(name="English", instructions="Only speak English")

triage_agent = Agent(
    name="Triage",
    instructions="Route to appropriate language agent",
    handoffs=[spanish_agent, english_agent]  # 自动生成 handoff tools
)

result = await Runner.run(triage_agent, "Hola")
# 自动 handoff 到 spanish_agent

# ═══════════════════════════════════════════════════════════
# Claude Agent SDK
# ═══════════════════════════════════════════════════════════
# 通过 hooks 实现（需手动管理 subagents）
# 详见 https://docs.anthropic.com/en/docs/claude-code/hooks
```

**代码对比 - Session**:

```python
# ═══════════════════════════════════════════════════════════
# OpenAI Agents SDK
# ═══════════════════════════════════════════════════════════
from agents import SQLiteSession

session = SQLiteSession("user_123")

result1 = await Runner.run(agent, "What's the capital of France?", session=session)
result2 = await Runner.run(agent, "What's its population?", session=session)
# session 自动管理历史

# ═══════════════════════════════════════════════════════════
# Claude Agent SDK
# ═══════════════════════════════════════════════════════════
# 需要手动管理历史:
messages = []

async for msg in query("What's the capital of France?"):
    messages.append(msg)

# 第二次需要传递历史（如果需要 context）
# 无内置 session 抽象
```

### 4.2 OpenAI Agents SDK vs. LangChain

| 维度 | OpenAI Agents SDK | LangChain |
|------|------------------|-----------|
| **设计哲学** | Lightweight, 核心原语 | Full-featured, 全栈框架 |
| **学习曲线** | ⭐⭐ 陡峭度低 | ⭐⭐⭐⭐ 陡峭度高 |
| **抽象层数** | 2 层（Agent → Runner） | 5+ 层（Chain → Agent → Executor → Tool → Memory → ...） |
| **代码行数** | ~15,000 lines (core) | ~500,000+ lines (framework) |
| **适用范围** | Multi-agent workflows | LLM apps (RAG, chains, agents, etc.) |
| **RAG 支持** | ❌ 无（需自行实现） | ✅ 内置（VectorStore, Retriever, etc.) |
| **Memory 抽象** | Session Protocol (简单) | ConversationBufferMemory, VectorStoreMemory, etc. (复杂) |
| **工具生态** | Python 函数 + 外部 MCP | LangChain Tools (1000+ integrations) |
| **Provider 支持** | 100+ LLMs (via LiteLLM) | 50+ LLMs (内置) |
| **Streaming** | ✅ 一等公民 | ✅ 支持但较复杂 |
| **可观测性** | Tracing Protocol (extensible) | LangSmith (托管服务) + Callbacks |
| **依赖项** | 轻量 (~10 deps) | 重量 (50+ deps) |

**何时选择 OpenAI Agents SDK**:
- ✅ 专注 multi-agent 编排
- ✅ 需要轻量级、可控的解决方案
- ✅ 想要 provider-agnostic
- ✅ 不需要 RAG/Vector stores

**何时选择 LangChain**:
- ✅ 需要 RAG、Vector stores、Document loaders
- ✅ 需要丰富的工具集成（1000+ tools）
- ✅ 构建复杂的 LLM 应用（不仅是 agents）
- ✅ 团队已熟悉 LangChain 生态

### 4.3 Feature Matrix (全面对比)

| Feature | OpenAI Agents SDK | Claude Agent SDK | LangChain | Swarm (deprecated) |
|---------|------------------|-----------------|-----------|-------------------|
| **Multi-agent** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ |
| **Handoffs** | ✅ ✅ ✅ (first-class) | ✅ ✅ (via hooks) | ✅ (via tools) | ✅ |
| **Sessions** | ✅ ✅ ✅ | ❌ | ✅ ✅ | ❌ |
| **Tracing** | ✅ ✅ ✅ | ❌ | ✅ (LangSmith) | ❌ |
| **HITL** | ✅ ✅ ✅ (RunState) | ✅ ✅ (Hooks) | ✅ (Callbacks) | ❌ |
| **Streaming** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ | ✅ |
| **Guardrails** | ✅ ✅ ✅ | ✅ (Hooks) | ⚠️ (需自定义) | ❌ |
| **MCP Support** | ⚠️ (via tools) | ✅ ✅ ✅ (native) | ❌ | ❌ |
| **RAG** | ❌ | ❌ | ✅ ✅ ✅ | ❌ |
| **Vector Stores** | ❌ | ❌ | ✅ ✅ ✅ | ❌ |
| **Provider-agnostic** | ✅ ✅ ✅ | ❌ | ✅ ✅ | ⚠️ (基础) |
| **Production-ready** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ | ❌ (教育性) |
| **License** | MIT | MIT + ToS | MIT | MIT |
| **Stars** | 19.2k | 5k | 100k+ | 21k |

---

## 5. 扩展开发指南

### 5.1 自定义工具开发

#### 5.1.1 基础函数工具

```python
from agents import function_tool, ToolContext

# ═══════════════════════════════════════════════════════════
# 简单工具（无 context）
# ═══════════════════════════════════════════════════════════
@function_tool
def get_weather(city: str) -> str:
    """Get current weather for a city.
    
    Args:
        city: The city name
    """
    # 调用天气 API
    return f"Weather in {city}: 72°F, sunny"

# ═══════════════════════════════════════════════════════════
# 带 context 的工具
# ═══════════════════════════════════════════════════════════
from typing import TypedDict

class MyContext(TypedDict):
    user_id: str
    api_key: str

@function_tool
def get_user_profile(
    context: RunContextWrapper[MyContext],
    field: str
) -> str:
    """Get user profile field.
    
    Args:
        field: The field to retrieve (e.g., 'name', 'email')
    """
    user_id = context.context["user_id"]
    api_key = context.context["api_key"]
    
    # 调用用户 API
    profile = fetch_user_profile(user_id, api_key)
    return profile.get(field, "Not found")

# ═══════════════════════════════════════════════════════════
# 异步工具
# ═══════════════════════════════════════════════════════════
@function_tool
async def search_database(query: str) -> str:
    """Search the database.
    
    Args:
        query: SQL query to execute
    """
    async with async_db_connection() as conn:
        results = await conn.execute(query)
        return json.dumps(results)

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
agent = Agent(
    name="Assistant",
    tools=[get_weather, get_user_profile, search_database]
)

context = MyContext(user_id="user_123", api_key="sk-...")
result = await Runner.run(agent, "What's the weather in SF?", context=context)
```

#### 5.1.2 需要审批的工具

```python
from agents import function_tool

# ═══════════════════════════════════════════════════════════
# 静态审批需求
# ═══════════════════════════════════════════════════════════
@function_tool(needs_approval=True)
async def delete_user(user_id: str) -> str:
    """Delete a user account.
    
    Args:
        user_id: The user ID to delete
    """
    # 执行删除
    await db.users.delete(user_id)
    return f"Deleted user {user_id}"

# ═══════════════════════════════════════════════════════════
# 动态审批逻辑
# ═══════════════════════════════════════════════════════════
async def should_approve_payment(
    context: RunContextWrapper[Any],
    params: dict[str, Any],
    call_id: str
) -> bool:
    """动态决定是否需要审批"""
    amount = params.get("amount", 0)
    
    # 小额支付自动批准
    if amount < 100:
        return True
    
    # 大额支付需要审批
    return False

@function_tool(needs_approval=should_approve_payment)
async def process_payment(amount: float, recipient: str) -> str:
    """Process a payment.
    
    Args:
        amount: Payment amount in USD
        recipient: Recipient user ID
    """
    # 处理支付
    tx_id = await payment_api.transfer(amount, recipient)
    return f"Paid ${amount} to {recipient}. Transaction: {tx_id}"

# ═══════════════════════════════════════════════════════════
# HITL 工作流
# ═══════════════════════════════════════════════════════════
agent = Agent(
    name="Payment Bot",
    tools=[process_payment]
)

# 第一次运行
result = await Runner.run(agent, "Pay $500 to user_456")

if result.interruptions:
    print(f"Approval needed: {result.interruptions}")
    # [ToolApprovalInterruption(tool_name='process_payment', arguments={'amount': 500, ...})]
    
    # 用户批准
    run_state = result.to_state()
    run_state.approve(result.interruptions[0])
    
    # 恢复执行
    final_result = await Runner.run(agent, run_state)
    print(final_result.final_output)
```

#### 5.1.3 高级 FunctionTool

```python
from agents import FunctionTool, ToolContext
import asyncio

# ═══════════════════════════════════════════════════════════
# 手动创建 FunctionTool（完全控制）
# ═══════════════════════════════════════════════════════════
async def my_tool_impl(context: ToolContext, args_json: str) -> str:
    import json
    args = json.loads(args_json)
    
    # 访问 context
    print(f"Called by agent: {context.agent.name}")
    print(f"Current usage: {context.usage}")
    
    # 执行逻辑
    await asyncio.sleep(1)  # 模拟耗时操作
    
    return f"Processed: {args}"

tool = FunctionTool(
    name="my_advanced_tool",
    description="Does advanced processing",
    params_json_schema={
        "type": "object",
        "properties": {
            "input": {"type": "string", "description": "Input data"},
            "mode": {"type": "string", "enum": ["fast", "thorough"]},
        },
        "required": ["input"],
        "additionalProperties": False,
    },
    on_invoke_tool=my_tool_impl,
    strict_json_schema=True,  # OpenAI Structured Outputs
    timeout_seconds=30.0,
    tool_input_guardrails=[...],  # 可选 guardrails
    tool_output_guardrails=[...],
)

agent = Agent(tools=[tool])
```

### 5.2 自定义 Session Backend

#### 5.2.1 基础 Session 实现

```python
from agents import SessionABC, TResponseInputItem
from typing import Literal
import asyncio

class MemorySession(SessionABC):
    """In-memory session (用于测试)"""
    
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.session_settings = None
        self._items: list[TResponseInputItem] = []
        self._lock = asyncio.Lock()
    
    async def get_items(
        self, 
        limit: int | None = None
    ) -> list[TResponseInputItem]:
        async with self._lock:
            if limit is None or limit <= 0:
                return self._items.copy()
            else:
                return self._items[-limit:]
    
    async def add_items(
        self, 
        items: list[TResponseInputItem]
    ) -> None:
        async with self._lock:
            self._items.extend(items)
    
    async def pop_item(self) -> TResponseInputItem | None:
        async with self._lock:
            if self._items:
                return self._items.pop()
            return None
    
    async def clear_session(self) -> None:
        async with self._lock:
            self._items.clear()
```

#### 5.2.2 Redis Session (生产级)

```python
from agents import SessionABC, TResponseInputItem, SessionSettings
from redis.asyncio import Redis
import json
import asyncio

class RedisSession(SessionABC):
    """Redis-backed session"""
    
    def __init__(
        self,
        session_id: str,
        *,
        redis_client: Redis,
        ttl_seconds: int | None = None,
        session_settings: SessionSettings | None = None,
    ):
        self.session_id = session_id
        self._redis = redis_client
        self._ttl = ttl_seconds
        self.session_settings = session_settings
        self._lock = asyncio.Lock()
        
        # Redis keys
        self._messages_key = f"agents:session:{session_id}:messages"
        self._counter_key = f"agents:session:{session_id}:counter"
    
    async def get_items(
        self, 
        limit: int | None = None
    ) -> list[TResponseInputItem]:
        from agents.memory.session import resolve_session_limit
        limit = resolve_session_limit(limit, self.session_settings)
        
        async with self._lock:
            if limit is None or limit <= 0:
                # 获取全部
                raw_items = await self._redis.lrange(
                    self._messages_key, 0, -1
                )
            else:
                # 获取最后 N 条
                raw_items = await self._redis.lrange(
                    self._messages_key, -limit, -1
                )
            
            items = [
                json.loads(item.decode('utf-8')) 
                for item in raw_items
            ]
            return items
    
    async def add_items(
        self, 
        items: list[TResponseInputItem]
    ) -> None:
        async with self._lock:
            pipe = self._redis.pipeline()
            
            for item in items:
                serialized = json.dumps(item).encode('utf-8')
                pipe.rpush(self._messages_key, serialized)
            
            # 设置 TTL
            if self._ttl:
                pipe.expire(self._messages_key, self._ttl)
            
            await pipe.execute()
    
    async def pop_item(self) -> TResponseInputItem | None:
        async with self._lock:
            raw_item = await self._redis.rpop(self._messages_key)
            if raw_item:
                return json.loads(raw_item.decode('utf-8'))
            return None
    
    async def clear_session(self) -> None:
        async with self._lock:
            await self._redis.delete(self._messages_key)
            await self._redis.delete(self._counter_key)
    
    @classmethod
    async def from_url(
        cls,
        session_id: str,
        url: str = "redis://localhost:6379/0",
        **kwargs
    ) -> "RedisSession":
        """从 Redis URL 创建 session"""
        redis_client = await Redis.from_url(url, decode_responses=False)
        return cls(session_id, redis_client=redis_client, **kwargs)

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
session = await RedisSession.from_url(
    "user_123",
    url="redis://localhost:6379/0",
    ttl_seconds=86400,  # 24 hours
)

result = await Runner.run(agent, "Hello", session=session)
```

#### 5.2.3 加密 Session

```python
from cryptography.fernet import Fernet
from agents import SessionABC, TResponseInputItem
import json

class EncryptedSession(SessionABC):
    """加密存储的 Session"""
    
    def __init__(
        self,
        session_id: str,
        encryption_key: bytes,
        backend_session: SessionABC,
    ):
        self.session_id = session_id
        self.session_settings = backend_session.session_settings
        self._cipher = Fernet(encryption_key)
        self._backend = backend_session
    
    def _encrypt_item(self, item: TResponseInputItem) -> TResponseInputItem:
        """加密单个 item"""
        # 只加密 content 字段
        encrypted_item = item.copy()
        
        if "content" in encrypted_item:
            original_content = json.dumps(encrypted_item["content"])
            encrypted_content = self._cipher.encrypt(original_content.encode())
            encrypted_item["content"] = encrypted_content.decode('utf-8')
        
        return encrypted_item
    
    def _decrypt_item(self, item: TResponseInputItem) -> TResponseInputItem:
        """解密单个 item"""
        decrypted_item = item.copy()
        
        if "content" in decrypted_item:
            encrypted_content = decrypted_item["content"]
            decrypted_content = self._cipher.decrypt(encrypted_content.encode())
            decrypted_item["content"] = json.loads(decrypted_content.decode())
        
        return decrypted_item
    
    async def get_items(
        self, 
        limit: int | None = None
    ) -> list[TResponseInputItem]:
        encrypted_items = await self._backend.get_items(limit)
        return [self._decrypt_item(item) for item in encrypted_items]
    
    async def add_items(
        self, 
        items: list[TResponseInputItem]
    ) -> None:
        encrypted_items = [self._encrypt_item(item) for item in items]
        await self._backend.add_items(encrypted_items)
    
    async def pop_item(self) -> TResponseInputItem | None:
        encrypted_item = await self._backend.pop_item()
        if encrypted_item:
            return self._decrypt_item(encrypted_item)
        return None
    
    async def clear_session(self) -> None:
        await self._backend.clear_session()

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
# 生成密钥
encryption_key = Fernet.generate_key()

# 包装 SQLite session
base_session = SQLiteSession("user_123")
encrypted_session = EncryptedSession("user_123", encryption_key, base_session)

result = await Runner.run(agent, "Sensitive data", session=encrypted_session)
```

### 5.3 自定义 Tracing Processor

#### 5.3.1 基础 Processor

```python
from agents import TracingProcessor, Trace, Span
from agents.tracing.span_data import AgentSpanData, GenerationSpanData, FunctionSpanData
from datetime import datetime

class ConsoleTracingProcessor(TracingProcessor):
    """输出到控制台的 Tracing Processor"""
    
    def __init__(self, verbose: bool = True):
        self.verbose = verbose
        self.indent_level = 0
    
    def _print(self, msg: str):
        indent = "  " * self.indent_level
        print(f"{indent}{msg}")
    
    def on_trace_start(self, trace: Trace) -> None:
        self._print(f"🚀 TRACE START: {trace.name} [{trace.trace_id}]")
        if trace.metadata:
            self._print(f"   Metadata: {trace.metadata}")
        self.indent_level += 1
    
    def on_trace_end(self, trace: Trace) -> None:
        self.indent_level -= 1
        self._print(f"✅ TRACE END: {trace.name}")
        
        # 导出统计
        data = trace.export()
        if data:
            total_tokens = sum(
                span.get("prompt_tokens", 0) + span.get("completion_tokens", 0)
                for span in data.get("spans", [])
                if span.get("type") == "generation"
            )
            self._print(f"   Total tokens: {total_tokens}")
    
    def on_span_start(self, span: Span) -> None:
        span_data = span.span_data
        
        if isinstance(span_data, AgentSpanData):
            self._print(f"🤖 AGENT: {span_data.agent_name} (model: {span_data.agent_model})")
        elif isinstance(span_data, GenerationSpanData):
            self._print(f"💭 LLM CALL: {span_data.model}")
        elif isinstance(span_data, FunctionSpanData):
            self._print(f"🔧 TOOL: {span_data.tool_name}")
            if self.verbose:
                self._print(f"   Args: {span_data.arguments}")
        
        self.indent_level += 1
    
    def on_span_end(self, span: Span) -> None:
        self.indent_level -= 1
        
        span_data = span.span_data
        
        if isinstance(span_data, GenerationSpanData):
            self._print(
                f"   ⏱️  Latency: {span_data.latency_ms:.2f}ms, "
                f"Tokens: {span_data.total_tokens}"
            )
        elif isinstance(span_data, FunctionSpanData):
            if span_data.error:
                self._print(f"   ❌ Error: {span_data.error}")
            elif self.verbose:
                self._print(f"   ✓ Result: {span_data.result[:100]}...")
    
    def shutdown(self) -> None:
        self._print("🛑 SHUTDOWN")
    
    def force_flush(self) -> None:
        pass  # Console doesn't need flushing

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
from agents import set_trace_processors

processor = ConsoleTracingProcessor(verbose=True)
set_trace_processors([processor])

result = await Runner.run(agent, "What's 2+2?")

# 输出:
# 🚀 TRACE START: workflow_name [trace_abc123]
#   🤖 AGENT: Assistant (model: gpt-4o)
#     💭 LLM CALL: gpt-4o
#       ⏱️  Latency: 345.67ms, Tokens: 25
# ✅ TRACE END: workflow_name
#    Total tokens: 25
```

#### 5.3.2 数据库 Tracing Processor

```python
from agents import TracingProcessor, Trace, Span
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy import Column, Integer, String, JSON, DateTime, Float
from datetime import datetime
import json

Base = declarative_base()

class TraceRecord(Base):
    __tablename__ = "traces"
    
    id = Column(Integer, primary_key=True)
    trace_id = Column(String, unique=True, index=True)
    workflow_name = Column(String)
    metadata = Column(JSON)
    data = Column(JSON)
    created_at = Column(DateTime, default=datetime.utcnow)

class SpanRecord(Base):
    __tablename__ = "spans"
    
    id = Column(Integer, primary_key=True)
    span_id = Column(String, unique=True, index=True)
    trace_id = Column(String, index=True)
    span_type = Column(String)
    data = Column(JSON)
    latency_ms = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)

class DatabaseTracingProcessor(TracingProcessor):
    """将 traces 存储到数据库"""
    
    def __init__(self, db_url: str):
        self.engine = create_async_engine(db_url)
        self.SessionLocal = sessionmaker(
            self.engine, 
            class_=AsyncSession, 
            expire_on_commit=False
        )
        
        # 创建表
        asyncio.create_task(self._create_tables())
    
    async def _create_tables(self):
        async with self.engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
    
    def on_trace_start(self, trace: Trace) -> None:
        pass  # 等待 trace_end 时一次性写入
    
    def on_trace_end(self, trace: Trace) -> None:
        asyncio.create_task(self._save_trace(trace))
    
    async def _save_trace(self, trace: Trace):
        data = trace.export()
        if not data:
            return
        
        async with self.SessionLocal() as session:
            record = TraceRecord(
                trace_id=trace.trace_id,
                workflow_name=trace.name,
                metadata=trace.metadata or {},
                data=data,
            )
            session.add(record)
            await session.commit()
    
    def on_span_start(self, span: Span) -> None:
        pass  # 等待 span_end
    
    def on_span_end(self, span: Span) -> None:
        asyncio.create_task(self._save_span(span))
    
    async def _save_span(self, span: Span):
        from agents.tracing.span_data import GenerationSpanData, FunctionSpanData
        
        span_data = span.span_data
        span_type = type(span_data).__name__
        
        # 提取 latency
        latency_ms = None
        if isinstance(span_data, (GenerationSpanData, FunctionSpanData)):
            latency_ms = span_data.latency_ms
        
        async with self.SessionLocal() as session:
            record = SpanRecord(
                span_id=span.span_id,
                trace_id=span.trace.trace_id if span.trace else None,
                span_type=span_type,
                data=span_data.__dict__,
                latency_ms=latency_ms,
            )
            session.add(record)
            await session.commit()
    
    def shutdown(self) -> None:
        asyncio.create_task(self.engine.dispose())
    
    def force_flush(self) -> None:
        pass  # 已经是即时写入

# ═══════════════════════════════════════════════════════════
# 查询 API
# ═══════════════════════════════════════════════════════════
class TraceQuery:
    def __init__(self, db_url: str):
        self.engine = create_async_engine(db_url)
        self.SessionLocal = sessionmaker(
            self.engine, 
            class_=AsyncSession, 
            expire_on_commit=False
        )
    
    async def get_trace(self, trace_id: str) -> dict | None:
        from sqlalchemy import select
        
        async with self.SessionLocal() as session:
            result = await session.execute(
                select(TraceRecord).where(TraceRecord.trace_id == trace_id)
            )
            record = result.scalar_one_or_none()
            
            if record:
                return {
                    "trace_id": record.trace_id,
                    "workflow_name": record.workflow_name,
                    "metadata": record.metadata,
                    "data": record.data,
                    "created_at": record.created_at.isoformat(),
                }
            return None
    
    async def get_recent_traces(self, limit: int = 10) -> list[dict]:
        from sqlalchemy import select
        
        async with self.SessionLocal() as session:
            result = await session.execute(
                select(TraceRecord)
                .order_by(TraceRecord.created_at.desc())
                .limit(limit)
            )
            records = result.scalars().all()
            
            return [
                {
                    "trace_id": r.trace_id,
                    "workflow_name": r.workflow_name,
                    "created_at": r.created_at.isoformat(),
                }
                for r in records
            ]

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
db_url = "sqlite+aiosqlite:///traces.db"

# 设置 processor
processor = DatabaseTracingProcessor(db_url)
set_trace_processors([processor])

# 运行 agent
result = await Runner.run(agent, "Hello")

# 查询 traces
query = TraceQuery(db_url)
recent = await query.get_recent_traces(limit=5)
print(recent)
```

#### 5.3.3 集成外部服务（Logfire 示例）

```python
from agents import TracingProcessor, Trace, Span
import logfire

class LogfireTracingProcessor(TracingProcessor):
    """集成 Logfire 的 Tracing Processor"""
    
    def __init__(self, project_name: str):
        logfire.configure(project_name=project_name)
        self.current_trace_span = None
    
    def on_trace_start(self, trace: Trace) -> None:
        # 创建 Logfire trace
        self.current_trace_span = logfire.span(
            name=trace.name,
            trace_id=trace.trace_id,
            attributes=trace.metadata or {},
        )
        self.current_trace_span.__enter__()
    
    def on_trace_end(self, trace: Trace) -> None:
        if self.current_trace_span:
            self.current_trace_span.__exit__(None, None, None)
            self.current_trace_span = None
    
    def on_span_start(self, span: Span) -> None:
        from agents.tracing.span_data import GenerationSpanData, FunctionSpanData
        
        span_data = span.span_data
        span_type = type(span_data).__name__
        
        # 创建 Logfire span
        attributes = {"span_type": span_type}
        
        if isinstance(span_data, GenerationSpanData):
            attributes.update({
                "model": span_data.model,
                "prompt_tokens": span_data.prompt_tokens,
                "completion_tokens": span_data.completion_tokens,
            })
        elif isinstance(span_data, FunctionSpanData):
            attributes.update({
                "tool_name": span_data.tool_name,
                "arguments": str(span_data.arguments),
            })
        
        logfire.log(
            level="info",
            message=f"Span started: {span_type}",
            attributes=attributes,
        )
    
    def on_span_end(self, span: Span) -> None:
        from agents.tracing.span_data import GenerationSpanData, FunctionSpanData
        
        span_data = span.span_data
        
        attributes = {}
        if isinstance(span_data, GenerationSpanData):
            attributes["latency_ms"] = span_data.latency_ms
            attributes["total_tokens"] = span_data.total_tokens
        elif isinstance(span_data, FunctionSpanData):
            attributes["latency_ms"] = span_data.latency_ms
            if span_data.error:
                attributes["error"] = span_data.error
        
        logfire.log(
            level="info",
            message=f"Span ended",
            attributes=attributes,
        )
    
    def shutdown(self) -> None:
        pass  # Logfire handles shutdown
    
    def force_flush(self) -> None:
        pass  # Logfire handles flushing

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
processor = LogfireTracingProcessor(project_name="my-agents")
set_trace_processors([processor])

# Traces 会自动发送到 Logfire
result = await Runner.run(agent, "Hello")
```

### 5.4 自定义 Model Provider

```python
from agents import Model, ModelProvider, ModelResponse
from typing import AsyncIterator

# ═══════════════════════════════════════════════════════════
# 自定义 Model
# ═══════════════════════════════════════════════════════════
class MyCustomModel(Model):
    def __init__(self, model_name: str, api_key: str):
        self.model_name = model_name
        self.api_key = api_key
    
    async def get_response(
        self,
        *,
        system: str | None = None,
        messages: list[dict],
        tools: list[dict] | None = None,
        output_schema: dict | None = None,
        **kwargs
    ) -> ModelResponse:
        # 调用你的 LLM API
        response = await self._call_api(
            system=system,
            messages=messages,
            tools=tools,
            output_schema=output_schema,
        )
        
        return ModelResponse(
            output=response["output"],
            usage=response["usage"],
            response_id=response["id"],
        )
    
    async def stream_response(
        self,
        *,
        system: str | None = None,
        messages: list[dict],
        tools: list[dict] | None = None,
        **kwargs
    ) -> AsyncIterator[dict]:
        # 流式调用
        async for chunk in self._call_api_streaming(
            system=system,
            messages=messages,
            tools=tools,
        ):
            yield chunk
    
    async def _call_api(self, **kwargs) -> dict:
        # 实现你的 API 调用逻辑
        ...

# ═══════════════════════════════════════════════════════════
# 自定义 ModelProvider
# ═══════════════════════════════════════════════════════════
class MyCustomModelProvider(ModelProvider):
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    def get_model(self, model_name: str | None) -> Model:
        model_name = model_name or "default-model"
        return MyCustomModel(model_name, self.api_key)

# ═══════════════════════════════════════════════════════════
# 使用
# ═══════════════════════════════════════════════════════════
provider = MyCustomModelProvider(api_key="sk-...")

agent = Agent(
    name="Assistant",
    model=provider.get_model("my-model-v1"),
)

result = await Runner.run(agent, "Hello")
```

---

## 6. 生产部署考量

### 6.1 性能优化

**并发控制**:
```python
import asyncio
from agents import Agent, Runner

# ═══════════════════════════════════════════════════════════
# 1. 限制并发 Agent 运行数
# ═══════════════════════════════════════════════════════════
semaphore = asyncio.Semaphore(10)  # 最多 10 个并发

async def run_with_semaphore(agent, input, **kwargs):
    async with semaphore:
        return await Runner.run(agent, input, **kwargs)

# 批量运行
tasks = [
    run_with_semaphore(agent, input)
    for input in inputs
]
results = await asyncio.gather(*tasks)

# ═══════════════════════════════════════════════════════════
# 2. Session 连接池（Redis 示例）
# ═══════════════════════════════════════════════════════════
from redis.asyncio import ConnectionPool, Redis

pool = ConnectionPool.from_url(
    "redis://localhost:6379/0",
    max_connections=50,
    decode_responses=False,
)

async def get_session(session_id: str):
    redis = Redis(connection_pool=pool)
    return RedisSession(session_id, redis_client=redis)

# ═══════════════════════════════════════════════════════════
# 3. 工具超时控制
# ═══════════════════════════════════════════════════════════
@function_tool
async def slow_tool(query: str) -> str:
    try:
        async with asyncio.timeout(5.0):  # 5 秒超时
            result = await expensive_operation(query)
            return result
    except asyncio.TimeoutError:
        return "Operation timed out"
```

**缓存优化**:
```python
from functools import lru_cache
import hashlib

# ═══════════════════════════════════════════════════════════
# 工具结果缓存
# ═══════════════════════════════════════════════════════════
cache = {}

@function_tool
async def cached_search(query: str) -> str:
    """带缓存的搜索工具"""
    cache_key = hashlib.md5(query.encode()).hexdigest()
    
    if cache_key in cache:
        return cache[cache_key]
    
    result = await perform_search(query)
    cache[cache_key] = result
    return result

# ═══════════════════════════════════════════════════════════
# Session 缓存层
# ═══════════════════════════════════════════════════════════
class CachedSession(SessionABC):
    def __init__(self, backend: SessionABC, cache_size: int = 100):
        self.session_id = backend.session_id
        self.session_settings = backend.session_settings
        self._backend = backend
        self._cache = []
        self._cache_size = cache_size
        self._dirty = False
    
    async def get_items(self, limit: int | None = None) -> list[TResponseInputItem]:
        if not self._cache or self._dirty:
            self._cache = await self._backend.get_items()
            self._dirty = False
        
        if limit is None or limit <= 0:
            return self._cache.copy()
        else:
            return self._cache[-limit:]
    
    async def add_items(self, items: list[TResponseInputItem]) -> None:
        self._cache.extend(items)
        self._dirty = True
        
        # 定期写回
        if len(self._cache) > self._cache_size:
            await self._backend.add_items(items)
            self._dirty = False
```

### 6.2 错误处理与重试

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

# ═══════════════════════════════════════════════════════════
# 1. Model 调用重试
# ═══════════════════════════════════════════════════════════
class RetryableModel(Model):
    def __init__(self, base_model: Model):
        self.base_model = base_model
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type((TimeoutError, ConnectionError)),
    )
    async def get_response(self, **kwargs) -> ModelResponse:
        return await self.base_model.get_response(**kwargs)

# ═══════════════════════════════════════════════════════════
# 2. Graceful degradation
# ═══════════════════════════════════════════════════════════
@function_tool
async def search_with_fallback(query: str) -> str:
    """带降级的搜索工具"""
    try:
        # 主搜索引擎
        return await primary_search(query)
    except Exception as e:
        logger.warning(f"Primary search failed: {e}")
        try:
            # 备用搜索引擎
            return await fallback_search(query)
        except Exception as e2:
            logger.error(f"Fallback search failed: {e2}")
            return "Search unavailable. Please try again later."

# ═══════════════════════════════════════════════════════════
# 3. Circuit Breaker
# ═══════════════════════════════════════════════════════════
from pybreaker import CircuitBreaker

breaker = CircuitBreaker(fail_max=5, timeout_duration=60)

@function_tool
async def external_api_call(endpoint: str) -> str:
    """带 circuit breaker 的 API 调用"""
    @breaker
    async def _call():
        return await http_client.get(endpoint)
    
    try:
        return await _call()
    except CircuitBreakerError:
        return "Service temporarily unavailable"
```

### 6.3 监控与日志

```python
import logging
from prometheus_client import Counter, Histogram
import time

# ═══════════════════════════════════════════════════════════
# 1. Structured Logging
# ═══════════════════════════════════════════════════════════
import structlog

logger = structlog.get_logger()

@function_tool
async def monitored_tool(input: str) -> str:
    logger.info(
        "tool_called",
        tool_name="monitored_tool",
        input_length=len(input),
    )
    
    try:
        result = await process(input)
        logger.info(
            "tool_completed",
            tool_name="monitored_tool",
            result_length=len(result),
        )
        return result
    except Exception as e:
        logger.error(
            "tool_failed",
            tool_name="monitored_tool",
            error=str(e),
        )
        raise

# ═══════════════════════════════════════════════════════════
# 2. Prometheus Metrics
# ═══════════════════════════════════════════════════════════
agent_runs = Counter("agent_runs_total", "Total agent runs", ["agent_name", "status"])
agent_latency = Histogram("agent_latency_seconds", "Agent latency", ["agent_name"])

async def run_with_metrics(agent: Agent, input: str, **kwargs):
    start = time.time()
    
    try:
        result = await Runner.run(agent, input, **kwargs)
        agent_runs.labels(agent_name=agent.name, status="success").inc()
        return result
    except Exception as e:
        agent_runs.labels(agent_name=agent.name, status="error").inc()
        raise
    finally:
        duration = time.time() - start
        agent_latency.labels(agent_name=agent.name).observe(duration)

# ═══════════════════════════════════════════════════════════
# 3. Custom Tracing Metrics Processor
# ═══════════════════════════════════════════════════════════
from agents import TracingProcessor, Trace, Span
from agents.tracing.span_data import GenerationSpanData

token_usage = Counter(
    "llm_tokens_total",
    "LLM token usage",
    ["model", "type"]  # type: prompt/completion
)

class MetricsTracingProcessor(TracingProcessor):
    def on_span_end(self, span: Span) -> None:
        if isinstance(span.span_data, GenerationSpanData):
            data = span.span_data
            token_usage.labels(model=data.model, type="prompt").inc(data.prompt_tokens)
            token_usage.labels(model=data.model, type="completion").inc(data.completion_tokens)
    
    def on_trace_start(self, trace: Trace) -> None:
        pass
    
    def on_trace_end(self, trace: Trace) -> None:
        pass
    
    def on_span_start(self, span: Span) -> None:
        pass
    
    def shutdown(self) -> None:
        pass
    
    def force_flush(self) -> None:
        pass

set_trace_processors([MetricsTracingProcessor()])
```

### 6.4 Security Best Practices

```python
# ═══════════════════════════════════════════════════════════
# 1. Input Sanitization Guardrail
# ═══════════════════════════════════════════════════════════
from agents import input_guardrail, InputGuardrailResult, GuardrailFunctionOutput

@input_guardrail
async def sanitize_input(
    context: RunContextWrapper[Any],
    input_items: list[TResponseInputItem]
) -> InputGuardrailResult:
    """检查并清理恶意输入"""
    for item in input_items:
        if item.get("role") == "user":
            content = str(item.get("content", ""))
            
            # 检查注入攻击
            dangerous_patterns = ["<script>", "DROP TABLE", "eval("]
            for pattern in dangerous_patterns:
                if pattern in content:
                    return InputGuardrailResult(
                        output=GuardrailFunctionOutput(
                            tripwire_triggered=True,
                            tripwire_reason=f"Dangerous pattern detected: {pattern}"
                        )
                    )
    
    return InputGuardrailResult(
        output=GuardrailFunctionOutput(tripwire_triggered=False)
    )

# ═══════════════════════════════════════════════════════════
# 2. Output Filtering Guardrail
# ═══════════════════════════════════════════════════════════
from agents import output_guardrail

@output_guardrail
async def filter_pii(
    context: RunContextWrapper[Any],
    output: str | dict
) -> OutputGuardrailResult:
    """过滤 PII（个人身份信息）"""
    import re
    
    output_str = str(output)
    
    # 移除 SSN
    output_str = re.sub(r'\d{3}-\d{2}-\d{4}', '[SSN REDACTED]', output_str)
    
    # 移除信用卡号
    output_str = re.sub(r'\d{4}-\d{4}-\d{4}-\d{4}', '[CARD REDACTED]', output_str)
    
    return OutputGuardrailResult(
        output=GuardrailFunctionOutput(
            tripwire_triggered=False
        ),
        modified_output=output_str,
    )

# ═══════════════════════════════════════════════════════════
# 3. Rate Limiting
# ═══════════════════════════════════════════════════════════
from collections import defaultdict
from datetime import datetime, timedelta

class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window = timedelta(seconds=window_seconds)
        self.requests = defaultdict(list)
    
    async def check(self, user_id: str) -> bool:
        now = datetime.utcnow()
        cutoff = now - self.window
        
        # 清理过期请求
        self.requests[user_id] = [
            ts for ts in self.requests[user_id]
            if ts > cutoff
        ]
        
        # 检查限制
        if len(self.requests[user_id]) >= self.max_requests:
            return False
        
        self.requests[user_id].append(now)
        return True

rate_limiter = RateLimiter(max_requests=10, window_seconds=60)

async def run_with_rate_limit(agent: Agent, input: str, user_id: str, **kwargs):
    if not await rate_limiter.check(user_id):
        raise Exception("Rate limit exceeded")
    
    return await Runner.run(agent, input, **kwargs)

# ═══════════════════════════════════════════════════════════
# 4. Secrets Management
# ═══════════════════════════════════════════════════════════
from typing import TypedDict

class SecureContext(TypedDict):
    user_id: str
    # 不要在 context 中存储明文 secrets

# 使用环境变量或 secrets manager
import os

@function_tool
async def call_external_api(
    context: RunContextWrapper[SecureContext],
    endpoint: str
) -> str:
    # 从环境变量获取 API key
    api_key = os.getenv("EXTERNAL_API_KEY")
    
    # 或从 secrets manager
    # api_key = await secrets_manager.get_secret("external_api_key")
    
    return await http_client.get(
        endpoint,
        headers={"Authorization": f"Bearer {api_key}"}
    )
```

---

## 7. 评估与建议

!!! success "核心优势"
    **架构优势**:

    1. ✅ **清晰的关注点分离**: Runner → AgentRunner → run_internal/* 三层架构，职责明确
    2. ✅ **Protocol-based 扩展**: Session/Model/Tracing 使用 Protocol 而非 ABC，降低耦合
    3. ✅ **一等公民抽象**: Handoff、Session、Tracing 都是框架核心，非事后添加
    4. ✅ **完整的状态管理**: RunState 支持 HITL、中断恢复、序列化
    5. ✅ **生产就绪**: 内置 tracing、session、guardrails、错误处理

    **开发者体验**:

    1. ✅ **简单的入门**: `Runner.run(agent, input)` 即可开始
    2. ✅ **渐进式复杂度**: 从简单到复杂（基础 → handoff → session → tracing → HITL）
    3. ✅ **类型安全**: 广泛使用 type hints，IDE 支持良好
    4. ✅ **丰富示例**: 12,775 行示例代码，覆盖各种场景

    **生态优势**:

    1. ✅ **Provider-agnostic**: 100+ LLMs（不绑定 OpenAI）
    2. ✅ **活跃社区**: 19.2k stars, 219 contributors
    3. ✅ **官方维护**: OpenAI 团队积极维护（1,135 commits）

!!! warning "局限性"
    **功能限制**:

    1. ❌ **无 RAG 支持**: 需要自行实现 vector stores、embeddings、retrieval
    2. ❌ **无 MCP 原生支持**: 虽然可通过 tools 接入，但不如 Claude SDK 的 in-process MCP
    3. ⚠️ **有限的工具生态**: 依赖开发者自定义工具（vs. LangChain 的 1000+ tools）
    4. ⚠️ **Session 功能基础**: 仅支持 append-only 历史（无摘要、compaction 等高级功能）

    **设计权衡**:

    1. ⚠️ **轻量 vs. 功能**: 专注 agent 编排，牺牲了 RAG、chains 等 LangChain 特性
    2. ⚠️ **Handoff 单向性**: 一次只处理一个 handoff（无并行 handoff）
    3. ⚠️ **Tracing 开销**: 内置 tracing 会增加少量性能开销（可禁用）

    **生产考量**:

    1. ⚠️ **缺少内置速率限制**: 需自行实现
    2. ⚠️ **缺少内置重试**: 需手动包装（如 tenacity）
    3. ⚠️ **Session 扩展性**: SQLite 不适合高并发（推荐生产环境用 Redis/PostgreSQL）

!!! tip "推荐场景"
    - ✅ 构建 multi-agent 编排系统（客服、工作流自动化等）
    - ✅ 需要跨 LLM provider（不绑定特定供应商）
    - ✅ 需要轻量级、可控的 agent 框架
    - ✅ 需要内置 HITL（Human-in-the-Loop）
    - ✅ Python 应用中嵌入 agent 能力
    - ✅ 需要完整的 tracing/observability

!!! danger "不推荐场景"
    - ❌ 需要 RAG/Vector stores（选 LangChain 或自行集成）
    - ❌ 需要丰富的预构建工具（选 LangChain）
    - ❌ 只需要简单的 LLM 调用（直接用 OpenAI SDK）
    - ❌ 需要 TypeScript/JavaScript（用 Agents SDK JS/TS 版本）

!!! warning "需评估场景"
    - ⚠️ 大规模生产部署（需自行实现 rate limiting、circuit breaker 等）
    - ⚠️ 复杂的 memory 需求（Session 功能较基础）
    - ⚠️ 需要 UI（需自行构建，SDK 本身无 UI）

### 7.4 最佳实践

**开发阶段**:
1. 从简单 agent 开始，逐步添加 handoffs
2. 早期使用 `ConsoleTracingProcessor` 调试
3. 为关键操作添加 guardrails
4. 为需要审批的工具设置 `needs_approval=True`

**测试阶段**:
1. 使用 `MemorySession` 进行单元测试
2. 测试 HITL 流程（interruptions + resume）
3. 测试 handoff 链路（确保 input_filter 正确）
4. 测试 max_turns 限制

**生产阶段**:
1. 使用 Redis/PostgreSQL 作为 Session backend
2. 配置 tracing 到外部服务（Logfire/AgentOps 等）
3. 实现 rate limiting 和 circuit breaker
4. 添加 PII 过滤 guardrails
5. 监控 token 使用量（通过 tracing metrics）

**性能优化**:
1. 启用 parallel guardrails（与 model 调用并行）
2. 使用 connection pool（Redis/DB）
3. 为耗时工具设置超时
4. 考虑工具结果缓存
5. 限制并发 agent 数量（semaphore）

---

## 附录

### A. 关键文件速查表

| 文件路径 | 行数 | 关键内容 |
|---------|-----|---------|
| `src/agents/run.py` | 1623 | AgentRunner.run() 主循环 (lines 396-1329) |
| `src/agents/run_internal/run_loop.py` | 1623+ | run_single_turn(), get_new_response() |
| `src/agents/run_internal/tool_execution.py` | ~1400 | execute_function_tool_calls() |
| `src/agents/run_internal/turn_resolution.py` | 1623 | execute_handoffs(), process_model_response() |
| `src/agents/run_state.py` | 2384 | RunState 序列化/反序列化 |
| `src/agents/handoffs/__init__.py` | 334 | Handoff 类定义 |
| `src/agents/memory/session.py` | 150 | Session Protocol |
| `src/agents/tracing/processor_interface.py` | 142 | TracingProcessor Protocol |
| `src/agents/agent.py` | 890 | Agent 类定义 |
| `src/agents/tool.py` | 1288 | FunctionTool, @function_tool |

### B. 术语表

| 术语 | 定义 |
|------|------|
| **Agent** | 配置化的 LLM 实例（instructions + tools + handoffs + guardrails + model） |
| **Handoff** | Agent 间控制权转移的专用工具 |
| **Session** | 自动管理对话历史的后端（SQLite/Redis/自定义） |
| **RunState** | 可序列化的运行状态（用于 HITL） |
| **Guardrail** | 输入/输出验证函数 |
| **Tracing** | 自动记录 agent 运行的可观测性系统 |
| **HITL** | Human-in-the-Loop，人工参与的工作流 |
| **NextStep** | Turn 结束后的下一步（FinalOutput/Handoff/Interruption/RunAgain） |
| **Turn** | 一次完整的 LLM 调用 + 工具执行周期 |
| **Span** | Tracing 中的操作单元（Agent/Generation/Function/Handoff/Guardrail） |

### C. 参考资源

**官方文档**:
- GitHub: https://github.com/openai/openai-agents-python
- Docs: https://openai.github.io/openai-agents-python/
- Examples: https://github.com/openai/openai-agents-python/tree/main/examples

**社区资源**:
- Discussions: https://github.com/openai/openai-agents-python/discussions
- Issues: https://github.com/openai/openai-agents-python/issues

**相关项目**:
- Swarm (deprecated): https://github.com/openai/swarm
- Agents SDK JS/TS: https://github.com/openai/openai-agents-js
- LiteLLM: https://github.com/BerriAI/litellm

---

**文档版本**: 1.0  
**作者**: 深度技术调研团队  
**最后更新**: 2026-02-26  
