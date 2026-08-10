# Agentic RAG

Agentic RAG moves from a "Linear Pipeline" to a **"Reasoning Loop."** Instead of retrieving once, an agent decides *when* and *what* to retrieve to resolve a query. The dominant production patterns are Self-RAG (model emits reflection tokens), Corrective RAG (retrieval evaluator with corrective routing), Adaptive RAG (classifier picks pipeline depth), ReAct over documents, and multi-hop query decomposition. LangGraph is the most common control-flow runtime for stateful loops; LlamaIndex Workflows is common for single-pipeline retrieval-heavy variants.

## Table of Contents

- [Linear vs. Agentic RAG](#comparison)
- [Self-RAG (Self-Reflection)](#self-rag)
- [Corrective RAG (CRAG)](#crag)
- [Multi-Hop Reasoning Loops](#multi-hop)
- [Agentic Filtering and Plan Revision](#planning)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Linear vs. Agentic RAG

| Model | Linear RAG | Agentic RAG |
|-------|------------|-------------|
| **Structure** | Predetermined sequence | Dynamic loop |
| **Self-Correction** | None | High (Can re-retrieve) |
| **Query Complexity**| Simple (1-step) | Hard (Multi-step) |
| **Latency** | Low (Fixed) | Variable (Multiple turns) |

**Principle**: Use Agentic RAG when the query requires "Synthesized Proof" rather than just a "Document Match." Budget for it: a 3-4 iteration loop typically takes 8-12s end-to-end, so route easy queries to a fast path (Adaptive RAG) if your UX needs sub-3s response.

---

## Self-RAG (Self-Reflection)

 popularized in 2024/2025, **Self-RAG** uses "Critic Tokens" to evaluate its own work.

1. **Retrieve**: Model pulls Top-K chunks.
2. **Evaluate**: Is the info relevant? (CRITIC: `Relevant`)
3. **Generate**: Is the answer supported? (CRITIC: `Supported`)
4. **Iterate**: If the answer isn't supported, the model *automatically* triggers a broader search.

> **Why (the rationale):** Standard RAG always retrieves and always generates — even when retrieval is unnecessary (factual queries the model already knows) or when retrieved content is irrelevant. Self-RAG trains the model to make an explicit decision about whether retrieval is needed at all, and to verify its own answer is actually supported by what was retrieved.
> **When to use:** When you want to reduce unnecessary retrieval for queries the model can answer from parametric knowledge, and when faithfulness (answers grounded in retrieved text) is a key requirement. Effective for mixed query workloads where some queries benefit from retrieval and others don't.
> **Nuances & gotchas:** Requires a specially fine-tuned model that has learned the critic-token vocabulary — you cannot use Self-RAG with an off-the-shelf LLM by prompting alone. The training data for critic tokens must be high quality to avoid learned biases (model always says "Supported" regardless of evidence). Adds multiple inference passes per query.

---

## Corrective RAG (CRAG)

CRAG adds a "Reliability Layer" between retrieval and generation.

- **The Logic**: 
  - If retrieval is **Correct**: Direct generation.
  - If retrieval is **Ambiguous**: Use a Web-Search tool to supplement.
  - If retrieval is **Incorrect**: Discard context and use external search or fallback logic.

> **Why (the rationale):** Standard RAG feeds retrieved chunks to the LLM regardless of quality. When retrieval fails (wrong domain, stale data, coverage gap), the LLM hallucinates using bad context as scaffolding. CRAG grades the retrieved context before generation, allowing the system to discard weak results and fall back to better sources rather than confidently generating from garbage.
> **When to use:** When your corpus has known coverage gaps (e.g., real-time events beyond the knowledge cutoff, niche domains), or when retrieval quality is variable and silently bad retrievals produce worse answers than "I don't know." Particularly valuable in customer-facing applications where hallucination risk is high.
> **Nuances & gotchas:** The retrieval evaluator itself can be wrong — if it incorrectly rates good context as "Incorrect" and discards it, the fallback (web search) may produce worse answers than using the original retrieval would have. The evaluator adds an extra inference step. Web-search fallbacks introduce latency and rate limits. CRAG does NOT fix poor precision (retrieved but mis-ranked) — that's a reranker's job.

---

## Multi-Hop Reasoning Loops

For questions like "Who is the CEO of the company that acquired Figma?", the system must:
1. **Hop 1**: Search for "Who acquired Figma?" (Result: Adobe).
2. **Hop 2**: Search for "CEO of Adobe" (Result: Shantanu Narayen).

**Agentic Pattern**: The agent maintains a "State Object" and updates its "Sub-goal" after every retrieval until the chain is complete.

> **Why (the rationale):** A static pipeline must formulate all retrieval queries up front, before any results are known. Multi-hop questions require information discovered in one retrieval step to form the next query — impossible without a loop. The agent's state object carries forward what was learned at each hop to inform the next.
> **When to use:** Questions whose answer depends on a chain of lookups where each step's result determines what to search next. Common in enterprise knowledge bases (org chart traversal), research assistants (citation chains), and any domain where information is spread across related documents with explicit entity references.
> **Nuances & gotchas:** Hop count directly multiplies latency and token cost. Error propagation is a significant risk: a wrong answer in hop 1 causes the hop-2 query to be wrong, compounding the error. Loops must be bounded (max hops, retrieval timeout) to prevent runaway execution. Consider GraphRAG as an alternative for predictable multi-hop patterns — it precomputes the traversal paths rather than discovering them at query time.

---

## Agentic Filtering and Plan Revision

Modern agents use **Sub-Step Plans**.
- Instead of one big retrieval, the agent writes a plan: "First I will check our internal database for X, then I will look at the public API for Y."
- **Revised planning**: If Step 1 fails, the agent *rewrites* Step 2.

> **Why (the rationale):** A flat retrieve-then-answer pipeline treats all information sources as equivalent. Plan-driven retrieval lets the agent reason about *which* source to query first, in what order, and how to adapt if a step returns insufficient information — enabling multi-source, adaptive information gathering.
> **When to use:** When the retrieval strategy itself is non-trivial — e.g., "check internal DB first, then fall back to web, then query SQL if neither has the answer." Most valuable for enterprise assistants with heterogeneous data sources (internal docs, APIs, databases) where query routing matters.
> **Nuances & gotchas:** Plan quality depends heavily on the underlying LLM — weak planners produce bad plans that waste retrieval steps or get stuck in loops. Plans must be constrained to a finite action space (constrained agent frameworks) to prevent the agent from inventing tools or steps that don't exist. Debugging failures is harder than in a fixed pipeline because the execution path varies per query.

---

## Interview Questions

### Q: What is the "Reasoning-Retrieval Balance" in Agentic RAG?

**Strong answer:**
Every "Reasoning turn" in an agentic loop adds token cost and user latency. The goal of a production engineer is to find the "Retrieval Threshold." We use **Token-Budgeting** where we allow the agent only 3-5 "turns" before forcing a final answer. We also use **Speculative Retrieval**—where the agent predicts the next 2 steps it will take and retrieves for both simultaneously to reduce round-trip latency.

### Q: Why does Agentic RAG often lead to higher quality but lower "Reliability" (Determinism)?

**Strong answer:**
Agentic RAG is non-deterministic because the model is "Deciding" its path at every step. A small change in the user query might cause the agent to pick a different tool or search strategy, leading to a different answer format. The standard mitigation is **Constrained Agent Frameworks** (like LangGraph or DSPy) where the "Graph of possible paths" is strictly defined, even if the choice *between* those paths is stochastic.

---

## References
- Asai et al. "Self-RAG: Learning to Retrieve, Generate, and Critique" (2024/2025)
- Yan et al. "Corrective Retrieval Augmented Generation (CRAG)" (2024)
- LangChain. "Agentic RAG with LangGraph" (2025)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Agentic RAG** | A retrieval pattern where a reasoning loop decides when and what to retrieve, iterating until the context is sufficient | Handles complex, multi-step, and ambiguous queries that a fixed linear pipeline cannot resolve |
| **Linear RAG** | A predetermined, fixed-order pipeline: retrieve once, then generate | Simple and fast; appropriate for well-scoped, single-step queries |
| **Reasoning Loop** | A cycle where the model reasons about its current knowledge, decides on a retrieval action, retrieves, and repeats until done | Enables the system to self-correct and gather missing information before generating an answer |
| **Self-RAG** | A variant where the model emits special "critic tokens" to evaluate whether retrieved passages are relevant and whether its answer is supported | Reduces hallucination by letting the model decide when to retrieve and when to trust its context |
| **Critic Token** | A special output token the Self-RAG model generates to signal a self-evaluation decision (e.g., "Relevant", "Supported") | Provides an explicit, inspectable signal for when to re-retrieve or accept the current context |
| **CRAG (Corrective RAG)** | A variant that grades the retrieved context and falls back to web search or query rewriting if the context is judged weak or incorrect | Prevents low-quality context from being passed to the generator |
| **Adaptive RAG** | A strategy that classifies each query's complexity and routes it to the appropriate pipeline depth (simple → fast path; complex → agentic loop) | Avoids paying the latency cost of an agentic loop for queries a simpler pipeline can answer |
| **Multi-Hop Reasoning** | Answering a question that requires chaining two or more separate retrievals — where the answer to step N determines what to search in step N+1 | The primary motivation for agentic loops; linear pipelines cannot discover the intermediate answer needed to form the next query |
| **State Object** | A data structure the agent maintains across loop iterations, tracking retrieved facts, completed sub-goals, and remaining questions | Lets the agent remember what it already knows so it does not repeat retrievals or lose earlier findings |
| **Sub-Goal** | A smaller question the agent breaks the original query into and resolves one at a time | Decomposes an intractable multi-hop question into tractable single-step retrievals |
| **Plan Revision** | The agent's ability to rewrite its remaining steps if an earlier step fails or returns unexpected results | Makes the loop robust to retrieval failures without requiring human intervention |
| **Token Budgeting** | Capping the number of reasoning turns an agentic loop is allowed to take before it must produce a final answer | Prevents runaway loops that accumulate excessive latency and token cost |
| **Speculative Retrieval** | Predicting the next N retrieval steps and issuing them in parallel rather than waiting for each to complete | Reduces round-trip latency in multi-hop loops by overlapping retrieval with reasoning |
| **LangGraph** | A graph-based agent orchestration framework for defining stateful reasoning loops with explicit state and transition logic | The most common control-flow runtime for constrained Agentic RAG in production |
| **DSPy** | A framework for programming (rather than prompting) language model pipelines, with built-in optimization | Provides structured, optimizable agent pipelines that reduce the non-determinism of free-form prompting |
| **LlamaIndex Workflows** | A retrieval-heavy orchestration framework for building multi-step RAG pipelines | Common alternative to LangGraph when the pipeline is retrieval-dominated rather than general-agent-shaped |
| **Constrained Agent Framework** | An agent runtime that defines a fixed graph of possible actions and transitions, even if the choice between them is stochastic | Improves reliability and determinism in Agentic RAG by bounding what the agent can do |
| **Non-Determinism** | The property that the same input may produce different outputs across runs | The key reliability challenge of Agentic RAG; mitigated by constrained frameworks and fixed action graphs |
| **Retrieval Threshold** | The minimum evidence quality required before the agent stops looping and generates a final answer | Balances answer quality against latency and token cost |

*Next: [Advanced Retrieval Patterns](09-advanced-retrieval-patterns.md)*
