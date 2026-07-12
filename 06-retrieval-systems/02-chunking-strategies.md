# Chunking Strategies

Chunking is the process of splitting a document into discrete segments for retrieval. Production pipelines have moved beyond blind fixed-size splits to **structure-aware and semantic segments**, with newer techniques like late chunking and contextual prepending now in the mainstream toolkit.

## Table of Contents

- [The Retrieval-Context Tension](#tension)
- [Recursive Structure Splitting](#recursive)
- [Semantic Chunking](#semantic)
- [Hierarchical (Parent-Child) Chunking](#hierarchical)
- [Content-Specific Strategies (Code, PDF, Tables)](#content-specific)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Retrieval-Context Tension

| Aspect | Small Chunks (100t) | Large Chunks (1000t) |
|--------|---------------------|----------------------|
| **Precision** | High (Exact match) | Low (Diluted) |
| **Context** | Poor (Broken sentences) | Rich (Surrounding info) |
| **Storage** | High (More vectors) | Low (Fewer vectors) |
| **Latency** | Low (Fast search) | High (Heavy retrieval) |

**Rule**: Smaller is better for *finding*, but larger is better for *thinking*. Use **Hierarchical Chunking** to get both.

**Why the tension exists.** A chunk becomes *one vector*. Pack a whole page into it and the embedding is an averaged blur of five topics — it matches everything weakly and nothing strongly (low precision). Shrink it to one sentence and the vector is sharp, but the sentence alone may be meaningless to the LLM ("It supports 40W." — *what* does?). So small wins **retrieval**, large wins **reasoning**.

**Chunk overlap** is the usual first mitigation: let consecutive chunks share ~10-20% of their text so a fact sitting on a boundary isn't cut in half. It's cheap insurance but wasteful (duplicated tokens, near-duplicate vectors) and doesn't fix the core precision-vs-context trade — which is why hierarchical chunking (below) is the real answer.

**Rules of thumb (starting points — then tune on your eval set):**

| Content | Chunk size | Overlap |
|---------|-----------|---------|
| Dense prose / docs | 256-512 tokens | 10-15% |
| Chat / short FAQ | 128-256 tokens | small |
| Code | by function/class (AST), not size | none |
| Legal / financial | larger (512-1024) + hierarchical | 15-20% |

---

## Recursive Structure Splitting

Instead of splitting at every 500 characters, we split at logical boundaries:
`[Double Newline] > [Single Newline] > [Period] > [Space]`.

**Best practice**: Use **Markdown-Aware Splitting**. If a document has `#` headers, ensure the header is prepended to *every* child chunk to preserve context (Contextual Chunking).

**How the recursion actually works.** The splitter tries the *biggest* separator first and only drops to a smaller one when a piece is still over the size limit — so it always cuts at the most natural boundary available:

```
try to split on  \n\n  (paragraphs)
  piece still too big?  -> split that piece on \n  (lines)
    still too big?      -> split on ". "  (sentences)
      still too big?    -> split on " "   (words)   <- last resort
```

The result: paragraphs stay whole whenever they fit, and you only ever break mid-sentence as a fallback. This is why `RecursiveCharacterTextSplitter` is the sane default — it's structure-aware *without* needing embeddings, so it's fast and free. Treat it as the baseline; reach for semantic chunking only when logical boundaries aren't enough.

---

## Semantic Chunking

Semantic chunking uses an embedding model to detect "topic shifts."

1. Split text into individual sentences.
2. Group sentences as long as their embedding similarity stays above a threshold (e.g., 0.82).
3. If similarity drops, start a new chunk.

**Nuance**: Production pipelines increasingly use **Cross-Encoder Segmenters**. A tiny model scans the text and predicts a "Separator token" at every semantic break. This is 10x more accurate than cosine-similarity thresholding.

**Why bother — and why it's fiddly.** The goal is that each chunk is *one complete idea*, so its vector isn't a blur (back to the tension above). But it carries real cost, which is why it isn't the default:
- **Slow / expensive to index** — you embed *every sentence* just to decide where to cut.
- **The threshold is brittle** — set the similarity cutoff too high and you get lots of tiny chunks; too low and unrelated sentences merge. The right value drifts by domain — which is exactly why a cross-encoder segmenter that *learns* the break points beats a hand-tuned cosine threshold.

**When it's worth it:** heterogeneous documents that jump between topics (transcripts, wikis, mixed reports). For clean, well-structured docs, recursive splitting is usually good enough at a fraction of the cost.

---

## Hierarchical (Parent-Child) Chunking

This is the industry standard for production RAG.

- **Process**: 
  1. Create "Parent" chunks of 1,500 tokens.
  2. Sub-divide each parent into 5 "Child" chunks of 300 tokens.
  3. **Index only the children**.
  4. At retrieval, if a child matches, **return the full parent context** to the LLM.
- **Why?**: The child is small and easy for the vector DB to match. The parent provides enough context for the LLM to actually reason correctly without "Broken Context" hallucinations.

**This is the direct answer to the retrieval-context tension:** search with the *small* thing (precision), feed the LLM the *big* thing (context) — you stop trading one for the other.

```
  INDEX (children, small)              RETURN (parent, big)
  parent P1 --+-- child c1  (query matches c1)
              +-- child c2   ---->   give the LLM all of P1
              +-- child c3           (c1..c5 + surrounding text)
```

**Two gotchas worth naming:**
- **De-duplicate parents** — if three children of the same parent all match, return the parent *once*, not three times, or you waste context and skew ranking.
- **Related patterns:** *sentence-window retrieval* (match one sentence, return it plus N neighbors) and *auto-merging* (if enough sibling children match, merge up to the parent automatically). Same idea — decouple the match unit from the context unit.

See [Contextual Retrieval](10-contextual-retrieval.md) for the complementary trick: prepending document-level context to each chunk *before* embedding it.

---

## Content-Specific Strategies

### 1. Code Chunking
- **Strategy**: Use AST (Abstract Syntax Tree) parsing.
- **Rule**: Never split a function mid-body. Keep imports and class declarations with their methods.

### 2. Table Chunking
- **Strategy**: Use Markdown formatting for tables.
- **Modern pattern**: "Summarized Tables." Store a natural language summary of the table in the vector DB, but return the full Markdown table to the LLM.

### 3. PDF/Layout Chunking
- **Strategy**: Use **Vision-Language Model (VLM)** pre-processing (e.g., ColPali).
- **Nuance**: Instead of just text, store embeddings that represent the *positional layout* of the page, ensuring charts and sidebars don't get mixed into body text.

**The unifying principle:** chunk on the content's *own* natural unit, not on a token count.
- **Code** — a function is the atomic unit of meaning; split it mid-body and you get a fragment that neither retrieves well nor runs. AST parsing cuts on function/class boundaries and keeps imports with the code that needs them.
- **Tables** — a raw Markdown grid embeds *terribly*: the vector is dominated by pipes, header words, and digits, not meaning, so it rarely matches a natural-language query. "Summarized tables" fix this — embed a sentence describing the table for *retrieval*, but hand the LLM the full table for *reasoning* (the same index-small / return-rich split as hierarchical chunking).
- **PDF / layout** — reading order is 2-D, not linear; naive text extraction interleaves a sidebar into a paragraph. Layout-aware/VLM parsing keeps each visual block (body, caption, chart, footnote) intact.

---

## Interview Questions

### Q: Why is fixed-size chunking with overlap problematic for production systems?

**Strong answer:**
Fixed-size chunking is "content-blind." It frequently splits sentences mid-thought, breaks mathematical equations, and separates headers from their descriptive text. While "Overlap" (e.g., 10%) mitigates this by duplicating 10% of text across chunks, it doesn't solve the core issue: the model's attention is forced to reconstruct meaning from fragmented strings. Modern pipelines prefer **Semantic or Logical Chunking** because it ensures each vector represents a "Complete Semantic Unit," leading to significantly higher retrieval precision.

### Q: What is "Contextual Retrieval" (the Anthropic pattern)?

**Strong answer:**
Contextual Retrieval involves prepending a 1-sentence global context to every chunk before embedding it. For example, if a chunk is about "battery life," but it's from a manual for a "2025 Model X Drone," the text `[Drone_Model_X_Manual]:` is added to the chunk. This ensures that the vector for "battery life" is influenced by the "Drone" context, preventing it from being accidentally retrieved for "phone battery" queries.

---

## References
- Anthropic. "Contextual Retrieval: Improving RAG Accuracy" (2024)
- LlamaIndex. "Advanced Chunking Strategies for RAG" (2025)
- LangChain. "RecursiveCharacterTextSplitter Benchmarks" (2024)

---

*Next: [Embedding Models](03-embedding-models.md)*
