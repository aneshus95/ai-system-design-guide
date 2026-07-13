# Graph RAG over the BIAN Knowledge Corpus

> **My project.** Designed a **Graph RAG** system over the **BIAN** (Banking Industry Architecture Network) corpus — ingesting **Service Domain** definitions, the **BOM** (Business Object Model), and **CR-BQ** (Control Record / Behavior Qualifier) structure into a **property graph** — enabling precise architecture-knowledge retrieval that helped **secure a client contract for BIAN-compliant solution design.**

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Architecture](#what-i-built--architecture)
- [Deep Dive 1 — BIAN Terminology (Get This Exactly Right)](#deep-dive-1--bian-terminology-get-this-exactly-right)
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

## Deep Dive 2 — Graph RAG vs Vector RAG

- **Naive/vector RAG:** chunk → embed → retrieve top-k similar chunks → stuff into the LLM. Great for "find a passage like my question," but retrieves **isolated chunks** with **no model of relationships**.
- **Graph RAG:** build a **knowledge graph** (entities = nodes, relationships = edges), then retrieve by **traversing** the graph (often plus vector search) — supplying **multi-hop, relational context** similarity search can't reconstruct.

**The key differentiator:** vector RAG fails on **global / connect-the-dots** questions ("What depends on X?", "What are the themes across the corpus?") because the answer isn't in any single similar chunk — it's distributed across many entities and their links.

**Two families (don't conflate):**
- **Microsoft GraphRAG** — LLM extracts a graph from *unstructured* text, then **Leiden community detection + LLM community summaries**, with **global** (map-reduce over communities) and **local** (fan-out from an entity) search. Powerful for thematic Q&A but **costly** and lossy on a corpus that's *already* structured.
- **Property-graph + retrieval (Neo4j / LlamaIndex / LangChain)** — build a labeled property graph and retrieve via **traversal, text-to-Cypher, vector, or hybrids**. This is the right family for a curated, structured corpus like BIAN.

Sources: [Microsoft Research — GraphRAG (arXiv 2404.16130)](https://arxiv.org/abs/2404.16130) · [Neo4j — GraphRAG](https://neo4j.com/labs/genai-ecosystem/graphrag/)

---

## Deep Dive 3 — Property Graph (and Why Not RDF)

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

*Previous: [Distributed ML Pipeline](04-distributed-ml-pipeline-pyspark-ray.md) | Up: [Guide Home](../README.md)*
