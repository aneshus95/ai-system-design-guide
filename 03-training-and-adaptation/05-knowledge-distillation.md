# Knowledge Distillation

Knowledge distillation is the process of transferring the intelligence from a large, complex model ("Teacher") to a smaller, more efficient one ("Student"). This is the secret to the high performance of today's small open-weight models that punch well above their parameter count.

## Table of Contents

- [The Teacher-Student Paradigm](#teacher-student-paradigm)
- [How Distillation Works](#how-distillation-works)
- [Feature vs. Output Distillation](#feature-vs-output)
- [Self-Distillation from Proof (SDP)](#self-distillation-proof)
- [Quantization-Aware Distillation](#quantization-aware-distillation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Teacher-Student Paradigm

Small models (e.g., Llama 4 8B, Gemini 3.1 Flash, Claude Haiku 4.5) are not trained on raw web data alone. They are trained on **Synthetic Data** generated or curated by a much larger model (e.g., GPT-5.5, Claude Opus 4.7, or Llama 4 405B).

> **Why (the rationale):** Training a small model from scratch on raw web data forces it to spend capacity filtering noise and learning low-quality patterns. A teacher model provides a "purified curriculum" — structured, high-quality examples with implicit signal about which features matter — allowing the student to learn more efficiently than from raw data alone.
> **When to use:** Distillation is the right first move whenever you need a capable small model and have access to a strong teacher (via open weights or API). It is almost always more compute-efficient than training a small model from scratch or applying RLVR directly to a small base.
> **Nuances & gotchas:** Distillation transfers the teacher's knowledge but also its biases and failure modes. A student trained exclusively on teacher outputs may score high on common benchmarks while being narrower and more brittle than a model trained on diverse human data. License restrictions (OpenAI, Anthropic API ToS) often forbid using API outputs to train competing models — verify before building a commercial distillation pipeline.

| Model | Role | Intelligence Source |
|-------|------|---------------------|
| **Teacher** | Large (100B+ Params) | Pretraining on 50T+ tokens |
| **Student** | Small (1B - 8B Params) | Teacher's filtered logic/output |

---

## How Distillation Works

### 1. Hard Label Distillation
The student learns from the teacher's final predictions (e.g., the answer to a question).

> **Why (the rationale):** Hard labels are easy to collect (just run inference on the teacher) and work well for straightforward tasks where the correct answer is unambiguous. They are effectively standard SFT on teacher-generated data.
> **When to use:** Use hard label distillation when you only have API access to the teacher (no logits available) or when the task has clear, discrete correct answers. For complex open-ended generation, soft labels are usually superior.
> **Nuances & gotchas:** Hard labels discard the teacher's uncertainty signal. The student sees only the winner, not how close the runner-up was — which means the student does not learn the rich relational structure in the teacher's knowledge.

### 2. Soft Label Distillation (Temperature Scaling)
The student learns from the teacher's **probability distribution** (Logits). This is much richer because it tells the student not just the right answer, but which wrong answers were "almost" right.

> **Why (the rationale):** The teacher's full probability distribution over tokens contains far more information than its top-1 prediction. A high-confidence prediction tells the student one thing; a soft distribution where several tokens have non-trivial probability tells the student about the entire relational structure of the domain.
> **When to use:** Prefer soft label distillation whenever you have access to the teacher's logits (open-weight teacher). The quality improvement over hard labels is significant, especially for tasks with nuanced or ambiguous correct answers.
> **Nuances & gotchas:** Soft label distillation requires white-box access to the teacher — you cannot extract logits from a closed API. The temperature hyperparameter (T) must be tuned: too low and you recover hard labels; too high and all tokens look equally probable, erasing the signal.

```python
# Distillation Loss (KL Divergence):
Loss = KL_Div(Teacher_Logits / T, Student_Logits / T)
```
*Where T is the Temperature (typically 2.0 - 5.0).*

---

## Feature vs. Output Distillation

### Output Distillation (Standard)
Student matches the teacher's text responses.
- **Pros**: Easy to implement via API.
- **Cons**: Only learns behavioral surface patterns.

> **Why (the rationale):** Output distillation is the only option when the teacher is a closed-API model. It is conceptually simple — generate teacher responses to a large prompt set, then SFT the student on them.
> **When to use:** Use output distillation when you need to distill from a proprietary model (via API) and have no choice but to use its text outputs. Also appropriate when the task is behaviorally simple enough that surface-pattern matching suffices.
> **Nuances & gotchas:** Output distillation is subject to "linguistic mimicry" — the student learns to sound like the teacher (confident, fluent) without acquiring the same reasoning depth. Hallucination risk is high if the student over-indexes on confident-sounding teacher outputs that happen to be wrong.

### Feature/Hidden State Distillation
Student matches the inner **Hidden States** (vector representations) of the teacher.
- **Requirement**: You need access to the teacher's weights (Open Weights).
- **Pro**: The student learns the teacher's "internal conceptual map," leading to much higher reasoning depth.

> **Why (the rationale):** Text outputs are a compressed, lossy projection of the teacher's internal representations. By aligning on hidden states directly, the student learns the intermediate computational structure — not just the final answer — leading to deeper reasoning transfer.
> **When to use:** Use feature distillation when you have a strong open-weight teacher and need the best possible quality transfer to the student. It is especially valuable for reasoning and math tasks where the intermediate conceptual structure matters.
> **Nuances & gotchas:** Hidden state distillation requires the teacher and student to have compatible architectures (same layer structure or an alignment projection layer). It adds training complexity and memory overhead. It is not possible with closed API models — open-weight access is a hard requirement.

---

## Self-Distillation from Proof (SDP)

**The reasoning breakthrough.**
Models like o1, DeepSeek-R1, and Claude Opus 4.7 use SDP to improve without new human data.

1. **Generation**: The model generates 100 possible solutions to a hard math/code problem.
2. **Verification**: A rule-based system (compiler/calculator) identifies the 1 correct solution.
3. **Distillation**: The model is fine-tuned on the "Chain of Thought" (CoT) that led to that correct solution.

**Result**: The model "distills itself" by keeping only the high-quality reasoning paths.

> **Why (the rationale):** Human-labeled reasoning traces are extremely expensive and limited in supply. SDP uses an external verifier (not a human) to filter the model's own outputs, enabling unlimited high-quality training data generation for verifiable domains without human annotators.
> **When to use:** SDP is the right approach for improving reasoning in domains with verifiable ground truth — math, formal proofs, code with test suites. It does not require a separate teacher model; the model improves by learning from its own best outputs.
> **Nuances & gotchas:** SDP requires a reliable external verifier — without it, you cannot distinguish correct from incorrect reasoning paths. It also requires the model to already be capable enough to produce correct solutions some fraction of the time (if the model never gets anything right, there is nothing to distill from). It does not transfer well to subjective or open-ended tasks where "correctness" cannot be programmatically verified.

---

## Quantization-Aware Distillation

Standard quantization (e.g., 16-bit to 4-bit) causes a small drop in accuracy.
**The Fix**: Use Knowledge Distillation *during* the quantization process. The 16-bit model acts as the teacher, guiding the 4-bit model to minimize its error. This is how modern 4-bit models match 16-bit performance.

> **Why (the rationale):** Post-training quantization (PTQ) introduces approximation errors that accumulate across layers. Quantization-aware distillation gives the quantized model a correction signal from the full-precision version during the quantization process itself, allowing it to compensate for the precision loss rather than just tolerating it.
> **When to use:** Use quantization-aware distillation when PTQ produces unacceptable quality degradation, especially for models below 7B or at aggressive quantization levels (3-bit or below). For standard 4-bit quantization of 7B+ models, PTQ alone (AWQ, GPTQ) is often sufficient.
> **Nuances & gotchas:** Quantization-aware distillation is significantly more expensive than PTQ — it requires a training run, not just a one-shot calibration. The full-precision teacher must remain accessible in memory during training, adding VRAM overhead. It is not a substitute for QLoRA (which trains new adapters); it is a method to recover the base model's quality after quantization.

---

## Interview Questions

### Q: Why is a distilled 8B model better than an 8B model trained from scratch on the same tokens?

**Strong answer:**
Training from scratch (Pretraining) on raw web data is noisy; the model spends a lot of capacity learning to navigate that noise. A distilled model, however, is trained on a "purified" curriculum. The teacher model acts as a high-quality filter, providing structured logic, clear explanations, and a cleaner distribution of language. Essentially, the teacher provides "hints" through its logit distribution that tell the student exactly which features of the language are most important to learn.

### Q: What are the risks of using GPT-4o as a teacher to distill a Llama student?

**Strong answer:**
1. **Model Collapse**: If the student only sees the teacher's output, it may lose the "long tail" of creative or diverse knowledge and only learn the teacher's narrow biases.
2. **License Violations**: Most proprietary models (OpenAI, Anthropic) have clauses forbidding the use of their outputs to train "competing" models. This is a major legal risk for enterprises distilling their own models from API outputs.
3. **Linguistic Mimicry**: The student might learn to *sound* confident (like the teacher) without actually having the same level of logical depth, leading to confident but incorrect hallucinations.

---

## References
- Hinton et al. "Distilling the Knowledge in a Neural Network" (2015)
- Gou et al. "Knowledge Distillation: A Survey" (2021)
- DeepSeek. "DeepSeek-R1: Incentivizing Reasoning Capability" (2025)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Knowledge Distillation** | The process of training a small "student" model to mimic the outputs or internal representations of a large "teacher" model | Transfers intelligence from an expensive large model into a cheaper small one without full retraining |
| **Teacher Model** | The large, highly capable model whose knowledge is being transferred (e.g., GPT-5.5, Llama 4 405B) | The source of high-quality reasoning and language understanding that the student learns from |
| **Student Model** | The smaller, more efficient model being trained to approximate the teacher's behavior (e.g., Llama 4 8B) | The deployment target — a cheap model that punches above its parameter count |
| **Hard Label Distillation** | Training the student on the teacher's final predicted answers only (e.g., "Paris" is the capital) | The simplest form of distillation; easier to implement but misses richer signal |
| **Soft Label Distillation** | Training the student to match the teacher's full probability distribution over all possible answers | Richer signal than hard labels because the student learns which wrong answers were "almost" right |
| **Logits** | The raw, unnormalized scores the model assigns to each possible next token before converting to probabilities | The carrier of soft label information — a window into the teacher's uncertainty and relative preferences |
| **Temperature (T)** | A divisor applied to logits before computing probabilities; higher temperature makes the distribution smoother | In distillation, raising T amplifies the signal in soft labels about near-correct alternatives |
| **KL Divergence** | A mathematical measure of how different two probability distributions are from each other | The standard loss function for soft-label distillation — minimizing it makes the student's distribution match the teacher's |
| **Output Distillation** | Distillation where the student only learns from the teacher's text responses | Easy to implement via API; captures behavioral patterns but not internal reasoning depth |
| **Feature / Hidden State Distillation** | Distillation where the student learns to match the teacher's internal vector representations at each layer | Transfers deeper reasoning structure, but requires access to the teacher's weights (open-weight models only) |
| **Hidden States** | The intermediate vector representations computed at each transformer layer inside the model | The "internal conceptual map" of the model — richer than output text for distillation |
| **Self-Distillation from Proof (SDP)** | A technique where a model generates many candidate solutions, keeps only the verified correct ones, and fine-tunes on those | Allows a model to improve itself without new human labels by using external verifiers to filter its own outputs |
| **Chain of Thought (CoT)** | A step-by-step reasoning trace written out before the final answer | The target for SDP — the model learns the reasoning path that led to a correct answer, not just the answer |
| **Quantization-Aware Distillation** | Using a full-precision teacher to guide a model being quantized to lower precision during training | Compensates for the accuracy loss of quantization by having the teacher correct the student's quantization errors |
| **Synthetic Data** | Text or examples generated by a model rather than written by humans | The medium through which teacher knowledge is transferred to students in distillation pipelines |
| **Model Collapse** | A failure mode where repeated training on model-generated data narrows the student's distribution and reduces diversity | The primary risk of distillation — mitigated by mixing in human data and using verifiable rewards |
| **Open Weights** | A model whose trained parameters are publicly released, allowing access to hidden states and logits | Required for feature distillation — proprietary API models only support output distillation |
| **Linguistic Mimicry** | When the student learns to sound like the teacher (confident, fluent) without gaining the teacher's actual reasoning depth | A distillation failure mode that leads to fluent but hallucinating student models |
| **License Violations** | Breaking a model provider's terms of service, which often forbid using their outputs to train competing models | A major legal risk when distilling from proprietary teachers like GPT or Claude via their APIs |
| **Perplexity** | A measure of how surprised a language model is by a given text; lower means more predictable text | Used in quality filtering — too high indicates nonsense, too low indicates boring or repetitive training data |

---

*Next: [Synthetic Data Generation](06-synthetic-data-generation.md)*
