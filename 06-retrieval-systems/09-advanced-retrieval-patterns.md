# Advanced Retrieval Patterns

Beyond the basics, production RAG systems use specialized patterns to handle complex query-document gaps. These patterns are the "secret sauce" of high-precision search and are increasingly bundled into managed RAG offerings.

## Table of Contents

- [Query Decomposition (Multi-Query)](#query-decomposition)
- [Hypothetical Document Embeddings (HyDE)](#hyde)
- [Contextual Retrieval (The Anthropic Pattern)](#contextual)
- [Iterative Document Enrichment](#enrichment)
- [In-Context Reranking](#reranking)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Query Decomposition (Multi-Query)

Complex user queries are often "Compound Queries."
- **User**: "Compare our Q3 vs Q4 revenue and explain the drop."
- **Decomposition**:
  1. "Q3 Revenue"
  2. "Q4 Revenue"
  3. "Reasons for Q4 revenue variance"
- **Implementation**: Use an LLM to generate these 3 sub-queries, search the DB for ALL of them, and aggregate the context.

---

## Hypothetical Document Embeddings (HyDE)

Queries are short; documents are long. This "Asymmetry" causes retrieval failure.
- **Pattern**: 
  1. Take the user query.
  2. Ask the LLM: "Write a 1-paragraph hypothetical answer to this."
  3. **Embed the hypothetical answer** instead of the query.
- **Why?**: The hypothetical answer is in the same "Vector neighborhood" as the real documents, leading to much higher recall.

---

## Contextual Retrieval (The Anthropic Pattern)

 standardized by Anthropic in late 2024, this pattern solves **Context Dilution**.

- **The Problem**: A chunk might say "It costs $200," but without the header, we don't know "It" is a "Widget-X."
- **The Pattern**: During ingestion, for every 300-token chunk, have an LLM write a 50-token context string (e.g., "This chunk is about the pricing for Widget-X in the North American market").
- **Benefit**: Increases retrieval precision by 30-50% for fragmented data.

---

## Iterative Document Enrichment

Instead of just storing the raw document, we store "Enriched" meta-data.
- **Summary**: Store a 1-paragraph summary of the document.
- **Q&A Generation**: Generate 5 questions this document answers and embed those *with* the document.
- **Status**: Most high-end RAG systems now embed **"Questions"** rather than **"Answers"** to match the user's query intent.

---

## In-Context Reranking

With 1M-2M context windows now standard (Claude Sonnet 4.6, Gemini 3.1 Pro), **Rank-by-Context** is a viable pattern.
1. Retrieve Top 100 docs.
2. Put all 100 in the context window.
3. Ask the model: "Read these 100 docs and identify the 5 most relevant. Then, use those 5 to answer."
- **Win**: This utilizes the model's **Long Context Reasoning** to perform reranking without needing a separate Cross-Encoder model.

---

## Interview Questions

### Q: Why is HyDE (Hypothetical Document Embedding) risky for some applications?

**Strong answer:**
HyDE relies on "Hallucinating" a baseline answer to find real data. If the user's query describes something non-existent or logically impossible, the LLM will still generate a hypothetical answer. This can pull in "Incorrect but Semantically Similar" data from the database, reinforcing the model's initial hallucination. The standard mitigation is a **Hybrid approach**: retrieve once with the real query (Keyword) and once with the HyDE query, then use **RRF** to combine them.

### Q: What is the "Asymmetric Retrieval" problem?

**Strong answer:**
Asymmetric retrieval refers to the fact that user queries are usually short (3-10 words) while document chunks are long (300-500 words). These inhabit different statistical distributions in the vector space, leading to "Distance Bias." High-performance systems solve this using **Asymmetric Encoders** (one model for queries, one for docs) or **Query Expansion** (HyDE) to "inflate" the query into a document-like distribution.

---

## References
- Gao et al. "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE, 2023/2024)
- Anthropic. "The Contextual Retrieval Playbook" (2024)
- LlamaIndex. "Query Transformation Cookbook" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **RAG (Retrieval-Augmented Generation)** | A system that fetches relevant documents first, then passes them to an LLM to generate an answer | Grounds LLM responses in real data instead of model memory |
| **Query Decomposition** | Breaking a complex user question into multiple simpler sub-questions | Ensures each part of a compound query gets its own retrieval pass |
| **Multi-Query** | Generating several paraphrased or sub-queries from one user query | Increases the chance of retrieving all relevant documents |
| **HyDE (Hypothetical Document Embeddings)** | Asking the LLM to write a fake answer, then embedding that fake answer instead of the raw query | Bridges the vocabulary gap between short queries and long documents |
| **Vector Neighborhood** | The region of embedding space where semantically similar items cluster together | Determines which documents a query will retrieve by proximity |
| **Asymmetric Retrieval** | The mismatch between short user queries and longer document chunks in embedding space | Explains why direct query embeddings can miss relevant documents |
| **Asymmetric Encoders** | Using separate embedding models — one tuned for queries, one for documents | Reduces the retrieval gap caused by length differences |
| **Query Expansion** | Adding synonyms, context, or hypothetical answers to the raw query before embedding | Improves recall by making the query resemble the documents |
| **Context Dilution** | When a chunk loses its meaning because the surrounding context is removed during chunking | The root cause of many RAG retrieval failures |
| **Contextual Retrieval** | Prepending an LLM-generated summary to each chunk before embedding it | Adds missing context so isolated chunks remain retrievable |
| **Iterative Document Enrichment** | Storing extra metadata (summaries, generated questions) alongside raw documents | Makes documents more discoverable from diverse query phrasings |
| **Compound Query** | A user question that spans multiple topics or requires multiple lookups | Signals that decomposition into sub-queries is needed |
| **In-Context Reranking** | Loading the top-100 retrieved documents into the LLM's context window and asking it to pick the best ones | Uses the LLM's reasoning ability instead of a separate reranker model |
| **Long Context Reasoning** | An LLM's ability to process and reason over very large inputs (1M+ tokens) | Enables in-context reranking and cross-document synthesis without extra models |
| **Cross-Encoder** | A model that reads the query and document together to produce a relevance score | Provides the most accurate reranking but is too slow for first-stage retrieval |
| **RRF (Reciprocal Rank Fusion)** | Combines multiple ranked lists by position rather than raw score | Merges results from keyword and vector search without needing compatible scores |
| **Hybrid Approach** | Combining keyword (BM25) and vector search results | Covers both exact-match and semantic-match retrieval needs |

*Next: [Agentic Systems](../07-agentic-systems/01-agent-fundamentals.md)*
