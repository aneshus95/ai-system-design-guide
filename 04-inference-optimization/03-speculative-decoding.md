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

> **Why (the rationale):** The root problem is that one weight-load from VRAM serves only one output token — a terrible bytes-moved-per-token ratio. Speculative decoding amortizes one expensive target-model pass over multiple tokens, converting serial decode steps into a parallel verification pass that looks like prefill. This exploits the fact that verification is parallel and cheap compared to sequential generation.
> **When to use:** When per-request latency (TPOT) is the bottleneck and the GPU is underutilized (low batch size). Most effective when draft and target model distributions are similar — e.g., same model family, low temperature, structured/code outputs with predictable continuations. Less effective at high concurrency where the GPU is already saturated.
> **Nuances & gotchas:** Speculative decoding gives zero quality loss only when the rejection sampling algorithm is applied correctly — the target distribution is guaranteed to be preserved. At high temperature or creative tasks, the acceptance rate drops and the verification pass is wasted compute, making it slower than standard decoding. A separate draft model doubles VRAM usage and requires its own KV cache; this is why Medusa Heads are preferred for memory-constrained deployments.

---

## Draft-Verify Paradigm

1. **Drafting**: A small, fast "Draft Model" (e.g., 1B or 7B) generates $K$ candidate tokens.
2. **Verification**: The large "Target Model" processes all $K$ tokens at once.
3. **Acceptance**: The target model's logits are used to accept or reject candidates. If token $i$ is rejected, all tokens after it are discarded.

> **Why (the rationale):** A small draft model (e.g., 1B) is ~50x cheaper per token than a 70B target model, so generating K speculative tokens with it costs far less than K target-model steps. The key insight is that verifying K tokens in one parallel pass costs roughly the same as generating 1 token — so if even a few draft tokens are accepted, you win.
> **When to use:** When you have a well-matched draft model from the same family as the target (e.g., Llama 8B drafting for Llama 70B), output temperature is low, and the task is predictable (code, factual Q&A, chat templates). K=4–8 is typical; tune based on observed acceptance rate.
> **Nuances & gotchas:** If token $i$ is rejected, tokens $i+1$ through $K$ are discarded even if they would have been accepted — early rejection wastes all subsequent draft work. The draft model requires its own VRAM and KV cache, which can be 10–20% of the total memory budget. Acceptance rate varies widely: ~80%+ for structured code, ~40–60% for chat, ~20–30% for high-temperature creative writing.

| Model | Size | Speed | Latency per token |
|-------|------|-------|-------------------|
| **Draft** | 1B | Fast | 5ms |
| **Target**| 70B| Slow | 50ms |
| **Speculative**| - | **Fast**| **15ms - 25ms** |

**Net Result**: 2x to 3x speedup in wall-clock time with **zero loss in quality**.

---

## Medusa & Multi-Token Heads

The industry has moved away from separate draft models (which add VRAM overhead) toward **Medusa Heads**.

> **Why (the rationale):** Separate draft models require a second model, a second KV cache, and synchronization overhead between two inference processes. Medusa Heads attach small linear prediction layers directly to the target model's final hidden state, generating multiple future token guesses within a single forward pass — eliminating all inter-model communication and extra VRAM.
> **When to use:** When you want speculative decoding speedups but cannot afford the VRAM overhead of a second model. Medusa is the preferred default for production deployments with tight memory budgets. Requires fine-tuning the target model to add and train the extra heads.
> **Nuances & gotchas:** Medusa Heads require additional training on the base model — you cannot apply them to an arbitrary checkpoint without fine-tuning. The heads are model-specific and must be retrained when you update the base model. Acceptance rate is generally lower than a well-matched separate draft model, but the lower VRAM and latency overhead often makes it a better overall tradeoff.

- **What it is**: Extra "heads" (small linear layers) attached to the last layer of the target model.
- **How it works**: Instead of predicting just token $t+1$, Head 1 predicts $t+1$, Head 2 predicts $t+2$, and so on.
- **Benefit**: No second model needed; 2.5x speedup with minimal VRAM increase.

---

## Lookahead Decoding

An alternative that uses the model's own past hidden states to find recurring patterns (n-grams) to "look ahead" and predict future tokens.

> **Why (the rationale):** Eliminates the need for any secondary model or fine-tuning by mining the model's own recent output for n-gram patterns that are likely to repeat. Works without modifying model architecture or weights.
> **When to use:** When code completion, log generation, structured text, or any highly repetitive content is the primary workload, and you want speculation gains without model changes or extra VRAM.
> **Nuances & gotchas:** Only effective when the output contains genuinely recurring n-grams — fails on free-form creative or conversational text where patterns don't repeat. Typically yields lower speedups than draft-model speculative decoding or Medusa on general tasks. Lookahead decoding does NOT preserve the exact target distribution the same way rejection-sampling-based speculation does.

- **Best For**: Structured data, code, and highly repetitive technical writing.

---

## Hardware-Aware Speculation

Frontier serving frameworks (vLLM, TensorRT-LLM) now use **Dynamic Draft Lengths**.

> **Why (the rationale):** A fixed draft length K is suboptimal across varying GPU loads. When the GPU is lightly loaded (few requests), larger K reduces per-user latency because the verification pass amortizes over more tokens. When GPU is saturated, larger K delays other users' decode steps without proportional gain.
> **When to use:** Automatically managed by modern serving frameworks — no manual tuning needed in vLLM or TensorRT-LLM. Relevant to understand when diagnosing why speculative decoding helps some requests more than others.
> **Nuances & gotchas:** Dynamic K adjustments are heuristic and cannot predict acceptance rate per-request in advance. At high batch sizes, speculative decoding may be disabled entirely because the verification pass is no longer cheaper than regular decode — the framework should handle this gracefully.
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
