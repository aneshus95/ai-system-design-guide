# RAG Fundamentals

How RAG evolved from naive vector search to agentic and graph-based retrieval. When to choose RAG vs. long context, and the three retrieval gaps that cause production failures.

Retrieval-Augmented Generation (RAG) is the architectural pattern of providing an LLM with external, verifiable context to ground its responses. It has evolved from "simple vector search" into a multi-stage reasoning pipeline: hybrid retrieval, reranking, contextual chunking, and agentic loops are now table stakes for production. Deeper material lives in [Chunking Strategies](02-chunking-strategies.md), [Vector Databases](04-vector-databases.md), [Reranking](06-reranking-strategies.md), [Contextual Retrieval](10-contextual-retrieval.md), [ColBERT Late Interaction](11-late-interaction-colbert.md), and the [GraphRAG reframe](07-graph-rag.md).

## Table of Contents

- [The Core Philosophy: Grounding vs. Training](#philosophy)
- [The RAG Taxonomy](#taxonomy)
- [RAG vs. 2M Context (The Hybrid Era)](#rag-vs-long-context)
- [The Retrieval Quality Gap](#quality-gap)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Core Philosophy: Grounding vs. Training

| Aspect | Fine-Tuning | RAG |
|--------|-------------|-----|
| **Knowledge Type** | Internalized (Weights) | externalized (Context) |
| **Update Cycle** | High Cost (Retraining) | Zero Cost (Update DB) |
| **Attribution** | None (Black box) | Explicit (Citations) |
| **Privacy** | Hard to "Unlearn" | Easy to filter/delete |

**Rule of thumb**: Fine-tuning is for **Form** (style, tone, syntax); RAG is for **Fact** (knowledge, data, grounding).

---

## The RAG Taxonomy

Production RAG systems are categorized by their "Agentic Depth":

### 1. Naive RAG (Retrieve-then-Generate)
- **Flow**: User Query -> Vector Search -> Top-K -> LLM.
- **Status**: Deprecated for production due to "Retrieval Gap" and low precision.

### 2. Advanced RAG (Multi-Stage)
- **Flow**: Query Transformation -> Hybrid Search -> Reranking -> LLM.
- **Key Nuance**: Uses **RRF (Reciprocal Rank Fusion)** to combine keyword and semantic results.

### 3. Agentic RAG (Loop-based)
- **Flow**: Agent analyzes query -> Decides which tools/indices to search -> Evaluates results -> Re-retrieves if info is missing.
- **Techniques**: Self-RAG, Corrective RAG (CRAG).

### 4. GraphRAG (Structured context)
- **Flow**: Extract entities/relationships -> Build Knowledge Graph -> Traverse graph to find "connected knowledge."
- **Win**: Solves "Aggregative Questions" (e.g., "Summarize all legal risks across 50 documents").

### The through-line: each level buys accuracy with latency, cost, and complexity

These four are a **ladder, not a menu of equals**. Each rung exists to fix the failure of the one below it, and you pay for that fix in latency, tokens, and operational complexity. The senior move is to **match the depth to the query difficulty**, not to reach for the fanciest by default.

```
  capability / cost
       ^
 GraphRAG |  connect facts across many docs  (global / "aggregative" questions)
 Agentic  |  loop + self-correct: re-query, switch index, verify  (multi-hop, ambiguous)
 Advanced |  precision: hybrid search + rerank  (the production default)
 Naive    |  one-shot retrieve-then-generate    (demos, simple FAQ)
       +-------------------------------------------------> query difficulty
```

| Level | Fixes the problem of... | Best for | Cost / latency |
|-------|-------------------------|----------|----------------|
| **Naive** | (baseline) | simple FAQ, demos | lowest — 1 LLM call |
| **Advanced** | low precision; keyword-vs-semantic mismatch | most production RAG | + rerank step |
| **Agentic** | the Retrieval Gap — missing info, multi-hop | ambiguous / multi-step queries | high — several LLM calls in a loop |
| **GraphRAG** | chunk retrieval can't *connect* scattered facts | global / aggregative questions | highest — build + maintain a graph |

**The technique names, decoded:**
- **RRF (Reciprocal Rank Fusion)** — merges the keyword (BM25) and semantic (vector) result lists by each item's *rank position*, not its raw score, so you never have to force two incomparable score scales to agree. Robust and near parameter-free — the default fusion for hybrid search (see [Hybrid Search](05-hybrid-search.md)).
- **Self-RAG** — the model is trained to decide *whether* retrieval is even needed and to critique its own retrieved passages ("relevant? is my answer supported?"), retrieving again if not.
- **Corrective RAG (CRAG)** — grades the retrieved set; if it's weak, it *falls back* (e.g., to a web search) and rewrites the query instead of answering from bad context (both: see [Agentic RAG](08-agentic-rag.md)).
- **GraphRAG** — pre-extracts entities and relationships into a knowledge graph so a query can *traverse* connections ("which risks recur across all 50 contracts?") that flat top-k chunking would only ever see one fragment of at a time.

**Interview soundbite:** *"I default to Advanced RAG — hybrid search plus a reranker — and only climb to Agentic when queries are multi-hop or the first retrieval can miss, or to GraphRAG when the questions are global/aggregative. Each level trades latency and cost for recall on harder queries; you match depth to the query, not to the demo."*

The four variants by agentic depth:

```mermaid
flowchart TD
    A[User query] --> B{RAG variant}
    B -->|Naive| C[Vector search]
    C --> CG[LLM generate]
    B -->|Advanced| D[Query rewrite]
    D --> E[Hybrid search]
    E --> F[Rerank]
    F --> FG[LLM generate]
    B -->|Agentic| G[Agent analyzes query]
    G --> H[Pick index or tool]
    H --> I[Retrieve]
    I --> J{Sufficient}
    J -->|No| G
    J -->|Yes| JG[LLM generate]
    B -->|GraphRAG| K[Extract entities]
    K --> L[Traverse KG]
    L --> LG[LLM generate]
```

---

## RAG vs. 2M Context (The "Hybrid Era")

With context windows like Gemini 1.5 Pro (2M+) and Claude Sonnet 4.6 (1M+), RAG is changing.

- **In-Context RAG (ICR)**: For datasets < 50k tokens, we skip the vector DB and put EVERYTHING in the prompt.
- **Prompt Caching**: Makes Long-Context RAG 90% cheaper by caching the "Background Knowledge" on the GPU.

**Architectural Decision**: 
- If your corpus is > 100k tokens and dynamic: Use **Standard RAG**.
- If your corpus is < 100k tokens: Use **In-Context RAG**.

Decision tree for picking between standard RAG and in-context RAG:

```mermaid
flowchart TD
    A[Corpus size] --> B{Tokens}
    B -->|Under 50k| C[In-context RAG<br/>Use prompt caching]
    B -->|50k to 100k| D{Update frequency}
    D -->|Daily or less| C
    D -->|Hourly| E[Standard RAG]
    B -->|Over 100k| E
    E --> F{Need cross-doc aggregation}
    F -->|Yes| G[GraphRAG]
    F -->|No| H[Advanced or Agentic RAG]
```

---

## The Retrieval Quality Gap

The "Retrieval Gap" is the #1 cause of RAG failure — but it is really **three different gaps, one per pipeline stage**. A chunk must survive all three to help the answer:

- **Gap 1 — Recall (never retrieved).** The right chunk didn't make the top-k. Two distinct causes that need *different* halves of retrieval:
  - **Vocabulary mismatch** — query and doc use different words for the same thing ("fast cars" vs "Porsche 911"). Keyword/BM25 shares no tokens, so it misses → this is a job for **dense / semantic** retrieval.
  - **Exact-term miss** — a rare code, name, or acronym the embedding mis-tokenizes ("HTTP 429", `NVIDIA_VISIBLE_DEVICES`). Dense misses it → this is a job for **sparse / BM25** keyword search.
  - Running *both* = **Hybrid Search** — which is why hybrid is the production recall baseline.
- **Gap 2 — Precision (retrieved but ranked low).** The right chunk was fetched but buried below junk, because bi-encoder cosine similarity is only a rough relevance proxy. Fixed by a **Reranker (cross-encoder)** that reads query + chunk *together* and reorders the candidates.
- **Gap 3 — Attention / Lost-in-the-Middle (in the prompt but ignored).** The chunk made it into the context, but the LLM's attention skips over it. Fixed by **Context Compression / reordering** — shrink to high-signal tokens and place the key chunk at the start or end.

### The deeper frame: three gaps = three *different stages* failing

Each gap fails at a different stage, so "RAG doesn't work" is never a single problem:

```
  question
     |
     v  [1] RETRIEVE  -- did we fetch the right chunk?      <- Gap 1 recall     -> Hybrid (dense + sparse)
     v  [2] RANK      -- is it at the TOP of the results?   <- Gap 2 precision  -> Reranker (cross-encoder)
     v  [3] GENERATE  -- does the LLM actually read it?     <- Gap 3 attention  -> Compression / reorder
     |
     v
   answer
```

| Gap | Stage | Symptom | Root cause | Fix | Metric to watch |
|-----|-------|---------|------------|-----|-----------------|
| **1a — Recall: vocabulary mismatch** | Retrieve | right chunk not in top-k; query &amp; doc use different words ("fast cars" vs "Porsche 911") | keyword can't bridge synonyms | **Dense / semantic** (the semantic arm of **Hybrid**) | recall@k |
| **1b — Recall: exact-term miss** | Retrieve | right chunk not in top-k; a rare code/name (`HTTP 429`) | embedding mis-tokenizes rare tokens | **Sparse / BM25** (the keyword arm of **Hybrid**) | recall@k |
| **2 — Precision** | Rank | retrieved, but buried below junk | bi-encoder cosine is a rough relevance proxy | **Reranker** (cross-encoder) | nDCG / MRR |
| **3 — Attention** | Generate | in the prompt, but the LLM ignores it | lost-in-the-middle attention gradient | **Compression / reorder** (key chunk to the edges) | faithfulness / answer quality |

See [Hybrid Search](05-hybrid-search.md), [Reranking Strategies](06-reranking-strategies.md), and [Context Engineering](../05-prompting-and-context/05-context-engineering.md) for each fix in depth.

**Careful with the labels (common interview trap):** a *vocabulary* mismatch ("fast cars" -> "Porsche 911") is a **recall** problem solved by **dense retrieval** — keyword search can't bridge synonyms, so this is exactly what the semantic arm of hybrid is for. A **reranker** is a *separate*, **precision** fix that only helps a chunk that was **already retrieved** but mis-ranked. Don't reach for a reranker to solve a retrieval miss.

**The senior insight — the gaps are ordered, so fix them in order.** A reranker (Gap 2) can't rescue a chunk that retrieval never fetched (fix Gap 1 first), and compression (Gap 3) can't help a chunk ranked so low it never entered the prompt. So *diagnose which stage is leaking* — measure **recall@k** first, then ranking (nDCG/MRR), then generation faithfulness — rather than reflexively bolting on a reranker.

---

## Interview Questions

### Q: Why would you still use RAG if frontier models ship 1M-2M token contexts?

**Strong answer:**
Three tiers of reasons:
1. **Cost and Latency**: Even with prompt caching, re-reading 2M tokens for every new user query is significantly more expensive and has higher TTFT (Time to First Token) than retrieving 5 relevant chunks (approx. 2k tokens). 
2. **Freshness**: RAG can access real-time APIs (Stock prices, News) which cannot be statically embedded in a context window.
3. **Scale**: Enterprise datasets (SharePoint, Terabyte logs) exceed even 2M tokens. RAG serves as the "Filter" to find the relevant 0.01% of data that *should* go into that high-value context window.

### Q: What is "Agentic RAG" and how does it differ from "Advanced RAG"?

**Strong answer:**
Advanced RAG is a **deterministic pipeline** (Linear: Rewrite -> Search -> Rerank). Agentic RAG is a **stochastic loop**. In Agentic RAG, the model is given tools to decide *how* to retrieve. For example, if the agent finds that the retrieved documents are irrelevant, it can decide to "Search Google" or "Query the SQL database" instead. It essentially adds a "Reasoning step" before and after retrieval to ensure the context is sufficient to answer the prompt.

---

## Key Takeaways

- Naive RAG (vector search + top-K + LLM) is deprecated for production; ship Advanced RAG (hybrid + RRF + rerank) as the new baseline.
- Long context windows do not kill RAG: cost, latency, freshness, and corpus scale all push you back to retrieval even at 2M context.
- Choose by corpus size: under 50k tokens go in-context with prompt caching; over 100k go standard RAG; aggregative questions go GraphRAG.
- Most RAG failures are retrieval failures, not generation failures; diagnose the three gaps (semantic, missing context, lost-in-the-middle) before tuning prompts.
- Agentic RAG vs. Advanced RAG is a stochastic-loop vs. deterministic-pipeline choice; only adopt agentic when query patterns are too varied for a fixed pipeline.

---

## References
- Gao et al. "Retrieval-Augmented Generation for LLMs: A Survey" (2024 update)
- Microsoft. "From RAG to GraphRAG" (2024)
- Google. "Long-context LLMs as Retrievers" (2025)
- [Anthropic. "Introducing Contextual Retrieval" (Sep 2024)](https://www.anthropic.com/news/contextual-retrieval)

---

*Next: [Chunking Strategies](02-chunking-strategies.md)*
