# LangGraph Orchestration

LangGraph is the **de facto standard** for building stateful, multi-agent systems. It reached v1.0 in late 2025 and surpassed CrewAI in GitHub stars in early 2026 thanks to enterprise adoption of its graph-based runtime. Unlike simple chains, LangGraph allows for **Cycles**, **State Persistence**, and **Human-in-the-Loop** interventions.

> **Why (the rationale):** LCEL chains are acyclic and stateless — LangGraph exists to fill the gap where agents need loops (ReAct retry), durable memory across sessions, and human approval gates that are architecturally impossible in a DAG.
> **When to use:** Choose LangGraph when your workflow has any of: conditional retry loops, multi-agent delegation, durable state that must survive crashes, or human-in-the-loop interrupts; simple one-shot pipelines do not need it.
> **Nuances & gotchas:** LangGraph gives explicit graph/state control at the cost of more boilerplate — you must define a typed state schema, explicit nodes, and edges; it is significantly more verbose than a two-line LCEL chain, so do not reach for it unless the cyclic or durable-state requirements are real.

## Table of Contents

- [The Graph Philosophy](#philosophy)
- [Cyclic vs. Acyclic Workflows](#cyclic)
- [State Management in LangGraph](#state)
- [Persistence and Checkpointing](#persistence)
- [Multi-Agent Orchestration Patterns](#multi-agent)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Graph Philosophy

In 2023, agents were "Black Boxes."
Today, agents are **Graphs**.
A graph consists of:
- **Nodes**: Python functions (The LLM, a tool, or data processing).
- **Edges**: Paths between nodes.
- **Conditional Edges**: Logic that determines the path based on the **State**.

> **Why (the rationale):** Representing agent logic as an explicit graph makes control flow auditable and testable — every decision point is a named edge rather than buried inside a prompt or a black-box API call.
> **When to use:** Use the graph philosophy any time you need to reason about, debug, or modify an agent's control flow — if you cannot draw the workflow as a diagram, a graph makes that explicit.
> **Nuances & gotchas:** More nodes is not always better — overly granular graphs with 20+ nodes become hard to reason about; group related operations into single nodes and use sub-graphs for modularity.

---

## Cyclic vs. Acyclic

Standard LangChain is **Acyclic** (Sequential).
LangGraph is **Cyclic**.
- **The Power of the Loop**: An agent can try a tool, see the error, and **cycle back** to the "Thinking" node to try again. This is the foundation of the **ReAct** pattern.

> **Why (the rationale):** Cycles are what separate a simple pipeline from a true agent — without them the system cannot self-correct, retry, or iterate on a plan; the loop is the core value of agentic design.
> **When to use:** Introduce cycles when the task has inherent uncertainty — tool calls that can fail, reasoning that needs multiple passes, or output quality that must be verified before proceeding.
> **Nuances & gotchas:** Cycles without a termination condition produce infinite loops — always define a max-iterations guard or a clear `END` condition in your conditional edge logic; LangGraph does not impose a default loop limit.

---

## State Management

The **State Schema** is the "Mind" of the graph.
```python
class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    plan: list[str]
    is_secure: bool
```
**Nuance**: Using `Annotated` with `add_messages` allows the graph to **Append** to history rather than overwriting it, preserving the full reasoning trajectory.

> **Why (the rationale):** A single, typed state struct shared across all nodes eliminates the coordination bugs that arise when different parts of a system maintain their own local variables — every node reads from and writes to one source of truth.
> **When to use:** Define state narrowly — include only fields that genuinely need to cross node boundaries; use sub-states for sub-graphs to keep each component's concerns isolated.
> **Nuances & gotchas:** State grows unboundedly if you store the full message history without trimming — use a `Trim Runnable` or set a token budget before the LLM node, or costs and latency will increase with every turn of a long conversation.

---

## Persistence and Checkpointing

Current LangGraph uses **Thread-based Persistence**.
- **The Concept**: Every session has a `thread_id`.
- **The Win**: If a user comes back after 2 days, the agent remembers the exact point it was at in a multi-step workflow.
- **Time-Travel**: Developers can "re-run" a specific thread from a previous state to debug a failure.

> **Why (the rationale):** Without checkpointing, any crash or timeout loses all intermediate reasoning — thread-based persistence makes long-running workflows durable and lets humans re-enter a paused workflow days later.
> **When to use:** Enable checkpointing for any workflow that takes more than a few seconds, spans multiple user turns, or requires human approval — stateless one-shot chains do not need it and the overhead is not worth paying.
> **Nuances & gotchas:** The checkpoint store (SQLite, Postgres, Redis) must be provisioned and maintained separately — the in-memory default is not suitable for production; also, time-travel debugging rewrites history in the thread, so be careful not to corrupt a live session when replaying for debug purposes.

---

## Multi-Agent Patterns

| Pattern | Description | Case Study |
|---------|-------------|------------|
| **Supervisor** | One "Manager" directs specialized workers. | Research Team |
| **Peer-to-Peer**| Agents hand off tasks to each other directly. | Customer Support |
| **Hierarchical**| Graphs within Graphs (Nested graphs). | Enterprise Engineering |

> **Why (the rationale):** Different tasks have different coordination topologies — a single all-knowing agent bottlenecks on context length and capability; specialized agents with a coordination pattern scale better and are easier to test in isolation.
> **When to use:** Start with the Supervisor pattern (easiest to reason about and debug); move to Hierarchical only when a single supervisor's context window becomes the bottleneck or when sub-domains are independently maintainable teams.
> **Nuances & gotchas:** Multi-agent systems multiply costs — every agent hop is at minimum one additional LLM call; measure total token spend per end-to-end task and set budgets before deploying; Peer-to-Peer patterns can produce hard-to-debug infinite delegation loops if agents do not have clear termination signals.

---

## Interview Questions

### Q: Why use LangGraph instead of OpenAI's "Assistant API"?

**Strong answer:**
**Control and Portability**. The Assistant API is a black box: you cannot see the exact prompts or control the logic gates. LangGraph is a **White Box framework**. I can use any model (OpenAI, Claude, Llama 3.3), control exactly when a tool is called, and inject my own custom validation logic between steps. More importantly, LangGraph is **Open Source** and can run locally/on-prem, which is critical for many enterprise security requirements.

### Q: How do you handle "State Overload" in a graph with 20+ nodes?

**Strong answer:**
We use **State Narrowing**. Instead of passing the entire global state to every node, we define specialized sub-states for sub-graphs. We also use **Trim Runnables** to prune the message history before it hits the LLM, ensuring we don't waste tokens while keeping the "Truth" preserved in the persistence layer. 

---

## References
- LangChain Team. "LangGraph: Multi-Agent Workflows at Scale" (2025)
- Anthropic. "Building Resilient Agents with State Machines" (2025)
- OpenSource AI. "Cycles and the Future of Agency" (2024 Tech Report)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **LangGraph** | A graph-based orchestration framework built on top of LangChain that supports cyclic, stateful agent workflows. | Provides the runtime for production multi-agent systems that need loops, memory, and human approval steps. |
| **Node** | A Python function in a LangGraph graph that performs a discrete unit of work such as calling an LLM or running a tool. | Encapsulates a single step in the agent's reasoning or execution pipeline. |
| **Edge** | A directed connection between two nodes in a LangGraph graph that defines the order of execution. | Determines which node runs next after the current node completes. |
| **Conditional Edge** | A LangGraph edge whose target node is chosen dynamically based on the current graph state at runtime. | Enables branching logic, letting the agent decide whether to loop, stop, or call a different tool. |
| **State Schema** | A typed data structure (TypedDict or Pydantic model) that holds all information shared across nodes in a LangGraph graph. | Acts as the agent's shared memory, ensuring every node reads from and writes to a single consistent source of truth. |
| **Cyclic Graph** | A graph that contains at least one loop, allowing execution to revisit a node more than once. | Powers the "try, observe, retry" pattern that lets agents recover from errors without human intervention. |
| **Acyclic (DAG)** | A graph where execution flows in one direction only, with no possibility of revisiting a previous step. | Describes simple sequential chains where every step runs exactly once. |
| **ReAct Pattern** | A reasoning strategy where the agent alternates between generating a thought, calling a tool, and observing the result. | Gives agents a principled way to decompose complex tasks into tool-assisted reasoning steps. |
| **Thread ID** | A unique identifier assigned to a single user session or workflow run in LangGraph. | Allows the persistence layer to store and retrieve the exact state of a long-running workflow. |
| **Checkpointing** | Saving the full graph state after each node execution to a durable store (database or file). | Enables a workflow to resume from its last saved step after a crash or after days of inactivity. |
| **Time-Travel** | A LangGraph feature that lets developers replay a workflow from any previously saved checkpoint. | Makes it possible to reproduce and debug failures by rewinding to the exact state where something went wrong. |
| **Thread-based Persistence** | A LangGraph storage model where every session's state is keyed by a thread ID. | Allows multiple users to run concurrent, independent workflows without their states interfering. |
| **Human-in-the-Loop (HITL)** | A workflow design where the agent pauses and waits for a human to approve or correct before continuing. | Adds a safety check at critical steps, ensuring humans stay in control of high-stakes decisions. |
| **Supervisor Pattern** | A multi-agent architecture where one orchestrator agent delegates tasks to specialized worker agents. | Scales complex tasks by splitting them across specialized agents while keeping overall coordination in one place. |
| **Peer-to-Peer (P2P) Pattern** | A multi-agent architecture where agents hand tasks directly to each other without a central coordinator. | Reduces bottlenecks for workflows where the next-best expert agent is determined by context, not a fixed plan. |
| **Hierarchical Pattern** | A multi-agent architecture that nests graphs inside other graphs, creating layers of supervisors and workers. | Handles enterprise-scale complexity by decomposing a large workflow into independently manageable sub-graphs. |
| **State Narrowing** | A technique of passing only the relevant sub-state to a given node instead of the entire global state. | Prevents nodes from being overwhelmed by irrelevant context and keeps each node's responsibility clear. |
| **Trim Runnable** | A LangGraph utility that removes older messages from the state before they are sent to the LLM. | Controls token usage and cost by keeping the active context window within the model's effective reasoning range. |
| **add_messages** | A LangGraph reducer function that appends new messages to the state list instead of overwriting it. | Preserves the full conversation history while avoiding the loss of earlier turns. |
| **White Box Framework** | A framework where the internal logic, prompts, and decision points are fully visible and controllable by the developer. | Contrasts with black-box APIs like the OpenAI Assistant API, enabling auditing, compliance, and custom validation. |

*Next: [LangSmith Observability](03-langsmith-observability.md)*
