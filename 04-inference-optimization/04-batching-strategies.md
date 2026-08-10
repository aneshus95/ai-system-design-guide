# Batching Strategies

Batching is the primary lever for increasing LLM throughput and reducing cost. Serving frameworks have moved beyond simple request-level batching to sub-token, iteration-level orchestration.

## Table of Contents

- [Static vs. Dynamic Batching](#static-vs-dynamic)
- [Continuous Batching](#continuous-batching)
- [In-Flight Batching (Prefill-Decode Fusion)](#in-flight-batching)
- [Chunked Prefill & RAD-O](#chunked-prefill)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Static vs. Dynamic Batching

In traditional ML (Classification), we use **Static Batching** where all requests must be the same size and start/end together. This is inefficient for LLMs due to variable response lengths.

> **Why (the rationale):** Static batching works for classification because outputs are always the same size (a single class label). LLM outputs are variable-length, so a batch of 32 requests might have outputs ranging from 5 to 5,000 tokens — the GPU idles for short requests waiting for the slowest one to finish.
> **When to use:** Static batching is appropriate only for offline batch jobs with homogeneous output lengths (e.g., running the same summarization task over a fixed-length dataset). Never use static batching for real-time serving with mixed workloads.
> **Nuances & gotchas:** Static batching's worst-case waste is proportional to the ratio of max-output to min-output length in a batch. Padding shorter sequences to the max length wastes both compute and memory. The "longest tail" problem means one unusually long response degrades the effective throughput for every other request in the batch.

---

## Continuous Batching (Iteration-level)

Continuous batching (pioneered by Orca and vLLM) allows new requests to join the batch and finished requests to leave at the end of every individual token generation step.

> **Why (the rationale):** The GPU's memory bandwidth capacity is wasted when short requests finish early and their slot sits idle inside a static batch. Continuous batching reclaims those slots instantly at the token level, filling them with queued requests so the GPU stays saturated throughout.
> **When to use:** The default for any production LLM serving system with concurrent users. Should be enabled in vLLM, SGLang, TGI, and TensorRT-LLM. Replace any static batching configuration immediately.
> **Nuances & gotchas:** Continuous batching increases aggregate throughput but can raise tail latency for individual requests — when new requests join mid-batch, they compete for the same KV cache VRAM. At very high concurrency, requests may queue at the prefill stage rather than the decode stage, leading to high TTFT. Continuous batching does NOT eliminate stalls from massive prefills — that's what Chunked Prefill solves.

| Aspect | Static Batching | Continuous Batching |
|--------|-----------------|---------------------|
| **Join/Leave** | Only at start/end | Any iteration |
| **GPU Utilization**| Low (waiting for longest) | High (always saturated) |
| **Throughput** | 1x | **4x - 10x** |
| **Latency** | Highest for shortest | Balanced |

---

## In-Flight Batching (Prefill-Decode Fusion)

Previously, serving engines processed a batch of "Prefill" (heavy compute) OR a batch of "Decode" (heavy memory). 
**In-Flight Batching** (TensorRT-LLM) allows mixing them:

> **Why (the rationale):** Prefill is compute-bound and decode is memory-bound — they saturate different GPU resources. Running them concurrently allows idle compute capacity during decode to be used for a new request's prefill, improving overall GPU utilization without compromising either phase.
> **When to use:** When your workload has a mix of new arriving requests and ongoing long generations. Most beneficial in TensorRT-LLM deployments on NVIDIA hardware; vLLM implements a variant via chunked prefill scheduling.
> **Nuances & gotchas:** Prefill and decode compete for KV cache VRAM even when their compute is overlapped — a large in-flight prefill can still evict decode KV blocks if VRAM is tight. The performance benefit depends on the specific arrival rate and request lengths; measure on your workload before claiming the gain.
- 1 request is in the Prefill phase.
- 15 requests are in the Decode phase.
- **Benefit**: The Prefill request utilizes the GPU's idle compute cores while the Decode requests utilize the memory bandwidth.

---

## Chunked Prefill & RAD-O

Massive context prompts (1M+ tokens) can hang a batch for seconds during the Prefill phase, causing "stalls."

**The fix: Chunked Prefill**
Instead of prefilling 128k tokens at once, the engine breaks the prefill into smaller chunks (e.g., 4k tokens each) and interleaves them with the ongoing Decode steps of other users. This maintains a steady **TPOT** even when heavy requests arrive.

> **Why (the rationale):** A single 1M-token prefill monopolizes the GPU for several seconds, preventing all decode requests from generating tokens during that window (TPOT spikes for everyone). Chunked Prefill breaks this monopoly by slicing the prefill into small chunks and interleaving them with decode steps — every user gets tokens at every iteration.
> **When to use:** Essential when serving mixed workloads where some requests have very long prompts (>32k tokens) alongside short-prompt requests in the same batch. Default-enabled in modern vLLM and SGLang configurations.
> **Nuances & gotchas:** Chunked prefill adds overhead because the KV cache for the chunked request is built incrementally — the request's first token is delayed by the number of chunks. The chunk size is a tunable tradeoff: smaller chunks reduce stalls but increase per-request overhead. Chunked prefill interacts with the block manager — partial prefill blocks must be allocated without knowing if the request will complete.

---

## Interview Questions

### Q: Why is Continuous Batching superior to Static Batching for LLMs?

**Strong answer:**
Static batching forces all requests in a batch to wait for the longest generation to complete (the "longest tail" problem). If one user asks for 500 tokens and another for 5 tokens, the GPU remains idle for the 5-token user for 495 cycles. Continuous batching allows the 5-token user's request to exit the GPU immediately after its last token, freeing up VRAM and compute slots for a new request from the queue. This maximizes "Tokens per Second" across the entire hardware cluster.

### Q: What is a "stall" in LLM serving, and how does Chunked Prefill mitigate it?

**Strong answer:**
A "stall" occurs when a massive new request arrives and its Prefill phase (which is compute-hungry) takes 2-3 seconds to complete. During this time, the GPU is so busy with the prefill that it cannot generate tokens for existing users in the "Decode" phase, causing their TPOT to spike. Chunked Prefill breaks that 3-second prefill into small 200ms "chunks," processing one chunk and then doing one round of decoding for everyone else, before returning to the next prefill chunk. This ensures a consistent, smooth experience for all users.

---

## References
- Yu et al. "Orca: A Distributed Serving System for [Transformer] Models" (2022)
- NVIDIA. "TensorRT-LLM: In-Flight Batching" (2023)
- vLLM Project. "Iteration-Level Scheduling" (2023)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Batching** | Processing multiple requests together in a single GPU forward pass | The primary lever for increasing throughput and lowering cost per token |
| **Static Batching** | A batching method where all requests must be collected before processing starts and the batch ends only when every request finishes | Simple to implement but wastes GPU cycles waiting for the slowest request in the group |
| **Continuous Batching** | A batching approach where individual requests join and leave the batch at every single token step | Keeps the GPU constantly saturated, achieving 4–10x higher throughput than static batching |
| **Iteration-Level Scheduling** | Processing decisions made at every individual token generation step rather than at the request level | Enables continuous batching; pioneered by the Orca research project and adopted by vLLM |
| **In-Flight Batching** | Mixing prefill-phase and decode-phase requests in the same batch simultaneously | Uses otherwise idle GPU compute units during decode by processing a new prefill at the same time |
| **Prefill-Decode Fusion** | Running a prefill request and multiple decode requests in the same GPU step | Improves overall GPU utilization by filling compute gaps left by the memory-bound decode phase |
| **Chunked Prefill** | Breaking a large prompt's prefill into smaller pieces (e.g., 4k tokens each) interleaved with decode steps | Prevents one massive prompt from stalling all other users' decode progress |
| **Stall** | A period where ongoing decode requests stop receiving tokens because the GPU is monopolized by a heavy prefill | The latency spike Chunked Prefill is designed to eliminate |
| **TPOT (Time Per Output Token)** | The delay between each successive token in a response | The metric most affected by stalls; a stall causes TPOT to spike for all concurrent users |
| **Longest-Tail Problem** | The issue in static batching where the entire batch waits idle for the slowest, longest request to finish | Wastes GPU cycles proportional to the gap between shortest and longest output in a batch |
| **GPU Utilization** | The percentage of time a GPU's compute or memory units are actively doing work | Low utilization means wasted money; continuous batching targets near-100% utilization |
| **Tokens per Second** | The total number of tokens generated across all users per second on a system | The throughput metric that directly determines cost per query |
| **Orca** | A research serving system (2022) that pioneered iteration-level scheduling and continuous batching | The foundational paper behind the batching approach used in vLLM and modern serving engines |
| **TensorRT-LLM** | NVIDIA's inference library that implements in-flight batching for mixed prefill-decode workloads | Achieves highest peak throughput on NVIDIA hardware by fusing prefill and decode operations |
| **RAD-O** | Retrieval Augmented Decoding — compresses very long prompt KV caches to reduce prefill overhead | Works alongside Chunked Prefill to handle 1M+ token contexts without unbounded stalls |
| **Variable Response Lengths** | The reality that different users' outputs can be 5 tokens or 5000 tokens, unpredictably | The root reason static batching is inefficient for LLMs and continuous batching is necessary |
| **Micro-batching** | A technique in pipeline parallelism that subdivides a batch into smaller pieces to reduce GPU idle time | Reduces "bubble time" — the gaps when one GPU waits for another pipeline stage to finish |
| **KV Cache Slot** | The VRAM space reserved for one request's key-value cache in the batch | A finite resource; when slots fill, new requests must queue; continuous batching frees slots as soon as a request finishes |

*Next: [PagedAttention](05-paged-attention.md)*
