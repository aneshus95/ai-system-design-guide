# Human-in-the-Loop Patterns

No agent is 100% reliable. **Human-in-the-Loop (HITL)** is the bridge that ensures safety and accuracy in high-stakes environments. Production stacks have moved beyond "Approval Buttons" to **Co-Reasoning** and **Interrupt-Based Steering**, exposed natively in frameworks like LangGraph (interrupt+resume) and Microsoft Agent Framework.

## Table of Contents

- [The HITL Spectrum](#spectrum)
- [Interrupts and Breakpoints](#interrupts)
- [Time-Travel Debugging (State Editing)](#time-travel)
- [Co-Reasoning (Shared Scratchpads)](#co-reasoning)
- [Confidence-Based Escalation](#escalation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The HITL Spectrum

| Pattern | Agent Autonomy | Human Role | Best For |
|---------|---------------|------------|----------|
| **Human-in-command** | Low | Drives every step | High-risk Legal/Medical |
| **Human-as-filter** | Medium | Approves/Edits final output | Content Generation |
| **Human-as-backup** | High | Only intervenes on error | Customer Support |
| **Human-on-the-loop** | Max | Audits logs after completion | High-volume analysis |

---

## Interrupts and Breakpoints

Modern architectures (LangGraph, Microsoft Agent Framework) use **Deterministic Breakpoints**.

- **The Pattern**: The system is hardcoded to "Pause" before a specific sensitive tool is called (e.g., `execute_purchase` or `delete_user`).
- **The Decision**: The environment waits for a user to send an `approve` or `reject` signal.
- **State Preservation**: The agent's reasoning state is "Frozen" in the DB until the human acts.

---

## Time-Travel Debugging (State Editing)

Standard agents are "One-way." If they make a mistake in Step 3, the session is usually ruined.
- **Innovation**: **State Injection**. A human reviewer can "Go back" to the state at Step 3, edit the agent's observation or thought, and then "Resume" execution.
- **Impact**: It allows humans to "Steer" the agent off a bad path without starting from zero.

---

## Co-Reasoning (Shared Scratchpads)

Instead of the human being a "Judge," they become a **"Partner."**
- The agent shows its **Scratchpad** (Internal Thinking) to the human.
- Characterized as: *"I am planning to use Tool A because of Fact B. Does that seem right to you?"*
- **Benefit**: Catching reasoning errors *before* they translate into actions.

---

## Confidence-Based Escalation

Using models that support "Logprobs" or built-in reasoning steps, we calculate an **Uncertainty Score**.

- If the score exceeds a threshold, the agent **Automatically Pauses** and sends a notification to a human operator.
- **Example**: An agent trying to resolve a complex billing dispute realizes the user's intent is ambiguous. It stops and says: *"I'm not 100% sure how to handle this specific refund case. One moment while I get a human expert to look at this."*

---

## Interview Questions

### Q: How do you design an HITL system that doesn't "Fatigue" the human operator?

**Strong answer:**
We use **Threshold Tuning**. We don't ask for approval on every action. We only trigger HITL for: 1) High-risk "Writing" tools, 2) Low-confidence reasoning steps, or 3) Actions that violate a "Policy" set by the business. Additionally, we provide the human with a **Contextual Summary**—instead of the whole log, we show them a 1-sentence "Diff" of what the agent wants to do. This reduces the "Review cognitive load" from minutes to seconds.

### Q: What is the "Over-Reliance" risk in HITL, and how do you mitigate it?

**Strong answer:**
Over-reliance happens when humans start clicking "Approve" without reading the logs. We mitigate this with **Forced Review Checkpoints** (e.g., the human MUST edit at least one word in the proposed plan) or **Synthetic Error Injections** (intentionally showing the human a "wrong" plan 1% of the time to see if they catch it). If they pass the "Trap," they continue; if they fail, they are flagged for additional training.

---

## References
- Wu et al. "Co-reasoning: Human-AI Collaboration Patterns" (2025)
- LangChain. "Human-in-the-loop in LangGraph" (2024/2025)
- Anthropic. "Designing for Safety and Human Oversight" (2024)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **HITL (Human-in-the-Loop)** | A design pattern where a human is required to review, approve, or correct agent actions at defined points | Provides a safety net for high-stakes or uncertain agent decisions |
| **Human-in-Command** | The lowest-autonomy HITL mode where a human explicitly approves every single step | Used for legal, medical, or irreversible operations where zero automated mistakes are acceptable |
| **Human-as-Filter** | A HITL mode where the agent operates independently but a human reviews and edits the final output before it is delivered | Common for content generation where quality matters more than speed |
| **Human-as-Backup** | A HITL mode where the human only intervenes when the agent signals an error or low confidence | Used in customer support and routing where most tasks are handled automatically |
| **Human-on-the-Loop** | The highest-autonomy HITL mode where a human only audits logs after the agent has completed its work | Appropriate for high-volume, low-risk analysis tasks |
| **Deterministic Breakpoint** | A hard-coded pause in the agent's flow before a specific sensitive action is executed | Prevents dangerous or irreversible actions from being taken without explicit human approval |
| **State Preservation** | Keeping the agent's reasoning state frozen in a database while waiting for human input | Allows the agent to resume exactly where it paused after a human approves or rejects |
| **Interrupt** | An event that causes the agent to pause execution and wait for human input before continuing | The technical mechanism that enables breakpoints in frameworks like LangGraph |
| **Time-Travel Debugging** | The ability for a human to go back to a prior state in the agent's session, edit it, and resume from that point | Allows steering the agent off a wrong path without losing all prior progress |
| **State Injection** | Manually inserting or overwriting an agent's memory or observation at a specific step during debugging | Enables precise human intervention to correct a specific bad decision |
| **Co-Reasoning** | A collaborative pattern where the agent shares its internal scratchpad with the human before acting | Catches reasoning errors before they become irreversible actions |
| **Scratchpad** | The private area of the agent's context where it writes notes and reasoning not shown to the end user | Enables the human to see exactly why the agent intends to take an action |
| **Logprobs** | Token-level probability scores output by some models for each generated token | Used to calculate a confidence score and decide whether to escalate to a human |
| **Uncertainty Score** | A numerical measure of how confident the agent is in a particular decision or action | Triggers automatic escalation to a human when it exceeds a defined threshold |
| **Confidence-Based Escalation** | Automatically pausing the agent and notifying a human when the agent's confidence falls below a threshold | Ensures human review is applied where the agent is most likely to be wrong |
| **Threshold Tuning** | Adjusting the sensitivity of the escalation trigger to balance human workload against safety | Prevents operator fatigue from too many approvals while maintaining meaningful oversight |
| **Contextual Summary** | A brief, human-readable "diff" of what the agent wants to do, rather than the full log | Reduces cognitive load for human reviewers to seconds instead of minutes |
| **Operator Fatigue** | The phenomenon where humans start rubber-stamping approvals because they are overwhelmed with review requests | A major risk in HITL systems that erodes the safety benefit of human oversight |
| **Forced Review Checkpoint** | A HITL requirement that the human must actively modify or engage with the proposed plan, not just click approve | Combats operator fatigue by ensuring meaningful review rather than passive approval |
| **Synthetic Error Injection** | Intentionally presenting a wrong plan to the human 1% of the time to test whether they are reading carefully | Catches reviewers who have stopped paying attention and flags them for additional training |
| **Over-Reliance Risk** | The danger that users or operators defer too much to AI outputs and fail to catch AI mistakes | Motivates active-engagement design patterns in HITL systems |

*Next: [Agentic Security and Sandboxing](09-agentic-security-and-sandboxing.md)*
