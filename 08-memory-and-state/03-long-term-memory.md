# Long-Term Memory

Long-term memory (L2 & L3) provides persistence across sessions. Production stacks have moved from simple "History RAG" to **Multi-Representation Stores** that combine Vector, Graph, and Relational data. Dedicated memory services (Zep, Mem0, Letta, Cognee) now wrap these stores with conversation summarization, entity extraction, and temporal awareness out of the box.

## Table of Contents

- [Episodic Memory (The Narrative)](#episodic)
- [Semantic Memory (The Knowledge)](#semantic)
- [Hybrid Vector-Graph Storage](#hybrid)
- [Memory Pruning and Decay](#pruning)
- [Privacy and Multi-Tenancy](#privacy)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Episodic Memory: The Personal Log

Episodic memory stores **Trajectories**: sequences of events and their outcomes.
- **Data Structure**: `(Timestamp, Interaction_ID, Trajectory_Summary, Embedding)`.
- **The Rationale**: If an agent successfully built a React component using a specific tool sequence last month, it should "Recall" that success when asked to build another one today.
- **Implementation Note**: We store the *Summary* for retrieval and the *Raw Logs* in cold storage (S3/GCS) for forensic analysis.

> **Why (the rationale):** Raw conversation logs are too verbose to inject wholesale into context, but discarding them entirely loses valuable outcome signals. Storing trajectory summaries with embeddings gives a compact, retrievable representation of what worked and what failed, without the token cost of replaying full sessions.
> **When to use:** Agents that execute multi-step plans (coding, research, workflow automation) where previous successes can serve as templates. Less valuable for simple Q&A agents where each query is independent.
> **Nuances & gotchas:** Episodic memory retrieves by semantic similarity, not by causal relevance — a past trajectory about a React project may be retrieved for a Vue.js question because they are semantically close, even if the specific tool sequence does not apply. Outdated trajectories (based on a deprecated API version) can be retrieved with high confidence and lead the agent astray. Cold-storage raw logs are costly to query at audit time if you have not indexed them by session ID and timestamp.

---

## Semantic Memory: The Fact Store

Semantic memory stores **Discovered Facts** about entities.
- **Entity Identification**: Using a "Fact Extraction Agent" to parse every user turn.
- **Example triplets**:
  - `(User_1, HAS_PREFERENCE, Dark_Mode)`
  - `(Company_X, USES_SDK, Stripe)`
- **Technology**: Knowledge Graphs (Neo4j, AWS Neptune) combined with relational tagging.

> **Why (the rationale):** Vector search over raw conversation logs finds semantically similar text but cannot answer structured questions like "what payment SDK does this company use?" or "what are all the preferences of this user?" Storing distilled triplets enables exact, deterministic lookups that vector search cannot provide.
> **When to use:** When you need the agent to personalize reliably over time (user preferences, domain-specific rules) or when entity relationships matter (org charts, software dependencies). Not necessary for stateless agents or those serving anonymous users.
> **Nuances & gotchas:** The fact extraction agent is a second LLM call per turn — in high-volume systems this adds cost and latency to every interaction. Extraction errors (wrong subject, wrong relation, hallucinated value) silently pollute L3 with bad facts that the main agent will later treat as ground truth. Fact conflicts (user says "I prefer Python" in week 1, "I prefer Go" in week 3) require a conflict resolution policy or you end up with contradictory beliefs in the store.

---

## Hybrid Vector-Graph Storage

Staff-level engineers use **GraphRAG-style Memory**.
- **Vector Search** finds "Related" nodes.
- **Graph Traversal** finds "Connected" nodes.
- **The Win**: If I search for "Project Alpha," vector search finds the name, but graph traversal finds the 10 developers, the deadline, and the linked code repos.

> **Why (the rationale):** Pure vector search returns semantically similar documents but cannot traverse relationships. Pure graph traversal requires you to know the exact starting node. Combining both lets you "land" on the right node via fuzzy search and then "walk" the graph to collect precise relational context.
> **When to use:** When your domain is relationship-rich (enterprise org data, code dependency graphs, knowledge management). Adds meaningful overhead in setup and query complexity — only justified when plain vector retrieval provably misses important connected facts.
> **Nuances & gotchas:** Graph traversal depth must be capped (typically 2-3 hops); unbounded traversal can return thousands of nodes, overwhelming the context window. Maintaining graph consistency when underlying data changes (e.g., a developer leaves the project) requires synchronization logic that is often underestimated. The vector-to-graph pointer linkage can become stale if embeddings are re-indexed without updating graph node IDs.

---

## Memory Pruning and Decay

Memory is a liability if it grows unchecked.
- **Temporal Decay**: Older memories lose their "relevance score" unless frequently accessed.
- **Consolidation**: Merging 10 separate interactions about "billing" into one high-quality summary node.
- **Explicit Forgetting**: Honoring GDPR "Right to be Forgotten" by deleting all episodic and semantic clusters associated with a user ID.

> **Why (the rationale):** An ever-growing memory index degrades retrieval precision (more noise in results), increases storage cost, and can serve stale or outdated information with misleadingly high confidence scores. Pruning and decay are quality maintenance, not just cost control.
> **When to use:** Implement decay from day one in any long-lived agent system. "We'll add pruning later" typically results in a bloated, low-precision index that is expensive to retroactively clean. GDPR deletion must be designed in from the start — retrofitting it across vector DBs, knowledge graphs, and relational tables is extremely difficult.
> **Nuances & gotchas:** Simple frequency-based decay is dangerous — a critical fact mentioned once (allergy, legal constraint) may be evicted alongside irrelevant chatter. Decay policies need importance weighting, not just recency weighting. Consolidation (merging 10 billing interactions into one node) requires careful deduplication logic; a poorly tuned merger can produce a summary that reflects none of the original entries accurately.

---

## Privacy and Multi-Tenancy

> [!CAUTION]
> **Cross-Session Leakage** is the #1 security risk in global memory. 
> Ensure that the `user_id` is a hard partition key in your vector DB metadata. Never rely on the LLM to filter results by user.

---

## Interview Questions

### Q: How do you choose between a Vector DB and a Knowledge Graph for long-term memory?

**Strong answer:**
I use **Vector DBs** for **Episodic Context** (unstructured logs, past conversations) because I need a "Fuzzy" match on meaning. I use **Knowledge Graphs** for **Structural Semantic Knowledge** (relationships, attributes, hierarchies) because I need "Deterministic" traversal. A production system uses a **Hybrid** approach: the vector index points to graph IDs, allowing the system to find the right "Starting Node" and then traverse for high-precision context.

### Q: What is "Catastrophic Forgetting" in the context of learned agentic memory?

**Strong answer:**
In fine-tuned agents, catastrophic forgetting happens when new training data wipes out old knowledge. In **Agentic Memory (RAG-based)**, it refers to **Index Overload**. If an agent adds 1,000 low-quality new "facts" to its memory, the retrieval precision drops, effectively making it "forget" the older, higher-quality facts because they are buried in noise. We mitigate this with **Quality-Weighted Retrieval**: memories with high "Verification Scores" from a supervisor are boosted over raw logs.

---

## References
- Neo4j. "Knowledge Graphs for Generative AI" (2025)
- Pinecone. "The Managed Memory Layer" (2025)
- GraphRAG. "Reasoning over Relationships" (2024/2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Episodic Memory** | Storage of past interaction sequences (trajectories) with timestamps and embeddings | Lets an agent recall what happened in previous sessions, enabling continuity across conversations |
| **Semantic Memory** | Storage of distilled, stable facts and entity relationships rather than raw conversation logs | Provides the persistent knowledge layer the agent uses to personalise responses over time |
| **Trajectory** | An ordered sequence of events, actions, and outcomes from one agent session | Stored in episodic memory so the agent can learn from past successes and failures |
| **Fact Extraction Agent** | A background LLM process that reads conversation turns and pulls out structured facts (triplets) | Converts unstructured dialogue into machine-readable knowledge that can be stored and queried |
| **Knowledge Triplet** | A structured fact in the form (Subject, Relation, Object), e.g., (User_1, HAS_PREFERENCE, Dark_Mode) | The atomic unit stored in a knowledge graph; enables precise, deterministic retrieval |
| **Knowledge Graph** | A graph database (Neo4j, AWS Neptune) where nodes are entities and edges are typed relationships | Enables traversal queries that find connected facts, not just similar ones |
| **Neo4j** | A popular property graph database used to store entity relationships | Commonly used as the L3 semantic memory backend in production agentic systems |
| **AWS Neptune** | Amazon's managed graph database service supporting property graphs and RDF triples | A cloud-native alternative to self-hosted Neo4j for teams on AWS |
| **Vector Database** | A database (Pinecone, Weaviate, Qdrant) that indexes and searches high-dimensional embedding vectors | Used for episodic memory where fuzzy semantic matching on unstructured logs is needed |
| **GraphRAG** | A retrieval pattern that combines vector search (to find a starting node) with graph traversal (to find connected facts) | Delivers richer context than pure vector search by surfacing relationships around the retrieved entity |
| **Hybrid Vector-Graph Storage** | Using a vector index and a knowledge graph together, where the vector search points to graph node IDs | Combines the fuzzy recall of embeddings with the precision of graph traversal |
| **Graph Traversal** | Walking the edges of a knowledge graph to find nodes connected to a starting point | Finds structured relationships (team members, deadlines, repos) that semantic search cannot surface |
| **Temporal Decay** | Gradually reducing the retrieval weight of memories that have not been accessed recently | Prevents stale information from outranking newer, more accurate facts |
| **Memory Consolidation** | Merging multiple related interaction records into a single high-quality summary node | Keeps the memory index lean and improves retrieval precision over time |
| **GDPR Right to be Forgotten** | A legal obligation to permanently delete all personal data about a user upon request | Requires that user_id-based partitioning be enforced so all associated memory clusters can be identified and erased |
| **Multi-Tenancy** | Running one system for many users or organisations while keeping their data strictly isolated | Critical for shared AI services to prevent data from one user appearing in another user's context |
| **Cross-Session Leakage** | A bug where one user's stored memories surface in another user's retrieval results | The top security risk in shared memory systems; prevented by enforcing user_id as a hard partition key |
| **Catastrophic Forgetting** | In RAG-based memory, the effect of flooding the memory index with low-quality entries that bury older high-quality ones | Motivates quality-weighted retrieval and memory pruning strategies |
| **Quality-Weighted Retrieval** | Boosting the rank of memories that have been verified or highly rated, relative to raw unverified logs | Ensures the most reliable facts surface first even in a crowded index |
| **Cold Storage** | Low-cost, high-latency storage (e.g., S3, GCS) used for raw logs that are kept for forensics but not actively retrieved | Separates cheap archival from the hot retrieval path to control costs |
| **Multi-Representation Store** | A memory backend that holds data in multiple formats simultaneously (vector, graph, relational) | Allows different query strategies (fuzzy, traversal, SQL) to be applied to the same underlying information |

*Next: [Agentic Memory with Mem0](04-agentic-memory-mem0.md)*
