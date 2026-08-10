# Quantization Deep Dive

Quantization is the process of reducing the precision of model weights (e.g., from 16-bit to 4-bit) to save memory and increase inference speed. This is the primary tool for deploying large models on consumer and single-GPU hardware.

## Table of Contents

- [The Precision-Performance Tradeoff](#precision-performance)
- [Quantization Methods (NF4, GPTQ, AWQ)](#methods)
- [GGUF vs. EXL2](#formats)
- [KV Cache Quantization (The VRAM Saver)](#kv-cache)
- [Quantization-Aware Fine-Tuning](#qaft)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Precision-Performance Tradeoff

Traditional models use **BF16** (16-bit). Quantization seeks to reduce this to **8-bit (FP8)**, **4-bit (Int4/NF4)**, or even **1.5-bit (BitNet)**.

| Precision | Bits | Weight size (8B Model) | Quality Loss | GPU Compatibility |
|-----------|------|------------------------|--------------|-------------------|
| **BF16** | 16 | 16 GB | 0% (Baseline) | All Modern |
| **FP8** | 8 | 8 GB | < 1% | H100 / B200 / RTX 4090 |
| **4-bit (NF4)**| 4 | 5 GB | 1-2% | All Modern |
| **2-bit** | 2 | 2.5 GB | 10-15% | Research / Specialized |

---

## Quantization Methods

### 1. NF4 (NormalFloat4)
The gold standard for fine-tuning (QLoRA). It assumes weights follow a normal distribution and maps them to a set of 16 values.

> **Why (the rationale):** Standard 4-bit integer formats place their quantization bins uniformly across the value range, which is a poor match for LLM weights that are clustered around zero in a normal distribution. NF4 spaces its 16 bins to match the normal distribution's density, preserving more information per bit.
> **When to use:** Use NF4 whenever you are running QLoRA fine-tuning. It is the right 4-bit format for training (base model quantization) because it minimizes quality loss. For pure inference deployment, AWQ or EXL2 may be faster on specific hardware.
> **Nuances & gotchas:** NF4 is optimized for the zero-centered normal distribution of pretrained weights — if weights have been significantly shifted from that distribution (e.g., heavily fine-tuned models), the benefit over standard Int4 diminishes. NF4 is a storage format; operations must dequantize to BF16 for computation, adding a small compute overhead.

### 2. AWQ (Activation-aware Weight Quantization)
Instead of quantizing all weights equally, AWQ identifies the **1% of "salient" weights** that are most important for quality and keeps them in higher precision.
- **Pro**: Better accuracy than GPTQ.

> **Why (the rationale):** All weights are not equally important — a small fraction of "salient" weights have an outsized impact on activations and output quality. Quantizing them to 4-bit causes disproportionate quality loss. AWQ protects only those critical weights at higher precision while aggressively quantizing the rest.
> **When to use:** Use AWQ for production inference deployments on Nvidia or other modern GPUs where you need the best accuracy-at-4-bit. It is the preferred inference quantization method over GPTQ when your framework supports it (vLLM, TGI, etc.).
> **Nuances & gotchas:** AWQ requires a calibration dataset to identify salient weights — the quality of calibration data affects which weights are flagged. If the calibration set is not representative of your use case, salient weights for your domain may not be protected. AWQ is also a post-training method — it does not involve any gradient updates, so it cannot recover quality lost to very aggressive quantization (2-3 bit) the way QAT can.

### 3. FP8 (Multi-Node Standard)
Hardware-native quantization supported by Nvidia's Transformer Engine.
- **Why it wins**: It provides the speed of Int8 but with the dynamic range of Float16, making it stable for both training and inference.

> **Why (the rationale):** Integer formats (Int8, Int4) have fixed exponent ranges and work well for inference but are unsuitable for training because gradient updates require dynamic range. FP8 is a floating-point format that preserves dynamic range while cutting bits in half versus BF16 — enabling it to be used natively in both training and inference workloads.
> **When to use:** Use FP8 on H100/B200 hardware for production inference and training. It is the right default for any serious large-scale deployment on modern Nvidia infrastructure. On older GPUs (A100 and below), fall back to BF16 for training and Int8/AWQ-4bit for inference.
> **Nuances & gotchas:** FP8 has two sub-formats (E4M3 and E5M2) with different precision/range tradeoffs — choosing the wrong one for a given operation causes instability. Requires Nvidia's Transformer Engine or equivalent to manage scaling correctly; naive FP8 without proper loss scaling can silently degrade quality. Not yet universally supported across all frameworks and model architectures.

---

## GGUF vs. EXL2

### GGUF (llama.cpp)
- **Deployment**: CPU + GPU offloading. 
- **Pros**: Cross-platform (Mac, Linux, Windows), single file, highly portable.
- **Cons**: Slower than pure GPU formats.

> **Why (the rationale):** Not all deployments have dedicated Nvidia GPUs. GGUF enables quantized LLM inference on consumer hardware — Mac, Windows, Linux — by supporting hybrid CPU+GPU execution where layers are split between system RAM and GPU VRAM as available.
> **When to use:** GGUF is the right choice when deploying to diverse or mixed hardware environments, for local/on-device inference, or when portability across operating systems matters more than maximum throughput. It is the standard for open-source local inference (Ollama, LM Studio, llama.cpp-based tools).
> **Nuances & gotchas:** GGUF/llama.cpp performance on CPU-only is significantly slower than GPU-accelerated alternatives — adequate for personal use but not production serving at scale. Throughput on Nvidia GPUs is lower than EXL2 or TRT-LLM even when GPU layers are enabled. The format supports a range of quantization levels (Q2 through Q8), with quality varying significantly across them.

### EXL2 (ExLlamaV2)
- **Deployment**: GPU-only (Nvidia).
- **Pros**: The **fastest 4-bit format on Nvidia GPUs**. Significant performance boost over AutoGPTQ/AWQ.
- **Cons**: Inflexible (Nvidia only).

> **Why (the rationale):** EXL2 is designed exclusively to maximize tokens-per-second on Nvidia GPUs, sacrificing portability for raw throughput. It achieves this through a custom CUDA kernel optimized specifically for 4-bit matrix multiplication at LLM inference workloads.
> **When to use:** Use EXL2 when you have a fixed Nvidia GPU inference environment and maximum throughput is the primary goal — e.g., a single-GPU inference server where latency or tokens-per-second is the binding constraint.
> **Nuances & gotchas:** EXL2 is Nvidia-only and incompatible with AMD, Mac (Metal), or CPU inference. It requires the ExLlamaV2 runtime and is not supported by mainstream serving frameworks (vLLM, TGI) in the same way GGUF or AWQ are. Quantization calibration quality matters — poorly calibrated EXL2 models can have quality regressions not seen in the perplexity numbers.

---

## KV Cache Quantization (The VRAM Saver)

In long-context RAG (1M+ tokens), the **KV Cache** often consumes more VRAM than the model weights themselves.

- **BF16 KV Cache**: 2M tokens ≈ 32GB VRAM (on 8B model).
- **FP8/Int4 KV Cache**: 2M tokens ≈ 8GB - 16GB VRAM.

**Nuance**: Modern serving frameworks (vLLM, SGLang, TensorRT-LLM) now support **Streaming Quantization** where the KV cache is compressed on-the-fly, allowing 4x higher concurrency on the same GPU.

> **Why (the rationale):** The KV cache stores computed key/value vectors for every token in the context window. For long contexts (100K–1M tokens), this dwarfs the model weights in VRAM consumption. Quantizing the KV cache to FP8 or Int4 dramatically reduces this footprint, enabling either longer contexts or higher concurrent request counts on the same hardware.
> **When to use:** Enable KV cache quantization whenever you are serving long-context requests (>32K tokens) or need to maximize concurrency on a fixed GPU. For short-context workloads, the VRAM savings are minimal and the added complexity is not worth it.
> **Nuances & gotchas:** KV cache quantization introduces approximation error that accumulates over long sequences — quality degradation is more noticeable at very long contexts (500K+ tokens) where errors compound. This is separate from weight quantization and can be enabled or disabled independently. Some attention heads are more sensitive to KV quantization than others; mixed-precision KV caches (FP8 for insensitive heads, BF16 for sensitive ones) are an emerging mitigation.

---

## Quantization-Aware Training (QAT)

Instead of quantizing a model *after* it's trained (Post-training Quantization), QAT simulates quantization *during* the training process.
- **Result**: The model learns to compensate for the lost precision.
- **Status**: Mandatory for models smaller than 3B parameters to remain useful at 4-bit.

> **Why (the rationale):** PTQ is fast but treats the already-trained model's weights as fixed and just rounds them — the model cannot adapt to the precision loss. QAT inserts fake quantization operations during training so the model's gradients flow through the quantization error, teaching the model to place weights in positions that are robust to rounding.
> **When to use:** QAT is necessary when PTQ produces unacceptable accuracy loss — primarily for models under 3B parameters at 4-bit, and for any model at 2-bit or lower. For larger models (7B+) at 4-bit, PTQ (AWQ, GPTQ) is usually sufficient and far cheaper.
> **Nuances & gotchas:** PTQ is fast but can drop accuracy significantly at aggressive quantization levels or small model sizes; QAT recovers most of that loss at the cost of a full training run. QAT adds significant training complexity and time — it is not a drop-in replacement for PTQ. The fake-quantization operations during training do not perfectly replicate hardware quantization behavior, so there can still be a small quality gap between QAT training simulation and actual deployment.

---

## Interview Questions

### Q: Why do we use NF4 instead of standard Float4 for QLoRA?

**Strong answer:**
Standard Float4 has a fixed grid that doesn't map well to the actual distribution of LLM weights, which typically follow a zero-centered normal distribution. NF4 (NormalFloat4) is a data type that is mathematically optimized so that each quantization bin contains an equal number of values from the normal distribution. This prevents "clustering" of weights and ensures that the model preserves as much information (entropy) as possible, leading to significantly higher accuracy than standard 4-bit integers.

### Q: How does AWQ differ from GPTQ?

**Strong answer:**
GPTQ is a "Layer-wise" quantization method that minimizes the mean squared error of the weights. AWQ (Activation-aware Weight Quantization) is "input-aware." It identifies which weights are the most "salient" based on the actual activation values seen during a small calibration run. By preserving only these important weights (usually 1%) in higher precision and quantizing the rest, AWQ achieves better perplexity than GPTQ, especially for smaller models or more aggressive quantization (e.g., 3-bit).

---

## References
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)
- Frantar et al. "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers" (2022)
- Lin et al. "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration" (2023)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Quantization** | Reducing the number of bits used to represent model weights (e.g., from 16-bit to 4-bit) | Cuts memory usage and increases inference speed, making large models deployable on limited hardware |
| **BF16 (BFloat16)** | A 16-bit floating-point format with a wide numerical range, designed for deep learning | The baseline precision for modern LLM training and inference; all quality comparisons start here |
| **FP8 (Float8)** | An 8-bit floating-point format natively supported on Nvidia H100/B200 and RTX 4090 GPUs | Halves memory versus BF16 while retaining dynamic range needed for both training and inference stability |
| **Int4 / 4-bit** | A 4-bit integer representation of weights, fitting roughly 4x more weights in VRAM than BF16 | The sweet spot for consumer-hardware inference — quality loss is small (1-2%) for most tasks |
| **BitNet (1.5-bit)** | An experimental architecture using only -1, 0, +1 weight values, averaging 1.5 bits per weight | Research-stage extreme quantization for edge deployment; quality loss is significant for current architectures |
| **NF4 (NormalFloat4)** | A custom 4-bit data type where the 16 quantization bins are spaced to match the normal distribution of LLM weights | Achieves significantly better accuracy than standard Int4 by preserving more information about the weight distribution |
| **AWQ (Activation-aware Weight Quantization)** | A quantization method that identifies the 1% of weights most important to output quality and preserves them in higher precision | Outperforms GPTQ in accuracy, especially for smaller models or aggressive quantization levels (3-bit) |
| **Salient Weights** | The small fraction of model weights that have an outsized impact on output quality, identified by AWQ using calibration data | The key insight of AWQ: quantizing non-salient weights is nearly lossless, so protecting only salient ones maximizes accuracy |
| **GPTQ** | A layer-wise post-training quantization method that minimizes the mean squared error of each layer's weight matrix | A widely used 4-bit quantization method; slower and slightly less accurate than AWQ but well-supported across tools |
| **Calibration Data** | A small representative dataset (typically a few hundred examples) run through the model to identify weight sensitivity | Used by AWQ and GPTQ to determine which weights matter most before quantizing the full model |
| **GGUF** | A file format used by llama.cpp for quantized models that supports CPU + GPU hybrid inference | The most portable quantization format — runs on Mac, Linux, and Windows across hardware types |
| **EXL2 (ExLlamaV2)** | A Nvidia-GPU-only quantization format that achieves the fastest 4-bit inference speed on consumer GPUs | The best choice for maximum throughput on a single Nvidia GPU; incompatible with other hardware |
| **llama.cpp** | An open-source C++ inference engine that runs quantized models efficiently on CPU and mixed CPU/GPU setups | The main runtime for GGUF models; makes LLM inference accessible without dedicated GPU hardware |
| **ExLlamaV2** | A high-performance Nvidia GPU inference engine for EXL2-quantized models | Delivers the highest tokens-per-second for 4-bit models on Nvidia hardware |
| **KV Cache** | Memory used to store the computed key and value vectors for all previous tokens in a conversation | Grows linearly with context length; in long-context RAG it can exceed model weights in VRAM usage |
| **KV Cache Quantization** | Compressing the KV cache to 8-bit or 4-bit to reduce VRAM usage during long-context inference | Enables 4x higher concurrency or 4x longer contexts on the same GPU at minimal quality cost |
| **Streaming Quantization** | Compressing the KV cache on-the-fly as it is generated, rather than storing it in full precision | Avoids pre-allocating large VRAM buffers for long contexts; supported by vLLM, SGLang, TensorRT-LLM |
| **Post-Training Quantization (PTQ)** | Quantizing a fully trained model after training has completed, without any further gradient updates | Fast and easy to apply but less accurate than QAT, especially below 4-bit |
| **Quantization-Aware Training (QAT)** | Simulating quantization noise during the training process so the model learns to tolerate it | Produces significantly better accuracy at low bit-widths; mandatory for models under 3B parameters at 4-bit |
| **Quantization Bin** | One of the discrete values a quantized weight can take; the number of bins equals 2^bits (e.g., 16 bins for 4-bit) | The resolution of quantized weights — more bins or better-placed bins means less information loss |
| **Normal Distribution** | A bell-curve probability distribution centered at zero that LLM weights closely follow | The statistical property that NF4 exploits by spacing its 16 bins to cover this distribution evenly |
| **Perplexity (in quantization)** | A measure of how accurately a quantized model predicts held-out text compared to its full-precision version | The standard benchmark for quantization quality — lower perplexity gap means less accuracy loss |
| **TensorRT-LLM** | Nvidia's optimized inference framework that compiles LLMs for maximum throughput on Nvidia GPUs | Supports FP8 and streaming KV cache quantization; used in large-scale production deployments |

---

*Next: [Training Reasoning Models: RLVR and GRPO](08-rlvr-and-reasoning-models.md)*
