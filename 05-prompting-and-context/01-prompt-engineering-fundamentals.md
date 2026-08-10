# Prompt Engineering Fundamentals

Prompt engineering is the design of inputs to steer LLM behavior. It has evolved from "trial and error" to a disciplined architectural practice, with frameworks like DSPy treating it as a compilation problem rather than a writing exercise.

## Table of Contents

- [The Core Philosophy (Intent + Constraint)](#core-philosophy)
- [The Instruction Hierarchy](#instruction-hierarchy)
- [Role Prompting](#role-prompting)
- [Instruction Clarity and Delimiters](#clarity)
- [Zero-Shot vs. Few-Shot Efficiency](#zero-vs-few)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Core Philosophy: Intent + Constraint

Effective prompting is about maximizing **Intent Disclosure** while minimizing **Output Variance**.

1. **Intent**: Precisely what the model should do.
2. **Constraint**: Exactly what the model should *avoid* (Safety, Tone, Format).

**Principle**: "Prompting is Programming in Natural Language." Treat your prompts like code (Version control, Unit tests).

> **Why (the rationale):** LLMs are generative systems with high variance — the same model will produce wildly different outputs for subtly different inputs. Explicit intent + constraint narrows the probability distribution of the model's output to the region you actually want, reducing unpredictable behavior in production.
> **When to use:** Always — this is the foundation of every prompt, not an optional layer. The intent+constraint frame is especially critical when deploying in production where you cannot inspect every output.
> **Nuances & gotchas:** "Constraint" must be *specific*. Saying "be professional" is not a constraint; "do not use bullet points, do not apologize, do not ask clarifying questions" is. Overly long constraint lists can cause the model to drop some constraints under attention pressure — prioritize the highest-stakes ones.

---

## The Instruction Hierarchy

Production systems use a tiered message structure:

| Role | Responsibility | Nuance |
|------|----------------|--------|
| **System** | High-level rules, persona, safety. | Stickiest for frontier models (H-rank). |
| **Developer** | Technical overrides (e.g., formatting). | Newer role for "un-opinionated" models. |
| **User** | The specific, dynamic query. | Susceptible to injection; must be isolated. |
| **Assistant**| History of previous turns. | Source of "recency bias." |

> **Why (the rationale):** Without a hierarchy, a user message can override safety rules or persona constraints set by the operator. The tiered structure means the model has a clear authority ranking to resolve conflicts, reducing the risk of instruction hijacking.
> **When to use:** Any multi-turn or multi-party system — chatbots with operator personas, API products where end-users shouldn't override operator policies, or agentic systems where tool results must not gain instruction authority.
> **Nuances & gotchas:** The hierarchy is enforced by RLHF training, not cryptographic guarantees — a sufficiently clever injection can still blur role boundaries. Assistant-turn messages are especially risky: injecting a fake "Assistant:" history can prime the model to continue in a compromised direction (recency bias).

---

## Role Prompting

Assigning a persona is no longer just "You are a teacher." It is a **Capabilities Anchor**.

- **Weak**: "You are a coder."
- **Strong**: "You are a Staff Software Engineer at a Tier-1 tech company specializing in high-concurrency Rust systems. You prioritize memory safety and zero-cost abstractions."

**Why it works**: It focuses the model's attention on the specific subset of its training data related to that high-level expertise, reducing irrelevant hallucinations.

> **Why (the rationale):** A vague persona leaves the model sampling from a broad distribution of "coder" behavior — ranging from beginner tutorials to expert architecture discussions. A precise, high-specificity persona narrows that distribution to the expert tier, improving output quality without any examples or extra tokens.
> **When to use:** Whenever the domain, quality bar, or stylistic register matters — code review, legal analysis, technical writing. Less necessary for frontier models on simple factual tasks where persona adds overhead without measurable gain.
> **Nuances & gotchas:** Role prompting does NOT give the model knowledge it doesn't have — it cannot make a model know a library it was never trained on. It steers *style and focus*, not capability. Overly fictional or contradictory personas (e.g., "You are a human, never an AI") can create fragile behavior under adversarial probing.

---

## Instruction Clarity and Delimiters

Current frontier models process massive contexts. Delimiters help the model distinguish between instructions and data.

```markdown
# Instructions
Analyze the following text for PII.

# Data to Analyze
--- START OF USER DATA ---
$USER_INPUT_HERE
--- END OF USER DATA ---

# Output Schema
{ "pii_found": boolean, "types": [] }
```

**Delimiters to use**: XML tags (`<context>`, `</context>`), Markdown headers (`#`), or triple quotes (`"""`).

> **Why (the rationale):** Without delimiters, the model must infer where instructions end and data begins. In long prompts, it can misread user-provided text as a directive — the root of many prompt injection vulnerabilities. Clear structural markers make the boundary unambiguous.
> **When to use:** Any prompt that mixes trusted instructions with untrusted data (user text, retrieved documents, tool outputs). Critical in RAG pipelines and agentic systems where external content is injected at runtime.
> **Nuances & gotchas:** Delimiter effectiveness depends on training — models like Claude are specifically trained to respect XML tag boundaries (H-Rank), while simpler models may still blend zones. Delimiters reduce injection risk but do not eliminate it; they must be combined with semantic defense layers for high-stakes applications.

---

## Zero-Shot vs. Few-Shot Efficiency

| Aspect | Zero-Shot | Few-Shot |
|--------|-----------|----------|
| **Latency** | Lowest (Short prompt) | Higher (Example tokens) |
| **Accuracy**| Variable | High (Format stability) |
| **Use Case**| Simple chat, Summarization | Specific formatting, Subtle logic |

**Strategy**: If the model is a "Frontier Reasoning" model (Claude Opus 4.7, GPT-5.5 with extended thinking, DeepSeek-R2), use **Zero-Shot + Clear Chain-of-Thought**. If it's a small model (8B), use **Few-Shot** to ground it.

> **Why (the rationale):** Zero-shot relies on the model's pre-trained instruction-following; few-shot additionally steers the model's output *format and style* by demonstration. Few-shot is the fastest way to lock in a consistent output structure without fine-tuning.
> **When to use:** Start with zero-shot. Add few-shot when: (1) output format is non-standard and the model keeps drifting, (2) the model is small or less instruction-tuned, or (3) a subtle classification boundary needs anchoring. Do not add examples just because more feels safer — each example eats context tokens and can bias toward the demonstrated label distribution.
> **Nuances & gotchas:** Few-shot steers format and behavior without updating weights, but it does NOT teach the model new factual knowledge. Example order matters — models show recency bias toward the last example. Using imbalanced label distributions in examples biases classification outputs. With frontier models, >5 examples rarely improves accuracy further but linearly increases cost.

---

## Interview Questions

### Q: Why do system prompts carry more weight than user prompts in modern LLMs?

**Strong answer:**
System prompts are typically prioritized by the model's architectural training (RLHF) and may be injected into a special "instruction-only" embedding space in some architectures. From a design perspective, the system prompt defines the "Constitution" of the interaction. If a user prompt contradicts a system prompt (e.g., asking for a bomb recipe), a well-aligned model is trained to prioritize the system's "Safety Constraint" over the user's "Task Intent."

### Q: What is the "Step-by-Step" prompt optimization?

**Strong answer:**
In 2022, "Think step by step" was a magic phrase to trigger Chain-of-Thought (CoT). The modern approach is **Programmatic CoT**. Instead of a vague phrase, we provide explicit reasoning milestones: "1. Identify the core problem. 2. List the constraints. 3. Propose 3 solutions. 4. Select the best one and justify." This provides a "deterministic path" for the model's internal attention, leading to much more reliable outputs for production agents.

---

## References
- OpenAI. "Prompt Engineering Guide" (2024-2025)
- Anthropic. "Claude Prompt Engineering Documentation" (2024)
- Google DeepMind. "The Power of Prompting" (2023)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Prompt Engineering** | The practice of carefully designing the text you give an LLM to steer its behavior | Improves reliability, accuracy, and safety of LLM outputs |
| **LLM (Large Language Model)** | A neural network trained on massive text corpora that generates human-like text | The core AI component being instructed via prompts |
| **Intent Disclosure** | Making the desired task crystal-clear in the prompt so the model has no ambiguity about what to do | Maximizes the chance the model does exactly what you want |
| **Output Variance** | How much the model's response fluctuates across repeated calls for the same input | Minimizing it produces predictable, stable production outputs |
| **Instruction Hierarchy** | A tiered system of roles (System → Developer → User → Assistant) that defines which instructions carry the most authority | Enforces safety and behavioral constraints even when users try to override them |
| **System Prompt** | The top-level message that sets rules, persona, and safety constraints before the conversation begins | Acts as the "constitution" the model tries to obey throughout the interaction |
| **Developer Role** | A mid-tier instruction layer for technical overrides such as output formatting | Lets engineers adjust model behavior without exposing controls to end users |
| **H-Rank** | A model training property that makes system-level instructions "stickier" than user-level ones | Prevents user messages from overriding critical safety or behavior rules |
| **Role Prompting** | Assigning the model a specific persona or expert identity (e.g., "You are a Staff Rust engineer") | Focuses the model's attention on the relevant subset of its training, reducing hallucinations |
| **Capabilities Anchor** | A detailed, high-specificity role description that locks the model into expert behavior | Produces more consistent, domain-appropriate outputs than vague role labels |
| **Delimiter** | A clear marker (XML tag, Markdown header, triple quotes) that separates instructions from raw data in the prompt | Prevents the model from confusing untrusted user data with trusted instructions |
| **PII (Personally Identifiable Information)** | Data that can identify a real person, such as names, emails, or SSNs | Important to detect and redact to protect user privacy |
| **Zero-Shot** | Prompting the model with only an instruction and no examples | Fast and low-cost; works well when the task is simple or the model is very capable |
| **Few-Shot** | Providing 2–10 worked examples inside the prompt alongside the instruction | Anchors the model to a specific output format or reasoning style |
| **Chain-of-Thought (CoT)** | Asking the model to show its intermediate reasoning steps before giving a final answer | Dramatically improves accuracy on math, logic, and multi-step problems |
| **Programmatic CoT** | Providing explicit numbered reasoning milestones instead of the vague phrase "think step by step" | Gives the model a deterministic reasoning path, improving reliability in production agents |
| **RLHF (Reinforcement Learning from Human Feedback)** | A training technique where human raters score model outputs to guide learning | Teaches the model to follow instructions and stay aligned with user and safety expectations |
| **Frontier Model** | A state-of-the-art LLM at the cutting edge of capability (e.g., Claude Opus 4.7, GPT-5.5) | Requires fewer examples and simpler prompts than smaller models |
| **DSPy** | A framework that treats prompt engineering as a compilation problem, automatically optimizing prompts | Removes the need for hand-tuning prompts when switching models or tasks |

*Next: [Few-Shot and In-Context Learning](02-few-shot-and-icl.md)*
