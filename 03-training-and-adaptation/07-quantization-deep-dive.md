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

### 2. AWQ (Activation-aware Weight Quantization)
Instead of quantizing all weights equally, AWQ identifies the **1% of "salient" weights** that are most important for quality and keeps them in higher precision.
- **Pro**: Better accuracy than GPTQ.

### 3. FP8 (Multi-Node Standard)
Hardware-native quantization supported by Nvidia's Transformer Engine.
- **Why it wins**: It provides the speed of Int8 but with the dynamic range of Float16, making it stable for both training and inference.

---

## GGUF vs. EXL2

### GGUF (llama.cpp)
- **Deployment**: CPU + GPU offloading. 
- **Pros**: Cross-platform (Mac, Linux, Windows), single file, highly portable.
- **Cons**: Slower than pure GPU formats.

### EXL2 (ExLlamaV2)
- **Deployment**: GPU-only (Nvidia).
- **Pros**: The **fastest 4-bit format on Nvidia GPUs**. Significant performance boost over AutoGPTQ/AWQ.
- **Cons**: Inflexible (Nvidia only).

---

## KV Cache Quantization (The VRAM Saver)

In long-context RAG (1M+ tokens), the **KV Cache** often consumes more VRAM than the model weights themselves.

- **BF16 KV Cache**: 2M tokens ≈ 32GB VRAM (on 8B model).
- **FP8/Int4 KV Cache**: 2M tokens ≈ 8GB - 16GB VRAM.

**Nuance**: Modern serving frameworks (vLLM, SGLang, TensorRT-LLM) now support **Streaming Quantization** where the KV cache is compressed on-the-fly, allowing 4x higher concurrency on the same GPU.

---

## Quantization-Aware Training (QAT)

Instead of quantizing a model *after* it's trained (Post-training Quantization), QAT simulates quantization *during* the training process.
- **Result**: The model learns to compensate for the lost precision.
- **Status**: Mandatory for models smaller than 3B parameters to remain useful at 4-bit.

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
