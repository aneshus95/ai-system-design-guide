# Agent Fundamentals

Agents are LLM-powered systems that move beyond "chat" into "autonomous problem solving." The definition has shifted from simple ReAct loops to **Closed-Loop Reasoning Systems** that use built-in "System 2" thinking (Claude Opus 4.7 extended thinking, GPT-5.5 reasoning, DeepSeek-R2, Gemini 3.1 Pro Deep Think).

## Table of Contents

- [The Agent Formula](#formula)
- [System 1 (LLM) vs. System 2 (Reasoning Model)](#systems)
- [Agency Levels (Autonomous Spectrum)](#levels)
- [Core Components](#components)
- [The Agent Lifecycle](#lifecycle)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Agent Formula

Modern agency is often described as:
`Agent = Reasoning Model + Tool Use + Persistent Memory + Environment Feedback`

**Nuance**: In 2023, agents were "wrappers" around chat models. Today, agents are increasingly **Integrated**. Frontier models (Claude Opus 4.7, GPT-5.5 with reasoning, DeepSeek-R2) have the "Thinking" process baked into pre-training, making the agent loop more stable and less prone to "stalling."

---

## System 1 vs. System 2 Thinking

Architecting an agent requires choosing the right "Thinking Mode":

| Mode | Cognitive Type | Analogy | Current stack |
|------|----------------|---------|---------------|
| **System 1** | Fast, intuitive, reactive | Reflexes | Claude Haiku 4.5 / Sonnet 4.6 / GPT-5.5-mini / Gemini 3.1 Flash |
| **System 2** | Slow, logical, planning | Deliberation | Claude Opus 4.7 / GPT-5.5 reasoning / DeepSeek-R2 / Gemini 3.1 Pro Deep Think |

**The Design Pattern**: Use System 1 models for "Fast UI" and "Routing." Use System 2 models for "Decision Gates" and "Complex Planning."

---

## Agency Levels

Not every autonomous system is an "Agent." We categorize them by the **Level of Agency**:

1. **L0: Scripted Chains**: Fixed sequence (e.g., standard LangChain).
2. **L1: Tool-Enabled**: Model picks a tool but doesn't plan.
3. **L2: ReAct Agent**: Simple loop of "Thought -> Action -> Observation."
4. **L3: Autonomous Planner**: Decomposes a goal into a graph of sub-tasks.
5. **L4: Ambient Agent**: Runs in the background, intervenes only when necessary.

---

## Core Components

### 1. The Reasoning Model (The Executive)
The CPU of the agent. It determines the "Path to Success."

### 2. Tools (The Limbs)
Interfaces (APIs, Browsers, DBs) that allow the agent to affect the world.
> [!Note]
> The **Model Context Protocol (MCP)** is now the industry standard for tool interoperability, with adoption from Anthropic, OpenAI, Google, Microsoft, and AWS. Governance moved to the Linux Foundation's Agentic AI Foundation in December 2025.

### 3. Memory (The Experience)
- **Short-term**: Context window (KV Cache).
- **Long-term**: Vector DBs or persistent state (e.g., Mem0).

---

## The Agent Lifecycle

1. **Intake**: Receive user goal.
2. **Decomposition**: Break goal into sub-steps.
3. **Execution**: Call tools and handle results.
4. **Reflection**: Evaluate if the observation got the agent closer to the goal.
5. **Completion**: Synthesize final proof for the user.

---

## Interview Questions

### Q: Why is a "Reasoning Model" (like Claude Opus 4.7 or GPT-5.5 with extended thinking) better for agency than a standard LLM?

**Strong answer:**
Standard LLMs (System 1) predict the *very next token* based on pattern matching. When they encounter an error in a tool call, they often hallucinate a fix instead of admitting the failure. Reasoning Models use **Chain-of-Thought (CoT)** during inference. They "think" through multiple hidden turns before outputting a response. For an agent, this means higher **Path Reliability**—the model is significantly less likely to enter an infinite loop or try the same failing action twice because it has already simulated the failure internally.

### Q: How do you prevent "Agentic Drift" in long-running tasks?

**Strong answer:**
Agentic Drift occurs when the sub-steps take the agent so far from the original goal that it loses context. The standard solution is **Goal Anchoring**: include the "Original Objective" as a pinned system message and use a **Secondary Observer Model** (a smaller, cheaper model) to score every agent action against the original objective. If the score drops below a threshold, the agent is forced to "re-plan" from the root.

---

## References
- Kahneman, D. "Thinking, Fast and Slow" (applied to AI, 2025)
- OpenAI. "Learning to Reason with LLMs" (2024)
- DeepSeek. "R1: Cold-Start Data for Reasoning" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Agent** | An LLM-powered system that autonomously takes actions to achieve a goal, rather than just answering questions | The core unit of autonomous AI work |
| **LLM (Large Language Model)** | A neural network trained on large amounts of text that can generate and understand language | The reasoning engine inside an agent |
| **Closed-Loop Reasoning System** | An agent that continuously feeds results back into its own thinking cycle until it reaches a goal | Enables self-correction and multi-step problem solving |
| **System 1 Thinking** | Fast, intuitive responses generated without deep deliberation — like reflexes | Best for routing, quick UI responses, and simple tasks |
| **System 2 Thinking** | Slow, deliberate reasoning that works through logic step by step before responding | Best for complex planning and high-stakes decisions |
| **ReAct** | A loop pattern where the model alternates between Thought, Action, and Observation steps | The foundational pattern for most agent control flows |
| **Chain-of-Thought (CoT)** | Having the model write out its reasoning steps before producing a final answer | Improves accuracy on multi-step tasks |
| **Extended Thinking** | A mode where a reasoning model spends extra compute internally working through a problem before responding | Boosts reliability for complex, multi-step agent tasks |
| **Reasoning Model** | An LLM that uses built-in chain-of-thought during inference rather than only at prompt time | Acts as the high-quality decision-maker inside an agent |
| **Agency Level (L0–L4)** | A scale from fully scripted chains (L0) to fully autonomous background agents (L4) | Helps designers choose the right autonomy depth for a use case |
| **Tool Use** | The ability for an agent to call external functions, APIs, or browsers to affect the real world | Extends the agent beyond text generation into real actions |
| **Model Context Protocol (MCP)** | An industry-standard protocol that defines how agents connect to external tools and data sources | Enables portable, interoperable tool integrations across different LLM providers |
| **Short-Term Memory** | Information held in the model's active context window for the current session | Lets the agent track what has happened in the current task |
| **Long-Term Memory** | Information stored in external databases (like vector stores) that persists across sessions | Allows the agent to recall past work and user preferences |
| **KV Cache** | A hardware-level cache that stores previously computed attention keys and values to speed up inference | Reduces cost and latency for agents with long, repeated context |
| **Vector DB** | A database that stores data as numerical embeddings, allowing retrieval by semantic similarity | Powers long-term memory and knowledge retrieval in agents |
| **Agentic Drift** | When an agent's sub-steps gradually move so far from the original goal that it loses track of what it was asked to do | A key failure mode in long-running agents |
| **Goal Anchoring** | Keeping the original objective pinned in the system prompt and re-checking against it throughout execution | Prevents the agent from drifting off-task over many steps |
| **Secondary Observer Model** | A smaller, cheaper model that monitors the main agent's actions and scores them against the original goal | A safety net that catches drift without full reprocessing |
| **Path Reliability** | How consistently an agent follows a valid sequence of steps to reach its goal without looping or hallucinating | A key quality metric for reasoning models in agentic use |
| **Hallucination** | When a model generates plausible-sounding but factually incorrect or fabricated information | A major failure mode that must be mitigated in production agents |
| **Decomposition** | Breaking a large goal into smaller, manageable sub-tasks | Enables agents to handle complex, multi-stage work |
| **Inference-Time Scaling** | Spending more compute during response generation (rather than training) to improve output quality | Allows agents to reason more deeply without retraining |

*Next: [Reasoning Loops: ReAct and Beyond](02-reasoning-loops-react-and-beyond.md)*
