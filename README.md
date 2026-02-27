# AI Agent Framework Analysis

> Deep Code Audit of 9 Open-Source Agent Frameworks

**English** | [中文](README.zh.md)

---

## 📖 Quick Navigation

- **[Read the Blog (English)](BLOG.en.md)** - Core insights from auditing 9 agent frameworks
- **[Read the Blog (中文)](BLOG.zh.md)** - 审计9个Agent框架的核心洞察
- **[Framework Comparison](COMPARISON.md)** - Detailed feature comparison matrix
- **[Deep Dive Reports](deep-dive/)** - Source code analysis for each framework

## 🎯 TL;DR

After auditing ~300KB of source code from 9 mainstream agent frameworks, we discovered a counter-intuitive fact:

> **The core Agent Loop logic is highly similar across all frameworks. The real differences lie in "Capability Differentiation"** — 30+ engineering features like state management, security controls, and protocol support.

This "capability differentiation" creates an invisible gap between academia and industry.

### Key Insights

1. **Layer 1 (Core Loop)**: No differentiation - all frameworks use the same pattern
2. **Layer 2 (Engineering Capabilities)**: Where the real competition happens
3. **Layer 3 (Protocol Support)**: MCP becoming the standard
4. **Layer 4 (Interaction Mode)**: Just the surface layer

### Frameworks Analyzed

| Framework | Language | Core Positioning | Key Differentiator |
|-----------|----------|-----------------|-------------------|
| [OpenAI Agents SDK](deep-dive/OpenAI-Agents-SDK-DEEP-DIVE.md) | Python | Production SDK | HITL, Tracing, Provider-agnostic |
| [Claude Agent SDK](deep-dive/Claude-Agent-SDK-Python-DEEP-DIVE.md) | Python | Claude Integration | 10+ Hooks, SDK MCP |
| [Codex CLI](deep-dive/Codex-CLI-DEEP-DIVE.md) | Rust | Enterprise Security | OS-level Sandbox |
| [OpenCode](deep-dive/OpenCode-DEEP-DIVE.md) | TypeScript | Open Source CLI | Permission Ruleset |
| [Kimi CLI](deep-dive/Kimi-CLI-DEEP-DIVE.md) | Python/TS | IDE Integration | ACP Server |
| [Gemini CLI](deep-dive/Gemini-CLI-DEEP-DIVE.md) | TypeScript | Google Ecosystem | 30+ Built-in Tools |
| [Qwen Code](deep-dive/Qwen-Code-DEEP-DIVE.md) | TypeScript | Qwen Ecosystem | Skills + SubAgents |
| [SWE-agent](deep-dive/SWE-agent-DEEP-DIVE.md) | Python | Research | Trajectory Recording |
| [OpenManus](deep-dive/OpenManus-DEEP-DIVE.md) | Python | Quick Experiment | MCP Dual Role |

## 📊 Research Methodology

- **Source Code Audit**: ~300KB of actual source code analyzed
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

## 📝 Citation

If you use this research in your work, please cite:

```bibtex
@misc{agent-framework-analysis-2026,
  title={AI Agent Framework Capability Differentiation: A Deep Code Audit},
  author={Research Team},
  year={2026},
  howpublished={\url{https://github.com/yourusername/ai-agent-framework-analysis}}
}
```

---

**Last Updated**: 2026-02-27
