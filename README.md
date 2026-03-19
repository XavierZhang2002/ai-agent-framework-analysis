# AI Agent Framework Analysis

> Deep Code Audit of 12 Open-Source Agent Frameworks

**English** | [中文](README.zh.md)

[![Website](https://img.shields.io/badge/Website-Live-blue?style=flat-square&logo=github)](https://XavierZhang2002.github.io/ai-agent-framework-analysis)

---

## 📖 Quick Navigation

- **[Website](https://XavierZhang2002.github.io/ai-agent-framework-analysis)** - Full knowledge base with navigation and search
- **[Read the Blog (English)](docs/BLOG.en.md)** - Core insights from auditing 12 agent frameworks
- **[Read the Blog (中文)](docs/BLOG.zh.md)** - 审计12个Agent框架的核心洞察
- **[Framework Comparison](docs/COMPARISON.md)** - Detailed feature comparison matrix
- **[Deep Dive Reports](docs/deep-dive/)** - Source code analysis for each framework

## 🎯 TL;DR

After auditing ~400KB of source code from 12 mainstream agent frameworks, we discovered a counter-intuitive fact:

> **The core Agent Loop logic is highly similar across all frameworks. The real differences lie in "Capability Differentiation"** — 35+ engineering features like state management, security controls, MCP support, and context compaction.

This "capability differentiation" creates an invisible gap between academia and industry.

### Key Insights

1. **Layer 1 (Core Loop)**: No differentiation - all frameworks use the same pattern
2. **Layer 2 (Engineering Capabilities)**: Where the real competition happens - 35+ features
3. **Layer 3 (Protocol Support)**: MCP becoming the standard (11/12 frameworks)
4. **Layer 4 (Interaction Mode)**: Just the surface layer

### Frameworks Analyzed

| Framework | Language | Core Positioning | Key Differentiator | Stars |
|-----------|----------|-----------------|-------------------|-------|
| [OpenAI Agents SDK](docs/deep-dive/OpenAI-Agents-SDK-DEEP-DIVE.md) | Python | Production SDK | HITL, Tracing, Provider-agnostic | 19.2K |
| [Claude Agent SDK](docs/deep-dive/Claude-Agent-SDK-Python-DEEP-DIVE.md) | Python | Claude Integration | 10+ Hooks, SDK MCP | 5K |
| [Codex CLI](docs/deep-dive/Codex-CLI-DEEP-DIVE.md) | Rust | Enterprise Security | 3-Layer Loop, OS Sandbox | 62.2K |
| [OpenCode](docs/deep-dive/OpenCode-DEEP-DIVE.md) | TypeScript | Open Source CLI | Smart Compaction, Permissions | 112K |
| [Kimi CLI](docs/deep-dive/Kimi-CLI-DEEP-DIVE.md) | Python/TS | IDE Integration | D-Mail Time Travel, ACP | 5.9K |
| [Gemini CLI](docs/deep-dive/Gemini-CLI-DEEP-DIVE.md) | TypeScript | Google Ecosystem | GEMINI.md Memory, Free Tier | 95.9K |
| [Qwen Code](docs/deep-dive/Qwen-Code-DEEP-DIVE.md) | TypeScript | Qwen Ecosystem | SubAgents, Skills, Templating | 18.1K |
| [SWE-agent](docs/deep-dive/SWE-agent-DEEP-DIVE.md) | Python | Research | Trajectory Recording, SWE-bench | 18.4K |
| [OpenManus](docs/deep-dive/OpenManus-DEEP-DIVE.md) | Python | Quick Experiment | MCP Dual Role, ReAct | - |
| [Aider](docs/deep-dive/Aider-DEEP-DIVE.md) | Python | Git-Native Coding | Repo Map, Tree-sitter | 41K |
| [Goose](docs/deep-dive/Goose-DEEP-DIVE.md) | Rust | MCP-Native Framework | MCP-Native, Extensions | 31.4K |
| [OpenHands](docs/deep-dive/OpenHands-DEEP-DIVE.md) | Python | Full Platform | Docker Sandbox, Micro-Agents | 68.3K |

### Detailed Capability Matrix

| Capability | OpenAI SDK | Claude SDK | Codex | OpenCode | Kimi | Gemini | Qwen | SWE | Manus | Aider | Goose | Hands |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **State Management** |||||||||||||
| Session Persistence | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ❌ | ✅ | ✅ | ✅ |
| HITL State | ✅ | ⚠️ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Context Compaction | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Security** |||||||||||||
| OS Sandbox | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ⚠️ | ✅ |
| Docker Sandbox | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Policy Engine | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ⚠️ | ⚠️ |
| **Protocol** |||||||||||||
| MCP Client | ⚠️ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ |
| MCP Server | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ |
| **Advanced** |||||||||||||
| Parallel Tools | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ⚠️ | ⚠️ | ✅ |
| Tracing | ✅ | ❌ | ⚠️ | ⚠️ | ❌ | ⚠️ | ❌ | ❌ | ❌ | ❌ | ⚠️ | ⚠️ |
| SubAgents | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ⚠️ | ❌ | ⚠️ | ✅ |

*Legend: ✅ Full support | ⚠️ Partial support | ❌ Not supported*

## 📊 Research Methodology

- **Source Code Audit**: ~400KB of actual source code analyzed
- **Line-Level Analysis**: Specific file paths and line numbers referenced
- **No Hallucination**: All technical claims backed by code evidence
- **Dual Perspective**: Both academic research and industrial production viewpoints

## 🔗 Related Resources

- [MCP Protocol](https://modelcontextprotocol.io/) - Model Context Protocol specification
- [Awesome AI Agents](https://github.com/e2b-dev/awesome-ai-agents) - Curated list of AI agents

## 🤖 Acknowledgment

This research and blog were assisted by [Opencode](https://opencode.ai), an AI-powered software engineering assistant.

## 📄 License

MIT License - See [LICENSE](LICENSE) for details.

---

**Last Updated**: 2026-02-27
