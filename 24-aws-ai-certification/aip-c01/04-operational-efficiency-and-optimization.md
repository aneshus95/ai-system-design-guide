# Domain 4: Operational Efficiency and Optimization

**Domain weight: 12% of scored questions (AIP-C01)**

This domain tests your ability to design and operate cost-effective, low-latency, observable generative AI workloads on AWS. You must know *when* to reach for each cost-control and performance lever, and *how* to measure the results with CloudWatch.

> **Plain English:** You have a great GenAI app — now you need it to be cheap enough to run at scale, fast enough for users to stay happy, and instrumented well enough that you spot problems before your customers do. This domain covers every dial you can turn in Amazon Bedrock to achieve that.

---

## Table of Contents

1. [Cost Optimization](#1-cost-optimization)
   - 1.1 [Token Pricing Fundamentals](#11-token-pricing-fundamentals)
   - 1.2 [Right-Sizing: Choosing the Right Model](#12-right-sizing-choosing-the-right-model)
   - 1.3 [Intelligent Prompt Routing](#13-intelligent-prompt-routing)
   - 1.4 [Prompt Caching](#14-prompt-caching)
   - 1.5 [Batch Inference](#15-batch-inference)
   - 1.6 [Provisioned Throughput vs On-Demand](#16-provisioned-throughput-vs-on-demand)
   - 1.7 [Prompt and Context Trimming](#17-prompt-and-context-trimming)
2. [Latency and Performance Optimization](#2-latency-and-performance-optimization)
   - 2.1 [Streaming Responses](#21-streaming-responses)
   - 2.2 [Semantic Response Caching](#22-semantic-response-caching)
   - 2.3 [Region Proximity and Cross-Region Inference](#23-region-proximity-and-cross-region-inference)
   - 2.4 [Parallelization](#24-parallelization)
   - 2.5 [Keeping Context Windows Lean](#25-keeping-context-windows-lean)
3. [Monitoring and Observability](#3-monitoring-and-observability)
   - 3.1 [CloudWatch Metrics for Bedrock](#31-cloudwatch-metrics-for-bedrock)
   - 3.2 [Model Invocation Logging](#32-model-invocation-logging)
   - 3.3 [Cost Tracking and Budgets](#33-cost-tracking-and-budgets)
   - 3.4 [Dashboards and Alerting](#34-dashboards-and-alerting)
4. [Glossary](#glossary)
5. [References](#references)

---

## 1. Cost Optimization

### 1.1 Token Pricing Fundamentals

Amazon Bedrock charges per **input token** and **output token** processed. Understanding the breakdown is the first step to controlling costs:

| Cost category | What drives it | Reduction strategy |
|---|---|---|
| Input tokens | System prompt + conversation history + retrieved chunks + user message | Prompt trimming, prompt caching, retrieval precision |
| Output tokens | Model-generated response | Set `max_tokens` ceiling, use concise instruction styles |
| Cache write tokens | Tokens written to prompt cache (billed at a slight premium) | Only cache large, stable prefixes; amortised cost drops fast |
| Cache read tokens | Tokens served from prompt cache (billed at a heavy discount) | Maximise cache hit rate through stable prefixes |

Prices vary by model family and region. Always verify current per-token rates on the [Amazon Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/).

#### 🎯 On the exam
- A question saying "the same large system prompt is sent with every request — how do you cut costs?" → **prompt caching**.
- "Output token costs are high" → set a tighter `max_tokens` and write more precise prompts.
- Input token pricing is usually lower than output token pricing for the same model.

---

### 1.2 Right-Sizing: Choosing the Right Model

Not every task needs the most capable (and most expensive) model. AWS publishes a spectrum from lightweight to frontier:

- **Amazon Nova Lite / Nova Micro** — ultra-low cost, high speed; suited for classification, summarisation, and slot-filling.
- **Amazon Nova Pro / Claude Haiku** — mid-tier; good for structured extraction and moderate reasoning.
- **Claude Sonnet / Claude Opus** — frontier reasoning, coding, long-context tasks.

**Model distillation** lets you train a smaller "student" model to mimic a larger "teacher" model using [Amazon Bedrock Model Distillation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-distillation.html). The distilled model retains quality on the target task while being cheaper and faster.

Decision rule:

1. Start with the smallest model that meets quality thresholds.
2. Benchmark on your eval dataset (→ Domain 5).
3. Promote to a larger model only if quality gates are not met.

#### 🎯 On the exam
- "Simple FAQ bot that classifies intent" → Amazon Nova Micro or Claude Haiku, **not** Claude Opus.
- "Need the highest reasoning for complex code generation" → Claude Opus or Claude Sonnet.
- "Train a cheaper model from a large one" → **Model Distillation** in Bedrock.

---

### 1.3 Intelligent Prompt Routing

> **Why (the rationale):** In mixed-complexity workloads (simple FAQ + complex reasoning in the same chatbot), always routing to the frontier model wastes 60–80% of spend on queries a smaller model handles equally well. Intelligent Prompt Routing applies per-request model selection with no application code change.
> **When to use:** Workloads within a single supported model family where request complexity varies (chatbots, support agents, search). Not applicable when all requests require frontier-model quality, or when routing across different providers.
> **Nuances & gotchas:** Works only **within a supported model family** (Claude tiers, Nova Lite/Pro, Llama variants) — not across providers. No additional charge for the routing decision. The router's quality prediction is heuristic-based; edge cases may route complex queries to the cheaper model — evaluate output quality with your workload before relying on it for critical tasks.

[Amazon Bedrock Intelligent Prompt Routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html) automatically routes each incoming request to the *cheapest model in a family that is predicted to handle it correctly*. It uses advanced prompt matching and model-understanding techniques at inference time — no change to your application code is required beyond specifying a **prompt router ARN** instead of a model ID.

Key facts:
- **Supported families:** Anthropic Claude (Haiku / Haiku 3.5 / Sonnet 3.5 v1 / Sonnet 3.5 v2), Meta Llama (3.1 8B–70B, 3.2 11B–90B, 3.3 70B), Amazon Nova (Lite and Pro).
- **Cost savings:** up to **30%** cost reduction without compromising accuracy, per AWS benchmarks ([source](https://aws.amazon.com/bedrock/intelligent-prompt-routing/)).
- **Pricing:** no additional charge for the routing decision; you pay only the selected model's token rate.
- Simple queries go to the cheaper model (e.g., Haiku); complex queries go to the more capable model (e.g., Sonnet).

#### 🎯 On the exam
- "Route simple questions cheaply while preserving quality on hard questions, with no code change" → **Intelligent Prompt Routing**.
- It works *within* a model family, not across different providers simultaneously.

---

### 1.4 Prompt Caching

> **Why (the rationale):** When the same large system prompt, tool definitions, or reference document is re-sent on every API call, the model recomputes the same KV cache from scratch every time — wasting compute and inflating costs. Prompt caching stores the KV-cache state so repeated prefixes are billed at a deep discount.
> **When to use:** Large, stable system prompts (>1,024 tokens for most Claude models); agentic loops with large tool definition blocks; multi-turn conversations where a large document is pasted at the start of every turn; user uploads a document and asks many questions.
> **Nuances & gotchas:** Default TTL is **5 minutes** — a cache built at minute 0 expires before a request at minute 6, triggering a cache miss and a write-cost charge. Models with 1-hour TTL (Claude Opus 4.5, Sonnet 4.5, Haiku 4.5) avoid this for long-running agent sessions. **Prompt caching is NOT available with batch inference.** The minimum prefix size is 1,024 tokens for most Claude models and 4,096 tokens for newer Haiku/Sonnet/Opus 4.x models — tiny system prompts will not be cached. Cache write tokens cost slightly MORE than standard input tokens but amortize across many cache-read hits. Cached tokens do NOT count toward the `inputTokens` field in the API response.

[Amazon Bedrock Prompt Caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) lets you mark a static portion of your prompt (a "prefix") for caching. On subsequent requests, Bedrock skips reprocessing the cached portion and serves those tokens at a significantly reduced rate.

#### How it works

1. You place one or more **cache checkpoints** (`cachePoint` in the Converse API, `cache_control` in InvokeModel for Claude) after static content.
2. Bedrock stores the KV-cache state for that prefix with a Time-To-Live (TTL).
3. Subsequent requests with the same prefix receive a **cache hit** — tokens are billed as `cacheReadInputTokens` at a heavy discount. New tokens that miss the cache are billed at standard input token rates.
4. Cache misses write to cache and are billed as `cacheWriteInputTokens` (at a slight premium over standard input rates — this cost is amortised across many cache hits).

#### TTL options

| TTL | Supported models | When to use |
|---|---|---|
| **5 minutes** (default) | Most Claude models, Amazon Nova | Frequent, sub-5-minute requests (chatbots, interactive RAG) |
| **1 hour** | Claude Opus 4.5, Claude Sonnet 4.5, Claude Haiku 4.5 | Agentic workflows where responses arrive > 5 min apart; long user sessions |
| **30 minutes** | GPT-5.6 Sol, Terra, Luna on Bedrock | OpenAI-compatible agentic loops |

#### Minimum token requirements

The cached prefix must meet a **minimum token count** per checkpoint before Bedrock will cache it:
- Claude 3.7 Sonnet, Claude Opus 4, Claude 3.5 Sonnet v2, Claude Sonnet 4.6: **1,024 tokens** minimum.
- Claude Opus 4.5, Claude Sonnet 4.5, Claude Haiku 4.5, Claude Opus 4.6: **4,096 tokens** minimum.

Maximum **4 cache checkpoints** per request for Claude models.

#### What to cache

Cache checkpoints can be placed in `system`, `messages`, and `tools` fields. Processing order is `tools → system → messages`; changing earlier fields invalidates later caches. Best practice: place large, static content (system prompts, tool definitions, reference documents) *before* dynamic user messages, and put the checkpoint immediately after the static block.

#### Cache metrics

The Converse API response includes:
- `cacheReadInputTokens` — tokens served from cache (cheap).
- `cacheWriteInputTokens` — tokens written to cache (slight premium).

> **Important:** `inputTokens` in the response represents only *non-cached* tokens. Total input = `inputTokens + cacheReadInputTokens + cacheWriteInputTokens`.

**Prompt caching is not supported with batch inference.**

#### 🎯 On the exam
- "Large system prompt or tool list sent with every request" → **prompt caching**.
- "User uploads a 50-page PDF and asks many questions" → cache the PDF content as a prompt prefix.
- "Agentic loop that calls tools repeatedly with the same tool definitions" → cache the `tools` block.
- Cache checkpoints require a *minimum token threshold* — a tiny system prompt will not be cached.

---

### 1.5 Batch Inference

> **Why (the rationale):** For offline jobs (nightly summarization, bulk classification, dataset labeling), paying real-time on-demand prices is unnecessary — the workload has no latency SLA. Batch inference runs the same model asynchronously on AWS-optimized batch capacity at ~50% of on-demand cost.
> **When to use:** Large-scale, non-latency-sensitive workloads: document summarization, embedding backfill, dataset annotation, nightly report generation. Minimum 100 records per job; results available in minutes to hours depending on job size.
> **Nuances & gotchas:** Batch inference is **NOT** compatible with prompt caching or provisioned throughput. It runs on a separate batch quota — your on-demand quota does not constrain batch jobs and vice versa. Jobs must have at least 100 records in the input JSONL; submitting fewer will fail. Poll `GetModelInvocationJob` or use EventBridge status-change events to detect completion — there is no push notification by default.

[Amazon Bedrock Batch Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html) allows you to submit large volumes of inference requests as a single asynchronous job. Results are written to Amazon S3 when the job completes.

Key facts:
- **Price:** approximately **50% cheaper** than equivalent on-demand inference for the same model ([source](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)).
- **Input format:** JSONL file in Amazon S3.
- **Turnaround:** hours, not milliseconds — suitable for offline processing, data labelling, bulk document summarisation, nightly report generation.
- **Not supported** with provisioned throughput or prompt caching.
- Supported on selected foundation models; check the [Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/) for the per-model batch discount.

When to use batch vs. on-demand:

| Scenario | Mode |
|---|---|
| Interactive chatbot, user waiting for response | On-demand |
| Nightly summarisation of 10,000 documents | Batch inference |
| High-volume real-time API with predictable load | Provisioned Throughput |
| Mix of simple and complex queries, real-time | Intelligent Prompt Routing + On-demand |

#### 🎯 On the exam
- "Large non-real-time job, save as much cost as possible" → **batch inference (~50% discount)**.
- "Need results in real time" → batch inference is **wrong** here.
- Batch inference is asynchronous; job status can be polled via the Bedrock API.

---

### 1.6 Provisioned Throughput vs On-Demand

#### On-Demand

Pay per token with no commitment. Requests share capacity across all customers. Suitable for:
- Development and testing.
- Variable or unpredictable traffic.
- Applications where occasional throttling is acceptable.

#### Provisioned Throughput

> **Why (the rationale):** On-demand capacity is shared across all Bedrock customers — at peak load you may get throttled even if your overall usage is modest. Provisioned Throughput reserves dedicated capacity, eliminating shared-pool contention and giving consistent latency and throughput for SLA-bound production workloads. It is also the **only way to invoke fine-tuned or custom-imported models**.
> **When to use:** Sustained, predictable high-volume production workloads (≥60–70% utilization where per-token cost exceeds hourly PT cost); any custom/fine-tuned model (mandatory); when ThrottlingExceptions persist despite quota increases.
> **Nuances & gotchas:** You **must pass `provisionedModelArn` as the `modelId`** in your API calls — using the base foundation model ARN leaves your reserved capacity idle while you keep paying the PT hourly rate AND on-demand per-token rates. PT is billed per model unit per hour for the committed term (1-month or 6-month) regardless of actual traffic. PT does NOT reduce inference latency the same way as latency-optimized inference mode — it guarantees throughput, not speed.

[Provisioned Throughput](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html) reserves dedicated **tokens-per-minute (TPM)** capacity for your account. You pay a fixed hourly rate regardless of usage.

| Attribute | On-Demand | Provisioned Throughput |
|---|---|---|
| Billing | Per token used | Fixed hourly (1-month or 3-month term) |
| Latency under load | May spike when throttled | Consistent; no shared-pool contention |
| Throughput guarantee | No | Yes — reserved TPM |
| Overhead | None | Upfront commitment |
| Best for | Variable/bursty workloads | Sustained, predictable high-volume workloads |

**When Provisioned Throughput pays off:**  
The break-even point depends on your utilisation rate. As a rule of thumb: if you consistently use ≥ 60–70% of on-demand capacity for a model, Provisioned Throughput costs less per token over a 1-month commitment. Use the [Bedrock pricing calculator](https://aws.amazon.com/bedrock/pricing/) to model your specific TPM needs.

#### On-Demand Tiers (Service Tiers)

AWS also offers [On-Demand Tiers](https://docs.aws.amazon.com/bedrock/latest/userguide/service-tiers-inference.html) — a middle ground that allows you to reserve prioritised capacity without a long-term commitment, billed at a fixed per-TPM rate monthly.

#### 🎯 On the exam
- "Predictable, high-throughput production workload that runs 24/7" → **Provisioned Throughput**.
- "Development environment, traffic is unpredictable" → **on-demand**.
- "Getting ThrottlingException at peak load but normal load is fine" → consider Provisioned Throughput or request a quota increase.

---

### 1.7 Prompt and Context Trimming

Every token in your input costs money and adds latency. Techniques to trim context:

- **Remove redundant instructions:** Consolidate repeated guidance into a single concise system prompt.
- **Sliding window conversation history:** Keep only the last *N* turns rather than the full history; summarise older turns into a compressed "memory block".
- **Retrieval precision:** In RAG applications, reduce the number of retrieved chunks (`top_k`) and improve chunking strategy so only the most relevant content is included (→ see Domain 5 for retrieval quality improvements).
- **Structured output formats:** Instructing the model to return JSON or a fixed schema often reduces output token counts compared to verbose prose.
- **Separate pre/post-processing:** Filter, clean, and truncate user input *before* sending it to the model.

---

## 2. Latency and Performance Optimization

### 2.1 Streaming Responses

Use the `ConverseStream` or `InvokeModelWithResponseStream` APIs to receive tokens as they are generated rather than waiting for the full response. This dramatically improves **perceived latency** (time to first token visible to the user) even when total generation time is unchanged.

CloudWatch tracks `TimeToFirstToken` (TTFT) for streaming APIs — monitor this metric to detect model-side slowdowns.

#### 🎯 On the exam
- "Users complain the UI hangs before anything appears" → enable **streaming** (`ConverseStream`).
- TTFT is only published for streaming APIs (`ConverseStream`, `InvokeModelWithResponseStream`).

---

### 2.2 Semantic Response Caching

> **Why (the rationale):** Prompt caching saves compute on repeated inputs; semantic caching goes further and eliminates the model call entirely for semantically similar repeated questions — achieving zero model cost and sub-millisecond latency for cache hits on common queries.
> **When to use:** High-traffic applications with a known distribution of popular queries (e.g., FAQ bots, product search, support chatbots). High cache hit rate is needed to justify the operational complexity of maintaining a query embedding index.
> **Nuances & gotchas:** Semantic caching introduces a staleness risk — a cached answer from three months ago may no longer be correct if the underlying knowledge base changed. You must implement TTL-based cache invalidation aligned with your knowledge base refresh cycle. Similarity threshold tuning is critical: too low → wrong answers served from cache; too high → almost every query misses.

Beyond prompt caching (which avoids recomputing input tokens), you can cache *complete model responses* at the application layer:

- **Exact match caching:** Hash the full prompt string; serve cached response on a hit. Simple but brittle — any prompt variation is a miss.
- **Semantic caching:** Embed the user query; perform a nearest-neighbour search against a vector store of past (query, response) pairs; serve the cached response when semantic similarity exceeds a threshold. Libraries such as GPTCache or LangChain's caching layer implement this pattern.

Semantic caching is complementary to prompt caching: prompt caching saves compute on repeated *inputs*; semantic caching saves an entire model call on repeated *questions with similar intent*.

---

### 2.3 Region Proximity and Cross-Region Inference

> **Why (the rationale):** Network round-trip adds measurable latency when the calling application and the Bedrock endpoint are in different regions. Cross-region inference additionally provides automatic capacity failover — when one region is congested, Bedrock routes to another region in the same geography.
> **When to use:** Deploy application and Bedrock in the same region (baseline). Enable cross-region inference when you're experiencing 503 capacity errors and cannot get a quota increase fast enough, or when building high-availability architectures that tolerate occasional cross-region routing.
> **Nuances & gotchas:** Cross-region inference routes traffic to other AWS regions — **data may leave your primary region**. This directly conflicts with strict data residency requirements (GDPR, sovereignty mandates). Disable cross-region inference for regulated workloads with hard residency constraints. Cross-region inference uses a different model ID prefix (cross-region inference ARN) — not the standard regional model ID.

- Deploy your Bedrock-calling application in the **same AWS Region** as your Bedrock endpoint to minimise network round-trip time.
- [Cross-Region Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html) automatically routes to the optimal AWS Region within your geography when capacity is constrained. This reduces throttling and improves availability, at the cost of occasionally higher network latency.

#### 🎯 On the exam
- "Getting 503 ServiceUnavailable errors due to capacity" → enable **cross-region inference** to leverage compute across regions.

---

### 2.4 Parallelization

For workflows that require multiple independent model calls (e.g., an agent that calls three tools simultaneously):

- Fire requests **in parallel** rather than sequentially.
- Use AWS Lambda concurrency, Amazon ECS tasks, or Python `asyncio`/`ThreadPoolExecutor` to fan out calls.
- Aggregate results after all parallel calls complete.

This pattern reduces wall-clock latency proportional to the number of parallel branches.

---

### 2.5 Keeping Context Windows Lean

Large context windows increase inference latency and cost. Strategies:

| Technique | Benefit |
|---|---|
| Summarise long conversation history | Fewer tokens, faster inference |
| Use a retrieval system (RAG) instead of stuffing all docs in context | Targeted context, not the entire corpus |
| Truncate retrieved chunks to relevant sentences | Reduces redundant tokens |
| Split long tasks into smaller subtasks | Avoids hitting context-length limits |

---

## 3. Monitoring and Observability

### 3.1 CloudWatch Metrics for Bedrock

> **Why (the rationale):** Without CloudWatch metrics, you discover latency spikes and quota exhaustion from user complaints rather than proactively from alarms. Bedrock's metrics provide real-time visibility into cost (token counts), performance (latency, TTFT), and reliability (throttles, error rates) across every model in use.
> **When to use:** Always instrument with CloudWatch metrics in production. Set alarms on `InvocationThrottles` (quota signal), `InvocationLatency` P99 (performance signal), and `InvocationServerErrors` (reliability signal) from day one.
> **Nuances & gotchas:** `TimeToFirstToken` (TTFT) is **only published for streaming APIs** (`ConverseStream`, `InvokeModelWithResponseStream`) — it does not appear for synchronous `Converse`/`InvokeModel`. `InputTokenCount` in CloudWatch metrics represents only non-cached tokens; cached tokens appear in `CacheReadInputTokens` and `CacheWriteInputTokens` — sum all three for true cost attribution. Use the `ModelId` dimension to filter by specific model; without it, metrics aggregate across all models.

The Amazon Bedrock `bedrock-runtime` endpoint publishes metrics to Amazon CloudWatch under the **`AWS/Bedrock`** namespace ([source](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)). Use the `ModelId` dimension to filter by a specific model.

#### Core runtime metrics

| Metric | Unit | What it measures |
|---|---|---|
| `Invocations` | SampleCount | Successful requests to Converse, ConverseStream, InvokeModel, InvokeModelWithResponseStream |
| `InvocationLatency` | Milliseconds | End-to-end time from request sent to last token received |
| `TimeToFirstToken` | Milliseconds | Time to first streamed token (streaming APIs only) |
| `InputTokenCount` | SampleCount | Tokens in the input (excluding cached tokens) |
| `OutputTokenCount` | SampleCount | Tokens in the output |
| `InvocationClientErrors` | SampleCount | Client-side errors (4xx) |
| `InvocationServerErrors` | SampleCount | Server-side errors (5xx) |
| `InvocationThrottles` | SampleCount | Requests throttled due to quota limits |
| `EstimatedTPMQuotaUsage` | SampleCount | Approximate tokens-per-minute quota consumption |
| `CacheReadInputTokens` | SampleCount | Input tokens served from prompt cache (low cost) |
| `CacheWriteInputTokens` | SampleCount | Input tokens written to prompt cache |
| `OutputImageCount` | SampleCount | Images generated (image generation models only) |

#### Model invocation logging delivery metrics (namespace: `AWS/Bedrock`)

| Metric | Description |
|---|---|
| `ModelInvocationLogsCloudWatchDeliverySuccess` | Logs successfully delivered to CloudWatch Logs |
| `ModelInvocationLogsCloudWatchDeliveryFailure` | Logs that failed to deliver to CloudWatch Logs |
| `ModelInvocationLogsS3DeliverySuccess` | Logs successfully delivered to S3 |
| `ModelInvocationLogsS3DeliveryFailure` | Logs that failed to deliver to S3 |

#### Key derived signals

- **Error rate:** `InvocationClientErrors / (Invocations + InvocationClientErrors + InvocationThrottles)`
- **Throttle rate:** `InvocationThrottles / (Invocations + InvocationThrottles)`
- **Cache hit rate:** `CacheReadInputTokens / (CacheReadInputTokens + CacheWriteInputTokens + InputTokenCount)`
- **OTPS (output tokens per second):** `OutputTokenCount / InvocationLatency` — helps distinguish slow model throughput from slow network.

Set CloudWatch **Alarms** on `InvocationThrottles` and `InvocationLatency` to catch quota and performance issues proactively.

#### 🎯 On the exam
- "How do you measure Bedrock latency?" → `InvocationLatency` and `TimeToFirstToken` (streaming) in CloudWatch.
- "How do you detect throttling automatically?" → CloudWatch Alarm on `InvocationThrottles`.
- "How do you track token usage for cost attribution?" → `InputTokenCount` + `OutputTokenCount` with `ModelId` dimension + cost allocation tags.

---

### 3.2 Model Invocation Logging

[Model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) captures the full request and response payload (including metadata) for every Bedrock API call in your account and region.

- **Disabled by default.** Must be explicitly enabled in the Bedrock console or via the `PutModelInvocationLoggingConfiguration` API.
- **Destinations:** Amazon CloudWatch Logs, Amazon S3, or both.
- **Log contents:** model ID, region, request timestamp, full input, full output, token counts, latency, user-agent.
- **Use cases:** debugging unexpected model outputs, compliance audit trails, offline analysis, input/output dataset collection for fine-tuning.
- Logs persist until the logging configuration is deleted (or until S3/CloudWatch retention policies expire).

> **Security note:** Invocation logs contain user data. Encrypt S3 buckets with AWS KMS and restrict access with IAM policies.

#### 🎯 On the exam
- "Audit every model call including full prompt and response" → enable **model invocation logging** to S3.
- "Debug why a specific user query returned a bad response" → check invocation logs in CloudWatch Logs.

---

### 3.3 Cost Tracking and Budgets

#### Cost allocation tags

Apply **cost allocation tags** to your Bedrock resources and [Application Inference Profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-create.html):

- Tags propagate to AWS Cost and Usage Report (CUR) line items.
- Example tags: `Project=CustomerServiceBot`, `Team=MLPlatform`, `Environment=Production`.
- Enables per-application, per-team cost attribution in AWS Cost Explorer.

#### Application Inference Profiles

> **Why (the rationale):** Without inference profiles, all Bedrock costs aggregate into a single line item in Cost Explorer — you can't distinguish Team A's chatbot spend from Team B's batch job. Inference profiles attach cost allocation tags at the routing level, enabling per-application, per-team cost attribution without code changes in each caller.
> **When to use:** Any multi-team or multi-application Bedrock deployment where chargeback, showback, or per-workload cost tracking is required.
> **Nuances & gotchas:** Tags on inference profiles propagate to CUR line items only after they are activated as cost allocation tags in the AWS Billing console — tag creation alone does not make them appear in Cost Explorer. Cross-region inference profiles (system-defined) use a different ARN format from application inference profiles (customer-defined).

[Application Inference Profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-create.html) are named configurations that wrap a model (or cross-region profile) with specific settings and tags. When you route all traffic for an application through its inference profile, all costs are automatically tagged consistently.

#### AWS Budgets and Cost Explorer

- Create an **AWS Budget** with a threshold alert (e.g., email when projected Bedrock spend exceeds $500/month).
- Use **AWS Cost Explorer** with the filter `Service = Amazon Bedrock` to analyse spend by model, region, and tag.
- Enable **CUR (Cost and Usage Report)** for the highest-granularity token-level breakdown via [understanding CUR data](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-understanding-cur-data.html).

#### 🎯 On the exam
- "Attribute Bedrock costs to different internal teams" → **cost allocation tags** on application inference profiles.
- "Get alerted when monthly Bedrock spend exceeds budget" → **AWS Budgets**.

---

### 3.4 Dashboards and Alerting

- CloudWatch **Generative AI Observability** provides pre-built dashboards for Bedrock invocation metrics automatically when you start using Bedrock. After enabling model invocation logging, you can also access invocation history tables in these dashboards ([source](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/GenAI-observability.html)).
- Build **custom CloudWatch dashboards** combining `Invocations`, `InvocationLatency`, `InvocationThrottles`, `InputTokenCount`, and `OutputTokenCount` widgets for a single-pane operational view.
- Use **CloudWatch Contributor Insights** to identify which model IDs or callers are driving the highest token consumption.
- Emit custom application-level metrics (e.g., RAG retrieval latency, guardrail trigger rate) to CloudWatch using the `PutMetricData` API for end-to-end observability.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| Prompt caching | Storing a processed prompt prefix in memory so future requests skip recomputing it | Reduces latency and input token cost for repeated large contexts |
| Cache checkpoint | A marker in your prompt that tells Bedrock where to save the cache | Defines the boundary of the cached prefix |
| Cache TTL | Time-To-Live — how long a cache checkpoint stays valid before expiring | Controls cache freshness vs. cost |
| Batch inference | Submitting thousands of prompts as an async job; results written to S3 | ~50% cheaper than on-demand for non-real-time workloads |
| Provisioned Throughput | Reserved tokens-per-minute capacity billed at a fixed hourly rate | Predictable cost and latency for sustained high-volume production traffic |
| On-Demand inference | Pay-per-token, no commitment; capacity shared with other customers | Best for variable or unpredictable workloads |
| Intelligent Prompt Routing | Bedrock feature that automatically picks the cheapest model predicted to handle a given request correctly | Up to 30% cost reduction within a model family |
| Model Distillation | Training a smaller "student" model to mimic a larger "teacher" model | Reduces inference cost and latency while preserving task quality |
| InvocationLatency | CloudWatch metric: end-to-end time from request to final token | Primary latency signal |
| TimeToFirstToken (TTFT) | CloudWatch metric: time until first streamed token arrives | Perceived latency for streaming UIs |
| InvocationThrottles | CloudWatch metric: count of throttled requests | Signals quota exhaustion |
| Application Inference Profile | A named Bedrock configuration wrapping a model + settings + cost tags | Enables per-application cost attribution |
| Model Invocation Logging | Full request/response audit trail sent to CloudWatch Logs or S3 | Debugging, compliance, dataset collection |
| Cross-Region Inference | Bedrock automatically routes to another region when capacity is constrained | Reduces throttling and improves availability |
| Semantic caching | Application-layer cache that serves previous responses to semantically similar new questions | Eliminates model calls entirely for near-duplicate queries |

---

## References

- [Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
- [Prompt Caching — Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)
- [Intelligent Prompt Routing — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html)
- [Batch Inference — Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Provisioned Throughput — Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html)
- [Service Tiers for Inference — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/service-tiers-inference.html)
- [Monitor bedrock-runtime Inference Using CloudWatch Metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)
- [Model Invocation Logging — Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [Generative AI Observability — Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/GenAI-observability.html)
- [Cross-Region Inference — Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Amazon Bedrock Cost Optimization](https://aws.amazon.com/bedrock/cost-optimization/)
- [Understanding CUR Data for Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-understanding-cur-data.html)
- [Reduce Costs and Latency with Intelligent Prompt Routing and Prompt Caching (AWS Blog)](https://aws.amazon.com/blogs/aws/reduce-costs-and-latency-with-amazon-bedrock-intelligent-prompt-routing-and-prompt-caching-preview/)
- [Effective Cost Optimization Strategies for Amazon Bedrock (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/effective-cost-optimization-strategies-for-amazon-bedrock/)
- [Bedrock service — ../services/bedrock.md](../services/bedrock.md)

---

*Part of the [AWS AI Certification course](../README.md) · AIP-C01 track.*
