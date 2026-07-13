# Pretraining Basics

Pretraining is the most computationally expensive phase of building an LLM, where a model learns general knowledge and language patterns from massive datasets.

## Table of Contents

- [The Pretraining Objective](#the-pretraining-objective)
- [Data Curriculum and Quality](#data-curriculum-and-quality)
- [Scaling Laws (Inference-Optimal)](#scaling-laws)
- [Computational Requirements](#computational-requirements)
- [Training Stability](#training-stability)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Pretraining Objective

Most modern LLMs are **Decoder-only** and use **Causal Language Modeling (CLM)**:

```python
# Objective: Minimize Cross-Entropy Loss
Loss = -sum(log P(token_i | token_1, ..., token_{i-1}))
```

The model predicts the next token given the context. This simple objective, at scale, leads to emergent reasoning capabilities.

---

## Data Curriculum and Quality

The focus has shifted from "More Data" to "Better Curriculum."

### The 100T Token Horizon
Frontier models (Llama 4, GPT-5.5, Claude Opus 4.7, Gemini 3.1 Pro) are trained on 15T to 100T tokens. At this scale, **Deduplication** and **Quality Filtering** are the primary differentiators.

### Data Mixture Standard
| Component | Percentage | Purpose |
|-----------|------------|---------|
| Web (CommonCrawl) | 50-60% | General knowledge, diverse styles |
| Code (Github, StackOverflow)| 15-20% | **Critical for Logic & Reasoning** |
| Books (Project Gutenberg) | 10% | Narrative coherence, long context |
| Academic (ArXiv, PubMed) | 10% | Specialized technical knowledge |
| Synthetic (Model-generated) | 5-10% | Math, Logic, and specific instruction paths |

**Nuance: The "Code Effect":**
Research shows that increasing code in the pretraining mix improves a model's performance on **non-coding** reasoning tasks (e.g., math, logic puzzles) by teaching structured thinking.

---

## Scaling Laws: Training vs. Inference Optimal

### The Chinchilla Paradigm (2022-2024)
`Data Tokens (D) ≈ 20 * Parameters (N)`
For a 70B model, this suggests ~1.4T tokens.

### The Inference-Optimal Paradigm
Modern models (Llama 3, Llama 4) are **heavily overtrained** relative to Chinchilla.
- **Why?**: Training cost is paid once; inference cost is paid billions of times.
- **Result**: Small models (8B) are now trained on 15T+ tokens, making them as capable as older 70B models but much cheaper to serve.

| Strategy | Token/Param Ratio | Best For |
|----------|-------------------|----------|
| Chinchilla | 20:1 | Research / Proof of Concept |
| **Inference-Optimal** | **200:1 to 500:1**| Production deployment |

---

## Training Stability

Training at the "Ultra" scale (100k+ GPUs) faces massive stability issues.

### 1. Loss Spikes
Sudden jumps in loss that can ruin a training run.
- **Standard fix**: **Periodic Checkpointing** and **Automatic Rollbacks**.
- **Architecture fix**: **Residual Scaling** (initializing weights such that the residual branch starts at near-zero).

### 2. Precision: FP8 vs BF16
- **BF16**: The 2023-2024 stability standard.
- **FP8**: The current production standard. Supported natively by H100/B200, it halves memory usage and doubles throughput while maintaining training stability through **Stochastic Rounding**.

---

## Interview Questions

### Q: Why train an 8B model on 15T tokens if Chinchilla says 160B tokens is optimal?

**Strong answer:**
Chinchilla optimality focuses on the best use of a fixed **training** compute budget. However, in production, we care about the **Total Cost of Ownership (TCO)**, which is dominated by inference. By overtraining a small model, we "bake in" more intelligence into fewer parameters. This results in a model that is significantly more efficient to serve (higher TPS, lower VRAM) while maintaining frontier-level quality.

### Q: What is the "curriculum" in LLM pretraining?

**Strong answer:**
Curriculum refers to the order and mixture of data. A common modern pattern is:
1. **General Knowledge Phase:** 80% of tokens (Web, Books).
2. **Reasoning Focus Phase:** 15% tokens (Code, Math, Logic).
3. **High-Quality "Cooling" Phase:** The last 1-5% of tokens are extremely high-quality, human-curated, or textbook data. This "cooling" phase helps the model jitter less and follow instructions better before any fine-tuning starts.

---

## References
- Kaplan et al. "Scaling Laws for Neural Language Models" (2020)
- Hoffmann et al. "Training Compute-Optimal Large Language Models" (Chinchilla, 2022)
- Meta AI. "The Llama 3/4 Herd of Models" (2024/2025)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Pretraining** | The first and most expensive training phase where a model reads massive amounts of text to learn general language patterns | Builds the foundational knowledge and language understanding before any specialization |
| **LLM (Large Language Model)** | A neural network with billions of parameters trained to predict and generate text | The core model type being trained and deployed in modern AI systems |
| **Decoder-only** | A transformer architecture that only generates text left-to-right, without a separate encoder component | The dominant architecture for modern LLMs (GPT, Llama, Claude) due to its simplicity and scalability |
| **Causal Language Modeling (CLM)** | A training objective where the model predicts each next token given only the tokens before it | Teaches the model to generate coherent, sequential text |
| **Cross-Entropy Loss** | A mathematical measure of how wrong the model's predictions are compared to the correct next token | The signal used to update model weights during training |
| **Token** | The smallest unit of text a model processes, roughly 3-4 characters or part of a word | The atomic input/output unit for all LLM computation |
| **Deduplication** | Removing near-identical or repeated documents from the training dataset | Prevents the model from over-fitting on repeated content and wastes compute |
| **Quality Filtering** | Automated or rule-based removal of low-quality text (spam, gibberish, boilerplate) from training data | Ensures the model learns from well-written, informative text rather than noise |
| **CommonCrawl** | A publicly available archive of web pages scraped from the internet | The largest single source of raw text for LLM pretraining |
| **Chinchilla Scaling Laws** | A 2022 research finding stating that model parameters and training tokens should scale together at roughly a 1:20 ratio | Defines the compute-optimal relationship between model size and training data volume |
| **Inference-Optimal Training** | Training smaller models on far more tokens than Chinchilla recommends, so they are cheaper to serve at scale | Optimizes for the total cost of running a model in production rather than training cost alone |
| **Total Cost of Ownership (TCO)** | The full cost of a model including training, serving, and infrastructure over its lifetime | The real-world metric that drives decisions to overtrain small models |
| **TPS (Tokens Per Second)** | How many output tokens a model can generate per second during inference | A key measure of inference efficiency and serving cost |
| **VRAM** | The high-speed memory on a GPU that holds model weights and activations during training and inference | The primary hardware constraint when running large models |
| **Loss Spikes** | Sudden large jumps in the training loss that can destabilize or ruin a training run | A major failure mode at large scale that requires checkpointing and rollback strategies |
| **Checkpointing** | Periodically saving the model's weights during training so you can resume from a safe state if something goes wrong | Insurance against catastrophic loss spikes or hardware failures |
| **Residual Scaling** | Initializing the residual branch weights near zero so the model starts training in a stable, near-identity configuration | Prevents instability at the start of large-scale training |
| **BF16 (BFloat16)** | A 16-bit floating-point number format with a wide dynamic range, designed for deep learning | The previous standard precision for stable LLM training |
| **FP8 (Float8)** | An 8-bit floating-point format natively supported by Nvidia H100/B200 GPUs | Cuts memory usage in half and doubles throughput versus BF16, now the production training standard |
| **Stochastic Rounding** | A technique that randomly rounds values up or down during low-precision computation to preserve statistical accuracy | Maintains training stability when using aggressive precision formats like FP8 |
| **Data Curriculum** | The ordered sequence and mixture of data types used across the phases of pretraining | Controls what the model learns and when, improving final quality beyond just adding more data |
| **Cooling Phase** | The final 1-5% of pretraining tokens, using only the highest-quality curated data | Stabilizes the model and improves instruction-following before fine-tuning begins |
| **Emergent Reasoning** | Capabilities like multi-step logic that appear in large models without being explicitly trained | A key motivation for scaling pretraining — complex skills arise from simple next-token prediction at scale |

---

*Next: [Fine-Tuning Strategies](02-fine-tuning-strategies.md)*
