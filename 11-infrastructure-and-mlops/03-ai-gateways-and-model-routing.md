# AI Gateways and Model Routing

The case for a routing layer is now empirical, not aspirational. Per Datadog's 2026 State of AI Engineering, **69% of companies run three or more models in production**, and **rate-limit / capacity errors are the single largest production failure mode** (about 5% of requests fail, and roughly 60% of those failures are capacity-driven). Datadog's own framing: operational complexity, not model intelligence, is now the primary barrier to reliable AI at scale.

Once you depend on multiple models and your traffic is non-trivial, a single provider becomes a single point of failure, and rate limits, not model quality, are what page you at 2am. An **AI gateway** is the standard mitigation. This chapter covers what it does, how routing and fallback work, the 2026 tool landscape, and when you actually need one.

## Table of Contents

- [What an AI Gateway Is](#what-an-ai-gateway-is)
- [Routing Strategies](#routing-strategies)
- [Fallback and Reliability](#fallback-and-reliability)
- [The 2026 Tool Landscape](#the-2026-tool-landscape)
- [Architecture Patterns](#architecture-patterns)
- [Do You Need a Gateway Yet?](#do-you-need-a-gateway-yet)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What an AI Gateway Is

An AI gateway (also LLM gateway, LLM proxy, or LLM router) is a **control plane between your applications and model providers**, exposing one consistent (almost always OpenAI-compatible) API and centralizing the cross-cutting concerns that would otherwise be smeared across every service.

| Job | What it does |
|-----|--------------|
| Unified API | One OpenAI-shaped interface across providers; no per-provider glue |
| Model routing | Picks which model/deployment serves each request |
| Fallback chains | On a retryable failure, tries the next provider/model in an ordered list |
| Load balancing | Spreads traffic across keys, regions, deployments, providers |
| Retries with backoff | Exponential backoff plus jitter, honoring `Retry-After` |
| Rate-limit handling | Detects 429s, cools down throttled deployments, reroutes |
| Virtual keys + budgets | Per-key/team keys with dollar budgets and hard caps |
| Spend tracking | Token- and dollar-level attribution per key/team/model |
| Observability | Centralized traces: who called, what was tried, why it failed, what won |
| Caching | Exact-match and semantic caching to cut cost and latency |
| Guardrails / PII | Prompt/response filtering and redaction at the choke point |

The mental model: a gateway converts an `N providers x M concerns` glue problem in application code into a single policy-enforced choke point. The cost is one extra network hop and a component you must keep highly available, which is the central tension below.

---

## Routing Strategies

Routing strategies form a ladder of increasing complexity and overhead. As a rough sense of scale (vendor/practitioner figures, order-of-magnitude only): rule-based routing adds under ~1ms, embedding/semantic routing ~5ms, and heavier ML classifiers or an LLM-as-router ~50-100ms, all against typical model latencies of 500-2000ms.

| Strategy | Decides by | Best for | Overhead | Failure modes |
|----------|-----------|----------|----------|---------------|
| Static / manual | Hard-coded model per route | Simple, predictable workloads | ~0ms | Brittle; no adaptation to outages or price changes |
| Task-based | Task type maps to a model | Known task taxonomy | <1ms | Misclassification; taxonomy goes stale |
| Cost-based | Cheapest model meeting constraints | Cost ceilings, batch work | <1ms | Can starve quality; cheapest may rate-limit |
| Latency-based | Lowest observed/expected latency | Real-time UX | ~1-5ms | Herding onto one fast endpoint; cold-start skew |
| Capability-based | Required capability (long context, vision, tools) | Heterogeneous needs | 1-5ms | Over-provisioning to the "best" model |
| Semantic | Embed the request, match to a route | Intent dispatch, mixed difficulty | ~5-50ms | Embedding drift; ambiguous tail; threshold tuning |
| LLM-as-router | A small LLM classifies and picks | The ambiguous tail embeddings miss | ~50-100ms | Adds an LLM call with its own cost and failure surface |
| Cascade | Cheap model first, escalate on low confidence | Cost-quality at scale | Sequential | Double-spend on escalated queries; noisy confidence |

Two notes that matter in practice. **Semantic routing** picks a route by embedding similarity without calling an LLM; a common production pattern is a two-stage hybrid that handles the confident majority semantically and sends the ambiguous tail to an LLM-as-router. **Cascades** are the highest-leverage cost play: RouteLLM (UC Berkeley / LMSYS, ICLR 2025) reported reaching ~95% of a frontier model's quality while routing only a fraction of calls to it, with cost reductions in the 45-85% range. Read that range as a benchmark-specific ceiling, not a guarantee; the escalation rate is the live cost variable, and a cascade that escalates most traffic saves little.

Routing and fallback compose: route for cost/quality/latency on the happy path, fall back across providers for reliability on the unhappy path. A cascade is essentially quality-driven routing with reliability fallback built in.

---

## Fallback and Reliability

This is the section that answers the Datadog data directly: rate limits cause most production failures, and fallback machinery is the antidote.

**Fallback chains** retry a request against the next provider/model whenever the primary returns a *retryable* failure (429 rate limit, 5xx, timeout, model-not-found). Only retry 429 and 5xx-class errors; never retry 400-class client errors, which just waste quota.

**Retries done right** follow three rules: exponential backoff (wait longer each failure), jitter (randomize the wait so clients do not retry in lockstep and create a thundering herd that worsens the outage), and honor `Retry-After` (both OpenAI and Anthropic return it on 429s; use `max(retry_after, computed_backoff)`). Ignoring `Retry-After` is the most common backoff bug.

**Circuit breakers** track endpoint health globally and open when the failure rate crosses a threshold, so you stop hammering a dead or throttled provider on every request. Pair with cooldowns that park a failing deployment for a fixed window before auto-recovery.

**Load balancing** across multiple keys, regions, and providers multiplies your effective rate-limit headroom, which is the most direct structural fix for capacity failures.

The caution to internalize: **blind retries amplify outages** by adding load during a failure. Backoff, jitter, circuit breakers, and `Retry-After` are what separate a gateway that fixes rate limits from one that worsens them. See [Reliability Patterns](../13-reliability-and-safety/03-reliability-patterns.md).

---

## The 2026 Tool Landscape

Treat versions and exact feature claims as point-in-time, and note that several comparison figures below come from vendor marketing.

- **LiteLLM** (open-source, self-hosted proxy plus SDK) is the de-facto standard: 100+ providers behind an OpenAI-format API, routing strategies (latency, usage, cost, least-busy), ordered fallbacks, virtual keys with dollar budgets, and native OpenTelemetry. The common critique is that YAML config strains at enterprise-governance scale.
- **OpenRouter** (managed aggregator) gives the fastest breadth and zero ops across 200+ models, with pass-through provider pricing (a documented BYOK fee applies). The tradeoffs are an external hop and data leaving your perimeter.
- **Portkey** (managed, with a self-host tier) positions as a full control plane: routing, fallbacks, token-level observability, semantic caching, and guardrails.
- **Cloudflare AI Gateway** (managed, edge) is strong on observability and caching with sequential provider fallback, but lighter on routing logic and budget enforcement.
- **Kong AI Gateway** (API-management platform) brings LLM routing (including semantic), retry/fallback, semantic caching, and a PII sanitizer into a mature gateway; best when you already run Kong.
- **Envoy AI Gateway** (open-source, CNCF ecosystem) offers infra-grade priority-based fallback, retries, and timeouts, Kubernetes-native.
- **Cloud-native** routers (AWS Bedrock intelligent prompt routing, Google Vertex) integrate tightly with their clouds; strict per-account rate limits still make a multi-provider layer useful.

**Self-hosted vs managed**, the core tradeoffs: self-hosted (LiteLLM, Kong, Envoy) keeps data in your perimeter, gives full control, and is required for strict compliance, but *you* must make the proxy highly available or it becomes the single point of failure. Managed (OpenRouter, Portkey, Cloudflare) is near-zero ops and fast to adopt, but data transits a third party and you inherit their availability as a hard dependency.

---

## Architecture Patterns

- **Where it sits:** between application services and providers, as a horizontally scaled service or sidecar. All LLM traffic flows through it, so it inherits the reliability requirements of any critical-path infrastructure.
- **The network-hop tax:** the gateway adds one hop. Mitigate by co-locating it with your app (same region/VPC/cluster) so the hop is sub-millisecond, keeping the proxy thin, and caching aggressively so hits skip the provider entirely. The hop is modest against 500-2000ms model latency, but real under load.
- **Do not let the gateway become the SPOF:** this is the central architectural risk. Run multiple stateless replicas behind a load balancer, externalize shared state (Redis for rate-limit and usage counters, a database for keys and spend), and health-check the replicas. A gateway that centralizes everything but runs as one instance has simply *moved* your single point of failure.
- **Multi-region:** deploy replicas per region, route region-locally, and use cross-region/cross-provider fallback so a regional outage fails over elsewhere.
- **Caching and observability:** exact-match for identical prompts, semantic caching for near-duplicates, and OpenTelemetry spans carrying user/key, models attempted, failure reasons, the winning fallback, per-step latency, and exact cost. This is what makes the rate-limit failure mode visible instead of mysterious. See [Observability](../14-evaluation-and-observability/02-observability.md).

---

## Do You Need a Gateway Yet?

**Probably not** (use a thin in-app abstraction) when you have a single provider, a prototype, or one or two providers behind a small wrapper that still fits in your head. Direct SDK calls are simpler with fewer moving parts. This is the same thin-layer idea argued in [Navigating Framework Churn](../09-frameworks-and-tools/12-navigating-framework-churn.md).

**You do** when several of these hold: 3+ models or multiple providers (the 69% majority); multiple teams sharing model access who need virtual keys, budgets, and spend attribution; rate-limit errors actually hitting you; scattered retry/fallback/key-management logic already in your codebase (the smell that adoption stops being premature); or you cannot answer "which provider served this request?" or "what did each team spend?" without a side project.

**Build vs buy:** a thin in-app abstraction is best at 1-2 providers; self-hosting an open-source gateway (LiteLLM first) is best when you need data residency or deep control and have platform-team bandwidth; buying managed is best when you want the capabilities without owning the infra and can accept the external dependency.

**Rollout** in order: start in shadow/observe mode, move non-critical workloads first, add virtual keys and budgets to get attribution before enforcement, add fallback chains targeting your worst rate-limit offenders, then layer in caching and routing once reliability is solid, and make the gateway highly available before it becomes load-bearing.

---

## Interview Questions

### Q: Rate-limit errors are your top production failure. How does a gateway help, and how could it make things worse?

**Strong answer:**
A gateway helps by load-balancing across multiple keys, regions, and providers (which multiplies rate-limit headroom) and by failing over to an alternate provider on a 429 through an ordered fallback chain, so a single provider's throttling stops being a hard outage. It also makes the failure visible: centralized traces show which provider was tried and why it failed. It makes things worse if retries are naive: blind, immediate retries add load during the exact moment the provider is overloaded, a thundering herd that deepens the outage. The fixes are exponential backoff with jitter, honoring the provider's `Retry-After` header, and a circuit breaker that stops hammering a dead endpoint globally rather than retrying per request. And the gateway itself must be highly available with externalized state, or it just relocates the single point of failure.

### Q: When is a full gateway overkill, and what would you do instead?

**Strong answer:**
For a single provider or a prototype, a gateway is overkill; it adds a network hop and an HA burden for capabilities you do not need yet. I would use a thin in-app abstraction: depend on the provider SDK behind a small interface of my own, with a basic fallback chain and backoff. I would adopt a real gateway once I cross into multiple providers, multiple teams needing budgets and spend attribution, or recurring rate-limit pain, which is roughly when retry and key-management logic starts getting copy-pasted across services. Even then I would roll it out in shadow mode first and make it highly available before it became load-bearing.

---

## References

- Datadog, [State of AI Engineering 2026](https://www.datadoghq.com/state-of-ai-engineering/) and the [press release](https://www.datadoghq.com/about/latest-news/press-releases/datadog-state-of-ai-engineering-report-2026/)
- [LiteLLM routing docs](https://docs.litellm.ai/docs/routing) and [load balancing](https://docs.litellm.ai/docs/proxy/load_balancing)
- RouteLLM, [LMSYS blog](https://www.lmsys.org/blog/2024-07-01-routellm/) and [GitHub](https://github.com/lm-sys/routellm)
- [Envoy AI Gateway: provider fallback](https://aigateway.envoyproxy.io/docs/0.5/capabilities/traffic/provider-fallback/)
- [Kong AI Gateway docs](https://developer.konghq.com/ai-gateway/)
- [OpenRouter BYOK docs](https://openrouter.ai/docs/guides/overview/auth/byok)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **AI Gateway** | A proxy layer that sits between your application and LLM providers, exposing a single unified API and handling cross-cutting concerns | Centralizes routing, fallback, rate-limit handling, cost tracking, and observability for all LLM traffic |
| **LLM Proxy** | Another name for an AI gateway; a server that forwards LLM requests on behalf of application services | Decouples application code from provider-specific SDKs and policies |
| **Control Plane** | The infrastructure layer that manages how requests are routed, secured, and governed, separate from the data path | Applies consistent policy across all model calls without burdening application code |
| **OpenAI-compatible API** | An API surface that follows the same request/response format as OpenAI's chat completions endpoint | Lets applications switch providers without code changes by normalizing the interface |
| **Virtual Keys** | Provider-independent API keys issued by the gateway, each mapped to dollar budgets and usage caps | Enables per-team or per-feature spend control without exposing real provider credentials |
| **Fallback Chain** | An ordered list of providers or models the gateway tries in sequence when the primary returns a retryable error | Converts a single-provider failure into a transparent retry on an alternate backend |
| **Circuit Breaker** | A mechanism that stops sending requests to a failing or throttled endpoint once its error rate crosses a threshold | Prevents thundering-herd amplification by globally cooling off a dead provider |
| **Exponential Backoff** | A retry strategy where the wait time between attempts doubles after each failure | Reduces retry storm load on an already-overloaded provider |
| **Jitter** | Random variation added to backoff wait times so multiple clients do not retry in synchronized lockstep | Smooths out retry bursts that would worsen a provider outage |
| **Retry-After** | An HTTP header that providers like OpenAI and Anthropic return on 429 errors, specifying how long to wait before retrying | The authoritative signal for when a rate-limited provider is ready to accept traffic again |
| **429 (Rate Limit Error)** | An HTTP status code returned by a provider when a client has exceeded its allowed request rate | The most common production failure mode for LLM APIs; the primary trigger for fallback logic |
| **Semantic Routing** | Routing a request by embedding it and matching it to a pre-defined route by similarity, without calling an LLM | Dispatches requests to the right model or handler based on intent at low latency and low cost |
| **Cascade Routing** | A strategy that sends requests to a cheap model first and escalates to a more capable model only if confidence is low | Achieves near-frontier quality while routing only a fraction of calls to expensive models |
| **RouteLLM** | An open-source UC Berkeley / LMSYS framework for cascade-based LLM routing | Demonstrated 45-85% cost savings at ~95% quality retention on standard benchmarks |
| **Load Balancing** | Distributing traffic across multiple API keys, regions, or providers to spread the load evenly | Multiplies effective rate-limit headroom and avoids hot spots on a single endpoint |
| **SPOF (Single Point of Failure)** | A component whose failure brings down the entire system | The gateway itself must be made highly available to avoid becoming the new SPOF |
| **OpenTelemetry** | An open-source observability framework for generating, collecting, and exporting traces, metrics, and logs | Provides vendor-neutral distributed tracing of every LLM request through the gateway |
| **LiteLLM** | A popular open-source self-hosted gateway supporting 100+ providers behind an OpenAI-format API | The de-facto standard for self-hosted LLM routing; supports fallbacks, virtual keys, and native OpenTelemetry |
| **OpenRouter** | A managed aggregator giving access to 200+ models through a single API with pass-through provider pricing | Fastest way to gain breadth across many models with zero operational overhead |
| **Portkey** | A managed AI gateway with routing, fallbacks, semantic caching, token-level observability, and guardrails | Full control-plane capabilities without self-hosting infrastructure |
| **Cloudflare AI Gateway** | A managed, edge-deployed gateway strong on observability and caching with sequential provider fallback | Low-ops option for teams already on Cloudflare's edge network |
| **Kong AI Gateway** | An LLM routing and fallback layer built into the Kong API management platform | Best choice for teams already running Kong that want LLM routing added to existing gateway infrastructure |
| **Envoy AI Gateway** | An open-source, CNCF-ecosystem gateway offering Kubernetes-native priority-based fallback and retries | Infrastructure-grade option for teams already using Envoy or Istio service mesh |
| **Exact-Match Caching** | Caching a full LLM response keyed on the literal request string | Zero false-positive cache hits; highly effective for identical repeated queries |
| **Semantic Caching** | Caching LLM responses and serving cached answers when a new query is sufficiently similar by embedding distance | Increases cache hit rate for natural-language traffic beyond what exact-match alone achieves |
| **BYOK (Bring Your Own Key)** | A model where a managed service uses the customer's own provider API key rather than the vendor's key | Keeps rate-limit quota and billing under the customer's own provider account |
| **Spend Attribution** | Assigning token and dollar costs to the specific team, feature, or tenant that incurred them | Enables per-feature unit economics and chargeback without manual reconciliation |

---

*Previous: [CI/CD for LLM Applications](02-cicd.md) · Next: [FinOps and Token Economics](04-finops-and-token-economics.md)*
