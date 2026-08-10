# KV Cache and Context Caching

The KV Cache is the most significant memory consumer in long-context AI systems. Managing this cache effectively is the difference between a system that scales to 2M tokens and one that crashes at 10k.

## Table of Contents

- [The KV Cache Problem](#kv-cache-problem)
- [GQA: Grouped Query Attention](#gqa)
- [Context Caching (Self-hosted)](#context-caching-self-hosted)
- [API-level Context Caching (Prompt Caching)](#api-prompt-caching)
- [RAD-O: Retrieval Augmented Decoding](#rad-o)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The KV Cache Problem

During generation, the model needs the Key (K) and Value (V) tensors for all previous tokens. Storing these in memory is expensive.

> **Why (the rationale):** Without a KV cache, each decode step would recompute attention over the entire prompt and all prior tokens from scratch — O(N²) recomputation per token. The cache trades VRAM for speed by storing these tensors once and reusing them on every subsequent step.
> **When to use:** Always enabled in modern serving stacks. Manage it carefully when context windows are long (>32k tokens) or concurrency is high — the cache dominates VRAM at those scales and directly caps batch size.
> **Nuances & gotchas:** KV cache size grows linearly with sequence length AND with batch size — it's not a fixed cost. At 128k context, the KV cache for a single user on Llama 4 70B (~42 GB) can rival the model weights themselves. It does NOT cache query tensors (queries are always recomputed for the current token). The KV cache is the primary reason long-context serving is hard and why PagedAttention, GQA, and quantized KV cache exist.

**VRAM Calculation (Llama 4 70B):**
- **Tokens**: 128,000
- **Precision**: BF16 (2 bytes/param)
- **Memory**: `2 (KV) * layers (80) * context (128k) * heads (8) * head_dim (128) * 2 bytes`
- **Total**: **~42 GB per user** in 128k context.

#### In plain English

**What is actually being stored.** For every token the model has already seen, each attention layer keeps two vectors: the **Key** ("what this token advertises") and the **Value** ("what this token contributes"). When the model generates the next token, its Query compares against *every* stored Key and pulls a blend of the matching Values — so it must have K and V for the entire history on hand. Queries are not cached: only the newest token needs one. (See the plain-English KV cache walkthrough in [01-foundations/01-llm-internals.md](../01-foundations/01-llm-internals.md) for why K/V are reusable but Q is always fresh.)

**Why "expensive" — it grows, and it lives in the scarcest place.** Two things make the cache the #1 memory headache:
1. **It grows linearly with every token generated.** The cache for token 100,000 must hold the K,V of all 99,999 predecessors, so a long conversation costs far more memory than a short one. Cache size is proportional to `sequence length × batch size`.
2. **It sits in GPU VRAM, right next to the model weights.** VRAM is fixed and small. The weights take a constant chunk; whatever is left is split among the KV caches of *every request in flight at the same time*.

**The real consequence — the KV cache, not the model, caps concurrency.** Because the leftover VRAM is shared across all concurrent requests, the cache directly limits how many users you can batch together — and batch size is what drives throughput (and therefore cost per query). Run out of KV space and you must reject requests, evict caches, or truncate context. The ~42 GB above is the cache for *one* user at 128k context — often a large fraction of an entire GPU board — so only a handful of long-context users fit alongside the weights before VRAM is gone. That is exactly why long-context serving is hard and why providers meter context length. ([KV cache memory arithmetic](https://pub.towardsai.net/llama-2-70b-has-64-query-heads-and-8-kv-heads-here-is-the-memory-arithmetic-nobody-shows-you-eb154f2b65e9))

```
GPU VRAM (fixed, e.g. 80 GB)
+----------------------------+---------+---------+---------+--------+
|    model weights (const)   | KV req1 | KV req2 | KV req3 |  free  |
+----------------------------+---------+---------+---------+--------+
                                  |         |         |
        each KV block grows +1 entry for every token generated  -->
   more users / longer context  ->  free space vanishes
                                 ->  reject requests, evict, or truncate context
```

**The formula is a menu of levers:**

```
KV cache = 2 (K and V) × layers × kv_heads × head_dim × sequence_length × bytes_per_value
```

Every term is something you can attack, and the rest of this module does exactly that: shrink `kv_heads` (**GQA/MQA**, next section), shrink `bytes_per_value` (**quantized KV cache**), and allocate `sequence_length × batch` smartly instead of contiguously (**PagedAttention**).

---

## GQA: Grouped Query Attention

GQA is the modern standard for reducing KV Cache size without losing performance.

> **Why (the rationale):** In Multi-Head Attention (MHA), each query head has its own dedicated KV head, requiring 64–128 KV heads per layer in large models. GQA reduces KV heads while keeping full query head count, shrinking cache size by 8x at a quality cost of under 0.2% — a very favorable tradeoff vs. the VRAM savings.
> **When to use:** Standard for any production model trained or fine-tuned after 2023 (Llama 3+, Mistral, Qwen, Gemma). Prefer GQA over MQA when quality is important; use MQA only in extreme memory-constrained edge/mobile scenarios where you can tolerate 2–3% quality degradation.
> **Nuances & gotchas:** GQA is a model architecture choice made at training time — you cannot apply it to an existing MHA model at inference without retraining. The 8x reduction applies to KV head count, not sequence length — long contexts still grow the cache linearly. GQA does not eliminate KV cache; it reduces the per-layer memory by the group ratio.

| Method | Ratio | KV Cache Reduction | Quality Loss |
|--------|-------|-------------------|--------------|
| **Multi-Head (MHA)** | 1:1 | 1x (Baseline) | 0% |
| **Grouped Query (GQA)** | 8:1 | **8x** | < 0.2% |
| **Multi-Query (MQA)** | All:1 | 64x-128x | 2-3% |

**Nuance**: GQA allows the model to attend to the same KV "memory" from multiple "reasoning" heads, drastically reducing the memory bandwidth needed during the Decode phase.

---

## Context Caching (Self-hosted)

Production systems use **Shared KV Caches** for prompts with common prefixes (e.g., a 100-page knowledge base shared by 1,000 users).

> **Why (the rationale):** When thousands of users share the same system prompt or knowledge base, recomputing the full prefill for each request wastes both GPU compute and latency. Storing the shared KV cache once and reusing it eliminates redundant prefill computation.
> **When to use:** When a common prefix (system prompt, document, tool schema) is shared across many requests and is long enough to make recomputation costly (typically >2k tokens). Use tiered caching (VRAM → HBM → SSD) when the shared corpus is larger than available VRAM.
> **Nuances & gotchas:** Cache hits require routing the same user (or same prefix) to the same GPU node that holds the cached KV blocks — sticky sessions at the gateway are required. Cache invalidation is hard: a single token change in the prefix breaks the cache for all downstream users. Disk-tier cache access (SGLang's SSD tier) adds latency that must be measured against the compute saving.

### Disk vs. VRAM Caching
- **VRAM Cache**: Instant access, strictly limited size.
- **Disk/SSD Cache**: Slower access, nearly unlimited. Frameworks like **SGLang** use a tiered system: `Most Recent (VRAM) -> Frequent (HBM) -> Occasional (SSD)`.

---

## API-level Context Caching (Prompt Caching)

Major providers (OpenAI, Anthropic, Google, DeepSeek) now offer **Prompt Caching** discounts.

> **Why (the rationale):** Providers pre-compute and store the KV cache for repeated prompt prefixes on their infrastructure, charging a lower rate for cache hits than for fresh computation. This transfers the caching benefit to API users without them managing any GPU state.
> **When to use:** When your system prompt, tool schema, or shared document is long (>1k tokens for Anthropic, varies by provider) and the same prefix is sent in many requests. At ≥1.1–1.5x reuse, caching is cheaper; at <1x reuse, the write premium makes it more expensive.
> **Nuances & gotchas:** Anthropic charges a 25% write premium on the first cache write — break-even is ~3–5x reuse for short prefixes. Cache TTL varies by provider (Anthropic: ~5 minutes unless refreshed; Google: billed hourly storage). The prefix must be an exact byte-for-byte match — any change, even a space, misses the cache. This does NOT provide semantic caching for different phrasings of the same question.

| Provider | Feature Name | Pricing (Cached input) | Best For |
|----------|--------------|------------------------|----------|
| **Anthropic** | Context Caching | 90% discount (Sonnet 4.6 cached: $0.30/1M) | Long system prompts, tool schemas |
| **OpenAI** | Prompt Caching | ~50% discount on cached input (GPT-5.5 cached: ~$2.50/1M) | Multi-turn chat |
| **Google** | Context Caching | Cache reads $0.20/1M (Gemini 3.1 Pro under 200K); hourly storage fee separate | Long shared corpora |
| **DeepSeek** | Context Caching | **$0.003625/M (V4 Pro) / $0.0028/M (V4 Flash)** | Massive codebase RAG; cheapest cache tier on the market |

**Break-even nuance**: If your cached prefix is reused more than **1.1x to 1.5x**, it is cheaper to use caching than raw tokens. Anthropic charges a 25% premium on cache writes, so for short prefixes the break-even is higher (3-5x reuse). DeepSeek cut the cache-hit price to 1/10 of launch on April 26, 2026. For cache-heavy workloads, V4 Flash now lands roughly 30-50x cheaper per cached token than GPT-5.5.

---

## RAD-O: Retrieval Augmented Decoding

RAD-O is a context-caching technique where the model **compresses** the KV cache of long documents into "Latent tokens."

> **Why (the rationale):** At very long contexts (1M+ tokens) the raw KV cache is prohibitively large — storing it in full VRAM is infeasible. RAD-O compresses the KV representations of a document into a smaller latent representation, allowing much longer effective contexts on the same hardware.
> **When to use:** When you need to serve 500k–2M token contexts on hardware with limited VRAM, and you can tolerate some quality loss from compression. Not needed for standard <128k context workloads where the full KV cache fits.
> **Nuances & gotchas:** Compression introduces approximation — latent tokens are lossy representations of the original document, so fine-grained factual recall may degrade. The 10x compression ratio is approximate and model/document dependent. RAD-O is a research-stage technique; production deployments are limited as of 2026.
- **How**: Instead of storing the full KV vectors for 1M tokens, it stores a compressed representation that is 10x smaller.
- **Impact**: Enables 2M+ token contexts on hardware that previously only supported 200k.

---

## Interview Questions

### Q: How does PagedAttention help with KV Cache management? (Simplified)

**Strong answer:**
Standard KV caches require contiguous memory allocation (one giant block of RAM). This leads to **External Fragmentation** (memory exists but is in unusable gaps). PagedAttention (used in vLLM) breaks the KV cache into small, fixed-size "pages" (like OS virtual memory). This allows the cache to be non-contiguous, meaning we can allocate memory exactly when needed and share pages between different requests that have the same prefix. This typically increases memory efficiency from 60% to 96%+.

### Q: Why is Context Caching better than RAG for a 50k token document?

**Strong answer:**
With cheap context caching (DeepSeek, Gemini, Anthropic), RAG is often "overkill" for medium-sized documents.
1. **Recall**: Context caching gives 100% recall (the whole doc is in the window), whereas RAG depends on retrieval accuracy.
2. **Coherence**: The model can see cross-references across the whole document.
3. **Economics**: At 50k tokens, the cost of a cached input is often lower than the complexity of maintaining a vector database and retrieval pipeline.

---

## References
- Kwon et al. "Efficient Memory Management with PagedAttention" (2023)
- Anthropic. "Prompt Caching Documentation" (2024)
- DeepSeek. "Context Caching Technical Report" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **KV Cache** | Memory that stores the Key and Value attention tensors for every token already seen in a conversation | Avoids recomputing past context on every decode step; the single largest VRAM consumer in long-context serving |
| **Key (K) Tensor** | A vector that "advertises" what a token represents; used in attention to find relevant context | One of the two tensors stored per token in the KV cache |
| **Value (V) Tensor** | A vector that holds a token's actual contribution to the output blend in attention | Read out of the cache and combined to produce each new token |
| **Query (Q) Tensor** | A vector representing what the current token is "looking for" in past context | Not cached — only the current token needs one per decode step |
| **VRAM** | Video RAM — the fixed high-speed memory on a GPU used for weights, activations, and the KV cache | The scarcest resource in LLM serving; the KV cache competes directly with model weights for space |
| **MHA (Multi-Head Attention)** | The original attention style with one KV head per query head (1:1 ratio) | Baseline for attention; produces the largest KV cache |
| **GQA (Grouped Query Attention)** | Attention where multiple query heads share one KV head, reducing cache size by up to 8x | The modern standard for efficient serving; under 0.2% quality loss |
| **MQA (Multi-Query Attention)** | Extreme variant where all query heads share a single KV head, reducing cache 64–128x | Highest memory savings but 2–3% quality loss; used in some mobile or edge models |
| **External Fragmentation** | Wasted memory caused by free space being split into gaps too small for a new large allocation | Why naive contiguous KV cache allocation leaves VRAM effectively unusable even when "free" |
| **PagedAttention** | A memory management system that stores KV cache in small non-contiguous pages, like OS virtual memory | Reduces KV cache memory waste from ~60–80% down to under 4%, enabling far more concurrent requests |
| **Context Caching (Self-hosted)** | Storing the pre-computed KV cache of a common prompt prefix so multiple users can reuse it | Eliminates redundant prefill compute when many users share the same system prompt or knowledge base |
| **Prompt Caching (API-level)** | A provider feature that discounts or skips charges for input tokens whose KV cache is already stored | Cuts input cost by 50–90% for repeated prefixes like long system prompts or shared documents |
| **Tiered Caching** | Storing recently used KV data in VRAM, frequently used data in HBM, and occasional data on SSD | Balances fast access with capacity so more cache can be retained without wasting expensive VRAM |
| **SGLang** | An open-source inference engine with advanced prefix-cache reuse (RadixAttention) and tiered caching | Used for high-throughput text-only workloads where prompt reuse is common |
| **RAD-O** | Retrieval Augmented Decoding — compresses long-document KV caches into smaller "latent tokens" | Enables 2M+ token contexts on hardware that would otherwise only support 200k |
| **Latent Tokens** | A compressed representation of a long document's KV cache, much smaller than the full tensors | Allow very long contexts to fit in VRAM by trading some precision for a 10x reduction in cache size |
| **Prefix Sharing** | A PagedAttention feature where multiple requests that share the same prompt prefix point to the same physical KV blocks | Stores a common 5k-token system prompt once instead of once per user, saving enormous VRAM |
| **Copy-on-Write** | A memory strategy where shared blocks are only duplicated when one user modifies them | Allows prefix sharing without corrupting other users' caches when unique tokens are generated |
| **Break-Even (caching)** | The minimum number of times a cached prefix must be reused for caching to be cheaper than re-sending tokens | Guides the decision of whether prompt caching saves money (typically 1.1–5x reuse depending on provider) |
| **RAG (Retrieval Augmented Generation)** | Fetching relevant document chunks from a vector database and injecting them into the prompt | An alternative to context caching for large document corpora; trades recall accuracy for lower token cost |
| **Vector Database** | A database that stores and searches embeddings (numeric document representations) by semantic similarity | The retrieval backbone of RAG; not needed if the full document fits in context via caching |
| **BF16** | Brain Float 16 — a 16-bit floating-point format with a wider exponent range than FP16 | Common precision for KV cache storage; 2 bytes per value in the VRAM formula |
| **Sequence Length** | The total number of tokens in a request (prompt plus output so far) | Directly multiplies KV cache size; longer conversations consume proportionally more VRAM |

*Next: [Speculative Decoding](03-speculative-decoding.md)*
