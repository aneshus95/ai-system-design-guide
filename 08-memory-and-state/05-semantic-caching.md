# Semantic Caching

Caching has evolved from exact string matching to **Semantic Matching**. Semantic caching reduces costs by **30-70%** and cuts latency from seconds to milliseconds by reusing completions for "equivalent" queries.

## Table of Contents

- [Exact Cache vs. Semantic Cache](#vs)
- [The Semantic Matching Pipeline](#pipeline)
- [RedisVL and GPTCache](#tech-stack)
- [Evaluation: Hit Rate vs. Hallucinated Drift](#eval)
- [Multimodal Semantic Caching](#multimodal)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Exact Cache vs. Semantic Cache

| Feature | Exact Cache (Redis/Memcached) | Semantic Cache (RedisVL/Qdrant) |
|---------|-------------------------------|---------------------------------|
| **Key** | Hashed query string | Query embedding vector |
| **Match**| 100% string identity | Cosine Similarity > Threshold |
| **Efficiency**| Low (Minor typos break cache) | High (Understands intent) |
| **Risk** | Zero | Semantic Drift (Returning wrong answer) |

---

## The Semantic Matching Pipeline

1. **Embed**: The incoming query is converted into a vector (e.g., using `text-embedding-3-small`).
2. **Search**: Search the cache for the nearest neighbor.
3. **Threshold Check**: If `distance < 0.05` (very similar), return the cached result.
4. **LLM Verification**: For high-stakes queries, a tiny "Verifier Model" (e.g., GPT-5.5-mini, Claude Haiku 4.5) checks if the cached response actually answers the new query.
5. **Update**: If no hit, call the LLM and store the new result in the vector cache.

---

## RedisVL and GPTCache

Standard stack:
- **RedisVL**: Provides low-latency vector search directly within a Redis instance.
- **Hybrid Caching**: Using Redis for both metadata (keys) and vector payloads.
- **TTL**: Semantic caches should have a TTL (Time-To-Live). The common pattern is **Dynamic TTL**: popular answers live longer while "stale" information is evicted regularly.

---

## Multimodal Semantic Caching

With native multimodal frontier models (Gemini 3.1 Pro, GPT-5.5, Claude Opus 4.7), we now cache **Image and Audio queries**.
- **Visual Similarity**: Caching the description of an image if a semantically similar image was processed before.
- **Audio Fingerprinting**: Caging transcripts for similar voice commands.

---

## Interview Questions

### Q: What is "Semantic Drift" in caching, and how do you prevent it?

**Strong answer:**
Semantic Drift occurs when the similarity threshold is too loose (e.g., 0.8 instead of 0.95). A query like *"How do I fix my car?"* might match a cached response for *"How do I wash my car?"*. To prevent this, we use **Multi-Stage Validation**: 1) Vector similarity check, 2) **Entity-Match check** (ensures both queries involve "Car" and the same "Verb"), and 3) **Threshold Tightening**: for technical or medical queries, we require $>0.98$ similarity to return a cached result.

### Q: Why is a Semantic Cache sometimes *more* expensive than a raw LLM call at low volume?

**Strong answer:**
Because a semantic cache requires its own **Embedding API call** and **Vector Search query**. If the embedding model costs $0.02 and the search takes 100ms, and your primary LLM call is only $0.05 and takes 500ms, the relative savings are small. Semantic caching only becomes a significant win at **High Scale** (millions of requests) where the cache hit rate is high enough to offset the "Embedding Tax" and drastically reduce aggregate latency.

---

## References
- Redis. "RedisVL: Python Client for Redis Vector Library" (2025)
- Akiba et al. "GPTCache: A Library for Creating Semantic Cache" (2024/2025)
- Google Cloud. "Generative AI Caching Patterns" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Semantic Cache** | A cache that stores LLM responses keyed by query meaning (embedding vector) rather than exact text | Allows cache hits even when two queries are worded differently but ask the same thing |
| **Exact Cache** | A traditional cache (Redis, Memcached) that matches queries by exact string or hash equality | Fast and simple but misses semantically equivalent queries that differ by even one character |
| **Embedding Vector** | A numerical representation of text in high-dimensional space produced by an embedding model | The key used in a semantic cache; similar queries produce vectors that are geometrically close |
| **Cosine Similarity** | A measure of the angle between two vectors; a score of 1.0 means identical direction (same meaning) | The distance metric used to decide whether an incoming query is close enough to a cached entry |
| **Similarity Threshold** | A minimum cosine similarity score (e.g., 0.95) required before a cached response is returned | Controls the trade-off between cache hit rate and the risk of returning a wrong answer |
| **Semantic Drift** | Returning a cached answer to a query that is similar but not semantically equivalent, producing a wrong result | The primary risk of semantic caching; prevented by tight thresholds and multi-stage validation |
| **RedisVL** | A Python client that adds vector search capabilities directly to Redis | Enables low-latency semantic cache lookups without a separate vector database |
| **GPTCache** | An open-source library for building semantic caches in front of LLM APIs | Simplifies wiring up embedding, search, and cache storage in a single pipeline |
| **TTL (Time-To-Live)** | A setting that automatically expires a cache entry after a fixed duration | Prevents stale cached answers from being served when underlying facts have changed |
| **Dynamic TTL** | A caching strategy where frequently accessed entries get longer TTLs and rarely accessed entries expire sooner | Balances freshness against cache efficiency by keeping popular answers alive longer |
| **Nearest Neighbour Search** | Finding the vector in the cache that is geometrically closest to the incoming query vector | The core retrieval operation of a semantic cache |
| **Embedding Tax** | The extra API cost and latency of calling an embedding model on every incoming query before cache lookup | The overhead that makes semantic caching unprofitable at low request volumes |
| **Cache Hit Rate** | The fraction of incoming requests served from the cache rather than the LLM | The key metric that determines whether a semantic cache actually saves money |
| **Verifier Model** | A small, cheap LLM called after a cache hit to confirm the cached answer actually addresses the new query | Adds a safety check for high-stakes queries where semantic drift would be costly |
| **Multi-Stage Validation** | A pipeline of checks (vector similarity → entity match → threshold tightening) before returning a cached result | Reduces semantic drift risk while keeping the cache useful for clearly equivalent queries |
| **Entity-Match Check** | Verifying that both the cached query and the incoming query reference the same key entities (nouns, verbs) | Catches cases where two queries are geometrically close but involve different subjects |
| **Multimodal Semantic Caching** | Extending semantic caching to image and audio queries, not just text | Reduces redundant LLM calls when users submit visually or acoustically similar inputs |
| **Audio Fingerprinting** | Creating a compact representation of an audio clip to detect similar voice commands | Used in multimodal caches to match semantically identical spoken queries |
| **Visual Similarity** | Detecting that two images are semantically equivalent (e.g., photos of the same object) using embedding distance | Enables caching of image-processing LLM responses for similar images |
| **Cache Miss** | A lookup that finds no sufficiently similar entry, requiring a fresh LLM call | The case where the cache provides no benefit; the response is then stored for future reuse |
| **text-embedding-3-small** | OpenAI's lightweight embedding model used to convert queries into vectors cheaply | A common choice for the embedding step in a semantic cache pipeline due to low cost per token |

*Next: [State Management Patterns](06-state-management-patterns.md)*
