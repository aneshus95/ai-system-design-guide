# LangGraph AI Coding Agent + Azure AI Search RAG

> **My project.** Helped build an AI coding agent on **LangGraph** — a multi-node conversation graph with **dynamic tool calls** — backed by a **vector-store RAG** layer (**Azure AI Search + OpenAI embeddings**) that grounds answers in proprietary documentation to reduce hallucination.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Architecture](#what-i-built--architecture)
- [Deep Dive 1 — LangGraph (Why a Graph, Not a Chain)](#deep-dive-1--langgraph-why-a-graph-not-a-chain)
- [Deep Dive 2 — Dynamic Tool Calls (the ReAct Loop)](#deep-dive-2--dynamic-tool-calls-the-react-loop)
- [Deep Dive 3 — Azure AI Search as the RAG Backend](#deep-dive-3--azure-ai-search-as-the-rag-backend)
- [Deep Dive 4 — OpenAI Embeddings](#deep-dive-4--openai-embeddings)
- [Deep Dive 5 — Grounding & Hallucination Reduction](#deep-dive-5--grounding--hallucination-reduction)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** A general LLM knows nothing about a company's *internal* codebase, APIs, and conventions — so when asked coding questions about proprietary systems, it confidently makes things up. A single linear "retrieve → answer" chain also couldn't handle real developer workflows that need to *look something up, run a step, observe, and decide what to do next.*

**Task.** Build an agent that (a) can **take actions** (read files, search code, retrieve docs) in a loop, and (b) **grounds** its answers in the company's proprietary documentation so it stops hallucinating.

**Action.** We modeled the agent as a **LangGraph state machine** — an LLM reasoning node, a tool node, and a retrieval path wired by **conditional edges** — so the agent can cycle "reason → call a tool → observe → reason again" until done. For grounding, we stood up a RAG backend on **Azure AI Search** (hybrid keyword+vector retrieval with a semantic reranker) over the proprietary docs, embedded with **OpenAI embeddings**, and injected retrieved passages into the prompt with citations.

**Result.** The agent answers questions about internal systems from *actual documentation* instead of parametric guesswork, with source citations that make answers auditable — materially reducing hallucination on proprietary-knowledge queries.

---

## What I Built — Architecture

```
                    ┌─────────── shared STATE (message history + scratch) ───────────┐
                    │                                                                 │
   user turn ──► ┌──────────┐   tool requested?   ┌──────────┐   result   ┌──────────┐
                 │  AGENT   │ ──── yes ─────────►  │  TOOL    │ ─────────► │  AGENT   │ ─► ... ─► answer
                 │ (LLM)    │ ◄──────────────────  │  NODE    │            │ (LLM)    │
                 └────┬─────┘        loop           └──────────┘            └──────────┘
                      │ no tool → END
                      ▼
          one tool = retrieve_docs ──► ┌─────────────────────────────────────┐
                                       │  AZURE AI SEARCH (RAG backend)        │
                                       │  hybrid: BM25 + vector (HNSW)         │
                                       │  → RRF merge → semantic reranker       │
                                       │  docs embedded w/ OpenAI embeddings    │
                                       └─────────────────────────────────────┘
```

---

## Deep Dive 1 — LangGraph (Why a Graph, Not a Chain)

**LangGraph** is a runtime for **stateful, cyclic, multi-step agent workflows** — built on top of LangChain, not a competitor. The key contrast:

- A **linear chain (LCEL)** is a one-pass DAG: `retrieve → augment → generate`, once. It can't loop or branch on runtime state.
- A **LangGraph `StateGraph`** models the app as a **directed graph that can cycle** — enabling retries, branching, and the "call tool → observe → decide → maybe call again" loop an agent needs.

**Core primitives:**
- **State** — a shared `TypedDict` every node reads/writes; nodes return *partial* updates that get merged. The message history typically uses an `add_messages` reducer that **appends** rather than overwrites.
- **Nodes** — Python functions `(state) → partial update` (LLM call, tool run, retrieval, validation…).
- **Edges** — normal (always A→B) or **conditional** (a router function reads state and returns the name of the next node → branching).
- **Checkpointing** — a checkpointer snapshots full state at each step, giving **memory across turns**, **fault-tolerant resume**, and **human-in-the-loop** pause/approve/resume (critical before letting the agent run a destructive tool).

**"Multi-node conversation graph"** = the conversation is specialized nodes (reason / tool / retrieve / validate / maybe human-review) wired by conditional edges that route each turn on state — not one monolithic prompt.

Sources: [LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api) · [langchain.com/blog/langgraph](https://www.langchain.com/blog/langgraph) · [Human-in-the-loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)

---

## Deep Dive 2 — Dynamic Tool Calls (the ReAct Loop)

**The LLM does not execute tools.** Given tool schemas (name, description, JSON args), the model *decides* which tool(s) to call and emits a structured **`tool_call`** with arguments; your code runs it and feeds back a **`ToolMessage`**. "**Dynamic**" = the model picks tools/args at runtime from the conversation, not a hardcoded call.

**LangGraph wiring:**
- **`ToolNode`** — prebuilt executor: reads the last AI message's `tool_calls`, dispatches to the matching `@tool` functions (can run several in parallel), returns `ToolMessage`s.
- **`tools_condition`** — prebuilt router for a conditional edge: if the last message has tool calls → route to `"tools"`; else → `END`.

**The loop *is* ReAct:** `agent (LLM)` → conditional edge → if tool requested → `tools (ToolNode)` → back to `agent` → repeat until the model answers with no tool call → `END`. For a coding agent the tools are things like `read_file`, `search_code`, `run_tests`, `retrieve_docs`.

**Tool-error handling:** `ToolNode` catches exceptions and returns the error as a `ToolMessage` so the LLM can self-correct/retry instead of crashing the graph; destructive tools get a validation or human-approval gate, plus retry limits to prevent infinite loops.

Sources: [LangGraph Quickstart (tools_condition)](https://docs.langchain.com/oss/python/langgraph/quickstart) · [ToolNode source](https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/langgraph/prebuilt/tool_node.py)

---

## Deep Dive 3 — Azure AI Search as the RAG Backend

Why a managed search service instead of a plain vector DB: it does **hybrid retrieval + reranking + ingestion** in one place.

- **Index** holds text, filters, and **vector fields** together — so you can filter/facet alongside similarity search. Vector search uses **HNSW** (approximate NN); text uses **BM25**.
- **Hybrid search** — one request carries both a keyword (`search`) and a `vectorQueries` part; they run in parallel and merge via **Reciprocal Rank Fusion (RRF)**. Vector captures conceptual similarity; BM25 captures exact matches (code symbols, API names, error codes, IDs) that embeddings represent poorly — exactly the mix a coding assistant's queries contain.
- **Semantic ranker** — an optional second-stage **cross-encoder** reranker (`queryType=semantic`) that re-scores the top ~50 candidates reading query + passage together, improving which chunks land on top. (Same bi-encoder→cross-encoder two-stage idea as [project 02](02-semantic-chunking-and-reranking-retrieval.md).)
- **Indexers + skillsets** — the ingestion pipeline: chunk proprietary docs and generate embeddings during indexing (**integrated vectorization**), or push pre-chunked/embedded content via the API.

Sources: [Azure AI Search — Hybrid search overview](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview) · [RAG in Azure AI Search](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview)

---

## Deep Dive 4 — OpenAI Embeddings

An **embedding** is a fixed-length float vector encoding text meaning; nearby vectors (cosine) ≈ semantically similar text. You embed each doc chunk at index time and the query at search time, then find nearest neighbors.

- **`text-embedding-3-small`** — default **1536** dims; a cost-effective default for RAG.
- **`text-embedding-3-large`** — default **3072** dims; higher accuracy, higher cost.
- **`dimensions` parameter (Matryoshka)** — both `-3` models were trained with Matryoshka Representation Learning, so you can **truncate** the vector (e.g., to 1024 or 256) and keep most of the meaning — trading a little accuracy for smaller/cheaper vectors and lower latency.
- **Must match the index:** the Azure AI Search vector field's dimension is fixed at index creation, so the model + `dimensions` choice has to line up.

Sources: [OpenAI — New embedding models](https://openai.com/index/new-embedding-models-and-api-updates/) · [OpenAI — Embeddings guide](https://developers.openai.com/api/docs/guides/embeddings)

---

## Deep Dive 5 — Grounding & Hallucination Reduction

- **Mechanism:** RAG **grounds** generation by injecting retrieved, authoritative passages into the prompt, constraining the model to answer *from provided context* rather than parametric memory.
- **Why proprietary docs specifically:** the base LLM has **zero** training knowledge of a private codebase/APIs — so without retrieval it *must* guess (high hallucination). Grounding supplies facts the model never saw in pretraining.
- **Citations/attribution:** returning source references (doc id, chunk) lets users verify and makes answers auditable; "answer only from these passages and cite the source for each claim" measurably lowers hallucination.
- **Honest limit:** RAG **reduces but doesn't eliminate** hallucination — if retrieval returns incomplete/irrelevant context, the model can still fabricate. So **retrieval quality (hybrid + rerank + good chunking) is the real lever.**

Sources: [RAG in Azure AI Search](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) · [Citation-enforced prompting in RAG (MDPI)](https://www.mdpi.com/2076-3417/16/6/3013)

---

## Interview Q&A

**Q: Why LangGraph over a simple chain?**
The agent needs cycles and conditional control flow — call a tool, observe, decide whether to call another, loop until done, with optional human approval. A chain is a one-pass DAG; it can't natively loop or branch on runtime state. LangGraph's StateGraph gives explicit nodes/edges, shared checkpointed state, and HITL breakpoints.

**Q: How do the dynamic tool calls actually route?**
The LLM node emits a message; a conditional edge runs `tools_condition` — if the message has `tool_calls`, route to the `ToolNode`, else `END`. `ToolNode` executes the tool(s) and appends `ToolMessage`s; an edge routes back to the LLM to continue the ReAct loop.

**Q: How do you handle tool errors / a tool that could do damage?**
`ToolNode` catches exceptions and returns the error as a ToolMessage so the model can self-correct; destructive tools get a validation or human-approval node (LangGraph interrupt/resume), plus retry limits to avoid infinite loops.

**Q: Why Azure AI Search instead of a plain vector DB?**
It does hybrid (BM25 + HNSW vector) merged by RRF, plus an optional semantic cross-encoder reranker and integrated vectorization for ingestion — better recall+precision than pure vector search, with enterprise security trimming. For a code/docs assistant, hybrid matters because queries mix natural language with exact symbol/API names.

**Q: Which embedding model and why?**
`text-embedding-3-small` at 1536 dims as a cost-effective default; the `dimensions` param (Matryoshka) lets me shrink vectors to cut storage/latency, or move to `-3-large` if retrieval quality needs it — as long as it matches the index's vector-field dimension.

**Q: How did you evaluate hallucination reduction?**
Build a ground-truth Q/A set from the proprietary docs; measure retrieval quality (recall@k, MRR/nDCG) and answer faithfulness/groundedness (is each claim supported by a retrieved passage) plus citation correctness, comparing no-RAG vs RAG. Only cite a specific reduction % if you actually measured it.

---

## Honest Caveats

Verify against your real implementation before an interview: the exact **state schema & reducers**, which **nodes** exist, your LangGraph version's **`handle_tool_errors`** behavior, whether you used **classic RAG** vs the newer **agentic retrieval** preview (describe classic unless you truly used the preview), your **embedding model + dimension** and that it matches the index, whether the **semantic ranker** was enabled (it's optional/separately billed), and your **top-k** values. RAG reduces but doesn't eliminate hallucination.

---

## References

- [LangGraph — Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api) · [Quickstart](https://docs.langchain.com/oss/python/langgraph/quickstart) · [Human-in-the-loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- [Azure AI Search — Hybrid search overview](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview) · [RAG overview](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview)
- [OpenAI — New embedding models](https://openai.com/index/new-embedding-models-and-api-updates/) · [Embeddings guide](https://developers.openai.com/api/docs/guides/embeddings)
- [Citation-enforced prompting in RAG (MDPI)](https://www.mdpi.com/2076-3417/16/6/3013)

---

*Previous: [Retrieval Optimization](02-semantic-chunking-and-reranking-retrieval.md) | Next: [Distributed ML Pipeline](04-distributed-ml-pipeline-pyspark-ray.md) | Up: [Guide Home](../README.md)*
