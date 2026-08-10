# Error Handling and Recovery

Agents fail in non-deterministic ways. Error handling has moved from "Try-Catch blocks" to **Agentic Self-Correction** and **Stateful Rollbacks**, with frameworks like LangGraph and Microsoft Agent Framework providing native checkpoint/resume primitives.

## Table of Contents

- [The Taxonomy of Agent Failures](#fail-types)
- [Self-Correction Loops](#correction)
- [Stateful Rollbacks (Checkpointing)](#rollbacks)
- [The "Stuck in a Loop" Fix](#stuck)
- [Graceful Degradation](#degradation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Taxonomy of Agent Failures

1. **Hallucinated Tools**: Calling a tool that doesn't exist.
2. **Schema Violation**: Passing the wrong arguments to a real tool.
3. **Environment Error**: Tool exists, but the external API is down.
4. **Logical Stall**: The agent performs the same failing action repeatedly (The ReAct Loop of Death).

> **Why (the rationale):** Agentic failures are fundamentally different from software exceptions — they are inputs to the reasoning process, not terminal stop conditions. Classifying them forces you to route each type to the right handler (self-correction, retry, rollback, or escalation) rather than applying a single blanket catch.
> **When to use:** Apply this taxonomy at the start of any agent system design to map out your failure surface before writing retry or recovery logic; especially critical when your agent has write-side-effect tools where the wrong handler can double-execute or silently corrupt state.
> **Nuances & gotchas:** The four categories are not mutually exclusive — a hallucinated tool call that partially executes before failing can also create an environment error; "Logical Stall" is the most expensive category because it only manifests over time, making it invisible to per-call monitoring.

---

## Self-Correction Loops

Errors are now treated as **Tokens of Information**.

> **Why (the rationale):** Traditional try/catch stops execution at the point of failure; self-correction loops convert the failure into a prompt observation, giving the model a chance to reason around it and try an alternative — the difference between a dead end and a course correction.
> **When to use:** Use when tool failures are recoverable (wrong arguments, transient API errors, wrong tool choice) and when the model has enough context to understand what went wrong; less appropriate when the failure is fundamental (the target resource genuinely does not exist) because the model will loop indefinitely trying variations.
> **Nuances & gotchas:** Retries with error injection can amplify load on a struggling downstream service; always pair with a maximum retry counter. Reasoning models do improve one-shot recovery rates, but they also consume significantly more tokens per correction cycle — budget accordingly. This does NOT handle silent failures where the tool returns success with wrong data.

- **Pattern**: When a tool fails, the error message is NOT just logged; it is fed back to the model as a prompt: *"Action failed with error: X. Reflect on why this happened and provide an alternative strategy."*
- **Reasoning Models** (Claude Opus 4.7 extended thinking, GPT-5.5 reasoning, DeepSeek-R2): These models excel at this because they "internalize" the error during their hidden Chain-of-Thought, leading to a much higher one-shot recovery rate.

---

## Stateful Rollbacks (Checkpointing)

For long-running agents, an error in Step 9 shouldn't crash the whole project.

> **Why (the rationale):** Without checkpoints, any failure in a long multi-step run means restarting from scratch, re-paying all prior token costs and re-executing all prior tool calls including non-idempotent ones. Checkpointing makes recovery cheap and safe by giving a known-good restore point.
> **When to use:** Essential for any agent run that takes more than a few steps, has non-idempotent tool calls, or runs longer than a single process lifetime; less critical for short, stateless, read-only agents where a full restart is cheap and side-effect-free.
> **Nuances & gotchas:** A state snapshot captures the agent's in-memory state but does NOT guarantee exactly-once execution of side effects — if the agent called an external API between the last checkpoint and the crash, the API may have executed while the checkpoint recorded nothing. Rolling back to Step 5 and replaying forward can re-fire those calls unless they use idempotency keys or a full durable-execution engine (see Chapter 11).

- **Checkpoints**: High-reliability systems (using LangGraph or similar) save the "State Snapshot" to a DB after every successful tool call.
- **The Rollback**: If the agent enters a logical stall, the supervisor agent can **Reset common-state** to Step 5—the last "Safe" state—and force a different path.

---

## The "Stuck in a Loop" Fix

Infinite loops are the #1 cost-sink in agentic systems.

> **Why (the rationale):** Models cannot reliably detect their own stagnation — they will keep generating plausible-sounding new reasons to retry the same failing call. The harness must enforce stagnation detection deterministically because leaving it to the model's self-assessment is the root cause of the failure.
> **When to use:** Counter-based intervention should be active in every production agentic system as a baseline circuit breaker; tune the threshold (e.g., 3 identical calls) based on your tool's expected retry profile — some tools are legitimately retried twice, so a threshold of 2 would produce false positives.
> **Nuances & gotchas:** Matching on the exact `(Tool, Args)` tuple can miss semantic loops where the agent tries slightly varied arguments to the same dead-end query; complement with plan-similarity checks (95%+ similarity between successive plans) to catch disguised oscillation. Pivot Instructions must be strong and imperative — a soft suggestion ("you might try a different approach") is often overridden by the model's in-context reasoning.

**Solution**: **Counter-Based Intervention**.
1. If the same `(Tool, Args)` tuple is seen 3 times in one session, the orchestrator interrupts the model.
2. It injects a mandatory **"Pivot Instruction"**: *"You have tried searching for 'X' three times. This path is dead. You MUST try a different tool or admit you are stuck."*

---

## Graceful Degradation

If the high-reasoning agent (Claude Opus 4.7, GPT-5.5 reasoning) keeps failing, we fall back to:

> **Why (the rationale):** A total failure returning nothing is almost always worse for the user than a partial or simplified response. Graceful degradation preserves user value even when the primary path is broken, and it reduces the blast radius of model or tool outages.
> **When to use:** Design degradation tiers upfront for any production agent; especially important when the primary agent uses expensive frontier models or unreliable third-party tools — the fallback should be meaningfully cheaper and more reliable, not just a slightly different version of the same failing path.
> **Nuances & gotchas:** RAG-only mode is safe but may mislead users who expect action-taking capability — always surface clear UI messaging that the agent is running in a limited mode. Human escalation is the most reliable fallback but creates a latency cliff and requires on-call staffing; it does NOT scale as a primary recovery path for high-volume systems.
- **Simplified Agent**: A smaller model with fewer, more reliable tools.
- **RAG-only Mode**: Disable actions and just provide a conceptual answer based on the knowledge base.
- **Human Escalation**: (See the next chapter).

---

## Interview Questions

### Q: Why is traditional "Exception Handling" (Try/Catch) insufficient for Agentic Systems?

**Strong answer:**
In traditional software, an exception is a "Stop" command. In an agentic system, the model is the "Driver." If the system just stops, the user task fails. We use **Error Injection** instead of Exception Handling. We catch the exception at the platform level and transform it into a **Synthesized Observation** for the model. This allows the model to "Reason" around the failure. A TRY/Catch only fixes the code; Error Injection allows the model to fix the **Plan**.

### Q: How do you handle "Silent Failures" (Where the tool returns 200 OK but the data is wrong)?

**Strong answer:**
Silent failures are the most dangerous. We implement **Output Validation Agents**. For critical steps, we don't just accept the tool output. We pipe the output to a "Verifier Agent" (often a smaller, faster model) whose only job is to check: *"Does this tool output actually answer the query provided?"* If the Verifier says "No," it triggers a self-correction loop as if it were a hard error.

---

## References
- LangGraph. "Persistence and Checkpointing" (2025)
- Shinn et al. "Reflexion: Learning from Errors" (2024 update)
- Microsoft. "Managing Hallucinations in Agentic Systems" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Hallucinated Tool** | When a model tries to call a tool that does not exist in its available tool list | A common agent failure mode that must be caught and handled gracefully |
| **Schema Violation** | Passing incorrectly typed or structured arguments to a real tool | Causes tool execution to fail even when the right tool is chosen |
| **Environment Error** | A failure where the external API or system the tool connects to is unavailable or broken | An external failure the agent cannot fix on its own, requiring a fallback strategy |
| **Logical Stall** | When an agent gets stuck repeating the same failing action without making progress | The most expensive agentic failure mode, often resulting in infinite loops |
| **ReAct Loop of Death** | An informal name for a stalled ReAct agent that retries the same broken action forever | Describes the most common infinite loop failure pattern |
| **Self-Correction Loop** | Feeding an error message back into the model as a prompt so it can reason about what went wrong and try a different approach | Treats errors as useful information rather than stopping conditions |
| **Error Injection** | Intercepting a runtime exception and converting it into a structured observation the model can reason about | Allows the agent to adapt its plan around failures rather than crashing |
| **Synthesized Observation** | A formatted error message constructed by the platform to be fed back to the model as if it were a tool result | Enables the model to treat errors the same way it treats any other observation |
| **Stateful Rollback** | Resetting an agent's state to a previously saved checkpoint after detecting it is stuck | Prevents a late-stage failure from invalidating all prior work |
| **Checkpoint** | A saved snapshot of the agent's state after each successful step, stored in a persistent database | Enables recovery from failures without restarting from the beginning |
| **State Snapshot** | A complete copy of the agent's memory and progress captured at a specific point in time | The artifact written at each checkpoint to enable rollback |
| **Counter-Based Intervention** | A rule that triggers when the same tool-and-arguments combination has been called N times in one session | Detects stuck loops reliably using simple counting, not complex ML |
| **Pivot Instruction** | A mandatory injected message that forces the agent to change strategy after a detected loop | Breaks the agent out of a dead-end without requiring human intervention |
| **Graceful Degradation** | Falling back to a simpler, more reliable mode of operation when the primary agent keeps failing | Ensures the user still gets some useful response even when the full agent cannot complete its task |
| **RAG-only Mode** | A fallback where the agent disables all action tools and only answers from its knowledge base | Provides a safe, reliable response when the action-taking agent is failing |
| **Output Validation Agent** | A smaller, fast model that checks whether a tool's output actually answers the intended query | Catches silent failures where a tool returns HTTP 200 but the data is wrong or irrelevant |
| **Silent Failure** | A failure where a tool returns a success code but the data it returns is incorrect or meaningless | The most dangerous failure type because it goes undetected without explicit output validation |
| **Try/Catch** | A traditional programming construct that catches exceptions when code fails | Insufficient for agentic systems because it stops execution instead of giving the agent a chance to recover |

*Next: [Human-in-the-Loop Patterns](08-human-in-the-loop-patterns.md)*
