# Graph RAG over the BIAN Knowledge Corpus

> **My project.** Designed a **Graph RAG** system over the **BIAN** (Banking Industry Architecture Network) corpus — ingesting **Service Domain** definitions, the **BOM** (Business Object Model), and **CR-BQ** (Control Record / Behavior Qualifier) structure into a **property graph** — enabling precise architecture-knowledge retrieval that helped **secure a client contract for BIAN-compliant solution design.**

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Architecture](#what-i-built--architecture)
- [Deep Dive 1 — BIAN Terminology (Get This Exactly Right)](#deep-dive-1--bian-terminology-get-this-exactly-right)
- [BIAN in Plain English — Why It's All About Relationships (Worked Example)](#bian-in-plain-english--why-its-all-about-relationships-worked-example)
- [Deep Dive 2 — Graph RAG vs Vector RAG](#deep-dive-2--graph-rag-vs-vector-rag)
- [Deep Dive 3 — Property Graph (and Why Not RDF)](#deep-dive-3--property-graph-and-why-not-rdf)
- [Deep Dive 4 — Modeling BIAN as a Property Graph](#deep-dive-4--modeling-bian-as-a-property-graph)
- [Deep Dive 5 — Retrieval & Traversal at Query Time](#deep-dive-5--retrieval--traversal-at-query-time)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** BIAN is a **semantic, service-oriented reference architecture** for banking — its whole value is in the **relationships** between building blocks (which service domains depend on which, what business objects they exchange, how control records break into behavior qualifiers). A naive vector RAG over BIAN docs retrieves *isolated similar paragraphs* and **can't answer relationship questions** like "what does this service domain depend on?" — which is exactly what architects ask when designing a BIAN-compliant solution.

**Task.** Build a retrieval system that answers **architecture/dependency questions precisely and auditably**, so it could support consulting work pitched to a client.

**Action.** I modeled BIAN as a **property graph** — Service Domains, Control Records, Behavior Qualifiers, and BOM business objects as **nodes**, and their containment / dependency / ownership relationships as **typed edges** — ingesting the official structure deterministically and using LLM extraction only for the looser prose. At query time, retrieval does **hybrid vector-then-traverse**: find entry-point nodes by similarity, then **traverse the graph** to assemble the relational context an answer needs, returning the traversed subgraph as **provenance**.

**Result.** Precise, explainable architecture-knowledge retrieval over BIAN — relationship queries that vector similarity alone can't reconstruct — which **directly contributed to securing a client contract** for BIAN-compliant solution design.

---

## What I Built — Architecture

```
 BIAN corpus
  ├─ Service Domain definitions ─┐
  ├─ BOM (Business Object Model) ─┼─► INGEST ─► PROPERTY GRAPH (Neo4j)
  └─ CR-BQ structure ────────────┘            nodes: ServiceDomain, ControlRecord,
                                              BehaviorQualifier, BusinessObject, ...
                                              edges: CONTAINS, HAS_CONTROL_RECORD,
                                                     HAS_BQ, OWNS, DEPENDS_ON, ...
        query ──► vector/full-text find entry nodes ──► TRAVERSE subgraph
                       │                                       │
                       └──────────► relational context ───────┘ ──► LLM ──► answer + provenance subgraph
```

---

## Deep Dive 1 — BIAN Terminology (Get This Exactly Right)

Nailing these definitions is what makes you credible in a BIAN interview.

**Hierarchy to memorize:** *Business Area → Business Domain → Service Domain → (Control Record → Behavior Qualifiers); BOM = the shared model of objects exchanged; Semantic API = the Service Domain's operations.*

| Term | Precise meaning |
|---|---|
| **Service Domain** | The **elemental building block** — the finest partition: a unique, discrete business capability that *"implements one pattern of behavior to instances of one type of business asset"* (one functional pattern × one asset type). In DDD terms it maps to an **aggregate** / one microservice. |
| **Service Landscape** | The classification hierarchy that organizes all Service Domains: **Business Area → Business Domain → Service Domain** (current versions up to v12). |
| **Control Record (CR)** | The **aggregate root** of a Service Domain — the primary business object it manages start-to-finish. Composed of an **asset type + generic artifact** that characterizes the functional pattern. E.g., asset *Current Account* + pattern *Fulfill* → CR = *"Current Account Fulfillment Arrangement."* |
| **Behavior Qualifier (BQ)** | A breakdown of the Control Record into **finer-grained sub-capabilities** — sub-aggregates owned by the CR (modifiable **only through the CR context**, which enforces consistency). E.g., Mortgage Loan CR → BQs: Billing, Collateral, Fees, Payments, Withdrawals. The CR↔BQ hierarchy literally shapes the API path: `/ServiceDomain/{cr}/BehaviorQualifier/{bq}/Operation`. |
| **BOM (Business Object Model)** | A **conceptual data model** capturing a shared business vocabulary — consistent definitions of Control-Record content and of the information **exchanged between Service Domains**. Each business object is **owned by exactly one Service Domain**. (Standardizes the *objects*, where ISO 20022 standardizes *messages*.) |
| **Semantic API** | The collection of **service operations** a Service Domain offers (typically REST), derived from BOM + domain scope + Control Record. |

So **"CR-BQ"** = the Control-Record → Behavior-Qualifier composition of a Service Domain — a natural **parent→child edge set** in a graph.

Sources: [BIAN — Service Landscape](https://bian.org/deliverables/service-landscape/) · [BIAN Semantic API Practitioner Guide V8.1 (PDF)](https://bian.org/wp-content/uploads/2024/12/BIAN-Semantic-API-Pactitioner-Guide-V8.1-FINAL.pdf) · [Open Group ArchiMate-BIAN — Service Landscape](https://digital-portfolio.opengroup.org/archimate-bian/latest/01-doc/chap03.html)

---

## BIAN in Plain English — Why It's All About Relationships (Worked Example)

**What BIAN is, plainly:** think of BIAN as a **standard set of Lego bricks for a bank**. Instead of every bank inventing its own way to describe "process a payment" or "open an account," BIAN pre-defines ~300+ reusable capabilities (**Service Domains**) with agreed names, responsibilities, and the data they exchange. Build your bank's software from these bricks and different systems (and vendors) can snap together.

**But the value isn't the bricks — it's how they *connect*.** A single Service Domain does almost nothing alone. Real banking processes are **choreographies**: one capability calls another, which needs a third, which reads data owned by a fourth. That web of **dependencies and data exchange** *is* the architecture. So the questions architects actually ask are **relationship questions**, and answering them means **walking the connections** — not finding a paragraph that "sounds similar."

### Worked example — "What does *Payment Execution* depend on to move money?"

To execute one outbound payment, the **Payment Execution** Service Domain doesn't act alone — it depends on a chain of other domains:

```
                         ┌─────────────────────┐
                         │  Payment Order       │  captures & validates the instruction
                         └──────────┬──────────┘
                                    │ initiates
                                    ▼
   ┌────────────────┐      ┌─────────────────────┐      ┌────────────────────────┐
   │ Party           │◄─── │  PAYMENT EXECUTION   │ ───► │ Fraud / Financial Crime │
   │ Authentication  │auth │  (the domain asked   │check │ (screen the payment)    │
   └────────────────┘      │   about)             │      └────────────────────────┘
                           └───────┬─────┬────────┘
                        debits     │     │  routes via
                                   ▼     ▼
                    ┌────────────────┐  ┌──────────────────────────┐
                    │ Current Account │  │ Correspondent Bank /      │
                    │ (funds check +  │  │ Payment Rails (ACH/SWIFT) │
                    │  debit)         │  │ (settle externally)       │
                    └────────────────┘  └──────────────────────────┘
```

**Why vector RAG fails this question:** the fact "Payment Execution depends on Current Account for the debit" is **not written in any single paragraph** — it lives in the *edges* between domains, spread across many documents (service definitions, business scenarios, BOM data-exchange specs). Embedding-similarity retrieval pulls back chunks that *mention* "payment," but it can't **assemble the dependency chain** or guarantee it's complete. Ask "what breaks if Current Account is unavailable?" and similarity search has nothing to traverse.

**Why the graph nails it:** model each domain as a **node** and each dependency as a typed **edge** (`DEPENDS_ON`, `DEBITS`, `ROUTES_VIA`), and the question becomes a one-hop (or multi-hop) **traversal**:

```cypher
MATCH (:ServiceDomain {name:"Payment Execution"})-[:DEPENDS_ON]->(d)
RETURN d          // → Payment Order, Party Authentication, Fraud, Current Account, Correspondent Bank
```

The same structure answers the deeper ones vector search can't: *"trace every domain touched by a cross-border transfer"* (multi-hop path), *"which domains own the objects exchanged in this payment?"* (follow `OWNS` edges to BOM entities), *"what's the blast radius if this domain changes?"* (reverse-traverse `DEPENDS_ON`). And because you return the **traversed subgraph as provenance**, the answer is auditable — essential when you're defending a "BIAN-compliant" design to a client.

> **The one-liner:** *BIAN is a graph of interdependent capabilities. Architecture questions are dependency-traversal questions. Vector similarity retrieves lookalike text; only a graph can walk the edges — completely and explainably.*

---

## Deep Dive 2 — Graph RAG vs Vector RAG

> **Why (the rationale):** BIAN's value is its *relational structure* — which domains depend on which, what objects they exchange, how CRs compose BQs. Vector similarity retrieves chunks that "sound like" the query but cannot reconstruct multi-hop chains or guarantee completeness of a dependency set. A knowledge graph makes relationships first-class and traversable: answering "what depends on X" is a one-hop Cypher query, not a hope that the answer is contained in a single similar paragraph.
> **When to use:** Graph RAG is the right choice when the corpus has rich, structured relational content and the key questions are multi-hop or dependency-traversal in nature (architecture, ontology, knowledge-graph Q&A). For corpora of independent prose documents where questions are about content similarity rather than relationships, vector RAG is simpler and usually sufficient.
> **Nuances & gotchas:** Graph construction is the bottleneck — LLM-based entity/relationship extraction is noisy and expensive; deterministic parsing is only possible when the corpus has well-structured source artifacts. Graph quality directly limits retrieval quality: a missing or wrong edge means the traversal produces an incomplete or incorrect answer with no fallback. Text-to-Cypher generation by an LLM can produce syntactically valid but semantically wrong queries, silently returning wrong subgraphs.

- **Naive/vector RAG:** chunk → embed → retrieve top-k similar chunks → stuff into the LLM. Great for "find a passage like my question," but retrieves **isolated chunks** with **no model of relationships**.
- **Graph RAG:** build a **knowledge graph** (entities = nodes, relationships = edges), then retrieve by **traversing** the graph (often plus vector search) — supplying **multi-hop, relational context** similarity search can't reconstruct.

**The key differentiator:** vector RAG fails on **global / connect-the-dots** questions ("What depends on X?", "What are the themes across the corpus?") because the answer isn't in any single similar chunk — it's distributed across many entities and their links.

**Two families (don't conflate):**
- **Microsoft GraphRAG** — LLM extracts a graph from *unstructured* text, then **Leiden community detection + LLM community summaries**, with **global** (map-reduce over communities) and **local** (fan-out from an entity) search. Powerful for thematic Q&A but **costly** and lossy on a corpus that's *already* structured.
- **Property-graph + retrieval (Neo4j / LlamaIndex / LangChain)** — build a labeled property graph and retrieve via **traversal, text-to-Cypher, vector, or hybrids**. This is the right family for a curated, structured corpus like BIAN.

Sources: [Microsoft Research — GraphRAG (arXiv 2404.16130)](https://arxiv.org/abs/2404.16130) · [Neo4j — GraphRAG](https://neo4j.com/labs/genai-ecosystem/graphrag/)

---

## Deep Dive 3 — Property Graph (and Why Not RDF)

> **Why (the rationale):** BIAN edges carry meaningful properties (functional pattern, asset type, API verb, landscape version) that need to be stored directly on the edge. RDF triples have no native edge properties — you'd need to *reify* each edge into an extra node, adding complexity and making traversal queries more verbose. LPG's Cypher query language and native graph algorithms (path finding, community detection) also pair naturally with the LLM tooling ecosystem (LlamaIndex, LangChain, Neo4j GraphRAG).
> **When to use:** Property graphs (Neo4j, TigerGraph) for traversal-heavy retrieval where edge attributes matter and dev ergonomics / LLM tooling integration are important. RDF/triples for linked-data, formal-ontology, or W3C standards compliance use cases (OWL reasoning, SHACL validation).
> **Nuances & gotchas:** Neo4j's native Cypher is not SQL and has its own learning curve; generated Cypher from an LLM can use nonexistent node labels or relationship types. Property graphs sacrifice the formal inference capabilities of OWL/SHACL — you can't automatically derive new facts through entailment. Graph schema migrations (adding a new node type or relationship) require careful versioning, especially when tied to a specific BIAN Service Landscape release.

A **Labeled Property Graph (LPG)** has **nodes** (with type labels) and **directed, typed edges**, where **both nodes and edges carry key–value properties**. (Neo4j's native model.)

| | **Property graph (LPG)** | **RDF (triples)** |
|---|---|---|
| Data unit | nodes + edges, both with properties | subject–predicate–object triples |
| Edge properties | **yes, directly** | no (must *reify* into an extra node) |
| Strength | dev ergonomics, edge attributes, **fast multi-hop traversal**, graph algorithms | W3C standards, linked data, formal semantics (OWL/SHACL) |

For a **traversal-heavy retrieval** workload — "walk from this domain to its dependencies / its CR's BQs / the domains that own these BOM objects" — the **property graph** wins on ergonomics and traversal speed, and pairs cleanly with LLM tooling (Neo4j GraphRAG, LlamaIndex Property Graph Index). RDF would force reification for edge attributes and is aimed at formal-semantics/linked-data use cases.

Sources: [LlamaIndex — Property Graph Index](https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/) · [RDF vs Property Graph — DZone](https://dzone.com/articles/rdf-triple-stores-vs-labeled-property-graphs-whats)

---

## Deep Dive 4 — Modeling BIAN as a Property Graph

> **Why (the rationale):** BIAN's official artifacts (Service Landscape, CR-BQ hierarchy, BOM) are semi-structured and deterministic — they map cleanly to node types and edge types without needing LLM inference. Keeping structural edges authoritative (not LLM-guessed) is critical: an LLM hallucinating a `DEPENDS_ON` edge would silently make the retrieval wrong, undermining the "BIAN-compliant" claim. LLM extraction is reserved only for looser prose (definitions, business scenarios) where deterministic parsing fails.
> **When to use:** Hybrid construction (deterministic parsing for structured content, LLM extraction for unstructured prose) is the right approach for any corpus that has both: structured artifacts with high precision and unstructured documentation with useful relational content. Pure LLM extraction works for fully unstructured corpora; pure deterministic parsing fails when the corpus includes natural-language content.
> **Nuances & gotchas:** The BIAN Service Landscape changes with each version (currently up to v12) — the graph must be versioned and re-ingested when BIAN releases a new version. Cross-domain `DEPENDS_ON` edges come from business scenarios and BOM exchange specs, which are the loosest artifacts; these edges are the most likely to be wrong or incomplete and carry the most retrieval risk.

The ingested inputs map directly onto graph structure:

**Nodes:** `BusinessArea`, `BusinessDomain`, `ServiceDomain`, `ControlRecord`, `BehaviorQualifier`, `BusinessObject` (BOM entity), `SemanticAPI`/`ServiceOperation`, `BusinessScenario`.

**Edges:**
```
 (BusinessArea)-[:CONTAINS]->(BusinessDomain)-[:CONTAINS]->(ServiceDomain)   ← Service Landscape
 (ServiceDomain)-[:HAS_CONTROL_RECORD]->(ControlRecord)-[:HAS_BQ]->(BehaviorQualifier)  ← CR-BQ structure
 (ServiceDomain)-[:OWNS]->(BusinessObject)   (ControlRecord)-[:REFERENCES]->(BusinessObject)   ← BOM
 (ServiceDomain)-[:EXPOSES]->(SemanticAPI)-[:HAS_OPERATION]->(ServiceOperation)
 (ServiceDomain)-[:DEPENDS_ON / :EXCHANGES_WITH]->(ServiceDomain)   ← cross-domain (from scenarios/BOM)
 (BusinessScenario)-[:USES]->(ServiceDomain)
```
Properties carry definitions, functional pattern, asset type, generic artifact, API verbs, landscape version, etc.

**Construction was hybrid:** the official BIAN artifacts are semi-structured, so I mapped them **deterministically** (landscape → Area/Domain/ServiceDomain + CONTAINS; CR-BQ → HAS_CONTROL_RECORD/HAS_BQ; BOM → BusinessObject + OWNS/REFERENCES) and used **LLM entity/relationship extraction only for the looser prose** (definitions, business scenarios). This keeps the structural edges **authoritative** (not LLM-guessed) — critical for defending "BIAN-compliant."

---

## Deep Dive 5 — Retrieval & Traversal at Query Time

> **Why (the rationale):** Vector search alone can't answer "what depends on X" — that's a graph traversal. Graph traversal alone requires knowing the exact entry node name — that's vector search's job. The two-step pattern (vector/full-text to find entry points → Cypher to traverse the connected subgraph) combines fuzzy semantic matching with precise structural walk, and returning the traversed subgraph as provenance makes answers auditable rather than opaque.
> **When to use:** Hybrid vector-then-traverse is the right retrieval pattern when questions mix "find the relevant entity" (semantic) with "walk from that entity" (relational). Pure Cypher suffices when entity names are known exactly. Pure vector suffices when questions are about content, not structure.
> **Nuances & gotchas:** The depth of traversal must be bounded (`MATCH (a)-[:REL*1..3]->(b)`) to avoid exponentially large subgraphs for highly connected nodes. For broad "how does this business area hang together" questions, local fan-out traversal generates too much context; community-summary approaches (like MS GraphRAG's global retrieval) are better suited — but this project focused on local/structured queries where traversal excels.

**Hybrid vector-then-traverse:**
1. **Find entry points** — vector/full-text search over node text finds the relevant starting nodes.
2. **Traverse** — a **Vector-Cypher**-style step walks out to the connected subgraph (dependencies, CR→BQ children, owning domains) to assemble relational context.
3. **Generate** — pass that subgraph to the LLM, and **return the traversed path/subgraph as provenance** so the answer is auditable.

Entity-specific questions use **local** fan-out; broad "how does this business area hang together" questions use community-summary/global-style retrieval.

**Concrete queries (text-to-Cypher):**
```cypher
// Dependencies of a service domain
MATCH (sd:ServiceDomain {name:$x})-[:DEPENDS_ON]->(d) RETURN d

// Behavior Qualifiers under a Control Record
MATCH (:ServiceDomain {name:$x})-[:HAS_CONTROL_RECORD]->(cr)-[:HAS_BQ]->(bq)
RETURN cr, bq
```

Sources: [Neo4j — graph traversal GraphRAG](https://neo4j.com/blog/developer/graph-traversal-graphrag-python-package/) · [LlamaIndex — Property Graph Index](https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/)

---

## Interview Q&A

**Q: Why graph RAG instead of vector RAG here?**
BIAN's value is the *relationships* — landscape hierarchy, CR→BQ composition, BOM object ownership, cross-domain dependencies. Architecture questions are multi-hop traversals ("what depends on X", "what BQs are under this CR"). Vector similarity retrieves isolated chunks and can't reconstruct those edges; a property graph makes them first-class and traversable, with explainable provenance.

**Q: How did you construct the graph?**
Hybrid — deterministic mapping of the official structured artifacts (landscape → nodes+CONTAINS; CR-BQ → HAS_CONTROL_RECORD/HAS_BQ; BOM → BusinessObject + OWNS/REFERENCES) plus LLM extraction only for loose prose (definitions, scenarios). Stored as a labeled property graph in Neo4j, with node embeddings for hybrid retrieval.

**Q: How do you retrieve/traverse at query time?**
Vector/full-text finds entry-point nodes; a Vector-Cypher step traverses to the connected subgraph (dependencies, CR→BQ, owning domains); that context goes to the LLM, and I return the subgraph as provenance. Local fan-out for entity questions, global/community-summary for broad ones.

**Q: Property graph vs RDF — why property graph?**
Edge properties, fast multi-hop traversal, Cypher, and LLM tooling ergonomics. RDF would force reification for edge attributes and targets formal-semantics/linked-data, not the traversal-heavy retrieval I needed.

**Q: Microsoft GraphRAG vs your approach?**
MS GraphRAG builds the graph purely by LLM extraction over unstructured text + Leiden community summaries — powerful but costly and lossy on a corpus that's *already* structured. BIAN gives authoritative structure, so a curated property-graph + hybrid retrieval yields higher precision and explainability; community summaries still help for broad "explain this business area" questions.

**Q: How do you keep it BIAN-compliant / trustworthy?**
Deterministic mapping of official artifacts (not LLM-guessed) for structural edges, versioned to a specific Service Landscape release (e.g., v12), and return the traversed subgraph as provenance so answers are auditable.

---

## Honest Caveats

- **"CR-BQ diagram"** isn't confirmed as a single named BIAN deliverable — it denotes the well-documented CR→BQ structure of a Service Domain; describe it as such.
- Some `bian.org` pages render client-side; the CR/BQ/BOM definitions above are corroborated by the **Open Group ArchiMate-BIAN** portfolio and the **BIAN Semantic API Practitioner Guide V8.1**.
- Microsoft's comprehensiveness/cost figures are "reported," not guaranteed for a given corpus.

---

## References

- [BIAN — Service Landscape](https://bian.org/deliverables/service-landscape/) · [BIAN Semantic API Practitioner Guide V8.1 (PDF)](https://bian.org/wp-content/uploads/2024/12/BIAN-Semantic-API-Pactitioner-Guide-V8.1-FINAL.pdf)
- [Open Group — ArchiMate-BIAN Service Landscape (chap 3)](https://digital-portfolio.opengroup.org/archimate-bian/latest/01-doc/chap03.html)
- [Microsoft Research — From Local to Global: A Graph RAG Approach (arXiv 2404.16130)](https://arxiv.org/abs/2404.16130) · [microsoft/graphrag](https://github.com/microsoft/graphrag)
- [Neo4j — GraphRAG](https://neo4j.com/labs/genai-ecosystem/graphrag/) · [Graph traversal GraphRAG](https://neo4j.com/blog/developer/graph-traversal-graphrag-python-package/)
- [LlamaIndex — Property Graph Index](https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **BIAN (Banking Industry Architecture Network)** | A global non-profit that defines a standard service-oriented reference architecture for banking with ~300+ reusable capabilities | Provides a shared vocabulary so banking systems and vendors can interoperate |
| **Service Domain** | BIAN's finest-grained building block — a unique, discrete banking capability that implements one functional pattern on one type of business asset | Maps to one microservice or DDD aggregate in a BIAN-compliant implementation |
| **Service Landscape** | The classification hierarchy that organizes all Service Domains: Business Area → Business Domain → Service Domain | Provides the containment structure modeled as CONTAINS edges in the graph |
| **Control Record (CR)** | The aggregate root of a Service Domain — the primary business object it manages end-to-end | Defines the Service Domain's API path root and enforces consistency for sub-capabilities |
| **Behavior Qualifier (BQ)** | A finer-grained sub-capability owned by a Control Record; modifiable only through the CR context | Shapes the REST API path `/ServiceDomain/{cr}/BehaviorQualifier/{bq}/Operation` |
| **BOM (Business Object Model)** | BIAN's conceptual data model defining the objects exchanged between Service Domains | Standardizes information exchange; each business object is owned by exactly one Service Domain |
| **Semantic API** | The collection of REST service operations a Service Domain exposes, derived from BOM and CR scope | The public interface through which Service Domains communicate |
| **Graph RAG** | A retrieval strategy that builds a knowledge graph and answers questions by traversing edges rather than retrieving similar text chunks | Enables multi-hop, relational answers that vector similarity search cannot reconstruct |
| **Property Graph (LPG)** | A graph model where both nodes and edges carry typed labels and key-value properties | Best fit for traversal-heavy retrieval; edge attributes require no special workaround |
| **RDF (Triples)** | A W3C standard graph format using subject-predicate-object triples without native edge properties | Targets formal semantics and linked data; less ergonomic for traversal-heavy workloads |
| **Neo4j** | A popular native property-graph database with the Cypher query language | Stores the BIAN graph and supports both traversal and vector search in one system |
| **Cypher** | Neo4j's declarative graph query language using `MATCH (a)-[:REL]->(b)` patterns | Used to traverse BIAN dependency chains, CR-BQ structures, and BOM ownership at query time |
| **Vector-Cypher (Hybrid Retrieval)** | Combining vector similarity search to find entry-point nodes with Cypher graph traversal to assemble relational context | Provides both semantic fuzzy matching and exact structural traversal |
| **Node (Graph)** | An entity in the property graph, e.g., a ServiceDomain or BusinessObject, with a type label and properties | Represents a BIAN concept; queried by similarity or exact name |
| **Edge (Graph)** | A directed, typed relationship between two nodes, e.g., `DEPENDS_ON`, `HAS_BQ`, `OWNS` | The primary way relational context is stored; what graph traversal walks |
| **LLM Entity/Relationship Extraction** | Using an LLM to parse unstructured text and produce graph nodes and edges | Supplements deterministic ingestion for looser BIAN prose (definitions, business scenarios) |
| **Microsoft GraphRAG** | A Graph RAG framework from Microsoft that uses LLM extraction, Leiden community detection, and LLM community summaries | Powerful for thematic Q&A over unstructured text; costly and lossy on already-structured corpora |
| **Leiden Community Detection** | A graph algorithm that partitions nodes into densely connected communities | Used by MS GraphRAG to identify topic clusters for global summarization |
| **Provenance (Subgraph)** | The traversed nodes and edges returned alongside an answer to explain where it came from | Makes answers auditable — critical when defending a BIAN-compliant design to a client |
| **DDD Aggregate** | Domain-Driven Design concept for a cluster of related objects managed as a unit by a root entity | Analogy used to explain what a BIAN Service Domain/Control Record represents |
| **Business Scenario** | A BIAN artifact describing an end-to-end banking process involving multiple Service Domains | Source of cross-domain DEPENDS_ON and EXCHANGES_WITH edges in the graph |

---

*Previous: [Distributed ML Pipeline](04-distributed-ml-pipeline-pyspark-ray.md) | Next: [Shipping-Time Forecasting](06-shipping-time-forecasting-catboost.md) | Up: [Guide Home](../README.md)*
