# SWE-agent - 深度技术调研

!!! info "项目概览"

    **调研日期**: 2026-02-26  
    **仓库**: https://github.com/SWE-agent/SWE-agent  
    **许可证**: MIT  
    **主要语言**: Python  
    **定位**: 面向"自动修 issue/基准评测"的研究型 Agent  

---

## 1. 项目概述

SWE-agent 是一个研究导向的自动化软件工程 agent 框架，核心特点：

- ✅ **YAML 驱动**: 完全配置化的 agent 行为
- ✅ **SWE-Bench 集成**: 自动评测 GitHub issue 解决能力
- ✅ **工具 Bundle 系统**: 可组合的 shell 工具集
- ✅ **Docker/Modal 部署**: 隔离执行环境
- ✅ **Trajectory 分析**: 完整的执行轨迹记录

---

## 2. 代码架构

### 目录结构

```
sweagent/
├── agent/           # 核心 agent 逻辑
│   ├── agents.py         # DefaultAgent, RetryAgent (1294 lines)
│   ├── models.py         # LLM 集成 (903 lines)
│   ├── problem_statement.py  # 问题定义
│   ├── history_processors.py # 消息历史操作
│   └── hooks/            # Agent 生命周期 hooks
├── environment/     # 执行环境管理
│   ├── swe_env.py        # SWEEnv 类 (276 lines)
│   ├── repo.py           # Repository 处理
│   └── hooks/            # 环境 hooks
├── tools/           # 工具/action 系统
│   ├── tools.py          # ToolConfig & ToolHandler (430 lines)
│   ├── commands.py       # Command 定义
│   ├── parsing.py        # 模型输出解析器
│   └── bundle.py         # 工具 bundle 加载器
├── run/             # CLI 和执行编排
│   ├── run.py            # 主 CLI 入口
│   ├── run_single.py     # 单实例执行
│   ├── run_batch.py      # 批量执行 (SWE-Bench)
│   └── hooks/            # Run hooks
└── utils/           # 工具函数

tools/               # 工具 bundles（安装在容器中）
├── edit_anthropic/  # 文件编辑 (str_replace_editor)
├── windowed/        # 窗口化文件查看/编辑
├── search/          # 文件/目录搜索
├── submit/          # 解决方案提交
└── [15+ more bundles]

config/              # YAML 配置文件
├── default_backticks.yaml
├── demo/
└── sweagent_0_7/
```

---

## 3. 底层实现原理

### 3.1 Agent Loop (DefaultAgent)

**文件**: `sweagent/agent/agents.py:443-1294`

**主循环** (`run()` 方法at line 1265):
```python
def run(self, env: SWEEnv, problem_statement: ProblemStatement, 
        output_dir: Path) -> AgentRunResult:
    self.setup(env, problem_statement, output_dir)
    self._chook.on_run_start()
    step_output = StepOutput()
    while not step_output.done:
        step_output = self.step()
        self.save_trajectory()
    self._chook.on_run_done(trajectory=self.trajectory, info=self.info)
    return AgentRunResult(info=self.info, trajectory=self.trajectory)
```

**步骤执行** (`step()` at line 1235):
1. 调用 `forward_with_handling(self.messages)` 获取模型输出
2. 添加步骤到历史和轨迹
3. 更新 info 字典
4. 触发 hooks

**Forward with handling** (line 1062):
- 调用 `forward()` which:
  1. 查询模型with history
  2. 解析输出提取 thought + action
  3. 通过 `handle_action()` 执行 action
- 使用重查询处理错误（最多 `max_requeries` 次）:
  - `FormatError`: 无效命令格式
  - `_BlockedActionError`: 黑名单命令
  - `BashIncorrectSyntaxError`: Bash 语法错误

**Action 处理** (`handle_action()` at line 936):
```python
def handle_action(self, step: StepOutput) -> StepOutput:
    if self.tools.should_block_action(step.action):
        raise _BlockedActionError()
    if step.action.strip() == "exit":
        step.done = True
        return step
    # 通过环境执行
    step.observation = self._env.communicate(
        input=run_action, 
        timeout=self.tools.config.execution_timeout
    )
    step.state = self.tools.get_state(env=self._env)
    return self.handle_submission(step)
```

### 3.2 YAML 驱动配置

**结构** (`config/default_backticks.yaml`):
```yaml
agent:
  templates:           # Jinja2 templates
    system_template: "..."
    instance_template: "..."
    next_step_template: "..."
  tools:
    bundles:          # 工具 bundles 加载
      - path: tools/registry
      - path: tools/edit_anthropic
    parse_function:    # 如何解析模型输出
      type: function_calling
  history_processors:  # 消息操作
    - type: cache_control
      last_n_messages: 2
  model:
    name: claude-sonnet-4-20250514
env:
  repo:               # Repository 配置
    github_url: https://github.com/...
  deployment:         # Docker/Modal/Local
    image: python:3.11
problem_statement:    # Issue/task 定义
  github_url: https://github.com/.../issues/1
```

### 3.3 工具 Bundle 系统

**工具 bundles** 是 `tools/` 中的目录，包含:
- `config.yaml`: 工具定义
- `bin/`: 可执行脚本
- `lib/`: Python 库（可选）
- `install.sh`: 设置脚本（可选）

**示例** (`tools/edit_anthropic/config.yaml:1-56`):
```yaml
tools:
  str_replace_editor:
    signature: "str_replace_editor <command> <path> [<file_text>]..."
    docstring: "Custom editing tool for viewing, creating and editing files"
    arguments:
      - name: command
        type: string
        enum: ["view", "create", "str_replace", "insert", "undo_edit"]
        required: true
      - name: path
        type: string
        required: true
state_command: "_state_anthropic"
```

**工具安装** (`sweagent/tools/tools.py:292`):
1. 上传 bundle 到 `/root/tools/<bundle_name>`
2. 添加 bin 目录到 PATH
3. 运行 install.sh（如果存在）
4. 验证命令可用

### 3.4 环境/运行时设置

**初始化** (`sweagent/environment/swe_env.py:176`):
1. 启动部署（Docker/Modal via SWE-ReX）
2. 使用启动命令创建 bash session
3. 设置环境变量
4. 复制 repository
5. 重置到 base commit
6. 运行启动后命令

**通信** (`communicate()` at line 197):
- 通过 SWE-ReX 向 bash session 发送命令
- 处理超时和退出码
- 返回 stdout/stderr

---

## 4. 扩展开发指南

### 4.1 创建自定义 Agents

```python
class MyCustomAgent(DefaultAgent):
    def forward(self, history: list[dict]) -> StepOutput:
        step = super().forward(history)
        # 后处理
        return step
```

### 4.2 定义自定义 Actions/Tools

**创建工具 bundle** in `tools/my_tool/`:

1. **config.yaml**:
```yaml
tools:
  my_command:
    signature: "my_command <arg1> [<arg2>]"
    docstring: "Does something useful"
    arguments:
      - name: arg1
        type: string
        required: true
```

2. **bin/my_command** (executable):
```bash
#!/usr/bin/env bash
arg1=$1
echo "Result: $arg1"
```

3. **使用**:
```yaml
agent:
  tools:
    bundles:
      - path: tools/my_tool
```

### 4.3 配置环境

```yaml
env:
  deployment:
    type: docker
    image: python:3.11
  repo:
    type: github
    github_url: https://github.com/owner/repo
    base_commit: main
```

### 4.4 创建自定义 Benchmarks

```yaml
instances:
  type: file
  path: path/to/instances.jsonl
```

**instances.jsonl**:
```json
{"instance_id": "id1", "problem_statement": "Fix bug X", "repo": "..."}
```

---

## 5. 评估

!!! success "核心优势"

    1. ✅ **研究导向**: 强大的评测框架
    2. ✅ **可重现**: YAML 配置确保一致性
    3. ✅ **SWE-Bench 集成**: 自动 benchmark
    4. ✅ **Trajectory 分析**: 完整执行记录

!!! warning "局限性"

    1. ⚠️ **研究框架**: 非产品化
    2. ❌ **无生产特性**: 缺少 session, tracing
    3. ⚠️ **学习曲线**: 需理解 YAML 配置

!!! tip "推荐场景"

    - ✅ SWE-bench 评测
    - ✅ 自动修复研究
    - ✅ CTF/安全任务研究

!!! danger "不推荐场景"

    - ❌ 日常编码助手
    - ❌ 生产应用

---

**文档版本**: 1.0  
**最后更新**: 2026-02-26
