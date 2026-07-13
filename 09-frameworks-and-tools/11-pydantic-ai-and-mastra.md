# Pydantic AI and Mastra: Typed Agent Frameworks (2026)

By May 2026 the agent framework debate has stopped being "LangGraph or LlamaIndex." Two newer entrants now own meaningful production share for teams that prioritize type safety over breadth: **Pydantic AI** in the Python world and **Mastra** in the TypeScript world. Both reject the "string in, string out" surface that older frameworks accepted, and both bet that a fully typed agent is easier to test, evaluate, and operate than a clever-but-untyped one.

## Table of Contents

- [What These Frameworks Are](#what-these-frameworks-are)
- [Pydantic AI: Typed Agents in Python](#pydantic-ai-typed-agents-in-python)
- [Mastra: TypeScript-First Agents](#mastra-typescript-first-agents)
- [Comparison with LangGraph](#comparison-with-langgraph)
- [Choosing a Framework](#choosing-a-framework)
- [Production References](#production-references)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What These Frameworks Are

Both Pydantic AI and Mastra grew out of frustration with framework lock-in and untyped prompt-stitching. They focus on the same set of ideas:

- The agent loop is **defined by code**, not by a YAML / JSON graph.
- Tool calls, structured outputs, and human-in-the-loop checkpoints are all **typed at the function signature**.
- Provider portability is a hard requirement: swap Anthropic for OpenAI for Google by changing one line.
- Evals, tracing, and deployment are first-class, not bolted on.

The differences are mostly stack-shaped: one targets Python services that already use Pydantic for HTTP validation; the other targets Next.js / Node teams that want a Vercel-style developer experience.

---

## Pydantic AI: Typed Agents in Python

### Current State

[Pydantic AI](https://ai.pydantic.dev/) shipped v1.0 in September 2025, settled the 1.x line at **v1.85.1** in April 2026, and entered the **v2.0 beta cycle on May 21, 2026** ([PyPI release history](https://pypi.org/project/pydantic-ai/#history)). The library is built by the team behind Pydantic itself, which also runs [Pydantic Logfire](https://pydantic.dev/logfire). It is open source under MIT.

Key surface area:

- `Agent` class parameterized by an output type and a list of typed tools.
- Provider adapters for Anthropic, OpenAI, Google, Mistral, Groq, Cohere, Ollama, and any OpenAI-compatible endpoint.
- Native OpenTelemetry tracing, exported either to Logfire or any OTLP collector.
- `pydantic_evals` for declarative eval suites with LLM-judge and code-graded scorers.
- A `Graph` API for explicit state machines when the simple `Agent` loop is not enough.

### Why Teams Pick It

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext

class RefundDecision(BaseModel):
    approved: bool
    amount_cents: int = Field(ge=0)
    reason: str

agent = Agent(
    "anthropic:claude-opus-4-7",
    output_type=RefundDecision,
    system_prompt="You are a refund analyst. Approve only if policy allows.",
)

@agent.tool
async def lookup_order(ctx: RunContext, order_id: str) -> dict:
    """Look up an order by id."""
    return await ctx.deps.orders.get(order_id)

result = await agent.run("Refund order 1234", deps=DepContainer(orders=db))
assert isinstance(result.output, RefundDecision)
```

Three properties make this attractive in production:

1. **The return type is enforced.** `result.output` is a `RefundDecision` or the call fails. There is no silent string drift.
2. **Tools are functions, not dicts.** A schema is generated from the Python signature and docstring at registration time, so you cannot accidentally drift the LLM-facing schema from the implementation.
3. **Dependency injection is explicit.** `ctx.deps` is a typed container, which makes the agent trivial to unit-test with mocks.

The [Pydantic AI evals docs](https://ai.pydantic.dev/evals/) describe a typical loop where the same Pydantic model used for the production schema is used for both the LLM output type and the eval scorer's `expected_output`.

### When Pydantic AI Is the Right Choice

- The service is **Python** and already uses Pydantic for HTTP validation (FastAPI is the canonical case).
- You want **strict schemas** end to end: HTTP boundary, LLM tool call, LLM output, database row.
- You want **provider portability** without writing your own adapter layer.
- You are happy to write the agent loop as imperative Python rather than as a graph definition.

### When It Is Not

- You want a **declarative graph** for multi-agent coordination with supervisor patterns. The `Graph` API exists but is more bare-bones than LangGraph.
- You want **time-travel debugging** with branch-from-any-node semantics.
- You need the breadth of the LangChain integration ecosystem (vector stores, document loaders, etc).

---

## Mastra: TypeScript-First Agents

### Current State

[Mastra](https://mastra.ai/) was founded by the team behind Gatsby (graduated YC W25), announced a **$13M seed** led by Lightspeed in October 2025 ([TechCrunch coverage](https://techcrunch.com/2025/10/16/mastra-typescript-agent-framework-seed/)), and **shipped v1.0 in January 2026**. By May 2026 the GitHub repository has crossed **22.3K stars** with **300K+ weekly npm downloads** ([mastra-ai/mastra](https://github.com/mastra-ai/mastra)). Mastra is open source under Elastic License v2.

Key surface area:

- `Agent`, `Workflow`, and `Tool` primitives, all defined as TypeScript with full inference.
- A built-in **local dev server** (`mastra dev`) with a playground UI, eval runner, and trace viewer.
- Tight integration with the **AI SDK** from Vercel for streaming, multi-step tool calls, and provider switching.
- Out-of-the-box memory and RAG with `libsql` / `pgvector` adapters.
- One-command deploy to **Mastra Cloud**, Vercel, Cloudflare Workers, or a Node server.

### Why Teams Pick It

```typescript
import { Agent } from "@mastra/core/agent";
import { createTool } from "@mastra/core/tools";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

const lookupOrder = createTool({
  id: "lookup-order",
  description: "Look up an order by id",
  inputSchema: z.object({ orderId: z.string() }),
  outputSchema: z.object({ status: z.string(), totalCents: z.number() }),
  execute: async ({ context }) => ordersDb.get(context.orderId),
});

export const refundAgent = new Agent({
  name: "refund-agent",
  model: anthropic("claude-opus-4-7"),
  instructions: "You are a refund analyst. Approve only if policy allows.",
  tools: { lookupOrder },
});
```

Three properties make this attractive:

1. **Inferred types end to end.** The Zod schemas drive the tool's runtime validation, the LLM-facing JSON Schema, and the TypeScript type of `context` inside `execute`. One source of truth.
2. **`mastra dev` is the killer feature.** It boots a local UI that lets you call any agent, replay any trace, run any eval, and inspect any tool input/output without writing a frontend.
3. **First-class workflows.** `createWorkflow` defines a typed graph of steps (each a Mastra tool or agent), with branching, suspend / resume, and human-in-the-loop, all type-checked.

The [Generative.inc Mastra guide](https://generative.inc/blog/mastra-typescript-agent-framework) walks through how teams replace Python orchestration with Mastra entirely when the rest of the stack is already TypeScript.

### When Mastra Is the Right Choice

- The team is **TypeScript-first** and the rest of the app is Next.js / Node / Bun / Cloudflare Workers.
- You want **Vercel-style DX**: a single CLI, a local playground, opinionated deployment.
- Streaming UI matters and you want to lean on the AI SDK's `useChat` and `streamText` primitives.
- You want **suspend / resume workflows** with human approval steps wired in by default.

### When It Is Not

- You need a **large library of prebuilt agents** or community integrations. The ecosystem is small compared to LangChain.
- Your team and most of your AI tooling is **Python**. Bridging TS to Python services across an HTTP layer is fine but adds latency.
- You need **academic-style** custom inference behavior (custom decoding, etc). Stay in Python.

---

## Comparison with LangGraph

| Dimension | Pydantic AI v1.85 | Mastra (May 2026) | LangGraph 1.x |
|-----------|-------------------|---------------------|----------------|
| Language | Python | TypeScript | Python and TypeScript |
| License | MIT | Elastic License v2 | MIT |
| Primary unit | Typed `Agent` with `output_type` | Typed `Agent` and `Workflow` | Graph of nodes over typed state |
| Schema source | Pydantic v2 | Zod | JSON Schema (Pydantic, Zod, Valibot, ArkType) |
| Provider neutrality | Built-in adapters | Through Vercel AI SDK | Through LangChain partner packages |
| Multi-agent | Manual or `Graph` API | `Workflow` + agent-as-tool | `create_supervisor`, swarm, custom graphs |
| State persistence | Manual or `pydantic_graph` checkpoint | Workflow snapshot + storage adapters | First-class checkpoint stores (Postgres, Redis, SQLite, in-memory) |
| Time-travel debugging | No | Replay in local playground | Yes, branch from any checkpoint |
| Eval framework | `pydantic_evals` | Mastra evals (built-in) | LangSmith or external |
| Tracing | OTLP / Logfire | OTLP / Mastra Cloud | LangSmith or OTLP |
| Coupling | None to LangChain | None to LangChain | Tight to LangChain ecosystem |
| Ecosystem size | Small but growing | Small but growing | Large (LangChain integrations) |

```mermaid
flowchart TD
    A[Picking an agent framework, May 2026] --> B{Primary language?}
    B -->|Python| C{Do you need a complex multi-agent graph with time-travel?}
    B -->|TypeScript| D{Do you want a managed dev playground and Vercel-style deploy?}
    B -->|Mixed| E[Keep agent code in one language; expose via HTTP/MCP to the other]

    C -->|Yes| F[LangGraph]
    C -->|No, single agent with strict types| G[Pydantic AI]
    C -->|Yes and you want LangChain integrations| F

    D -->|Yes| H[Mastra]
    D -->|No, you want max provider neutrality with no framework opinion| I[Vercel AI SDK direct]
    D -->|You also need Python parity| J[LangGraph TS]

    G --> K[Smallest surface, FastAPI fits cleanly]
    H --> L[mastra dev playground, workflows, deploy adapters]
    F --> M[Mature checkpoints, supervisor patterns, broad ecosystem]
```

---

## Choosing a Framework

Three decision drivers, in order of weight:

1. **Language of the existing service.** Pydantic AI and LangGraph (Python) for Python services. Mastra and LangGraph TS for TypeScript services. Crossing the boundary is almost always a worse trade than picking the right side.
2. **Shape of the complexity.** If the agent is essentially "LLM + a few tools + strict output type," Pydantic AI or Mastra is enough and cheaper to operate. If you have many cooperating agents with branching, retries, and approvals, LangGraph's graph + checkpoint model pulls ahead.
3. **Ecosystem coupling.** LangGraph buys you LangChain integrations, LangSmith eval, and the rest of that surface. Pydantic AI and Mastra buy you cleaner type guarantees and faster cold paths but you wire your own integrations.

A useful heuristic: if the longest thing on the page is the tool list, pick Pydantic AI or Mastra. If the longest thing on the page is the state machine, pick LangGraph.

---

## Production References

These are public references where each framework is in serious use as of May 2026:

- **Pydantic AI**
  - The [Pydantic Logfire dashboards](https://pydantic.dev/logfire) themselves use Pydantic AI for their internal triage agents.
  - The [Sourcegraph Cody](https://sourcegraph.com/cody) team has [blogged about using Pydantic AI](https://ai.pydantic.dev/) for typed code-action agents in their server-side workflows.
  - Many FastAPI shops have adopted it because the same Pydantic model serves the HTTP boundary and the LLM output type.
- **Mastra**
  - [Stripe](https://stripe.com/) developer-experience prototypes ([mastra.ai](https://mastra.ai/)).
  - [Resend](https://resend.com/), [Liveblocks](https://liveblocks.io/), and [Vercel](https://vercel.com/) demo apps.
  - The seed announcement ([TechCrunch](https://techcrunch.com/2025/10/16/mastra-typescript-agent-framework-seed/)) lists production users in fintech and developer tools.
- **LangGraph** (for reference)
  - [LinkedIn's SQL Bot](https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics), [Uber's coding assistants](https://www.uber.com/en-IN/blog/genie-uber-genai-on-call-copilot/), [Klarna](https://www.klarna.com/), [Elastic](https://www.elastic.co/) AI Assistant, [Replit](https://replit.com/), and dozens more on the [LangChain customer page](https://www.langchain.com/built-with-langgraph).

---

## Interview Questions

### Q: When would you choose Pydantic AI over LangGraph for a Python service?

**Strong answer:**
I would choose Pydantic AI when the agent is essentially one LLM with a typed output and a few tools, and the rest of the service is already Pydantic-shaped (FastAPI, SQLModel, etc.). The win is that the same Pydantic model defines the HTTP response, the LLM output, and the eval scorer's expected shape, so there is no schema drift. LangGraph is worth the heavier surface when I need a real multi-agent graph with checkpoint-based time travel, supervisor patterns, or the LangChain integration ecosystem. The deciding question I ask is whether the most complicated part of the design is the tool list or the state machine. Tool list, Pydantic AI. State machine, LangGraph.

### Q: Is Mastra a Vercel AI SDK replacement?

**Strong answer:**
No. Mastra builds on top of the Vercel AI SDK for the actual provider calls and streaming. What Mastra adds is the **agent abstraction**, **workflow engine**, **memory**, **RAG**, **evals**, and the **`mastra dev` playground**. If you only need to call an LLM with streaming and tool calls in a Next.js app, the AI SDK alone is plenty. If you want a typed agent with workflows, suspend / resume, memory, and a local playground, Mastra is the layer that adds those without forcing you to write them yourself.

### Q: What does "typed agent framework" actually buy you in production?

**Strong answer:**
Three things. First, **fewer bad inputs leak through**. The LLM-facing schema is derived from the same Pydantic / Zod definition that validates the runtime payload, so if the LLM hallucinates a field, the parse step rejects it before any downstream code runs. Second, **clean unit tests**. A typed tool is just a function with a Pydantic / Zod boundary, so I can test it without any LLM in the loop. Third, **schema-aware evals**. The eval framework can compare two typed objects field by field rather than diffing strings, which catches subtle regressions like a field becoming optional or an enum gaining a new value.

---

## References

- Pydantic AI v1.85 release notes: https://github.com/pydantic/pydantic-ai/releases
- Pydantic AI documentation: https://ai.pydantic.dev/
- Pydantic AI evals: https://ai.pydantic.dev/evals/
- Mastra repository: https://github.com/mastra-ai/mastra
- Mastra documentation: https://mastra.ai/
- TechCrunch, "Mastra raises $13M seed for TypeScript agent framework" (October 2025): https://techcrunch.com/2025/10/16/mastra-typescript-agent-framework-seed/
- Generative.inc Mastra guide: https://generative.inc/blog/mastra-typescript-agent-framework
- LangGraph 1.x docs: https://docs.langchain.com/oss/python/langgraph/
- LangChain "Built with LangGraph" customer list: https://www.langchain.com/built-with-langgraph
- Vercel AI SDK: https://ai-sdk.dev/
- AIMultiple "Agentic AI frameworks compared" (2026): https://research.aimultiple.com/agentic-ai-frameworks/

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Pydantic AI** | A Python agent framework built by the Pydantic team that enforces strict output types and typed tool signatures. | Eliminates silent schema drift by making the LLM output a validated Pydantic model rather than a raw string. |
| **Mastra** | A TypeScript-first agent framework with a built-in local dev server, workflows, memory, RAG, and one-command cloud deploy. | Provides a Vercel-style developer experience for building and shipping typed agents in the JavaScript/TypeScript ecosystem. |
| **Typed Agent** | An agent whose output type, tool inputs, and tool outputs are all enforced by the type system at both compile time and runtime. | Catches schema mismatches between the LLM and downstream code before they cause silent production failures. |
| **output_type** | A Pydantic AI `Agent` parameter that specifies the Pydantic model the agent must return. | Guarantees that `result.output` is the declared type or the call raises an error, eliminating untyped string outputs. |
| **RunContext** | Pydantic AI's typed dependency container passed to every tool function, carrying injected services and settings. | Makes agents trivially unit-testable by allowing tools to receive mock dependencies instead of real ones. |
| **Dependency Injection** | A design pattern where a component receives its dependencies from an external caller rather than constructing them internally. | Enables clean unit tests of agent tools by replacing database or API clients with mocks. |
| **pydantic_evals** | Pydantic AI's declarative evaluation module for running LLM-judge and code-graded eval suites. | Allows the same Pydantic model used for production output to serve as the expected-output type in evaluations. |
| **Graph API (Pydantic AI)** | An optional Pydantic AI abstraction for defining explicit state machines when the simple agent loop is insufficient. | Provides a more structured alternative for multi-step workflows without requiring a full LangGraph dependency. |
| **Provider Adapter** | A thin wrapper in Pydantic AI or Mastra that translates calls to a specific LLM provider API into the framework's unified interface. | Allows switching between Anthropic, OpenAI, Google, and local models by changing one line. |
| **OpenTelemetry (OTLP)** | An open-source, vendor-neutral standard for collecting and exporting traces, metrics, and logs from applications. | Enables Pydantic AI traces to be sent to any compatible observability backend (Logfire, Jaeger, Datadog, etc.). |
| **Pydantic Logfire** | A cloud observability platform built by the Pydantic team that ingests OpenTelemetry traces with Pydantic-aware display. | Provides a first-party dashboard for monitoring Pydantic AI agents with rich schema-aware trace visualization. |
| **Mastra dev** | The local development server shipped with Mastra that provides a playground UI, eval runner, and trace viewer. | Eliminates the need to write a custom frontend to interactively test agents and inspect tool inputs and outputs. |
| **Workflow (Mastra)** | A typed graph of steps (agents and tools) with branching, suspend/resume, and human-in-the-loop, all type-checked. | Provides a structured, type-safe alternative to hand-coding multi-step agent logic with explicit control flow. |
| **Suspend / Resume** | A workflow capability that pauses execution at a defined step and continues after receiving an external signal or human input. | Enables durable, long-running workflows that can wait for approvals or external events without polling. |
| **Vercel AI SDK** | Vercel's open-source library for building AI-powered applications with streaming, tool calls, and multi-provider support in JavaScript/TypeScript. | The underlying provider-call layer that Mastra builds on top of for streaming and model interaction. |
| **Zod** | A TypeScript schema declaration and validation library used by Mastra for tool input/output schemas. | Provides the single source of truth for both runtime validation and TypeScript type inference in Mastra tools. |
| **Schema Drift** | A silent divergence between the schema the LLM is told to produce and the schema the application code expects to receive. | The core failure mode that typed agent frameworks like Pydantic AI and Mastra are designed to prevent. |
| **Elastic License v2** | A source-available software license that allows free use and modification but restricts commercialized hosting of the software as a service. | The license under which Mastra is published, permitting most enterprise production use while protecting the project's commercial interests. |
| **libsql / pgvector** | Storage adapters for SQLite (via libsql) and PostgreSQL (via the pgvector extension) used by Mastra for memory and RAG. | Provides out-of-the-box vector storage for agent memory without requiring a dedicated vector database. |
| **useChat / streamText** | Vercel AI SDK primitives for streaming chat UI and streaming text generation in Next.js applications. | Give Mastra-powered agents a ready-made real-time streaming interface without custom WebSocket or SSE code. |
| **FastAPI** | A Python web framework for building APIs with automatic Pydantic-based request and response validation. | The canonical pairing for Pydantic AI: the same Pydantic models validate both the HTTP boundary and the LLM output type. |
| **Time-Travel Debugging** | The ability to replay a workflow from any previously saved checkpoint, not just the most recent one. | Enables engineers to reproduce and fix failures by branching from the exact historical state where a problem occurred. |

*Next: See the [Framework Selection Guide](08-framework-selection-guide.md) for cross-framework selection criteria.*
