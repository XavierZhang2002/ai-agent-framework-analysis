# OpenHands - 深度技术调研

**调研日期**: 2026-02-27  
**仓库**: https://github.com/All-Hands-AI/OpenHands  
**许可证**: MIT  
**版本**: v0.25.0  
**Stars**: 67.5k | **Forks**: 7.2k | **Contributors**: 498  
**融资**: $18.8M (AI Grant, Factorial Capital)

---

## 目录

1. [项目概述](#1-项目概述)
2. [平台化架构深度分析](#2-平台化架构深度分析)
3. [Docker 沙箱实现](#3-docker-沙箱实现)
4. [与竞品对比](#4-与竞品对比)
5. [评估与建议](#5-评估与建议)

---

## 1. 项目概述

### 1.1 定位与背景

OpenHands（原 OpenDevin）是 **最完整的开源 Agent 平台**，不仅是一个 CLI 工具，而是一套完整的软件工程 Agent 基础设施：

**核心定位**:
- 🚀 **Full Platform**: 不只是 CLI，包含 Web UI、API、微代理生态
- 🚀 **Docker Sandboxing**: 安全的代码执行环境
- 🚀 **Micro-Agents**: 可组合的专用 Agent
- 🚀 **Evaluation Ready**: 内置 SWE-bench 评估支持

**关键数据**:
- 67.5K GitHub Stars（开源 Agent 平台第一）
- $18.8M 融资，全职团队维护
- 支持多种 LLM（OpenAI、Anthropic、本地模型等）

### 1.2 架构哲学

```
OpenHands 设计哲学
├─ Platform First: 提供完整平台，而非单一工具
├─ Safety by Default: Docker 沙箱是核心，非可选
├─ Composable Agents: 微代理架构，可组合解决复杂问题
└─ Research Ready: 内置评估基准，学术友好
```

---

## 2. 平台化架构深度分析

### 2.1 整体架构

```
OpenHands 平台架构
┌─────────────────────────────────────────────────────────┐
│                    Web UI / CLI                         │
├─────────────────────────────────────────────────────────┤
│                 OpenHands Server                        │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │   Session   │  │   Agent     │  │  Evaluation    │  │
│  │   Manager   │  │   Runtime   │  │   Engine       │  │
│  └─────────────┘  └─────────────┘  └────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                 Docker Sandbox                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Isolated Container (Ubuntu + Dev Tools)        │   │
│  │  ├─ Runtime (Python/Node.js/Java/...)           │   │
│  │  ├─ Shell (bash/zsh)                            │   │
│  │  ├─ Browser (Chromium for web tasks)            │   │
│  │  └─ Git                                         │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
openhands/
├── openhands/
│   ├── server/                   # 服务器核心 ★★★
│   │   ├── session/
│   │   │   ├── manager.py        # 会话管理 (800+ lines)
│   │   │   └── session.py        # 会话状态
│   │   ├── agent/
│   │   │   ├── agent.py          # Agent 运行 (1,200+ lines) ★★★★★
│   │   │   └── runtime.py        # 运行时管理
│   │   └── listen.py             # 主服务入口
│   │
│   ├── controller/               # Agent 控制器 ★★★
│   │   ├── agent_controller.py   # 主控制器 (900+ lines)
│   │   └── action_execution.py   # 动作执行
│   │
│   ├── runtime/                  # 运行时实现 ★★★★★
│   │   ├── runtime.py            # Runtime 接口
│   │   ├── docker/               # Docker 沙箱
│   │   │   ├── exec_box.py       # Docker 执行 (600+ lines)
│   │   │   └── image.py          # 镜像管理
│   │   └── browser/              # 浏览器运行时
│   │
│   ├── agenthub/                 # 微代理库 ★★★
│   │   ├── codeact_agent/        # CodeAct Agent (默认)
│   │   ├── browsing_agent/       # 浏览器 Agent
│   │   └── delegator_agent/      # 委派 Agent
│   │
│   ├── evaluation/               # 评估框架 ★★★
│   │   ├── swe_bench/            # SWE-bench 支持
│   │   └── metrics.py            # 评估指标
│   │
│   └── core/                     # 核心抽象
│       ├── schema/               # 数据模型
│       └── exceptions.py         # 异常定义
│
├── frontend/                     # React Web UI
├── evaluation/                   # 评估脚本
└── microagents/                  # 微代理定义
```

---

## 3. Docker 沙箱实现

### 3.1 为什么 Docker 沙箱很重要？

**安全执行代码的三大方案**:

| 方案 | 安全性 | 易用性 | 性能 | 代表 |
|------|-------|-------|------|------|
| **无沙箱** | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | OpenManus |
| **OS-Level** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Claude Code, Codex |
| **Docker** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | OpenHands, Goose |

**OpenHands 选择 Docker 的原因**:
- 最高级别的进程隔离
- 可复制的环境（容器镜像）
- 资源限制（CPU、内存、网络）
- 预装开发工具链

### 3.2 Docker Runtime 实现

**核心文件**: `openhands/runtime/docker/exec_box.py: 600+ lines`

```python
class DockerExecBox:
    """Docker 沙箱执行环境"""
    
    def __init__(self, image_name, container_name):
        self.client = docker.from_env()
        self.image_name = image_name
        self.container_name = container_name
        self.container = None
    
    def start(self):
        """启动沙箱容器"""
        # 1. 拉取或构建镜像
        self._ensure_image()
        
        # 2. 启动容器（带资源限制）
        self.container = self.client.containers.run(
            self.image_name,
            name=self.container_name,
            detach=True,
            # 资源限制
            mem_limit='4g',
            cpu_quota=100000,  # 1 CPU
            # 网络隔离
            network='bridge',
            # 文件系统隔离
            volumes={
                '/workspace': {
                    'bind': '/workspace',
                    'mode': 'rw'
                }
            },
            # 安全选项
            security_opt=['no-new-privileges:true'],
            cap_drop=['ALL'],
            cap_add=['CHOWN', 'SETGID', 'SETUID'],
        )
    
    def execute(self, cmd, timeout=120):
        """在沙箱中执行命令"""
        # 1. 通过 docker exec 执行
        result = self.container.exec_run(
            cmd,
            workdir='/workspace',
            demux=True,  # 分离 stdout/stderr
        )
        
        # 2. 检查超时
        if result.exit_code == -1:
            raise TimeoutError(f"Command timed out after {timeout}s")
        
        return result.output
    
    def execute_action(self, action: Action) -> Observation:
        """执行 Agent 动作"""
        if isinstance(action, CmdRunAction):
            # 执行 shell 命令
            return self.execute(action.command)
        
        elif isinstance(action, FileReadAction):
            # 读取文件
            content = self._read_file(action.path)
            return FileReadObservation(content=content)
        
        elif isinstance(action, FileWriteAction):
            # 写入文件
            self._write_file(action.path, action.content)
            return FileWriteObservation()
        
        # ... 更多动作类型
```

### 3.3 浏览器运行时

**Web 任务支持**:
```python
class BrowserRuntime:
    """支持浏览器任务的运行时"""
    
    def __init__(self):
        self.browser = None
        self.page = None
    
    async def browse(self, url: str) -> str:
        """浏览网页并返回内容"""
        if not self.browser:
            self.browser = await chromium.launch()
            self.page = await self.browser.new_page()
        
        await self.page.goto(url)
        content = await self.page.content()
        
        # 提取文本内容
        text = extract_text(content)
        return text
    
    async def click(self, selector: str):
        """点击页面元素"""
        await self.page.click(selector)
    
    async def type(self, selector: str, text: str):
        """在输入框中输入文本"""
        await self.page.fill(selector, text)
```

**使用场景**:
- 抓取网页文档
- 填写表单
- 执行端到端测试

---

## 4. 微代理架构

### 4.1 什么是 Micro-Agents？

**传统单体 Agent**:
```
User Request → [Big Agent] → Result
              (万能，但复杂)
```

**OpenHands 微代理**:
```
User Request → [Delegator] → 任务分解
                    ↓
            ┌──────┼──────┐
            ↓      ↓      ↓
        [Coder] [Browser] [Verifier]
        (编码)   (搜索)    (验证)
            └──────┼──────┘
                   ↓
               [结果整合]
```

### 4.2 微代理实现

**CodeAct Agent** (默认):
```python
class CodeActAgent:
    """通过代码行动的 Agent"""
    
    system_prompt = """
    You are an autonomous coding agent.
    You can execute bash commands and edit files.
    Always explain your plan before taking action.
    """
    
    def step(self, state: State) -> Action:
        # 1. 构建上下文
        context = self._build_context(state)
        
        # 2. 调用 LLM
        response = self.llm.completion(context)
        
        # 3. 解析响应，提取动作
        action = self._parse_response(response)
        
        return action
```

**Browsing Agent**:
```python
class BrowsingAgent:
    """专门处理网页任务的 Agent"""
    
    def step(self, state: State) -> Action:
        # 使用浏览器运行时
        if "search" in state.user_request:
            return BrowseAction(url=f"https://google.com/search?q={query}")
        
        elif "click" in state.user_request:
            return ClickAction(selector=extract_selector(state))
```

**Delegator Agent**:
```python
class DelegatorAgent:
    """委派 Agent，负责任务分解和调度"""
    
    def step(self, state: State) -> Action:
        # 1. 分析任务复杂度
        complexity = self._analyze_complexity(state)
        
        if complexity == "simple":
            # 自己处理
            return self._handle_simple(state)
        else:
            # 委派给子 Agent
            sub_agent = self._select_sub_agent(state)
            return DelegateAction(agent=sub_agent, task=state.task)
```

---

## 5. 与竞品对比

### 5.1 与 Claude Code 对比

| 特性 | OpenHands | Claude Code |
|------|-----------|-------------|
| 架构 | ⭐⭐⭐⭐⭐ 完整平台 | ⭐⭐⭐⭐ 深度 CLI |
| 沙箱 | ⭐⭐⭐⭐⭐ Docker | ⭐⭐⭐⭐⭐ OS-Level |
| Web UI | ⭐⭐⭐⭐⭐ 完善 | ❌ 无 |
| Sub-agents | ⭐⭐⭐⭐ 微代理 | ⭐⭐⭐⭐⭐ 内置 |
| IDE 集成 | ⭐⭐⭐ 有限 | ⭐⭐⭐⭐⭐ 深度 |
| 评估支持 | ⭐⭐⭐⭐⭐ 内置 SWE-bench | ⭐⭐⭐ 有限 |

**结论**:
- **OpenHands 优势**: 平台化、Docker 沙箱、Web UI、评估框架
- **Claude Code 优势**: 更深的功能集、IDE 集成、Seatbelt 沙箱

### 5.2 与 Goose 对比

| 特性 | OpenHands | Goose |
|------|-----------|-------|
| 沙箱 | ⭐⭐⭐⭐⭐ Docker | ⭐⭐⭐⭐ Docker |
| MCP | ⭐⭐⭐ 支持 | ⭐⭐⭐⭐⭐ 原生 |
| UI | ⭐⭐⭐⭐⭐ Web + CLI | ⭐⭐⭐ CLI 简单 |
| Stars | 67.5K | 29.9K |

**结论**: OpenHands 更适合需要完整平台的场景，Goose 更适合 MCP 生态。

---

## 6. 评估与建议

### 6.1 适用场景

**强烈推荐使用 OpenHands 的场景**:
1. ✅ **完整平台需求**: 需要 Web UI + API + CLI
2. ✅ **安全优先**: 必须 Docker 沙箱隔离
3. ✅ **学术研究**: 需要 SWE-bench 评估
4. ✅ **团队协作**: 需要共享 Agent 环境
5. ✅ **微代理**: 需要可组合的 Agent 系统

**不建议使用 OpenHands 的场景**:
1. ❌ **快速 CLI**: 只想简单命令行使用
2. ❌ **资源受限**: 无法运行 Docker
3. ❌ **MCP 生态**: 重度依赖 MCP 服务器

### 6.2 学术价值

**研究贡献**:
- 首次将 SWE-bench 集成到开源 Agent 平台
- 微代理架构的工程实践
- Docker 沙箱在 Agent 中的应用

### 6.3 工程价值

**架构借鉴**:
- 平台化设计模式
- Docker 沙箱集成
- 微代理调度机制

---

## 附录：核心代码位置

| 模块 | 文件 | 代码行数 | 关键功能 |
|------|------|---------|---------|
| 服务器 | `openhands/server/session/manager.py` | ~800 | 会话生命周期管理 |
| Agent 控制器 | `openhands/controller/agent_controller.py` | ~900 | 主控制逻辑 |
| Docker 沙箱 | `openhands/runtime/docker/exec_box.py` | ~600 | 容器执行环境 |
| 评估框架 | `openhands/evaluation/` | ~1,500 | SWE-bench 支持 |
| 微代理 | `openhands/agenthub/` | ~2,000 | 多 Agent 实现 |

---

**参考**: 
- [OpenHands GitHub](https://github.com/All-Hands-AI/OpenHands)
- [OpenHands Documentation](https://docs.all-hands.dev/)
