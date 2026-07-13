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

### Dynamic (Adaptive)
The agent writes a plan, but **Re-evaluates** after every tool call.
- **Best practice**: Use **Checkpointed Planning**. The agent is forced to "Commit" its progress to a state store after every major sub-goal to allow for recovery and "Backtracking" if the plan fails.

---

## CoT and o1 Reasoning

The model's internal "Thinking" window (Inference scaling) acts as a **Hidden Planner**.
- Instead of using a separate "Planner LLM," we use a reasoning model (Claude Opus 4.7, GPT-5.5 extended thinking, DeepSeek-R2) to generate a "Mental Draft."
- This draft is translated into a **Task DAG (Directed Acyclic Graph)** that the orchestrator executes.

---

## Recursive Task Decomposition

For massive tasks (e.g., "Build a full-stack app"), we use **Sub-Agent Spawning**.
1. **Master Agent**: Decomposes "Project" into "Frontend," "Backend," and "DB."
2. **Sub-Agents**: Each receives a "Sub-Goal" and performs its own decomposition.
3. **Consolidation**: The Master Agent merges the results.

**Critical Nuance**: Each sub-agent is given a **Minimal Context** (only what it needs) to prevent token bloat and hallucination.

---

## Tree Search (MCTS)

For high-stakes decisions, we use **Monte Carlo Tree Search (MCTS)** within the agent loop.
- The agent "Simulates" 10 possible tool calls.
- A **Reward Model** (or a separate LLM prompt) scores each simulation.
- The agent follows the path with the highest reward.

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
