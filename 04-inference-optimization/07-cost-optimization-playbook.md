# Cost Optimization Playbook

AI costs are no longer "magic." They are measurable, predictable, and highly optimizable. With API pricing down 30-60% over the past year, the cost lever is now mostly about *routing* and *caching*, not just picking a cheaper provider. This chapter covers the strategies to reduce inference costs by 10x without sacrificing quality.

## Table of Contents

- [The Unit Economics of AI](#unit-economics)
- [Model Cascading (Efficiency Tiers)](#model-cascading)
- [Small Language Models (SLMs)](#slms)
- [Spot Instance Strategies](#spot-instances)
- [The "Token Tax" Optimization](#token-tax)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Unit Economics of AI

We measure success by **Tokens per Dollar ($)**.

| Component | Cost Driver | Optimization |
|-----------|-------------|--------------|
| **Compute** | GPU Time ($/hr) | Better utilization (Batching). |
| **VRAM** | KV Cache Size | GQA, Quantization. |
| **Network** | Payload Size | Compression, Local serving. |
| **API** | Per-token pricing | Caching, Model selection. |

---

## Model Cascading (Efficiency Tiers)

The most effective cost-saving strategy is to use the **cheapest model capable of the task.**

**The cascade pattern:**
1. **Classifier**: A tiny model (0.5B) determines query complexity ($0.00).
2. **Tier 1 (SLM)**: 90% of queries (greetings, simple Q&A) go to an 8B model ($).
3. **Tier 2 (Frontier)**: 9% of queries (complex reasoning) go to a 405B/Claude Sonnet 4.6 / GPT-5.5 / Gemini 3.1 Pro tier model ($$$).
4. **Tier 3 (Reasoning)**: 1% of queries (expert-level) go to thinking models like Claude Opus 4.7 or GPT-5.5 with extended thinking ($$$$$).

**Net result**: 80% cost reduction vs. sending all traffic to Tier 2.

---

## Small Language Models (SLMs) for Production

3B-8B models (Llama 4 8B, Gemini 3.1 Flash, Claude Haiku 4.5) now match or beat the original GPT-4 from 2023 on most benchmarks.
- **Use Case**: Entity extraction, sentiment analysis, simple RAG.
- **Cost**: 100x cheaper to run than frontier models.
- **Latency**: < 100ms response times.

### The DeepSeek V4 Floor

DeepSeek V4 Flash (released April 24, 2026) reset the floor for cheap frontier-class inference at **$0.14 / $0.28 per 1M tokens** with a 1M context window and cache-hit input at $0.0028/M. DeepSeek V4 Pro is roughly 10x cheaper than Claude Opus 4.7 ($0.435 / $0.87 vs $5 / $25 per 1M) after the 75% discount was made permanent on May 22, 2026. For cache-heavy, high-volume workloads where the prefix is reused often (RAG with shared knowledge bases, batch classification, codebase agents), V4 Flash or V4 Pro is now the dominant cost-optimization lever before you even start cascading. Verify on the [DeepSeek pricing page](https://api-docs.deepseek.com/quick_start/pricing) before committing.

---

## Spot Instance Strategies

For non-real-time workloads (batch processing, data extraction), use **GPU Spot Instances** (AWS Spot, Azure Spot, Lambda Labs).

- **Risk**: GPU can be reclaimed with 30-sec notice.
- **Mitigation**: **Live KV-Cache Migration**. Serving frameworks can stream the KV cache of ongoing requests to another node as soon as the "Reclamation Signal" is received, ensuring no work is lost.

---

## The "Token Tax" Optimization

- **System Prompt Caching**: Hard-code common prefixes to get 90% discounts.
- **Output Truncation**: Strictly limit `max_tokens`.
- **Negative Prompting**: "Don't be wordy" saves ~15% in output tokens (and thus cost).

---

## Interview Questions

### Q: How do you justify the cost of an AI system to a CFO?

**Strong answer:**
I focus on the **ROI of Efficiency.** First, I implement "Model Cascading" to ensure that 90% of our traffic is handled by sub-cent-per-million-token models. Second, I implement "Semantic Caching" to prevent paying for the same answer twice. Third, I set up "Inference Quotas" and "Chargeback Models" so each business unit is accountable for their usage. By treating AI as a "Commodity Resource" with tiered pricing, we can transition from "unbounded experimentation" to a "predictable OpEx" model.

### Q: When is a self-hosted individual GPU cluster cheaper than an API?

**Strong answer:**
The "Crossover Point" usually happens at **constant high throughput.** If your application has a baseline of 5-10 requests per second, 24/7, the fixed cost of an H100 reservation becomes cheaper than the variable token cost of an API. However, if your traffic is "spiky" or heavily weighted toward business hours, API providers are usually cheaper because they allow you to "pay for the silence" during off-peak hours. For most enterprises, the break-even is around 500 million tokens per month for a 70B-tier model.

---

## References
- Google Cloud. "Cost Optimization for Generative AI" (2024)
- Anyscale. "LLM Inference: API vs. Self-Hosted Costs" (2024)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Unit Economics** | The per-unit cost and revenue of a service — here, cost per token or per request | Frames AI spending as a measurable, optimizable metric rather than a vague infrastructure cost |
| **Tokens per Dollar** | The number of tokens generated for every dollar spent on compute or API fees | The primary cost-efficiency metric for AI systems; higher is better |
| **Model Cascading** | Routing each query to the cheapest model tier that can handle it, from a tiny classifier up to frontier reasoning models | Reduces total AI spend by 80%+ by avoiding expensive models for simple tasks |
| **Classifier Model** | A tiny (0.5B) model used to estimate the complexity of an incoming query before routing it | The first stage of a cascade; its job is to sort traffic cheaply, not to answer questions |
| **SLM (Small Language Model)** | A 3B–8B parameter model that matches older frontier-model quality at a fraction of the cost | Handles the majority of production queries (greetings, simple Q&A) at sub-cent cost |
| **Frontier Model** | A top-tier large model (e.g., Claude Sonnet 4.6, GPT-5.5, Gemini 3.1 Pro) for complex tasks | Used for the small fraction of queries requiring strong reasoning or knowledge |
| **Reasoning / Thinking Model** | A model that generates extended internal reasoning steps before answering (e.g., Claude Opus 4.7 with extended thinking) | Reserved for the hardest 1% of queries; most expensive tier in the cascade |
| **Semantic Caching** | Storing and reusing LLM responses for queries that are semantically similar, not just exact matches | Prevents paying for the same answer twice even when users phrase the same question differently |
| **Prompt Caching** | A provider feature that stores the KV cache of repeated prompt prefixes and charges a discounted rate for cache hits | Cuts input token cost by 50–90% for long system prompts or shared document prefixes |
| **System Prompt Caching** | Hard-coding common prefix text (instructions, tool schemas) so its KV cache is reused across requests | The simplest way to capture prompt caching discounts — no application logic changes needed |
| **Output Truncation** | Setting a strict `max_tokens` limit on responses to prevent unnecessarily long outputs | Directly reduces output token cost; output tokens are typically 3–5x more expensive than input tokens |
| **Negative Prompting** | Instructing the model to be concise (e.g., "don't be wordy") to reduce output length | A prompt-engineering trick that saves ~15% in output tokens with minimal impact on answer quality |
| **GPU Spot Instances** | Cloud GPU capacity sold at steep discounts (50–90% off) but subject to reclamation with short notice | Dramatically reduces compute cost for batch, non-real-time workloads that can tolerate interruption |
| **Live KV-Cache Migration** | Streaming the KV cache of an in-flight request to another GPU node before the current one is reclaimed | Allows spot instance cost savings without losing user sessions when a GPU is preempted |
| **Reclamation Signal** | The ~30-second warning a cloud provider sends before taking back a spot GPU instance | The trigger for live KV-cache migration to preserve ongoing request state |
| **Token Tax** | Unnecessary tokens in prompts or outputs that add cost without adding value | The target of output truncation, negative prompting, and system prompt caching optimizations |
| **Crossover Point** | The usage volume at which self-hosting a GPU becomes cheaper than paying per-token API fees | Typically around 500M tokens/month for a 70B-tier model at constant high throughput |
| **OpEx (Operational Expenditure)** | Recurring costs of running a service, as opposed to one-time capital purchases | AI cost optimization aims to make AI OpEx predictable and proportional to business value |
| **Chargeback Model** | Attributing AI compute costs back to the business unit or product that generated them | Creates accountability for usage and surfaces which teams or features consume the most AI spend |
| **Inference Quota** | A hard or soft limit on how many tokens a team or user can consume per period | Prevents runaway costs from unconstrained experimentation or bugs |
| **DeepSeek V4 Flash** | A very cheap frontier-class model ($0.14/$0.28 per 1M tokens) with a 1M context window | The lowest-cost option for cache-heavy, high-volume workloads like batch classification and codebase RAG |
| **Break-Even (self-hosted vs. API)** | The request rate above which owning GPU hardware beats paying per-token API pricing | Typically 5–10 requests per second sustained 24/7, or roughly 500M tokens per month for a 70B model |

*Next: [Diffusion Language Models](08-diffusion-llms.md)*
