# Retrieval Optimization — Cluster Semantic Chunker + Hybrid Retriever + Reranker

> **My project.** Re-engineered the retrieval stack of a RAG system: a **cluster-based semantic chunker** with a **keyword (BM25) retriever** that beat fixed-size and recursive-character chunking by **+10% context recall**, then added a **cross-encoder reranker** that lifted **context precision by +15%**.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Methodology](#what-i-built--methodology)
- [Deep Dive 1 — Chunking Strategies](#deep-dive-1--chunking-strategies)
- [Deep Dive 2 — Keyword Retriever & Hybrid Search (RRF)](#deep-dive-2--keyword-retriever--hybrid-search-rrf)
- [Deep Dive 3 — Reranking (Cross-Encoder)](#deep-dive-3--reranking-cross-encoder)
- [Deep Dive 4 — Why Recall and Precision Moved Separately](#deep-dive-4--why-recall-and-precision-moved-separately)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** A RAG system was returning answers that missed key facts or padded the context with irrelevant chunks. The root cause wasn't the LLM — it was **retrieval**: the naive chunker was slicing documents by character count, cutting sentences and topics in half, so the evidence an answer needed was scattered or truncated across chunks.

**Task.** Improve *what* the retriever hands the LLM — measured rigorously, not by vibes — so answers are both **complete** (nothing missing) and **focused** (little noise).

**Action.** I reworked the retrieval pipeline in three moves:
1. Replaced fixed/recursive chunking with a **cluster-based semantic chunker** that groups sentences by meaning, so a chunk is one coherent topic.
2. Added a **keyword (BM25) retriever** fused with the dense retriever, so exact terms (IDs, codes, names) that embeddings smooth over still get matched.
3. Added a **cross-encoder reranker** as a second stage to re-score and reorder the top candidates with full query-document attention.

**Result.** Measured with **RAGAS**: **+10% context recall** from the chunker + hybrid retriever, and **+15% context precision** from the reranker — a clean, attributable split (coverage vs. signal-to-noise).

---

## What I Built — Methodology

```
 documents
    │
    ▼
 ┌──────────────────────────┐
 │ CLUSTER SEMANTIC CHUNKER │  embed sentences → group by semantic
 │                          │  similarity → coherent, topic-pure chunks
 └────────────┬─────────────┘
              ▼
 ┌──────────────────────────────────────────────┐
 │           HYBRID RETRIEVAL (stage 1)          │  high RECALL
 │   BM25 (keyword) ──┐                           │
 │                    ├─► Reciprocal Rank Fusion ─┼─► top-K candidates
 │   Dense (vector) ──┘        (k = 60)           │
 └────────────────────────┬──────────────────────┘
                          ▼
 ┌──────────────────────────────────────────────┐
 │       CROSS-ENCODER RERANKER (stage 2)        │  high PRECISION
 │   re-score [query, chunk] pairs → reorder      │
 └────────────────────────┬──────────────────────┘
                          ▼
                 top-N chunks → LLM
```

**Evaluated** end-to-end with **RAGAS context recall & context precision** on a fixed eval set, comparing all chunkers under the same retriever and judge model so the deltas were attributable.

---

## Deep Dive 1 — Chunking Strategies

> **Why (the rationale):** Chunk boundaries determine what evidence survives in a single retrievable unit. Fixed/recursive splitting cuts by character count or structural separators, so one coherent argument can be scattered across two chunks — lowering context recall. Cluster semantic chunking groups sentences by meaning, keeping related evidence together so more of an answer's required claims are found in one retrieved chunk.
> **When to use:** Cluster semantic chunking pays off when documents are long and topic-diverse (encyclopedic, multi-section docs, knowledge bases). For short, uniform documents (product descriptions, FAQ entries), fixed-size or recursive splitting is often sufficient and far cheaper.
> **Nuances & gotchas:** Cluster semantic chunking discards sentence-proximity/positional context — non-adjacent sentences on the same topic merge, but sentences that naturally belong together at a fine granularity can be split. It also requires embedding every sentence at index time, which is significantly more expensive than character splitting. A NAACL 2025 paper showed it doesn't consistently beat fixed-size chunking once compute cost is accounted for and can over-fragment into very small chunks that starve the LLM.

| Method | Boundary based on | Trade-off |
|---|---|---|
| **Fixed-size** | character/token count | Cheapest; **cuts mid-thought** — splits one idea across chunks |
| **Recursive character split** (LangChain `RecursiveCharacterTextSplitter`) | structural separators `["\n\n","\n"," ",""]` in order | Respects *structure* (paragraph→sentence→word) but is **topic-blind** — doesn't know when the subject changes |
| **Semantic (breakpoint)** | cosine-similarity **drop** between adjacent sentence embeddings | Topic-aware; but variable size and can **over-fragment** into tiny chunks |
| **Cluster semantic** (mine) | **global clustering** of sentence embeddings | Groups a topic even when revisited non-adjacently; cost = every sentence embedded, and can **lose sentence-proximity/positional context** |

**In plain English:** recursive splitting is like cutting a book at chapter/paragraph marks — tidy, but it'll still split one argument across two chunks or merge two arguments into one. **Cluster semantic chunking** reads the meaning of each sentence and pools sentences about the same thing into one chunk, so more of an answer's evidence survives *inside a single retrievable unit* → recall goes up.

> **Be precise about "cluster semantic":** it means clustering sentence embeddings (e.g., k-means/agglomerative) to form topic-pure chunks — as opposed to the sequential breakpoint variant. State exactly which you built; don't over-claim.

Sources: [LangChain RecursiveCharacterTextSplitter](https://reference.langchain.com/python/langchain-text-splitters/character/RecursiveCharacterTextSplitter) · [Semantic chunking explained — Superlinked](https://superlinked.com/vectorhub/articles/semantic-chunking) · [Firecrawl — chunking strategies](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)

---

## Deep Dive 2 — Keyword Retriever & Hybrid Search (RRF)

> **Why (the rationale):** Dense retrieval smooths over rare exact tokens (product codes, error codes, API names) because embeddings represent them as similar to related concepts — losing the precise match. BM25 matches exactly on token overlap but misses paraphrase and synonymy. The two fail in *orthogonal* ways, so combining them recovers what either misses. RRF merges the two ranked lists by rank-position only, avoiding the need to normalize incompatible score scales (BM25 is unbounded; cosine is −1..1).
> **When to use:** Any retrieval task where queries mix natural language with exact identifiers — coding assistants, customer support over technical docs, enterprise search. Pure dense retrieval is usually sufficient for purely semantic queries on homogeneous prose.
> **Nuances & gotchas:** RRF's k=60 constant dampens high-rank documents; tuning k changes the fusion sensitivity. Adding BM25 increases index size and adds an extra retrieval path to maintain. If the keyword and dense results have very different coverage (e.g., one returns 100 docs, the other 10), the fusion can be lopsided. Hybrid search also raises latency because both retrievers must run in parallel.

- **BM25** = sparse, lexical retrieval (TF saturation + IDF + length normalization). Nails **exact matches** — product codes, IDs, error codes, rare named entities — but can't handle paraphrase/synonymy.
- **Dense (vector)** retrieval handles **semantic** similarity but under-weights rare exact tokens the embedding never learned well.
- **Why combine:** the two **fail in orthogonal ways**, so hybrid recovers what either misses.

**Reciprocal Rank Fusion (RRF)** merges the two ranked lists without needing compatible score scales (BM25 scores are unbounded; cosine is −1..1). It fuses **by rank only**:

```
  RRF_score(d) = Σ over lists  1 / (k + rank_i(d))          k = 60
```

A document ranked highly in *either* list gets rewarded; no score normalization required. (Cormack, Clarke & Büttcher, SIGIR 2009.)

Sources: [Cormack et al., RRF (SIGIR 2009)](https://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf) · [Weaviate — Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained)

---

## Deep Dive 3 — Reranking (Cross-Encoder)

> **Why (the rationale):** A bi-encoder must compress all possible query meanings into a single fixed document vector computed before the query exists — a lossy compression that sacrifices relevance accuracy for speed. A cross-encoder reads the full `[query, document]` pair jointly, so attention can flow between query and document tokens, producing a far more accurate relevance score. Running it over the entire corpus would be O(N) forward passes per query — infeasible — so it's applied only to the top-K from stage 1, where the added latency is bounded.
> **When to use:** Any two-stage retrieval system where stage-1 recall is good but precision is poor — top-K from the bi-encoder contains too much noise. Cross-encoders become especially valuable when queries are nuanced or ambiguous and the top-ranked chunks significantly affect answer quality.
> **Nuances & gotchas:** The cross-encoder can only reorder — it cannot retrieve a document stage 1 missed, so recall is permanently capped by stage 1's top-K. Typical latency cost is 100–300 ms extra for a top-50 to top-5 rerank. Cross-encoder gains diminish if stage-1 precision is already high, making it a poor ROI in that case.

The retriever is a **bi-encoder**: it embeds query and documents *independently*, so document vectors are computed **before the query exists** and must compress all meanings into one fixed vector → fast and high-recall, but lossy.

A **cross-encoder reranker** takes the **`[query, document]` pair as one joint input**, so **full attention runs across query and document tokens together** → a far more accurate relevance score. It's too slow to run over the whole corpus (one forward pass per pair), so it runs only over the **top-K** from stage 1.

```
 bi-encoder (stage 1):   query ──►[vec]      docs ──►[vec] ... precomputed, ANN search  →  RECALL
 cross-encoder (stage 2): [query + doc] ──► transformer ──► single relevance score       →  PRECISION
```

**The two-stage pattern = "retrieve cheaply, then rerank precisely":** stage 1 casts a wide net (top 50–100); stage 2 re-scores those pairwise and returns a precision-tuned top 5–10 to the LLM. Popular rerankers: **Cohere Rerank**, **BAAI/bge-reranker-v2-m3**. Typical gain: ~15–30% precision for ~100–300 ms added latency — consistent with my +15%.

Sources: [Pinecone — Rerankers & Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/) · [Cross-encoders & reranking — TDS](https://towardsdatascience.com/advanced-rag-retrieval-cross-encoders-reranking/)

---

## Deep Dive 4 — Why Recall and Precision Moved Separately

> **Why (the rationale):** The two metrics measure structurally different things and each intervention targets exactly one. Context recall measures whether the retrieval *set* contains all necessary evidence — improved by putting more relevant content into retrievable chunks (chunker) and casting a wider net (hybrid). Context precision measures whether the *top-ranked* chunks are relevant — improved by reordering the existing candidate set (reranker). Because the reranker only reorders and cannot add new documents, it cannot move recall; because the chunker and retriever don't control ranking order within the top-K, they have limited precision impact.
> **When to use:** This framework — attribute recall gains to upstream (chunking/retrieval) and precision gains to downstream (reranking) — applies to any two-stage RAG evaluation. It makes interventions independently attributable and justifies the architectural split.
> **Nuances & gotchas:** RAGAS context precision is rank-weighted, so it rewards pushing relevant chunks to the top even when the total relevant count doesn't change. LLM-judge metrics (RAGAS uses an LLM to label chunk usefulness) have variance across runs and judge models — the deltas are more reliable than absolute scores. A reranker that raises precision can *appear* to help recall if RAGAS's per-chunk judgment changes with position.

This is the crisp story that makes the project defensible — the two metrics measure different things, and each intervention targets one.

- **Context recall** = *did we retrieve everything the answer needs?* (Coverage.) RAGAS: fraction of the reference answer's claims that are attributable to the retrieved context. **Better chunking + a wider hybrid net pull more of the required evidence into the retrieved set → recall ↑.**
- **Context precision** = *is the retrieved context mostly signal, and is the signal ranked high?* RAGAS computes a **rank-weighted average precision** over per-chunk usefulness judgments. **A reranker only reorders stage-1 candidates — it can't add new documents — so it pushes relevant chunks up and noise down → precision ↑, while recall is capped by stage 1.**

```
  chunker + hybrid retriever ──► fills the candidate set ──► RECALL  (+10%)
  cross-encoder reranker     ──► reorders the candidates ──► PRECISION (+15%)
```

Sources: [RAGAS — Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) · [RAGAS — Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/)

---

## Key Decisions & Tradeoffs

| Decision | Options considered | Why chosen | Tradeoff accepted |
|---|---|---|---|
| **Cluster semantic chunking vs breakpoint/recursive splitting** | Fixed-size, recursive character split (LangChain default), sequential breakpoint semantic split | Cluster-based grouping keeps topic-related sentences together even when non-adjacent, giving topic-pure chunks and raising context recall | Requires embedding every sentence at index time (significantly more expensive); can lose sentence-proximity context; doesn't consistently beat fixed-size on all corpora (NAACL 2025) |
| **Hybrid BM25 + dense retrieval vs pure vector search** | Vector-only ANN retrieval, BM25-only keyword search, learned sparse (SPLADE) | The two fail orthogonally — dense misses rare exact tokens (codes, IDs, names), BM25 misses paraphrase — combining recovers what either alone would miss | Larger index (two retrieval paths to maintain), extra latency from running both in parallel, need to tune RRF k |
| **RRF fusion vs score normalization** | Min-max normalize then add scores, learned fusion weights | RRF fuses by rank only, so incompatible score scales (BM25 unbounded; cosine −1..1) need no normalization; well-validated (Cormack et al., SIGIR 2009) | RRF's k=60 dampens top-ranked docs; tuning k is another hyperparameter; learned weights can outperform RRF but require labeled training data |
| **Cross-encoder reranker as stage 2 vs single-stage bi-encoder** | Single bi-encoder at top-K, learned cross-encoder over all docs, colBERT late interaction | Cross-encoder reads `[query, doc]` jointly — full attention across tokens — giving far higher precision; restricting it to top-K keeps latency bounded | Adds 100–300 ms latency over the bi-encoder alone; cannot recover documents stage 1 missed (recall permanently capped) |
| **RAGAS as evaluation framework vs manual annotation** | Human labelers, traditional IR metrics (nDCG, MRR), custom LLM judge | RAGAS provides reproducible automated metrics (context recall, context precision) with an LLM judge so evaluation can be rerun after each change without labeler cost | LLM-judge variance means absolute scores are noisy; results depend on which judge model and prompt version are used; must hold these constant across comparisons |
| **Evaluate chunker and reranker in isolation vs end-to-end** | Only measure end-to-end answer quality (faithfulness), only measure nDCG | Isolating context recall and context precision to the relevant stage (chunker/retriever vs reranker) makes each intervention independently attributable | Two-stage evaluation is more setup work; context recall/precision don't directly measure final answer quality (faithfulness) |

---

## Production Issues & Fixes

*Representative issues for this system class — mapped to what I actually hit; tailor to your own incidents.*

**1. Embedding model updated silently, recall degraded across the board**
- **Symptom:** Context recall dropped ~8 percentage points over two weeks with no code changes; answer quality complaints increased.
- **Root cause:** The embedding provider rolled out a new default model version. Document embeddings in the index were computed with the old model; query embeddings at search time used the new one. Cosine similarity between old and new embedding spaces is meaningless — nearest-neighbor results became nearly random.
- **Fix:** Pin the embedding model version explicitly in both the indexing pipeline and the query path; rebuild the index after any model version change; version-stamp the index with the model ID so mismatches are caught at startup.
- **Prevention/monitoring:** Track context recall in a nightly regression job; alert on a drop >3% from the prior 7-day average; add an index metadata check that compares the stored model ID to the query-time model ID on startup.

**2. Cross-encoder reranker latency exceeded SLA at high QPS**
- **Symptom:** P99 latency spiked to >1.5 s during a traffic burst — the reranker was the bottleneck; everything upstream was fast.
- **Root cause:** The cross-encoder ran top-50 → top-5 reranking synchronously in the request path. At high QPS, GPU inference queue depth grew; each extra request added ~80 ms of queuing on top of the base ~250 ms inference time.
- **Fix:** Reduced top-K fed to reranker from 50 to 20 (upstream recall was sufficient to cover 20); moved the reranker to an async worker with a bounded queue; added a latency-based circuit breaker that bypasses reranking and serves the raw stage-1 top-5 when p95 reranker latency exceeds 400 ms.
- **Prevention/monitoring:** Alert on reranker p95 latency >300 ms and on circuit-breaker trip rate >1% of requests; monitor context precision separately when reranker is bypassed to quantify the precision hit.

**3. Chunk boundary error: a critical answer split across two chunks, both retrieved but neither sufficient alone**
- **Symptom:** Questions requiring a multi-part technical definition returned answers that were half right; the retrieved chunks each contained one half of the required information.
- **Root cause:** The cluster semantic chunker merged sentences topically but the topic changed mid-paragraph in a few documents, splitting a tightly coupled definition across two cluster boundaries.
- **Fix:** Post-process chunks below a minimum token threshold by merging with the adjacent chunk; add a small overlap (e.g., the last 1–2 sentences of chunk N prepended to chunk N+1) for boundary-straddling answers.
- **Prevention/monitoring:** Add a minimum chunk size guard (e.g., drop or merge chunks below 50 tokens); include chunk-size distribution in the index build report; flag chunks below threshold for manual review on new document types.

**4. Stale index — documents updated in source but old content still retrieved**
- **Symptom:** The RAG system confidently cited a policy number that had been revised two weeks earlier; users reported the system was giving outdated information.
- **Root cause:** Document re-indexing was triggered manually and had been missed; no freshness check existed between the source document store and the vector index.
- **Fix:** Implemented a daily delta-indexing job that compares document modification timestamps in the source to last-indexed timestamps in the index metadata; reindexes only changed/new documents.
- **Prevention/monitoring:** Track per-document index age; alert when any document in the index is more than N days behind its source modification date; add a "last indexed" timestamp to retrieved chunk metadata so users can see document freshness.

**5. RRF k=60 constant producing lopsided fusion when BM25 and dense lists had very different coverage**
- **Symptom:** Queries for exact product codes (BM25 excels) were not receiving the expected boost from keyword matches — dense results dominated the fused ranking.
- **Root cause:** The dense retriever returned 100 candidates; BM25 returned 8 (narrow corpus for that query type). RRF score contributions from 8 BM25 hits were overwhelmed by the larger dense list at the same k=60 dampening.
- **Fix:** Tuned top-K per retrieval path separately (BM25 top-20, dense top-50) so the sparse list was not undersized relative to the dense list; experimented with k=20 to give higher weight to genuinely top-ranked BM25 results.
- **Prevention/monitoring:** Log per-retrieval-path result counts and average RRF contribution per source; alert when BM25 contributes to fewer than 20% of final top-10 results across a rolling window of queries.

**6. Over-retrieval — large top-K passed to LLM hurt precision and increased cost**
- **Symptom:** LLM answer costs per query increased 40%; faithfulness scores dropped slightly despite high context recall; the LLM started citing irrelevant chunks.
- **Root cause:** Stage-1 top-K was set to 100 "to be safe on recall." The reranker pruned to top-10 before the LLM, but top-10 was still too many for the LLM to stay focused; token cost scaled linearly.
- **Fix:** Systematically swept top-K after reranking from 10 down to 3, measuring faithfulness and answer completeness at each; found top-5 gave the best cost/quality balance for this corpus and query mix.
- **Prevention/monitoring:** Track per-query token cost and faithfulness score; add a cost-per-query budget alert; A/B test top-K changes with a frozen eval set before rolling out.

---

## Interview Q&A

**Q: Why cluster semantic chunking over recursive character splitting?**
Recursive split respects structure but is topic-blind — it'll split one topic across chunks or merge two. Clustering sentence embeddings makes each chunk one coherent topic (even if the topic is revisited non-adjacently), so more of an answer's required claims survive in a single retrieved chunk → higher context recall.

**Q: How did you merge BM25 and vector results with incompatible score scales?**
Reciprocal Rank Fusion — drop raw scores, fuse by rank: `score(d) = Σ 1/(k+rank)`, k=60. Rank-only, so unbounded BM25 scores and bounded cosine sims don't need normalization; documents high in either list win.

**Q: Bi-encoder vs cross-encoder — why not use the cross-encoder for everything?**
A cross-encoder needs a forward pass per query-doc pair — O(N) at query time, infeasible over millions of docs. Bi-encoder embeddings are precomputed and ANN-searchable → fast, high recall. So bi-encoder/hybrid for recall over the corpus, cross-encoder for precision over the top-K.

**Q: Why did the reranker move precision (+15%) but not recall?**
It only reorders stage-1 candidates; it can't retrieve documents that weren't fetched, so recall is capped by stage 1. Context precision is rank-weighted, so pushing relevant chunks up and noise down directly raises it. Recall gains came from the chunker + hybrid retriever upstream.

**Q: What K did you retrieve before reranking?**
Have a number ready (e.g., top-50 hybrid → rerank to top-5). Larger K raises the stage-1 recall ceiling but adds cross-encoder latency; I tuned K on the recall/latency curve so the reranker had enough candidates without blowing the latency budget.

**Q: Are +10%/+15% trustworthy?**
They're RAGAS measurements on a fixed eval set with a fixed judge model, compared as relative deltas across chunkers under the same retriever. LLM-judge metrics have variance, so I hold the judge/prompt/eval-set constant and report deltas, not absolute scores.

---

## Honest Caveats

- **Semantic chunking isn't universally better.** A NAACL 2025 Findings paper ("Is Semantic Chunking Worth the Computational Cost?") showed it doesn't consistently beat simple fixed-size chunking once compute is accounted for, and can over-fragment into tiny chunks that starve the LLM. So **+10% is a dataset-specific measurement, not a universal law** — say so.
- **LLM-judge variance** — fix the judge model, prompt, and eval set; compare deltas.
- **Garbage in, garbage out** — a reranker can't recover a document stage 1 never fetched, which is exactly why the recall work had to come first.

Source: [NAACL 2025 — "Is Semantic Chunking Worth the Computational Cost?" (arXiv 2410.13070)](https://arxiv.org/html/2410.13070v1)

---

## References

- [Cormack, Clarke & Büttcher — Reciprocal Rank Fusion (SIGIR 2009)](https://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf)
- [Pinecone — Rerankers and Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/)
- [Weaviate — Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained)
- [LangChain — RecursiveCharacterTextSplitter](https://reference.langchain.com/python/langchain-text-splitters/character/RecursiveCharacterTextSplitter)
- [RAGAS — Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/) · [Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/)
- [NAACL 2025 — Is Semantic Chunking Worth the Cost? (arXiv 2410.13070)](https://arxiv.org/html/2410.13070v1)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **RAG (Retrieval-Augmented Generation)** | An architecture that retrieves relevant text chunks from a corpus and injects them into an LLM prompt before generating an answer | Grounds LLM responses in actual documents, reducing hallucination |
| **Chunking** | Splitting a document into smaller pieces before indexing so they fit within a retrieval unit | Controls the granularity of evidence the retriever can return |
| **Fixed-Size Chunking** | Splitting text by a fixed character or token count | Cheapest method; often cuts sentences and topics mid-thought |
| **Recursive Character Splitting** | Splitting at structural separators (paragraph → sentence → word) in order of preference | Respects document structure but is topic-blind |
| **Semantic Chunking** | Splitting by detecting drops in cosine similarity between adjacent sentence embeddings | Topic-aware; can over-fragment short paragraphs |
| **Cluster Semantic Chunking** | Grouping sentence embeddings globally via clustering so sentences about the same topic end up in one chunk even if non-adjacent | Improves context recall by keeping related evidence together |
| **BM25** | A lexical (keyword) search algorithm that scores documents by term frequency and inverse document frequency with length normalization | Nails exact matches (codes, IDs, rare names) that dense embeddings smooth over |
| **Dense (Vector) Retrieval** | Finding documents by comparing embedding vectors in a high-dimensional space | Handles paraphrase and synonymy but under-weights rare exact tokens |
| **Hybrid Search** | Combining keyword (BM25) and dense vector retrieval results | Recovers matches each method would miss alone |
| **Reciprocal Rank Fusion (RRF)** | A rank-based score fusion formula `score(d) = Σ 1/(k+rank)` that merges two ranked lists without needing compatible score scales | Fuses BM25 and dense rankings fairly; widely used in hybrid retrieval |
| **Bi-Encoder** | A retrieval model that embeds query and documents independently with the same encoder | Enables fast pre-computed document vectors and approximate nearest-neighbor search |
| **Cross-Encoder (Reranker)** | A model that takes a query-document pair as a single joint input and outputs a relevance score | Produces far more accurate scores than a bi-encoder; used in the second retrieval stage |
| **Two-Stage Retrieval** | First stage casts a wide net (high recall) with a bi-encoder; second stage re-scores the top-K with a cross-encoder (high precision) | Balances retrieval recall against the latency cost of exact scoring |
| **Context Recall** | The fraction of claims in a reference answer that are supported by the retrieved chunks | Measures whether the retrieved set covers everything the answer needs |
| **Context Precision** | A rank-weighted average precision over retrieved chunks — how much signal versus noise is at the top | Measures whether the most relevant chunks rank highest |
| **RAGAS** | An open-source framework for evaluating RAG pipelines with LLM-based metrics including context recall and context precision | Provides reproducible, automated RAG quality scores |
| **ANN (Approximate Nearest Neighbor)** | A fast algorithm for finding near-matches in a vector space without exhaustive search | Makes dense retrieval practical over large corpora |
| **Embedding** | A fixed-length float vector encoding the meaning of a text chunk | Enables similarity search in a vector index |
| **Top-K** | The K highest-scoring documents returned by stage 1 before reranking | Controls the recall ceiling and the latency cost of the reranking stage |

---

*Previous: [Keystroke Dynamics](01-keystroke-dynamics-biometric-verification.md) | Next: [LangGraph Coding Agent](03-langgraph-coding-agent-with-rag.md) | Up: [Guide Home](../README.md)*
