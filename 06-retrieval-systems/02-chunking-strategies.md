# Chunking Strategies

Chunking is the process of splitting a document into discrete segments for retrieval. Production pipelines have moved beyond blind fixed-size splits to **structure-aware and semantic segments**, with newer techniques like late chunking and contextual prepending now in the mainstream toolkit.

## Table of Contents

- [The Retrieval-Context Tension](#tension)
- [Recursive Structure Splitting](#recursive)
- [Semantic Chunking](#semantic)
- [Hierarchical (Parent-Child) Chunking](#hierarchical)
- [Content-Specific Strategies (Code, PDF, Tables, Charts, Excel)](#content-specific)
- [Handling Tables, Images & Charts in Production — Step by Step](#handling-tables-images--charts-in-production--step-by-step)
- [The Modern Default (2026): Late Chunking & Contextual Retrieval](#modern-default)
- [Choosing a Strategy: Trigger -> What It Improves](#choosing)
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

**PDF/layout in more depth — two modern approaches:**

1. **Parse → structure → chunk.** A layout model or VLM parser (**Docling / GraniteDocling**, **Marker**, **LlamaParse**) reconstructs the document *tree* — headings, body, tables, formulas, captions, and their section path, even across multi-column pages — then you chunk *that* structure (section-aware). Practical params: ~512-1000 token targets, a hard cap (~1200) for table chunks, drop tiny fragments (< ~30 tokens). Keep tables/figures whole and pair them with the summarized-table trick above. ([LlamaIndex — best AI PDF parsers](https://www.llamaindex.ai/insights/best-ai-pdf-parsers))
2. **Skip parsing — embed the page image (ColPali / ColQwen2).** Instead of OCR → layout → caption → text-embed, a vision-language retriever embeds the *rendered page* directly as a **multi-vector** (ColBERT-style late interaction), matching the query against the page's visual layout itself. Charts, scans, and complex tables "just work," and it beats the classic extract-and-embed pipeline on visually rich docs — at the cost of a heavier multi-vector index. ([ColPali, arXiv](https://arxiv.org/abs/2407.01449); [Vespa — retrieval with VLMs](https://blog.vespa.ai/retrieval-with-vision-language-models-colpali/))

Rule of thumb: **parse-then-chunk** for mostly-text PDFs where you want clean text in the prompt; **ColPali-style visual retrieval** when figures/tables/scans dominate and the *layout is the information*.

### 4. Charts, Images & Figures
- **The danger:** the answer is often *in the picture* (a trend line, a bar height, a diagram) — text-only pipelines silently discard it.
- **Two strategies:**
  - **Caption-and-embed (image → text):** a vision model (GPT-4o, Qwen-VL) writes a description ("line chart: revenue rises $2M→$5M, 2020-2024"), then you chunk/embed *that text*. Cheap to store, works with a text-only index, and keeps the pipeline simple.
  - **Multimodal / visual embedding:** embed the image directly with CLIP/SigLIP or an enterprise multimodal model (**Cohere Embed 4**), or use **ColPali/ColQwen** to embed the rendered page — no captioning step to lose information.
- **Always keep the caption + surrounding paragraph** with the figure — they carry the interpretation the pixels alone don't.

### 5. Excel / Spreadsheets
- **The danger:** spreadsheets are often **huge** and pure grid — one giant blob is useless, one-row-per-chunk loses the headers.
- **Strategy:**
  - Chunk **per sheet**, and within a sheet by **logical block / row range**, always carrying the **header row + sheet name** into each chunk so every value keeps its meaning.
  - For very large or numeric sheets, treat it as **structured data, not text**: load into a dataframe/SQL table and use **text-to-SQL** to query it — embedding 100k near-identical rows is expensive and low-signal.
  - Add a **per-sheet summary chunk** (column names + what the sheet contains) for discoverability.

---

## Handling Tables, Images &amp; Charts in Production — Step by Step

The naive pipeline (extract text → fixed-size chunk → embed) throws away the highest-value content in enterprise documents. Here is the production pipeline that preserves it.

```
 document (PDF / DOCX / XLSX / scan)
        │
        ▼
 STEP 1  DETECT & PARTITION      layout model splits the page into typed elements:
         (layout-aware parse)    text · table · figure/chart · title · caption
        │
        ▼
 STEP 2  ROUTE BY ELEMENT TYPE   each type goes down its own branch
        │
   ┌─────────────────────┬─────────────────────────┬────────────────────┐
   ▼                     ▼                         ▼                    ▼
 TEXT                  TABLE                    FIGURE/CHART          (metadata)
 recursive/            keep atomic +            VLM caption OR         source, page,
 semantic chunk        structured (MD/JSON)     multimodal embed       section, type,
                       + NL summary                                    permissions
        │                     │                         │
        └─────────────────────┴────────────┬────────────┘
                                            ▼
 STEP 3  BUILD DUAL REPRESENTATION   embed a retrieval-friendly summary/text;
         (index-small, return-rich)  store the FULL original (table markdown /
                                     image / rows) linked by ID for the LLM
        │
        ▼
 STEP 4  EMBED & INDEX          text embeddings for summaries; optional
                               multimodal/multi-vector index for images/pages
        │
        ▼
 STEP 5  RETRIEVE → RESOLVE     retrieve on the summary, then fetch and pass the
                               FULL original element to the (multimodal) LLM
```

**Step 1 — Detect & partition (layout-aware parsing).** Run a layout parser (**Unstructured**, **Docling**, **Marker**, **Azure AI Document Intelligence**, **LlamaParse**) that classifies each region as *text / table / figure / title / caption* and recovers reading order across multi-column pages. For scanned pages, **OCR happens here** — and OCR quality gates matter because OCR noise cascades downstream. ([Unstructured — ingestion](https://unstructured.io/insights/rag-pipeline-challenges-from-data-ingestion-to-retrieval))

**Step 2 — Route each element type to its own handler.** Don't treat everything as text:
- **Text** → recursive/semantic chunking (the standard path).
- **Table** → keep the table **atomic** and in a **structured format** (Markdown/HTML/JSON) so the LLM can reason over rows/columns; for a table too big to fit, split by **row groups and repeat the header row** in each piece.
- **Figure/chart** → either **caption it with a VLM** (image→text) or **embed it directly** (CLIP/SigLIP/Cohere Embed 4/ColPali).

**Step 3 — Build a dual representation (the key trick).** For tables and figures, **embed a concise natural-language summary for *retrieval*, but store the full original** (table markdown, image bytes, raw rows) linked by an ID and **return that to the LLM for *reasoning***. This is the same *index-small / return-rich* pattern as parent-child chunking — a raw table or image embeds terribly against a text query, but a one-sentence summary matches well. ([Databricks — multimodal RAG](https://www.databricks.com/blog/unite-your-patients-data-multi-modal-rag))

**Step 4 — Embed & index.** Text/summaries go to a normal text embedding index; images/pages can go to a **multimodal or multi-vector (ColPali-style)** index. Two common architectures: (a) **one unified multimodal embedding space** (text + images together, e.g. Cohere Embed 4), or (b) **separate indexes** (text index + image index) unified by a query router. ([multimodal RAG 2026](https://bigdataboutique.com/blog/multimodal-rag-retrieval-over-images-pdfs-and-text))

**Step 5 — Retrieve → resolve → generate.** Retrieve on the summaries, then **de-reference the ID to fetch the full element** and pass the original table/image to a **multimodal LLM** at generation time (so it sees the actual chart, not just your caption of it).

**Production tips:**
- **Tiered image encoding for cost** — full resolution for complex diagrams/charts, thumbnails for decorative images, **skip** tiny/low-information images entirely.
- **Vision-first shortcut** — Cohere Embed 4 / ColQwen can ingest **raw PDF pages without a parsing pipeline** at all; simplest to operate, at the cost of a heavier multi-vector index. ([Cohere Embed 4 / multimodal 2026](https://helain-zimmermann.com/blog/multimodal-rag-2026-vision-and-text-for-state-of-the-art-pipelines))
- **Always attach metadata** (source, page, section, element type, permissions) at partition time — it powers filtering, citations, and access control later.
- **Treat ingested images as untrusted** — visual-document RAG is poisonable with a single crafted image; scan/validate before indexing.

---

## The Modern Default (2026): Late Chunking &amp; Contextual Retrieval

**Is there one strategy that wins everywhere? Not quite — but there is a clear default and two general-purpose upgrades.** 2026 benchmarks still put **recursive splitting at ~512 tokens** as the best *starting* point: highest end-to-end accuracy across mixed document types, and effectively free (no model calls). Everything else is an upgrade you add *only when a metric justifies it*. ([denser.ai — 2026 chunking benchmarks](https://denser.ai/blog/rag-chunking-strategies/), [DigitalApplied — 2026 playbook](https://www.digitalapplied.com/blog/rag-chunking-strategies-2026-retrieval-quality-playbook))

Two upgrades help across almost any corpus:

**Late Chunking — the cheap, broad win.** Flip the order: embed the *whole document* first with a long-context embedding model, *then* split the resulting token embeddings into chunks (mean-pool each span). Because every chunk's vector was computed with the full document in view, it carries long-range context — a chunk that only says "It supports 40W" inherits the surrounding "Model X charger" signal. No per-chunk LLM call, so it's cheap. Best when chunks are ambiguous alone (pronouns, headers, cross-references). ([FutureAGI — advanced chunking](https://futureagi.com/blog/advanced-chunking-techniques-for-rag/))

**Contextual Retrieval — the max-accuracy win.** Prepend a short, LLM-generated context blurb to each chunk *before* embedding it ("This is from the 2025 Model X drone manual, battery section: ..."). Anthropic reported this cuts retrieval failures by **49%**, and **67% combined with a reranker** — a bigger gain than upgrading from a cheap to an expensive embedding model. The cost is one LLM call per chunk at index time, heavily amortized by prompt caching. ([Anthropic — Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval))

| | Late Chunking | Contextual Retrieval |
|---|---|---|
| How | embed whole doc, then split the embeddings | LLM writes a context blurb, prepend it, then embed |
| Cost | cheap — one long-context embedding pass | LLM call per chunk (cache it) |
| Best at | cross-references, pronouns, headers | making chunks self-contained; max accuracy |
| Stacks with | hierarchical + reranking | hierarchical + reranking (the 67% figure) |

**Recommended default stack:** recursive **512-token** splitting → **Late Chunking** (or **Contextual Retrieval** when accuracy is paramount and you can pay the index-time LLM cost) → **hierarchical parent-child** for context → a **reranker** at query time. Start simple, measure recall@k, and add each layer only when the numbers justify it — there is no universal winner, just a strong default plus upgrades matched to your bottleneck.

---

## Choosing a Strategy: Trigger -> What It Improves

Don't pick a strategy by fashion — pick it by which **metric is leaking**. Chunking mostly moves two of the three retrieval-gap metrics (see [Retrieval Quality Gap](01-rag-fundamentals.md)): **recall@k** (did a well-formed chunk get retrieved at all?) and **faithfulness / answer quality** (is the returned chunk complete enough to reason over?). Ranking **precision** is mainly a *reranker's* job — though sharper chunks help it.

| Strategy | Reach for it when... | Primarily improves | Cost / trade-off |
|----------|----------------------|--------------------|------------------|
| **Fixed-size + overlap** | quick prototype; uniform text | baseline; overlap nudges **recall@k** (fewer boundary cuts) | wasted storage, near-duplicate vectors |
| **Recursive (structure-aware)** | your default for mixed docs | **precision + chunk coherence** vs fixed (complete units) | almost none — no models, fast |
| **Semantic** | chunks blur several topics; precision is the bottleneck | **retrieval precision / relevance** (nDCG, context precision) — one-idea vectors | slow/expensive indexing; brittle threshold |
| **Hierarchical (parent-child)** | LLM answers wrong for *lack of surrounding context* | **faithfulness / context sufficiency** — without losing match precision | extra storage; must de-dup parents |
| **Content-specific** (code AST / tables / PDF-layout) | corpus is code, tables, or layout-heavy PDFs | **recall + faithfulness on that content type** | per-type parsing (AST, VLM) |
| **Late chunking** | chunks are ambiguous alone (pronouns, headers, cross-refs) | **recall@k** on context-dependent chunks | one long-context embedding pass (cheap) |
| **Contextual retrieval** | max accuracy; chunks aren't self-contained | **recall@k** — the single biggest win (~49% fewer failures, ~67% with a reranker) | LLM call per chunk at index time (cache it) |

**How to use this — diagnose, then pick:**
- **recall@k low** (the right chunk was never retrieved) -> add overlap, then **late chunking**, then **contextual retrieval** (and hybrid search at the retrieval layer).
- **precision / nDCG low** (junk ranked above the answer) -> **semantic chunking** for sharper vectors, and add a **reranker**.
- **retrieval is fine but the answer is wrong/incomplete** (low faithfulness) -> **hierarchical** (return the parent) or larger / content-specific chunks.

The trap to avoid: reaching for a fancier chunker when your leaking metric is *precision* (fix it with a reranker) or *recall of exact terms* (fix it with hybrid search) — chunking can't repair a stage it doesn't touch.

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
- Anthropic. "Contextual Retrieval: Improving RAG Accuracy" (2024) — https://www.anthropic.com/news/contextual-retrieval
- Günther et al. (Jina AI). "Late Chunking: Contextual Chunk Embeddings Using Long-Context Models" (2024)
- Faysse et al. "ColPali: Efficient Document Retrieval with Vision Language Models" (arXiv:2407.01449, 2024)
- LlamaIndex. "Best AI PDF Parsers" (2026) — https://www.llamaindex.ai/insights/best-ai-pdf-parsers
- Denser.ai / DigitalApplied. "RAG Chunking Strategies 2026" (benchmarks)
- LlamaIndex. "Advanced Chunking Strategies for RAG" (2025)
- LangChain. "RecursiveCharacterTextSplitter Benchmarks" (2024)

---

*Next: [Embedding Models](03-embedding-models.md)*
