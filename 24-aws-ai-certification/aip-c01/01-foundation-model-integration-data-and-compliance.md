# Domain 1: Foundation Model Integration, Data Management, and Compliance

This is the largest and most foundational domain of the **AWS Certified Generative AI Developer – Professional (AIP-C01)** exam. Domain 1 accounts for **31% of scored questions** — roughly 20 of 65 scored questions — and its concepts cascade into every other domain: the RAG architecture you design here becomes a security question in Domain 3, a cost question in Domain 4, and a quality question in Domain 5.

> **Exam context:** AIP-C01 is AWS's professional-level GenAI *developer* certification. 170 minutes, 65 scored + 10 unscored questions, pass score 750/1000, $300. Registration opened March 2026. Official exam guide: [AIP-C01 Exam Guide](https://docs.aws.amazon.com/aws-certification/latest/examguides/ai-professional-01.html).

> **Plain English:** You are a builder who needs to pick the right AI model off a shelf (Bedrock's catalog), wire it to your company's knowledge (Knowledge Bases / RAG), feed it good data (pipelines), and make sure you can prove to legal that none of it leaks sensitive information or crosses a border it shouldn't (compliance). Domain 1 tests whether you can make all four of those decisions correctly in a real-world scenario.

---

## Table of Contents

- [1. Selecting a Foundation Model from the Bedrock Catalog](#1-selecting-a-foundation-model)
  - [1.1 The Model Catalog](#11-the-model-catalog)
  - [1.2 Key Selection Dimensions](#12-key-selection-dimensions)
  - [1.3 Bedrock vs SageMaker JumpStart](#13-bedrock-vs-sagemaker-jumpstart)
- [2. Retrieval-Augmented Generation (RAG) End-to-End](#2-retrieval-augmented-generation-rag-end-to-end)
  - [2.1 Why RAG?](#21-why-rag)
  - [2.2 Chunking Strategies](#22-chunking-strategies)
  - [2.3 Embedding Models](#23-embedding-models)
  - [2.4 Retrieval: Top-k, Hybrid Search, Reranking](#24-retrieval-top-k-hybrid-search-reranking)
  - [2.5 Context Injection into the Prompt](#25-context-injection-into-the-prompt)
- [3. Amazon Bedrock Knowledge Bases (Managed RAG)](#3-amazon-bedrock-knowledge-bases-managed-rag)
  - [3.1 Data Sources](#31-data-sources)
  - [3.2 Ingestion and Sync](#32-ingestion-and-sync)
  - [3.3 Chunking Configuration](#33-chunking-configuration)
  - [3.4 Supported Vector Stores](#34-supported-vector-stores)
  - [3.5 Metadata Filtering](#35-metadata-filtering)
  - [3.6 Structured Data Retrieval (NL-to-SQL)](#36-structured-data-retrieval-nl-to-sql)
- [4. Data Ingestion Pipelines for GenAI](#4-data-ingestion-pipelines-for-genai)
  - [4.1 S3 as the Data Lake](#41-s3-as-the-data-lake)
  - [4.2 Event-Driven Ingestion Architecture](#42-event-driven-ingestion-architecture)
  - [4.3 Document Parsing with Textract](#43-document-parsing-with-textract)
  - [4.4 PII Handling Before Ingestion](#44-pii-handling-before-ingestion)
- [5. Compliance and Data Management](#5-compliance-and-data-management)
  - [5.1 Bedrock Data Privacy Guarantees](#51-bedrock-data-privacy-guarantees)
  - [5.2 Data Residency and Region Selection](#52-data-residency-and-region-selection)
  - [5.3 Data Classification and Retention](#53-data-classification-and-retention)
  - [5.4 Compliance Certifications](#54-compliance-certifications)
- [Cross-Domain Links](#cross-domain-links)
- [Glossary](#glossary)
- [References](#references)

---

## 1. Selecting a Foundation Model

### 1.1 The Model Catalog

Amazon Bedrock is a **fully managed, serverless API** that gives developers access to foundation models (FMs) from multiple providers through a single AWS API — with no servers to manage and no ML infrastructure to provision. As of mid-2026, the catalog spans more than 100 models from providers including Anthropic, Amazon (Nova family), Meta, Mistral AI, Cohere, AI21 Labs, Stability AI, DeepSeek, Writer, and others. ([Amazon Bedrock model catalog](https://aws.amazon.com/bedrock/))

> **Why (the rationale):** Bedrock eliminates the need to manage ML infrastructure, negotiate model-provider contracts separately, or build multi-provider abstraction layers. It gives one IAM-controlled, VPC-capable, audit-logged endpoint for 100+ models — the alternative is separate accounts with Anthropic, Cohere, Meta, etc., each with different data-privacy terms.
> **When to use:** Any time you need to build GenAI apps on AWS without managing GPU instances or provider contracts; default starting point for all AIP-C01 scenarios unless the question specifically requires SageMaker-level control.
> **Nuances & gotchas:** Not all models are available in all regions — before recommending a model for a data-residency scenario, verify region availability; a model missing from `eu-central-1` breaks the GDPR answer. Third-party models in the Bedrock Marketplace are excluded from some compliance certifications (HIPAA BAA, ISO 27001 scope).

Models are accessed in three ways:

| Access Mode | Description | When to Use |
|---|---|---|
| **On-Demand Inference** | Pay-per-token, no capacity commitment | Default; dev/test, variable traffic |
| **Provisioned Throughput** | Reserved model units (MUs) billed hourly | Predictable high-volume production |
| **Batch Inference** | Asynchronous jobs at ~50% on-demand cost | Large-scale offline tasks, no latency SLA |

**Intelligent Prompt Routing** (generally available 2025–2026) lets you define a model family (e.g., Claude Haiku → Claude Sonnet) and Bedrock's router predicts at request-time which model delivers the best quality/cost ratio for that specific prompt, routing automatically. ([AWS Blog: Model Selection](https://aws.amazon.com/blogs/machine-learning/simplify-model-selection-in-amazon-bedrock-with-the-open-source-model-profiler/))

> **Why (the rationale):** Intelligent Prompt Routing avoids paying frontier-model prices for every request when many queries are simple enough for a cheaper model — it achieves up to 30% cost reduction without any application code change.
> **When to use:** Mixed-complexity workloads within one model family (e.g., a chatbot that handles both trivial FAQ and complex reasoning); when you want automatic cost/quality balancing without writing routing logic yourself.
> **Nuances & gotchas:** Works only *within* a supported model family (Claude tiers, Nova Lite/Pro, Llama variants) — it does NOT route across providers (e.g., Claude to Nova). There is no extra charge for the routing decision itself; you pay only the selected model's token rate. The router's quality prediction is probabilistic — it does not guarantee identical quality to always using the larger model.

---

### 1.2 Key Selection Dimensions

> **⚠️ Verify the numbers, learn the reflexes.** Specific model versions, GA dates, context-window sizes, and per-token prices below are illustrative snapshots and **change frequently** — always confirm current values on the [Bedrock model catalog](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) and [Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/). The exam does **not** test exact prices or version numbers; it tests the *selection reasoning* (context window vs output limit, latency vs accuracy, cost tiers, on-demand vs batch vs provisioned).

#### Context Window

The context window defines the maximum tokens a model can hold in a single call — system prompt + chat history + retrieved RAG chunks + the answer. Overflowing it silently truncates your context.

| Model | Context Window (tokens) | Notes |
|---|---|---|
| Anthropic Claude Opus 4.7 | 1,000,000 | GA on Bedrock April 2026; up to 128K output |
| Amazon Nova 2 Lite | 1,000,000 | GA Dec 2025; text + image + video; cost-optimized |
| Meta Llama 4 Maverick | 1,000,000 | Multimodal |
| Meta Llama 4 Scout | 1,000,000 | Multimodal |
| Claude 3.5 Sonnet v2 | 200,000 | Strong reasoning + coding |
| Amazon Nova Pro | 300,000 | Strong multimodal |
| Amazon Nova Micro | 128,000 | Text-only, ultra-low latency |

> **🎯 On the exam:** A scenario says the app must keep 500 pages (≈ 400K tokens) in context at once. The correct answer is a 1M-token model (Claude Opus 4.7 or Nova 2 Lite), not Claude 3.5 Sonnet (200K limit). Trap: confusing output token limits with context window size.

#### Latency

Latency varies by model size, modality, and inference type. Key signals:
- **Amazon Nova Micro / Haiku**: Lowest latency; suitable for real-time chat, stream-first UX.
- **Latency-optimized inference** on Bedrock (uses specialized hardware): Enables lower time-to-first-token for latency-sensitive workloads.
- **Provisioned Throughput** guarantees a throughput floor but doesn't *reduce* latency the same way as latency-optimized mode.

> **🎯 On the exam:** "Customer requires sub-100ms first-token" → latency-optimized inference tier or a micro/haiku-class model, NOT a large reasoning model even if accuracy is great.

#### Cost (Input / Output Tokens)

Models differ by more than 400× on price. Representative on-demand pricing as of 2026 ([Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)):

| Model | Input ($/1M tokens) | Output ($/1M tokens) |
|---|---|---|
| Amazon Titan Text Lite | ~$0.30 | ~$0.40 |
| Meta Llama 2 Chat 13B | $0.75 | $1.00 |
| Amazon Nova Micro | ~$0.35 | ~$1.40 |
| Claude 3.5 Sonnet v2 | $6.00 | $30.00 |
| Cohere Rerank 3.5 | $2.00 / 1K queries | — |

Key cost levers:
- **Batch inference**: ~50% savings vs on-demand for non-latency-sensitive workloads.
- **Prompt caching** (Anthropic Claude): Cache-write at $7.50/1M tokens, cache-read at $0.60/1M tokens — massive savings when the system prompt is repeated across many requests.
- **Output tokens cost more than input tokens** (typically 3–5×) — minimizing output length (structured JSON responses, concise prompts) meaningfully reduces spend.
- **Amazon Nova**: Up to 75% cheaper than equivalently capable models for many tasks.

> **🎯 On the exam:** "Reduce cost for a workload that sends the same 10K-token system prompt on every request" → **prompt caching**, not a smaller model or provisioned throughput.

#### Modality

| Modality | Models |
|---|---|
| Text-only (fast, cheap) | Amazon Nova Micro, Titan Text Lite, Mistral models |
| Text + Image input | Claude 3.5 Sonnet, Nova Pro, Nova Lite, Llama 4 Maverick |
| Text + Image + Video input | Amazon Nova 2 Lite, Llama 4 Maverick/Scout |
| Image generation | Amazon Nova Canvas, Stability AI SDXL |
| Embeddings | Amazon Titan Text Embeddings V2, Cohere Embed v3 |
| Reranking | Amazon Rerank 1.0, Cohere Rerank 3.5 |

> **🎯 On the exam:** A question about processing video clips for a customer support bot → multimodal model (Nova 2 Lite, Llama 4 Maverick). A question about embeddings for a vector store → Titan Embeddings V2 or Cohere Embed v3. These are *different* model classes; using a text-generation model for embeddings is wrong.

#### Capability & Licensing

- **Anthropic Claude** (Sonnet, Opus): Strongest at reasoning, long-context analysis, code, multilingual. Constitutional AI training. Proprietary.
- **Amazon Nova** family: AWS-native; optimized for AWS integrations; often cheapest-per-capability in class. Proprietary.
- **Meta Llama**: Open weights (Meta's custom license); can be fine-tuned and self-hosted via SageMaker. Lower cost on Bedrock.
- **Cohere**: Strong at RAG-specific tasks (Embed, Rerank). Proprietary.
- **Mistral**: European-origin, efficient models with open weights on some sizes.

> **🎯 On the exam:** Open-weights models (Llama, some Mistral) can be fine-tuned and run on your own infrastructure. Proprietary models (Claude, Nova) can be fine-tuned only via Bedrock's managed fine-tuning APIs — you never see the weights.

---

### 1.2.1 AWS Well-Architected Generative AI Lens (Task 1.1.3)

The **AWS Well-Architected Tool** includes a **Generative AI Lens** — a set of best-practice questions and guidance specifically for reviewing GenAI architectures against the six pillars (Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability). Use it to standardise and audit a GenAI solution design before production launch.

> **🎯 On the exam:** "Team needs a structured framework to review their GenAI architecture for best practices" → **AWS Well-Architected Tool with the Generative AI Lens** (not a generic Architecture Review Board process). ([AWS Well-Architected Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/genai-lens.html))

---

### 1.2.2 Dynamic Model Selection with AWS AppConfig (Task 1.2.2)

**AWS AppConfig** (part of AWS Systems Manager) enables **dynamic model selection and provider switching at runtime, without code changes or redeployments**. Store the active `modelId`, routing weights, or provider configuration as an AppConfig feature flag or freeform configuration. A Lambda reads the config on each invocation (with AppConfig client-side caching) — changing the active model is an AppConfig deployment, not a Lambda deployment.

**Pattern:** AppConfig (model routing config) → Lambda (reads config on start) → API Gateway → Bedrock `Converse`

> **Why:** Eliminates the need to redeploy application code when switching between model providers or versions (e.g., migrating from Claude 3.5 Sonnet to Nova Pro, or A/B testing two models). Pair with Lambda environment variables for region and Bedrock endpoint.
> **🎯 On the exam:** "Change the active FM without a code deployment" → **AWS AppConfig** for externalised model routing config. ([AWS AppConfig docs](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html))

---

### 1.3 Bedrock vs SageMaker JumpStart

> **Why (the rationale):** Bedrock is zero-ops for app builders; SageMaker JumpStart is full-control for ML engineers who need custom training scripts, dedicated GPU instances, or models not in the Bedrock catalog.
> **When to use:** Choose Bedrock when you want a managed API with native RAG, agents, and guardrails. Choose SageMaker JumpStart when you need to fine-tune open-weight models with custom code, need dedicated persistent endpoints, or need a model that is not in the Bedrock catalog.
> **Nuances & gotchas:** SageMaker JumpStart bills per GPU-hour even when idle — a forgotten endpoint costs money continuously. Bedrock on-demand costs $0 when not in use. A hybrid pattern (fine-tune in SageMaker → import into Bedrock via Custom Model Import) combines the strengths of both, but model architectures must be on Bedrock's supported import list.

Both services give access to foundation models, but they serve different developer needs. ([AWS Decision Guide](https://docs.aws.amazon.com/pdfs/decision-guides/latest/bedrock-or-sagemaker/bedrock-or-sagemaker.pdf))

| Dimension | Amazon Bedrock | SageMaker JumpStart |
|---|---|---|
| **Access model** | Serverless API; zero infrastructure | Deploys to compute instances (you pick instance type) |
| **Billing** | Per-token (on-demand) or hourly (provisioned throughput) | Per-hour compute; billed even when idle |
| **Model catalog** | 100+ curated FMs; proprietary + open weights | Hugging Face, open-weight FMs, some proprietary |
| **Fine-tuning** | Managed (no instance management); stored as model customizations | Full control: scripts, hyperparameters, instance type |
| **RAG / Knowledge Bases** | Native (Bedrock Knowledge Bases, Agents) | DIY; integrate with OpenSearch / your own vector DB |
| **Operational overhead** | Near-zero | Moderate-to-high (instance sizing, scaling, patching) |
| **Best for** | App builders: chatbots, RAG, agents, rapid prototyping | ML engineers: custom training, specialized fine-tuning, long-running inference |
| **Cost when idle** | $0 (on-demand) | Billed per hour even if no requests |

> **🎯 On the exam:** A startup wants a customer-service chatbot with RAG on company documents; no ML team → **Bedrock + Knowledge Bases**. A research team needs to fine-tune Llama on proprietary data using custom training scripts → **SageMaker JumpStart**. Trap: SageMaker JumpStart is not the right answer when the scenario asks for "managed RAG" or "serverless inference."

You can also **deploy a SageMaker JumpStart model and register it with Bedrock**, allowing access through Bedrock APIs — but this is an advanced integration pattern, not the default exam answer.

### 1.3.1 Parameter-Efficient Fine-Tuning: LoRA / Adapters (Task 1.2.4)

When full fine-tuning is too expensive or slow, **LoRA (Low-Rank Adaptation)** and similar **adapter** techniques offer a parameter-efficient alternative: only a small set of additional weight matrices (the "adapter") is trained and stored — the base model weights are frozen. This makes domain-specific fine-tuning dramatically cheaper and faster, and the tiny adapter weights are swappable without re-serving the full model.

| Approach | What changes | Cost relative to full fine-tuning | Use case |
|---|---|---|---|
| **Full fine-tuning** | All weights updated | High | Large behaviour shift; maximum adaptation |
| **LoRA / Adapters** | Only low-rank adapter layers trained | Low (< 1% of parameters typically) | Domain vocabulary, style, task format; faster iteration |

**Lifecycle on AWS:**
- Fine-tune (including LoRA) via **SageMaker Training Jobs** (open-weight models: Llama, Mistral).
- Version and track adapter artifacts with **SageMaker Model Registry** — supports model approval workflows, lineage, and metadata.
- Automated deployment with rollback: use **SageMaker Pipelines** or **EventBridge + Lambda** to trigger deployments; register new versions and use Model Registry approval gates for automated rollback if metrics regress.
- Retire old adapters via Model Registry lifecycle (archive/deprecate state).

> **🎯 On the exam:** "Cheapest/fastest way to fine-tune a domain-specific model" → **LoRA / parameter-efficient fine-tuning** (not full fine-tuning, not continued pre-training). "Versioning and lifecycle management of fine-tuned models" → **SageMaker Model Registry**.

---

## 2. Retrieval-Augmented Generation (RAG) End-to-End

### 2.1 Why RAG?

Foundation models are frozen at their training cutoff and have no access to proprietary, real-time, or internal data. RAG solves this by: (1) retrieving relevant context from your own knowledge sources at query time, and (2) injecting that context into the prompt before the model generates an answer. This avoids hallucination on domain-specific facts without the cost and risk of retraining or fine-tuning.

```
User Query
   │
   ▼
[Embedding Model] ──── Query Vector ────▶ [Vector Store]
                                                │
                                         Top-k Chunks
                                                │
                                                ▼
                              [Prompt = System + Context + Query]
                                                │
                                                ▼
                                    [Foundation Model] ──▶ Answer
```

> **Plain English:** Think of RAG as open-book exam mode for the AI. Instead of relying only on what it memorized, it can look up the right pages of your internal documentation and incorporate them into its answer.

---

### 2.2 Chunking Strategies

> **Why (the rationale):** Embedding models have token limits (typically 512–8192 tokens); a 100-page PDF cannot be embedded as one unit. Chunking converts long documents into embeddable, retrievable segments. Chunk quality directly controls retrieval quality — poor chunking is the most common root cause of RAG failures.
> **When to use:** Required for any document-based RAG ingestion pipeline. Strategy choice depends on document type: fixed-size for uniform plain text, semantic for dense technical docs with abrupt topic changes, hierarchical (parent-child) when you need both retrieval precision and rich response context.
> **Nuances & gotchas:** In Bedrock Knowledge Bases, the chunking configuration is **immutable per data source** — changing it requires deleting and recreating the data source and re-ingesting all documents. Hierarchical chunking with very large parent sizes can exceed S3 Vectors' 1 KB metadata limit per vector; switch to OpenSearch if that happens.

Before documents can be embedded, they must be split into chunks — smaller text units that fit within embedding model limits and carry a single coherent idea. ([Bedrock Knowledge Bases chunking documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html))

| Strategy | How it works | Best for | Tradeoff |
|---|---|---|---|
| **Fixed-size** | Split every N tokens, with M% overlap between chunks | Simple corpora, uniform text | May split mid-sentence; cheap to compute |
| **Recursive / sentence-aware** | Split at natural boundaries (paragraph → sentence → word), respecting semantics | General-purpose, structured docs | Slightly more compute |
| **Semantic** | Embed adjacent sentences; cut where similarity drops below a threshold | Dense technical docs where topic shifts mid-paragraph | Higher compute cost; requires embedding at ingest |
| **Hierarchical (Parent-Child)** | Two layers: small child chunks (precise matching) + large parent chunks (rich context) | Complex documents needing both precision and context | Doubles storage; more complex retrieval logic |
| **No chunking / Custom (Lambda)** | Each file = one chunk, or apply a custom Lambda transformation post-chunking | Very short documents; specialized parsing needs | Not scalable for large documents |

**Critical exam fact:** In Amazon Bedrock Knowledge Bases, the **chunking configuration is fixed once a data source is created**. Changing it requires creating a new data source and re-ingesting all documents. ([hidekazu-konishi.com](https://hidekazu-konishi.com/entry/amazon_bedrock_knowledge_bases_retrieval_quality_engineering.html))

> **🎯 On the exam:** A scenario where answers miss surrounding context → switch from fixed to **hierarchical (parent-child)** chunking. A scenario where chunks cut off mid-topic → **semantic chunking**. Never answer "just retrieve more chunks" — fix the chunking strategy first.

---

### 2.3 Embedding Models

> **Why (the rationale):** Embedding models are the translation layer that converts raw text into the numeric vector space where similarity search operates. Your choice of embedding model determines the semantic richness and language coverage of your retrieval — a poor embedding model means even perfect chunking produces irrelevant results.
> **When to use:** Choose Amazon Titan Text Embeddings V2 as the default (AWS-native, configurable dimensions, binary vector support). Switch to Cohere Embed v3 when multilingual support (100+ languages) is required or when you need int8 quantization.
> **Nuances & gotchas:** The vector dimensions chosen at embedding time must **exactly match** the dimensions configured in the vector store index — a mismatch causes ingestion failures with no graceful fallback. If you change the embedding model or dimension, you must re-embed and re-ingest all documents. Binary vectors (for ~32× storage compression) are only supported on OpenSearch — not on S3 Vectors, MemoryDB, or Aurora pgvector.

An embedding model converts text into a dense numeric vector that captures semantic meaning. Vectors for semantically similar text cluster together in the embedding space.

| Model | Dimensions | Notes |
|---|---|---|
| **Amazon Titan Text Embeddings V2** | 256 / 512 / 1024 (configurable) | AWS-native; default in Bedrock Knowledge Bases; supports binary vectors |
| **Cohere Embed v3** | 1024 | Strong multilingual; English + 100 languages; int8 quantization support |

**Why dimensions matter:**
- Higher dimensions → richer representation but more storage and slower retrieval.
- Amazon Titan Embeddings V2's configurable dimensions let you trade off quality vs cost: 256 for cost optimization, 1024 for maximum fidelity.
- The vector dimensions you choose at embedding time **must match** the dimensions configured in your vector store index — a mismatch causes ingestion failures.

**Binary vectors:** OpenSearch Serverless and OpenSearch Managed Clusters are the only Bedrock-supported vector stores that support **binary vector embeddings** (using Hamming distance), which can reduce storage by ~32× compared to float32.

> **🎯 On the exam:** "Reduce vector storage costs without changing the model" → use **lower embedding dimensions** (256 instead of 1024 on Titan V2) or **binary vectors on OpenSearch**. Trap: switching to a completely different model when a dimension config change would suffice.

---

### 2.4 Retrieval: Top-k, Hybrid Search, Reranking

> **Why (the rationale):** Retrieval quality determines the ceiling for RAG answer quality. Even a frontier model cannot produce correct answers when the retrieved chunks are wrong or incomplete. The retrieval stack (top-k → hybrid search → reranking → query decomposition) progressively improves precision and recall.
> **When to use:** Start with top-k semantic search. Add hybrid search when exact-match queries fail (product codes, model numbers, IDs). Add reranking when the top-k set is relevant in aggregate but the order is wrong. Add query decomposition when multi-part questions return incomplete answers.
> **Nuances & gotchas:** Hybrid search in Bedrock Knowledge Bases silently falls back to semantic-only on unsupported vector stores (e.g., Pinecone, Redis) — you won't get an error, the keyword path just doesn't run. Reranking requires a separate reranker model call (Amazon Rerank 1.0 or Cohere Rerank 3.5) billed per query — it is not free. Query decomposition is only available on `RetrieveAndGenerate`, not the bare `Retrieve` API.

**Top-k Retrieval**

The simplest retrieval strategy: embed the user query, find the k nearest vectors in the store (by cosine or Euclidean distance), and return those k chunks. Default k is often 5–20. Larger k increases recall but also injects more noise into the prompt, consuming tokens and potentially confusing the model.

**Hybrid Search**

Pure vector (semantic) search fails on exact-match lookups: a model number like "EC2-r8g.48xlarge" might not be close to other "r8g" chunks in embedding space. Hybrid search combines:
- **Vector search**: semantic similarity
- **Full-text (BM25/keyword) search**: exact term matching

Results from both are merged (typically via Reciprocal Rank Fusion) and the best of both worlds is presented to the reranker or the model.

```
Query ──▶ [Vector Search] ──▶ semantic candidates ┐
     └──▶ [Keyword Search] ──▶ keyword candidates ─┤──▶ [Merge + Rerank] ──▶ Top-N chunks
```

Hybrid search is available in Bedrock Knowledge Bases using `overrideSearchType: HYBRID`. Supported on OpenSearch Serverless, Aurora PostgreSQL (pgvector), and MongoDB Atlas. It **silently falls back** to semantic-only on unsupported stores. ([hidekazu-konishi.com](https://hidekazu-konishi.com/entry/amazon_bedrock_knowledge_bases_retrieval_quality_engineering.html))

**Reranking**

After retrieving a generous candidate set (e.g., top-20 with hybrid search), a **cross-encoder reranker** scores each candidate against the query more precisely — but with higher compute cost per candidate. The output is a smaller, better-ordered set (e.g., top-5) sent to the model.

The **"retrieve wide, rerank narrow"** pattern (retrieve 15–20, rerank to 5) consistently outperforms retrieving 5 directly. ([Bedrock Advanced RAG](https://medium.com/@suhasmallesh/bedrock-knowledge-base-advanced-rag-with-terraform-chunking-hybrid-search-and-reranking-02b15c5bc763))

Available rerankers in Bedrock:

| Reranker | Model ID |
|---|---|
| Amazon Rerank 1.0 | `amazon.rerank-v1:0` |
| Cohere Rerank 3.5 | `cohere.rerank-v3-5:0` |

Configure via `rerankingConfiguration` inside `vectorSearchConfiguration` on the `RetrieveAndGenerate` API.

**Query Decomposition**

For multi-part questions ("What is our refund policy and what were Q3 revenue figures?"), Bedrock Knowledge Bases can decompose the query into sub-queries, retrieve for each independently, and pool the results before generation. Enabled via `orchestrationConfiguration` with `type: QUERY_DECOMPOSITION` — only available on `RetrieveAndGenerate`, not the bare `Retrieve` API.

> **🎯 On the exam:** "RAG answers miss exact product codes" → enable **hybrid search**. "Top-5 chunks look relevant but the final answer is wrong" → try **reranking** (retrieve 20, rerank to 5). "Complex multi-part question returns incomplete answers" → enable **query decomposition**.

---

### 2.5 Context Injection into the Prompt

Retrieved chunks are assembled into the context window in this order (standard Bedrock pattern):

```
[System prompt]           ← Role, instructions, output format
[Retrieved context]       ← Chunks injected here, usually with source citations
[Conversation history]    ← Prior turns (in chat applications)
[User query]              ← Current question
```

**Prompt engineering tips for RAG (exam-relevant):**
- Instruct the model to **only use the provided context** and to say "I don't know" if context is insufficient — prevents hallucination from parametric memory.
- Include **source document identifiers** in chunks so the model can cite sources.
- Place retrieved context **before** the user question for best attention over long contexts.
- If context is too large, **reranking** (above) helps select only the most relevant chunks.

---

## 3. Amazon Bedrock Knowledge Bases (Managed RAG)

Amazon Bedrock Knowledge Bases is AWS's **fully managed RAG service**. It handles data ingestion, chunking, embedding, vector store loading, and retrieval — abstracting away every component of the DIY RAG pipeline. ([Bedrock Knowledge Bases documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html))

> **Plain English:** Instead of wiring together S3 + Lambda + an embedding call + OpenSearch + your own retrieval code, Knowledge Bases does all of that for you. You point it at your data, pick a vector store, and call `RetrieveAndGenerate`.

---

### 3.1 Data Sources

> **Why (the rationale):** Native connectors eliminate the need to build custom ETL pipelines for common enterprise sources. Without them you'd need a Lambda + API polling loop for each source, managing auth, pagination, rate limits, and incremental sync yourself.
> **When to use:** Use native connectors (SharePoint, Confluence, Salesforce, Google Drive, web crawler) whenever the data already lives in one of the supported platforms. Fall back to S3 + custom ETL only when the source is not in the supported list or requires complex transformations before ingestion.
> **Nuances & gotchas:** Not all connectors support real-time sync — most are scheduled or on-demand. The Web Crawler connector crawls only public URLs; it cannot authenticate to behind-login pages. Smart Parsing uses an FM to parse complex PDFs and tables, which incurs additional FM invocation costs on top of ingestion costs.

Knowledge Bases supports native connectors to the following data sources ([AWS Blog: Additional Data Connectors](https://aws.amazon.com/blogs/aws/knowledge-bases-for-amazon-bedrock-now-supports-additional-data-connectors-in-preview/)):

| Data Source | Notes |
|---|---|
| **Amazon S3** | Primary source; supports PDF, Word, HTML, Markdown, CSV, JSON, TXT |
| **Confluence** | Crawls Confluence spaces and pages |
| **SharePoint** | Microsoft SharePoint Online |
| **Web Crawler** | Crawls public web URLs up to a configurable depth |
| **Salesforce** | CRM objects and knowledge articles |
| **Google Drive** | Documents and Drive folders |
| **OneDrive** | Microsoft OneDrive |

**Smart Parsing** (managed feature, 2026): Bedrock automatically selects the right parsing strategy per content type — including using an FM to parse complex PDFs, tables, and charts — without custom code.

> **🎯 On the exam:** "Company has docs in SharePoint and S3" → **Bedrock Knowledge Bases with multiple data sources**. No need for a custom Lambda to pull SharePoint; a native connector exists. Trap: building a custom ETL pipeline when a native connector covers the data source.

---

### 3.2 Ingestion and Sync

Documents flow through this managed pipeline:

```
Data Source (S3/SharePoint/…)
        │
        ▼
   [Document Parser]  ← Extracts text; optionally uses FM for complex docs
        │
        ▼
   [Chunker]          ← Applies configured chunking strategy
        │
        ▼
   [Embedding Model]  ← Titan Embeddings V2 or Cohere Embed v3
        │
        ▼
   [Vector Store]     ← Writes vectors + metadata + raw text
```

**Sync modes:**
- **On-demand sync**: Triggered manually or via API. Processes new/changed/deleted documents since last sync.
- **Scheduled sync**: Available for some connectors; periodic refresh.
- **Full ingestion**: Re-processes all documents; needed after chunking config changes.

---

### 3.3 Chunking Configuration

(Detailed in Section 2.2 above; exam-specific detail for Knowledge Bases:)

- Configuration is **immutable per data source** — changing it requires a new data source + full re-ingestion.
- Hierarchical chunking with very high token counts can **exceed S3 Vectors metadata size limits** because parent-child relationships are stored as metadata per vector. Reduce parent chunk size or switch to OpenSearch if this occurs.

---

### 3.4 Supported Vector Stores

> **Why (the rationale):** Vector store choice affects cost, latency, operational overhead, and available features (binary vectors, SQL joins, graph traversal). There is no universally best store — each has a dominant use case, and mixing them in a tiered architecture (e.g., S3 Vectors for cold + OpenSearch for hot) is a valid exam-tested pattern.
> **When to use:** See the decision tree in the "On the exam" callout below. OpenSearch Serverless is the correct default for most scenarios. Only deviate when a specific differentiator (graph, sub-ms latency, billions-of-vectors cost, existing PostgreSQL app) is mentioned.
> **Nuances & gotchas:** S3 Vectors supports only **floating-point** embeddings — not binary (that's OpenSearch). Neptune Analytics' vector search index can only be set at **graph creation time** and cannot be changed later. MemoryDB is in-memory and multi-AZ durable but costs more than S3 Vectors. Aurora pgvector requires a **GIN index** for metadata filtering; range filters need a separate expression index. Pinecone and Redis Enterprise are third-party — credentials must be stored in AWS Secrets Manager.

This is a **high-frequency exam topic**. Know each store's sweet spot and the exam signal that points to it. ([AWS Prescriptive Guidance: Choosing a Vector Database](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html))

| Vector Store | Default? | Best For | Key Differentiator |
|---|---|---|---|
| **Amazon OpenSearch Serverless** | ✅ Yes (auto-created) | General-purpose production RAG; hybrid search; binary vectors | No capacity planning; scales automatically; supports HNSW/Faiss; only store supporting binary embeddings |
| **Amazon OpenSearch Managed Cluster** | No | High-throughput, GPU-accelerated queries; large-scale | More control over instance type; requires domain management |
| **Amazon Aurora PostgreSQL (pgvector)** | No | Existing PostgreSQL apps needing vector + relational queries | Run SQL JOIN + vector similarity in one query; HNSW + IVFFlat indexes |
| **Amazon Neptune Analytics (GraphRAG)** | No | Knowledge graphs; relationship-traversal + vector similarity | Only choice for Graph RAG; combines graph algorithms (path-finding, community detection) with vector search; up to 80× faster than existing graph solutions |
| **Amazon MemoryDB** | No | Real-time, sub-millisecond latency; high throughput | Fastest vector search on AWS; in-memory; multi-AZ durable; up to 32,768 dimensions |
| **Amazon S3 Vectors** | No | Billions of vectors at lowest cost; infrequent access | Up to 90% cheaper than specialized vector DBs; sub-100ms latency; 11-nine durability; serverless; no provisioning |
| **Pinecone** | No | Teams already using Pinecone | Third-party managed; credentials via Secrets Manager |
| **Redis Enterprise Cloud** | No | Teams already using Redis | Third-party managed; TLS required |
| **MongoDB Atlas** | No | Teams using MongoDB; document + vector in one DB | Requires manual metadata filter config in Atlas index |

**OpenSearch Neural (neural search) plugin (Task 1.4.1):**
Amazon OpenSearch Service includes a **Neural plugin** that integrates natively with Amazon Bedrock to **automatically generate embeddings at both ingest time and query time** — eliminating the need for a separate embedding step in your pipeline. You configure a Bedrock embedding model in an OpenSearch `ml_config`, and the plugin calls Bedrock automatically when you index a document or run a `neural_query`. This makes OpenSearch a self-contained vector search solution: ingest text, get vectors; query text, get semantically ranked results.

> **🎯 On the exam:** "Vector search pipeline without a separate Lambda to call an embedding model" → **OpenSearch Service with the Neural plugin + Bedrock integration** — the plugin handles embedding automatically. ([OpenSearch neural search docs](https://opensearch.org/docs/latest/search-plugins/neural-search/))

**S3 Vectors deep-dive (new 2025/2026 — high exam priority):**
- First cloud object store with **native vector storage and query**.
- Stores up to **2 billion vectors per index**, up to 10,000 indexes per vector bucket.
- Sub-100ms query latency optimized for **infrequent access** (not sub-10ms like MemoryDB).
- Up to **90% cost reduction** vs specialized vector databases.
- Serverless — no infrastructure to provision.
- Only supports **floating-point embeddings** (not binary).
- Metadata: up to 1 KB + 35 keys per vector; exceeding this during ingestion throws an exception.
- Works in a **tiered architecture**: S3 Vectors for cold/archival storage, OpenSearch for hot/frequent-access queries. ([Amazon S3 Vectors documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html))

**Neptune Analytics (GraphRAG) deep-dive:**
- Integrates directly with Bedrock Knowledge Bases for fully managed **GraphRAG**.
- Create an **empty graph with a vector search index** first — the vector index can **only be set at graph creation time** and cannot be changed later.
- Combines vector similarity with graph traversal algorithms: path-finding, community detection, centrality.
- Ideal for: customer 360 views, fraud detection networks, knowledge graphs, explainable AI with relationship context. ([Neptune Analytics documentation](https://docs.aws.amazon.com/neptune-analytics/latest/userguide/what-is-neptune-analytics.html))

> **🎯 On the exam (vector store decision tree):**
> - "Managed RAG, no ops team, corpus < 10M chunks" → **OpenSearch Serverless** (default)
> - "Graph relationships matter (customer 360, fraud network)" → **Neptune Analytics** (GraphRAG)
> - "Cheapest large-scale vectors, infrequent queries, billions of vectors" → **S3 Vectors**
> - "Sub-millisecond latency, real-time recommendations" → **MemoryDB**
> - "Existing PostgreSQL app, need SQL + vector in same query" → **Aurora PostgreSQL / pgvector**
> - "Binary embeddings for storage compression" → **OpenSearch Serverless or Managed Cluster** (only stores supporting binary vectors)

---

### 3.5 Metadata Filtering

> **Why (the rationale):** Without metadata filtering, a vector search over a multi-tenant or multi-category knowledge base retrieves chunks from all tenants/categories. Metadata filtering enforces hard boundaries before semantic ranking, preventing cross-tenant data bleed and dramatically improving precision.
> **When to use:** Any time retrieval scope must be constrained by attribute (department, tenant ID, date range, document type, security classification). Use explicit filtering when the application knows the filter value (e.g., logged-in user's department). Use implicit filtering when you want the model to infer the filter from the user's natural-language query.
> **Nuances & gotchas:** Metadata must be pre-attached as sidecar `.metadata.json` files in S3 alongside the source document **before** ingestion — you cannot add metadata retroactively without re-ingesting. Aurora pgvector requires a GIN index on the metadata column for filter performance; range-based numeric filters need an additional expression index or they become full table scans.

Metadata filtering constrains the vector search to documents matching specific attribute criteria before semantic ranking — dramatically improving precision for filtered use cases (e.g., "only search documents from the Legal department tagged 2025").

**Two modes:**
1. **Explicit filtering**: Your application passes a filter object in the API request. Operators: `equals`, `notEquals`, `in`, `notIn`, `greaterThan`, `lessThan`, `stringContains`, `startsWith`, plus logical `andAll` / `orAll`.
2. **Implicit filtering**: Bedrock infers the filter from the natural-language query using Claude, against a declared metadata schema you provide. No code required in the request.

**Attaching metadata:** Place a sidecar JSON file (same key prefix as the source document, `.metadata.json` extension) in S3 alongside each document. Bedrock ingests these attributes into the vector store metadata fields.

**Aurora-specific note:** Metadata filtering on Aurora requires a **GIN index** on the metadata column. For range filters on numeric fields (e.g., `year < 2020`), add a dedicated expression index on that key for performance.

> **🎯 On the exam:** "HR chatbot should only search documents tagged with the employee's department" → **metadata filtering with explicit filter** passed in the API call. "User says 'show me docs from last year' and expects automatic scoping" → **implicit metadata filtering**.

---

### 3.6 Structured Data Retrieval (NL-to-SQL)

> **Why (the rationale):** Business users need to query data warehouses without knowing SQL. A text-to-SQL layer lets them ask plain-English questions and get data-driven answers without analyst intervention. Bedrock's managed NL2SQL eliminates the need to build a custom text-to-SQL pipeline with schema introspection, query validation, and execution.
> **When to use:** When the question is about structured/tabular data (metrics, aggregates, time-series, sales figures) stored in Redshift or SageMaker Lakehouse. Not a substitute for unstructured RAG — use both (one Knowledge Base for documents, NL2SQL for data) if you need to answer questions spanning both.
> **Nuances & gotchas:** NL2SQL works only with **Amazon Redshift** and **SageMaker Lakehouse** — not RDS, Aurora, Athena, or DynamoDB directly. Bedrock analyzes the schema at query time; poorly named columns or missing table comments reduce SQL accuracy. No data is moved; Bedrock queries the live source, so IAM permissions must grant Bedrock access to the Redshift cluster or Lakehouse catalog.

Beyond unstructured text RAG, Bedrock Knowledge Bases supports **natural language to SQL (NL2SQL)** queries over structured data sources. ([AWS Blog: Structured Data Retrieval](https://aws.amazon.com/blogs/machine-learning/build-conversational-interfaces-for-structured-data-using-amazon-bedrock-knowledge-bases/))

- Bedrock analyzes the database **schema, table relationships, and historical queries** to generate accurate SQL.
- Supported structured data sources: **Amazon Redshift** and **Amazon SageMaker Lakehouse** (via AWS Glue Data Catalog).
- No data movement required — Bedrock queries the source at retrieval time.
- Available in all commercial regions where Bedrock Knowledge Bases is supported.

**Practical flow:**
```
User: "What were Q3 2025 sales by region?"
         │
         ▼
  [Bedrock NL2SQL Module]
         │  analyzes Redshift schema
         ▼
  SELECT region, SUM(revenue) FROM sales
  WHERE quarter='Q3' AND year=2025
  GROUP BY region;
         │
         ▼
  [Redshift executes query]
         │
         ▼
  [Results injected into FM prompt → Natural-language answer]
```

> **🎯 On the exam:** "Business users want to ask plain English questions about their Redshift data warehouse" → **Bedrock Knowledge Bases structured data retrieval (NL2SQL over Redshift)**. Not Athena, not a custom Lambda, not a text-to-SQL agent built from scratch.

---

## 4. Data Ingestion Pipelines for GenAI

### 4.1 S3 as the Data Lake

Amazon S3 is the **canonical staging area** for all GenAI data ingestion on AWS. Documents land in S3 first, regardless of their original source (email, CRM export, database dump, scanned PDF). This is the "single source of truth" that Bedrock Knowledge Bases, Textract, Comprehend, and Macie all operate against.

**Best practices:**
- Use **S3 versioning** to track document changes and enable rollback.
- Apply **S3 Object Tags** (e.g., `classification=PII`, `department=Legal`) as the source-of-truth for metadata that flows into the vector store via sidecar JSON files.
- Use **S3 Intelligent-Tiering** for large document archives that are infrequently accessed.
- Store raw documents in one S3 prefix and processed/clean documents in a separate prefix — never ingest raw without PII scanning.

---

### 4.2 Event-Driven Ingestion Architecture

A fully serverless, event-driven pipeline for continuous RAG data ingestion:

```
S3 (raw documents land)
        │  S3 Event Notification (ObjectCreated)
        ▼
   AWS Lambda (trigger)
        │  starts Step Functions execution
        ▼
   AWS Step Functions (State Machine)
        │
        ├──▶ [Amazon Textract] ← Extract text from PDFs/scanned images
        │
        ├──▶ [Amazon Comprehend] ← Detect + redact PII entities
        │
        ├──▶ [Amazon Macie] ← Classify S3 object sensitivity
        │
        ├──▶ [Write cleaned text to processed S3 prefix]
        │
        └──▶ [Trigger Bedrock Knowledge Bases sync]
```

**Step Functions** is the orchestration layer for error handling, retries, branching (e.g., skip Textract for plain-text files), and parallel processing. Step Functions **Express Workflows** are ideal for high-volume, short-duration ingestion steps; **Standard Workflows** for long-running, auditable pipelines.

**Lambda** is the event bridge: it receives the S3 event, validates the file, and starts the Step Functions execution. Lambda alone is insufficient for complex multi-step orchestration with error handling.

> **🎯 On the exam:** "Documents are uploaded to S3 throughout the day and must be processed automatically" → **S3 event notification → Lambda → Step Functions → Bedrock KB sync**. Trap: "Lambda alone" — Lambda times out after 15 minutes and has no native state management for multi-step workflows.

---

### 4.3 Document Parsing with Textract

> **Why (the rationale):** PDFs and scanned images are the most common enterprise document formats, but they cannot be directly read as plain text by a vector embedding pipeline. Textract converts these binary formats into structured text (with table and form awareness) that downstream chunking and embedding can process.
> **When to use:** Any time source documents are scanned images, multi-page PDFs with tables/forms, or complex layouts. Skip Textract when documents are already plain text, HTML, Markdown, or CSV — Bedrock Knowledge Bases parses those natively without a Textract call.
> **Nuances & gotchas:** `DetectDocumentText` is synchronous and single-page only; use `StartDocumentAnalysis` (async) for multi-page PDFs in production ingestion pipelines. Textract does NOT read password-protected PDFs — you must decrypt them before passing to Textract. Table extraction via `AnalyzeDocument` returns a structured JSON block graph, not raw text rows — you need additional parsing logic to flatten tables for embedding.

**Amazon Textract** extracts text, tables, and forms from scanned images and PDFs (including multi-column, complex layouts). It goes far beyond basic OCR.

| Textract API | Use Case |
|---|---|
| `DetectDocumentText` | Simple text extraction (single page) |
| `AnalyzeDocument` | Tables, forms, key-value pairs |
| `StartDocumentAnalysis` | Async; large multi-page PDFs |
| `AnalyzeExpense` | Invoice, receipt parsing |
| `AnalyzeID` | Identity documents |

For RAG pipelines, `StartDocumentAnalysis` is the standard choice for processing large batches of multi-page PDFs asynchronously.

> **🎯 On the exam:** "Scanned insurance claims in PDF format need to be searchable in RAG" → **Textract** for extraction (not Lambda + regex, not S3 Select). If the docs are already plain text/HTML/Markdown, Textract is unnecessary — Bedrock Knowledge Bases parses those natively.

---

### 4.4 PII Handling Before Ingestion

Ingesting PII (names, SSNs, emails, health data) into a vector store creates compliance risk: the data becomes embedded in potentially hundreds of vector chunks and is hard to selectively delete. The correct approach is a **layered PII defense** *before* data reaches the vector store. ([AWS PII detection patterns](https://hidekazu-konishi.com/entry/pii_detection_and_redaction_patterns_for_generative_ai_on_aws.html))

**Layer 1 — Classify at the bucket level: Amazon Macie**
- Macie is a fully managed data security service that uses ML to discover and classify sensitive data in S3.
- Scan all incoming S3 objects with Macie to identify which documents contain PII categories.
- Use Macie findings to route documents: quarantine high-sensitivity docs, flag for review, or pass to redaction.

**Layer 2 — Detect and redact at the text level: Amazon Comprehend**
- After Textract extraction, call `DetectPiiEntities` to identify entity types (NAME, SSN, EMAIL, PHONE, etc.) with their character offsets.
- Replace detected spans with placeholder tokens (e.g., `[REDACTED_NAME]`) before chunking and embedding.
- Comprehend supports 18+ PII entity types in English; 5+ in other languages.

**Layer 3 — Block at the model boundary: Bedrock Guardrails**
- Even after pre-ingestion redaction, use **Bedrock Guardrails** to intercept PII in model inputs/outputs at inference time as a safety net.
- Configure sensitive information filters to mask or block specified PII types in both prompt and response.

**Decision guide:**
| Service | Where it operates | What it does |
|---|---|---|
| **Amazon Macie** | S3 (object level) | Classifies and alerts on sensitive data at rest |
| **Amazon Comprehend** | Text string (in-pipeline) | Detects and redacts PII spans before embedding |
| **Bedrock Guardrails** | Prompt/response (inference time) | Blocks or masks PII that slips through earlier layers |

> **🎯 On the exam:** "Prevent SSNs from being stored in the vector database" → **Comprehend PII detection in the ingestion pipeline before embedding**. "Alert the security team when sensitive data lands in S3" → **Macie**. "Block PII from appearing in model outputs" → **Bedrock Guardrails**. These are complementary, not alternatives.

---

## 5. Compliance and Data Management

### 5.1 Bedrock Data Privacy Guarantees

> **Why (the rationale):** Enterprise customers must satisfy legal and contractual obligations that prohibit their data from being used to train external AI models. Bedrock's privacy guarantee is the contractual basis for using Anthropic, Meta, or Cohere models without violating those obligations — the alternative (calling the provider API directly) sends data to the provider without the same guarantee.
> **When to use:** Always cite these guarantees when a scenario involves regulated industries, customer PII, proprietary IP, or any context where "will AWS/Anthropic see my data?" is the key concern.
> **Nuances & gotchas:** These guarantees apply to **Bedrock-mediated inference** only — if you call Anthropic's API directly (not through Bedrock), your data is subject to Anthropic's own privacy policy, not AWS's. Fine-tuning with your data creates a private copy of the model trained on your data; the guarantee is that this copy is isolated to your account, not that your fine-tuning data is kept confidential from the training process itself.

Amazon Bedrock has explicit data privacy commitments that are **directly tested on AIP-C01**:

1. **Your data is not used to train base models.** Prompts, completions, embeddings, and fine-tuning data submitted to Bedrock are never used to improve the underlying foundation models — neither Anthropic's, Meta's, nor Amazon's. ([Amazon Bedrock data privacy](https://aws.amazon.com/bedrock/faqs/))
2. **Your data is not shared with model providers.** AWS operates the API layer; model providers do not receive your data.
3. **Encryption in transit and at rest.** All data uses TLS 1.2+ in transit. At-rest encryption uses **AWS KMS** — you can use AWS-managed keys or your own customer-managed keys (CMK).
4. **Fine-tuning on private copies.** When you fine-tune a model on Bedrock, AWS creates a private copy of the model for your account. No weight sharing.
5. **VPC support.** You can access Bedrock through an **AWS PrivateLink** endpoint — traffic never traverses the public internet.

> **🎯 On the exam:** "Legal asks whether prompts sent to Claude on Bedrock are used by Anthropic to improve the model" → **No; Bedrock's privacy guarantee explicitly prevents this.** Trap: confusing direct API use (data goes to Anthropic) with Bedrock use (data stays within AWS).

---

### 5.2 Data Residency and Region Selection

> **Why (the rationale):** GDPR, sovereign cloud mandates, and financial regulators require that data not cross specific borders. Deploying in the wrong region — or enabling cross-region inference without understanding its implications — creates regulatory exposure even if the application otherwise functions correctly.
> **When to use:** Define residency requirements before selecting the Bedrock region. For EU/GDPR: deploy in `eu-west-1`, `eu-central-1`, or `eu-west-3` and disable cross-region inference. For healthcare/HIPAA: verify the chosen region is HIPAA-eligible and that AWS has signed your BAA.
> **Nuances & gotchas:** **Cross-region inference routes data to other regions** — enabling it for performance while claiming strict GDPR compliance is contradictory. Not every FM is available in every region; verify your chosen model exists in the target region before committing to the architecture. Knowledge Base vector stores, embeddings, and raw text reside in whichever region the Knowledge Base was created in — you cannot split the store across regions.

**Data residency** means ensuring that data — including vectors, model inputs/outputs, and fine-tuning datasets — is processed and stored only in specific geographic regions, often to satisfy GDPR, CCPA, financial regulations, or government data sovereignty requirements.

**Key Bedrock data-residency considerations:**

| Feature | Residency Implication |
|---|---|
| **Standard Bedrock invocation** | Data processed in the region where you make the API call; stored in that region |
| **Cross-Region Inference** | Routes traffic to additional regions for capacity; data may leave your primary region — **verify before enabling on sensitive workloads** |
| **Bedrock Knowledge Bases** | Vector store, chunked text, and embeddings reside in the region you create the Knowledge Base in |
| **Model fine-tuning data (S3)** | S3 bucket must be in the **same AWS account** as the Knowledge Base; configure S3 bucket region accordingly |

**Region selection strategy:**
- Choose the AWS region that is both **closest to users** (latency) and **compliant with data residency law** (legal constraint).
- For EU data: deploy in `eu-west-1`, `eu-central-1`, or `eu-west-3` — ensure the FM you need is available in that region (not all models are available in all regions).
- For healthcare (HIPAA): AWS signs a Business Associate Agreement (BAA) covering Bedrock. Check scope exclusions — Bedrock Marketplace (third-party models) may be excluded.

> **🎯 On the exam:** "GDPR requires data stays in the EU" → deploy Bedrock in an EU region; **disable cross-region inference**; verify your chosen FM is available in that region. Trap: enabling cross-region inference for performance while also claiming GDPR compliance.

---

### 5.3 Data Classification and Retention

**Data classification for GenAI:**

| Classification Tier | Examples | Controls |
|---|---|---|
| **Public** | Marketing copy, published docs | No restriction; safe to ingest |
| **Internal / Sensitive** | Employee handbooks, internal processes | Access controls; Macie scanning |
| **Confidential / PII** | HR records, customer PII, financial data | Redact before ingestion; separate vector index; audit trail |
| **Restricted / Regulated** | Health data (HIPAA), financial data (PCI-DSS) | Data processed only in BAA-covered services; encryption with CMK; strict access logging |

**Retention:**
- Bedrock does **not** persist prompts or completions by default — they are transient. Model Invocation Logging must be explicitly enabled to capture them.
- **Model Invocation Logging** (optional): Logs all API requests/responses to S3 and/or CloudWatch Logs — useful for audit but must be governed as it contains prompt+response data.
- **Bedrock Knowledge Bases** data (vectors, raw text in vector store) is retained as long as the vector store exists. You control deletion via data source sync (with deletions) or by deleting the knowledge base.
- Apply **S3 Lifecycle policies** to raw ingestion buckets to auto-delete or transition to Glacier after a retention period.

---

### 5.4 Compliance Certifications

Amazon Bedrock is in scope for the following compliance programs ([AWS Compliance Programs](https://aws.amazon.com/compliance/services-in-scope/)):

| Certification | Status | Notes |
|---|---|---|
| **SOC 1 / 2 / 3** | In scope | Standard AWS audit programs |
| **ISO/IEC 27001:2022** | In scope | Excludes Bedrock Marketplace |
| **HIPAA** | BAA available | AWS signs BAA; cross-region inference and Marketplace may be excluded |
| **GDPR** | Compliant framework | Data Processing Addendum (DPA) available; deploy in EU region |
| **PCI-DSS** | Eligible | Cardholder data must stay in compliant architecture |
| **FedRAMP** | In progress / select regions | Check current scope for government workloads |

**Relevant supporting services:**
- **AWS CloudTrail**: Logs every Bedrock API call (who called what, when, from where) — mandatory for compliance audits.
- **AWS Config**: Tracks resource configuration changes (e.g., "was KMS encryption enabled on this Knowledge Base?").
- **AWS KMS**: Provides encryption key management; use CMKs for regulated data to maintain key rotation control.
- **AWS IAM**: Fine-grained access control to Bedrock models, Knowledge Bases, and fine-tuned model versions.

> **🎯 On the exam:** "Auditors require proof that nobody called the Bedrock API without authorization" → **AWS CloudTrail** (logs all API calls). "Healthcare startup needs to use Bedrock for patient data analysis" → ensure AWS **BAA is signed**, use **KMS CMK encryption**, deploy in a **HIPAA-eligible region**, and avoid cross-region inference.

---

## Cross-Domain Links

Domain 1 concepts connect to all other AIP-C01 domains:

- **Security guardrails for RAG outputs** → see [`../services/security-and-governance.md`](../services/security-and-governance.md) — Bedrock Guardrails, IAM policies, VPC endpoints.
- **Bedrock service deep-dive** (agents, model customization, evaluation) → see [`../services/bedrock.md`](../services/bedrock.md)
- **Cost optimization for high-volume RAG** → see cost domain notes; key levers: batch inference, prompt caching, lower embedding dimensions, S3 Vectors for cold storage.
- **Data analytics and Redshift integration** → see [`../services/data-and-analytics.md`](../services/data-and-analytics.md) — Glue, Redshift, Athena, Lake Formation.

---

## Glossary

| Term | Simple Explanation | Purpose in AIP-C01 |
|---|---|---|
| **Amazon Bedrock** | Managed, serverless API for 100+ foundation models | Primary FM access layer; no infrastructure management |
| **Foundation Model (FM)** | A large pre-trained model that can be adapted to many tasks | The AI engine powering GenAI applications |
| **On-Demand Inference** | Pay per token; no reserved capacity | Default; variable workloads |
| **Provisioned Throughput** | Reserved model capacity billed hourly | Predictable high-volume production |
| **Batch Inference** | Asynchronous; ~50% cheaper | Large offline tasks, no latency SLA |
| **Prompt Caching** | Cache system prompts to avoid re-processing | Major cost saver for repeated large system prompts |
| **Intelligent Prompt Routing** | Auto-selects cheapest model that meets quality target | Optimize cost/quality without manual routing logic |
| **RAG (Retrieval-Augmented Generation)** | Retrieve relevant docs → inject as context → FM generates grounded answer | Grounds FM answers in your private data without retraining |
| **Chunking** | Splitting documents into smaller units before embedding | Determines retrieval granularity and context richness |
| **Fixed-size Chunking** | Split every N tokens with overlap | Simple, fast, may break semantic units |
| **Semantic Chunking** | Split where embedding similarity drops | Produces coherent topical chunks |
| **Hierarchical (Parent-Child) Chunking** | Small precise child + large context-rich parent | Best of precision + context; doubles storage |
| **Embedding** | Dense numeric vector encoding semantic meaning | Enables similarity search in vector stores |
| **Amazon Titan Text Embeddings V2** | AWS-native embedding model; 256/512/1024 dims configurable | Default embedding model in Bedrock Knowledge Bases |
| **Cohere Embed v3** | Multilingual embedding model; strong in non-English | Multilingual RAG workloads |
| **Vector Store** | Database optimized for similarity search on embeddings | Stores and retrieves chunks by semantic similarity |
| **Amazon OpenSearch Serverless** | Managed, auto-scaling vector search; default KB store | Default Bedrock Knowledge Bases vector store; hybrid search; binary vectors |
| **Amazon Aurora PostgreSQL / pgvector** | Relational DB + vector extension | SQL + vector in same query; existing PostgreSQL teams |
| **Amazon Neptune Analytics** | Graph analytics + native vector search | GraphRAG; knowledge graphs; relationship traversal |
| **Amazon MemoryDB** | In-memory, Redis-compatible vector DB; sub-ms latency | Real-time applications needing fastest search |
| **Amazon S3 Vectors** | Object storage with native vector capabilities; 90% cheaper | Cheapest large-scale vector storage; billions of vectors; infrequent access |
| **Pinecone / Redis Enterprise / MongoDB Atlas** | Third-party managed vector stores | Teams with existing investment in these platforms |
| **Top-k Retrieval** | Return k nearest vectors to a query embedding | Basic semantic retrieval |
| **Hybrid Search** | Combine vector search + keyword (BM25) search | Improves exact-match and ID-based lookups |
| **Reranking** | Cross-encoder model re-scores candidates against query | Improves top-N precision after broad retrieval |
| **Amazon Rerank 1.0** | AWS-native reranker | Cross-encoder reranking in Bedrock |
| **Cohere Rerank 3.5** | Cohere's reranker | Reranking option in Bedrock |
| **Query Decomposition** | Break multi-part query into sub-queries, retrieve for each | Multi-part question handling |
| **Metadata Filtering** | Constrain search to documents matching attribute criteria | Tenant isolation, date scoping, category filtering |
| **Bedrock Knowledge Bases** | Managed, end-to-end RAG service | No-ops RAG; handles ingestion → embedding → retrieval |
| **NL2SQL (Structured Data Retrieval)** | Natural language → SQL → Redshift/Lakehouse query | Chat with structured/tabular data |
| **SageMaker JumpStart** | ML platform for deploying and fine-tuning open-weight FMs | Custom model training; ML engineer audience |
| **Amazon Textract** | ML-powered document text/table/form extraction | Parse scanned PDFs and images before RAG ingestion |
| **Amazon Comprehend** | NLP service; PII detection, entity extraction, sentiment | Detect and redact PII before embedding |
| **Amazon Macie** | ML-powered S3 data classification for sensitive data | Classify S3 objects; alert on PII presence |
| **Bedrock Guardrails** | Policy-based filter on FM inputs and outputs | Mask PII, block topics, prevent harmful outputs at inference |
| **AWS Step Functions** | Serverless workflow orchestration | Orchestrate multi-step ingestion pipelines with error handling |
| **AWS Lambda** | Serverless function execution | Event-driven trigger; short pre-processing steps |
| **Data Residency** | Ensuring data is processed/stored only in specified regions | GDPR, sovereignty, healthcare regulations |
| **Cross-Region Inference** | Bedrock routes traffic to other regions for capacity | Risk: data may leave primary region; disable for strict residency |
| **Model Invocation Logging** | Logs all Bedrock API calls (prompt + response) to S3/CloudWatch | Audit trail; compliance evidence |
| **AWS KMS (CMK)** | Customer-managed encryption keys | Encrypt vector stores, S3 data, fine-tuning data at rest |
| **AWS CloudTrail** | API call audit log | Prove who called what Bedrock API, when |
| **Binary Vectors** | Bit-packed embeddings vs float32 | ~32× storage reduction; only on OpenSearch |
| **Context Window** | Max tokens in a single FM call (input + output) | Determines how much retrieved context can be injected |
| **Prompt Caching** | Cache repeated large system prompts | Reduces cost for multi-turn or multi-user apps with shared prompts |
| **AWS Well-Architected Generative AI Lens** | Best-practice review framework for GenAI architectures (via AWS WA Tool) | Standardise and audit GenAI designs before production |
| **AWS AppConfig** | Runtime configuration service; externalise model ID / routing config | Dynamic model selection without code redeployment |
| **LoRA / Adapters** | Parameter-efficient fine-tuning — only small adapter weights trained | Cheaper/faster domain fine-tuning; managed via SageMaker Model Registry |
| **OpenSearch Neural plugin** | OpenSearch plugin that auto-generates embeddings via Bedrock at ingest and query time | Self-contained vector search — no separate embedding Lambda needed |

---

## References

- [AIP-C01 Official Exam Guide (AWS)](https://docs.aws.amazon.com/aws-certification/latest/examguides/ai-professional-01.html)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html)
- [Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
- [Amazon Bedrock Knowledge Bases – Vector Store Setup](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html)
- [Bedrock Knowledge Bases – Chunking Strategies](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)
- [AWS Prescriptive Guidance: Choosing a Vector Database for RAG](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html)
- [Amazon S3 Vectors Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html)
- [Amazon Neptune Analytics Documentation](https://docs.aws.amazon.com/neptune-analytics/latest/userguide/what-is-neptune-analytics.html)
- [Amazon MemoryDB Vector Search](https://docs.aws.amazon.com/memorydb/latest/devguide/vector-search.html)
- [Aurora PostgreSQL as Bedrock Knowledge Base Vector Store](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.VectorDB.html)
- [Bedrock Knowledge Bases – Additional Data Connectors](https://aws.amazon.com/blogs/aws/knowledge-bases-for-amazon-bedrock-now-supports-additional-data-connectors-in-preview/)
- [Bedrock Knowledge Bases – Structured Data Retrieval (NL2SQL)](https://aws.amazon.com/about-aws/whats-new/2024/12/amazon-bedrock-knowledge-bases-structured-data-retrieval)
- [AWS Blog: Structured Data Retrieval via Bedrock](https://aws.amazon.com/blogs/machine-learning/build-conversational-interfaces-for-structured-data-using-amazon-bedrock-knowledge-bases/)
- [AWS Decision Guide: Bedrock vs SageMaker](https://docs.aws.amazon.com/pdfs/decision-guides/latest/bedrock-or-sagemaker/bedrock-or-sagemaker.pdf)
- [Bedrock Knowledge Bases – Retrieval Quality Engineering (Chunking, Hybrid Search, Reranking)](https://hidekazu-konishi.com/entry/amazon_bedrock_knowledge_bases_retrieval_quality_engineering.html)
- [PII Detection and Redaction Patterns for GenAI on AWS](https://hidekazu-konishi.com/entry/pii_detection_and_redaction_patterns_for_generative_ai_on_aws.html)
- [AWS Blog: Simplify Model Selection with Model Profiler](https://aws.amazon.com/blogs/machine-learning/simplify-model-selection-in-amazon-bedrock-with-the-open-source-model-profiler/)
- [Amazon Bedrock OpenSearch Managed Cluster Support](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-opensearch-cluster-vector-storage)
- [AWS Compliance Services in Scope](https://aws.amazon.com/compliance/services-in-scope/)
- [Amazon Bedrock FAQs (Data Privacy)](https://aws.amazon.com/bedrock/faqs/)
- [Bedrock Advanced RAG with Terraform: Chunking, Hybrid Search, Reranking](https://medium.com/@suhasmallesh/bedrock-knowledge-base-advanced-rag-with-terraform-chunking-hybrid-search-and-reranking-02b15c5bc763)
- [AWS Well-Architected Generative AI Lens — user guide](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/genai-lens.html)
- [AWS AppConfig — user guide](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html)
- [OpenSearch neural search plugin — docs](https://opensearch.org/docs/latest/search-plugins/neural-search/)
- [SageMaker Model Registry — user guide](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html)

---

*Part of the [AWS AI Certification course](../README.md) · AIP-C01 track.*
