# Embeddings and Vector Spaces

Embeddings are dense vector representations of text that capture semantic meaning. They are foundational to RAG systems, semantic search, and many AI applications.

## Table of Contents

- [What Are Embeddings](#what-are-embeddings)
- [Embedding Model Architectures](#embedding-model-architectures)
- [Training Objectives](#training-objectives)
- [Distance Metrics](#distance-metrics)
- [Embedding Model Comparison](#embedding-model-comparison)
- [Matryoshka and Adaptive Dimensions](#matryoshka-and-adaptive-dimensions)
- [Late Interaction vs. Late Chunking](#late-chunking-and-interaction)
- [Binary and Scalar Quantization](#quantization-for-scale)
- [Practical Considerations (Batching, Caching)](#practical-considerations)
- [Embedding Drift and Versioning](#embedding-drift-and-versioning)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What Are Embeddings

Embeddings map discrete text (words, sentences, documents) to continuous vector spaces where semantic similarity corresponds to geometric proximity.

**Key properties:**
- Similar meanings are close together
- Relationships can be encoded as vector operations (king - man + woman = queen)
- Enable efficient similarity search through approximate nearest neighbor algorithms

**Mental model:**
Think of embeddings as coordinates in a very high-dimensional space. Dimensionality (512 to 4096) provides expressiveness. Each dimension captures some aspect of meaning, though individual dimensions are not interpretable.

---

## Embedding Model Architectures

### Word Embeddings (Historical)

Early approaches embedded individual words:

| Model | Year | Approach | Limitation |
|-------|------|----------|------------|
| Word2Vec | 2013 | Skip-gram, CBOW | Static: "bank" same in all contexts |
| GloVe | 2014 | Co-occurrence matrix | Static |
| FastText | 2017 | Subword embeddings | Static, but handles OOV |

**Key limitation:** Same word gets same embedding regardless of context.

### Contextual Embeddings

Transformer-based models produce context-dependent embeddings:

```python
# Static embedding (Word2Vec)
embed("bank") = [0.1, 0.3, ...]  # Same vector always

# Contextual embedding (BERT)
embed("river bank") = [0.1, 0.3, ...]   # Geography sense
embed("bank account") = [0.5, 0.2, ...]  # Finance sense
```

### Sentence/Document Embeddings

For retrieval, we need to embed entire texts:

| Approach | Method | Pros | Cons |
|----------|--------|------|------|
| Mean pooling | Average token embeddings | Simple | Loses information |
| CLS token | Use [CLS] token embedding | Standard for BERT | May not capture full text |
| Last token | Use final token | Works for decoder models | Position bias |
| Trained pooling | Learn pooling weights | Better quality | Requires training |

Modern embedding models are trained specifically for sentence/document embedding, not just adapted from language models.

### Bi-Encoder Architecture

Standard retrieval embedding architecture:

```
Document -> Encoder -> Document Embedding
Query    -> Encoder -> Query Embedding

Similarity = cosine(doc_embedding, query_embedding)
```

**Properties:**
- Documents can be pre-computed and indexed
- Query embedding computed at query time
- O(1) similarity computation per document (with ANN)

### Cross-Encoder Architecture

Alternative that processes query and document together:

```
[Query, Document] -> Encoder -> Relevance Score
```

**Properties:**
- More accurate (sees both together)
- Cannot pre-compute: O(n) inference for n documents
- Used for reranking, not retrieval

---

## Training Objectives

### Contrastive Learning

Most modern embedding models use contrastive learning:

```python
# Simplified contrastive loss
def contrastive_loss(anchor, positive, negatives):
    pos_sim = cosine_similarity(anchor, positive)
    neg_sims = [cosine_similarity(anchor, neg) for neg in negatives]
    
    # Push positive close, negatives far
    loss = -log(exp(pos_sim / tau) / 
                (exp(pos_sim / tau) + sum(exp(neg_sim / tau) for neg_sim in neg_sims)))
    return loss
```

**Key factors:**
- **Positive pairs:** Semantically similar texts (parallel sentences, query-document pairs)
- **Hard negatives:** Similar but not matching texts (BM25 retrieved non-relevant)
- **In-batch negatives:** Other batch items as negatives (efficient)

### Training Data Sources

| Source | Positive Pairs | Quality | Scale |
|--------|---------------|---------|-------|
| Parallel sentences | Translation pairs | High | Medium |
| Query-document | Search logs | High | Medium |
| Title-body | Document structure | Medium | Large |
| Paraphrase | NLI datasets | High | Small |
| Generated | LLM creates pairs | Variable | Large |

### Instruction-Tuned Embeddings

Recent models accept task instructions:

```python
# Instruction-tuned (e.g., E5, BGE)
query_embedding = embed("Represent this query for retrieval: What is RAG?")
doc_embedding = embed("Represent this document for retrieval: RAG combines...")
```

This improves performance by specifying the intended use.

---

## Distance Metrics

### Cosine Similarity

Most common for text embeddings:

```python
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

**Properties:**
- Range: [-1, 1] (for normalized vectors, [0, 1] if positive)
- Measures angle, not magnitude
- Invariant to vector length

**When to use:** Default choice for text embeddings.

### Dot Product

```python
def dot_product(a, b):
    return np.dot(a, b)
```

**Properties:**
- Magnitude matters
- Unbounded range
- Equivalent to cosine for normalized vectors

**When to use:** When embeddings are already normalized, or magnitude is meaningful.

### Euclidean Distance

```python
def euclidean_distance(a, b):
    return np.linalg.norm(a - b)
```

**Properties:**
- Measures absolute difference
- Affected by magnitude
- For normalized vectors: sqrt(2 - 2 * cosine)

**When to use:** Rarely for text; more common for image embeddings.

### Metric Selection

| Metric | Vector Databases | Common Use |
|--------|------------------|------------|
| Cosine | Pinecone, Qdrant, Weaviate | Text embeddings |
| Dot Product | All major DBs | Normalized embeddings |
| Euclidean | All major DBs | Image, multimodal |

---

## Embedding Model Comparison

### Current Top Models (December 2025)

| Model | Dimensions | Max Tokens | MTEB Retrieval | Cost / 1M tokens |
|-------|------------|------------|----------------|------------------|
| OpenAI text-embedding-4 | 3072 | 16k | 68.2 | $0.10 |
| Voyage-4 | 1024 | 128k | 70.1 | $0.05 |
| Cohere embed-v3.5 | 1024 | 512 | 67.5 | $0.10 |
| Google text-embedding-005 | 768 | 8k | 67.2 | $0.02 |

*MTEB scores are approximate and vary by benchmark subset. Always verify current values. The English leaderboard is currently led by Gemini Embedding 001 (68.32); the multilingual leaderboard by Qwen3-Embedding-8B (70.58) and Llama-Embed-Nemotron-8B.*

### Open Source Models

| Model | Dimensions | Max Tokens | MTEB Retrieval | Notes |
|-------|------------|------------|----------------|-------|
| BGE-large-en-v1.5 | 1024 | 512 | 63.9 | Strong open model |
| E5-large-v2 | 1024 | 512 | 62.4 | Instruction-tuned |
| GTE-large | 1024 | 512 | 63.1 | Alibaba |
| Nomic-embed-text-v1.5 | 768 | 8192 | 62.3 | Long context, open |

### Selection Criteria

| Factor | Considerations |
|--------|----------------|
| Quality (MTEB) | Higher is better, but task-specific evaluation matters more |
| Dimensions | Higher = more expressive but more storage/compute |
| Max tokens | Must accommodate your document sizes |
| Cost | API vs self-hosting tradeoffs |
| Latency | Embedding generation time |
| Multilingual | If serving non-English content |

---

## Matryoshka and Adaptive Dimensions

### The Idea

Matryoshka Representation Learning (MRL) trains embeddings such that prefixes of the full embedding are also meaningful:

```python
full_embedding = model.encode(text)  # 1024 dimensions

# All these are valid embeddings with decreasing quality
dim_512 = full_embedding[:512]  
dim_256 = full_embedding[:256]
dim_128 = full_embedding[:128]
dim_64 = full_embedding[:64]
```

### Why It Matters

| Use Case | Dimension | Tradeoff |
|----------|-----------|----------|
| Full Retrieval | 1024-3072 | Peak Accuracy |
| **Two-Stage Retrieval**| 128 -> 1024 | **Production standard**: retrieve 1000 with 128-d, refine top 100 with 1024-d. |
| Cost-sensitive | 256 | 12x storage savings, <2% MRR loss |
| Edge / Mobile | 64 | Maximum speed, handles simple intent |

### Models with Matryoshka Support

- OpenAI text-embedding-3-* (native)
- Nomic-embed-text-v1.5
- Several fine-tuned models

### Using Matryoshka Embeddings

```python
from openai import OpenAI
client = OpenAI()

# Request smaller dimensions
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="Your text here",
    dimensions=256  # Request 256 instead of full 3072
)
```

---

### Late Chunking (The 2025 Shift)

**Traditional Chunking:**
`Document -> Split into chunks -> Embed chunks individually`
- **Issue**: Chunk 2 loses the context from Chunk 1.

**Late Chunking (introduced by Jina AI/Voyage):**
`Full Document -> Model Encoder -> Token-level Embeddings -> Pool into chunk boundaries`
- **Benefit**: Each chunk's embedding contains information from the **entire document** because the transformer's self-attention was applied to the full sequence before pooling.
- **Requirement**: A model with long-context support (at least 8k+ tokens).

---

## Quantization for Scale

To handle billions of vectors, **Binary** and **Scalar (Int8)** quantization are now standard.

| Type | Data Size | Memory Savings | Quality Loss | Supported By |
|------|-----------|----------------|--------------|--------------|
| Float32 | 4 bytes/dim | Baseline | 0% | All |
| Int8 | 1 byte/dim | 4x | <1% | Cohere, BGE |
| **Binary** | **1 bit/dim** | **32x** | ~5-10% | Cohere v3, v4 |

**Binary Quantization Pattern:**
1. Retrieve top 1000 using Binary embeddings (extreme speed).
2. Rerank top 50 using Float32 or a Cross-Encoder (peak accuracy).

### When to Use ColBERT

- Retrieval precision is critical
- Can afford storage overhead
- Query latency budget > 50ms

### Implementation

```python
# Using RAGatouille
from ragatouille import RAGPretrainedModel

model = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents
model.index(
    collection=documents,
    index_name="my_index"
)

# Search
results = model.search(query="What is RAG?", k=10)
```

---

## Practical Considerations

### Batch Processing

```python
# Inefficient: one API call per document
embeddings = [embed(doc) for doc in documents]

# Efficient: batch API calls
batch_size = 100
embeddings = []
for i in range(0, len(documents), batch_size):
    batch = documents[i:i + batch_size]
    batch_embeddings = embed_batch(batch)
    embeddings.extend(batch_embeddings)
```

### Chunking for Embeddings

Long documents must be chunked before embedding:

```python
def embed_document(document: str, max_tokens: int = 512) -> list[np.array]:
    chunks = chunk_document(document, max_tokens=max_tokens)
    embeddings = []
    for chunk in chunks:
        embedding = embed(chunk)
        embeddings.append(embedding)
    return embeddings
```

**Considerations:**
- Chunk size should be less than model max tokens
- Overlap helps preserve context across chunk boundaries
- Store chunk-to-document mapping for retrieval

### Normalization

Many systems expect normalized embeddings:

```python
def normalize(embedding):
    norm = np.linalg.norm(embedding)
    return embedding / norm

# Cosine similarity of normalized vectors = dot product
similarity = np.dot(normalize(a), normalize(b))
```

Most vector databases and embedding APIs handle normalization, but verify.

### Caching

Embedding computation is expensive. Cache aggressively:

```python
import hashlib

def get_embedding(text: str, cache: dict) -> np.array:
    key = hashlib.sha256(text.encode()).hexdigest()
    
    if key in cache:
        return cache[key]
    
    embedding = compute_embedding(text)
    cache[key] = embedding
    return embedding
```

---

## Embedding Drift and Versioning

### The Problem

Embeddings are not comparable across:
- Different models
- Different versions of the same model
- Sometimes different API calls (some APIs have non-determinism)

### Consequences

If you update your embedding model:
- All existing embeddings become incompatible
- Must re-embed entire corpus
- Search results will be inconsistent during migration

### Mitigation Strategies

**1. Version your embeddings:**
```python
embedding_metadata = {
    "model": "text-embedding-3-large",
    "model_version": "2024-01",
    "dimensions": 3072,
    "created_at": "2025-12-16"
}
```

**2. Plan for re-embedding:**
- Estimate cost and time for full re-embed
- Build pipelines that can run in background
- Test new embeddings before switching

**3. Blue-green deployment:**
```
Index A: Current embeddings
Index B: New embeddings (building)

Query -> Both indexes -> Merge or switch
```

**4. Track embedding quality:**
- Monitor retrieval metrics continuously
- Detect drift in embedding distributions
- Alert on quality degradation

---

## Interview Questions

### Q: How do embedding models learn semantic similarity?

**Strong answer:**
Embedding models are trained with contrastive learning. The objective is to make embeddings of semantically similar texts close together and dissimilar texts far apart.

Training process:
1. Positive pairs: Texts that should be similar (query-document pairs, paraphrases, translations)
2. Negative pairs: Texts that should be dissimilar (often from same batch or hard negatives from BM25)
3. Loss function: Pushes positive pairs close, negative pairs far

The model learns to place texts in a high-dimensional space where distance correlates with semantic similarity. This enables retrieval: embed the query, find nearest neighbors in the document embedding space.

Modern models like E5 and BGE are also instruction-tuned, where you prefix with task instructions to specialize the embedding.

### Q: When would you use ColBERT over a bi-encoder?

**Strong answer:**
ColBERT uses late interaction: instead of one embedding per document, it keeps per-token embeddings. At query time, it computes token-level similarity.

Choose ColBERT when:
- Retrieval precision is critical (legal, medical, high-stakes)
- You can afford 10-100x storage overhead per document
- Query latency budget is 50ms+ (slightly slower than bi-encoder)
- Your queries benefit from lexical matching (technical terms)

Choose bi-encoder when:
- Storage is constrained
- Need sub-20ms latency
- Retrieval precision from bi-encoder is sufficient
- Frequent re-indexing (ColBERT reindex is expensive)

In practice, a common pattern is: bi-encoder for first-stage retrieval (top 100), then cross-encoder or ColBERT for reranking.

### Q: How do you handle embedding drift when updating models?

**Strong answer:**
Embedding models produce vectors that are only meaningful relative to the same model. If you update the model, all old embeddings become incompatible.

My approach:
1. **Never update in place.** Create a parallel index with new embeddings.
2. **Test before switching.** Compare retrieval quality on a test set with both old and new embeddings.
3. **Background rebuild.** Re-embed the entire corpus with the new model in the background.
4. **Atomic switch.** Once the new index is complete and validated, switch traffic atomically.
5. **Rollback plan.** Keep the old index available for quick rollback.

For cost estimation: if you have 10M documents at 500 tokens average, and text-embedding-3-large costs $0.13/1M tokens, re-embedding costs about $650. Plan for this cost when considering model updates.

### Q: How do you choose dimensions for embeddings?

**Strong answer:**
Higher dimensions capture more information but cost more storage and computation.

Considerations:
- **Storage:** 1024-d float32 = 4 KB per embedding. At 10M docs = 40 GB just for embeddings.
- **Search speed:** Higher dimensions = slower nearest neighbor search.
- **Quality:** Diminishing returns above certain dimensions for most tasks.

Practical approach:
1. Start with the model's recommended dimensions.
2. If using Matryoshka models (like text-embedding-3), experiment with lower dimensions on your task.
3. Benchmark quality at different dimensions: often 256-512 is 95% of full quality.
4. For two-stage retrieval: use low dimensions for first stage, full dimensions for reranking.

For most applications, 768-1024 dimensions provide good balance. The exception is very high-precision requirements where 2048-4096 may help.

---

## References

- Reimers and Gurevych. "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks" (2019)
- Khattab and Zaharia. "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" (2020)
- Wang et al. "Text Embeddings by Weakly-Supervised Contrastive Pre-training" (E5, 2022)
- Xiao et al. "C-Pack: Packaged Resources To Advance General Chinese Embedding" (BGE, 2023)
- Kusupati et al. "Matryoshka Representation Learning" (MRL, 2022)
- MTEB Leaderboard: https://huggingface.co/spaces/mteb/leaderboard
- OpenAI Embeddings Guide: https://platform.openai.com/docs/guides/embeddings

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Embedding** | A dense floating-point vector that represents text (word, sentence, or document) in a high-dimensional space | Encodes semantic meaning so that similar texts end up close together; enables similarity search |
| **Vector Space** | A high-dimensional coordinate system where text is placed such that semantic relationships become geometric ones | Foundation of embedding-based retrieval; similarity = closeness in this space |
| **Dimensionality** | The number of coordinates (float values) in an embedding vector, typically 256–4096 | Higher dimensions can capture more information but cost more storage and search time |
| **Word2Vec** | A 2013 model that learns static word embeddings using skip-gram or CBOW objectives | Demonstrated that word meaning can be encoded as vectors with arithmetic properties (king − man + woman ≈ queen) |
| **GloVe** | A word embedding model trained on global word co-occurrence statistics from a corpus | Produces static embeddings; computationally efficient but context-insensitive like Word2Vec |
| **FastText** | A word embedding model that represents words as sums of their subword character n-grams | Handles out-of-vocabulary words by decomposing them into known subword pieces |
| **Static Embedding** | One fixed vector per word regardless of context | Limitation of early models; "bank" in "river bank" and "bank account" gets the same vector |
| **Contextual Embedding** | A vector for a word that changes depending on surrounding words | Produced by transformer models; captures polysemy and nuance |
| **Bi-Encoder** | Architecture where query and document are each encoded separately into a single vector, then compared | Allows documents to be pre-computed and indexed offline; O(1) lookup per document with ANN |
| **Cross-Encoder** | Architecture where query and document are concatenated and fed jointly into the model | More accurate than bi-encoder but cannot pre-index; used for reranking top-K candidates |
| **Mean Pooling** | Averaging all token embeddings across a sequence to produce one sentence vector | Simple and effective aggregation; most common pooling strategy in modern embedding models |
| **CLS Token** | A special [CLS] token prepended to BERT-style inputs; its output is used as the sequence embedding | Standard aggregation for BERT-family models; trained to summarize the full input |
| **Contrastive Learning** | Training objective that pulls embeddings of similar pairs closer and pushes dissimilar pairs apart | The dominant training paradigm for modern embedding models; produces semantically structured spaces |
| **Hard Negatives** | Training examples that are similar to the anchor but not truly relevant (e.g., BM25 near-misses) | Makes contrastive training harder and more discriminative; critical for retrieval quality |
| **In-Batch Negatives** | Using other items in the same training batch as negative examples for each anchor | Efficient training strategy; larger batches provide more negatives and better gradient signal |
| **Temperature (τ)** | A scaling factor in contrastive loss controlling how sharply the model distinguishes positives from negatives | Lower τ makes the model more discriminating; a key hyperparameter for embedding quality |
| **Instruction-Tuned Embeddings** | Embedding models that accept a task description prefix to specialize their output | Allows one model to serve multiple retrieval tasks; improves performance when task is specified |
| **Cosine Similarity** | Similarity measure based on the angle between two vectors, ignoring their magnitudes | Default metric for text embeddings; range [−1, 1]; invariant to vector length |
| **Dot Product** | Sum of element-wise products of two vectors; equals cosine similarity for unit-norm vectors | Computationally efficient; preferred when embeddings are already normalized |
| **Euclidean Distance** | Straight-line distance between two vectors in the embedding space | Less common for text; affected by magnitude; more natural for image embeddings |
| **ANN (Approximate Nearest Neighbor)** | Algorithms that find the closest vectors to a query without comparing against all candidates | Enables sub-millisecond retrieval over millions of embeddings; used in all vector databases |
| **MTEB (Massive Text Embedding Benchmark)** | A standard benchmark suite covering retrieval, classification, clustering, and reranking tasks | The primary leaderboard for comparing embedding model quality across tasks |
| **Matryoshka Representation Learning (MRL)** | Training embeddings such that any prefix of the full vector is also a valid, useful embedding | Enables using shorter embeddings for fast first-stage retrieval and full embeddings for reranking |
| **Late Chunking** | Embedding the full document first, then pooling token-level embeddings at chunk boundaries | Each chunk's embedding retains full-document context from self-attention; improves retrieval precision |
| **Traditional Chunking** | Splitting a document into pieces first, then embedding each chunk independently | Simpler but each chunk loses context from other parts of the document |
| **Binary Quantization** | Reducing each float in an embedding to a single bit (positive → 1, negative → 0) | 32× storage reduction; enables billions of vectors in memory; ~5–10% quality loss |
| **Scalar (Int8) Quantization** | Reducing each float in an embedding to an 8-bit integer | 4× storage reduction; under 1% quality loss; widely supported in vector databases |
| **ColBERT** | Late-interaction retrieval model that stores per-token embeddings and computes MaxSim at query time | Higher precision than single-vector bi-encoders; 10–100× storage cost; used for critical retrieval |
| **Late Interaction** | Computing token-level similarity between query and document embeddings at query time rather than using one vector per item | Preserves more lexical precision than single-vector retrieval; the core idea behind ColBERT |
| **MaxSim** | ColBERT's scoring function: for each query token, find its maximum similarity to any document token, then sum | Captures fine-grained lexical matches while remaining efficient within the late-interaction framework |
| **RAGatouille** | Python library providing a high-level interface for ColBERT indexing and retrieval | Simplifies integration of ColBERT-style late interaction into RAG pipelines |
| **BM25** | A classical lexical ranking algorithm based on term frequency and inverse document frequency | Used to mine hard negatives during embedding training and as a baseline for hybrid retrieval |
| **Embedding Drift** | Incompatibility between embeddings produced by different models or different versions of the same model | Requires re-embedding the entire corpus when the embedding model changes; must be planned for |
| **Blue-Green Deployment** | Running two parallel indexes (old and new embeddings) simultaneously during a migration | Enables atomic, zero-downtime cutover when updating the embedding model |
| **RAG (Retrieval-Augmented Generation)** | Architecture that retrieves relevant documents via embeddings and feeds them as context to an LLM | Combines the accuracy of retrieval with the generative power of LLMs; depends heavily on embedding quality |
| **Vector Database** | A database system optimized for storing, indexing, and querying high-dimensional embedding vectors | Infrastructure layer enabling semantic similarity search at production scale |
| **Pinecone / Qdrant / Weaviate** | Popular managed and open-source vector databases supporting cosine, dot product, and Euclidean metrics | Infrastructure choices for embedding-based retrieval; differ in scalability, filtering, and deployment model |
| **Normalization** | Scaling an embedding vector to unit length (L2 norm = 1) | Makes dot product equivalent to cosine similarity; standard preprocessing for most retrieval systems |
| **Reranking** | A second-pass scoring step that applies a more expensive model (cross-encoder or ColBERT) to re-order top-K candidates | Improves retrieval precision after fast first-stage ANN retrieval |

*Previous: [Transformer Architecture](04-transformer-architecture.md) | Next: [Inference Pipeline](06-inference-pipeline.md)*
