# Planning and Decomposition

Planning is the "System 2" component that allows agents to solve multi-stage problems without "wandering." Production agents have moved from simple "Chain-of-Thought" to **Recursive Decomposition** and **Tree Search**, with reasoning-native models (Claude Opus 4.7, GPT-5.5 extended thinking, DeepSeek-R2) doing the heavy planning internally.

## Table of Contents

- [The Planning Spectrum](#spectrum)
- [Static vs. Dynamic Planning](#static-vs-dynamic)
- [Chain-of-Thought (CoT) and o1 Reasoning](#cot)
- [Recursive Task Decomposition](#decomposition)
- [Tree Search (MCTS) for Agent Paths](#mcts)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Planning Spectrum

| Method | Strategy | Complexity | Best For |
|--------|----------|------------|----------|
| **Linear** | One step at a time | Low | Simple tools |
| **Branching** | If-Then-Else logic | Medium | Conditional flows |
| **Hierarchical** | Master-Plan -> Sub-Plans | High | Software engineering |
| **Search-Based** | Try multiple paths internally | Max | Scientific Research |

---

## Static vs. Dynamic Planning

### Static (Plan-and-Solve)
The agent writes a 10-step plan and follows it strictly.
- **Pros**: High performance, easy to parallelize.
- **Cons**: Brittle. If step 2 fails, steps 3-10 are useless.

> **Why (the rationale):** Writing the full plan upfront lets the model reason about dependencies and order before execution begins, enables parallel dispatch of independent steps, and prevents the model from getting distracted by noisy tool results mid-run.
> **When to use:** Use static planning when the steps are well-understood in advance and the environment is stable (e.g., generating a financial report from five known APIs, running a fixed ETL pipeline, executing a pre-defined test suite). The task must be predictable enough that a plan written at step 0 is still valid at step 8.
> **Nuances & gotchas:** Static plans are brittle — one unexpected tool result at step 2 can invalidate steps 3–10. Re-planning is more expensive than initial planning (it requires context re-evaluation of all prior steps). Static planning does NOT handle tasks where the required actions depend on what earlier observations actually return.

### Dynamic (Adaptive)
The agent writes a plan, but **Re-evaluates** after every tool call.
- **Best practice**: Use **Checkpointed Planning**. The agent is forced to "Commit" its progress to a state store after every major sub-goal to allow for recovery and "Backtracking" if the plan fails.

> **Why (the rationale):** Dynamic planning keeps the plan in sync with reality — if an early tool call returns unexpected data, the plan updates before taking the next step. This trades the parallelism advantage of a static plan for correctness in unpredictable environments.
> **When to use:** Use dynamic planning for exploratory or research tasks where early results determine what to investigate next (web research, debugging unknown systems, data exploration). Checkpointed planning is essential for any task that takes more than a few minutes — without checkpoints, any failure requires a full restart.
> **Nuances & gotchas:** Re-evaluation after every step is expensive in both tokens and latency. Without a step budget, adaptive plans can wander indefinitely. Backtracking to a checkpoint is only useful if the checkpoint state is actually durable — in-memory checkpoints are lost on any crash.

---

## CoT and o1 Reasoning

The model's internal "Thinking" window (Inference scaling) acts as a **Hidden Planner**.
- Instead of using a separate "Planner LLM," we use a reasoning model (Claude Opus 4.7, GPT-5.5 extended thinking, DeepSeek-R2) to generate a "Mental Draft."
- This draft is translated into a **Task DAG (Directed Acyclic Graph)** that the orchestrator executes.

> **Why (the rationale):** Using the model's internal chain-of-thought as the planner eliminates the need for a separate Planner LLM call — the reasoning model plans and acts in one step. Extended thinking tokens are hidden from the user but do the same work as an external plan-generation step, making the architecture simpler and cheaper.
> **When to use:** Use inference-time scaling (CoT / extended thinking) as the primary planning mechanism when using a reasoning-native model (Claude Opus 4.7, GPT-5.5 reasoning, DeepSeek-R2). It is the right default for single-agent complex tasks. Add an external explicit planner only when you need the plan to be inspectable, editable, or used to spawn parallel sub-agents.
> **Nuances & gotchas:** Extended thinking tokens count against the context window and cost more per token. The internal plan is NOT visible or auditable unless the model is configured to output it. For compliance or debugging use cases, an explicit external plan (outputted as structured text before execution) is preferable to a hidden mental draft. Reasoning models can still produce wrong plans — internal CoT reduces the rate but does not eliminate it.

---

## Recursive Task Decomposition

For massive tasks (e.g., "Build a full-stack app"), we use **Sub-Agent Spawning**.
1. **Master Agent**: Decomposes "Project" into "Frontend," "Backend," and "DB."
2. **Sub-Agents**: Each receives a "Sub-Goal" and performs its own decomposition.
3. **Consolidation**: The Master Agent merges the results.

**Critical Nuance**: Each sub-agent is given a **Minimal Context** (only what it needs) to prevent token bloat and hallucination.

> **Why (the rationale):** Some tasks are too large and complex for a single agent's context window or for a flat list of steps. Recursive decomposition maps the problem structure to an agent hierarchy — each level only needs to reason about its own sub-scope, keeping every context window small and focused.
> **When to use:** Use recursive decomposition for software engineering scale tasks (full application builds, large codebase refactors), long research projects where sub-domains are truly independent, or any task where a flat plan would exceed 20+ steps. The sub-tasks must be genuinely decomposable without hidden cross-dependencies.
> **Nuances & gotchas:** Decomposition depth must be hard-capped (typically 3 levels) to prevent recursive spawning from becoming a fork bomb. Hidden dependencies between sub-agents (e.g., the frontend needs an API contract from the backend before it can proceed) break the independence assumption and cause integration failures at consolidation. Consolidation itself is non-trivial — merging outputs from 3+ sub-agents requires the master agent to understand how each piece fits, which adds context and reasoning load at exactly the point where the context is already largest.

---

## Tree Search (MCTS)

For high-stakes decisions, we use **Monte Carlo Tree Search (MCTS)** within the agent loop.
- The agent "Simulates" 10 possible tool calls.
- A **Reward Model** (or a separate LLM prompt) scores each simulation.
- The agent follows the path with the highest reward.

> **Why (the rationale):** MCTS evaluates multiple action paths internally before committing to any real-world tool call. This reduces costly external API calls and avoids irreversible actions based on a single greedy prediction — the agent commits only to the path with the best simulated outcome.
> **When to use:** Use MCTS (or inference-time search more broadly) for high-stakes, low-frequency decisions where correctness is worth significant compute — strategic planning, scientific experiment design, security reasoning. It is not appropriate for high-frequency, latency-sensitive tasks where simulating 10 paths per step is prohibitively slow and expensive.
> **Nuances & gotchas:** MCTS requires a reward model — and the reward model itself can be wrong. A poorly calibrated reward function will confidently select the wrong path. Simulation of tool calls is inherently approximate; the simulated outcome and the real tool result may diverge significantly in dynamic environments. The compute cost scales with the branching factor and depth, so MCTS is rarely used for more than 2–3 look-ahead steps in practice.

---

## Interview Questions

### Q: How do you prevent an agent from "Infinite Recursion" during task decomposition?

**Strong answer:**
We implement **Decomposition Depth Limits** (usually 3 levels) and **Granularity Checks**. Before spawning a sub-agent, we ask the Supervisor model: "Is this task small enough to be solved by a single tool call?" If yes, we execute. If no, we decompose. We also use a **Global Controller** that tracks the total "Agent Count" to prevent a recursive bomb (fork bomb) that could drain the API budget.

### Q: Why is "Plan Revision" often more expensive than "Plan Generation"?

**Strong answer:**
Plan generation is a "Fresh Start." Plan revision requires **Context Re-evaluation**—the model must understand what was *already done*, why the *previous step failed*, and how to fix it without undoing previous successes. This requires a much higher "Reasoning Density." In production, we often use a larger model (e.g., Sonnet 3.7 or o1) for the **Revision** step, while using a smaller model for the initial plan generation.

---

## References
- Silver et al. "Mastering the game of Go with deep neural networks and tree search" (Applied to LLMs, 2024/2025)
- Wang et al. "Self-Consistency Improves Chain of Thought Reasoning" (2022/2025 update)
- LangGraph. "Multi-Agent Planning Patterns" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Planning** | The act of an agent deciding a sequence of steps to achieve a goal before or during execution | Prevents aimless tool use and reduces wasted API calls |
| **Chain-of-Thought (CoT)** | A technique where the model writes out intermediate reasoning steps before producing a final answer | Improves accuracy on multi-step tasks by making logic explicit |
| **Recursive Decomposition** | Breaking a large goal into sub-goals, and breaking those sub-goals into even smaller tasks recursively | Enables agents to tackle massive, complex projects like building full applications |
| **Task DAG (Directed Acyclic Graph)** | A graph representing task dependencies, where each node is a sub-task and edges show which tasks depend on which | Allows the orchestrator to run independent tasks in parallel and sequence dependent ones correctly |
| **Static Planning (Plan-and-Solve)** | Creating a full plan upfront and executing it in order without mid-run changes | Best for predictable workflows where all steps are known in advance |
| **Dynamic Planning (Adaptive)** | Generating a plan but re-evaluating and updating it after every tool call based on new observations | Best for unpredictable environments where early results change what needs to happen next |
| **Checkpointed Planning** | Saving the agent's progress to a state store after each major sub-goal so it can recover without restarting from scratch | Prevents total failure when a late step in a long plan encounters an error |
| **Backtracking** | Reverting to a previous safe plan state when the current path fails | Allows recovery without re-executing all prior steps |
| **Sub-Agent Spawning** | A master agent creating specialized child agents to handle individual sub-goals | Scales work across parallel specialized agents for large tasks |
| **Minimal Context** | Giving each sub-agent only the information it needs for its specific sub-goal | Reduces hallucination and token costs by avoiding irrelevant context |
| **Monte Carlo Tree Search (MCTS)** | An algorithm that simulates many possible action sequences and scores them to find the best path | Used in high-stakes agent decisions where exploring alternatives before acting is worth the compute |
| **Reward Model** | A model or scoring function that evaluates how good a simulated action sequence is | Guides MCTS to prefer paths most likely to achieve the goal |
| **Hierarchical Planning** | Organizing plans as a master plan with nested sub-plans at different levels of granularity | Allows complex software-engineering-scale tasks to be managed systematically |
| **Inference Scaling** | Using extra compute at response time (rather than at training time) to generate better, more thoughtful plans | Allows a reasoning model to act as a hidden planner without needing a separate planner component |
| **Mental Draft** | The internal plan a reasoning model generates during its extended thinking phase before outputting actions | Keeps planning lightweight by using the model's own reasoning rather than an external planner |
| **Decomposition Depth Limit** | A hard cap on how many levels of sub-task nesting an agent is allowed to create | Prevents infinite recursion and runaway API budget consumption |
| **Granularity Check** | A test the agent runs before spawning a sub-agent to determine if the task is already small enough to execute directly | Ensures sub-agent spawning only happens when genuinely needed |
| **Global Controller** | A system-level component that tracks total agent count and total cost across all sub-agents | Prevents a runaway "fork bomb" of recursively spawning agents |
| **Plan Revision** | Updating an existing plan mid-execution after a step fails, while preserving work already done | More expensive than initial planning because it requires understanding what succeeded before deciding what to change |
| **Context Re-evaluation** | The process of reviewing all prior completed steps before generating a revised plan | Required for plan revision, making it costlier than fresh plan generation |

*Next: [Error Handling and Recovery](07-error-handling-and-recovery.md)*
