# FinOps and Token Economics

This chapter is about the **economics and discipline** of running LLMs at scale: how to model, attribute, budget, and structurally reduce AI spend. It is not a rehash of inference internals; for the tactical levers (quantization, batching, speculative decoding, KV cache) see [Cost Optimization Playbook](../04-inference-optimization/07-cost-optimization-playbook.md) and the [inference chapters](../04-inference-optimization/01-inference-fundamentals.md).

The anchor finding that frames the whole chapter, from Datadog's 2026 State of AI Engineering: **system prompts are about 69% of input tokens, yet only about 28% of calls use prompt caching.** The single largest, lowest-effort cost lever in most production stacks is sitting unused. AI products also run as a cost-of-goods business, not zero-marginal-cost SaaS, so margin thinking is now an engineering concern.

## Table of Contents

- [The Cost Model](#the-cost-model)
- [Caching: The Top Cost Lever](#caching-the-top-cost-lever)
- [Batch and Async Economics](#batch-and-async-economics)
- [The FinOps Discipline](#the-finops-discipline)
- [Structural Cost Decisions](#structural-cost-decisions)
- [Cost Anti-Patterns](#cost-anti-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Cost Model

Pricing is quoted per million tokens, split input versus output, and **output costs materially more than input**, commonly 3-5x and sometimes more, because generation is autoregressive and compute-bound while input is a parallel prefill pass. (Current per-model prices live in [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md); they deflate fast, so model the *structure*, not the cents.)

Every request decomposes into stacked spend layers. Modeling each separately is what makes cost predictable and attributable:

| Layer | Driven by | Behavior | Primary lever |
|-------|-----------|----------|---------------|
| System prompt / instructions | Fixed scaffolding, tool defs, few-shot | ~69% of input tokens; paid every call if uncached | Prompt caching |
| Retrieved / context tokens | RAG chunks, injected docs, long context | Scales with k and chunk size; can dwarf everything | RAG vs long context; chunk budgeting |
| Conversation / memory | Chat history, agent scratchpad | Grows unbounded without summarization | Windowing, summarization, compaction |
| Model tier | Frontier vs mid vs small/self-host | 10-100x spread across tiers | Right-sizing, cascades, routing |
| Output length | Verbosity, format, max_tokens | Billed at the higher output rate | `max_tokens` caps, terse output contracts |
| Reasoning / thinking tokens | Extended-thinking modes | Billed at output rate, invisible in the response | Gate thinking by task complexity |
| Retry / overhead | Transient errors, guardrail re-runs | Multiplies on failure | Bounded retries, circuit breakers |
| Agent multi-step | Plan-act-observe loops, sub-agents | Multiplies the whole stack per step | Step ceilings, per-run budgets |

**Reasoning models and agents are cost multipliers, and the two worst because they are invisible.** Extended-thinking tokens are billed at the output rate but do not appear in the response, so a "short" call can cost an order of magnitude more than its visible output suggests; reported analyses put the multiplier anywhere from ~3x to ~15x depending on the task. Agents multiply the *entire* token stack on every step, so the right unit of measurement is **cost per task, not cost per call**. Reported bands: a chat turn is cents, while an agentic multi-step task can run from tens of cents to several dollars.

A teachable cost-per-request formula:

```
cost = Σ(layer_input_tokens × in_rate) + (output_tokens + reasoning_tokens) × out_rate
cost_per_task = cost_per_request × expected_steps × (1 + retry_rate)
```

with caching applied as a discount on the cacheable input fraction.

---

## Caching: The Top Cost Lever

This is the headline lever precisely because of the anchor stat: the layer that is ~69% of input tokens (the system prompt) is static and ideal for caching, yet only ~28% of calls cache it. The gap between potential and actual is the biggest, cheapest saving available.

**Provider prefix caching** lets you pay a steep discount on a repeated prompt prefix. The discounts are reported in the range of roughly 50% (OpenAI, automatic above a threshold, no write fee) to ~90% (Anthropic, explicit `cache_control` breakpoints with a small write premium that breaks even after a couple of reads) to ~75% (Google). The crucial caveat to state plainly when teaching: the headline "50-95% savings" applies to the **cached prefix only**, not the whole bill.

How to actually capture it (the discipline):
- **Order prompts static to dynamic.** Put system instructions, tool definitions, and few-shot examples first (the stable prefix), and the user query last. Exact-prefix matching means any change near the front invalidates the entire downstream cache.
- **Stabilize the prefix.** No timestamps, request IDs, or per-call nonces in the cached region; pin tool-definition ordering.
- **Watch the TTL economics.** Where there is a write premium, caching only pays once you clear a break-even number of reads, so bursty low-reuse traffic may not benefit.
- **Instrument cache-hit rate per call** as a first-class metric. Prompt rewrites, model version bumps, and reordering silently drop hit rate.

Distinct from prefix caching, **exact-match and semantic caching** serve whole responses for repeated *queries*. Exact-match keys on the literal request (cheap, zero false positives); semantic caching embeds the query and serves cached responses for similar hits (higher hit rate on natural-language traffic, but a false-hit risk worth guarding). Layer them: exact-match, then semantic, then prefix, and measure hit rate per layer. Conceptually this is the billing-layer monetization of the same KV reuse described in [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md).

---

## Batch and Async Economics

Both OpenAI and Anthropic offer a roughly **50% discount on batch processing** (input and output), asynchronous, with a completion ceiling around 24 hours. The decision rule is simple: **use batch whenever no human or system is waiting on the token.** High-value batch workloads include evaluation and regression suites, bulk classification and labeling, corpus-scale summarization and document processing, backfills after a prompt or model change, and A/B testing prompt variants. For the entire offline tier of a product, not batching leaves about half the money on the table.

The third lane is **provisioned/reserved throughput** (AWS Bedrock Provisioned Throughput, Azure OpenAI PTUs): reserved capacity at an hourly rate regardless of usage, reported to save on the order of 15-70% on sustained workloads, economical only at high, predictable utilization. The mental model mirrors cloud compute: pay-per-token (including batch) for spiky or uncertain demand, and reserved capacity once utilization is high and steady.

---

## The FinOps Discipline

The FinOps Foundation's framing: **inference is 80-90% of total GenAI spend** in many deployments, so the discipline centers on per-request inference economics, not training. The operational core:

- **Attribution.** Tag every call by team, feature, customer/tenant, model, route, and environment. The technical enabler is a token proxy or [gateway](03-ai-gateways-and-model-routing.md) in front of the API that identifies the source of each call. Without attribution there is no way to compute unit economics or see which use cases earn their cost.
- **Showback before chargeback.** Start with visibility dashboards (per-provider, per-model, per-team, per-tenant, with daily forecasts and spike alerts), then graduate to billing teams once the tags are trustworthy.
- **Unit economics.** Track cost per user, per conversation, per resolved ticket or case, and AI cost as a percentage of revenue and of gross margin. Teach an AI product like a cost-of-goods business.
- **Margin reality.** Reported snapshots put AI-product gross margins roughly 25-30 points below the 80-90% of traditional SaaS, because every request has a variable token cost. This is why **outcome-based pricing** (per resolved ticket, per completed task) is rising, with reported anchors like a fixed price per resolved support ticket. The imperative: know your cost per resolution before you price per resolution.
- **Tooling.** Gateways give real-time per-request control and spend caps; FinOps platforms (Helicone, Vantage, Finout, Amnic, and cloud cost tools) give cross-cloud allocation and chargeback. Mature stacks run both. Verify a tool's current status before standardizing on it; this category churns.

---

## Structural Cost Decisions

These architecture-level choices move cost by 2-50x, beyond per-call tuning:

- **Right-sizing, cascades, and routing.** Route to the cheapest model that clears a quality bar, and escalate only on low confidence. Reported savings of 45-85% at ~95% quality retention (FrugalGPT is the canonical reference), with the escalation rate as the live cost variable. See [AI Gateways and Model Routing](03-ai-gateways-and-model-routing.md).
- **Self-host vs API break-even.** The reported break-even against a frontier API sits in the high tens to hundreds of millions of tokens per month, but the load-bearing warning is hidden cost: raw GPU rental is only 30-40% of true cost, so apply a ~2.5-3x multiplier, and engineering labor often exceeds infrastructure. For most teams in 2026, managed APIs are cheaper once the full stack is counted; self-host wins at high, predictable, well-utilized volume or for data-residency reasons. See [LLM Infrastructure](01-llm-infrastructure.md).
- **RAG vs long context.** Retrieval is dramatically cheaper per query than stuffing a long context, since you pay for a few relevant chunks instead of a giant prompt. Long context wins for small static document sets; RAG wins for large or frequently changing corpora and high query volume. See [RAG Fundamentals](../06-retrieval-systems/01-rag-fundamentals.md).
- **Distillation.** Fine-tuning a small model to within a couple of accuracy points of a frontier model on a locked eval is reported to cut per-token cost by 5-40x, with payback in weeks to months at high volume; it wins on narrow, high-volume tasks and fails on open-ended long-tail work. See [Knowledge Distillation](../03-training-and-adaptation/05-knowledge-distillation.md) and the [distillation case study](../16-case-studies/19-customer-distillation-pipeline.md).
- **Output and prompt engineering.** Terse output contracts, `max_tokens` caps, structured outputs, and trimming few-shot examples once a model is reliable are reported to cut tokens 20-40% at minimal quality loss.

---

## Cost Anti-Patterns

| Anti-pattern | Mechanism | Fix |
|--------------|-----------|-----|
| No caching | Re-paying for the static system prompt (~69% of input) every call | Stable prefix plus provider prefix caching |
| Oversized model | Frontier model on tasks a small model handles | Right-size, cascade, route |
| Unbounded output | No `max_tokens`, verbose formats | Caps and terse output contracts |
| Reasoning on by default | Extended thinking for trivial tasks | Gate thinking by task complexity |
| Retry storms | Transient error triggers unbounded retries | Bounded retries and circuit breakers |
| Runaway agent loops | A plan-act loop never terminates; tool errors read as "retry" | Hard step, token, and retry ceilings inside the loop |
| Unbounded memory | History accrues without summarization | Windowing and summarization |
| Long-context stuffing | A giant context as the default retrieval | RAG for large or changing corpora |
| No attribution | Untagged shared spend | Gateway token proxy plus tags |
| Real-time for offline work | Sync API for evals, backfills, labeling | Batch API |

Reported real incidents make the agent-loop row concrete: runaway agents have burned tens of thousands of dollars over a single weekend before anyone noticed. Hard ceilings inside the loop, not after-the-fact alerts, are the defense.

---

## Interview Questions

### Q: Your LLM bill doubled month over month with flat traffic. How do you find and fix it?

**Strong answer:**
First, attribution: if every call is not tagged by feature, team, model, and route through a gateway or proxy, that is the first fix, because you cannot debug what you cannot see. With attribution I would break spend into the token-spend layers and look for the usual culprits: a prompt change that broke cache-hit rate (the system prompt is ~69% of input tokens, so a cache regression is huge), extended thinking switched on for simple tasks (billed at the output rate and invisible in the response), an agent loop whose step count crept up, unbounded output or conversation history, or a retry storm. The highest-ROI fix is almost always restoring prompt caching, then right-sizing the model and capping output. I would also move any offline work (evals, backfills) to the batch API for roughly half off, and set per-feature budgets with spike alerts so the next doubling pages someone on day one.

### Q: Why do AI products have worse gross margins than SaaS, and what do engineers do about it?

**Strong answer:**
Because every request carries a variable token cost, so an AI product behaves like a cost-of-goods business rather than zero-marginal-cost software; reported margins run roughly 25-30 points below typical SaaS. Engineers attack it on two fronts. Structurally: cache the static prompt prefix, right-size and cascade models, prefer RAG to long-context stuffing, distill high-volume narrow tasks onto a small model, and batch the offline tier. Operationally: instrument unit economics (cost per conversation, per resolved outcome) so pricing can move toward outcome-based models, which only works if you know your cost per resolution. The cost levers are an engineering responsibility, not just a finance one.

---

## References

- Datadog, [State of AI Engineering 2026](https://www.datadoghq.com/state-of-ai-engineering/)
- FinOps Foundation, [Optimizing GenAI Usage](https://www.finops.org/wg/optimizing-genai-usage/) and [FinOps for AI](https://www.finops.org/wg/finops-for-ai-overview/)
- Anthropic, [prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching); OpenAI, [prompt caching](https://platform.openai.com/docs/guides/prompt-caching)
- Anthropic, [Message Batches API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing); OpenAI, [Batch API](https://platform.openai.com/docs/guides/batch)
- Chen et al., "FrugalGPT" arXiv:2305.05176
- Bessemer, [the AI pricing and monetization playbook](https://www.bvp.com/atlas/the-ai-pricing-and-monetization-playbook)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **FinOps** | A discipline that combines financial accountability with cloud/AI operations to optimize spend | Brings cost visibility and control to AI infrastructure the same way it does for cloud compute |
| **Token Economics** | The study of how LLM costs accumulate per token and how to structure systems to minimize spend | Enables engineers to treat AI inference as a cost-of-goods item and optimize accordingly |
| **Input Tokens** | The tokens sent to the model as part of the prompt, including system instructions, context, and user query | The portion of cost you control through prompt design, caching, and retrieval strategy |
| **Output Tokens** | The tokens the model generates in its response | Typically cost 3-5x more than input tokens and are driven by verbosity and max_tokens settings |
| **Prompt Caching** | A provider feature that stores the result of processing a repeated prompt prefix so subsequent calls pay a discounted rate | The single highest-ROI cost lever for most production stacks because the system prompt (~69% of input) is static |
| **Prefix Caching** | Another name for provider-level prompt caching; the stable leading portion of a prompt is cached and reused | Reduces input token cost by 50-90% on the cached prefix for subsequent calls |
| **Cache-Hit Rate** | The fraction of requests that successfully reuse a cached prompt prefix or response instead of re-computing | The metric that directly determines how much caching actually saves in practice |
| **System Prompt** | Fixed instructions prepended to every request defining the model's behavior and persona | Typically the largest single token cost layer (~69% of input); the primary target for prefix caching |
| **RAG (Retrieval-Augmented Generation)** | A pattern that fetches a small set of relevant documents and injects them into the prompt at query time | Dramatically cheaper per query than stuffing a full document corpus into a long context window |
| **Long Context** | Using a very large context window to include many documents or a long conversation history in a single prompt | Convenient but expensive; best reserved for small, static document sets with infrequent queries |
| **Autoregressive Generation** | The way LLMs generate output one token at a time, where each new token depends on all previous ones | Explains why output is compute-bound and costs more per token than the parallel input prefill pass |
| **Extended Thinking / Reasoning Tokens** | Hidden chain-of-thought tokens some models generate internally before producing the visible response | Billed at the output rate but invisible in the reply; can multiply apparent cost 3-15x on complex tasks |
| **Agentic Loop / Multi-Step** | An AI agent that repeatedly plans, calls tools, observes results, and plans again to complete a task | Multiplies the entire token stack on every step, making cost per task the right unit of measurement |
| **Batch API** | An asynchronous API mode where requests are queued and processed with a ~50% discount within ~24 hours | The correct choice for any offline work (evals, backfills, labeling) where latency is not critical |
| **Provisioned Throughput** | Reserved model capacity billed at an hourly rate regardless of usage (e.g., AWS Bedrock PTUs, Azure PTUs) | Cost-effective at high, predictable utilization; mirrors the economics of reserved cloud compute instances |
| **Cascade / Routing** | A strategy that sends requests to the cheapest model first and escalates to a stronger model only when needed | Reported to achieve ~95% quality at 45-85% cost reduction (FrugalGPT / RouteLLM benchmarks) |
| **Right-Sizing** | Choosing the least powerful (cheapest) model that still meets quality requirements for a given task | Often the highest-leverage cost reduction move beyond caching |
| **Distillation** | Fine-tuning a small, cheap model on outputs from a large frontier model for a specific narrow task | Can reduce per-token cost 5-40x on high-volume, well-defined tasks where a small model can match frontier quality |
| **max_tokens** | A parameter that caps the maximum length of the model's generated response | Prevents runaway verbose outputs and directly bounds per-request output cost |
| **Cost per Task** | Total token cost across all steps and retries required to complete one end-to-end agent task | The correct unit of measurement for agentic workloads where cost per call understates true spend |
| **Unit Economics** | Per-unit cost and revenue metrics such as cost per user, cost per conversation, or cost per resolved ticket | Necessary to determine whether an AI feature is profitable and to set outcome-based prices |
| **Outcome-Based Pricing** | Charging customers per completed outcome (e.g., per resolved support ticket) rather than per seat or per call | Aligns pricing with value delivered and requires knowing your cost per resolution before setting the price |
| **Showback** | Reporting AI spend attribution to teams without actually billing them for it | First step toward chargeback; builds trust in cost data before enforcement |
| **Chargeback** | Billing internal teams or tenants for their actual AI consumption | Drives cost-conscious behavior and enables unit-economics tracking per team or product |
| **Retry Storm** | A situation where many clients simultaneously retry a failing provider, worsening the outage | A top cost anti-pattern; prevented by bounded retries, exponential backoff, and circuit breakers |
| **Token Proxy / Gateway** | A service that sits in front of the LLM API to tag, count, and attribute every token call | The technical enabler for spend attribution, budget enforcement, and showback/chargeback |
| **FrugalGPT** | A 2023 Stanford paper (arXiv:2305.05176) proposing cascade routing to match frontier quality at reduced cost | The canonical academic reference for cost-quality trade-off via model routing and cascades |
| **Windowing / Summarization** | Truncating or compressing conversation history to keep it within a fixed token budget | Prevents unbounded memory growth that would otherwise make long conversations exponentially expensive |

---

*Previous: [AI Gateways and Model Routing](03-ai-gateways-and-model-routing.md)*
