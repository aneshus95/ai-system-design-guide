# Embedding Models

Embedding models convert text into high-dimensional vectors. The frontier has moved past static single-vector representations to **multi-resolution, late-interaction, and multimodal** embeddings.

## Table of Contents

- [The Embedding Frontier (Matryoshka)]( #matryoshka)
- [Late Interaction (ColBERT v2)]( #late-interaction)
- [Binary and Int8 Quantization]( #quantization)
- [Model Selection Criteria]( #selection)
- [Multimodal Embeddings (Vision + Text)]( #multimodal)
- [Interview Questions]( #interview-questions)
- [References]( #references)

---

## The Embedding Frontier: Matryoshka Embeddings

Traditionally, if you embedded text into 1,536 dimensions, you were stuck using all 1,536 dimensions for search. 

**Matryoshka Representation Learning (MRL)**
- Models are trained to "store" the most important info in the first few dimensions.
- **The Win**: You can embed at 1,536 dims, but index only the first **64 dims** for a "fast search" pass, then refine the top results with the full 1,536 dims.
- **Efficiency**: 20x reduction in memory/index size with <2% drop in accuracy.

---

## Late Interaction: ColBERT v2

Standard embeddings are "Bi-Encoders" (one vector per chunk). **ColBERT** (Contextualized Late Interaction over BERT) uses a "token-level" approach.

- **How**: Instead of 1 vector per chunk, ColBERT stores 1 vector **per token**.
- **Interaction**: At query time, the model compares every token in your query to every token in the documents (the "MaxSim" operation).
- **Status**: ColBERT v2 (and successors like ColPali, ColQwen2.5, ColNomic for documents and pages-as-images) is drastically compressed via PLAID indexing, making it feasible for production. It achieves much higher precision for "needle in a haystack" technical queries.

---

## Binary and Int8 Quantization

Storing `float32` vectors is expensive. Production indexes lean heavily on **in-model quantization**.

- **Binary Embeddings**: Convert vectors to 1s and 0s. 
  - **Memory**: 32x reduction.
  - **Speed**: Hamming distance (XOR operations) is 10x faster than Cosine similarity on modern CPUs.
- **Int8/Int4**: Supported natively by models like `text-embedding-3-small`.

---

## Model Selection Criteria

| Model | Provider | Features | Context |
|-------|----------|----------|---------|
| **Gemini Embedding 001** | Google | Multimodal (text, image, video, audio, PDF), shared 3072-dim space, MTEB-English leader | 8k |
| **Qwen3-Embedding-8B** | Open Source | MTEB-Multilingual leader, instruction-tuned, long-doc strength | 32k |
| **Llama-Embed-Nemotron-8B** | NVIDIA | Top multilingual scores, open weights | 8k |
| **Cohere Embed v4** | Cohere | Multimodal (text + image), Matryoshka, binary quantization | 128k |
| **Voyage-Multimodal-3.5** | Voyage AI | Unified text/image, retrieval-tuned | 32k |
| **OpenAI text-embedding-3-large** | OpenAI | Matryoshka, Native Int8, broad support | 8k |
| **BGE-M3** | Open Source | Multilingual, multi-granularity (dense + sparse + late-interaction) | 8k |
| **Jina-Embeddings-v3** | Jina AI | Late-interaction support, long context | 128k |

Open-weight models (Qwen3, Llama-Embed-Nemotron, BGE) now match or beat the commercial APIs on pure MTEB scores. Pick commercial when you want managed infra and SLAs; pick open weights when cost-per-query at high volume matters more than latency floor.

---

## Multimodal Embeddings

Text-only RAG silently throws away the charts, tables, diagrams, and layout signal that often hold the answer. Modern stacks treat pages, screenshots, and figures as first-class retrieval objects:

- **Unified vision-text embeddings**: Cohere Embed v4, Voyage-Multimodal-3.5, Gemini Embedding 001 all share a single vector space, so you can query "where is the emergency shutoff valve?" against schematics.
- **Page-as-image with late interaction**: ColPali, ColQwen2.5, and ColNomic embed each page render directly, skipping fragile OCR and preserving visual hierarchy.
- **CLIP-family models**: Still useful for image-heavy catalogs (e-commerce, media) where text-image alignment is the core signal.

---

## Interview Questions

### Q: What is the "Vocabulary Mismatch" problem in embeddings?

**Strong answer:**
Embeddings rely on the semantic space learned during training. If a user query uses a newer term (e.g., a model name released after the embedding model's cutoff) that wasn't in the embedding model's training set, the model might assign it a generic "AI" vector, missing the specific nuances. The standard fix is **Hybrid Search** (using BM25 to catch the specific keyword) plus **Cross-Encoder Reranking**, which handles out-of-distribution vocabulary better by looking at query and document tokens simultaneously.

### Q: Why would you choose a Matryoshka model for a 1-billion-vector index?

**Strong answer:**
Scaling to 1 billion vectors with standard `float32` 1536-dim embeddings requires ~6TB of high-speed RAM for an HNSW index, which is prohibitively expensive. With a Matryoshka model, I can use the first 128 dimensions (Binary quantized) for the initial retrieval. This reduces the memory footprint by over 90%, allowing the "Top 1,000" candidates to be found on significantly cheaper hardware. I can then fetch the full-resolution vectors for just those 1,000 candidates to perform the final reranking.

---

## References
- Kusupati et al. "Matryoshka Representation Learning" (2022/2024 update)
- Khattab et al. "ColBERT v1 & v2: Efficient Late Interaction" (2021/2023)
- OpenAI. "Introducing New Embedding Models with Matryoshka Support" (2024)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Embedding Model** | A neural network that converts text (or other content) into a fixed-length numerical vector | Produces the dense representations that enable semantic similarity search |
| **Dense Vector** | A high-dimensional array of floating-point numbers where most values are non-zero | The core data structure stored in a vector database and compared at query time |
| **MRL (Matryoshka Representation Learning)** | A training approach where the most important information is packed into the earliest dimensions of a vector, like nested Russian dolls | Enables a two-pass search: fast low-dimension screening followed by high-dimension re-scoring |
| **Matryoshka Embeddings** | Vectors trained with MRL so any prefix subset of dimensions is still a valid, usable embedding | Reduce memory and index size by 20x with less than 2% accuracy loss |
| **Bi-Encoder** | An architecture that encodes the query and each document independently into separate vectors | Pre-computes document vectors offline so retrieval is a fast similarity lookup at query time |
| **ColBERT v2** | A late-interaction retrieval model that stores one vector per token rather than one per document | Achieves higher precision on technical "needle in a haystack" queries by comparing token-level signals |
| **MaxSim Operation** | ColBERT's scoring function: for each query token, find its most similar document token, then sum those maximum similarities | Captures fine-grained token-level relevance that a single-vector cosine similarity misses |
| **PLAID Indexing** | A compression scheme for ColBERT that dramatically reduces the storage cost of per-token vectors | Makes ColBERT feasible for production-scale indexes |
| **Late Interaction** | Delaying the comparison between query and document representations until query time, using token-level vectors | Balances the accuracy of cross-encoders with the scalability of bi-encoders |
| **Float32** | 32-bit floating-point number; the default precision for embedding vectors | Standard storage format; expensive at large scale, motivating quantization |
| **Binary Embeddings** | Vectors reduced to a sequence of 1s and 0s | Achieve a 32x memory reduction; distance computed via fast XOR (Hamming distance) operations |
| **Int8 / Int4 Quantization** | Representing each vector dimension as an 8-bit or 4-bit integer instead of 32-bit float | Reduces index memory by 4-8x with modest accuracy loss |
| **Hamming Distance** | The number of bit positions where two binary vectors differ | Enables very fast binary vector comparison using CPU XOR instructions |
| **MTEB (Massive Text Embedding Benchmark)** | A standardized benchmark suite covering retrieval, classification, clustering, and other embedding tasks | The standard leaderboard used to compare embedding models across languages and domains |
| **Multilingual Embedding Model** | An embedding model trained on text from many languages so queries and documents in different languages can be compared | Enables cross-lingual retrieval without translation |
| **Multimodal Embeddings** | Vectors that represent both text and images in the same shared space | Allow a single index to support queries that mix text and visual content |
| **Vocabulary Mismatch** | When a query uses different words than the document for the same concept, causing semantic embeddings to miss the match | The core weakness that makes hybrid search (BM25 + dense) necessary |
| **Out-of-Distribution Vocabulary** | Terms not seen during the embedding model's training, causing poor vector representations | Motivates hybrid search; BM25 handles novel tokens that embeddings mis-encode |
| **Cross-Encoder Reranking** | A second-stage model that reads query and document together to produce a precise relevance score | Corrects ranking errors from bi-encoder cosine similarity, especially for out-of-distribution terms |
| **BGE-M3** | An open-source multilingual embedding model supporting dense, sparse, and late-interaction retrieval in a single model | Versatile choice for multilingual or multi-granularity retrieval without separate models |
| **Instruction-Tuned Embedding** | An embedding model fine-tuned to follow task-specific instructions prepended to the query | Improves retrieval accuracy when retrieval intent varies across query types |

*Next: [Vector Databases](04-vector-databases.md)*
