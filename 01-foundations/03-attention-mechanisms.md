# Attention Mechanisms

Attention is the core innovation that enables transformers. This chapter covers the mathematical foundations, variants, and optimizations that are essential for system design and interviews.

## Table of Contents

- [Attention Fundamentals](#attention-fundamentals)
- [Scaled Dot-Product Attention](#scaled-dot-product-attention)
- [Multi-Head Attention](#multi-head-attention)
- [Attention Patterns](#attention-patterns)
- [Efficient Attention Variants](#efficient-attention-variants)
- [Flash Attention (v2 & v3)](#flash-attention)
- [Multi-head Latent Attention (MLA)](#multi-head-latent-attention-mla)
- [KV Cache Optimizations & Context Caching](#kv-cache-optimizations)
- [Practical Implications](#practical-implications)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Attention Fundamentals

### The Core Idea

Attention allows each position in a sequence to gather information from all other positions. Unlike recurrence (which passes information step by step), attention creates direct connections.

**Mental model for distributed systems engineers:**
- RNN: Message passing along a chain
- Attention: Pub/sub where every node can query every other node

### Query, Key, Value Framework

Attention uses three projections of the input:

| Component | Role | Analogy |
|-----------|------|---------|
| Query (Q) | What am I looking for? | Search query |
| Key (K) | What do I contain? | Document index |
| Value (V) | What do I contribute? | Document content |

```python
# Input: x of shape [batch, seq_len, d_model]

Q = x @ W_q  # [batch, seq_len, d_k]
K = x @ W_k  # [batch, seq_len, d_k]
V = x @ W_v  # [batch, seq_len, d_v]
```

---

## Scaled Dot-Product Attention

The fundamental attention operation:

```python
def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.shape[-1]
    
    # Compute attention scores
    scores = Q @ K.transpose(-2, -1)  # [batch, seq_len, seq_len]
    scores = scores / math.sqrt(d_k)  # Scale
    
    # Apply mask (for causal attention)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # Convert to probabilities
    attention_weights = F.softmax(scores, dim=-1)
    
    # Weighted sum of values
    output = attention_weights @ V
    
    return output, attention_weights
```

### Why Scale by Square Root of d_k?

**Interview favorite**: This question tests numerical intuition.

Without scaling, dot products grow with dimension:
- For random unit vectors q and k of dimension d
- E[q . k] = 0, but Var[q . k] = d
- Standard deviation = sqrt(d)

When d is large (512 or more), dot products can be very large or very small. Softmax on large values approaches one-hot, causing vanishing gradients.

```python
# Demonstration
import numpy as np

d = 512
q = np.random.randn(d)
k = np.random.randn(d)

unscaled = np.dot(q, k)      # Magnitude ~ sqrt(512) ~ 22
scaled = unscaled / np.sqrt(d)  # Magnitude ~ 1
```

### Causal Masking

For autoregressive generation, each position can only attend to previous positions:

```python
def create_causal_mask(seq_len):
    # Lower triangular matrix
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask

# Example for seq_len=4:
# [[1, 0, 0, 0],
#  [1, 1, 0, 0],
#  [1, 1, 1, 0],
#  [1, 1, 1, 1]]
```

Positions with mask=0 get score of negative infinity, becoming 0 after softmax.

---

## Multi-Head Attention

Instead of one attention function, use multiple "heads" that attend to different aspects:

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, x, mask=None):
        batch_size, seq_len, d_model = x.shape
        
        # Project to Q, K, V
        Q = self.W_q(x)  # [batch, seq_len, d_model]
        K = self.W_k(x)
        V = self.W_v(x)
        
        # Reshape to multiple heads
        Q = Q.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        # Now: [batch, num_heads, seq_len, d_k]
        
        # Attention per head
        attn_output, _ = scaled_dot_product_attention(Q, K, V, mask)
        
        # Concatenate heads
        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(batch_size, seq_len, d_model)
        
        # Final projection
        output = self.W_o(attn_output)
        return output
```

**Why multiple heads?**
1. Different heads learn different patterns (syntax, semantics, coreference)
2. Provides representational diversity (ensemble effect)
3. Enables parallel computation across heads

### Head Count Patterns

| Model | d_model | Heads | d_k per head |
|-------|---------|-------|--------------|
| BERT-base | 768 | 12 | 64 |
| GPT-2 | 768 | 12 | 64 |
| GPT-3 175B | 12288 | 96 | 128 |
| Llama 2 70B | 8192 | 64 | 128 |

The d_k of 64 or 128 is remarkably consistent across model sizes.

---

## Attention Patterns

### What Attention Learns

Different heads specialize in different patterns:

| Pattern Type | What It Captures | Example |
|--------------|------------------|---------|
| Positional | Adjacent tokens | Next/previous word |
| Syntactic | Grammatical relations | Subject-verb |
| Semantic | Meaning relations | Coreference |
| Delimiter | Punctuation, structure | Section boundaries |
| Rare | Infrequent patterns | Rare word copying |

### Visualizing Attention

Attention weights can be visualized as heatmaps showing which positions attend to which:

```
Query positions (rows) vs Key positions (columns)

"The cat sat on the mat"

         The  cat  sat  on   the  mat
The     [□    ○    ○    ○    ○    ○ ]
cat     [●    □    ○    ○    ○    ○ ]
sat     [○    ●    □    ○    ○    ○ ]
on      [○    ○    ●    □    ○    ○ ]
the     [○    ○    ○    ○    □    ○ ]
mat     [○    ●    ○    ●    ●    □ ]

● = high attention, ○ = low attention
```

"mat" attends strongly to "cat" (semantic), "on" (syntactic), and "the" (determiner).

---

## Efficient Attention Variants

Standard attention is O(n^2) in sequence length. Many variants reduce this:

### Sparse Attention

Attend only to a subset of positions rather than all:

| Variant | Pattern | Complexity | Example |
|---------|---------|------------|---------|
| Local | Window around each position | O(n * w) | Longformer |
| Strided | Every k-th position | O(n^2 / k) | Sparse Transformer |
| Global | Special tokens attend everywhere | O(n * g) | Longformer, BigBird |
| Block | Block-diagonal attention | O(n * b) | BigBird |

**Longformer pattern:**
```
Local window + Global tokens

[G] [L] [L] [L] [L] [G] [L] [L] [L] [L]

G: Global tokens (attend to/from all)
L: Local tokens (attend within window)
```

### Linear Attention

Replace softmax with linearizable alternatives:

```python
# Standard attention (quadratic)
attention = softmax(Q @ K.T) @ V

# Linear attention approximation
attention = (Q @ (K.T @ V))  # Associativity trick
```

**Variants:**
- Performer: Random feature approximation
- Linear Transformer: elu(Q) @ (elu(K).T @ V)

**Tradeoff:** Faster but quality degrades, especially for tasks requiring precise attention.

### Complexity Comparison

| Method | Time | Space | Quality | Notes |
|--------|------|-------|---------|-------|
| Standard | O(n^2) | O(n^2) | Best | Baseline |
| Sparse (Longformer) | O(n) | O(n) | Near best | For long docs |
| Linear (Performer) | O(n) | O(n) | Degraded | Best for very long |
| Flash Attention | O(n^2) | O(n) | Best | Best of both |

---

## Flash Attention

Flash Attention is the state-of-the-art implementation that achieves O(n) memory while computing exact attention.

### The Problem It Solves

Standard attention requires materializing the n x n attention matrix:
- For 8K context: 64M floats = 256 MB per layer per head
- For 100K context: 10B floats = 40 GB per layer per head

This memory requirement limits batch sizes and context lengths.

### How It Works

Flash Attention uses tiling and recomputation to avoid storing the full attention matrix:

```
Standard: Q, K -> Attention Matrix (n x n) -> Output
Flash:    Q, K -> Tiles (block_size x block_size) -> Incremental Output
```

**Key ideas:**
1. Process attention in blocks that fit in SRAM
2. Never materialize full attention matrix in HBM
3. Recompute attention during backward pass (faster than loading from HBM)

### Performance Impact

### FlashAttention-2 (Work Partitioning)
Optimized for A100/H100 by improving parallelism across heads and sequence length.

### FlashAttention-3 (FP8 & H100 Optimization)
**The current standard for H100/B200 clusters:**
- **Asynchronous Execution**: Overlaps GEMM (matrix mult) and softmax operations using TMA (Tensor Memory Accelerator) on H100.
- **FP8 Support**: Native support for FP8 precision, doubling throughput compared to FP16 while maintaining attention accuracy via stochastic rounding.
- **Speedup**: ~1.5x-2.0x faster than FlashAttention-2 for long context prefill.

---

## Multi-head Latent Attention (MLA)

Introduced by DeepSeek (V2/V3), **MLA is the modern alternative to GQA** for extreme KV cache pressure.

Instead of just grouping heads, MLA compresses the Key and Value vectors into a **low-dimensional latent space** before storing them in the cache.

```
Query (Up-projected) ────────┐
                             ▼
Key, Value (Down-projected) ─▶ [Low-dim Latent Cache] ─▶ [Output]
                             ▲
                             └─ Projection Matrices
```

| Metric | MHA | GQA | MLA (Dec 2025) |
|--------|-----|-----|----------------|
| KV Cache Size | 100% | 12.5% | **~5%** |
| Quality | Baseline | Near-Baseline | **Superior to GQA** |
| Latency | Baseline | Faster | **Fastest (Reduced I/O)** |

**Why MLA wins**: It uses "Decoupled Rotary Positional Embeddings" which allow the compressed latent KV to be reused without decoding, saving massive memory bandwidth during long-context generation.

---

## KV Cache Optimizations & Context Caching

### Context Caching (System-level)
API providers (OpenAI, Gemini, Anthropic) now offer **Context Caching**. 
- **How it works**: Pre-computes and stores the KV tensors for a long "prefix" (e.g., a 100k token law book).
- **Benefit**: Reduces TTFT (Time to First Token) by 90% and cost by 50-90% for repeated prefixes.

### Sliding Window Attention (SWA)
Used in Mistral/Gemma models to limit the attention depth to a fixed window (e.g., 4096 tokens), preventing the KV cache from growing indefinitely.

### Multi-Query Attention (MQA)

Share a single K and V across all query heads:

```python
# Standard MHA
Q: [batch, num_heads, seq, d_k]  # 32 heads
K: [batch, num_heads, seq, d_k]  # 32 separate K
V: [batch, num_heads, seq, d_k]  # 32 separate V

# MQA
Q: [batch, num_heads, seq, d_k]  # 32 heads
K: [batch, 1, seq, d_k]          # 1 shared K
V: [batch, 1, seq, d_k]          # 1 shared V
```

**Effect:** 32x reduction in KV cache size, some quality loss.

### Grouped-Query Attention (GQA)

Share K and V across groups of query heads:

```python
# GQA with 8 KV heads for 64 query heads (8:1 ratio)
Q: [batch, 64, seq, d_k]  # 64 query heads
K: [batch, 8, seq, d_k]   # 8 KV heads
V: [batch, 8, seq, d_k]   # 8 KV heads

# Each KV head serves 8 query heads
```

**Effect:** 8x KV cache reduction with minimal quality loss.

**Models using GQA:**
- Llama 2 70B: 8 KV heads for 64 query heads
- Mistral 7B: 8 KV heads for 32 query heads
- Gemma: Various configurations

### Comparison

| Attention | KV Cache | Quality | Models |
|-----------|----------|---------|--------|
| MHA | Full | Best | GPT-3 |
| GQA | 1/8 typical | Near best | Llama 2, Mistral |
| MQA | 1/n_heads | Reduced | PaLM, Falcon |

---

## Practical Implications

### For System Design

1. **Batch size vs context tradeoff:**
   - Total GPU memory = Model + KV cache * batch_size
   - Longer contexts mean smaller batches
   - GQA models can serve more concurrent requests

2. **Latency budget allocation:**
   - Attention is O(n^2) compute, O(n) with Flash
   - Prefill (processing prompt) scales with prompt length
   - Decode (generating) scales with generated + prompt length

3. **Memory bandwidth bottleneck:**
   - Generation is often memory-bound
   - Loading KV cache for each token dominates
   - Larger batches amortize this cost

### Prefill vs Decode

| Phase | Compute Pattern | Bottleneck |
|-------|-----------------|------------|
| Prefill | Process all input tokens | Compute (GPU cores) |
| Decode | Generate one token at a time | Memory (bandwidth) |

This is why TTFT (time to first token) and TPS (tokens per second) are measured separately.

### Context Length Scaling

| Context | Attention Compute | KV Cache (Llama 70B) |
|---------|-------------------|---------------------|
| 4K | Baseline | 10.7 GB |
| 8K | 4x | 21.5 GB |
| 32K | 64x | 86 GB |
| 128K | 1024x | 344 GB |

Long context requires:
- Flash Attention (memory efficient)
- GQA or MQA (smaller KV cache)
- Potentially model parallelism

---

## Interview Questions

### Q: Explain the attention mechanism and why it scales quadratically.

**Strong answer:**
Attention computes pairwise interactions between all positions. For n positions:

1. Q @ K^T produces an n x n matrix of scores
2. Each attention score is a dot product of a query and key
3. Total: n^2 dot products

This is quadratic in sequence length. For 8K tokens, that is 64 million pairwise scores per layer per head. For 128K tokens, it is 16 billion.

The quadratic scaling limits context length. Solutions include:
- Flash Attention: O(n^2) compute but O(n) memory
- Sparse attention: O(n) by attending to subsets
- Linear attention: O(n) approximations

### Q: What is the KV cache and why is it critical for serving?

**Strong answer:**
During autoregressive generation, we produce one token at a time. Without caching, each new token would require recomputing K and V for all previous positions.

The KV cache stores K and V tensors from previous positions. On each new token:
1. Compute Q, K, V only for the new position
2. Append new K, V to cache
3. Attend to full cached K, V

This reduces per-token complexity from O(n) to O(1) for projection computation.

The cost is memory: KV cache scales linearly with sequence length. For Llama 70B at 8K context, it is about 21 GB per request. This directly limits batch size and throughput.

GQA and MQA reduce this by sharing K, V across query heads.

### Q: Compare MHA, GQA, and MQA.

**Strong answer:**
| Variant | K,V heads | KV Cache | Quality | Use Case |
|---------|-----------|----------|---------|----------|
| MHA | Equal to Q heads | Full | Best | Training, quality-critical |
| GQA | Fewer than Q heads | Reduced | Near MHA | Production serving |
| MQA | 1 | Minimal | Reduced | Memory-constrained |

MHA: Each query head has its own K and V. Best quality but largest KV cache.

GQA: Groups of query heads share K and V. Llama 2 uses 8 KV heads for 64 query heads (8:1 ratio). 8x smaller cache with minimal quality loss.

MQA: All query heads share one K and V. Maximum memory savings but measurable quality reduction. Used by PaLM.

For serving, GQA is the best tradeoff. It enables larger batch sizes (higher throughput) with quality nearly identical to MHA.

### Q: How does Flash Attention achieve O(n) memory?

**Strong answer:**
Standard attention materializes the full n x n attention matrix in GPU memory. Flash Attention avoids this by:

1. **Tiling:** Process blocks of Q and K that fit in on-chip SRAM
2. **Online softmax:** Compute softmax incrementally without storing all scores
3. **Recomputation:** During backward pass, recompute attention rather than loading saved values

The key insight is that GPU SRAM (20 MB per SM) is 10x faster than HBM (80 GB). By doing more arithmetic in SRAM and fewer HBM reads/writes, Flash Attention is both faster and uses less memory.

The result is exact attention (not an approximation) with O(n) memory and 2-4x speedup.

---

## References

- Vaswani et al. "Attention Is All You Need" (2017)
- Dao et al. "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" (2022)
- Dao "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning" (2023)
- Beltagy et al. "Longformer: The Long-Document Transformer" (2020)
- Ainslie et al. "GQA: Training Generalized Multi-Query Transformer Models" (2023)
- Shazeer "Fast Transformer Decoding: One Write-Head is All You Need" (MQA, 2019)
- [Flash Attention Repository](https://github.com/Dao-AILab/flash-attention)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Attention** | A mechanism that lets each position in a sequence gather weighted information from all other positions | Core innovation of the transformer; replaces sequential recurrence with parallel direct connections |
| **Scaled Dot-Product Attention** | The fundamental attention operation: compute Q·Kᵀ, scale by √d_k, apply softmax, multiply by V | The mathematical primitive that every attention variant is built upon |
| **Query (Q)** | A linear projection of a token representing what it is searching for | Drives matching in attention; paired with keys to produce relevance scores |
| **Key (K)** | A linear projection of a token representing what it contains or advertises | Acts as the index that other tokens' queries are matched against |
| **Value (V)** | A linear projection of a token representing the content it contributes once attended to | Carries the actual information transferred to the attending token after matching |
| **√d_k Scaling** | Dividing raw dot-product scores by the square root of the key dimension | Keeps score variance at 1 regardless of head size; prevents softmax saturation and vanishing gradients |
| **Softmax** | Function that converts a row of scores into a probability distribution summing to 1 | Turns raw similarity scores into attention weights used to blend values |
| **Causal Masking** | Setting upper-triangle scores to −∞ before softmax so future positions get zero weight | Enforces autoregressive (left-to-right) generation; required for decoder-only models |
| **Multi-Head Attention (MHA)** | Running h independent attention operations in parallel, then concatenating outputs | Different heads specialize in different patterns (syntax, semantics, coreference) |
| **Attention Head** | A single scaled dot-product attention operation operating on a d_k-dimensional subspace | One of h parallel attention functions; each learns different relational patterns |
| **d_k (Head Dimension)** | The dimension of query and key vectors within a single attention head | Remarkably consistent at 64–128 across model sizes; determines scaling factor √d_k |
| **W_O (Output Projection)** | A learned linear matrix that projects concatenated multi-head outputs back to d_model | Mixes information across heads after concatenation; final step of multi-head attention |
| **Attention Pattern** | The distribution of attention weights showing which positions a token focuses on | Visualized as a heatmap; reveals what linguistic structures a head has learned |
| **Positional Attention** | Attention heads that primarily attend to adjacent or nearby tokens | Captures local context like n-gram relationships |
| **Syntactic Attention** | Attention heads that capture grammatical relationships such as subject-verb pairs | Enables models to understand sentence structure without explicit parse trees |
| **Coreference Attention** | Attention heads that link pronouns to their referents across the sequence | Example: "it" → "animal" in "The animal was tired because it…" |
| **Sparse Attention** | Attention computed only for a subset of position pairs rather than all n² pairs | Reduces complexity to O(n) for long documents; used in Longformer and BigBird |
| **Local (Windowed) Attention** | Attending only to tokens within a fixed-size window around each position | O(n·w) complexity; effective for tasks where relevant context is nearby |
| **Global Tokens** | Special tokens that attend to and are attended by all positions in sparse attention models | Provide long-range information flow within otherwise local attention patterns |
| **Strided Attention** | Attending to every k-th position rather than every adjacent position | Captures periodic structure with O(n²/k) complexity |
| **Block Attention** | Dividing the sequence into blocks and only computing attention within each block | Simple form of sparse attention; O(n·b) where b is block size |
| **Longformer** | A transformer using local window plus global token sparse attention for long documents | Extends BERT to handle documents of 4K–16K tokens at linear cost |
| **BigBird** | A transformer combining local, global, and random attention for very long sequences | Provably Turing-complete sparse attention used for genomics and long documents |
| **Linear Attention** | Approximates softmax attention using kernel feature maps to reduce O(n²) to O(n) | Suitable for extremely long sequences but trades off quality compared to exact attention |
| **Performer** | A linear attention model using random Fourier features to approximate the softmax kernel | Achieves O(n) complexity while approximating standard attention |
| **Flash Attention** | An IO-aware attention implementation that tiles computation in fast SRAM to avoid reading/writing the n×n matrix to HBM | Exact attention with O(n) memory; 2–4× speedup from reducing slow memory accesses |
| **SRAM (Static RAM)** | Fast on-chip memory in each GPU streaming multiprocessor (~20 MB per SM) | Flash Attention keeps intermediate results here to avoid slow HBM round-trips |
| **HBM (High Bandwidth Memory)** | The main GPU VRAM where model weights and KV cache are stored (~80 GB on H100) | Much larger than SRAM but ~10× slower; minimizing HBM reads is key to Flash Attention's speed |
| **Tiling** | Breaking large matrix operations into smaller blocks that fit in SRAM | Core technique of Flash Attention; enables memory-efficient computation of large matrices |
| **Online Softmax** | Computing softmax incrementally over tiles without needing all scores at once | Allows Flash Attention to produce exact softmax results without materializing the full n×n matrix |
| **FlashAttention-2** | Improved Flash Attention with better work partitioning across thread blocks and warps | Optimized for A100/H100; ~2× faster than v1 through better GPU parallelism |
| **FlashAttention-3** | Further improved Flash Attention adding FP8 support and asynchronous GEMM/softmax overlap | Current standard for H100/B200 clusters; 1.5–2× faster than v2 for long-context prefill |
| **FP8** | 8-bit floating-point precision for matrix multiplications | Doubles compute throughput versus FP16 on H100; used in FlashAttention-3 |
| **TMA (Tensor Memory Accelerator)** | An H100 hardware unit that handles asynchronous memory transfers between HBM and SRAM | Enables FlashAttention-3 to overlap memory loads with computation for higher utilization |
| **Multi-head Latent Attention (MLA)** | DeepSeek's attention variant that compresses KV projections into a low-dimensional latent vector before caching | Reduces KV cache to ~5% of MHA with quality superior to GQA; introduced in DeepSeek-V2/V3 |
| **Grouped-Query Attention (GQA)** | Sharing Key/Value heads across groups of Query heads to shrink the KV cache | 8× smaller KV cache than MHA with near-identical quality; current production standard |
| **Multi-Query Attention (MQA)** | All query heads share a single Key and Value head | Maximum memory savings but measurable quality loss; used by PaLM and Falcon |
| **KV Cache** | Stored Key and Value tensors from all prior positions to avoid recomputation during generation | Reduces per-step compute from O(n) to O(1) for KV projections; trades memory for speed |
| **Context Caching** | API-level feature that pre-computes and stores KV tensors for a fixed long prefix | Cuts TTFT by 90% and cost by 50–90% for requests sharing a common prefix like a system prompt |
| **Sliding Window Attention (SWA)** | Limiting attention depth to a fixed window of past tokens so KV cache does not grow unboundedly | Used in Mistral/Gemma; enables practical long-context at controlled memory cost |
| **TTFT (Time to First Token)** | The elapsed time from sending a request until receiving the first generated token | Key user-experience latency; dominated by prompt length and prefill compute speed |
| **TPS (Tokens Per Second)** | The rate of token generation after the first token appears | Measures decode throughput; dominated by memory bandwidth and batch size |
| **Prefill** | The parallel processing of all input prompt tokens to populate the KV cache | Compute-bound; scales with prompt length; determines TTFT |
| **Decode** | The sequential one-token-at-a-time generation phase after prefill | Memory-bandwidth-bound; each step loads the full KV cache; determines TPS |
| **O(n²) Complexity** | Attention cost growing as the square of sequence length due to all-pair interactions | Fundamental bottleneck for long contexts; doubling sequence length quadruples attention cost |
| **Model Parallelism** | Distributing model layers or tensors across multiple GPUs to handle models that do not fit on one GPU | Required for serving very large models (70B+) or very long contexts |

*Previous: [Tokenization Deep Dive](02-tokenization-deep-dive.md) | Next: [Transformer Architecture](04-transformer-architecture.md)*
