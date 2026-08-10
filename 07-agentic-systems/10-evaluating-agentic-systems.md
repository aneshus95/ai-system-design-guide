# Evaluating Agentic Systems

Evaluating agents is fundamentally different from evaluating RAG. While RAG is about "Accuracy," Agents are about **"Reliability," "Efficiency," and "Safety."** Production agent eval relies on **Trajectory Benchmarks** and **LLM-as-Judge** for multi-step reasoning, with tools like Langfuse, LangWatch, Braintrust, and Arize Phoenix offering native trace-level scoring.

> [!NOTE]
> For standard RAG evaluation (Retrieval vs. Generation metrics), see [06-retrieval-systems/09-advanced-retrieval-patterns.md](../06-retrieval-systems/09-advanced-retrieval-patterns.md) and Section 14. This chapter focuses specifically on the *Execution Path* of an agent.

## Table of Contents

- [The Evaluation Shift](#shift)
- [Trajectory Benchmarks (The GOLD Standard)](#benchmarks)
- [Key Metrics: Success, Cost, and Duration](#metrics)
- [LLM-as-Judge for Step Quality](#judge)
- [Production Evaluation (A/B Testing Agents)](#production)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Evaluation Shift

| Metric | RAG App | Agentic App |
|--------|---------|-------------|
| **Unit of Eval** | Single Response | The **Trajectory** (All steps) |
| **Success Criteria**| Groundedness/Faithfulness | Task Completion / Logical Soundness |
| **Complexity** | Low (Text similarity) | High (Tool state validation) |

> **Why (the rationale):** An agent that reaches the right answer via a hallucinated intermediate step or by making 40 redundant tool calls is not production-reliable, even if the final text looks correct. Evaluating only the output hides the failure modes that will cause real-world problems at scale — cost overruns, latency spikes, and fragile execution paths that break on slight environment changes.
> **When to use:** Apply trajectory-based evaluation from the first prototype, not just before release — catching inefficient or brittle paths early is far cheaper than retrofitting a whole evaluation framework after discovering the problem in production.
> **Nuances & gotchas:** Trajectory evaluation requires a reference trace or an objective goal predicate to compare against, which is expensive to create for open-ended tasks; not all tasks have a single correct path, so efficiency scoring can penalize valid alternative approaches. Tool state validation (checking the external world's state, not just the text) requires test infrastructure that many teams do not have at the start.

---

## Trajectory Benchmarks

Modern eval scores the **"Path to the Result."**
1. **Optimal Path**: The shortest sequence of tools to solve the task.
2. **Agent Path**: The actual steps taken.
3. **The Score**: `Efficiency = (Optimal Steps / Agent Steps)`. A score of `0.2` means the agent meandered or looped excessively.

> **Why (the rationale):** Efficiency scoring exposes the difference between an agent that "got lucky" on a task and one that followed a reliable, reasoning-driven path. In production, cost and latency scale with steps taken — a low efficiency score is a direct predictor of high operating cost.
> **When to use:** Use trajectory benchmarks when you need to compare agent versions objectively (A/B testing), when optimizing for cost or latency, or when debugging why an agent is slow or expensive despite completing tasks; less useful for exploratory or creative tasks where path diversity is intentional.
> **Nuances & gotchas:** The "optimal path" baseline requires expert knowledge of each task — constructing it is labor-intensive and may itself be wrong or incomplete. Efficiency scores penalize valid alternative approaches that happen to take more steps; always pair with Task Success Rate to avoid optimizing for short-but-wrong paths. Public benchmarks like SWE-bench use fixed environments that may not represent your specific task distribution.

**Common Benchmarks**:
- **SWE-bench**: Fixing GitHub issues (Code Agency).
- **WebArena**: Navigating menus and forms (Browser Agency).
- **GAIA**: General tool-use tasks (Assistant Agency).

---

## Key Metrics

### 1. Task Success Rate (TSR)
The percentage of tasks where the final state is correct.
> [!IMPORTANT]
> A "Correct Answer" via a "Wrong Path" is a score of 0 in senior production settings.

> **Why (the rationale):** TSR is the single most honest signal of whether your agent is actually working from the user's perspective. Every other metric is a diagnostic that explains why TSR is what it is — TSR is the number you optimize for.
> **When to use:** TSR should be tracked for every agent in production, continuously, not just during evaluation sprints; distinguish `pass@1` (first attempt succeeds) for latency-sensitive flows from `pass@k` (at least one of k attempts succeeds) for retry-tolerant flows, as they diverge sharply at any realistic success rate.
> **Nuances & gotchas:** TSR requires a defined, verifiable "final state" — for open-ended tasks this is difficult or impossible to automate. Humans rating "correct" often disagree on edge cases, inflating TSR variance. A high TSR with low efficiency score means the agent works but burns money — both metrics are needed together.

### 2. Action Success Rate (ASR)
The percentage of individual tool calls that returned valid data (not errors or hallucinations).

> **Why (the rationale):** ASR isolates tool-level reliability from task-level success, letting you distinguish between "the agent made bad decisions" (low TSR, high ASR) and "the tools are broken or the agent calls them wrong" (low ASR directly causing low TSR). The distinction drives very different fixes.
> **When to use:** Track ASR per tool to identify which tools are the reliability bottleneck; especially useful during new tool integration when the call schema may be imperfect or when the downstream API is known to be flaky.
> **Nuances & gotchas:** ASR only measures whether the tool call succeeded — it does NOT catch silent failures where the tool returns valid-looking data that is actually wrong. A high ASR with a low TSR often signals a silent failure or a reasoning error, not a tool error.

### 3. Unit Cost per Task
Total tokens + infrastructure cost (Sandboxes, API calls) per completed goal.

> **Why (the rationale):** Per-call token cost hides the true economic picture — an agent that uses 3x the tokens but avoids 90% of human escalations can be cheaper overall. Unit cost per task is the metric that determines whether the agent is economically viable to operate at scale.
> **When to use:** Track from the first prototype to establish a cost baseline before optimizing; use to evaluate whether model routing (smaller models for simpler steps), prompt caching, or loop efficiency improvements actually reduce the real bill.
> **Nuances & gotchas:** Infrastructure cost (sandboxes, API calls to external tools) is frequently omitted and can be substantial for code-executing agents. Unit cost improves with scale through caching but worsens with scope creep as new tools and longer contexts accumulate. Cost is only meaningful relative to the value the completed task produces — a $2 task that automates a $200 human operation is economical even if it seems expensive in absolute terms.

---

## LLM-as-Judge for Step Quality

We use a stronger model (Claude Opus 4.7, GPT-5.5 reasoning) to review the **Reasoning Log** of a smaller agent.

> **Why (the rationale):** Human review of every agent step is not scalable at production volume. LLM-as-judge enables automated evaluation of reasoning quality across thousands of traces, making it practical to catch reasoning anti-patterns (redundant searches, faulty logic chains) and feed them back into training.
> **When to use:** Use when you have too many traces to review manually, when reasoning quality (not just final output) is a key quality dimension, or when building a DPO/RLHF pipeline that needs preference pairs at scale; always calibrate the judge on a human-labeled sample first to ensure its scores correlate with human judgment.
> **Nuances & gotchas:** The judge model can share biases with the agent model if they are from the same family — preference for its own style of reasoning can inflate scores. LLM judges are known to exhibit position bias (preferring the first option), verbosity bias (preferring longer responses), and self-enhancement bias; mitigate with structured rubrics and adversarial prompting. DPO training from LLM-judge pairs amplifies whatever biases the judge has, so judge calibration is not optional.

- **Thought Quality**: Did the agent's logic for using Tool X follow from Observation Y?
- **Redundancy Check**: Did the agent repeat a search it just performed?
- **Feedback Loop**: This "Judge" output is then used for **DPO (Direct Preference Optimization)** to align the agent's future behavior.

---

## Production Evaluation

Production teams use **Shadow Execution**.

> **Why (the rationale):** Offline benchmarks use curated, fixed test sets that may not represent real production traffic distribution. Shadow execution evaluates the experimental agent on real, live queries without risking user impact, giving the most ecologically valid comparison of two agent versions.
> **When to use:** Use before promoting any significant agent change to production — model swaps, tool additions, system prompt changes, loop architecture changes; too operationally expensive for continuous use on every commit, so pair with offline eval for frequent changes and shadow execution for major releases.
> **Nuances & gotchas:** Shadow execution doubles your LLM call volume (and cost) for every query during the shadow period — budget accordingly. If V2 takes non-idempotent actions in the sandbox (even a hidden one), those actions may have real effects depending on sandbox isolation; ensure sandbox tools are fully mocked or the shadow is read-only. Comparing trajectories requires automated scoring; manual comparison at production volume is not feasible.

1. **V1 Agent** responds to the user.
2. **V2 (Experimental) Agent** runs the same query in a "Hidden Sandbox."
3. **The Comparison**: We compare the two trajectories. If V2 consistently solves tasks in fewer steps without safety violations, we promote it to production.

---

## Interview Questions

### Q: How do you evaluate an agent when the environment is non-deterministic (e.g., the web)?

**Strong answer:**
We use **Mock Environments** or **Snapshotted States**. For high-fidelity testing, we use a containerized browser that resets to a clean state for every test run. We then compare the agent's trajectory against a **Reference Trace**. If the environment is truly live, we use **State-Based Verification**—instead of comparing the text, we check the external world's state (e.g., "Is there a new row in the database with the correct values?").

### Q: Why is "Meandering" (taking too many steps) a critical failure in Staff-level Agent design?

**Strong answer:**
Meandering leads to three failures: 1) **Cost**: Every step is an LLM call; 2) **Latency**: Every step adds 2-5 seconds; 3) **Entropy**: The longer the trajectory, the higher the chance of the agent encountering a weird edge case that triggers a hallucination. The standard fix is **Step Budgets**: if an agent doesn't solve a task in 10 steps, we terminate it and escalate to a human to prevent a "Token Leak."

---

## References
- Jimenez et al. "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" (2024/2025 update)
- Microsoft Research. "AgentBench: A Comprehensive Benchmark for AI Agents" (2024)
- RAGAS. "Agentic Evaluation Module" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Trajectory** | The complete, ordered sequence of thoughts, tool calls, and observations that an agent produced during a run | The unit of evaluation in agentic systems — scoring the path, not just the final answer |
| **Trajectory Benchmark** | A test suite that scores agents on the quality and efficiency of their full execution path, not only their final output | Captures failures that produce correct answers via wrong or inefficient methods |
| **Task Success Rate (TSR)** | The percentage of tasks where the agent's final state is fully correct | The primary top-level metric for agent reliability |
| **Action Success Rate (ASR)** | The percentage of individual tool calls that returned valid, non-hallucinated results | Measures step-level reliability independent of overall task completion |
| **Unit Cost per Task** | The total token and infrastructure cost (sandboxes, API calls) consumed per successfully completed goal | Enables cost-efficiency comparison across agent versions |
| **Optimal Path** | The shortest valid sequence of tool calls to solve a task | The baseline against which an agent's actual path is compared for efficiency scoring |
| **Efficiency Score** | Optimal steps divided by agent steps — lower means the agent took unnecessary detours | Quantifies how much the agent "meandered" relative to the best possible path |
| **Meandering** | Taking far more steps than necessary to complete a task, increasing cost and hallucination risk | The failure mode measured by efficiency score; leads to higher latency and token spend |
| **Step Budget** | A hard cap on the number of steps an agent is allowed before it is terminated and escalated | Prevents runaway loops from consuming unlimited compute |
| **LLM-as-Judge** | Using a stronger model to review the reasoning log of a weaker agent and score its step quality | Scalable automated evaluation of reasoning quality without human review for every run |
| **DPO (Direct Preference Optimization)** | A training technique that uses paired "good" and "bad" responses to align a model toward preferred behavior | Used to feed LLM-as-Judge evaluations back into agent training to improve future behavior |
| **SWE-bench** | A benchmark that tests agents on fixing real GitHub issues in open-source codebases | The standard benchmark for evaluating code-agency capabilities |
| **WebArena** | A benchmark testing agents on navigating real web interfaces, filling forms, and clicking through menus | The standard benchmark for evaluating browser-based agents |
| **GAIA** | A benchmark testing agents on general tool-use tasks requiring multi-step reasoning | Used to evaluate general-purpose assistant agents |
| **Shadow Execution** | Running an experimental agent on the same live queries as the production agent, in a hidden sandbox, to compare trajectories | A safe way to validate a new agent version before promoting it to production |
| **Mock Environment** | A containerized, reset-able simulation of the real environment used for deterministic agent testing | Enables reproducible evaluation in non-deterministic environments like the web |
| **Reference Trace** | A pre-recorded expert trajectory used as the gold-standard comparison for evaluating an agent's path | Provides a stable target when the real environment changes between test runs |
| **State-Based Verification** | Checking the actual state of the external world (e.g., a database row) rather than comparing text outputs | More reliable than text comparison for verifying that an agent actually completed its intended action |
| **Groundedness** | A measure of whether a model's output is supported by actual retrieved or observed evidence | A key quality metric for RAG systems, contrasted with trajectory-based metrics for agents |
| **Token Leak** | An agent that continues running indefinitely, consuming tokens and money without making progress | The failure mode that step budgets and escalation rules are designed to prevent |
| **Langfuse / LangWatch / Braintrust / Arize Phoenix** | Observability and evaluation platforms that capture agent traces and enable LLM-as-judge scoring at scale | Production tooling for continuous agent evaluation and debugging |

*Next: [Durable Execution for Long-Running Agents](11-durable-execution.md)*
