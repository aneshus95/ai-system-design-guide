# Short-Term Context Management

Short-term context (L1 Memory) is the high-speed interface where reasoning happens. Managing it well is no longer about "message lists" but about **KV Cache Optimization** and **Dynamic Context Allocation**.

## Table of Contents

- [The Context Lifecycle](#lifecycle)
- [KV Cache Tiling and PagedAttention](#paged-attention)
- [Prefix Caching (System Prompt Preservation)](#prefix-caching)
- [Sliding Windows vs. Summarization](#sliding-vs-summary)
- [Contextual Compression (Selective Dropping)](#compression)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Context Lifecycle

Context goes through three stages:
1. **Intake**: User query + recent history + system instructions.
2. **Processing**: The GPU computes the KV cache for the new tokens.
3. **Eviction**: Removing old tokens to make room for new ones once the limit is reached.

> **Why (the rationale):** Understanding the lifecycle is what separates engineers who blindly append messages from those who design context budgets. Every token in L1 costs money and attention — proactive eviction and strategic intake prevent runaway costs and degraded reasoning.
> **When to use:** Apply lifecycle thinking from the first request. Even at low volume, failing to plan eviction leads to context overflows or quality regressions that are hard to debug retrospectively.
> **Nuances & gotchas:** Eviction is not free — deciding what to drop requires its own logic (recency, importance scoring), and a naive FIFO eviction can silently drop a system-prompt rule that was near the beginning of the context. The model never signals that eviction degraded its output; it just produces worse results.

---

## KV Cache Tiling

Modern inference engines (vLLM, TensorRT-LLM) use **PagedAttention**.
- **Idea**: Instead of allocating a contiguous block of GPU memory for the context, memory is broken into **Blocks** (pages).
- **Efficiency**: Reduces memory fragmentation by **60-80%**, allowing for significantly larger batch sizes and longer context windows on the same hardware.

> **Why (the rationale):** Contiguous KV cache allocation wastes GPU memory because different requests have different lengths, leaving large unused gaps. PagedAttention treats GPU memory like OS virtual memory — pages are allocated on demand and reclaimed when freed, radically improving utilization.
> **When to use:** Any self-hosted inference setup serving more than a handful of concurrent requests. PagedAttention is the default in vLLM; if you are using a managed API (OpenAI, Anthropic), this is handled server-side — you benefit without configuring it.
> **Nuances & gotchas:** PagedAttention improves throughput and memory efficiency but does not reduce the per-token compute cost of attention itself — it does not make long contexts cheaper to process, only more feasible to serve concurrently. It is also specific to the inference server; application-level context management decisions (what tokens to include) are orthogonal and equally important.

---

## Prefix Caching

This is the **Holy Grail of Latency** for any production LLM stack.
- **The Problem**: Every time an agent calls an LLM, it sends the same 2,000-token System Prompt + 50 Tool Schemas. This wastes compute.
- **The Solution**: **Persistent Prefix Caching**. The server keeps the KV cache for the "Static" part of the prompt (the prefix) in memory.
- **Result**: You only pay for (and wait for) the compute on the *new* part of the message.

> **Why (the rationale):** In high-throughput agents with large system prompts and tool schemas, the static prefix can be 50-80% of total input tokens. Without caching, every call re-computes the same KV values — wasting GPU cycles and adding latency proportional to prefix length.
> **When to use:** Any production agent with a fixed system prompt or large tool schema. The savings are proportional to the prefix-to-total-prompt ratio; short system prompts see less benefit. Both Anthropic and OpenAI offer server-side prompt caching; for self-hosted models, configure vLLM's prefix-caching flag.
> **Nuances & gotchas:** The cache is only valid if the prefix is byte-for-byte identical across requests. Inserting dynamic content (current date, user name, session ID) anywhere before the static block breaks the cache entirely. A common mistake is putting a personalized greeting at the top of the system prompt — this invalidates every user's cache and negates the benefit.

---

## Sliding Windows vs. Summarization

| Method | Mechanics | Pro | Con |
|--------|-----------|-----|-----|
| **Sliding Window** | Keep last N tokens exactly. | High fidelity for recent. | "Dory" effect (forgets start). |
| **Summarization** | Compress old turns into text. | Preserves "Key Facts". | Loses nuance/formatting. |
| **Hybrid** | Keep last 10 turns + 1 summary. | Best of both worlds. | Slightly higher complexity. |

> **Why (the rationale):** Both pure approaches fail in opposite directions: a sliding window amputates early context the model may need, while pure summarization discards detail that cannot be recovered. The hybrid keeps verbatim recency (where precision matters most) and compressed history (where gist is enough).
> **When to use:** Use a sliding window for task-focused agents where only the immediate plan matters. Use summarization for customer-support or coaching agents where early context (user's stated goal, initial complaint) must survive a long conversation. Default to hybrid for most production agents.
> **Nuances & gotchas:** Summarization adds an extra LLM call per compression cycle — this is a cost and latency hit that compounds in high-volume systems. The summarizer can hallucinate or omit facts, and there is no reliable way to detect this. Summaries are also lossy for structured data (tool output JSON, code snippets) — consider storing those in L2 and pointing to them rather than summarizing inline.

---

## Contextual Compression

Current frontier models support **Prompt Hardening**.
- **Selective Dropping**: Automatically stripping out irrelevant "Thought" blocks from previous turns to save space.
- **Token Pruning**: Using a smaller model to rewrite a long user message into a 50% shorter, equivalent prompt before sending it to the "Reasoning" model.

> **Why (the rationale):** Agent "thought" traces and verbose tool outputs can dwarf the actual useful signal in the context. Selective dropping reclaims those tokens cheaply (rule-based, no extra LLM call) so they can be filled with high-value content.
> **When to use:** When your agent uses chain-of-thought or scratchpad reasoning in previous turns — these thought blocks are only needed by the agent at the time they are generated and should be stripped before the next turn. Token pruning is most justified when users frequently paste long documents or code that contains much boilerplate irrelevant to their actual question.
> **Nuances & gotchas:** Token pruning introduces a second model in the hot path — if the rewriter model is slow or introduces errors, you pay in both latency and quality. The rewriter can subtly alter the meaning of a query (e.g., dropping a negation), which causes incorrect downstream reasoning with no visible failure signal. Always keep a verbatim copy of the original user message in logs for debugging.

---

## Interview Questions

### Q: What is the difference between "Model Context Window" and "Application Context Window"?

**Strong answer:**
The **Model Context Window** is the hard limit defined by the architecture (e.g., 128K for GPT-4o). The **Application Context Window** is a configuration set by the engineer (e.g., 16K limit) to manage **Latency and Cost**. In production, we rarely use the full model window for every turn because attention overhead increases with context size, leading to slower generation. We use a **Buffer Zone** to leave space for the model's new response.

### Q: How does "Prefix Caching" change how you design System Prompts?

**Strong answer:**
It forces me to move **Static content to the front** and **Dynamic content to the back**. Early-LLM patterns often put the user's name or date at the very top. That breaks the prefix cache. I put "Immutable Rules" and "Tool Schemas" at the beginning and the "User Context" (which changes every turn) at the end. This ensures the first 5,000 tokens are identical across all users, maximizing cache hits on the inference server.

---

## References
- vLLM Team. "PagedAttention: Software-Defined Memory for LLM Serving" (2024/2025)
- NVIDIA. "Optimizing Inference with TensorRT-LLM" (2025)
- Anthropic. "Prompt Caching: Scale while reducing costs" (2024/2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **KV Cache** | Pre-computed key-value attention matrices stored on GPU memory so unchanged tokens are not reprocessed on each call | The primary mechanism for reducing latency and compute cost in repeated or shared-prefix requests |
| **Dynamic Context Allocation** | Adjusting how many tokens are reserved for different parts of the prompt (history, tools, system prompt) based on real-time conditions | Maximises useful information within a fixed token budget |
| **PagedAttention** | An inference technique (used in vLLM) that stores KV cache in non-contiguous memory pages like an OS pager | Cuts GPU memory fragmentation by 60-80%, enabling larger batches and longer contexts on the same hardware |
| **vLLM** | An open-source LLM inference server that implements PagedAttention for efficient GPU memory use | The most widely used high-throughput inference engine for self-hosted models |
| **TensorRT-LLM** | NVIDIA's optimised inference library that compiles LLM models into highly efficient GPU kernels | Used in production for maximum throughput and minimum latency on NVIDIA hardware |
| **Prefix Caching** | Storing and reusing the KV cache for the static leading portion of every prompt (e.g., system prompt and tool schemas) | Eliminates redundant compute on tokens that are identical across requests |
| **Static Prefix** | The part of a prompt that never changes between requests, such as rules or tool definitions | Should always appear first in the prompt so its KV cache can be shared across all users |
| **Dynamic Content** | The part of the prompt that differs per request or user, such as recent message history | Should be placed at the end of the prompt to avoid invalidating the cached prefix |
| **Sliding Window** | Keeping only the last N tokens of conversation history and discarding older tokens | Maintains a constant context size with zero compression overhead, at the cost of forgetting early turns |
| **Summarisation** | Compressing older conversation turns into a concise text block using an LLM | Preserves key facts from early turns while freeing tokens for new content |
| **Hybrid Context Strategy** | Combining a sliding window for recent turns with a summarised block for older history | Balances fidelity of recent content with retention of important earlier facts |
| **Contextual Compression** | Automatically removing low-value content (e.g., agent thought traces) from the context before each call | Frees tokens for content that actually affects the model's response quality |
| **Token Pruning** | Using a smaller model to rewrite a long user message into a shorter semantically equivalent version before sending it to the main model | Cuts input token cost while preserving the user's intent |
| **Prompt Hardening** | Structuring prompts so they resist token bloat and cache invalidation as conversations grow | Keeps latency and cost stable across long sessions |
| **Model Context Window** | The hard architectural limit on how many tokens an LLM can process in one call (e.g., 128K for GPT-4o) | Sets the absolute ceiling; engineers must manage usage within this limit |
| **Application Context Window** | The engineer-configured token budget, typically smaller than the model limit, used in production | Controls latency and cost by preventing over-large prompts even when the model could handle more |
| **Buffer Zone** | A portion of the context window deliberately left empty to accommodate the model's output tokens | Prevents the model from being cut off mid-response due to a full context |
| **Attention Overhead** | The compute cost of self-attention, which grows quadratically with context length | The core reason that using fewer tokens is almost always faster and cheaper |
| **Eviction** | Removing older tokens from the active context to make room for new ones | The mechanism that enforces context window limits during long conversations |
| **Memory Fragmentation** | Wasted GPU memory gaps that arise when attention tensors are allocated as contiguous blocks | PagedAttention was invented to eliminate this and increase effective memory utilisation |

*Next: [Long-Term Memory](03-long-term-memory.md)*
