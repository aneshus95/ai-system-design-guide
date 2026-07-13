# Multi-Agent Orchestration

Complex systems are rarely one agent. They are teams of specialized agents. Orchestration has matured from "Blind Managers" to **Hierarchical Supervisors**, **Dynamic Swarms**, and **Cross-Vendor Agent Networks** enabled by interoperability protocols like A2A. Gartner projects that 40% of enterprise applications will feature task-specific AI agents by end of 2026, up from under 5% in early 2025.

## Table of Contents

- [Why Multi-Agent?](#why)
- [The Supervisor Pattern](#supervisor)
- [The Pipeline Pattern](#pipeline)
- [Swarms and Peer-to-Peer (P2P)](#swarms)
- [Graph-Based Orchestration (2026 Dominant Pattern)](#graph-orchestration)
- [Cross-Vendor Agent Orchestration via A2A](#cross-vendor)
- [The 2026 Framework Landscape for Multi-Agent](#framework-landscape)
- [State Management in Agent Teams](#state)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Multi-Agent?

A single agent with 50 tools experiences **Cognitive Load**.
1. **Specialization**: A "Code Agent" can use a model optimized for Python, while a "Search Agent" uses a model optimized for RAG.
2. **Parallelism**: Multiple agents can work on independent sub-tasks simultaneously.
3. **Decoupled Evaluation**: You can evaluate the "Writer Agent" separately from the "Researcher Agent."

---

## The Supervisor Pattern (Hierarchical)

The most common enterprise pattern as of 2026.

- **The Supervisor**: A high-reasoning model (Claude Opus 4.7, GPT-5.5 reasoning, Gemini 3.1 Pro Deep Think) that decomposes the user prompt and delegates to workers.
- **Workers**: Fast, cost-efficient models (Claude Haiku 4.5, Gemini 3.1 Flash, GPT-5.5-mini) that perform the work.
- **Reviewer**: A separate agent that validates the consolidated output against the supervisor's original plan.

**Architecture**: LangGraph remains the dominant framework for implementing these state-aware hierarchical loops. The Claude Agent SDK, Google ADK, and Microsoft Agent Framework all support this pattern natively as of 2026.

---

## Swarms (The OpenAI Pattern)

 popularized in late 2024, **Swarms** focus on "Handoffs."

- One agent "Hands off" the conversation to another.
- **Key concept**: `Handoff(TargetAgent)`.
- **Benefit**: No central "Manager" bottleneck. The conversation flows naturally between specialized entities.

---

## Graph-Based Orchestration (2026 Dominant Pattern)

The architectural momentum in 2026 has shifted decisively toward **graph-based orchestration**, where agent workflows are modeled as directed graphs with typed state.

### Why Graphs Won

- **Explicit control flow**: Nodes are agents or functions; edges define transitions, including conditional branches and loops
- **Visualizable**: Teams can inspect and debug the workflow as a diagram
- **State-aware**: Typed state objects pass through the graph, enabling checkpointing and resumption

### Framework Support

| Framework | Graph Model | Key Differentiator |
|-----------|-------------|-------------------|
| **LangGraph** (24k stars) | Imperative DAG with typed state | Most mature, broadest community |
| **Google ADK** (17k stars) | Agent graphs with built-in A2A | Native Google Cloud integration |
| **Microsoft Agent Framework** | Workflow graphs (sequential, concurrent, handoff) | Unified .NET + Python, enterprise governance |
| **Claude Agent SDK** | Supervisor-based hierarchical trees | Built-in tools (bash, editor), production-ready |

### The Paperclip Pattern (Hierarchical Agents at Scale)

A notable 2026 development is **Paperclip** (44,900 GitHub stars within three weeks of its March 2026 launch). It uses a hierarchical model where a CEO agent receives a top-level goal, decomposes it, and delegates to manager agents who spawn and coordinate worker agents. This pattern demonstrates how deeply hierarchical multi-agent trees can handle complex real-world tasks.

> *Verified May 2026.*

---

## Cross-Vendor Agent Orchestration via A2A

The **Agent-to-Agent (A2A) protocol** (see [Tool Use and MCP](03-tool-use-and-mcp.md#a2a)) enables a new multi-agent pattern: **cross-vendor orchestration**. Before A2A, multi-agent systems required all agents to share the same framework and runtime. Now:

1. **Agent Discovery**: An orchestrator finds specialist agents via their **Agent Cards** (JSON metadata describing capabilities)
2. **Task Delegation**: The orchestrator sends a structured task to a remote agent via HTTP/SSE
3. **Async Progress**: The remote agent streams status updates back; the orchestrator can delegate to other agents in parallel
4. **Result Collection**: Final artifacts are returned and integrated into the orchestrator's state

**Production example**: A procurement system where the orchestrator (LangGraph) delegates compliance checking to a specialized agent (Google ADK), inventory lookup to an MCP-connected tool, and contract generation to a CrewAI crew — all communicating via A2A and MCP respectively.

> *Verified May 2026. Source: a2a-protocol.org*

---

## The 2026 Framework Landscape for Multi-Agent

Every major AI lab now ships an agent framework. The multi-agent orchestration landscape as of May 2026:

| Framework | Provider | Multi-Agent Model | Status |
|-----------|----------|-------------------|--------|
| **LangGraph** | LangChain | Graph-based, most flexible | Production (126k stars) |
| **Claude Agent SDK** | Anthropic | Supervisor trees with built-in tools | GA (Python + TypeScript) |
| **Google ADK** | Google | Graph-based with A2A native support | GA (Python, TS, Java, Go) |
| **Microsoft Agent Framework** | Microsoft | Workflows + group chat patterns | RC 1.0 (Feb 2026), GA Q2 2026 |
| **OpenAI Agents SDK** | OpenAI | Handoff-based swarms with guardrails | GA (Python + TypeScript) |
| **CrewAI** | CrewAI Inc. | Role-based crews with Flows | v1.13 (60%+ Fortune 500) |
| **Smolagents** | HuggingFace | Lightweight, open-source | Active development |

**Key trend**: No single framework excels at all four multi-agent patterns (supervisor, swarm, pipeline, debate). Teams increasingly combine frameworks — e.g., LangGraph for complex orchestration with CrewAI for business-user-facing automations.

> *Verified May 2026.*

---

## State Management

The biggest challenge in multi-agent systems is the **Shared Blackboard**.

1. **Local State**: Context only visible to a specific agent.
2. **Global State**: Shared memory (e.g., the final draft) visible to all.
3. **Write Conflicts**: When two agents try to modify the same Global State.
   - **Best practice**: Use **Transactional Handoffs**. An agent can only write to the global state when it "Owns" the lock.

---

## Peer-to-Peer (P2P) Debate

For high-accuracy tasks (e.g., Legal or Medical), we use **Agentic Debate**.
- **Agent A**: Proposes an answer.
- **Agent B**: Tries to find flaws in Agent A's answer.
- **Agent A**: Refines the answer based on B's critique.
- **Result**: Convergence on a higher-quality result than any single agent could produce.

---

## Interview Questions

### Q: What are the main failure modes of a "Supervisor" multi-agent architecture?

**Strong answer:**
The primary failure mode is **Decomposition Failure**. If the Supervisor agent breaks a task into sub-tasks that are logically inconsistent or have hidden dependencies, the workers will produce correct answers to the *wrong questions*. The standard fix is **Iterative Planning**: the Supervisor must get "Confirmation of sub-task feasibility" from the workers before they begin execution. Another failure is **Context Dilution**, where the global state becomes so bloated with worker logs that the Supervisor loses the "Big Picture."

### Q: How do you choose between a "Sequence of Chains" and a "Multi-Agent Graph"?

**Strong answer:**
I use a **Sequence of Chains** when the task is linear and deterministic (e.g., Extract -> Translate -> Summarize). I use a **Multi-Agent Graph** (like LangGraph) when the task is **Non-Linear** or requires **Conditional Loops**. For example, if the "Translate" step might fail and need to go back to "Extract" for more context, a static chain breaks, but a graph can self-correct by routing back to an earlier node.

### Q: When would you use A2A for multi-agent orchestration versus keeping all agents in a single framework?

**Strong answer:**
I keep agents in a single framework when the team owns all agents, they share the same runtime, and low latency between agent calls is critical. I introduce A2A when crossing **organizational or vendor boundaries** — for example, when my orchestrator needs to delegate to a compliance agent maintained by a different team, or when integrating a third-party specialized agent (e.g., a legal review service). A2A adds HTTP overhead but provides **vendor neutrality**, **independent scaling**, and **capability discovery** via Agent Cards. The rule of thumb: same team, same framework; different team or vendor, use A2A.

---

## References
- Wu et al. "AutoGPT: An Autonomous GPT-4 Experiment" (Historical/2025 update)
- Li et al. "Camel: Communicative Agents for 'Mind' Exploration" (2023/2025)
- OpenAI. "Swarms Framework" (2024/2025)
- Google. "Agent2Agent Protocol" (2025/2026)
- Gartner. "Predicts 2026: AI Agent Market" (2025)
- Andrew Ng. "Agentic Design Patterns" (2025/2026)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Multi-Agent System** | An architecture where multiple specialized agents collaborate to complete a task that one agent alone cannot handle well | Enables parallelism, specialization, and independent evaluation of sub-tasks |
| **Orchestration** | The process of coordinating multiple agents — directing who does what and when | Ensures agents work together in a coherent, goal-directed way |
| **Supervisor Pattern** | A hierarchical design where one high-reasoning model decomposes tasks and delegates to worker agents | The most common enterprise multi-agent architecture |
| **Supervisor Agent** | A powerful model that plans, decomposes a goal, and assigns sub-tasks to worker agents | Acts as the "manager" that ensures work is done correctly |
| **Worker Agent** | A smaller, cheaper model assigned a specific sub-task by the supervisor | Executes the actual work efficiently within its specialty |
| **Reviewer Agent** | A dedicated agent that validates consolidated output against the supervisor's original plan | Catches errors or inconsistencies before final results are returned |
| **Swarm** | A multi-agent pattern (popularized by OpenAI) where agents hand off tasks to each other without a central manager | Eliminates bottlenecks by allowing organic conversation flow between agents |
| **Handoff** | The act of one agent transferring control and context to another agent to continue the task | Enables decentralized, peer-to-peer agent collaboration |
| **Graph-Based Orchestration** | Modeling an agent workflow as a directed graph where nodes are agents and edges are transitions | Makes control flow explicit, visualizable, and state-aware |
| **DAG (Directed Acyclic Graph)** | A graph where edges have direction and there are no cycles, used to model task dependencies | Ensures tasks execute in the right order without circular dependencies |
| **LangGraph** | A Python framework for building stateful, graph-based agent workflows | The dominant framework for complex multi-agent orchestration |
| **Google ADK** | Google's Agent Development Kit, a framework supporting graph-based agents with native A2A protocol support | Enables Google Cloud-native multi-agent systems |
| **CrewAI** | A framework for building role-based "crews" of collaborating agents | Popular for business-process automations with clearly defined agent roles |
| **Agent-to-Agent (A2A) Protocol** | An open protocol for one agent to delegate tasks to another agent from a different vendor or runtime | Enables cross-vendor multi-agent coordination without shared code |
| **Agent Cards** | JSON metadata documents describing an agent's capabilities, endpoint, and authentication for discovery | Allow orchestrators to find and integrate specialist agents from other vendors |
| **Cross-Vendor Orchestration** | Coordinating agents built on different frameworks or by different organizations in a single workflow | Breaks down silo boundaries between AI systems from different vendors |
| **Cognitive Load** | The degradation in performance when a single agent has too many tools or responsibilities at once | Motivates splitting work across specialized agents |
| **Shared Blackboard** | A global state store that all agents in a system can read from and write to | Enables information sharing between agents but introduces write conflict risks |
| **Transactional Handoff** | A pattern where only one agent can hold the write lock on global state at a time | Prevents data corruption from two agents modifying the same state simultaneously |
| **Agentic Debate (P2P)** | A pattern where two agents argue — one proposes, one critiques — until they converge on a high-quality answer | Improves accuracy for high-stakes decisions like legal or medical reasoning |
| **Decomposition Failure** | When the supervisor breaks a task into sub-tasks with hidden dependencies or logical conflicts | Causes workers to produce correct answers to the wrong questions |
| **Context Dilution** | When the shared global state grows so large the supervisor loses track of the overall goal | A failure mode in long-running hierarchical agent systems |
| **Paperclip Pattern** | A deeply hierarchical agent architecture (CEO → manager → worker) for handling large, complex goals | Demonstrates how multi-level delegation can tackle real-world scale tasks |
| **RAG (Retrieval-Augmented Generation)** | A technique where an agent queries an external knowledge base to ground its responses in retrieved facts | Used in research and search agents to reduce hallucination |

*Next: [Agent Memory and State](05-agent-memory-and-state.md)*
