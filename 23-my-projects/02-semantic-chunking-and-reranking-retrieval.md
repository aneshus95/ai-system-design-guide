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

This is the crisp story that makes the project defensible — the two metrics measure different things, and each intervention targets one.

- **Context recall** = *did we retrieve everything the answer needs?* (Coverage.) RAGAS: fraction of the reference answer's claims that are attributable to the retrieved context. **Better chunking + a wider hybrid net pull more of the required evidence into the retrieved set → recall ↑.**
- **Context precision** = *is the retrieved context mostly signal, and is the signal ranked high?* RAGAS computes a **rank-weighted average precision** over per-chunk usefulness judgments. **A reranker only reorders stage-1 candidates — it can't add new documents — so it pushes relevant chunks up and noise down → precision ↑, while recall is capped by stage 1.**

```
  chunker + hybrid retriever ──► fills the candidate set ──► RECALL  (+10%)
  cross-encoder reranker     ──► reorders the candidates ──► PRECISION (+15%)
```

Sources: [RAGAS — Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) · [RAGAS — Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/)

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

*Previous: [Keystroke Dynamics](01-keystroke-dynamics-biometric-verification.md) | Next: [LangGraph Coding Agent](03-langgraph-coding-agent-with-rag.md) | Up: [Guide Home](../README.md)*
