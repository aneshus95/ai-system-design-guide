# Few-Shot and In-Context Learning (ICL)

In-Context Learning (ICL) is the ability of an LLM to learn a new task simply by seeing examples in the prompt, without any weight updates. Maximizing ICL efficiency is a key lever for prompt stability.

## Table of Contents

- [The Anatomy of a Few-Shot Example](#anatomy)
- [How many examples?](#how-many)
- [Dynamic Example Selection](#dynamic-selection)
- [The Importance of Labelling Nuance](#labelling)
- [Advanced ICL: Analogy and Retraining-lite](#advanced-icl)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Anatomy of a Few-Shot Example

A high-quality example consists of three parts:
1. **Input**: A realistic sample of potential user data.
2. **Reasoning (Optional)**: A short explanation of *why* the output is what it is.
3. **Output**: The "Gold Standar" result.

```markdown
User: "The weather is okay, but the flight was late."
Reasoning: The user is neutral about the weather but negative about the service.
Sentiment: Mixed
```

---

## How many examples?

| Model Size | Sweet Spot | Scaling Behavior |
|------------|------------|------------------|
| **Small (8B)** | 5 - 10 | Gains continue until ~20 examples. |
| **Medium (70B)**| 3 - 5 | Plateaus early; more examples increase latency. |
| **Frontier (405B)**| 1 - 2 | Highly capable; "Instruction Following" usually suffices. |

**Rule of thumb**: If you need more than 20 examples to get a stable output, your task is likely too complex for the model, or you should consider **Fine-tuning**.

---

## Dynamic Example Selection

In production RAG or Classification, don't use the same static examples for every user.
**The Dynamic Pattern:**
1. User provides a query.
2. Search a "Vector DB of Gold Examples" for the 3 most **semantically similar** cases.
3. Inject those 3 specific cases into the prompt.

**Result**: Drastically higher accuracy because the model sees "local" patterns relevant to the current user.

---

## The Importance of Labelling Nuance

Frontier models are sensitive to **Distribution Bias** in examples.
- If you provide 5 "Positive" examples and 1 "Negative," the model will bias toward "Positive."
- **Fix**: Always use **Label Balancing**. Ensure your few-shot examples roughly mirror the expected output distribution or are perfectly balanced (1:1).

---

## Advanced ICL: Analogy and "Few-Shot CoT"

**Analogy Prompting**: Instead of saying "Do X," provide an analogy. 
"Translate this code like a translator would move a poem from French to English—preserving the soul (logic) but changing the syntax."

**Few-Shot CoT**: Providing 2 examples where the reasoning is explicit. This "primes" the model's attention to focus on logic rather than just mimicking the output string.

---

## Interview Questions

### Q: Why not just provide all 50 examples we have in the prompt?

**Strong answer:**
There are three primary reasons:
1. **Context Window Latency**: Every example adds tokens, increasing the "Prefill" time and the cost per request.
2. **Attention Dilution**: Even with 128k context, models can "lose" specific constraints if buried under too much irrelevant data (the "lost-in-the-middle" effect).
3. **Overfitting**: Providing too many narrow examples can cause the model to mimic the *format* of the examples too strictly, losing its general capability to handle edge cases outside that set.

### Q: What is "Label Bias" in In-Context Learning?

**Strong answer:**
Label bias occurs when the model predicts a specific label more frequently simply because it appeared more often in the few-shot examples or because it appeared at the end of the list. The standard mitigations are:
1. Shuffling the order of examples for different requests.
2. Ensuring an equal number of positive/negative/neutral samples.
3. Using "Permutation Testing" during prompt development to ensure the model responds to the content, not the order.

---

## References
- Brown et al. "Language Models are Few-Shot Learners" (2020)
- Min et al. "Rethinking the Role of Demonstrations: What Makes In-Context Learning Work?" (2022)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **In-Context Learning (ICL)** | An LLM's ability to learn a new task from examples placed inside the prompt, without changing its weights | Enables rapid task adaptation at inference time without expensive fine-tuning |
| **Few-Shot Prompting** | Providing the model with a small number (2–20) of worked input-output pairs before the real query | Stabilizes output format and improves accuracy on tasks with specific patterns |
| **Zero-Shot Prompting** | Sending only an instruction with no examples | Lowest latency and cost; works well for capable frontier models on straightforward tasks |
| **Gold Standard Example** | A hand-verified, high-quality input-output pair used as a demonstration in the prompt | Sets the quality bar the model should match in its outputs |
| **Fine-tuning** | Updating a model's internal weights on a task-specific dataset | Used when prompt-based approaches can't achieve stable accuracy, especially for small models |
| **Dynamic Example Selection** | Choosing which few-shot examples to inject at runtime based on semantic similarity to the current query | Boosts accuracy by showing the model cases closely related to what the user asked |
| **Vector DB (Vector Database)** | A database that stores embeddings and supports similarity search to find related items | Used to retrieve the most relevant few-shot examples for a given query |
| **Semantic Similarity** | A measure of how close two pieces of text are in meaning rather than exact wording | Drives dynamic example selection so the model sees relevant demonstrations |
| **Distribution Bias** | When the label proportions in your few-shot examples skew the model's predictions toward the majority label | Must be corrected to prevent the model from ignoring minority classes |
| **Label Balancing** | Ensuring each label appears roughly the same number of times in the few-shot set | Removes distribution bias and produces fair, representative model outputs |
| **Label Bias** | The tendency for a model to favor a label because it appeared more often or last in the example list | A key pitfall in ICL that reduces the reliability of classification results |
| **Permutation Testing** | Running the same prompt with different orderings of examples to detect order-dependent behavior | Confirms the model responds to content, not just the sequence of examples |
| **Context Window** | The maximum number of tokens an LLM can process in a single call | Limits how many examples and how much data can be crammed into one prompt |
| **Attention Dilution** | The degradation of a model's focus on specific instructions when the prompt is very long | Motivates keeping prompts lean and using dynamic selection instead of dumping all examples |
| **Lost-in-the-Middle Effect** | The phenomenon where models pay less attention to information buried in the middle of a long context | Informs placement of critical examples at the start or end of the prompt |
| **Prefill Time** | The latency cost of the model processing all input tokens before generating the first output token | Longer prompts (more examples) directly increase this cost |
| **Analogy Prompting** | Framing the task as an analogy (e.g., "translate like a poet, not a dictionary") | Helps the model grasp subtle quality goals that are hard to specify literally |
| **Few-Shot CoT (Chain-of-Thought)** | Providing examples where each one includes explicit reasoning steps alongside the final answer | Primes the model to reason carefully rather than just pattern-match output strings |
| **RAG (Retrieval-Augmented Generation)** | Combining a retrieval step (finding relevant documents) with generation so the model answers with grounded facts | Allows dynamic few-shot selection from a large pool of examples without exceeding the context window |

*Next: [Chain-of-Thought](03-chain-of-thought.md)*
