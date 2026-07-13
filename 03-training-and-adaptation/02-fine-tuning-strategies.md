# Fine-Tuning Strategies

Fine-tuning adapts a pretrained model to specific tasks, domains, or styles. Today, fine-tuning is less about "teaching facts" and more about "teaching format and behavior."

## Table of Contents

- [When to Fine-Tune](#when-to-fine-tune)
- [Supervised Fine-Tuning (SFT)](#supervised-fine-tuning)
- [Continued Pretraining (Domain Adaptation)](#continued-pretraining)
- [Instruction Tuning](#instruction-tuning)
- [PEFT vs. Full-Parameter](#peft-vs-full-parameter)
- [Hyperparameter Tuning](#hyperparameter-tuning)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## When to Fine-Tune

Before fine-tuning, ask: **Can this be solved with Prompt Engineering or RAG?**

| Requirement | Better Solution | Why |
|-------------|-----------------|-----|
| New Facts / Knowledge | **RAG** | LLMs are bad at memorizing facts from FT; RAG is easier to update. |
| Specific Output Format | **Fine-Tuning** | Teaches the model to reliably output JSON/XML without complex prompting. |
| Tone / Persona | **Fine-Tuning** | Much more consistent than system prompts. |
| Latency Reduction | **Fine-Tuning** | Reduces the need for long few-shot prompts. |
| Private Domain Language| **Continued Pretraining** | Teaches specialized vocabulary (medical, legal, custom code). |

---

## Supervised Fine-Tuning (SFT)

The first step after pretraining. The model is trained on `(Prompt, Response)` pairs.

### The Quality Hierarchy
**1,000 "Perfect" examples beat 1,000,000 noisy examples.**
- **Golden Sets:** Hand-curated by domain experts (PhD level for technical tasks).
- **Negative Constraint Training:** Including examples of what the model **should not** do (e.g., "Don't apologize," "Don't mention you are an AI").

---

## Continued Pretraining (Domain Adaptation)

Also known as "Second-stage Pretraining."
- **How**: Train on raw text from a specific domain (e.g., all SEC filings for a finance model).
- **Objective**: Learn the statistical distribution of the domain language.
- **Nuance**: Requires a much lower learning rate (~1/10th of original) to prevent "catastrophic forgetting."

---

## PEFT vs. Full-Parameter

| Feature | Full-Parameter FT | PEFT (LoRA, QLoRA) |
|---------|-------------------|--------------------|
| GPU VRAM | Very High (Model Size * 4-12) | Low (Model Size * 1.5) |
| Speed | Base | 2x-3x Faster |
| Risk | High (Catastrophic Forgetting) | Low |
| Deployment | One model per task | One base model + multiple adapters |
| **Verdict**| Reserved for foundation training | **The Production Standard** |

---

## Hyperparameter Tuning

### 1. Learning Rate (LR)
- **SFT**: `1e-5` to `5e-5` is standard.
- **Too high**: Model "collapses" and starts repeating or speaking gibberish.

### 2. Rank (r) for LoRA
- Higher ranks (`r=64` to `r=256`) for complex reasoning tasks.
- Lower ranks (`r=8`) for simple style/tone changes.

### 3. Packaged Training (Packing)
To maximize throughput, we "pack" multiple short examples into a single 4k or 8k sequence, separated by EOS tokens.
- **Challenge**: Self-attention might leak across examples.
- **Solution**: **FlashAttention with block-masking** to prevent cross-example attention.

---

## Interview Questions

### Q: Why use Continued Pretraining instead of just putting domain data in the SFT set?

**Strong answer:**
SFT is "expensive" in terms of data creation—you need prompt/answer pairs. Continued Pretraining allows you to leverage massive amounts of raw, unlabeled domain text to teach the model's inner representations the specialized vocabulary and style. Once the model "speaks the language," you use a small SFT set to teach it the "tasks" (e.g., classification, summarization) in that language.

### Q: How do you prevent a model from "unlearning" general capabilities during fine-tuning?

**Strong answer:**
This is "Catastrophic Forgetting." Two main mitigations:
1. **Rehearsal:** Mix in 5-10% of the original pretraining data into your fine-tuning set.
2. **PEFT (LoRA):** Since we only train a small percentage of weights, the original "knowledge" remains frozen in the base model weights, significantly reducing the risk of forgetting.

---

## References
- Hu et al. "LoRA: Low-Rank Adaptation of Large Language Models" (2021)
- Ouyang et al. "Training language models to follow instructions" (InstructGPT, 2022)
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Fine-Tuning** | Further training a pretrained model on a smaller, task-specific dataset to adjust its behavior | Adapts a general model to specific tasks, formats, or domains without rebuilding from scratch |
| **Supervised Fine-Tuning (SFT)** | Training on labeled (Prompt, Response) pairs where the correct output is known | The foundational alignment step that teaches a model to respond helpfully to user instructions |
| **RAG (Retrieval-Augmented Generation)** | Fetching relevant documents at query time and including them in the prompt rather than baking facts into weights | A better alternative to fine-tuning when the goal is adding updatable factual knowledge |
| **Prompt Engineering** | Crafting input text carefully to elicit better outputs without changing model weights | The lowest-cost way to improve model behavior before resorting to fine-tuning |
| **Continued Pretraining (Domain Adaptation)** | Running the standard pretraining objective on raw domain text after initial pretraining | Teaches a model the vocabulary, style, and statistical patterns of a specialized domain like medicine or law |
| **Catastrophic Forgetting** | A model losing previously learned general skills when trained too aggressively on new data | The primary risk of fine-tuning that must be managed through low learning rates or PEFT methods |
| **PEFT (Parameter-Efficient Fine-Tuning)** | A family of techniques that update only a small fraction of model weights during fine-tuning | Dramatically reduces memory, compute, and forgetting risk compared to full fine-tuning |
| **LoRA (Low-Rank Adaptation)** | A PEFT method that adds small trainable rank-decomposition matrices alongside frozen pretrained weights | Enables fine-tuning of very large models on limited GPU hardware |
| **QLoRA** | A variant of LoRA that quantizes the base model to 4-bit precision to further reduce memory usage | Makes 70B+ model fine-tuning feasible on a single consumer or research GPU |
| **Full-Parameter Fine-Tuning** | Updating every weight in the model during training | Achieves maximum adaptation quality but requires enormous GPU resources and risks catastrophic forgetting |
| **Golden Sets** | A small number of hand-curated, expert-quality training examples | The highest-impact training data — quality beats quantity in SFT |
| **Negative Constraint Training** | Including examples of behaviors the model should avoid (e.g., never apologize unnecessarily) | Teaches the model what not to do, which prompting alone cannot reliably enforce |
| **Learning Rate (LR)** | A hyperparameter controlling how large each weight update step is during training | Too high causes instability or collapse; too low causes slow or ineffective learning |
| **Rank (r)** | In LoRA, the number of dimensions in the low-rank adapter matrices | Higher rank captures more complex adaptations but uses more memory; tuning it balances quality and cost |
| **Packing (Sequence Packing)** | Concatenating multiple short training examples into one long sequence separated by end-of-sequence tokens | Maximizes GPU utilization by avoiding wasted padding tokens in short-example batches |
| **EOS Token** | A special token marking the end of a sequence or individual example | Signals the model where one input ends and the next begins, preventing cross-example confusion |
| **FlashAttention** | An optimized attention algorithm that uses block-wise computation to save memory and increase speed | Enables longer context windows and more efficient training; block masking prevents cross-example attention leakage during packing |
| **Block-Masking** | Preventing the self-attention mechanism from attending across packed example boundaries | Ensures examples in a packed batch cannot influence each other's loss calculation |
| **Rehearsal** | Mixing a small fraction of original pretraining data into the fine-tuning set | A simple technique to reduce catastrophic forgetting by reminding the model of its general knowledge |
| **Instruction Tuning** | Fine-tuning on diverse (instruction, response) pairs covering many task types | Teaches a model to follow natural-language instructions across a broad range of user requests |

---

*Next: [LoRA, QLoRA, and PEFT](03-lora-qlora-peft.md)*
