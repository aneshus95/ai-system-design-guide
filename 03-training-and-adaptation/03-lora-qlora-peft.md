# LoRA, QLoRA, and PEFT

Parameter-Efficient Fine-Tuning (PEFT) is the industry standard for adapting LLMs. This chapter covers the mechanics and advanced variants of LoRA and other PEFT methods.

## Table of Contents

- [The PEFT Revolution](#the-peft-revolution)
- [LoRA Mechanics](#lora-mechanics)
- [QLoRA: 4-bit Fine-Tuning](#qlora)
- [Advanced Variants (DoRA, Vera, RS-LoRA)](#advanced-variants)
- [Multi-LoRA Serving (Adapters)](#multi-lora-serving)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The PEFT Revolution

Full fine-tuning of frontier models (GPT-5.5, Claude Opus 4.7, Llama 4 405B) is economically unfeasible for most enterprises. PEFT allows:
1. **Memory Efficiency**: Train 70B models on a single A100.
2. **Speed**: 2x faster training by updating <1% of weights.
3. **Modularity**: Swap "skills" (adapters) onto a shared base model without reloading weights.

---

## LoRA Mechanics

LoRA (Low-Rank Adaptation) injects trainable rank-decomposition matrices into the transformer layers.

```python
# The LoRA Equation for a Weight Matrix W:
h = Wx + (BA)x * (alpha/r)
```
- **W**: Pretrained weights (Frozen, Gradient = None)
- **A, B**: LoRA adapters (Trainable)
- **r**: Rank (e.g., 8, 16, 64)
- **alpha**: Scaling factor (typically 2 * rank)

> **Why (the rationale):** Any weight update ΔW can be approximated as a low-rank matrix product B×A when the update lives in a low-dimensional subspace — which empirically holds for most fine-tuning tasks. This lets LoRA train orders of magnitude fewer parameters than full fine-tuning while achieving comparable task adaptation quality.
> **When to use:** LoRA is the right default for any fine-tuning task where you don't need to alter the model's deep knowledge base — style, format, tone, specific skill sharpening. It is the production standard for adapting 7B–70B models on a single A100 or smaller.
> **Nuances & gotchas:** LoRA trains a tiny fraction of params so it is cheap and swappable, but may underperform full fine-tuning on very large domain shifts where the required weight update is not well-approximated by a low-rank matrix. The adapter adds inference latency only if you have to load/swap it; if merged into the base weights it adds zero latency.

### Principal Nuance: Target Modules
Historically, we only targeted query/value projections (`q_proj`, `v_proj`).
**Modern standard**: Target **all** linear layers (`q, k, v, o, gate, up, down`) for maximum stability and performance, even at lower ranks.

---

## QLoRA: 4-bit Fine-Tuning

QLoRA pushes efficiency further by quantizing the base model to 4-bit (NF4) while maintaining 16-bit gradients.

| Optimization | Method | Benefit |
|--------------|--------|---------|
| **NF4 Quantization** | Normalized Float 4 | Better info density than standard Int4 |
| **Double Quant** | Quantizing the quant constants | Saves ~0.5 GB VRAM per model |
| **Paging** | Unified Memory (Nvidia) | Prevents OOM by spilling to CPU RAM |

> **Why (the rationale):** Standard LoRA still requires the base model in full (BF16) precision, which for a 70B model is ~140 GB VRAM — requiring multiple A100s. QLoRA quantizes the frozen base to 4-bit (NF4), compressing it to ~35 GB, making 70B fine-tuning feasible on a single A100 80GB or even a pair of consumer GPUs.
> **When to use:** Use QLoRA when you need to fine-tune a large model (30B+) on limited GPU resources, or when cost is more important than maximum adapter quality. For smaller models (7B–13B) that already fit in standard LoRA, QLoRA's additional savings are less critical.
> **Nuances & gotchas:** QLoRA quantizes the base to fit a big model on one GPU at some quality and speed cost — the quantized base introduces approximation error that standard LoRA does not have. Gradient computation happens in 16-bit (dequantized) so training is slightly slower than LoRA on the same hardware. The quality gap vs. full 16-bit LoRA is small for most tasks but can be measurable on complex reasoning tasks.

---

## Advanced Variants

### 1. DoRA (Weight-Decomposed Low-Rank Adaptation)
DoRA decomposes the weight update into **Magnitude** and **Direction**.
- **Result**: Learns 2x faster than LoRA and performs closer to full fine-tuning.
- **Why it wins**: It allows the model to adjust how much it changes vs. what it's changing independently.

> **Why (the rationale):** Standard LoRA couples magnitude and direction changes in a single low-rank update, which constrains expressivity. Decoupling them (like weight normalization) lets each adapt independently, improving convergence speed and final quality — particularly at low ranks where coupling is most limiting.
> **When to use:** Prefer DoRA over standard LoRA for high-stakes domain adaptation where final quality matters and you want to stay at low-to-medium rank. It is a drop-in replacement in most PEFT frameworks with minimal overhead.
> **Nuances & gotchas:** DoRA adds a small memory overhead (the magnitude scalar per weight matrix). It does not eliminate the fundamental LoRA constraint of low-rank updates — on tasks requiring very large weight changes, both LoRA and DoRA may fall short of full fine-tuning.

### 2. Vera (Vector-based Random Aggregation)
Instead of low-rank matrices `A` and `B`, Vera uses fixed random projections with a small trainable vector.
- **Efficiency**: Reduces adapter size by **10x** compared to LoRA.
- **Use Case**: Massive-scale Multi-LoRA serving.

> **Why (the rationale):** When serving hundreds of specialized adapters simultaneously, even LoRA adapter sizes add up in VRAM. Vera's fixed random projections are not stored per-adapter — only the tiny trainable vectors are. This makes the per-adapter footprint extremely small.
> **When to use:** Vera is the right choice when adapter storage and swapping overhead at massive-scale multi-tenant serving is a bottleneck (hundreds or thousands of adapters). For most standard fine-tuning use cases, the quality tradeoff versus LoRA is not worth it.
> **Nuances & gotchas:** Using fixed random projections limits the expressivity of each adapter — quality can be lower than LoRA for complex adaptations. Vera is a storage/serving optimization, not a training quality improvement.

### 3. RS-LoRA (Rank-Stabilized LoRA)
Uses a scaling factor of `alpha / sqrt(r)`.
- **Benefit**: Allows you to increase rank (to 256+) without the model becoming unstable or requiring a lower learning rate.

> **Why (the rationale):** Standard LoRA's `alpha/r` scaling causes the update magnitude to decrease as rank increases, forcing you to retune the learning rate every time you change rank. RS-LoRA's `alpha/sqrt(r)` keeps the update magnitude more stable across ranks, freeing you to scale rank up without hyperparameter search.
> **When to use:** Use RS-LoRA when you need high-rank adapters (r=128+) for complex task adaptation and want to avoid per-rank learning rate retuning. For low ranks (r≤64) where the standard LoRA scaling works well, RS-LoRA offers little additional benefit.
> **Nuances & gotchas:** RS-LoRA is a scaling fix, not an architectural change — it does not add adapter capacity or change which weights are targeted. The improvement is purely in training stability and hyperparameter transferability across ranks.

---

## Multi-LoRA Serving (Adapters)

Production systems now serve one base model (e.g., Llama 4 70B) and dynamically swap adapters in the same batch.

```python
# vLLM/LMCache Multi-LoRA Pattern:
# Request 1 -> Base + Finance_Adapter
# Request 2 -> Base + Legal_Adapter
# Request 3 -> Base + Medical_Adapter
```
**The Tech**: **Continuous Batching + PagedAttention v3** allows serving 100+ adapters with only a 5-10% latency overhead compared to the base model.

> **Why (the rationale):** Deploying a separate fine-tuned model per use case multiplies VRAM requirements by the number of models. Multi-LoRA serving keeps one base model in VRAM and loads only the tiny adapter weights per request, enabling hundreds of specialized "models" at the cost of one.
> **When to use:** When you have multiple distinct use cases (finance, legal, medical, coding) and want to serve them all cost-effectively. Multi-LoRA becomes economically compelling once you have three or more distinct adapters that would otherwise each require a full model deployment.
> **Nuances & gotchas:** Adapter swapping within a batch can cause routing complexity and requires frameworks like vLLM with explicit multi-LoRA support. Adapter weights must fit in VRAM alongside the base model — very large adapters (high-rank, all-layer) reduce the number you can concurrently serve. Merging adapters into the base eliminates swapping overhead but loses the multi-adapter flexibility.

---

## Interview Questions

### Q: Why is the LoRA alpha parameter usually set to 2x the rank?

**Strong answer:**
The `alpha` parameter is a scaling factor for the LoRA update. When we initialize LoRA matrices, B is usually zero-initialized, and A is random. As we train, the update size depends on the rank `r`. By setting `alpha=2r` (or any constant), we ensure that if we decide to change the rank later (e.g., from 8 to 16), we don't need to retune the learning rate. The scaling factor `alpha/r` normalizes the update magnitude relative to the learning rate.

### Q: What is DoRA, and why would you use it over standard LoRA?

**Strong answer:**
DoRA (Weight-Decomposed Low-Rank Adaptation) is a 2024 technique that separates the pretrained weight updates into magnitude and direction components, similar to Weight Normalization. While standard LoRA updates magnitude and direction simultaneously, DoRA allows them to be learned independently. Empirically, DoRA shows much better convergence and higher accuracy, often matching full-parameter fine-tuning even at low ranks, making it the preferred choice for high-stakes domain adaptation.

---

## References
- Hu et al. "LoRA: Low-Rank Adaptation of Large Language Models" (2021)
- Liu et al. "DoRA: Weight-Decomposed Low-Rank Adaptation" (2024)
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **PEFT (Parameter-Efficient Fine-Tuning)** | A class of techniques that fine-tune only a tiny fraction of a model's weights rather than all of them | Enables fine-tuning of large models on limited hardware with low risk of catastrophic forgetting |
| **LoRA (Low-Rank Adaptation)** | A PEFT method that injects small, trainable rank-decomposition matrices (A and B) alongside the frozen pretrained weight matrices | Achieves adaptation quality close to full fine-tuning while updating less than 1% of parameters |
| **Rank-Decomposition** | Expressing a large weight update as the product of two smaller matrices (A and B), where the bottleneck dimension is the rank r | Reduces the number of trainable parameters from (d × d) to (d × r + r × d) |
| **Rank (r)** | The bottleneck dimension of the LoRA adapter matrices, controlling how much capacity the adapter has | Higher rank = more expressive but more memory; lower rank = cheaper but limited for complex tasks |
| **Alpha (α)** | A LoRA scaling factor applied as alpha/r to the adapter output, normalizing the magnitude of updates | Decouples the learning rate from the rank so you don't need to retune LR when changing r |
| **Target Modules** | The specific weight matrices inside the transformer (q, k, v, o, gate, up, down projections) that LoRA adapters are attached to | Targeting all linear layers rather than just q/v gives better stability and performance |
| **QLoRA** | An extension of LoRA where the base model is quantized to 4-bit (NF4) precision, with LoRA adapters remaining in 16-bit | Enables fine-tuning of 70B parameter models on a single A100 GPU |
| **NF4 (NormalFloat4)** | A 4-bit data type optimized for the normal distribution that LLM weights follow, mapping them to 16 equal-density bins | Achieves higher information density than standard Int4, minimizing accuracy loss at 4-bit precision |
| **Double Quantization** | Quantizing the constants used to dequantize NF4 weights (the quantization constants themselves) | Saves roughly 0.5 GB of VRAM per model with negligible quality impact |
| **Paging (Unified Memory)** | Using Nvidia's unified memory system to spill GPU VRAM overflow to CPU RAM seamlessly | Prevents out-of-memory crashes during fine-tuning without requiring a smaller batch size |
| **DoRA (Weight-Decomposed Low-Rank Adaptation)** | A LoRA variant that separately learns the magnitude and direction of weight updates, analogous to weight normalization | Converges faster than standard LoRA and often matches full fine-tuning quality even at low ranks |
| **Magnitude** | In DoRA, a scalar that controls how much the weight changes (the size of the update vector) | Decoupled learning of magnitude and direction allows the model to adjust each independently, improving convergence |
| **Direction** | In DoRA, the unit vector that controls which way the weight changes in parameter space | Separating direction from magnitude is what gives DoRA its convergence advantage over standard LoRA |
| **Vera (Vector-based Random Aggregation)** | A PEFT method that uses fixed random projection matrices with a small trainable scalar vector instead of full A and B matrices | Reduces adapter size by ~10x versus LoRA, making it practical for massive-scale multi-adapter serving |
| **RS-LoRA (Rank-Stabilized LoRA)** | A LoRA variant that scales the adapter output by alpha/sqrt(r) instead of alpha/r | Allows stable training at very high ranks (256+) without needing to lower the learning rate |
| **Multi-LoRA Serving** | Running a single base model in memory while dynamically swapping different LoRA adapters per request | Enables serving many specialized models (finance, legal, medical) with the memory cost of one base model |
| **Adapter** | A set of lightweight trained weights (e.g., a LoRA A+B pair) that modify a frozen base model's behavior | The modular unit that can be swapped or stacked to give the base model different skills |
| **Continuous Batching** | Dynamically adding new requests to the inference batch as others complete, rather than waiting for a full batch | Maximizes GPU utilization and throughput in production serving |
| **PagedAttention** | A memory management technique that stores KV cache in non-contiguous pages, similar to OS virtual memory | Greatly increases the number of concurrent requests a single GPU can serve |
| **Weight Normalization** | Decomposing a weight matrix into magnitude and direction for more stable optimization | The inspiration for DoRA's design; improves gradient conditioning during training |

---

*Next: [RLHF and DPO](04-rlhf-and-dpo.md)*
