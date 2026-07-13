# Speculative Decoding

Speculative decoding is a now-standard technique that allows large Models (LLMs) to generate multiple tokens per forward pass, effectively breaking the memory-bandwidth bottleneck for sequential decoding.

## Table of Contents

- [The Core Concept](#the-core-concept)
- [Draft-Verify Paradigm](#draft-verify)
- [Medusa & Multi-Token Heads](#medusa)
- [Lookahead Decoding](#lookahead-decoding)
- [Hardware-Aware Speculation](#hardware-aware)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Core Concept

LLM decoding is memory-bound: loading 140GB of weights (70B model) to produce a single 2-byte token is inefficient. 
**Speculative Decoding** uses a cheaper method to "guess" the next $N$ tokens and uses the large model to verify them all in a single parallel "Prefill-style" pass.

---

## Draft-Verify Paradigm

1. **Drafting**: A small, fast "Draft Model" (e.g., 1B or 7B) generates $K$ candidate tokens.
2. **Verification**: The large "Target Model" processes all $K$ tokens at once.
3. **Acceptance**: The target model's logits are used to accept or reject candidates. If token $i$ is rejected, all tokens after it are discarded.

| Model | Size | Speed | Latency per token |
|-------|------|-------|-------------------|
| **Draft** | 1B | Fast | 5ms |
| **Target**| 70B| Slow | 50ms |
| **Speculative**| - | **Fast**| **15ms - 25ms** |

**Net Result**: 2x to 3x speedup in wall-clock time with **zero loss in quality**.

---

## Medusa & Multi-Token Heads

The industry has moved away from separate draft models (which add VRAM overhead) toward **Medusa Heads**.

- **What it is**: Extra "heads" (small linear layers) attached to the last layer of the target model.
- **How it works**: Instead of predicting just token $t+1$, Head 1 predicts $t+1$, Head 2 predicts $t+2$, and so on.
- **Benefit**: No second model needed; 2.5x speedup with minimal VRAM increase.

---

## Lookahead Decoding

An alternative that uses the model's own past hidden states to find recurring patterns (n-grams) to "look ahead" and predict future tokens.
- **Best For**: Structured data, code, and highly repetitive technical writing.

---

## Hardware-Aware Speculation

Frontier serving frameworks (vLLM, TensorRT-LLM) now use **Dynamic Draft Lengths**.
- If the GPU is underutilized (small batch), the system increases the number of draft tokens ($K$).
- If the GPU is saturated (large batch), it decreases $K$ to prioritize throughput over individual request latency.

---

## Interview Questions

### Q: Why doesn't Speculative Decoding work well for high-temperature creative writing?

**Strong answer:**
Speculative decoding relies on the "Draft Model" being able to accurately predict what the "Target Model" would say. In high-temperature creative writing, the probability distribution is "flatter," and the model is encouraged to pick less-likely tokens. This leads to a very low **Acceptance Rate** (the draft model's guesses are frequently rejected). When a guess is rejected, the target model's parallel pass was wasted compute, and the system falls back to standard sequential decoding, adding the overhead of the draft model's latency.

### Q: How does Medusa differ from traditional Speculative Decoding?

**Strong answer:**
Traditional speculative decoding requires a separate, smaller model (the Draft Model) which takes up extra VRAM and requires its own KV cache management. Medusa, instead, adds multiple "heads" to the base model's final hidden state. Each head is trained to predict a different offset (e.g., next token, next+1, next+2). This eliminates the need for a second model and minimizes the communication overhead between steps, as all "guesses" are generated within the same base model architecture during a single forward pass.

---

## References
- Chen et al. "Accelerating Transformer Decoding via Speculative Decoding" (2023)
- Cai et al. "Medusa: Simple LLM Acceleration via Multiple Decoding Heads" (2024)
- Fu et al. "Lookahead Decoding" (2024)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Speculative Decoding** | A technique where a cheap model drafts several candidate tokens and a large model verifies them all in one parallel pass | Generates multiple tokens per memory-load, breaking the sequential decode bottleneck with zero quality loss |
| **Draft Model** | A small, fast model (e.g., 1B or 7B parameters) used to propose candidate tokens quickly | Produces guesses cheaply so the target model can verify many at once instead of generating one at a time |
| **Target Model** | The large, high-quality model that verifies the draft model's token predictions | The authoritative model whose output distribution is preserved; determines final quality |
| **Draft-Verify Paradigm** | The two-step loop of speculative decoding: draft K candidates, then verify all K in a single parallel pass | The core mechanism that turns a slow sequential process into a batch verification step |
| **Acceptance Rate** | The fraction of draft tokens the target model agrees with and keeps | The key quality metric for speculative decoding; low acceptance (e.g., creative writing) wastes the verification pass |
| **Logits** | The raw unnormalized scores a model outputs for each possible next token before converting to probabilities | Used during verification to compare the target model's predictions against the draft model's choices |
| **Forward Pass** | One full computation through all layers of a neural network for a given input | The expensive operation speculative decoding tries to amortize across multiple tokens |
| **K (draft length)** | The number of candidate tokens the draft model proposes in one speculative step | Larger K means more tokens verified per pass, but more wasted compute when acceptance rate is low |
| **Medusa Heads** | Extra small linear layers added to the top of a target model, each predicting a different future token offset | Eliminates the need for a separate draft model by embedding speculation inside the base model |
| **Medusa** | A speculative decoding variant that uses multi-head prediction within a single model | Achieves ~2.5x speedup without the VRAM cost of a second model |
| **Lookahead Decoding** | An alternative speculation method that uses the model's own past hidden states to predict n-gram continuations | Works without a draft model; best for structured or repetitive content like code and technical text |
| **N-gram** | A sequence of N consecutive tokens used as a pattern for prediction | The unit Lookahead Decoding tries to match and reuse from previous hidden states |
| **Hidden States** | The intermediate vector representations produced by each layer of a Transformer | Carry the model's internal "understanding" of the sequence; reused in Lookahead Decoding to find patterns |
| **Dynamic Draft Length** | A serving engine feature that adjusts K (draft length) based on current GPU load | Increases K when the GPU is underutilized to reduce per-user latency; decreases K under heavy load to maximize throughput |
| **Wall-Clock Time** | The actual elapsed real-world time a user waits, not just compute time | The metric speculative decoding optimizes — faster perceived response even if total FLOPs are similar |
| **VRAM Overhead** | Extra GPU memory consumed by a component beyond the base model | The main cost of a separate draft model; why Medusa Heads are preferred in production |
| **Temperature** | A sampling parameter that controls how random a model's token choices are | High temperature flattens probability distributions, causing draft tokens to be rejected more often |
| **Memory-Bandwidth Bottleneck** | The constraint that loading model weights from VRAM is slower than generating a single token | The root problem speculative decoding solves by amortizing one weight-load over multiple verified tokens |
| **vLLM** | An open-source serving system that implements continuous batching, PagedAttention, and speculative decoding | The most widely used open inference engine for production LLM serving |
| **TensorRT-LLM** | NVIDIA's high-performance inference library with custom kernels and speculative decoding support | Peak throughput on NVIDIA hardware; uses Dynamic Draft Lengths for hardware-aware speculation |

*Next: [Batching Strategies](04-batching-strategies.md)*
