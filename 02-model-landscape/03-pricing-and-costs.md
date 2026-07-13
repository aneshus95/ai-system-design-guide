# Pricing and Costs

Understanding the cost structure of LLM systems is essential for production planning. This chapter covers pricing models, cost optimization strategies, and total cost of ownership analysis.

## Table of Contents

- [Pricing Models](#pricing-models)
- [Current API Pricing](#current-api-pricing)
- [Cost Calculation](#cost-calculation)
- [Cost Optimization Strategies](#cost-optimization-strategies)
- [Context Caching Economics](#context-caching-economics)
- [Self-Hosting & GPU Cloud Arbitrage](#self-hosting-economics)
- [Total Cost of Ownership](#total-cost-of-ownership)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Pricing Models

### Token-Based Pricing

Most LLM APIs charge per token:

```
Cost = (input_tokens × input_rate) + (output_tokens × output_rate)
```

**Key observations:**
- Output tokens cost 2-5x more than input tokens
- Pricing varies significantly by model tier
- Some providers offer batch discounts

### Tiered Pricing

Some providers offer volume discounts:

| Tier | Monthly Spend | Discount |
|------|---------------|----------|
| Standard | $0 - $5K | 0% |
| Growth | $5K - $50K | 10-20% |
| Enterprise | $50K+ | Custom negotiation |

### Commitment-Based Pricing

Pre-purchase tokens at discounted rates:

```
Standard: $2.50 / 1M input tokens
Committed (1-year): $2.00 / 1M input tokens (20% savings)
```

---

## Current API Pricing

### May 2026 Pricing

> **Last verified: June 10, 2026.** Prices change frequently. Always re-check: [OpenAI](https://developers.openai.com/api/docs/pricing), [Anthropic](https://platform.claude.com/docs/en/about-claude/pricing), [Google](https://ai.google.dev/gemini-api/docs/pricing), [xAI](https://docs.x.ai/developers/models), [DeepSeek](https://api-docs.deepseek.com/quick_start/pricing)
>
> **Deprecations effective in 2026:** OpenAI retired GPT-4o, GPT-4.1, GPT-4.1-mini, o4-mini from ChatGPT on Feb 13, 2026; gpt-5.2-chat-latest and gpt-5.3-chat-latest deprecated May 8, 2026; Realtime API Beta removed May 12, 2026; Sora app shut down April 26, 2026 (API EOL Sep 24, 2026). Anthropic retires Claude Sonnet 4 and Claude Opus 4 on June 15, 2026, and Claude Opus 4.1 on August 5, 2026. Google Vertex retired `gemini-3-pro-preview` Mar 26, 2026; Project Mariner shut down May 4, 2026. Gemini 2.5 Pro/Flash deprecated June 17, 2026.
>
> **Price moves:** Anthropic released **Claude Fable 5** on June 9, 2026 at $10 / $50 per 1M: its most capable widely released model (Mythos-class with safeguards), priced at 2x Opus 4.8 but less than half of Claude Mythos Preview. **Claude Mythos 5** (same model, safeguards lifted, Glasswing-only) shares the $10 / $50 price. Anthropic released **Claude Opus 4.8** on May 28, 2026 at the same $5 / $25 per 1M as Opus 4.7, with an optional fast mode at $10 / $50 per 1M (about 2.5x faster and 3x cheaper than the Opus 4.7 fast mode, which was $30 / $150). DeepSeek made its 75% V4 Pro discount **permanent** on May 22, 2026: from June 1, 2026 the new list price drops to 25% of the original ($0.435 / $0.87 per 1M input/output), and the cache-hit input price for all DeepSeek models was cut to 1/10 of the launch price on April 26, 2026. DeepSeek V4 Flash ($0.14 / $0.28 per 1M, 1M context) is the cheapest frontier-class API by a wide margin.

#### OpenAI (GPT-5.x Generation)
| Model | Input / 1M | Output / 1M | Notes |
|-------|------------|-------------|-------|
| **GPT-5.5** ⭐ NEW | $5.00 | $30.00 | Released April 23, 2026. 1M context. New class of multimodal flagship. |
| **GPT-5.5 Instant** ⭐ NEW | check latest | check latest | Default in ChatGPT and `chat-latest` since May 5, 2026. 52.5% fewer hallucinations on high-stakes prompts. |
| **GPT-Realtime-2** ⭐ NEW | $32.00 (audio) | $64.00 (audio) | Released May 7, 2026. GPT-5-class realtime voice. |
| **GPT-Realtime-Translate** ⭐ NEW | (audio pricing) | (audio pricing) | 70+ input → 13 output languages. |
| **GPT-5.4 Pro** | $30.00 | $180.00 | Maximum reasoning; long-context doubles to $60/$270 |
| **GPT-5.4** | $2.50 | $15.00 | Flagship; native computer use; cached input $1.25 |
| **GPT-5.4-mini** | $0.75 | $4.50 | Best cost/performance in GPT-5 tier |
| **GPT-5.4-nano** | check latest | check latest | Smallest GPT-5.4 variant; released March 2026 |
| **GPT-4o** | $2.50 | $10.00 | Retired from ChatGPT Feb 13, 2026; API access varies |
| **GPT-4o-mini** | $0.15 | $0.60 | Legacy; check API availability |

#### Anthropic (Claude Fable + 4.x Generation)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Claude Fable 5** ⭐ NEW | $10.00 | $50.00 | 1M | Released June 9, 2026 (`claude-fable-5`) on Claude API, Claude Platform on AWS, Bedrock, Vertex AI, Microsoft Foundry. Most capable widely released Anthropic model (Mythos-class with safeguards; sensitive queries fall back to Opus 4.8 in under 5% of sessions). Adaptive thinking always on; 128K max output; 30-day data retention applies. |
| **Claude Mythos 5** ⭐ NEW | $10.00 | $50.00 | 1M | Same underlying model as Fable 5 with safeguards lifted in some areas. Limited availability: Project Glasswing partners and select biology researchers. Succeeds Mythos Preview at less than half its price. |
| **Claude Opus 4.8** | $5.00 | $25.00 | 1M | Released May 28, 2026 on API, Bedrock, Vertex AI. Dynamic Workflows research preview with parallel subagents. Optional fast mode at $10 / $50 per 1M (about 2.5x faster, 3x cheaper than the Opus 4.7 fast mode). SWE-bench Verified 88.6%; SWE-Bench Pro 69.2%; OSWorld-Verified 82.3%. |
| **Claude Opus 4.7** | $5.00 | $25.00 | 1M | Released April 16, 2026 on API, Bedrock, Vertex, Microsoft Foundry. Higher-resolution vision, improved SWE. Fast mode: $30 / $150 per 1M. |
| **Claude Opus 4.6** | $5.00 | $25.00 | 1M | 128K max output; adaptive thinking at standard rates. |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | 1M | Covers most Opus-level tasks at lower cost. **Still no Sonnet 4.8 as of June 10, 2026.** |
| **Claude Haiku 4.5** | $1.00 | $5.00 | 200K | Fastest Anthropic model; cache hit input $0.10 / 1M. |
| **Claude Mythos Preview** | n/a | n/a | - | Restricted research preview (~11 Glasswing partners); succeeded by Claude Mythos 5 on June 9, 2026. |

> [!NOTE]
> **Claude 1M context at standard pricing**: Fable 5, Opus 4.8, Opus 4.7, Opus 4.6, and Sonnet 4.6 include the full 1M token context window at standard rates with no premium tier for long context. Batch API offers a 50% discount. Cache hits cost 10% of the standard input price. Fast mode pricing on Opus 4.8 ($10 / $50 per 1M) and Opus 4.7 / 4.6 ($30 / $150 per 1M) stacks with caching multipliers but is not available on the Batch API or Claude Platform on AWS. No Fable-tier fast mode at launch.

#### Google (Gemini 3.x Generation)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Gemini 3.1 Pro** | $2.00 | $12.00 | 1M | 200K+ context: $4.00/$18.00 |
| **Gemini 3.1 Flash** | $0.10 | $3.00 | 1M | Best price/performance; high-volume |
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | 1M | Deprecated June 2026 |

> [!WARNING]
> **Gemini 2.5 deprecation**: Gemini 2.5 Pro and 2.5 Flash are scheduled for deprecation on June 17, 2026. Migrate to Gemini 3.x models.

#### xAI (Grok)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Grok 4** | $3.00 | $15.00 | 256K | Native tool use; real-time search |
| **Grok 4.1 Fast** | $0.20 | $0.50 | 2M | High-volume, low-cost |
| **Grok 3 mini** | check latest | check latest | - | Faster, less accurate |

#### Open-Weight Models via API (May 2026)
| Model | Input / 1M | Output / 1M | Context | Provider Examples |
|-------|------------|-------------|---------|-------------------|
| **DeepSeek-V3.2** | $0.28 | $0.42 | 128K | DeepSeek API. 98% cache-hit discount. Effective rates can drop 10–30× via routing. |
| **DeepSeek V4 Pro** ⭐ NEW | $0.435 | $0.87 | 1M | DeepSeek API. 75% promotional discount made **permanent**: from June 1, 2026 the new list price is 25% of the original ($1.74 / $3.48). Cache-hit input: $0.003625/M. ~27% compute / 10% memory of V3.2 at 1M tokens. |
| **DeepSeek V4 Flash** ⭐ NEW | $0.14 | $0.28 | 1M | DeepSeek API. Cache-hit input: $0.0028/M (98% discount). 13B-active MoE. Currently the cheapest frontier-class 1M-context API. |
| **Mistral Medium 3.5** ⭐ NEW | $1.50 | check latest | 256K | Mistral API. Unified chat/reasoning/coding/vision; 77.6% SWE-Bench Verified. |
| **Kimi K2.6** ⭐ NEW | check latest | check latest | - | Moonshot API. 1T MoE / 32B active; agent swarm to 300 sub-agents. |
| **Qwen 3.6-35B-A3B** ⭐ NEW | check latest | check latest | - | Apache 2.0 weights; self-host or via API providers. |
| **Llama 4 Scout** | $0.11 | $0.34 | 10M | Together AI, Groq, Fireworks. Note: effective context degrades fast past 32K. |
| **Llama 4 Maverick** | $0.27 | $0.85 | 1M | Together AI, Groq, Fireworks. MoE-aware serving required. |
| **DeepSeek-V3** | $0.25 | $1.10 | 128K | DeepSeek API, Together AI |
| **DeepSeek-R1** | $0.55 | $2.19 | 128K | DeepSeek API |
| **Mistral Large 3** | $0.50 | $1.50 | 256K | Mistral API, AWS Bedrock |
| **Llama 3.3 70B** | ~$0.10–0.20 | ~$0.30–0.60 | 128K | Groq, Together AI |
| **Qwen2.5-Coder-32B** | ~$0.50 | ~$1.00 | 32K | Together AI |
| **Gemma 4 (31B / 26B-A4B MoE / E4B / E2B)** ⭐ NEW | self-host | self-host | 256K | Apache 2.0. 140+ languages; native vision/audio; function calling. |

#### Embedding Models (May 2026)
| Model | Cost / 1M tokens | Dimension |
|-------|------------------|-----------|
| **Cohere Embed 4** ⭐ NEW | $0.10 | 256 / 512 / 1024 / 1536 (Matryoshka) |
| **text-embedding-3-large** | $0.13 | 3072 |
| **text-embedding-3-small** | $0.02 | 1536 |
| **Voyage-3** | $0.06 | 1024 |
| **Cohere embed-v3** | $0.10 | 1024 |

> [!IMPORTANT]
> **Inference-time Compute Costs:** For models with "Extended Thinking" or reasoning modes (GPT-5.4 Pro, Claude Opus 4.6), you are charged for **internal thinking tokens** even if not shown to the user. This can increase total request cost by 2x-10x for logic-heavy tasks. Always set a `budget_tokens` cap in production.

---

## Cost Calculation

### Basic Cost Formula

```python
def calculate_request_cost(
    input_tokens: int,
    output_tokens: int,
    model: str
) -> float:
    pricing = {
        "gpt-5.4": {"input": 2.50, "output": 15.00},
        "gpt-5.4-mini": {"input": 0.75, "output": 4.50},
        "claude-sonnet-4.6": {"input": 3.00, "output": 15.00},
        "claude-opus-4.6": {"input": 5.00, "output": 25.00},
        "gemini-3.1-flash": {"input": 0.10, "output": 3.00},
    }
    
    rates = pricing[model]
    cost = (
        (input_tokens / 1_000_000) * rates["input"] +
        (output_tokens / 1_000_000) * rates["output"]
    )
    return cost
```

### Example Cost Calculations

**Scenario 1: RAG Chatbot**
```
Per request:
- System prompt: 500 tokens
- Retrieved context: 2,000 tokens
- User message: 100 tokens
- Response: 300 tokens

Input: 2,600 tokens, Output: 300 tokens

GPT-5.4 cost: (2600 × $2.50 + 300 × $15) / 1M = $0.0110 per request

At 10,000 requests/day:
Daily: $95
Monthly: $2,850
```

**Scenario 2: Document Summarization**
```
Per document:
- Document: 8,000 tokens
- Summary: 500 tokens

GPT-5.4 cost: (8000 × $2.50 + 500 × $15) / 1M = $0.0275

1,000 documents: $27.50
10,000 documents: $275
```

### Monthly Cost Projection

```python
def project_monthly_cost(
    requests_per_day: int,
    avg_input_tokens: int,
    avg_output_tokens: int,
    model: str
) -> dict:
    per_request = calculate_request_cost(
        avg_input_tokens, avg_output_tokens, model
    )
    
    daily = per_request * requests_per_day
    monthly = daily * 30
    yearly = monthly * 12
    
    return {
        "per_request": per_request,
        "daily": daily,
        "monthly": monthly,
        "yearly": yearly
    }

# Example
costs = project_monthly_cost(
    requests_per_day=50000,
    avg_input_tokens=2000,
    avg_output_tokens=400,
    model="gpt-5.4"
)
# Output: ~$18,750/month
```

---

## Cost Optimization Strategies

### Strategy 1: Model Routing

Route requests to appropriate model tiers:

```python
class ModelRouter:
    def __init__(self):
        self.classifier = load_complexity_classifier()
    
    def route(self, query: str, context: str) -> str:
        complexity = self.classifier.predict(query)
        
        if complexity < 0.3:
            return "gpt-5.4-mini"  # Simple queries
        elif complexity < 0.7:
            return "gpt-5.4-mini"  # Medium, try cheap first
        else:
            return "gpt-5.4"  # Complex queries

    def route_with_fallback(self, query: str, context: str) -> str:
        # Try cheap model first
        response = self.try_model("gpt-5.4-mini", query, context)

        if self.is_quality_sufficient(response):
            return response

        # Fallback to expensive model
        return self.try_model("gpt-5.4", query, context)
```

**Potential savings:** 50-70% with minimal quality impact

### Strategy 2: Prompt Optimization

Reduce token count without losing quality:

```python
# Before: 2,500 tokens
system_prompt = """
You are a helpful customer support assistant for Acme Corp. 
You have access to our product documentation and should answer 
questions accurately and helpfully. Always be polite and professional.
If you don't know something, say so rather than making things up.
Format your responses clearly with bullet points when listing items.
[... more verbose instructions ...]
"""

# After: 800 tokens
system_prompt = """
You are Acme Corp's support assistant.
Rules:
- Answer from provided context only
- Admit uncertainty
- Use bullet points for lists
- Be concise
"""

# Savings: 1,700 tokens × $2.50/1M = $0.00425 per request
# At 10K requests/day: $42.50/day = $1,275/month
```

### Strategy 3: Caching

Cache responses for repeated or similar queries:

```python
class ResponseCache:
    def __init__(self, ttl_seconds: int = 3600):
        self.exact_cache = TTLCache(maxsize=10000, ttl=ttl_seconds)
        self.semantic_cache = SemanticCache(threshold=0.95)
    
    def get_or_generate(self, query: str, context: str) -> tuple[str, bool]:
        # Check exact cache
        cache_key = self.make_key(query, context)
        if cache_key in self.exact_cache:
            return self.exact_cache[cache_key], True  # Cache hit
        
        # Check semantic cache
        similar = self.semantic_cache.find_similar(query)
        if similar:
            return similar.response, True  # Semantic hit
        
        # Generate new response
        response = self.generate(query, context)
        self.exact_cache[cache_key] = response
        self.semantic_cache.add(query, response)
        
        return response, False  # Cache miss

# With 30% cache hit rate:
# Baseline: $3,000/month
# With caching: $2,100/month
# Savings: $900/month
```

### Strategy 4: Batch Processing

Process multiple requests together for efficiency:

```python
# Real-time: pay full price
for query in queries:
    response = model.generate(query)

# Batch API (OpenAI offers 50% discount):
batch_responses = model.batch_generate(queries)
# Cost: 50% of real-time pricing
```

### Strategy 5: Output Length Control

Limit response length appropriately:

```python
# Reduce unnecessary output
response = model.generate(
    prompt=prompt,
    max_tokens=300,  # Limit output
    stop=["\n\n"]    # Stop at natural break
)

# Cost impact:
# Before: avg 500 output tokens = $0.0075 per request (GPT-5.4)
# After: avg 250 output tokens = $0.00375 per request
# Savings: 50% on output costs
```

### Cost Optimization Summary

| Strategy | Effort | Potential Savings |
|----------|--------|-------------------|
| Model routing | Medium | 50-70% |
| **Context Caching** | Low | **60-90% (Input)** |
| Prompt optimization | Low | 20-40% |
| Response caching | Medium | 20-40% |
| Batch processing | Low | 50% (OpenAI/Anthropic) |

---

## Context Caching Economics

**The "Golden Rule" for RAG (still true in 2026).**
If you have a fixed system prompt or a shared knowledge base (prefix) larger than 10,000 tokens, **Context Caching** is mandatory.

**Break-even Analysis (Claude Sonnet 4.6):**
- **Standard Input**: $3.00 / 1M tokens
- **Cached Input**: $0.30 / 1M tokens (90% discount)
- **Cache Write Fee**: $3.75 / 1M tokens (5-min TTL at 1.25x); $6.00 (1-hour TTL at 2x)

`Break-even = (Write Fee) / (Standard Rate - Cached Rate) ≈ 1.4 requests (5-min) or 2.2 requests (1-hour)`

If your long prefix is used by **more than 2 users**, caching it is strictly cheaper than sending it raw every time. Both OpenAI and Anthropic now offer batch API discounts (50% off) that stack with caching.

---

## Self-Hosting & GPU Cloud Arbitrage

**The Reserved vs. Serverless Tradeoff:**

| Model Size | Serverless (RunPod/Together) | Reserved (Lambda/AWS) |
|------------|-----------------------------|-----------------------|
| **Burst Capacity** | Infinite (cold starts) | Fixed |
| **Utilization** | Pay only for compute time | 24/7 fixed cost |
| **TCO Break-even**| **Cost-effective < 40% util** | **Cost-effective > 40% util** |

**Principal-level Nuance:**
"GPU Cloud Arbitrage" involves moving production workloads between providers based on **spot instance availability**. Tools like **Skypilot** automate this, saving up to 60% on self-hosting costs by following "low-demand" regions globally. The rise of MoE models (Llama 4 Scout fits on a single H100, Maverick on ~2x H100, DeepSeek V4 Flash on 4x H100) has further reduced self-hosting GPU requirements compared to dense models.

### When Self-Hosting Makes Sense

```
Break-even analysis:

API cost at scale:
- 1M requests/month
- 2,500 tokens average
- GPT-5.4: ~$37,500/month
- Claude Sonnet 4.6: ~$30,000/month

Self-hosted equivalent (Llama 4 Maverick via MoE):
- 2x H100 80GB: ~$6/hour × 730 = $4,380/month
- Engineering time: $5,000/month (0.5 FTE)
- Ops overhead: $2,000/month
- Total: ~$11,380/month

Savings vs GPT-5.4: $26,120/month = 70%
Savings vs Claude Sonnet 4.6: $18,620/month = 62%
```

### Self-Hosting Cost Components

| Component | Monthly Cost | Notes |
|-----------|--------------|-------|
| GPU compute | $5K-20K | Depends on model size |
| Storage | $200-500 | Model weights, logs |
| Networking | $100-500 | Egress, load balancing |
| Engineering | $5K-15K | Partial FTE for ops |
| Monitoring | $100-500 | Observability tools |

### GPU Requirements by Model Size

| Model Size | GPU Config | Estimated Cost/Month |
|------------|------------|---------------------|
| 7B (INT4) | 1x A10G | $500-800 |
| 7B (FP16) | 1x A100 40GB | $1,500-2,500 |
| 70B (INT4) | 2x A100 80GB | $5,000-8,000 |
| 70B (FP16) | 4x A100 80GB | $10,000-15,000 |
| 405B (INT4) | 8x H100 | $20,000-30,000 |

### Decision Framework

```
Choose API when:
- Volume < 100K requests/month
- No ML ops expertise
- Need highest quality (frontier models)
- Fast iteration needed

Choose self-hosting when:
- Volume > 500K requests/month
- Have ML infrastructure team
- Data privacy requirements
- Predictable, stable workload
- Custom fine-tuning needed
```

---

## Total Cost of Ownership

### TCO Components

```python
def calculate_tco(scenario: dict) -> dict:
    # Direct costs
    api_or_compute = scenario["monthly_api_cost"]
    
    # Engineering costs
    development = scenario["dev_hours"] * scenario["engineer_rate"]
    maintenance = scenario["maintenance_hours"] * scenario["engineer_rate"]
    
    # Infrastructure
    vector_db = scenario["vector_db_cost"]
    monitoring = scenario["monitoring_cost"]
    
    # Indirect costs
    downtime_risk = scenario["expected_downtime_hours"] * scenario["revenue_per_hour"]
    
    monthly_tco = (
        api_or_compute +
        development / 12 +  # Amortized over year
        maintenance +
        vector_db +
        monitoring +
        downtime_risk
    )
    
    return {
        "monthly_tco": monthly_tco,
        "yearly_tco": monthly_tco * 12,
        "breakdown": {
            "llm": api_or_compute,
            "engineering": development / 12 + maintenance,
            "infrastructure": vector_db + monitoring,
            "risk": downtime_risk
        }
    }
```

### Example TCO Comparison

**Scenario: Customer Support Bot (50K requests/month)**

| Cost Component | API-Based | Self-Hosted |
|----------------|-----------|-------------|
| LLM costs | $5,000 | $3,000 |
| Vector DB | $70 | $200 |
| Engineering (monthly) | $500 | $3,000 |
| Monitoring | $100 | $200 |
| **Monthly Total** | **$5,670** | **$6,400** |

*At this scale, API is cheaper due to engineering overhead.*

**Scenario: Large-Scale RAG (2M requests/month)**

| Cost Component | API-Based | Self-Hosted |
|----------------|-----------|-------------|
| LLM costs | $50,000 | $15,000 |
| Vector DB | $500 | $1,000 |
| Engineering (monthly) | $1,000 | $8,000 |
| Monitoring | $200 | $500 |
| **Monthly Total** | **$51,700** | **$24,500** |

*At this scale, self-hosting is significantly cheaper.*

---

## Interview Questions

### Q: How would you optimize costs for a high-volume RAG application?

**Strong answer:**
I would approach cost optimization in layers:

**1. Architecture optimization:**
- Model routing: Use cheap model for simple queries
- Caching: 30-40% of queries may be cacheable
- Prompt compression: Minimize system prompt tokens

**2. Model selection:**
```
Simple queries (60%): GPT-5.4-mini at $0.003/request
Complex queries (40%): GPT-5.4 at $0.011/request
Weighted avg: $0.0062/request (vs $0.011 all GPT-5.4)
Savings: 44%
```

**3. Infrastructure:**
- Batch embedding updates (50% cheaper)
- Right-size vector DB
- Use spot instances where possible

**4. Monitoring:**
- Track cost per query type
- Alert on anomalies
- Regular cost reviews

### Q: When would you recommend self-hosting vs using APIs?

**Strong answer:**
Decision depends on multiple factors:

**Volume threshold:**
- Below 100K/month: Almost always API
- 100K-500K: Evaluate case by case
- Above 500K: Often self-hosting wins

**Team capabilities:**
- No ML ops: API regardless of scale
- Strong infra team: Consider self-hosting earlier

**Quality requirements:**
- Need absolute best: APIs (frontier models)
- Good enough works: Self-hosted open models

**Other factors:**
- Data privacy: May force self-hosting
- Latency control: Self-hosting gives more control
- Fine-tuning needs: Self-hosting enables more customization

**My recommendation process:**
1. Start with APIs for fastest iteration
2. Build abstraction layer for model switching
3. Evaluate self-hosting when spend exceeds $10K/month
4. Pilot with shadow deployment before committing

---

## References

- OpenAI Pricing: https://developers.openai.com/api/docs/pricing
- Anthropic Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Google AI Pricing: https://ai.google.dev/gemini-api/docs/pricing
- xAI Pricing: https://docs.x.ai/developers/models
- Mistral Pricing: https://docs.mistral.ai/getting-started/changelog
- Lambda Labs GPU Pricing: https://lambdalabs.com/service/gpu-cloud
- RunPod Pricing: https://www.runpod.io/pricing
- LLM Pricing Comparison: https://pricepertoken.com/

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Token** | The basic unit an LLM reads and writes; roughly 0.75 English words on average | All API pricing is expressed per million tokens; understanding tokens is essential to cost math |
| **Input Tokens** | Tokens in the request sent to the model (system prompt + user message + retrieved context) | Charged at the input rate; usually cheaper than output tokens |
| **Output Tokens** | Tokens in the model's generated response | Charged at the output rate, typically 2–5× the input rate; long responses are the main cost driver |
| **Thinking Tokens** | Internal reasoning tokens a model generates before its visible output in extended-thinking mode | Billed even though not shown to the user; can multiply request cost by 2–10× |
| **Prompt Caching / Context Caching** | A provider feature that stores a reused prompt prefix so cache-hit requests pay a fraction of the normal input price | Can cut input costs 60–90% when the same long system prompt or document is used repeatedly |
| **Cache Write Fee** | The one-time cost to store a prefix in the provider's cache (typically 1.25–2× normal input rate) | Paid once; breaks even after roughly 2 reuses, then saves money on every subsequent hit |
| **Cache Hit** | A request where the prefix is found in the cache, charged at the discounted cache-hit rate | The economics of context caching: hit rate determines actual savings |
| **Batch API** | An asynchronous endpoint that accepts many requests at once and returns results hours later at a 50% discount | Suitable for non-time-sensitive work such as document processing, overnight evaluations, or bulk embeddings |
| **Tiered Pricing** | Volume discounts applied when monthly spend crosses certain thresholds | Negotiated by large enterprises to reduce per-token cost at scale |
| **Commitment-Based Pricing** | Pre-purchasing a token volume for a year in exchange for a lower per-token rate | Saves roughly 20% versus pay-as-you-go when consumption is predictable |
| **Token-Based Pricing** | The standard API billing model: `cost = (input_tokens × input_rate) + (output_tokens × output_rate)` | The fundamental formula for all LLM cost projections |
| **Model Routing** | Classifying each incoming request and sending it to the cheapest model that can handle it adequately | The highest-leverage cost optimization; can cut spend 50–70% with minimal quality impact |
| **Semantic Cache** | A cache that uses embedding similarity to recognize paraphrased versions of previously answered queries | Extends response caching beyond exact duplicates to semantically equivalent questions |
| **TTL (Time-to-Live)** | How long a cached item is kept before expiring; in context caching, typically 5 minutes or 1 hour | Determines the window during which a prefix stays hot in the cache |
| **Batch Processing** | Grouping many requests and sending them together instead of individually | Enables batch API discounts and reduces per-request overhead |
| **Self-Hosting** | Running an open-weight model on your own GPU infrastructure instead of calling a provider API | Trades per-token savings for upfront GPU cost and engineering overhead; profitable above ~500K requests/month |
| **GPU Cloud Arbitrage** | Automatically moving workloads to whichever GPU cloud region has the lowest spot-instance price at a given moment | Tools like Skypilot automate this to cut self-hosting costs by up to 60% |
| **Spot Instance** | A cloud GPU rented at a steep discount in exchange for the risk that the cloud provider can reclaim it with short notice | Cheap but interruptible; best for fault-tolerant batch jobs, not latency-sensitive inference |
| **Serverless Compute** | Cloud infrastructure that scales from zero and charges only for actual execution time, with no idle cost | Used by AWS Glue and similar services; ideal for bursty or periodic workloads |
| **MoE (Mixture-of-Experts)** | An architecture where only a subset of the model's parameters are active per token, reducing compute per inference | Allows trillion-parameter models to run on much less hardware than a dense model of similar quality |
| **DPU (Data Processing Unit)** | The billing unit for AWS Glue compute capacity; each DPU represents a fixed amount of CPU and memory | Used when estimating AWS Glue job costs |
| **H100** | NVIDIA's latest flagship data-center GPU; the standard hardware reference for frontier model inference | Determines self-hosting cost benchmarks; a single H100 can run models like Llama 4 Scout or Gemma 4 |
| **A100** | The previous-generation NVIDIA data-center GPU, still widely used for medium-size model inference | Cost reference for 7B–70B self-hosted models |
| **TCO (Total Cost of Ownership)** | All costs associated with running an AI system: compute, storage, networking, engineering time, monitoring, and downtime risk | The correct comparison unit when evaluating API versus self-hosting decisions |
| **Break-even Analysis** | Calculating at what usage volume (or number of cache hits) a more expensive option becomes cheaper | Required before switching from API to self-hosting or enabling a cache write |
| **Vector DB** | A database that stores embedding vectors and supports fast approximate-nearest-neighbor search | Infrastructure cost component in RAG systems; typically $70–500/month depending on scale |
| **INT4 / FP16** | Numerical precisions for storing model weights; INT4 uses 4 bits per weight (quantized), FP16 uses 16 bits | Lower precision reduces VRAM requirements and cost; INT4 allows a 70B model to fit on 2× A100 instead of 4× |
| **Prompt Optimization** | Rewriting system prompts to convey the same instructions in fewer tokens | Low-effort cost reduction; cutting a 2,500-token prompt to 800 tokens saves money on every single call |
| **Output Length Control** | Using `max_tokens` and stop sequences to cap how long the model's response can be | Eliminates unnecessary verbosity; cutting average output from 500 to 250 tokens halves output cost |
| **Idempotency Key** | A unique ID attached to a request so that a retry or duplicate call is recognized and not processed twice | Critical for cost control in async job pipelines where the same job might be submitted more than once |
| **Fast Mode** | A model operating mode that sacrifices some reasoning depth for roughly 2.5× faster throughput at a separate price tier | Useful for latency-sensitive agentic loops that don't need maximum reasoning quality |

---

*Previous: [Capability Assessment](02-capability-assessment.md) | Next: [Model Selection Guide](04-model-selection-guide.md)*
