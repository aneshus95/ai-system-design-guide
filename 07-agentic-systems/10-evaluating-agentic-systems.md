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

---

## Trajectory Benchmarks

Modern eval scores the **"Path to the Result."**
1. **Optimal Path**: The shortest sequence of tools to solve the task.
2. **Agent Path**: The actual steps taken.
3. **The Score**: `Efficiency = (Optimal Steps / Agent Steps)`. A score of `0.2` means the agent meandered or looped excessively.

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

### 2. Action Success Rate (ASR)
The percentage of individual tool calls that returned valid data (not errors or hallucinations).

### 3. Unit Cost per Task
Total tokens + infrastructure cost (Sandboxes, API calls) per completed goal.

---

## LLM-as-Judge for Step Quality

We use a stronger model (Claude Opus 4.7, GPT-5.5 reasoning) to review the **Reasoning Log** of a smaller agent.
- **Thought Quality**: Did the agent's logic for using Tool X follow from Observation Y?
- **Redundancy Check**: Did the agent repeat a search it just performed?
- **Feedback Loop**: This "Judge" output is then used for **DPO (Direct Preference Optimization)** to align the agent's future behavior.

---

## Production Evaluation

Production teams use **Shadow Execution**.
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
