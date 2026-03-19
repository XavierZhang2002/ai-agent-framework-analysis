# AI Agent Framework Analysis

> Deep Code Audit of 12 Open-Source Agent Frameworks

After auditing ~400KB of source code from 12 mainstream agent frameworks, one counter-intuitive fact emerges:

> **The core Agent Loop logic is highly similar across all frameworks. The real differences lie in "Capability Differentiation"** — 35+ engineering features like state management, security controls, MCP support, and context compaction.

## Quick Navigation

<div class="grid cards" markdown>

-   :material-post-outline: **Blog**

    ---

    Core insights from the audit — academic vs. industrial capability gap explained

    [:octicons-arrow-right-24: English](BLOG.en.md) · [:octicons-arrow-right-24: 中文](BLOG.zh.md)

-   :material-table-large: **Comparison Matrix**

    ---

    Detailed feature comparison of all 12 frameworks across 35+ engineering capabilities

    [:octicons-arrow-right-24: View Matrix](COMPARISON.md)

-   :material-magnify: **Deep Dive Reports**

    ---

    Source-code-level analysis for each framework, with file paths and line numbers

    [:octicons-arrow-right-24: Browse Reports](deep-dive/OpenCode-DEEP-DIVE.md)

</div>

## Key Insights

1. **Layer 1 (Core Loop)**: No differentiation — all 12 frameworks use the same ReAct-style pattern
2. **Layer 2 (Engineering Capabilities)**: Where real competition happens — 35+ features
3. **Layer 3 (Protocol Support)**: MCP becoming the standard (11/12 frameworks)
4. **Layer 4 (Interaction Mode)**: Just the surface layer

## Frameworks Analyzed

| Framework | Language | Core Positioning | Stars |
|-----------|----------|-----------------|-------|
| [OpenAI Agents SDK](deep-dive/OpenAI-Agents-SDK-DEEP-DIVE.md) | Python | Production SDK | 19.2K |
| [Claude Agent SDK](deep-dive/Claude-Agent-SDK-Python-DEEP-DIVE.md) | Python | Claude Integration | 5K |
| [Codex CLI](deep-dive/Codex-CLI-DEEP-DIVE.md) | Rust | Enterprise Security | 62.2K |
| [OpenCode](deep-dive/OpenCode-DEEP-DIVE.md) | TypeScript | Open Source CLI | 112K |
| [Kimi CLI](deep-dive/Kimi-CLI-DEEP-DIVE.md) | Python/TS | IDE Integration | 5.9K |
| [Gemini CLI](deep-dive/Gemini-CLI-DEEP-DIVE.md) | TypeScript | Google Ecosystem | 95.9K |
| [Qwen Code](deep-dive/Qwen-Code-DEEP-DIVE.md) | TypeScript | Qwen Ecosystem | 18.1K |
| [SWE-agent](deep-dive/SWE-agent-DEEP-DIVE.md) | Python | Research | 18.4K |
| [OpenManus](deep-dive/OpenManus-DEEP-DIVE.md) | Python | Quick Experiment | — |
| [Aider](deep-dive/Aider-DEEP-DIVE.md) | Python | Git-Native Coding | 41K |
| [Goose](deep-dive/Goose-DEEP-DIVE.md) | Rust | MCP-Native Framework | 31.4K |
| [OpenHands](deep-dive/OpenHands-DEEP-DIVE.md) | Python | Full Platform | 68.3K |

---

*Research assisted by [OpenCode](https://opencode.ai). MIT License.*
