# Prompt Optimization (DSPy)

Prompting has moved from the "Hand-tuning" era to the "Programmatic" era. **DSPy (Declarative Self-improving Language Programs)** is the de-facto standard for building robust LLM pipelines where prompts are optimized automatically by algorithms. The 3.x line (DSPy 3.1.3 shipped Feb 5, 2026, with point releases through May 2026) introduces tighter integration with reasoning-native models and a cleaner async runtime.

## Table of Contents

- [The DSPy Philosophy: Programming vs. Prompting](#philosophy)
- [Signatures & Modules](#signatures-modules)
- [Teleprompters (Optimizers)](#optimizers)
- [The "Prompt as Weight" Analogy](#prompt-as-weight)
- [Metric-Driven Optimization](#metrics)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The DSPy Philosophy: Programming vs. Prompting

In traditional prompting, changing a model (e.g., from GPT-5.5 to Claude Sonnet 4.6 or Llama 4) requires re-writing all your prompts.
**DSPy separates Logic from Formatting.**

- **Logic**: Defined by **Modules** (e.g., ChainOfThought, ReAct).
- **Optimization**: The system automatically finds the best prompt and examples for a *specific* model to fulfill that logic.

---

## Signatures & Modules

Instead of writing a prompt, you define a **Signature**: what the input is and what the output should be.

```python
# Signature pattern
class MultiHopQA(dspy.Signature):
    """Answer questions that require multiple context retrievals."""
    context = dspy.InputField()
    question = dspy.InputField()
    answer = dspy.OutputField(desc="A concise 1-sentence answer")

# Logic is handled by a Module
qa_system = dspy.ChainOfThought(MultiHopQA)
```

---

## Teleprompters (Optimizers)

Teleprompters are algorithms that iterate on your program to improve accuracy.
1. **BootstrapFewShot**: Automatically finds high-quality examples for your prompt.
2. **MIPROv2**: A Bayesian optimizer that tries different instruction phrasings and selects the one that maximizes your score. Still the flagship optimizer in the 3.x line.

**Why it matters**: You no longer guess if "Be helpful" or "Think carefully" is better. The optimizer proves it with data.

---

## The "Prompt as Weight" Analogy

In DSPy, your prompt is like a weight in a neural network. You don't "hardcode" weights; you train them.
- If you change your model, you just **Re-compile** (re-train) your program. The optimizer will find new few-shot examples that the new model understands better.

---

## Metric-Driven Optimization

Optimization requires a **Metric** (a function that returns a score).
- **Exact Match**: `prediction.answer == target.answer`
- **LLM-as-Judge**: Use a larger model (Claude Opus 4.7, GPT-5.5 reasoning) to grade the output of a smaller model (Llama 4 8B, Claude Haiku 4.5).

---

## Interview Questions

### Q: How does DSPy solve the "fragility" of prompt engineering?

**Strong answer:**
DSPy moves the complexity of "formatting" and "grounding" away from the human and into the compiler. When we hand-write prompts, we are effectively "hard-coding" behavior that is specific to one model at one specific time (point-in-time tuning). If that model is updated or swapped, the prompt breaks. DSPy treats the prompt as a learnable parameter. By defining a clear **Signature** and a **Metric**, we allow the system to "search" for the most effective prompt through thousands of simulated iterations, making the final system much more resilient to model changes.

### Q: What is a "Teleprompter" in the context of DSPy?

**Strong answer:**
A Teleprompter is a programmatic optimizer. Its job is to take a DSPy program (which might be a complex chain of modules) and a small set of training examples, and then "compile" them into an optimized version. It does this by generating potential "thinking patterns" and examples, testing them against a metric, and selecting the most effective ones. In short, a Teleprompter is the "Gradient Descent" of the prompt engineering world.

---

## References
- Khattab et al. "DSPy: Compiling Declarative Language Models" (2023/2024)
- Stanford NLP. "DSPy Documentation and Tutorials" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **DSPy (Declarative Self-improving Language Programs)** | A framework that treats prompts as learnable parameters and optimizes them automatically with algorithms | Eliminates brittle hand-tuned prompts by making optimization data-driven and reproducible |
| **Signature** | A DSPy declaration that names the input fields and output fields of a task without specifying how to do it | Separates the "what" (task definition) from the "how" (prompt formatting), enabling model-agnostic logic |
| **Module** | A DSPy building block (e.g., ChainOfThought, ReAct) that implements a reasoning strategy for a Signature | Composable units that can be chained into complex multi-step pipelines |
| **Teleprompter / Optimizer** | A DSPy algorithm that iterates over example-prompt combinations to find the highest-scoring configuration | The automated alternative to manually guessing which prompt phrasing works best |
| **BootstrapFewShot** | A DSPy optimizer that automatically generates and selects high-quality few-shot examples for a prompt | Replaces the tedious process of hand-curating examples by searching for effective ones programmatically |
| **MIPROv2** | A Bayesian optimization algorithm in DSPy that tests different instruction phrasings and selects the best-scoring one | Flagship DSPy 3.x optimizer; proves empirically which wording maximizes metric score |
| **Bayesian Optimizer** | An optimization approach that builds a probabilistic model of the search space to choose experiments efficiently | More sample-efficient than random search, finding good prompts with fewer LLM calls |
| **Metric** | A function in DSPy that takes a prediction and a ground-truth label and returns a numeric score | The objective the optimizer maximizes; without a good metric, optimization has no signal |
| **Exact Match** | A metric that scores 1 if the predicted answer is identical to the target and 0 otherwise | Simple and fast; suitable for classification or short factual answers |
| **LLM-as-Judge** | Using a large capable model to evaluate the quality of a smaller model's output as a metric | Enables nuanced, human-aligned scoring for tasks where exact match is too strict |
| **Prompt as Weight** | The DSPy analogy that treats a prompt like a neural network weight — something to be learned, not hardcoded | Reframes prompt engineering as an optimization problem, enabling automated improvement |
| **Re-compile** | In DSPy, re-running the optimizer when the underlying model changes to find new optimal prompts | Eliminates the need to manually re-write prompts every time you swap models |
| **Programmatic Optimization** | Using algorithms (not human intuition) to search for the best prompt text and examples | Makes LLM pipelines more robust and reproducible than hand-tuned prompts |
| **Gradient Descent (analogy)** | A classic ML optimization algorithm; Teleprompters are described as "the gradient descent of prompting" | Frames prompt optimization as an iterative, signal-driven improvement process |
| **ChainOfThought (DSPy Module)** | A DSPy module that instructs the model to reason step-by-step before answering | Applies CoT reasoning to any Signature without manually writing the CoT prompt |
| **ReAct (DSPy Module)** | A DSPy module that interleaves reasoning steps with tool calls (Reason + Act) | Enables agent-style loops where the model thinks, acts, observes, and thinks again |
| **Hard-coding (prompt context)** | Writing a fixed prompt string that is specific to one model at one point in time | The fragile baseline that DSPy's compilation approach replaces |
| **Point-in-Time Tuning** | Optimizing a prompt for a model's behavior at a specific snapshot, which breaks when the model is updated | The core fragility that DSPy's learnable prompt approach overcomes |
| **Multi-Hop QA** | A question-answering task requiring the model to retrieve and chain together multiple pieces of information | A canonical use case for DSPy pipelines that combine retrieval and reasoning modules |
| **Async Runtime** | DSPy 3.x's ability to run multiple LLM calls in parallel rather than sequentially | Reduces end-to-end pipeline latency for programs with independent sub-tasks |

*Next: [Prompt Injection and Defense](08-prompt-injection-defense.md)*
