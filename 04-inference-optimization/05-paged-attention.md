# PagedAttention

PagedAttention is the foundational algorithm behind high-throughput serving engines (vLLM, SGLang, TensorRT-LLM). It solves the "Memory Fragmentation" problem that previously limited LLM scalability.

## Table of Contents

- [The Contiguous Memory Problem](#contiguous-memory)
- [How PagedAttention Works](#how-it-works)
- [Managing Virtual Memory (Block Manager)](#block-manager)
- [KV Cache Sharing (Copy-on-Write)](#sharing)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Contiguous Memory Problem

Standard deep learning frameworks allocate memory in large, contiguous blocks. 
For an LLM request, you might pre-allocate memory for a `max_sequence_length` of 8192 tokens.

**The Waste:**
1. **Internal Fragmentation**: If the user only generates 10 tokens, 99.9% of that reserved block is wasted.
2. **External Fragmentation**: Memory is broken into gaps too small for a new "large block," even if total free memory is high.

---

## How PagedAttention Works (vLLM)

PagedAttention draws inspiration from Virtual Memory in Operating Systems.

1. **Tokens to Blocks**: The KV cache for a request is broken into small, fixed-size **Blocks** (e.g., 16 tokens per block).
2. **Logical vs. Physical**: The model thinks it's attending to a contiguous sequence (Logical memory), but the blocks are scattered throughout VRAM (Physical memory).
3. **The Lookup Table**: A **Block Table** maps logic indices to physical addresses.

**Primary Benefit**: Memory waste drops from ~60-80% down to **less than 4%**.

---

## Managing Virtual Memory (Block Manager)

Serving frameworks (vLLM, SGLang) act as "mini-OSs" for GPUs.

- **Allocation**: When a new request starts, the Block Manager assigns it a set of empty physical blocks.
- **Eviction**: If VRAM is full, the manager can "swap" inactive KV blocks to CPU RAM and bring them back when needed (Paged Swap).

---

## KV Cache Sharing (Copy-on-Write)

PagedAttention enables effortless sharing of "Common Prefixes."

**The Scenario**: 100 users are chatting with the same 5,000-token system prompt.
- **Traditional**: Store that 5,000-token KV cache 100 times (**500k tokens** in VRAM).
- **PagedAttention**: Store it **once** via the Block Table and have all 100 users point to the same physical blocks.
- **Copy-on-Write**: If a user generates a unique token, a new block is created just for them, while the shared blocks remain unchanged.

---

## Interview Questions

### Q: Why does PagedAttention significantly increase throughput?

**Strong answer:**
PagedAttention increases throughput by allowing for much larger **batch sizes**. Because it eliminates internal and external memory fragmentation, we can pack many more requests into the same GPU VRAM. In traditional serving, we might only fit 4 requests because we have to "reserve" max-length blocks; with PagedAttention, we can fit 20-30 requests because we only use memory for the tokens that actually exist. Larger batches lead to better GPU utilization and significantly higher aggregate tokens per second.

### Q: Explain the "Block Table" in the context of vLLM.

**Strong answer:**
The Block Table is a mapping structure that bridges the Gap between the model's expectation of contiguous data and the physical reality of scattered memory. Each entry in the table corresponds to a "Logical Block" of tokens. It stores the physical address of the GPU memory where that block's key and value tensors are stored. This allows the framework to dynamically allocate and free memory in small chunks, enabling advanced features like prefix sharing and efficient multi-threading.

---

## References
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (SOSP 2023)
- vLLM Documentation. "PagedAttention Logic" (2024)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **PagedAttention** | A memory management algorithm that stores KV cache in small, non-contiguous fixed-size pages instead of one large contiguous block | Cuts KV cache memory waste from 60–80% down to under 4%, enabling far larger batch sizes |
| **Contiguous Memory Allocation** | The traditional approach of reserving one unbroken block of RAM for a request's entire max possible sequence length | Simple but wasteful — unused reserved space cannot be given to other requests |
| **Internal Fragmentation** | Wasted memory inside an allocated block that is reserved but never used | Happens when a user generates far fewer tokens than the pre-allocated max sequence length |
| **External Fragmentation** | Wasted memory between allocations — free space exists but in gaps too small for a new large block | Makes total free VRAM appear large while no individual allocation can actually fit |
| **Block (KV Block)** | A fixed-size unit of KV cache memory, typically holding 16 tokens' worth of key-value pairs | The basic allocation unit in PagedAttention; allocated on demand so no memory is reserved upfront |
| **Block Table** | A lookup table that maps a request's logical block indices to the physical GPU memory addresses where those blocks live | Lets the model treat memory as contiguous even though blocks are physically scattered in VRAM |
| **Logical Memory** | The sequential view of a request's tokens that the attention mechanism sees during computation | An abstraction layer that hides the physical scatter of blocks from the model |
| **Physical Memory** | The actual GPU VRAM addresses where KV blocks are stored, potentially scattered across many locations | The real layout managed by PagedAttention's block manager |
| **Block Manager** | The component inside vLLM or SGLang that allocates, frees, evicts, and swaps KV blocks on behalf of all active requests | Acts like an operating system's memory manager for GPU VRAM |
| **Paged Swap** | Moving inactive KV blocks from GPU VRAM to CPU RAM when VRAM is full, then reloading them when needed | Allows the system to handle more concurrent requests than fit in VRAM at once |
| **Virtual Memory (OS analogy)** | The operating system concept that inspired PagedAttention — programs see a contiguous address space that maps to scattered physical RAM | The design pattern PagedAttention borrows to manage non-contiguous KV cache |
| **Prefix Sharing** | A PagedAttention capability where multiple requests that share the same prompt prefix point to the same physical KV blocks | Stores a common system prompt once in VRAM instead of once per user |
| **Copy-on-Write (CoW)** | A strategy where shared physical blocks are only duplicated when one request needs to modify them | Enables safe prefix sharing — unique tokens get a new block, while common-prefix blocks remain shared |
| **Memory Efficiency** | The fraction of allocated VRAM that is actually used for valid token data | PagedAttention raises this from ~20–40% (naive) to 96%+ |
| **Batch Size** | The number of concurrent requests processed together in one GPU step | Higher batch size means better GPU utilization and throughput; PagedAttention enables larger batches by eliminating waste |
| **vLLM** | The open-source serving engine that first implemented PagedAttention for production LLM serving | The reference implementation and most widely deployed engine for continuous batching with paged memory |
| **SGLang** | An inference engine that also uses paged KV cache management alongside RadixAttention for prefix reuse | The throughput leader for structured-output and function-calling workloads |
| **Max Sequence Length** | The maximum number of tokens a model or serving engine is configured to handle per request | In naive serving, VRAM is pre-reserved for this length per request, causing internal fragmentation |
| **Aggregate Throughput** | Total tokens per second produced across all users on a system | The metric PagedAttention improves most directly by allowing more concurrent requests per GPU |

*Next: [Serving Infrastructure](06-serving-infrastructure.md)*
