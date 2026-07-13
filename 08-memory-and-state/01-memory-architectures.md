# Memory Architectures

LLM memory has evolved from "history buffers" to a **Three-Tiered Cognitive Architecture**. This hierarchy mimics human cognitive systems (L1-L3) to balance speed, cost, and recall capacity. Production agent stacks now lean on dedicated memory layers (Mem0, Zep, Letta, Cognee) on top of vector stores rather than rolling their own.

## Table of Contents

- [The Three-Tiered Hierarchy](#hierarchy)
- [Tier 1: Working Memory (Context)](#tier-1)
- [Tier 2: Episodic Memory (Events)](#tier-2)
- [Tier 3: Semantic Memory (Knowledge)](#tier-3)
- [Memory Consolidation Patterns](#consolidation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Three-Tiered Hierarchy

| Tier | Type | Human Analogy | Technology | Latency |
|------|------|---------------|------------|---------|
| **L1** | Working Memory | Immediate focus | Context Window / KV Cache | <50ms |
| **L2** | Episodic Memory | Past experiences | Vector DB / Local Graph | 100-300ms |
| **L3** | Semantic Memory | General knowledge | Global Graph / SQL / Mem0 | >500ms |

---

## Tier 1: Working Memory (L1)

L1 is the **active focus** of the model. 
- **Context Window**: 128K - 2M tokens on current frontier models (Claude Opus 4.7, Claude Sonnet 4.6, GPT-5.5, Gemini 3.1 Pro).
- **KV Cache**: The GPU "RAM" that stores pre-computed keys and values.
- **Management Strategy**: **Sliding Windows** and **Prefix Caching** (vLLM/PagedAttention).
- **Redundancy Note**: We only keep the most recent turns and critical system instructions in L1.

---

## Tier 2: Episodic Memory (L2)

L2 stores "What happened previously" in this session or past sessions with this user.
- **Storage**: Vector databases (Pinecone, Weaviate, Qdrant).
- **Retrieval**: Semantic search. If the user asks "What did we talk about last Tuesday?", L2 provides the answer.
- **Pattern**: **Experience Replay**. Agents retrieve successful past trajectories to guide current decisions.

---

## Tier 3: Semantic Memory (L3)

L3 stores **Immutable Facts** and **Learned Rules**.
- **Knowledge Graphs**: Store relationships (e.g., `User` -- `WORKS_FOR` --> `Company_X`).
- **Mem0**: A managed service that extract facts (e.g., "User likes Dark Mode") and makes them available globally.
- **Truth Anchoring**: L3 acts as the "Ground Truth" when L1 and L2 provide conflicting information.

---

## Memory Consolidation Patterns

Memories move between tiers via **Consolidation**:
1. **Extraction**: An LLM "Reviewer" extracts facts from L1 at the end of a session.
2. **Indexing**: Facts are stored in L2 (as vectors) and L3 (as graph nodes).
3. **Decay**: Old, non-reinforced memories are moved from L2 to cold storage (L3) or deleted.

---

## Interview Questions

### Q: Why not just use a 2M token context window for all memory (L1-L3)?

**Strong answer:**
While possible, it is **Economically and Cognitively inefficient**. 
1. **Cost**: Calling a model with 1M+ tokens for every turn costs significantly more than a 10K token call with RAG-recalled context.
2. **Attention Dilution**: Even with "Long Context" models, "Lost in the Middle" remains a factor. If the context is cluttered with irrelevant historical turns, the model's reasoning on the *current* task degrades.
3. **Latency**: TTFT (Time to First Token) scales with context size due to KV cache loading. 
A staff-level architecture uses **Strategic Retrieval** to keep the context window lean and focused.

### Q: How do you handle "Privacy Leakage" in Tier 3 (Global Semantic Memory)?

**Strong answer:**
Tier 3 (Semantic Memory) must be **Sharded by Namespace**. Each user or organization gets a unique `namespace_id` in the vector DB and Knowledge Graph. We implement **RLS (Row Level Security)** at the database layer. Additionally, we use a **PII-Scrubbing Layer** during the Consolidation step to ensure that sensitive data (passwords, PII) never moves from the transient L1 context into the persistent L3 knowledge store.

---

## References
- Pack et al. "Generative Agents" (2023/2025 Context)
- OpenAI. "Context Window Optimization" (2025)
- Mem0 Documentation. "Dynamic Memory Management" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Three-Tiered Cognitive Architecture** | A layered memory model (L1, L2, L3) that mirrors how human cognition organises fast recall, recent experiences, and deep knowledge | Provides a framework for designing AI memory systems that balance speed, cost, and depth |
| **L1 (Working Memory)** | The active, in-context "scratch pad" the model reads and writes to during a single inference call | Fastest tier; holds whatever the model is currently reasoning about |
| **L2 (Episodic Memory)** | Storage of past events and interaction histories retrieved via semantic search | Allows an agent to recall what happened in previous sessions |
| **L3 (Semantic Memory)** | Permanent store of facts, rules, and relationships that rarely change | Acts as the ground truth knowledge base across all users and sessions |
| **Context Window** | The maximum number of tokens an LLM can read at one time (e.g., 128K tokens) | Sets the hard upper bound on how much information fits in L1 at once |
| **KV Cache** | Pre-computed key-value pairs stored on GPU memory so repeated tokens are not reprocessed | Reduces latency and compute cost for calls that share a common prefix |
| **Sliding Window** | A context management strategy that retains only the most recent N tokens and drops older ones | Keeps context size constant without summarisation overhead |
| **Prefix Caching** | Reusing the cached KV state for a static prefix (e.g., system prompt) across many requests | Cuts compute cost and latency for repeated prefixes like tool schemas |
| **Vector Database** | A specialised database (Pinecone, Weaviate, Qdrant) that stores and searches high-dimensional embedding vectors | Enables fast fuzzy semantic retrieval for episodic and semantic memories |
| **Semantic Search** | Finding documents or memories by meaning rather than exact keyword match | Lets an agent retrieve relevant history even when wording differs |
| **Experience Replay** | Retrieving past successful trajectories and using them to inform current decisions | Helps agents avoid repeating mistakes and leverage proven strategies |
| **Knowledge Graph** | A graph-based data structure where nodes are entities and edges are relationships (e.g., Neo4j) | Enables traversal of connected facts, useful for structural semantic knowledge |
| **Mem0** | A managed memory service that extracts structured facts from conversations and stores them persistently | Provides a ready-made L3 layer without building custom extraction pipelines |
| **Zep** | A production memory service with built-in temporal awareness for agent conversations | Tracks when facts were learned and prioritises recency |
| **Letta** | An agent memory framework designed for long-running agents with OS-style paging of memory in and out | Handles memory that exceeds what fits in a single context window |
| **Cognee** | A memory service that organises knowledge as a graph-first structure for RAG use cases | Ideal when relationship traversal between facts is more important than fuzzy search |
| **Memory Consolidation** | The process of moving information from transient L1 into persistent L2 or L3 stores | Ensures valuable session content is not lost when the context is cleared |
| **Temporal Decay** | Reducing the relevance score of older memories that have not been accessed recently | Prevents stale data from crowding out fresher, more accurate memories |
| **Namespace Sharding** | Partitioning a vector DB or knowledge graph so each user/org has an isolated shard | Prevents data from one tenant leaking into another tenant's queries |
| **RLS (Row Level Security)** | A database feature that enforces access control at the individual row level | Ensures users can only retrieve their own memory records |
| **PII-Scrubbing Layer** | A processing step that removes personally identifiable information before data is stored long-term | Prevents sensitive data like passwords or health details from persisting in L3 |
| **TTFT (Time to First Token)** | The delay between sending a request and receiving the first output token | A key latency metric; grows with context size because the KV cache must be loaded |
| **Lost in the Middle** | A known LLM failure mode where relevant information buried in the middle of a long context is ignored | Motivates keeping context focused rather than stuffing every available token |
| **RAG (Retrieval-Augmented Generation)** | A pattern where relevant external documents or memories are retrieved and injected into the prompt before generation | Lets the model answer questions using up-to-date or user-specific knowledge it was not trained on |

*Next: [Short-Term Context Management](02-short-term-context.md)*
