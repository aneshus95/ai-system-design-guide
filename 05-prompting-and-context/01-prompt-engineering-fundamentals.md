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

---

## The Instruction Hierarchy

Production systems use a tiered message structure:

| Role | Responsibility | Nuance |
|------|----------------|--------|
| **System** | High-level rules, persona, safety. | Stickiest for frontier models (H-rank). |
| **Developer** | Technical overrides (e.g., formatting). | Newer role for "un-opinionated" models. |
| **User** | The specific, dynamic query. | Susceptible to injection; must be isolated. |
| **Assistant**| History of previous turns. | Source of "recency bias." |

---

## Role Prompting

Assigning a persona is no longer just "You are a teacher." It is a **Capabilities Anchor**.

- **Weak**: "You are a coder."
- **Strong**: "You are a Staff Software Engineer at a Tier-1 tech company specializing in high-concurrency Rust systems. You prioritize memory safety and zero-cost abstractions."

**Why it works**: It focuses the model's attention on the specific subset of its training data related to that high-level expertise, reducing irrelevant hallucinations.

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

---

## Zero-Shot vs. Few-Shot Efficiency

| Aspect | Zero-Shot | Few-Shot |
|--------|-----------|----------|
| **Latency** | Lowest (Short prompt) | Higher (Example tokens) |
| **Accuracy**| Variable | High (Format stability) |
| **Use Case**| Simple chat, Summarization | Specific formatting, Subtle logic |

**Strategy**: If the model is a "Frontier Reasoning" model (Claude Opus 4.7, GPT-5.5 with extended thinking, DeepSeek-R2), use **Zero-Shot + Clear Chain-of-Thought**. If it's a small model (8B), use **Few-Shot** to ground it.

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
