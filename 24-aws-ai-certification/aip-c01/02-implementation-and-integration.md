# Domain 2: Implementation and Integration

> **AIP-C01 · 26% of the exam · ~17 scored questions**

This domain is where you prove you can actually *build* things. It moves beyond knowing what AWS services exist (that's Domain 1) and tests whether you understand the API shapes, the agentic wiring, the right customization lever, and the integration patterns that connect a foundation model to a real production application.

See also: [Amazon Bedrock service reference](../services/bedrock.md) · [Amazon SageMaker service reference](../services/sagemaker.md)

---

> **Plain English:**
> Domain 2 is the "show me the code" domain. Every question gives you a scenario — a dev team building a chat UI, a workflow that needs tool calls, a cost problem that a smaller model could solve — and asks which API, which service, or which customization path fits. Burn in three reflexes: **Converse API for any multi-model / tool-use / streaming need**, **AgentCore for production agents that need managed memory + identity**, and **distillation when you want a smaller-cheaper model that behaves like the big one**.

---

## Table of Contents

1. [Bedrock Runtime APIs](#1-bedrock-runtime-apis)
2. [Prompt Engineering (Developer-Grade)](#2-prompt-engineering-developer-grade)
3. [Agentic AI — Agents, AgentCore, Strands, MCP](#3-agentic-ai)
4. [Model Customization Choices](#4-model-customization-choices)
5. [Integration Patterns](#5-integration-patterns)
6. [Glossary](#glossary)
7. [References](#references)

---

## 1. Bedrock Runtime APIs

### 1.1 `InvokeModel` — the low-level escape hatch

[`InvokeModel`](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html) sends a raw JSON body directly to one model. The request shape is model-specific (Claude's body looks different from Llama's), the response is model-specific, and there is no built-in multi-turn history. Use it only when:

- You need a feature not yet in the Converse API (e.g., embeddings via `InvokeModel` on Titan Embeddings).
- You are wrapping a model that does not support the Messages API at all.
- You want the absolute lowest abstraction layer for experimentation.

**IAM permission required:** `bedrock:InvokeModel`
**Streaming variant:** `InvokeModelWithResponseStream` — same model-specific body, but returns a chunked event stream.

---

### 1.2 `Converse` and `ConverseStream` — the unified API

Announced May 2024, [`Converse`](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime/client/converse.html) is now the **recommended runtime API** for virtually every new build. It provides one normalised request/response shape across every Bedrock model that supports the Messages API.

**What Converse abstracts away:**
- Per-model JSON body differences
- Per-model inference-parameter names
- Multi-turn history management (you pass the full `messages` array each call)
- Tool use / function calling (uniform `toolConfig` block)
- Streaming (via `ConverseStream`)

**IAM permissions:**
- `Converse` requires `bedrock:InvokeModel`
- [`ConverseStream`](https://docs.aws.amazon.com/boto3/latest/reference/services/bedrock-runtime/client/converse_stream.html) requires `bedrock:InvokeModelWithResponseStream`

As of **February 2026**, Bedrock batch inference also supports the Converse API format — you can use the same request shape for both real-time and batch jobs. ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-bedrock-batch-inference-supports-converse-api-format))

---

### 1.3 Converse API request structure

```python
import boto3, json

client = boto3.client("bedrock-runtime", region_name="us-east-1")

response = client.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    system=[{"text": "You are a helpful financial advisor. Be concise."}],
    messages=[
        {"role": "user",    "content": [{"text": "What is dollar-cost averaging?"}]},
        {"role": "assistant","content": [{"text": "Dollar-cost averaging is..."}]},  # prior turn
        {"role": "user",    "content": [{"text": "Give me a 3-step plan to start."}]},
    ],
    inferenceConfig={
        "temperature": 0.3,   # 0.0–1.0; lower = more deterministic
        "topP": 0.9,          # nucleus sampling; often paired with temperature
        "maxTokens": 512,
        "stopSequences": ["<END>"],
    },
)

output_text = response["output"]["message"]["content"][0]["text"]
print(output_text)
```

**Key fields:**
| Field | Purpose |
|---|---|
| `system` | System prompt(s) — model-level persona and constraints |
| `messages` | Alternating `user`/`assistant` turns; you maintain the full history |
| `inferenceConfig.temperature` | Randomness (0 = greedy, 1 = max creative) |
| `inferenceConfig.topP` | Nucleus sampling — keep tokens whose cumulative probability ≤ topP |
| `inferenceConfig.topK` | Passed inside `additionalModelRequestFields` for models that support it |
| `inferenceConfig.maxTokens` | Hard cap on generated tokens |
| `inferenceConfig.stopSequences` | List of strings that stop generation early |

---

### 1.4 Streaming with `ConverseStream`

Use `ConverseStream` whenever latency-to-first-token matters — chat UIs, copilots, voice pipelines:

```python
response = client.converse_stream(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[{"role": "user", "content": [{"text": "Explain quantum entanglement simply."}]}],
    inferenceConfig={"maxTokens": 300, "temperature": 0.5},
)

stream = response.get("stream")
for event in stream:
    if "contentBlockDelta" in event:
        delta = event["contentBlockDelta"]["delta"]
        if "text" in delta:
            print(delta["text"], end="", flush=True)
```

The event stream emits `messageStart`, `contentBlockStart`, `contentBlockDelta` (text chunks), `contentBlockStop`, and `messageStop` events. Parse only the ones you need.

---

### 1.5 Structured output / Tool use via `toolConfig`

The Converse API handles function calling through the `toolConfig` block. The model emits a `toolUse` content block when it wants to call a tool; you execute the function and return a `toolResult` block. The model returns tool inputs conforming to the JSON Schema you declare in `inputSchema`, which sharply reduces malformed or out-of-schema tool calls — though you should still validate the input before executing, as with any external input.

```python
tools = [
    {
        "toolSpec": {
            "name": "get_stock_price",
            "description": "Returns the latest price for a stock ticker symbol.",
            "inputSchema": {
                "json": {
                    "type": "object",
                    "properties": {
                        "ticker": {"type": "string", "description": "Stock ticker, e.g. AMZN"}
                    },
                    "required": ["ticker"],
                }
            },
        }
    }
]

# --- Round 1: model decides to call a tool ---
response = client.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[{"role": "user", "content": [{"text": "What is Amazon's stock price?"}]}],
    toolConfig={"tools": tools},
)

# Inspect the response
for block in response["output"]["message"]["content"]:
    if block.get("toolUse"):
        tool_name  = block["toolUse"]["name"]          # "get_stock_price"
        tool_input = block["toolUse"]["input"]         # {"ticker": "AMZN"}
        tool_use_id = block["toolUse"]["toolUseId"]

# --- Execute the real function ---
price = 198.42  # result from your actual stock API

# --- Round 2: return result to the model ---
follow_up = client.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[
        {"role": "user",      "content": [{"text": "What is Amazon's stock price?"}]},
        {"role": "assistant", "content": response["output"]["message"]["content"]},
        {
            "role": "user",
            "content": [
                {
                    "toolResult": {
                        "toolUseId": tool_use_id,
                        "content": [{"json": {"price": price, "currency": "USD"}}],
                    }
                }
            ],
        },
    ],
    toolConfig={"tools": tools},
)
print(follow_up["output"]["message"]["content"][0]["text"])
```

> **🎯 On the exam**
>
> - **"Unified multi-model API" or "need tool use / function calling across models"** → **`Converse` / `ConverseStream`**. Never `InvokeModel` for this.
> - `InvokeModel` is the answer only when the question mentions embeddings, a model outside the Converse-supported list, or a legacy integration.
> - `topK` is **not** in `inferenceConfig` — pass it in `additionalModelRequestFields` for models that support it; getting this wrong is a common distractor.
> - Streaming answer = `ConverseStream` (not `InvokeModelWithResponseStream` for new code).
> - Batch jobs: since Feb 2026, use Converse format — same schema, async S3-in / S3-out.

---

## 2. Prompt Engineering (Developer-Grade)

### 2.1 System prompts

A system prompt sets the persistent persona, rules, and constraints for the entire conversation. In the Converse API it is the `system` array at the top level — it is **not** a user message. Best practices:

- Be explicit about format: "Respond only with valid JSON matching this schema: …"
- Set hard constraints: "Never discuss competitor products."
- Define scope: "You are a Tier-1 AWS support agent. Escalate billing questions."

### 2.2 Few-shot / in-context learning

Provide 2–5 worked examples in the `messages` history before the real request. The model pattern-matches against them at inference time — no weight updates, no training cost:

```
user:    "Classify: 'My order hasn't arrived.' → "
assistant: "SHIPPING"
user:    "Classify: 'I need a refund.' → "
assistant: "BILLING"
user:    "Classify: 'The app keeps crashing.' → "   ← real query
```

### 2.3 Chain-of-thought (CoT)

Add "Think step by step" or include a scratchpad pattern in the system prompt. Extended thinking models (e.g., Claude 3.7 Sonnet) can emit reasoning tokens natively. For budget-sensitive use-cases, you can cap reasoning tokens with `budgetTokens`.

### 2.4 Prompt templates

Parameterise prompts as strings with placeholders (`{{customer_name}}`, `{{product}}`). Separate the template from the code so prompt engineers can iterate without code deploys.

---

### 2.5 Amazon Bedrock Prompt Management

[Bedrock Prompt Management](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html) (GA November 2024) is a centralised service for creating, testing, versioning, and sharing prompts:

- **Versions are immutable** — once created a version cannot be modified; update the draft, then cut a new version.
- Prompts are referenced by ARN or logical identifier; you pass the `promptIdentifier` to `Converse` / `InvokeModel` instead of embedding text in code.
- Supports A/B testing between versions and rollback.
- Decouples prompt engineering from application deployments.

### 2.6 Amazon Bedrock Flows (Prompt Flows)

[Bedrock Flows](https://aws.amazon.com/bedrock/flows/) is a visual drag-and-drop orchestration builder for chaining prompts, Knowledge Bases, Lambda functions, and other nodes into reproducible workflows:

- **Draft → Version → Alias** lifecycle mirrors Lambda aliases for safe production rollouts.
- Nodes include: Prompt, Knowledge Base, Lambda, Condition, Iterator, Collector, Input, Output.
- Versions are read-only; create a new draft to change a flow.
- Flows are exposed as an ARN that your application calls — the logic lives in Bedrock, not in application code.

> **🎯 On the exam**
>
> - **Versioned, reusable prompt stored in Bedrock** → Prompt Management.
> - **Visual chaining of prompts + KB + Lambda without writing orchestration code** → Bedrock Flows.
> - CoT = instruct the model to reason before answering; it does not require fine-tuning.
> - Few-shot examples go in the `messages` history, **not** the system prompt.

---

## 3. Agentic AI

### 3.1 Amazon Bedrock Agents (Classic)

> **Note (July 2026):** AWS has designated Bedrock Agents as "Classic" and closed it to new customers as of July 30, 2026. Existing Classic agents continue to work; **new builds should use AgentCore** (see §3.2). The concepts below still appear on the exam because many production environments run Classic agents.

[Bedrock Agents Classic](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) wraps a foundation model with:

| Component | Role |
|---|---|
| **Orchestration loop** | ReAct-style planner: Thought → Action → Observation → … → Final Answer |
| **Action groups** | Sets of API operations the agent can call; each action group is backed by a **Lambda function** and described by an OpenAPI schema |
| **Knowledge Bases** | RAG attachment — the agent can retrieve context before acting |
| **Session memory** | Optional per-session context store for multi-turn conversations |
| **Guardrails** | Content filtering applied to both input and output |

**Action group → Lambda contract:**

The agent sends a structured JSON event to your Lambda specifying `apiPath`, `httpMethod`, `parameters`, and `requestBody`. Your Lambda must return a response matching the OpenAPI schema. One Lambda per action group handles all operations defined in that group. ([AWS docs](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html))

```python
# Lambda handler skeleton for a Bedrock Agent action group
def lambda_handler(event, context):
    api_path   = event["apiPath"]          # e.g., "/getOrderStatus"
    parameters = event.get("parameters", [])
    
    if api_path == "/getOrderStatus":
        order_id = next(p["value"] for p in parameters if p["name"] == "orderId")
        status   = lookup_order(order_id)  # your business logic
        return {
            "messageVersion": "1.0",
            "response": {
                "actionGroup": event["actionGroup"],
                "apiPath":     api_path,
                "httpMethod":  event["httpMethod"],
                "httpStatusCode": 200,
                "responseBody": {
                    "application/json": {"body": json.dumps({"status": status})}
                },
            },
        }
```

**Multi-step workflows with Step Functions:**
For deterministic, auditable multi-step flows (e.g., "validate → enrich → submit → notify"), integrate Bedrock Agents with [AWS Step Functions](https://aws.amazon.com/step-functions/). The agent handles NL reasoning; Step Functions handles state, retries, branching, and observability.

---

### 3.2 Amazon Bedrock AgentCore — the production path (2026)

[Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/aws/introducing-amazon-bedrock-agentcore-securely-deploy-and-operate-ai-agents-at-any-scale/) is the managed production runtime for AI agents. It is built from a set of individually-billed managed components — **Runtime, Memory, Gateway, Identity, Browser, Code Interpreter, Observability, Evaluations, and Policy**; the managed agent **harness** reached general availability on June 17, 2026. The core components:

| Component | What it does |
|---|---|
| **Runtime** | Serverless, session-isolated execution environment for agent code; supports long (multi-hour) async execution and the A2A protocol |
| **Memory** | Two-tier (session + long-term) context store; you control what the agent remembers across sessions |
| **Gateway** | Transforms existing REST APIs and Lambda functions — and connects to existing MCP servers — into agent-compatible tools; supports IAM and OAuth authorization. Server-side tool execution reduces tool-call round-trip latency versus client-side orchestration |
| **Browser** | Managed browser instances for web automation (scraping, form-filling, UI testing) |
| **Code Interpreter** | Isolated sandbox to execute LLM-generated code (Python, shell) safely |
| **Identity** | Built-in OAuth/OIDC so agents can authenticate to downstream services without custom auth code |
| **Observability** | Traces, metrics, logs for agent sessions |
| **Evaluations** | Built-in agent quality evaluation |

**Why AgentCore over Classic:**
- True session isolation (no cross-tenant bleed)
- Managed memory without building your own DynamoDB schema
- Gateway unifies MCP + REST without a proxy layer you maintain
- Identity removes the need to embed credentials in Lambda

```python
# Invoking an AgentCore Runtime agent (conceptual)
import boto3

agentcore = boto3.client("bedrock-agentcore-runtime", region_name="us-east-1")

response = agentcore.invoke_agent(
    agentId="my-agent-id",
    sessionId="user-session-abc123",
    inputText="Book a flight to Seattle next Tuesday under $400.",
)
# Response streams back via event stream similar to ConverseStream
```

---

### 3.3 Strands Agents SDK (open-source)

[Strands Agents](https://strandsagents.com/) is AWS's open-source SDK (Python + TypeScript) released May 2025. It is **model-agnostic** — works with Bedrock, Anthropic direct API, OpenAI, Gemini, and others — so teams can swap backends without rewriting agent code.

Key capabilities ([AWS blog](https://aws.amazon.com/blogs/machine-learning/strands-agents-sdk-a-technical-deep-dive-into-agent-architectures-and-observability/)):

- First-class **MCP** support — agents get access to thousands of MCP-compatible tools out of the box
- Multi-agent patterns: **Graph**, **Swarm**, and **Workflow**
- **A2A protocol** for cross-framework agent interoperability
- Step-level safety controls (guardrails per tool call)
- Native integration with AgentCore Runtime and Memory

```python
from strands import Agent
from strands.tools.mcp import MCPClient

# Connect to an MCP server (e.g., a local Filesystem MCP server)
mcp = MCPClient("stdio", command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp"])

agent = Agent(
    model="us.anthropic.claude-3-5-sonnet-20241022-v2:0",
    tools=[mcp],
    system_prompt="You are a file management assistant.",
)

result = agent.run("List all Python files in /tmp and summarise what each does.")
print(result)
```

---

### 3.4 Model Context Protocol (MCP)

[MCP](https://aws.amazon.com/about-aws/whats-new/2025/10/model-context-protocol-proxy-available) is an open standard (originally from Anthropic, now broadly adopted) for connecting AI models to external context sources and tools via a structured protocol. Think of it as USB-C for AI tools — a single plug that works regardless of which model or framework is running.

**MCP components:**
- **MCP Server** — exposes tools/resources/prompts via the protocol (e.g., a GitHub MCP server, a database MCP server)
- **MCP Client** — the agent's side; discovers and calls tools from MCP servers
- **Transport** — stdio (local), SSE (HTTP streaming), or WebSocket

**AgentCore Gateway as MCP proxy:** Gateway can wrap existing Lambda functions and REST APIs and surface them as MCP tools — no need to rewrite your existing integrations.

> **🎯 On the exam**
>
> - **"Managed agent runtime with memory and identity"** → **AgentCore** (not Classic Agents)
> - **New agent builds after July 2026** → AgentCore is the default path; Classic = legacy
> - **Action group backed by Lambda** = Classic Agents pattern; know the Lambda event shape
> - **Open-source, model-agnostic agent SDK + MCP** → **Strands Agents**
> - **Deterministic multi-step workflow** alongside an agent → add **Step Functions** for the state machine; let the agent handle the NL reasoning
> - **Tool integration standard across frameworks** → **MCP**
> - AgentCore Gateway executes tools server-side, reducing tool-round-trip latency vs. client-side orchestration (and can wrap existing REST/Lambda or connect to existing MCP servers)

---

## 4. Model Customization Choices

Bedrock offers a spectrum of customization, and **choosing the right level** is a recurring exam scenario. Each option trades off cost, data requirements, latency, and control.

### 4.1 Decision tree

```
Start here: Does the base model give acceptable outputs with a good system prompt?
│
├─ YES → use prompt engineering (zero cost, zero latency overhead)
│
└─ NO → Does the model lack domain *knowledge* (facts, vocabulary)?
         │
         ├─ YES, but data is in documents → RAG (Knowledge Base)
         │
         └─ YES, need the weights to learn → customization track:
                  │
                  ├─ Labelled input-output pairs available → Fine-Tuning
                  │
                  ├─ Large unlabelled domain corpus → Continued Pre-Training
                  │
                  └─ Want a smaller/cheaper model that mimics a large one → Distillation
```

### 4.2 Prompt engineering

Zero infrastructure cost. The model stays unchanged. Works well when the task is well-specified and outputs are acceptable with few-shot examples + a strong system prompt. Always try this first.

### 4.3 Retrieval-Augmented Generation (RAG) / Knowledge Bases

Bedrock Knowledge Bases ingest documents (S3, Confluence, SharePoint, web crawl), chunk them, embed them, and store in a vector store (OpenSearch Serverless, Aurora, Pinecone, MongoDB Atlas, Redis). At inference time the agent or your code retrieves relevant chunks and injects them into the prompt.

**When to use:** Model needs access to frequently-updated private data (product catalogue, policy docs, customer records). No weight changes.

### 4.4 Fine-tuning on Bedrock

Supervised fine-tuning with labelled (`prompt`, `completion`) pairs. Adjusts the model's weights for a specific task or style. Requires a training dataset in S3 (JSONL format), a supported base model, and a training job in Bedrock.

**When to use:** Consistent output format, domain-specific tone, or task the base model performs poorly on even with few-shot examples.

**Supported bases (as of 2026):** Amazon Nova, Titan, certain Llama models via Custom Model Import pathway.

### 4.5 Continued pre-training

Trains on **unlabelled** domain text to bake vocabulary, facts, and style into the weights. More expensive than fine-tuning (needs more tokens), but teaches the model domain concepts it never saw in pre-training (e.g., internal medical terminology, proprietary code patterns).

**When to use:** The domain is so specialised that even fine-tuning doesn't close the gap because the base model's vocabulary for the domain is weak.

### 4.6 Model distillation on Bedrock

[Distillation](https://aws.amazon.com/bedrock/faqs/) transfers knowledge from a large **teacher** model to a smaller **student** model. The teacher generates synthetic labelled examples; the student trains on them. Result: a small, fast, cheap model that approximates the teacher's quality on your task.

**When to use:** You need the quality of a large frontier model but the cost/latency of a small one at high call volume. Classic exam scenario: "We currently use Claude Opus-equivalent quality but are paying too much — how do we keep quality while cutting cost?" → Distillation.

### 4.7 SageMaker JumpStart for open-weight models

[SageMaker JumpStart](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-jumpstart.html) provides one-click deployment of open-weight models (Llama, Falcon, Mistral, Stable Diffusion, etc.) to SageMaker endpoints. Use it when:

- You need a model not available in Bedrock's managed catalogue
- You require a dedicated GPU endpoint (provisioned throughput, no cold start)
- You want to fine-tune open-weight models on SageMaker and then import them

### 4.8 Bedrock Custom Model Import

[Custom Model Import](https://docs.aws.amazon.com/bedrock/latest/userguide/import-pre-trained-model.html) lets you bring weights trained or fine-tuned outside Bedrock (SageMaker, EC2, on-premises) into Bedrock's serverless invocation layer. Supported architectures include Mistral, Mixtral, Flan, Llama 2/3/3.1/3.2.

**Flow:** Fine-tune in SageMaker JumpStart → export weights to S3 → import into Bedrock Custom Model → invoke via standard Bedrock APIs.

> **🎯 On the exam**
>
> - **"Reduce a large model's cost while keeping quality"** → **Distillation** (not fine-tuning)
> - **"Model lacks domain knowledge, data is in documents"** → **RAG / Knowledge Bases** (no weight change)
> - **"Model performs consistently wrong on a specific task despite prompt engineering"** → **Fine-tuning**
> - **"Massive unlabelled domain corpus"** → **Continued pre-training**
> - **"Open-weight model, dedicated endpoint"** → **SageMaker JumpStart**
> - **"Bring SageMaker-trained weights into Bedrock APIs"** → **Custom Model Import**
> - Distillation requires a teacher model and Bedrock generates the synthetic dataset; you don't curate pairs manually

---

## 5. Integration Patterns

### 5.0 Bedrock inference types — end-to-end flows

Amazon Bedrock exposes **three inference types**, plus **Provisioned Throughput** as a capacity/billing mode layered on top of the real-time types. Knowing the exact flow of each — and when to reach for it — is a recurring AIP-C01 scenario. ([Bedrock batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html))

| Inference type | API | Sync / async | Latency | Cost | Best for |
|---|---|---|---|---|---|
| **Synchronous (real-time)** | `InvokeModel` / `Converse` | Synchronous | Full answer in seconds | Per-token (on-demand) | Interactive single-turn Q&A, classification, extraction |
| **Streaming (real-time)** | `InvokeModelWithResponseStream` / `ConverseStream` | Synchronous, streamed | First token in ms | Per-token (on-demand) | Chat UIs, copilots, voice — anything where perceived latency matters |
| **Batch (asynchronous)** | `CreateModelInvocationJob` | Asynchronous | Minutes–hours | **~50% of on-demand** | Bulk, non-urgent jobs: nightly summaries, dataset labelling, embeddings backfill |
| **Provisioned Throughput** *(capacity mode)* | same real-time APIs via `provisionedModelArn` | Synchronous | Consistent, no shared-pool contention | Hourly per **model unit (MU)** + term commitment | Steady high-volume production; **required for custom/fine-tuned models** |

> **Key distinction:** Bedrock has **no separate native "asynchronous endpoint"** like SageMaker's async inference. **Batch inference is the native async mode.** The "asynchronous invocation" pattern in §5.2 (SQS → Lambda → Bedrock) is an *application-level* pattern that wraps **synchronous** calls — it is not a distinct Bedrock inference type.

#### Flow 1 — Synchronous (on-demand) invocation

```
Client
  │  build messages[] + inferenceConfig
  ▼
Converse / InvokeModel  ──►  Bedrock  ──►  runs model to completion
  │                                             │
  │◄──────────── full response (one payload) ◄──┘
  ▼
Parse output["message"]["content"][0]["text"]
```
One request, one blocking response. Simplest path; the caller waits for the entire generation.

#### Flow 2 — Streaming invocation

```
Client
  │  call ConverseStream
  ▼
Bedrock opens an event stream ──►  messageStart
                                   contentBlockStart
                                   contentBlockDelta  ← text chunk ┐
                                   contentBlockDelta  ← text chunk │ render as they arrive
                                   contentBlockDelta  ← text chunk ┘
                                   contentBlockStop
                                   messageStop  (+ usage/metrics)
```
Same synchronous request, but tokens arrive incrementally — the UI renders partial output immediately, cutting *perceived* latency. Monitor `TimeToFirstToken` in CloudWatch (streaming APIs only).

#### Flow 3 — Batch inference (asynchronous job)

```
1. Prepare input JSONL in S3   (one record per line, minimum 100 records)
      { "recordId": "0001", "modelInput": { …Converse or InvokeModel body… } }
2. CreateModelInvocationJob(
        modelId, roleArn,
        inputDataConfig  = s3://bucket/in/,
        outputDataConfig = s3://bucket/out/ )   ──►  returns jobArn
3. Bedrock runs the job asynchronously on a SEPARATE batch quota (~50% of on-demand price)
4. Track status:  Submitted → Validating → InProgress → Completed / Failed / Stopped
      • poll GetModelInvocationJob(jobArn), or
      • trigger on an EventBridge status-change event
5. Read output JSONL from S3:
      { "recordId": "0001", "modelInput": {…}, "modelOutput": {…} }
      (records keyed by recordId so you can rejoin to inputs)
```
No persistent endpoint; you submit files and collect results later. Ideal when latency tolerance is minutes-to-hours and cost matters. **Prompt caching is not available with batch.**

#### Flow 4 — Provisioned Throughput (capacity mode over real-time)

```
1. PurchaseProvisionedModelThroughput(
        modelId, modelUnits, commitmentDuration )   ──►  provisionedModelArn
        (term commitment, e.g. 1-month or 6-month; custom/fine-tuned models require PT)
2. Invoke with modelId = provisionedModelArn using the SAME
   Converse / InvokeModel / …Stream APIs  (Flows 1 & 2 unchanged)
3. Billed per model unit per hour for the term — regardless of traffic —
   giving reserved, contention-free throughput and predictable cost
```
> **⚠ ARN pitfall:** you must pass the **`provisionedModelArn`** as `modelId`. Passing the base foundation-model ID leaves your reserved capacity idle while you keep paying on-demand rates.

> **🎯 On the exam**
>
> - **Interactive answer, wait for full text** → **synchronous** `Converse`/`InvokeModel`.
> - **"UI hangs before anything appears" / typing-style UX** → **streaming** `ConverseStream`.
> - **Large, non-urgent, cost-sensitive job (10K+ prompts)** → **batch inference** (`CreateModelInvocationJob`, ~50% off, S3 in/out, min 100 records).
> - **Steady 24/7 high volume, or invoking a fine-tuned/custom model** → **Provisioned Throughput** (via `provisionedModelArn`).
> - **"Bedrock async endpoint that scales to zero"** is a distractor — that's SageMaker; Bedrock's async path is **batch inference**.

---

### 5.1 Synchronous invocation

The caller blocks until the model responds. Use for:
- Interactive chat (with `ConverseStream` for streaming UX)
- Low-latency single-turn Q&A
- Real-time classification/extraction embedded in a request path

**Topology:** Client → API Gateway → Lambda → Bedrock `Converse`/`ConverseStream` → response

```python
# API Gateway + Lambda front end (Lambda handler)
def handler(event, context):
    body    = json.loads(event["body"])
    user_msg = body["message"]
    session_history = body.get("history", [])  # client maintains history

    session_history.append({"role": "user", "content": [{"text": user_msg}]})

    resp = bedrock.converse(
        modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
        system=[{"text": SYSTEM_PROMPT}],
        messages=session_history,
        inferenceConfig={"maxTokens": 512, "temperature": 0.4},
    )

    assistant_msg = resp["output"]["message"]["content"][0]["text"]
    session_history.append({"role": "assistant", "content": [{"text": assistant_msg}]})

    return {
        "statusCode": 200,
        "body": json.dumps({"response": assistant_msg, "history": session_history}),
    }
```

**Conversational memory / session state options:**

| Approach | Where state lives | Best for |
|---|---|---|
| Client-side history | Browser / mobile app sends full `messages` array | Simple chat apps |
| Server-side DynamoDB | Lambda writes/reads turns by `sessionId` | Multi-device, cross-session |
| AgentCore Memory | Managed by AgentCore (session + long-term tiers) | Production agents |
| ElastiCache (Redis) | Fast in-memory, TTL-based | High-throughput, short sessions |

### 5.2 Asynchronous invocation

Decouple the caller from the model call. Use for:
- Long-running inference (document summarisation, code generation)
- Fan-out to multiple models
- Retry logic without client timeout

**Topology:** Client → SQS → Lambda → Bedrock → DynamoDB (results) → SNS/EventBridge notification

### 5.3 Batch inference

[Bedrock Batch Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html) processes large volumes of prompts at **significantly lower cost** (up to 50% off on-demand pricing) with a file-based interface:

1. Upload JSONL file to S3 (each line = one Converse-format request since Feb 2026)
2. Call `CreateModelInvocationJob`
3. Poll or use EventBridge for completion
4. Read output JSONL from S3

```python
s3_input  = "s3://my-bucket/batch-input/prompts.jsonl"
s3_output = "s3://my-bucket/batch-output/"

job = bedrock.create_model_invocation_job(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    jobName="nightly-summary-job",
    inputDataConfig={"s3InputDataConfig": {"s3Uri": s3_input, "s3InputFormat": "JSONL"}},
    outputDataConfig={"s3OutputDataConfig": {"s3Uri": s3_output}},
)
```

### 5.4 API Gateway + Lambda front end

The standard serverless pattern for exposing Bedrock to web / mobile clients:

- **API Gateway** handles TLS, auth (Cognito authoriser / API key / IAM), throttling, WAF
- **Lambda** validates input, manages session state, calls Bedrock, post-processes output
- **Lambda response streaming** (since Lambda 2023) lets you stream `ConverseStream` chunks directly to the HTTP client without buffering

### 5.5 Provisioned Throughput vs On-Demand

| Mode | Billing | When to use |
|---|---|---|
| **On-Demand** | Per input+output token | Variable / unpredictable traffic |
| **Provisioned Throughput** | Reserved model units (MUs) per hour | High, steady throughput; predictable cost; also required for **fine-tuned / custom models** |

> **🎯 On the exam**
>
> - **Large offline batch job, cost-optimised** → Batch Inference (S3 in/out, async)
> - **Streaming UX in a serverless API** → API Gateway + Lambda response streaming + `ConverseStream`
> - **Client maintains conversation history by sending full `messages` array** → standard stateless Converse pattern
> - **Need session state across devices or agent memory** → DynamoDB (server-side) or AgentCore Memory
> - **Custom / fine-tuned model must use** → Provisioned Throughput (cannot use on-demand for custom models)
> - Batch inference pricing ≈ 50% of on-demand; use when latency SLO > minutes

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| `InvokeModel` | Raw, model-specific Bedrock API | Max control; embeddings; legacy |
| `Converse` | Unified, model-agnostic chat API | New builds; tool use; multi-turn |
| `ConverseStream` | Streaming variant of Converse | Real-time typing-style UX |
| System prompt | Top-level model instructions, not a user message | Persona, constraints, format rules |
| Few-shot | Example input-output pairs in the message history | Guide model behaviour without training |
| Chain-of-thought | Instructing the model to reason before answering | Improves accuracy on complex tasks |
| Prompt Management | Bedrock service for versioning and reusing prompts | Decouples prompts from app code |
| Bedrock Flows | Visual orchestration of prompt/KB/Lambda chains | No-code/low-code workflow builder |
| Bedrock Agents (Classic) | NL-driven agent with action groups + Lambda | Legacy; closed to new customers Jul 2026 |
| AgentCore | Managed serverless agent runtime (Runtime, Memory, Gateway, Browser, Code Interpreter, Identity) | Production agents, 2026+ |
| Action group | Set of API operations an agent can call, backed by Lambda | Gives Classic agent tools |
| Strands Agents | AWS open-source agent SDK (Python/TypeScript) | Model-agnostic, MCP-native agent building |
| MCP | Model Context Protocol — open standard for tool/context integration | Universal tool plug for AI agents |
| Fine-tuning | Supervised weight update with labelled pairs | Task/style specialisation |
| Continued pre-training | Weight update on unlabelled domain text | Domain vocabulary/knowledge |
| Distillation | Small student model learns from large teacher model | Cost/latency reduction with quality preservation |
| RAG | Retrieve relevant doc chunks and inject into prompt | Private/dynamic data access without weight changes |
| Custom Model Import | Import SageMaker/EC2-trained weights into Bedrock | Run custom weights via Bedrock APIs |
| JumpStart | SageMaker one-click open-weight model deployment | Dedicated endpoints for open-weight models |
| Batch Inference | File-based async bulk inference job | Cost-optimised, non-real-time processing |
| Provisioned Throughput | Reserved model capacity billed per hour | High steady traffic; required for custom models |
| `toolConfig` | Converse API field for declaring callable tools | Enables structured output / function calling |
| `toolUse` / `toolResult` | Content blocks in the agentic loop | Model requests a tool call; you return the result |
| Session state | Conversation history or context persisted across turns | Multi-turn coherence |

---

## References

- [Amazon Bedrock Converse API — Boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime/client/converse.html)
- [Amazon Bedrock ConverseStream — Boto3 docs](https://docs.aws.amazon.com/boto3/latest/reference/services/bedrock-runtime/client/converse_stream.html)
- [Bedrock batch inference now supports Converse API format (Feb 2026)](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-bedrock-batch-inference-supports-converse-api-format)
- [Bedrock batch inference — user guide](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Bedrock Prompt Management — user guide](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html)
- [Bedrock Prompt Management GA announcement (Nov 2024)](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-bedrock-prompt-management-available)
- [Amazon Bedrock Flows product page](https://aws.amazon.com/bedrock/flows/)
- [Bedrock Agents — action groups](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html)
- [Bedrock Agents — Lambda integration](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html)
- [Introducing Amazon Bedrock AgentCore (AWS Blog)](https://aws.amazon.com/blogs/aws/introducing-amazon-bedrock-agentcore-securely-deploy-and-operate-ai-agents-at-any-scale/)
- [Strands Agents SDK — technical deep dive (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/strands-agents-sdk-a-technical-deep-dive-into-agent-architectures-and-observability/)
- [Strands Agents — official site](https://strandsagents.com/)
- [MCP Proxy for AWS GA announcement (Oct 2025)](https://aws.amazon.com/about-aws/whats-new/2025/10/model-context-protocol-proxy-available)
- [Custom Model Import — user guide](https://docs.aws.amazon.com/bedrock/latest/userguide/import-pre-trained-model.html)
- [Bedrock FAQs (distillation, fine-tuning)](https://aws.amazon.com/bedrock/faqs/)
- [AIP-C01 official exam guide](https://docs.aws.amazon.com/aws-certification/latest/examguides/ai-professional-01.html)
- [Building an AI gateway to Bedrock with API Gateway (AWS Architecture Blog)](https://aws.amazon.com/blogs/architecture/building-an-ai-gateway-to-amazon-bedrock-with-amazon-api-gateway/)

---

*Part of the [AWS AI Certification course](../README.md) · AIP-C01 track.*
