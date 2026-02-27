# AI Agent Framework Capability Differentiation: A Deep Code Audit

**[中文版本](BLOG.zh.md)** | English

---

> **TL;DR**: After auditing ~300KB of source code from 9 mainstream agent frameworks ([OpenAI Agents SDK](deep-dive/OpenAI-Agents-SDK-DEEP-DIVE.md), [Claude Agent SDK](deep-dive/Claude-Agent-SDK-Python-DEEP-DIVE.md), [Codex CLI](deep-dive/Codex-CLI-DEEP-DIVE.md), [OpenCode](deep-dive/OpenCode-DEEP-DIVE.md), [Kimi CLI](deep-dive/Kimi-CLI-DEEP-DIVE.md), [Gemini CLI](deep-dive/Gemini-CLI-DEEP-DIVE.md), [Qwen Code](deep-dive/Qwen-Code-DEEP-DIVE.md), [SWE-agent](deep-dive/SWE-agent-DEEP-DIVE.md), [OpenManus](deep-dive/OpenManus-DEEP-DIVE.md)), we discovered a counter-intuitive fact: **The core Agent Loop logic is highly similar across frameworks**. The real differences lie not in "which algorithm is used," but in **engineering capability combinations**—30+ features including state management, security controls, and protocol support. This "capability differentiation" creates an invisible gap, making it difficult for academia to leverage the most advanced agent capabilities.

> **Navigation**:
> - Sections 1-3: Code audit observations (facts)
> - Section 4: Specific manifestations of academic vs. industrial capability requirements (facts + inference)
> - Section 5: Future convergence trends (author's judgment)
> - Section 6: Framework selection and practice recommendations (actionable)
> - [Detailed Framework Comparison Matrix](COMPARISON.md)
> - [In-depth Source Code Analysis](deep-dive/)

---

## I. Debunking Myths: The Agent Core Loop is Highly Stable

> **Important Scope**: This article focuses on the **Single-Agent Tool-Calling Pattern**, which is the mainstream architecture across the 9 open-source frameworks surveyed. Multi-Agent orchestration (e.g., Planning, inter-agent communication), Planner-Executor separation, and other paradigms are outside the scope of this discussion.

### 1.1 Nine Frameworks, One Loop

Let me first present a surprising fact. Here are the core loop pseudocodes for the 9 frameworks:

**OpenAI Agents SDK** (Python, production-grade SDK):
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

**Claude Agent SDK** (Python, SDK + closed-source CLI):
```python
# _internal/query.py:172-232
while not done:
    response = await cli.communicate(messages)
    
    if response.type == "tool_use":
        # Processed through Hook system
        result = await handle_hooks("PreToolUse", response.tool)
        observation = await execute_tool(result)
        messages.append(observation)
    elif response.type == "stop":
        done = True
```

**Codex CLI** (Rust, enterprise-grade security):
```rust
// codex-rs/core/src/codex.rs:500-800
loop {
    let response = self.llm.generate(&context).await?;
    
    match response.finish_reason {
        FinishReason::ToolCalls(calls) => {
            // Security checks + approval
            self.approval_manager.evaluate(&calls).await?;
            let observations = self.execute_tools(calls).await?;
            context.extend(observations);
        }
        FinishReason::Stop => break,
    }
}
```

**SWE-agent** (Python, research framework):
```python
# sweagent/agent/agents.py:1265
while not step_output.done:
    step_output = self.step()
    # Inside step():
    #   thought, action = parse_react_output(llm_response)
    #   observation = execute_action(action)
```

**OpenManus** (Python, rapid experimentation):
```python
# app/agent/react.py:11-38
while self.current_step < self.max_steps:
    step_result = await self.step()  # think() -> act()
    if self.is_stuck():  # Detect repetitive responses
        self.handle_stuck_state()
```

**OpenCode** (TypeScript, 100% open source):
```typescript
// src/session/prompt.ts:274-724
while (true) {
    const result = await processor.process({ messages, tools });
    
    if (result === "stop") break;
    if (result === "compact") await compactContext();  // Feature: Auto-compaction
    
    // Handle subtasks, compaction, overflow, etc.
}
```

**See?** All frameworks follow the same pattern:

```
Input → Build Context → Call LLM → Parse Output → 
If tool calls exist → Execute tools → Add observations → Repeat
If no tool calls → Complete → Return result
```

This is the **Agent Loop**. Since the ReAct paper was published in 2022, **the essential logic has remained highly stable**.

### 1.2 So, Where Do the Differences Lie?

If the core loop is the same, why do these frameworks feel so different in actual use?

The answer is: **Capability combinations**.

Like an operating system, everyone uses Linux for the kernel (Loop), but the differences between distributions (frameworks) lie in:
- Which package manager? (Tool system)
- Which desktop environment? (UI/interaction mode)
- Which security mechanism? (Permission controls)
- Which file system? (State management)

Agent frameworks are the same. The core loop is the "kernel," while **capabilities** are "distribution customizations."

---

## II. The Four-Layer Capability Differentiation

Based on the code audit, I've categorized Agent framework differences into a **four-layer capability system**:

### Layer 1: Core Loop - Same Across All Frameworks

This is the **invisible infrastructure**. Regardless of which framework you use, you're working with the same logic:
- Maintain message history
- Call LLM
- Parse output
- Execute tools
- Repeat until completion

**Key Conclusion**: When choosing a framework, **don't be misled by "which algorithm is used."** All frameworks support ReAct, Function Calling, and even Tree-of-Thoughts (just different prompts).

### Layer 2: Engineering Capabilities - Differentiated Competition

This is the layer that **truly affects user experience**. I've grouped 30+ capabilities into 5 categories:

#### 2.1 State Management Capabilities

**Capability: Session Persistence**

| Framework | Implementation | Code Location |
|-----------|---------------|---------------|
| **OpenAI Agents SDK** | SQLite / Redis / Custom (Protocol-based) | `src/agents/memory/session.py:14-55` |
| **OpenCode** | SQLite (Drizzle ORM) | `src/storage/database.ts` |
| **Codex CLI** | JSONL files + SQLite metadata | `codex-rs/core/src/state_db.rs` |
| **Kimi CLI** | JSONL files | `src/session.py:context_file` |
| **SWE-agent** | Trajectory files only (research-oriented) | `trajectory.jsonl` |
| **OpenManus** | ❌ None (In-memory only) | - |

**Why it matters**: Production environments must support conversation history persistence. If an Agent crashes, users expect to resume the conversation, not start over.

**Academic Dilemma**: SWE-agent and OpenManus lack general production-grade Session management (SWE-agent focuses on trajectory persistence), making research code difficult to deploy directly in production.

---

**Capability: State Serialization and HITL (Human-in-the-Loop)**

This is the **hallmark of production-grade frameworks**.

**OpenAI Agents SDK** implementation (most complete):
```python
# src/agents/run_state.py:2384 lines
class RunState(Generic[TContext]):
    """Serializable run state supporting interruption and resumption"""
    input: list[TResponseInputItem]
    output: list[RunItem]
    _current_step: NextStep  # Current execution step
    _last_processed_response: ProcessedResponse
    
    def approve(self, interruption: Interruption) -> None:
        """Resume after human approval"""
        
    def to_state_dict(self) -> dict:
        """Serialize to JSON for storage/restoration"""
```

**Usage Scenarios**:
1. Agent proposes executing `rm -rf /`, system pauses for human confirmation
2. User reviews and chooses "Reject" or "Execute with modifications"
3. Resume from the pause point without rerunning

**Adoption Rate**: Currently, only OpenAI SDK provides complete built-in support. Claude SDK can partially implement via Hooks; other frameworks mostly require custom implementation.

---

**Capability: Context Compaction**

**OpenCode** representative implementation:
```typescript
// src/session/prompt.ts:274-724
if (await SessionCompaction.isOverflow(sessionID)) {
    // Auto-trigger compaction
    await SessionCompaction.create({
        sessionID,
        reason: "token_limit_approaching"
    });
    continue;  // Re-loop with compacted context
}
```

**Why it matters**: Long-context LLMs (like Gemini 1M tokens) exist, but cost remains a concern. OpenCode's Smart Compaction helps reduce token consumption in long-session scenarios (specific benefits depend on task and configuration).

**Academic Dilemma**: Academia typically uses short conversations (< 10 turns), with lower compaction needs; but production long conversations (> 50 turns) often require this capability more.

---

#### 2.2 Security Control Capabilities

This is the **core differentiation point for enterprise frameworks**. Five security models coexist:

**Model 1: No Security**
- **OpenManus**: Relies on runtime environment security
- **Applicable scenarios**: Controlled academic research environments

**Model 2: Guardrails**
- **OpenAI Agents SDK**:
```python
@input_guardrail
async def check_pii(context, input_items):
    """Check for personally identifiable information"""
    
@output_guardrail  
async def check_sensitive_output(context, output):
    """Check if output is sensitive"""

@tool_guardrail
def dangerous_tool_guardrail(context, tool_call):
    """Check if tool call is dangerous"""
```

- **Characteristics**: Three-layer protection (input/output/tools), Python decorator implementation
- **Limitations**: Pure software layer, cannot block underlying system calls

---

**Model 3: Hooks**
- **Claude Agent SDK** (representative implementation):
```python
# 10+ hook events
hooks = {
    "PreToolUse": [check_safety],           # Before tool execution
    "PostToolUse": [log_result],            # After tool execution
    "UserPromptSubmit": [check_prompt],     # When user submits
    "SubagentStart": [init_subagent],       # Subagent startup
    "SubagentStop": [cleanup_subagent],     # Subagent stop
    # ... more
}
```

- **Characteristics**: Ultimate flexibility, can intervene at any step
- **Cost**: Requires deep developer involvement, high learning curve

---

**Model 4: Policy Engine**
- **OpenCode** (Wildcard Pattern):
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

- **Gemini CLI** (TOML Policy):
```toml
[policy]
approval_mode = "manual"
trusted_folders = ["/home/user/safe"]

[[policy.rules]]
tool = "shell"
pattern = "sudo *"
action = "deny"
```

- **Characteristics**: Declarative configuration, Last Match Wins
- **Advantages**: Non-developers can understand and modify

---

**Model 5: Sandbox Isolation**
- **Codex CLI** (representative implementation):
```rust
// codex-rs/core/src/seatbelt.rs (macOS)
// codex-rs/core/src/landlock.rs (Linux)  
// codex-rs/core/src/windows_sandbox.rs (Windows)

pub struct SandboxPolicy {
    pub policy_type: PolicyType,
    pub writable_roots: Vec<PathBuf>,  // Whitelist of writable directories
    pub network_access: bool,          // Network access control
}

// Three-layer security
1. Platform Sandbox (OS-level process isolation)
2. Approval Policy (.rules file, command-level control)
3. Network Control (protocol/host/port-level control)
```

- **Characteristics**: Provides the most complete OS-level isolation capability among surveyed frameworks
- **Cost**: Rust implementation, complex configuration, performance overhead

---

**Security Capability Summary**:

| Security Level | Representative Framework | Applicable Scenarios | Intrusiveness |
|---------------|-------------------------|---------------------|---------------|
| Level 0 | OpenManus | Academic research | None |
| Level 1 | OpenAI SDK | General applications | Low (decorators) |
| Level 2 | Claude SDK | Needs flexible control | Medium (Hooks) |
| Level 3 | OpenCode, Gemini | Configurable policies | Low (config files) |
| Level 4 | Codex CLI | Enterprise/Finance/Healthcare | High (Sandbox) |

**Academic Dilemma**: Academia typically uses Level 0-1, but industry (especially finance, healthcare) needs Level 3-4. This makes academic code difficult to deploy directly.

---

#### 2.3 Tool System Capabilities

**Capability: MCP (Model Context Protocol) Support Depth**

| Depth Level | Framework | Implementation |
|------------|-----------|---------------|
| **Level 0** | SWE-agent | ❌ Not supported |
| **Level 1** | OpenAI SDK | ⚠️ Can connect to external MCP via tools |
| **Level 2** | OpenCode, Kimi, Gemini, Qwen | ✅ MCP Client (connects to external servers) |
| **Level 3** | OpenManus | ✅ MCP Client + Server (FastMCP) |
| **Level 4** | Claude SDK | ✅✅✅ SDK MCP (In-Process, zero overhead) |
| **Level 5** | Codex CLI | ✅✅✅✅ Dual roles (Client + Server) |

**Claude SDK's SDK MCP** (innovative):
```python
# Run MCP server in Python process, no subprocess needed
@tool("fibonacci", "Calculate fibonacci", {"n": int})
async def fibonacci(args):
    return {"content": [{"type": "text", "text": f"fib({args['n']})"}]}

server = create_sdk_mcp_server("math-tools", tools=[fibonacci])
```

**Why it matters**: Traditional MCP often requires launching subprocesses (npx, python, etc.), which brings additional startup and communication overhead. Claude's SDK MCP runs directly in-process, significantly reducing this overhead.

---

**Capability: Parallel Tool Execution**

**OpenAI Agents SDK**:
```python
# Automatic parallelization (asyncio.gather)
await asyncio.gather(
    *[execute_tool(tc) for tc in response.tool_calls]
)
```

**SWE-agent/OpenManus**:
```python
# Sequential execution
for tool_call in response.tool_calls:  # Execute one by one
    result = execute_tool(tool_call)
```

**Performance difference**: If an Agent needs to "query weather, check calendar, search email" simultaneously, parallel execution is 2-3x faster.

**Academic Dilemma**: Academic research typically uses sequential execution (easier to analyze execution order), but production environments often need parallel execution more.

---

#### 2.4 Observability Capabilities

**Capability: Structured Tracing**

**OpenAI Agents SDK** (most complete):
```python
# 6 Span types
AgentSpan        # Agent execution
GenerationSpan   # LLM call
FunctionSpan     # Tool call
HandoffSpan      # Agent handoff
GuardrailSpan    # Security check
CustomSpan       # Custom

# Export to 6+ platforms
set_trace_processors([
    LogfireProcessor(),      # Logfire
    AgentOpsProcessor(),     # AgentOps
    BraintrustProcessor(),   # Braintrust
    # ...
])
```

**Why it matters**: You can't improve what you can't measure. Production environments typically need to know:
- How much did each LLM call cost?
- Which tool is the slowest?
- Why is the Agent stuck in a loop?

**Adoption Rate**: Only OpenAI SDK provides relatively complete built-in structured Tracing; other frameworks mostly have experimental telemetry, hooks, or basic logging.

**Academic Dilemma**: Academia uses Trajectory files (SWE-agent), but those are for humans to read, not for machine analysis.

---

#### 2.5 Performance Optimization Capabilities

**Capability: Streaming**

**Widespread adoption**: 7/9 frameworks support it.

**Importance**: User experience. If an Agent thinks for 10 seconds before outputting all at once, users feel "it's frozen." Streaming output lets users see "thinking..."

---

**Capability: Smart Caching**

**Current Status**: **No mature built-in implementation observed** in this survey scope.

**Why it's needed**:
```python
# Scenario: Agent calls the same tool multiple times
get_weather(city="Beijing")  # 1st time, call API
# ... 10 turns later ...
get_weather(city="Beijing")  # 2nd time, should use cache, but no framework supports it
```

**Academic Opportunity**: This is an **unmet need**. Implementing Tool Result Cache (with TTL and invalidation strategies) is a valuable contribution.

---

### Layer 3: Protocol Support - Ecosystem Compatibility

This layer isn't determined by individual frameworks, but by **industry standards**.

#### 3.1 MCP (Model Context Protocol)

**Current Status**: 8/9 frameworks support it, but at different depths (see Section 2.3).

**Why it matters**: MCP is becoming the "USB interface" of the Agent ecosystem. Once tools are published as MCP servers, all MCP-supporting frameworks can use them.

**Academic Opportunity**: MCP-ize academic benchmarks (like SWE-Bench) evaluation tools, enabling fair comparison with industry frameworks.

---

#### 3.2 ACP (Agent Client Protocol)

**Current Status**: Kimi CLI has the most complete ACP support; OpenCode also has ACP support, but ecosystem and maturity are still evolving.

**Usage**: IDE integration. Enables VS Code, Zed, JetBrains to uniformly access different Agents.

**Prospects**: May become the standard for IDE integration, but currently early-stage ecosystem.

---

### Layer 4: Interaction Mode - The Surface Layer

This is what **users directly perceive**:

| Mode | Framework | Applicable Scenarios |
|------|-----------|---------------------|
| **Python SDK** | OpenAI SDK, Claude SDK | Embedded applications, Jupyter Notebook |
| **CLI TUI** | OpenCode, Codex, Kimi | Terminal-heavy users |
| **Web UI** | Kimi, Qwen | Desktop users |
| **IDE Plugin** | Gemini, Qwen | Developer workflows |

**Note**: This is just the "frontend." The Layer 2-3 capabilities behind it are what matter.

---

## III. Analysis of Capability "Non-Uniformity"

### 3.1 No "Standard Capability Set"

Here is a **capability adoption heatmap** for each project (✅ = Full support, ⚠️ = Partial support, ❌ = No support):

| Capability | OpenAI SDK | Claude SDK | OpenCode | Codex | Kimi | Gemini | Qwen | SWE-agent | OpenManus |
|-----------|-----------|-----------|---------|-------|------|--------|------|-----------|-----------|
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

**Observations**:
1. **Each framework's capability combination is highly differentiated**
2. **Each framework has relatively unique focus capabilities**:
   - OpenAI SDK: HITL State, complete Tracing
   - Claude SDK: SDK MCP, 10+ Hooks
   - OpenCode: Context Compaction, Permission Ruleset
   - Codex: OS Sandbox, Approval Policy
   - Kimi: ACP Server
   - Qwen: Context Variable Templating
   - SWE-agent: Trajectory Recording
   - OpenManus: MCP dual roles (Client+Server)

### 3.2 Low Adoption Capabilities (Opportunities)

The following capabilities have **no mature built-in implementation observed in this survey scope**, but may be needed in production environments:

1. **Circuit Breaker**: Prevents cascading failures
2. **Tool Result Cache**: Avoids repeated calls
3. **Semantic Memory**: Long-term memory (not simple message history)
4. **Multi-Modal Tool**: Unified handling of text/image/audio tools

**Academic Opportunities**: These are valuable research/engineering directions.

---

## IV. Academic vs. Industrial Capability Requirements: Observations from the Audit

> This section analyzes the different requirement patterns for Agent framework capabilities between academic and industrial scenarios, based on code audit and comparison table observations. These differences may cause friction when migrating academic results to industry.

### 4.1 Academia's "Minimum Viable Implementation"

Academia tends to choose frameworks with **fewer capabilities**:

**SWE-agent** (research preferred):
- ✅ ReAct Loop (easy to analyze)
- ✅ Trajectory Recording (paper material)
- ❌ Session (persistence not needed)
- ❌ Security (controlled environment)
- ❌ MCP (self-contained system)
- ❌ Parallel Tools (sequential is easier to analyze)

**OpenManus** (rapid experimentation):
- ✅ Simple ReAct chain
- ✅ MCP dual roles (exploratory)
- ❌ Most other capabilities

**Why?**
- Papers focus on **algorithm innovation** (new Planning strategies)
- Don't need **engineering capabilities** (Session, Security, Tracing)
- Simplified experimental environment (easier to reproduce)

### 4.2 Industry's "Production-Grade Requirements"

Industry needs a **complete capability stack**:

**Finance/Healthcare Industry** (chooses Codex CLI):
- ✅ OS Sandbox (data isolation)
- ✅ Approval Policy (human approval)
- ✅ Network Control (prevents data leakage)
- ✅ Audit Trail (auditable)

**SaaS Products** (chooses OpenAI Agents SDK):
- ✅ Session (user conversation history)
- ✅ HITL (human intervention)
- ✅ Tracing (cost monitoring)
- ✅ Provider-agnostic (avoids lock-in)

**Why?**
- Production environments cannot crash, leak data, or go out of control
- Need observability (know what happened)
- Need cost control (Token = money)

### 4.3 Problems Caused by the Differences

Based on the above capability comparisons, the following potential problems can be inferred:

**Scenario 1: Paper Reproduction Friction**
- Paper implements a new algorithm using SWE-agent
- Industry wants to reproduce using OpenAI SDK, finds:
  - No Trajectory Recording (cannot directly compare)
  - Need to add Session, Security, Tracing (additional engineering work)
- **Possible outcome**: Good algorithm innovation, but migrating to production requires extra engineering investment

**Scenario 2: Benchmark Comparability Challenges**
- SWE-Bench uses SWE-agent's tools (`str_replace_editor`)
- Other frameworks (OpenAI SDK, Codex) difficult to run under same tool conditions
- **Possible outcome**: Paper comparison comparability limited, "my framework is better than yours" conclusions may lack credibility

**Scenario 3: Advanced Capability Availability Differences**
- Claude Code (closed source) has powerful Coding Agent capabilities
- Academia cannot use (not reproducible)
- Claude Agent SDK (open source) has SDK MCP, 10+ Hooks
- But depends on closed-source CLI, academia may hesitate
- **Possible outcome**: Academia may tend toward simpler frameworks (SWE-agent, OpenManus), rather than frameworks with most complete engineering capabilities

---

## V. Future Prediction: Will Capabilities Converge? (Author's Judgment)

> This section is trend judgment, based on preceding facts, not equivalent to data-validated conclusions.

### 5.1 Layer 2 (Engineering Capabilities): Difficult to Converge in the Short Term

**Reasons**:
1. **Differentiated competition**: Frameworks establish moats through unique capabilities
   - Codex: "Our safest Agent"
   - Claude SDK: "Most powerful Hooks system"
   - OpenCode: "100% open source + strongest permission control"

2. **Scenario differentiation**: Different scenarios need different capability combinations
   - Research: Simple + interpretable
   - SaaS: Cost + observability
   - Enterprise: Security + compliance

3. **Technology stack binding**: Language choice limits capability implementation
   - Rust (Codex): Good for Sandbox, but not for rapid iteration
   - Python (OpenAI SDK): Good for AI ecosystem, but performance limited
   - TypeScript (OpenCode): Good for Web, but few ML libraries

### 5.2 Layer 3 (Protocol Support): More Likely to Converge

**MCP is becoming a de facto standard**:
- 8/9 frameworks support it (only SWE-agent doesn't)
- Anthropic pushes it, OpenAI also accepts
- Once tool ecosystem MCP-izes, frameworks without MCP support will be significantly less competitive

**Prediction**:
- Within 2 years, MCP Client may become a basic capability
- Within 3 years, MCP Server becomes differentiated competitiveness (similar to now)

### 5.3 Future Key Capability Predictions

**Tier 1: Becoming high priority soon (next 1-2 years)**
1. **Context Compaction**: Long-context LLMs are powerful, but cost-sensitive
2. **Structured Tracing**: Observability becomes high priority for production
3. **HITL State Management**: Human intervention is the safety baseline

**Tier 2: Enterprise high priority (next 2-3 years)**
4. **Smart Caching**: Tool Result Cache (not yet implemented)
5. **Multi-Modal Tool**: Unified handling of text/image/audio
6. **Semantic Memory**: Long-term memory (not simple history)

**Tier 3: Frontier exploration (long-term)**
7. **Self-Reflection**: Automatic improvement (like Reflexion, but production-grade)
8. **Multi-Agent Protocol**: Standardized inter-agent collaboration

---

## VI. Conclusions and Actionable Recommendations: A New Paradigm for Evaluating Agent Frameworks

### 6.0 Meta-Conclusion: The Essence of Agent Framework Competition

Based on the code audit of 9 frameworks, the core insight of this article is:

> **Agent framework competition is essentially engineering system competition, not reasoning algorithm competition.**

Between 2022-2026, the Agent field underwent a shift from "algorithm innovation-driven" to "systems engineering-driven":

- **Algorithm Innovation** (2022-2023): ReAct, Reflexion, Tree of Thoughts, and other new reasoning paradigms
- **Engineering Systems** (2024-2026): Session, Tracing, Sandbox, MCP, and other infrastructure capabilities

The landmark event of this shift was the release of **OpenAI Agents SDK** (2025): It didn't propose new reasoning algorithms, but productized Swarm's engineering practices, providing standardized Session, Tracing, and Guardrails capabilities.

**Implications for Academia and Industry**:
- If academia remains at "proposing new reasoning algorithms," it may underestimate the importance of engineering capabilities
- If industry ignores engineering system construction, even the best algorithms are difficult to deploy in production
- Future breakthroughs may come from **algorithm and engineering synergistic innovation** (e.g., Learned Context Compaction, Adaptive Tool Selection)

### 6.1 First Avoid These Three Low-Information Questions

❌ "Did you use Function Calling?" → Just parsing method, not architectural difference
❌ "Do you support MCP?" → 8/9 support in this survey, depth is what matters
❌ "How many Stars?" → Popularity ≠ suitable for your scenario

### 6.2 Then Use These Four Question Sets for Scenario-Based Selection

✅ **"Does the Layer 2 capability combination fit my scenario?"**
- Doing research? → Choose Stateless + Trajectory (SWE-agent)
- Doing SaaS? → Choose Session + Tracing (OpenAI SDK; can supplement compaction for long sessions)
- Doing enterprise? → Choose Sandbox + Policy (Codex)

✅ **"Do I need HITL?"**
- If involving dangerous operations (deleting data, transfers), prioritize frameworks with HITL state management (like OpenAI SDK)

✅ **"What security requirement level?"**
- Level 1-2: OpenAI SDK (Guardrails)
- Level 3: OpenCode/Gemini (Policy Engine)
- Level 4: Codex (Sandbox)

✅ **"Provider Lock-in tolerance?"**
- Zero tolerance → OpenAI SDK (Provider-agnostic)
- Only using Claude → Claude SDK

### 6.3 Recommendations for Academia (Actionable)

1. **Choose OpenAI Agents SDK as the baseline**:
   - Provider-agnostic (can compare GPT-4/Claude/Gemini)
   - Complete capabilities (Session, Tracing, Guardrails)
   - Production-ready (easy to migrate to industry)

2. **MCP-ize evaluation tools**:
   - Publish tools like `str_replace_editor` as MCP servers
   - Let all frameworks compare under same conditions
   - Improve paper comparability and impact

3. **Focus on unimplemented capabilities**:
   - Tool Result Cache (Caching)
   - Circuit Breaker (Fault Tolerance)
   - Semantic Memory (Long-term Memory)
   - These are valuable engineering/research contributions

### 6.4 Recommendations for Industry (Actionable)

1. **Don't reinvent the wheel**:
   - Core loops are all the same, use mature frameworks directly (OpenAI SDK, Codex)
   - Focus effort on business logic, not infrastructure

2. **Integrate proprietary tools through MCP**:
   - Don't fork framework code
   - Expose internal APIs via MCP Server

3. **Invest in observability**:
   - Connect Tracing from Day 1 (OpenAI SDK's Logfire/AgentOps)
   - You don't know what pitfalls you'll hit until monitoring tells you

---

## Related Work (Previously Published Related Articles)

> This section positions this article relative to existing publicly published articles: which views have been discussed, and what is the increment of this article.

### A. Framework Selection and Engineering Capability Comparison (High Relevance)

1. **Choosing the Right AI Framework** (Enhancial, 2025-10)  
    https://enhancial.substack.com/p/choosing-the-right-ai-framework-a  
    - Covers framework selection for OpenAI Agents SDK, Claude Agent SDK, LangGraph, MCP.  
    - Similar to this article: Emphasizes engineering capabilities (session, guardrails, tracing) are more important than "model fame."

2. **How to build AI agents with MCP: 12 framework comparison** (ClickHouse, 2025-10)  
    https://clickhouse.com/blog/how-to-build-ai-agents-mcp-12-frameworks  
    - Compares 12 frameworks horizontally with MCP as the main thread, providing runnable examples.  
    - Similar to this article: Discusses MCP depth differences and security/tool governance.

3. **OpenAI AgentKit vs Claude Agents SDK** (Bind AI, 2025-10)  
    https://blog.getbind.co/openai-agentkit-vs-claude-agents-sdk-which-is-better/  
    - Focuses on governance differences between platformization (AgentKit) and SDK-ization (Claude + MCP).  
    - Similar to this article: Differences lie in permissions, observability, control plane, not "who is smarter."

### B. CLI Agent Ecosystem Horizontal Review (Medium-High Relevance)

4. **EVERY CLI CODING AGENT, COMPARED** (Michael Livshits, 2026-02)  
    https://michaellivs.com/blog/cli-coding-agents-compared/  
    - Panoramic comparison of Codex/Gemini/Qwen/Kimi/OpenCode/SWE-agent, etc.  
    - Similar to this article: Values sandbox, hooks, memory files, MCP, and other engineering capabilities.

5. **The 2026 Guide to Coding CLI Tools: 15 AI Agents Compared** (Tembo, 2026-02)  
    https://www.tembo.io/blog/coding-cli-tools-comparison  
    - Systematic classification from "vendor lock-in, autonomy, cost, model flexibility" dimensions.  
    - Similar to this article: Emphasizes tool selection should consider constraints and scenarios, not single metrics.

6. **The Ultimate Comparison of Claude Code Alternatives** (Kevnu, 2025-11)  
    https://www.kevnu.com/en/posts/the-ultimate-comparison-of-claude-code-alternatives-a-complete-analysis-of-the-10-strongest-cli-ai-programming-tools  
    - Covers capability matrices for OpenCode, Qwen, Kimi, Codex, Gemini, and other CLI tools.  
    - Similar to this article: Provides multi-tool horizontal capability comparison.

### C. Official Methodology and Single Framework Deep Dives (Medium Relevance)

7. **Building agents with the Claude Agent SDK** (Anthropic, 2025-09)  
    https://claude.com/blog/building-agents-with-the-claude-agent-sdk  
    - Proposes gather → act → verify → repeat agent loop and compaction/subagent/MCP practices.  
    - Similar to this article: Supports the observation that "core loops are highly stable, differences lie in engineering implementation."

8. **I tested 5 AI CLI tools** (LogRocket, 2025-12)  
    https://blog.logrocket.com/tested-5-ai-cli-tools/  
    - Unified task-based comparison of multiple CLIs for usability and quality.  
    - Similar to this article: Emphasizes engineering usability and real workflow performance.

### This Article's Incremental Positioning

- Compared to the above articles, this article's main increment is:
  1) Organizing 9 projects with a unified Layer 1-4 framework;  
  2) Systematizing differences into engineering dimensions like session/security/policy/MCP/tracing/compaction;  
  3) Explicitly proposing the "capability migration cost" problem between academic implementation and industrial deployment.

---

## Appendix: Core Code Locations for the 9 Frameworks

| Framework | Core Loop File | Capability Files | Total Code Size |
|-----------|---------------|-----------------|-----------------|
| **OpenAI Agents SDK** | `src/agents/run.py:396-1329` | `src/agents/handoffs/`, `src/agents/tracing/`, `src/agents/memory/` | ~15k lines |
| **Claude Agent SDK** | `_internal/query.py` | `_internal/query.py:286-299` (Hooks) | SDK ~3k lines |
| **OpenCode** | `src/session/prompt.ts:274-724` | `src/permission/next.ts`, `src/session/compaction.ts` | ~47k lines |
| **Codex CLI** | `codex-rs/core/src/codex.rs` | `codex-rs/core/src/exec_policy.rs`, `codex-rs/core/src/seatbelt.rs` | Core ~10k lines |
| **Kimi CLI** | `src/kimi_cli/soul/kimisoul.py` | `src/kimi_cli/acp/`, `src/kimi_cli/session.py` | Not counted |
| **Gemini CLI** | `packages/core/src/core/client.ts` | `packages/core/src/policy/`, `packages/core/src/tools/mcp-client.ts` | ~866 files |
| **Qwen Code** | `packages/core/src/core/client.ts` | `packages/core/src/subagents/`, `packages/core/src/skills/` | Not counted |
| **SWE-agent** | `sweagent/agent/agents.py:1265` | `sweagent/tools/`, `sweagent/agent/` | Agent ~1.3k lines |
| **OpenManus** | `app/agent/react.py:11-38` | `app/mcp/`, `app/flow/` | Not counted |

---

## References

- **Deep Research Documents**: `agent_deep_dive/*-DEEP-DIVE.md` (~300 KB code audit)
- **GitHub Repositories**: See individual framework links
- **MCP Specification**: https://modelcontextprotocol.io/

---

## Author and Acknowledgments


**AI Assistance**: This research and blog writing process used [Opencode](https://opencode.ai) for code auditing, architecture analysis, and content writing. Opencode is an AI-powered software engineering assistant that helped complete the analysis and documentation generation of approximately 300KB of source code.

**Research Methodology**: All technical details are based on actual source code audits. Specific file paths and line numbers can be verified in the framework-specific deep research reports in the [deep-dive/](deep-dive/) directory.

**Feedback and Contributions**: If you find errors or wish to supplement other framework analyses, please submit an Issue or PR.

**Version**: 2026-02-27

**License**: [MIT](LICENSE)
