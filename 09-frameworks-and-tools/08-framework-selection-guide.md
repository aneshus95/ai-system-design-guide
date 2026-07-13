# Framework Selection Guide

The landscape of AI frameworks has consolidated significantly over the past year. Every major AI lab now ships an agent SDK, Microsoft merged AutoGen and Semantic Kernel into a unified Agent Framework, and interoperability protocols (MCP, A2A) are table stakes. This guide provides the **Decision Matrix** for choosing your stack based on production requirements, team expertise, and system scale.

## Table of Contents

- [The Framework Landscape](#landscape)
- [The Decision Matrix](#matrix)
- [Build vs. Buy vs. Framework](#build-vs-buy)
- [Anti-Patterns to Avoid](#anti-patterns)
- [Staff-Level Recommendation](#recommendation)
- [Interview Questions](#interview-questions)

---

## The Framework Landscape

### Orchestration & Agent Frameworks

| Framework | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **LangGraph** | L1 (Core) | Precise state control, graph-based | Complexity, steep learning curve |
| **DSPy** | L1 (Core) | Reliability & Optimization | Upfront cost (Training) |
| **LlamaIndex**| L2 (Data) | Advanced Retrieval (RAG) | Logic flexibility |
| **CrewAI** | L3 (App) | Business process speed, enterprise RBAC | Hides failures |
| **MS Agent Framework** | L1 (Enterprise) | Unified .NET + Python, replaces AutoGen + SK | RC status (GA Q2 2026) |

### Agent SDKs (Lab-Specific)

| Framework | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **Claude Agent SDK** | L1 (Agent) | Built-in tools, production agent loop | Requires Anthropic API |
| **OpenAI Agents SDK** | L1 (Agent) | Lightweight handoffs, guardrails | OpenAI-centric |
| **Google ADK** | L1 (Agent) | Multi-language, native A2A + Google Cloud | Google ecosystem bias |

### Coding Agents

| Framework | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **Claude Code** | L1 (Coding) | Autonomous CLI coding agent | Requires Anthropic API |
| **Cursor / Windsurf** | L2 (IDE) | Tight IDE + agent integration | Closed-source infra |
| **OpenHands** | L2 (Coding) | Open-source autonomous agent | Requires self-hosting |

> **April 2026 note**: Semantic Kernel is no longer listed as a standalone framework. It has been merged into the Microsoft Agent Framework. Existing SK users should plan migration.

---

## The Decision Matrix

**Use this logic to select your stack:**

### Core Orchestration
1. **Is it a pure RAG app?** → **LlamaIndex**.
2. **Does it require long-running state/Human-in-the-loop?** → **LangGraph**.
3. **Is high reliability (99%+) and cross-model portability critical?** → **DSPy**.
4. **Are you a C#/.NET enterprise shop?** → **Microsoft Agent Framework** (replaces Semantic Kernel + AutoGen).
5. **Are you building high-level automations for business users?** → **CrewAI + Flows**.

### Agent SDKs (choose based on your primary model provider)
6. **Building agents on Claude / Anthropic API?** → **Claude Agent SDK** (Python/TS, built-in tools for file/code/command).
7. **Building agents on OpenAI API?** → **OpenAI Agents SDK** (lightweight handoffs, guardrails, MCP support).
8. **Building agents on Google Cloud / Gemini?** → **Google ADK** (native A2A, Vertex AI deployment, multi-language).
9. **Need cross-vendor agent communication?** → Use **A2A protocol** on top of any framework above.

### Coding Agents
10. **Are you doing autonomous file-system level coding tasks?** → **Claude Code** (CLI) or **Cline** (VS Code).
11. **Need open-source coding agent that works with any LLM?** → **OpenHands** (Docker).
12. **Want the best IDE experience with AI?** → **Cursor** (closed) or **Windsurf** (Codeium).

---

## Build vs. Buy vs. Framework

As a Staff Engineer, you must resist **Framework Bloat**.

- **Use a Framework** when it solves a **Non-Trivial Computer Science Problem** (e.g., State persistence, Bayesian prompt optimization, Vector-Graph linking).
- **Build Custom (Thin Wrapper)** when you are just making simple calls to an LLM. Frameworks add latency, update-churn, and debugging overhead that isn't worth it for a single-turn agent.

---

## Anti-Patterns to Avoid

1. **Framework Tunnelling**: Trying to force a complex logic flow into a framework that doesn't support it (e.g., using a pure RAG library for a coding agent).
2. **The Golden Hammer**: Using LangChain just because it's popular, when a 50-line Python script would be faster and cheaper.
3. **Ignoring Observability**: Deploying any framework without an LLOps layer (LangSmith/Phoenix).

---

## Staff-Level Recommendation

For a modern, production-grade agentic system:
- **Orchestration**: LangGraph (for state and loops) or Microsoft Agent Framework (for .NET shops).
- **Agent SDK**: Match to your model provider — Claude Agent SDK (Anthropic), Agents SDK (OpenAI), ADK (Google). All support MCP for tool access.
- **Optimization**: DSPy (to compile prompts for different model tiers).
- **Retrieval**: LlamaIndex (for multi-stage RAG).
- **Observability**: LangSmith (for tracing and evaluation).
- **Cross-vendor agents**: A2A protocol for agent-to-agent coordination across organizational boundaries.
- **Autonomous coding**: Claude Code (CLI) or Cline (VS Code) for file-level editing tasks.
- **Open coding agent**: OpenHands for self-hosted or CI pipeline integration.

**The 2026 insights**:
1. Agentic coding tools (Claude Code, Cursor, OpenHands) are not replacements for orchestration frameworks — they are a **new category** that operates at the file-system level, above the LLM API but below the application logic.
2. The protocol layer has matured: **MCP for agent-to-tool** and **A2A for agent-to-agent** are becoming infrastructure standards, not optional add-ons. Design your architecture to support both.
3. Every lab shipping its own agent SDK creates a **vendor lock-in risk**. Mitigate by using MCP for tool access (portable across SDKs) and A2A for agent coordination (vendor-neutral).

> *Updated May 2026.*

---

## Interview Questions

### Q: Why do we see a trend towards "Programming" (DSPy) instead of "Prompting"?

**Strong answer:**
**Industrialization**. Prompt engineering is "Alchemy": it is inconsistent and does not scale. Programming LLMs via Frameworks like DSPy allows us to treat AI as a **Software Engineering discipline**. We can apply CI/CD, unit testing (metrics), and automated optimization. This moves AI from "Nondeterministic Magic" to a **Predictable Component** of a larger distributed system, which is a requirement for any mission-critical production environment.

### Q: If you had to build a system that works across OpenAI, Anthropic, and local Llama models, how would you architect it?

**Strong answer:**
I would use **DSPy** for the prompt layer and **LangGraph** for the orchestration layer. DSPy's **Signatures** allow me to decouple the task definition from the model's specific behavior. I would then use a **Universal Model Gateway** (like LiteLLM or an internal proxy) to handle the different API formats. For tool access, I would use **MCP** — it is model-agnostic, so the same MCP servers work regardless of which LLM backend is active. If I need cross-team agent coordination, I would use **A2A** at the boundary layer. This stack ensures that if I need to switch from GPT-4o to Claude Sonnet 4 for cost or latency reasons, I do not have to rewrite 50 prompts; I just re-compile or update the config.

### Q: With every AI lab shipping its own agent SDK (Claude Agent SDK, OpenAI Agents SDK, Google ADK), how do you avoid vendor lock-in?

**Strong answer:**
The key is to **separate the orchestration layer from the model layer**. I use a framework-agnostic orchestrator like LangGraph or a thin custom wrapper for the core workflow logic. Model-specific SDKs are useful for prototyping or when you are committed to a single provider, but for production multi-vendor systems, I keep the model interaction behind an abstraction (LiteLLM gateway or DSPy signatures). For tool access, **MCP** provides portability — the same MCP server works with any SDK. For agent coordination, **A2A** provides vendor-neutral agent-to-agent communication. The practical rule: use lab-specific SDKs at the leaf nodes (individual agent implementations) but keep the orchestration graph vendor-neutral.

---

## References
- Google Cloud. "Enterprise Generative AI Reference Architecture" (2025)
- Gartner. "Magic Quadrant for AI Application Frameworks" (2025)
- Gartner. "Predicts 2026: 40% of Enterprise Apps to Feature AI Agents" (2025)
- Thoughtworks. "Technology Radar: The Rise of Agentic Frameworks" (Nov 2024/2025)
- Microsoft. "Agent Framework Overview" (2026)
- Anthropic. "Claude Agent SDK" (2026)
- Google. "Agent Development Kit" (2026)
- OpenAI. "Agents SDK" (2026)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Decision Matrix** | A structured table that maps requirements to framework choices based on explicit criteria. | Removes subjective guesswork from framework selection by anchoring the decision to concrete production requirements. |
| **Framework Bloat** | The tendency for frameworks to add unnecessary abstractions, dependencies, and complexity over time. | Understanding this risk helps teams choose the leanest tool that solves their actual problem. |
| **Framework Tunnelling** | The anti-pattern of forcing a use case into a framework that is a poor fit for it. | Highlights the importance of matching the tool's strength to the problem shape before committing to a framework. |
| **Golden Hammer** | The anti-pattern of using a familiar tool for every problem regardless of fit. | Warns against defaulting to a popular framework like LangChain when a simpler solution would be faster and cheaper. |
| **LLOps Layer** | The observability and evaluation infrastructure for LLM systems, analogous to MLOps for traditional ML. | Provides the monitoring, tracing, and evaluation capabilities needed to operate a production AI system safely. |
| **MCP (Model Context Protocol)** | An open standard interface for connecting agents to external tools. | Enables portability: the same MCP server works with any agent SDK, avoiding vendor lock-in at the tool layer. |
| **A2A Protocol (Agent-to-Agent)** | An open standard for communication between agents from different vendors or frameworks. | Provides vendor-neutral agent coordination across organizational boundaries. |
| **LiteLLM** | A proxy library that translates calls to many different LLM provider APIs into a single unified interface. | Allows teams to switch between OpenAI, Anthropic, Gemini, and local models without rewriting call sites. |
| **Universal Model Gateway** | A service that sits in front of multiple LLM providers and exposes a single API to the application. | Decouples the application from specific model providers, enabling cost-driven or capability-driven model routing. |
| **Vendor Lock-in** | A situation where a system is so dependent on a specific provider's tools that switching to another is prohibitively costly. | Understanding this risk motivates using open standards (MCP, A2A) and abstraction layers at provider boundaries. |
| **State Persistence** | The ability to save and restore the current state of a workflow so it can resume after an interruption. | Required for long-running agent workflows that span multiple sessions or days. |
| **Bayesian Prompt Optimization** | A statistical search method used by DSPy to find optimal prompt text by sampling candidate prompts intelligently. | Produces reliable prompts with fewer LLM calls than exhaustive search, lowering compilation cost. |
| **Vector-Graph Linking** | An indexing approach that connects embedding-based vector chunks to structured graph nodes. | Enables retrieval that combines semantic similarity with structured relational reasoning. |
| **CI/CD Pipeline** | The automated sequence of steps that build, test, and deploy software changes. | Provides the infrastructure into which autonomous coding agents (Claude Code, OpenHands) can be embedded. |
| **Autonomous Coding Agent** | A software agent that can read a codebase, write code, run tests, and iterate until a task is complete without human assistance at each step. | Represents a new category of tool that operates at the file-system level above the LLM API. |
| **OpenHands** | An open-source autonomous coding agent that runs in a Docker sandbox and supports any LLM backend. | Provides a self-hostable alternative to Claude Code for teams with open-source or data-privacy requirements. |
| **Cline** | An open-source VS Code extension that acts as an autonomous coding agent with full MCP support. | Gives developers a free, model-flexible coding agent embedded directly in their editor. |
| **Pure RAG** | A system architecture where the only AI workflow is retrieving relevant documents and generating a response. | The simplest agentic pattern; when this is the full scope, LlamaIndex is typically the best fit. |
| **DSPy Signature** | A declarative class that defines a task's inputs and outputs without specifying the prompt text. | Provides the model-agnostic task definition that DSPy compiles into optimized prompts for any LLM. |

*Next: [Navigating Framework Churn](12-navigating-framework-churn.md)*
