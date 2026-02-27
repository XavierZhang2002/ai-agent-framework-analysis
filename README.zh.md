# AI Agent 框架分析

> 12 个开源 Agent 框架的深度代码审计

[English](README.md) | **中文**

---

## 📖 快速导航

- **[阅读博客（中文）](BLOG.zh.md)** - 审计12个Agent框架的核心洞察
- **[Read the Blog (English)](BLOG.en.md)** - Core insights from auditing 12 agent frameworks
- **[框架对比](COMPARISON.md)** - 详细的特性对比矩阵
- **[深度调研报告](deep-dive/)** - 每个框架的源代码分析

## 🎯 核心观点

在审计了 12 个主流 Agent 框架约 400KB 源代码后，我们发现了一个反直觉的事实：

> **各框架的核心 Agent Loop 逻辑高度相似。真正的差异在于"特性分化"**——状态管理、安全控制、协议支持等 30+ 个工程特性的不同实现。

这种"特性分化"导致学术界难以使用最先进的 Agent 能力，形成了一道隐形的鸿沟。

### 关键洞察

1. **Layer 1（核心循环）**：无分化——所有框架使用相同模式
2. **Layer 2（工程特性）**：真正的竞争发生在这里
3. **Layer 3（协议支持）**：MCP 正在成为标准
4. **Layer 4（交互方式）**：只是表层

### 分析的框架

| 框架 | 语言 | 核心定位 | 关键差异化特性 |
|------|------|---------|--------------|
| [OpenAI Agents SDK](deep-dive/OpenAI-Agents-SDK-DEEP-DIVE.md) | Python | 生产级 SDK | HITL、Tracing、Provider-agnostic |
| [Claude Agent SDK](deep-dive/Claude-Agent-SDK-Python-DEEP-DIVE.md) | Python | Claude 深度集成 | 10+ Hooks、SDK MCP |
| [Codex CLI](deep-dive/Codex-CLI-DEEP-DIVE.md) | Rust | 企业级安全 | OS-level Sandbox |
| [OpenCode](deep-dive/OpenCode-DEEP-DIVE.md) | TypeScript | 开源 CLI | Permission Ruleset |
| [Kimi CLI](deep-dive/Kimi-CLI-DEEP-DIVE.md) | Python/TS | IDE 集成 | ACP Server |
| [Gemini CLI](deep-dive/Gemini-CLI-DEEP-DIVE.md) | TypeScript | Google 生态 | 30+ 内置工具 |
| [Qwen Code](deep-dive/Qwen-Code-DEEP-DIVE.md) | TypeScript | Qwen 生态 | Skills + SubAgents |
| [SWE-agent](deep-dive/SWE-agent-DEEP-DIVE.md) | Python | 研究框架 | Trajectory Recording |
| [OpenManus](deep-dive/OpenManus-DEEP-DIVE.md) | Python | 快速实验 | MCP 双角色 |
| [Aider](deep-dive/Aider-DEEP-DIVE.md) | Python | Git-Native 编码助手 | Repo Map、Tree-sitter |
| [Goose](deep-dive/Goose-DEEP-DIVE.md) | Rust | MCP-Native 框架 | MCP-Native 架构 |
| [OpenHands](deep-dive/OpenHands-DEEP-DIVE.md) | Python | 完整平台 | Docker 沙箱、微代理 |

## 📊 研究方法

- **源代码审计**：分析了约 400KB 实际源代码
- **行级分析**：引用具体文件路径和行号
- **无幻觉**：所有技术主张都有代码证据支持
- **双重视角**：学术研究 + 工业生产两个视角

## 🔗 相关资源

- [MCP 协议](https://modelcontextprotocol.io/) - Model Context Protocol 规范
- [Awesome AI Agents](https://github.com/e2b-dev/awesome-ai-agents) - AI Agent 精选列表

## 🤖 致谢

本研究和博客由 [Opencode](https://opencode.ai) 协助完成，Opencode 是一个 AI 驱动的软件工程助手。

## 📄 许可证

MIT 许可证 - 详见 [LICENSE](LICENSE)

---

**最后更新**：2026-02-27
