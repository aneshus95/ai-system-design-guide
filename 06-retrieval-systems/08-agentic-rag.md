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

---

## Corrective RAG (CRAG)

CRAG adds a "Reliability Layer" between retrieval and generation.

- **The Logic**: 
  - If retrieval is **Correct**: Direct generation.
  - If retrieval is **Ambiguous**: Use a Web-Search tool to supplement.
  - If retrieval is **Incorrect**: Discard context and use external search or fallback logic.

---

## Multi-Hop Reasoning Loops

For questions like "Who is the CEO of the company that acquired Figma?", the system must:
1. **Hop 1**: Search for "Who acquired Figma?" (Result: Adobe).
2. **Hop 2**: Search for "CEO of Adobe" (Result: Shantanu Narayen).

**Agentic Pattern**: The agent maintains a "State Object" and updates its "Sub-goal" after every retrieval until the chain is complete.

---

## Agentic Filtering and Plan Revision

Modern agents use **Sub-Step Plans**.
- Instead of one big retrieval, the agent writes a plan: "First I will check our internal database for X, then I will look at the public API for Y."
- **Revised planning**: If Step 1 fails, the agent *rewrites* Step 2.

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
