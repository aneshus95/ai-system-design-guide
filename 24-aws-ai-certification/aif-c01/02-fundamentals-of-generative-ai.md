# Domain 2: Fundamentals of Generative AI

> **This is the single biggest domain on AIF-C01 — 24% of scored content**, more than any other. Master the vocabulary here (tokens, embeddings, transformers, diffusion, FM lifecycle) and the AWS "which service when" reflexes, and roughly 1 in 4 exam questions become easy points. This page teaches every Domain 2 task statement intuitively first, then gives you the exact AWS terms and the traps.

Sources: [AIF-C01 exam guide — Domain 2](https://docs.aws.amazon.com/aws-certification/latest/ai-practitioner-01/ai-practitioner-01-domain2.html) · [AIF-C01 exam guide (home)](https://docs.aws.amazon.com/aws-certification/latest/ai-practitioner-01/ai-practitioner-01.html)

---

## Table of Contents
- [How the official objectives map to this page](#objectives-map)
- [🧠 Task 2.1 — Foundational GenAI concepts](#concepts)
  - [Tokens & tokenization](#tokens)
  - [Embeddings & vectors](#embeddings)
  - [Chunking](#chunking)
  - [Transformers & self-attention](#transformers)
  - [Foundation models, LLMs, multi-modal & diffusion models](#fm-types)
  - [Prompt engineering & context engineering](#prompt-context)
  - [Agentic AI, MCP & multi-agent patterns](#agentic)
- [GenAI use cases](#use-cases)
- [The foundation model lifecycle](#fm-lifecycle)
- [🧠 Task 2.2 — Capabilities & limitations for business](#caps-limits)
  - [Advantages](#advantages)
  - [Disadvantages](#disadvantages)
  - [Factors for selecting a model](#selection-factors)
  - [Business value & metrics](#business-metrics)
- [⚙️ Task 2.3 — AWS infrastructure & technologies for GenAI](#aws-infra)
  - [Bedrock, SageMaker JumpStart, PartyRock, Amazon Q & the agent stack](#aws-services)
  - [Advantages of AWS GenAI services](#aws-advantages)
  - [Benefits of AWS infrastructure (security, compliance, safety)](#aws-benefits)
  - [Cost tradeoffs: token pricing & provisioned throughput](#cost-tradeoffs)
- [Exam traps & quick-fire review](#exam-traps)
- [References](#references)

---

## How the official objectives map to this page <a name="objectives-map"></a>

The official AIF-C01 guide breaks Domain 2 into three task statements. Here is the **verbatim** objective list mapped to where it's taught:

| Task | Official objective (from the exam guide) | Section |
|---|---|---|
| **2.1** | Define foundational GenAI concepts (tokens, chunking, embeddings, vectors, prompt engineering, transformer-based LLMs, FMs, multi-modal models, diffusion models) | [Concepts](#concepts) |
| **2.1** | Identify potential use cases (image/video/audio generation; summarization; AI assistants; translation; code generation; customer service agents; search; recommendation engines) | [Use cases](#use-cases) |
| **2.1** | Describe the FM lifecycle (data selection, model selection, pre-training, fine-tuning, evaluation, deployment, feedback) | [FM lifecycle](#fm-lifecycle) |
| **2.1** | Describe the token-based pricing model and its effect on cost and performance for inference | [Cost tradeoffs](#cost-tradeoffs) |
| **2.1** | Describe the role of **context engineering** in FM applications | [Prompt & context](#prompt-context) |
| **2.1** | Define foundational **agentic AI** concepts (multi-agent patterns, **Model Context Protocol [MCP]**, memory, tool usage, workflow orchestration) | [Agentic AI](#agentic) |
| **2.2** | Advantages of GenAI (adaptability, responsiveness, conversational capabilities, content generation) | [Advantages](#advantages) |
| **2.2** | Disadvantages (hallucinations, interpretability, inaccuracy, nondeterminism) | [Disadvantages](#disadvantages) |
| **2.2** | Factors to select a model (model types, performance, capabilities, constraints, compliance, cost, latency, complexity) | [Selection factors](#selection-factors) |
| **2.2** | Business value & metrics (cross-domain performance, ROI, efficiency, conversion rate, ARPU, accuracy, CLV) | [Business metrics](#business-metrics) |
| **2.3** | AWS services to develop GenAI apps (Bedrock, SageMaker AI, JumpStart, Amazon Q/Quick, Kiro, Strands Agents, Bedrock AgentCore) | [AWS services](#aws-services) |
| **2.3** | Advantages of AWS GenAI services (accessibility, lower barrier, efficiency, cost, speed to market, meeting objectives) | [AWS advantages](#aws-advantages) |
| **2.3** | Benefits of AWS infrastructure (security, compliance, responsibility, safety) | [AWS benefits](#aws-benefits) |
| **2.3** | Cost tradeoffs (responsiveness, availability, redundancy, performance, regional coverage, token pricing, provisioned throughput, custom models) | [Cost tradeoffs](#cost-tradeoffs) |

> **Note on freshness:** The AIF-C01 guide was updated to add **context engineering**, **agentic AI / MCP**, and the newer agent-building services (**Amazon Q / Quick, Kiro, Strands Agents, Bedrock AgentCore**). This page covers both the classic Domain 2 material and these additions so you're not surprised on exam day. Source: [Domain 2 exam guide](https://docs.aws.amazon.com/aws-certification/latest/ai-practitioner-01/ai-practitioner-01-domain2.html).

---

## 🧠 Task 2.1 — Foundational GenAI concepts <a name="concepts"></a>

**Plain English:** A **generative AI** model doesn't just classify or predict a number — it *generates new content* (text, images, audio, code). The workhorse is the **foundation model (FM)**: one very large model, pre-trained on enormous data, that you then *adapt* to many tasks. Everything in this section is the plumbing that lets an FM read your words, "understand" them as numbers, and write a coherent reply.

Here's the whole pipeline in one picture — memorize this flow, several questions ride on it:

```mermaid
flowchart LR
    A["Raw text:<br/>'The cat sat'"] -->|Tokenization| B["Tokens:<br/>The | cat | sat"]
    B -->|"Embedding lookup"| C["Vectors:<br/>numbers per token"]
    C -->|"Self-attention<br/>(Transformer)"| D["Context-aware<br/>vectors"]
    D -->|"Predict next token"| E["Output token:<br/>'quietly'"]
    E -->|"Loop / decode"| F["Generated text"]
```

### Tokens & tokenization <a name="tokens"></a>

**Plain English:** A model can't read letters or words directly. It first chops text into **tokens** — small chunks that are often whole words, but frequently *sub-words* or characters. A rough rule of thumb: **~1 token ≈ 0.75 English words**, or **~4 characters**.

| Term | Meaning | Example |
|---|---|---|
| **Token** | The smallest unit the model processes | `reading` → `read` + `ing` |
| **Tokenization** | Splitting text into tokens | `"unbelievable"` → `un`, `believ`, `able` |
| **Context window** | Max tokens (input + output) a model can handle at once | e.g. 8K, 200K, 1M tokens depending on model |
| **Input tokens** | Tokens in your prompt (you pay for these) | The question you send |
| **Output tokens** | Tokens the model generates (usually pricier) | The answer it returns |

> **Why tokens matter for the exam:** **Pricing is per token**, and separately for input vs output. Longer prompts and longer answers = more tokens = higher cost and higher latency. See [Cost tradeoffs](#cost-tradeoffs). Source: [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/).

### Embeddings & vectors <a name="embeddings"></a>

**Plain English:** An **embedding** turns a token (or a whole sentence/document) into a list of numbers — a **vector** — that captures *meaning*. Words with similar meaning land close together in this numeric space; unrelated words land far apart. "King − Man + Woman ≈ Queen" is the classic intuition.

```mermaid
flowchart TB
    subgraph Space["Embedding vector space (semantic map)"]
        K["king"]
        Q["queen"]
        M["man"]
        W["woman"]
        C["car"]
    end
    K -.near.- Q
    M -.near.- W
    K -.far.- C
```

| Term | Meaning |
|---|---|
| **Embedding** | A numeric vector representing meaning of text/image/audio |
| **Vector** | The list of numbers itself (e.g. 256, 512, 1024, or 1536 dimensions) |
| **Dimensions** | How many numbers in the vector — more = richer, but bigger/costlier |
| **Vector database** | A store optimized to find "nearest" (most similar) vectors fast |
| **Semantic search** | Retrieval by *meaning* (vector similarity) instead of keyword match |

- **On AWS**, the go-to embedding models are **Amazon Titan Text Embeddings** (V2 outputs **256, 512, or 1024** dimensions; V1 outputs **1536**) and **Amazon Titan Multimodal Embeddings**. These are used heavily in **RAG** and search. Source: [Amazon Titan Text Embeddings models](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html).
- Vector stores commonly paired with AWS GenAI: **Amazon OpenSearch Service**, **Amazon Aurora PostgreSQL / pgvector**, **Amazon Neptune Analytics**, and **Amazon Kendra** for managed retrieval.

> **Exam reflex:** "Convert text into numbers that capture meaning for similarity search" → **embeddings / a vector**. "Find documents by meaning, not keywords" → **semantic search over a vector store**.

### Chunking <a name="chunking"></a>

**Plain English:** Big documents don't fit in one embedding (or one context window). **Chunking** splits a long document into smaller passages *before* embedding, so each chunk becomes its own vector. When a user asks a question, you retrieve only the most relevant chunks. This is a core step in **Retrieval-Augmented Generation (RAG)**.

```mermaid
flowchart LR
    D["Long PDF /<br/>knowledge base"] -->|Chunking| C1["Chunk 1"] & C2["Chunk 2"] & C3["Chunk 3"]
    C1 & C2 & C3 -->|Embed| V["Vectors stored in<br/>vector database"]
    Q["User question"] -->|Embed + search| V
    V -->|Top-k relevant chunks| P["Prompt to FM + answer"]
```

- **Trade-off:** chunks too **large** → dilute relevance and waste tokens; too **small** → lose context. Overlap between chunks preserves continuity.
- **On AWS**, **Amazon Bedrock Knowledge Bases** handles chunking, embedding, storage, and retrieval for you (fixed-size, semantic, hierarchical chunking options).

### Transformers & self-attention <a name="transformers"></a>

**Plain English:** The **transformer** is the neural-network architecture behind virtually every modern LLM (the "T" in GPT = Transformer). Its superpower is **self-attention**: when processing each token, the model *looks at every other token* and decides which ones matter for meaning. That's how it resolves "it" → the right noun, or picks the right meaning of "bank" (river vs money) from surrounding words.

**The intuition of self-attention (Query / Key / Value):**

Think of each token as attending a meeting:
- **Query (Q):** "Here's what *I'm* looking for."
- **Key (K):** "Here's what *I* can offer."
- **Value (V):** "Here's the actual info I carry."

Each token's Query is compared against every token's Key. High match = high **attention weight** = that token's Value strongly influences the result. Do this for all tokens, in parallel, across multiple **attention heads** (each head learns a different kind of relationship), stack many layers, and you get deep contextual understanding.

```mermaid
flowchart TB
    T["Token: 'it'"] -->|emits Query| ATT{"Attention:<br/>compare Query<br/>to every Key"}
    K1["'animal' (Key+Value)"] --> ATT
    K2["'street' (Key+Value)"] --> ATT
    K3["'tired' (Key+Value)"] --> ATT
    ATT -->|"high weight on 'animal'"| R["'it' understood as<br/>the animal"]
```

**Why transformers beat older RNNs (exam-relevant):**

| Property | RNN / LSTM (older) | Transformer |
|---|---|---|
| Processing | Sequential, one token at a time | **Parallel** — all tokens at once |
| Long-range context | Weak (forgets distant words) | **Strong** (attends to any position) |
| Training speed | Slow | **Fast on GPUs** — enabled today's huge FMs |
| Basis of | Legacy NLP | **All modern LLMs / FMs** |

> **Exam reflex:** "Which architecture underlies modern LLMs?" → **Transformer**. "Mechanism that weighs the importance of other words when encoding a word?" → **(self-)attention**. Source (intuition): [Transformer architecture & self-attention — Codecademy](https://www.codecademy.com/article/transformer-architecture-self-attention-mechanism).

### Foundation models, LLMs, multi-modal & diffusion models <a name="fm-types"></a>

**Plain English:** A **foundation model (FM)** is a large, general-purpose model pre-trained on massive data that can be adapted to *many* downstream tasks. Not all FMs are text; not all are transformers.

| Model type | What it is | Modality | Example on AWS |
|---|---|---|---|
| **Foundation model (FM)** | Large pre-trained, general-purpose model you adapt | Any | Amazon Titan, Claude, Llama, Mistral (via Bedrock) |
| **Large language model (LLM)** | An FM specialized in text/language (transformer-based) | Text in → text out | Anthropic Claude, Meta Llama, Amazon Nova |
| **Multi-modal model** | Handles *more than one* modality (e.g. text + image) | Mixed in/out | Amazon Nova, Titan Multimodal Embeddings, Claude (vision) |
| **Diffusion model** | Generates images by starting from noise and *removing* it step by step, guided by a prompt | Text → image/video | Stable Diffusion, Amazon Nova Canvas (image), Nova Reel (video) |

**How a diffusion model works (intuition):**

```mermaid
flowchart LR
    N["Pure random noise"] -->|"Step 1 denoise"| S1["Blurry shapes"]
    S1 -->|"Step 2..."| S2["Clearer image"]
    S2 -->|"Final step"| IMG["Sharp image<br/>matching the prompt"]
    P["Text prompt:<br/>'a red bicycle'"] -.guides every step.-> S1 & S2 & IMG
```

**Plain English (diffusion):** Training teaches the model to *add* noise to real images and then *reverse* it. At generation time it starts from static and gradually "denoises" toward an image that matches your text prompt. This is why image generators are diffusion models, whereas chatbots are transformer LLMs.

> **Exam trap:** Don't assume "foundation model = LLM = transformer." **Diffusion models are FMs but are NOT LLMs and are used for images/video, not text chat.** If you see "generate an image from text," think **diffusion model** (e.g. Stable Diffusion / Nova Canvas), not an LLM.

### Prompt engineering & context engineering <a name="prompt-context"></a>

**Plain English:** **Prompt engineering** = crafting the *instructions* you send so the model does what you want. **Context engineering** (newer objective) = the broader discipline of assembling *everything the model sees* at inference time — instructions **plus** retrieved documents, tool outputs, memory, and examples — so the model has the right information to answer well.

| Technique | What it does |
|---|---|
| **Zero-shot** | Ask directly, no examples |
| **Few-shot** | Include a few worked examples to steer format/behavior |
| **Chain-of-thought** | Ask the model to reason step by step |
| **System prompt** | Set persona, rules, and constraints up front |
| **Context engineering** | Curate the *whole* context: RAG chunks + tools + memory + prompt, within the token budget |

> **Exam reflex:** "Improve answers *without* retraining the model, just by changing what you send it" → **prompt engineering / context engineering** (cheapest lever). Deep dive lives in Domain 3 — see `03-applications-of-foundation-models.md`.

### Agentic AI, MCP & multi-agent patterns <a name="agentic"></a>

**Plain English:** An **AI agent** is an FM that can *act*, not just chat — it can call **tools/APIs**, remember state (**memory**), and take multiple steps to reach a goal (**workflow orchestration**). **Agentic AI** is building systems out of one or more such agents.

| Concept | Plain English |
|---|---|
| **Tool use** | The agent calls an external function/API (search, calculator, database) |
| **Memory** | The agent retains context across turns/sessions (short- and long-term) |
| **Workflow orchestration** | Breaking a goal into steps and coordinating them |
| **Multi-agent system** | Several specialized agents collaborate (e.g. planner + researcher + writer) |
| **Model Context Protocol (MCP)** | An **open standard** (originated by Anthropic) that gives agents a *universal way* to connect to external tools, data, and services |

**MCP in one line:** it's the "USB-C for AI tools" — one standard protocol so any MCP-compatible agent can plug into any MCP-compatible data source or tool.

```mermaid
flowchart LR
    U["User goal"] --> AG["Agent (FM)"]
    AG -->|"MCP client"| MCP{"Model Context<br/>Protocol"}
    MCP --> T1["Tool: database"]
    MCP --> T2["Tool: web search"]
    MCP --> T3["Tool: internal API"]
    AG <-->|"memory"| MEM["Short/long-term memory"]
```

- **On AWS**, agents are built with **Amazon Bedrock Agents / Bedrock AgentCore**, **Strands Agents** (an open-source agent SDK), and orchestrated experiences via **Amazon Q**. Source: [Harness MCP servers with Amazon Bedrock Agents](https://aws.amazon.com/blogs/machine-learning/harness-the-power-of-mcp-servers-with-amazon-bedrock-agents/) · [MCP on AWS](https://aws.amazon.com/blogs/machine-learning/unlocking-the-power-of-model-context-protocol-mcp-on-aws/).

> **Exam reflex:** "Standard protocol for connecting an agent to external tools/data" → **MCP**. "FM that plans, calls tools, and keeps state to complete a task" → **agent / agentic AI**.

---

## GenAI use cases <a name="use-cases"></a>

**Plain English:** The exam wants you to recognize *what problems GenAI is good for*. Match the scenario to the capability.

| Use case | What GenAI does | Typical AWS pairing |
|---|---|---|
| **Image / video / audio generation** | Create media from a prompt | Bedrock (Stable Diffusion, Nova Canvas/Reel) |
| **Summarization** | Condense long text | Any LLM via Bedrock |
| **Chatbots / AI assistants** | Conversational Q&A | Bedrock, Amazon Q |
| **Translation** | Language → language | LLMs; Amazon Translate for classic MT |
| **Code generation** | Write/fix/explain code | **Amazon Q Developer**, **Kiro** |
| **Customer-service agents** | Automate support with tools + knowledge | Bedrock Agents, Amazon Q, Amazon Connect |
| **Search** | Semantic / conversational search | Embeddings + vector store, Amazon Kendra |
| **Recommendation engines** | Personalized suggestions | FMs + Amazon Personalize |

> **Exam trap:** GenAI is *not* the answer to everything. For a **numeric prediction from tabular data** (forecast sales, detect fraud from features) the right tool is **classic ML** (regression/classification), not a generative FM — that reasoning is tested in Domain 1.

---

## The foundation model lifecycle <a name="fm-lifecycle"></a>

**Plain English:** An FM isn't "trained once and done." It moves through a lifecycle: pick data → pick a model → pre-train (huge, expensive, usually done by the model provider) → fine-tune/adapt → evaluate → deploy → gather feedback → improve. Know the **order**; ordering-type questions love this.

```mermaid
flowchart LR
    A["1. Data selection"] --> B["2. Model selection"]
    B --> C["3. Pre-training"]
    C --> D["4. Fine-tuning /<br/>customization"]
    D --> E["5. Evaluation"]
    E --> F["6. Deployment"]
    F --> G["7. Feedback"]
    G -->|"iterate"| D
```

| Stage | What happens | Who usually does it |
|---|---|---|
| **Data selection** | Choose large, high-quality, diverse training data | Model provider |
| **Model selection** | Pick architecture/size for the goal | Provider (or you, choosing which FM to adopt) |
| **Pre-training** | Learn general language/vision from raw data — **very costly, GPU-heavy** | Provider (e.g. Anthropic, Meta, Amazon) |
| **Fine-tuning / customization** | Adapt the FM to your task/domain data | **You** (Bedrock custom models, SageMaker) |
| **Evaluation** | Measure quality, accuracy, bias, safety | You (Bedrock model evaluation) |
| **Deployment** | Serve for inference | You (Bedrock, SageMaker endpoints) |
| **Feedback** | Collect real usage, human feedback (**RLHF**), errors → refine | You |

> **Exam reflex:** As a **practitioner**, you rarely **pre-train** (too costly). You **select** an existing FM and **fine-tune / customize** it or use **RAG / prompt engineering**. "Cheapest way to adapt behavior" → prompt engineering; "adapt to private data at query time" → **RAG**; "bake domain knowledge/style into the weights" → **fine-tuning**.

---

## 🧠 Task 2.2 — Capabilities & limitations for business <a name="caps-limits"></a>

### Advantages <a name="advantages"></a>

**Plain English:** Why businesses reach for GenAI.

| Advantage | Meaning |
|---|---|
| **Adaptability** | One FM handles many tasks (write, summarize, translate, code) |
| **Responsiveness** | Fast, on-demand generation and interaction |
| **Conversational capabilities** | Natural-language interface — no special skills needed to use it |
| **Ability to generate content** | Creates *new* text/images/code, not just classifications |
| **Simplicity** | Lowers effort to build capable apps (few lines / no code) |

### Disadvantages <a name="disadvantages"></a>

**Plain English:** The failure modes AWS wants you to name.

| Disadvantage | Plain English | Mitigation (know these!) |
|---|---|---|
| **Hallucinations** | Confidently states false info | **RAG**, grounding, citations, human review |
| **Interpretability** | Hard to explain *why* it answered as it did (black box) | Explainability tooling, guardrails, logging |
| **Inaccuracy** | Output may be wrong or outdated | RAG with fresh data, evaluation, guardrails |
| **Nondeterminism** | Same prompt can give different answers each time | Lower **temperature**, set **top-p / top-k**, fixed seeds |

> **Exam reflex:** "Model invents plausible but false facts" → **hallucination**; fix with **RAG / grounding**. "Different output each run" → **nondeterminism**; reduce by lowering **temperature**. "Can't explain the decision" → **interpretability / lack of explainability**.

### Factors for selecting a model <a name="selection-factors"></a>

**Plain English:** Choosing an FM is a trade-off, not "biggest is best."

```mermaid
flowchart TB
    G(("Pick an FM")) --> T["Model type<br/>(text? image? multimodal?)"]
    G --> P["Performance / quality<br/>needed"]
    G --> C["Capabilities<br/>(context window, languages, vision)"]
    G --> K["Constraints<br/>(latency, throughput)"]
    G --> M["Compliance & data residency"]
    G --> $["Cost per token"]
    G --> X["Model complexity / size"]
```

| Factor | Question to ask |
|---|---|
| **Model type** | Do I need text, image, audio, or multi-modal? |
| **Performance requirements** | How accurate / high-quality must outputs be? |
| **Capabilities** | Context window size, languages, reasoning, vision? |
| **Constraints** | Latency and throughput limits? |
| **Compliance** | Data residency, regulatory, privacy needs? |
| **Cost** | Per-token price; a smaller model may be "good enough" and far cheaper |
| **Latency / complexity** | Bigger models are smarter but slower and pricier |

> **Exam reflex:** "Need low latency and low cost, task is simple" → pick a **smaller/cheaper FM**. "Need strict data residency / regulated industry" → **compliance** drives the choice (region, private models, no data used for training).

### Business value & metrics <a name="business-metrics"></a>

**Plain English:** GenAI value is proven with *business* metrics, not just model accuracy.

| Metric | Meaning |
|---|---|
| **Cross-domain performance** | How well one model performs across varied tasks/domains |
| **ROI (return on investment)** | Value gained vs cost of building/running |
| **Efficiency** | Time/effort/cost saved (e.g. faster resolution) |
| **Conversion rate** | % of users who take the desired action |
| **Average revenue per user (ARPU)** | Revenue ÷ number of users |
| **Accuracy** | Correctness of outputs |
| **Customer lifetime value (CLV / CLTV)** | Total revenue expected from a customer over time |

> **Exam trap:** Distinguish **model/technical metrics** (accuracy, BLEU, ROUGE, perplexity) from **business metrics** (ROI, conversion rate, ARPU, CLV, efficiency). A question asking for **business value** wants the *business* metric.

---

## ⚙️ Task 2.3 — AWS infrastructure & technologies for GenAI <a name="aws-infra"></a>

### Bedrock, SageMaker JumpStart, PartyRock, Amazon Q & the agent stack <a name="aws-services"></a>

**Plain English:** AWS gives you a ladder — from *no-code playground* up to *full model control*. Pick the lowest rung that meets your need.

```mermaid
flowchart TB
    PR["PartyRock<br/>(no-code playground)"] --> BR["Amazon Bedrock<br/>(managed FMs via one API)"]
    BR --> JS["SageMaker JumpStart<br/>(deploy/customize models,<br/>more control)"]
    JS --> SM["Amazon SageMaker AI<br/>(full build/train/deploy)"]
    Q["Amazon Q<br/>(ready-made AI assistant)"] -.uses FMs.- BR
```

| Service | What it is | Reach for it when… | Deep dive |
|---|---|---|---|
| **Amazon Bedrock** | Fully managed, **serverless** access to many FMs (Amazon, Anthropic, Meta, Mistral, Cohere, Stability AI, and more) through **one API**; includes Knowledge Bases (RAG), Agents, Guardrails, model evaluation, fine-tuning | You want to build a GenAI app fast **without managing infrastructure** | [`../services/bedrock.md`](../services/bedrock.md) |
| **PartyRock (Amazon Bedrock Playground)** | **No-code** web playground to build/share small GenAI apps; free daily usage, no AWS account/credit card needed to start | Learning, prototyping, demos before writing code | [`../services/bedrock.md`](../services/bedrock.md) |
| **SageMaker JumpStart** | An **ML hub** inside SageMaker: browse, **one-click deploy**, and **fine-tune** open and proprietary **foundation models** and built-in algorithms | You need **more control** — self-hosted endpoints, deeper customization, or specific open models | [`../services/sagemaker.md`](../services/sagemaker.md) |
| **Amazon SageMaker AI** | Full ML platform to build, train, tune, and deploy models end to end | You're doing serious custom ML / MLOps | [`../services/sagemaker.md`](../services/sagemaker.md) |
| **Amazon Q** | A ready-made **generative AI assistant**. **Q Business** = answers from your enterprise data/docs; **Q Developer** = AI coding help in the IDE and across AWS | You want a turnkey assistant, not to build one | [`../services/amazon-q.md`](../services/amazon-q.md) |
| **Bedrock Agents / AgentCore, Strands Agents, Kiro** | Tools to build **agents** (tool use, memory, orchestration) and agentic dev workflows | You need agents that *act*, call tools, and run multi-step tasks | [`../services/bedrock.md`](../services/bedrock.md) |

**Key distinctions the exam loves:**

| If you see… | Pick… |
|---|---|
| "No infrastructure to manage, choose from many FMs via one API" | **Amazon Bedrock** |
| "No-code / free playground to experiment with GenAI" | **PartyRock** |
| "One-click deploy or fine-tune open-source/proprietary FMs, self-hosted endpoint, more control" | **SageMaker JumpStart** |
| "Enterprise assistant that answers from *our own documents*" | **Amazon Q Business** |
| "AI pair-programmer in the IDE" | **Amazon Q Developer** |
| "Build a custom model from scratch / full MLOps" | **Amazon SageMaker AI** |

Sources: [SageMaker JumpStart foundation models](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models.html) · [SageMaker JumpStart product page](https://aws.amazon.com/sagemaker/ai/jumpstart/) · [Bedrock or SageMaker decision guide](https://docs.aws.amazon.com/decision-guides/latest/bedrock-or-sagemaker/bedrock-or-sagemaker.html) · [Amazon Bedrock pricing/overview](https://aws.amazon.com/bedrock/pricing/).

### Advantages of AWS GenAI services <a name="aws-advantages"></a>

| Advantage | Plain English |
|---|---|
| **Accessibility** | Powerful FMs reachable via a simple API / console |
| **Lower barrier to entry** | No need to train models or run GPUs yourself |
| **Efficiency** | Managed services remove undifferentiated heavy lifting |
| **Cost-effectiveness** | Pay-as-you-go; no upfront model-training spend |
| **Speed to market** | Ship GenAI features in days, not months |
| **Ability to meet business objectives** | Focus effort on the app, not the infrastructure |

### Benefits of AWS infrastructure (security, compliance, safety) <a name="aws-benefits"></a>

**Plain English:** Beyond convenience, AWS supplies the *guardrails* enterprises need.

| Benefit | How AWS delivers it |
|---|---|
| **Security** | Data encrypted in transit/at rest; **IAM** access control; private networking (**VPC / PrivateLink**); **your data is not used to train the base FMs** |
| **Compliance** | Broad certifications and regional controls; data residency choices |
| **Responsibility** | **Bedrock Guardrails** to filter harmful content and block sensitive topics/PII; the **AWS shared responsibility model** |
| **Safety** | Content filters, denied topics, contextual grounding checks, watermarking on generated images (Nova/Titan) |

> **Exam reflex:** "Block toxic content / PII / off-topic answers from a GenAI app" → **Amazon Bedrock Guardrails**. "Who secures what?" → **shared responsibility model** (AWS secures the cloud; you secure what you put in it). Detail in Domain 5 — `05-security-compliance-and-governance.md`.

### Cost tradeoffs: token pricing & provisioned throughput <a name="cost-tradeoffs"></a>

**Plain English:** GenAI cost is driven mostly by **tokens** and by **how you buy capacity**. You balance cost against **responsiveness, availability, redundancy, performance, and regional coverage**.

**1) Token-based (On-Demand) pricing — pay per token, nothing reserved.**

- You pay separately for **input tokens** and **output tokens**; **output is usually more expensive**.
- Price varies by **model**, **modality**, and **region**.
- No commitment — ideal for **spiky, low, or unpredictable** traffic.

**Worked example (illustrative rates — always check the live pricing page):**

> Suppose a model costs **\$3.00 per 1M input tokens** and **\$15.00 per 1M output tokens**. A chatbot request with a **1,500-token prompt** and a **500-token answer** costs:
> - Input: 1,500 / 1,000,000 × \$3.00 = **\$0.0045**
> - Output: 500 / 1,000,000 × \$15.00 = **\$0.0075**
> - **Total ≈ \$0.012 per request.** At 1,000,000 requests/month ≈ **\$12,000/month**.
>
> Takeaways the exam tests: shorter prompts/answers cut cost and **latency**; **output tokens dominate** when answers are long; a **smaller/cheaper model** can slash cost if quality is still acceptable.

**2) Provisioned Throughput — reserve capacity for a fixed hourly price.**

- You buy **Model Units (MUs)**; each MU guarantees a **fixed number of input/output tokens per minute** for a specific model.
- Billed **hourly**, whether or not you use the capacity.
- Commitment terms: **No commitment** (delete anytime), **1 month**, or **6 months** — **longer commitment = bigger discount**.
- **Required** to serve a **custom (fine-tuned) model**.
- Best for **steady, high-volume, predictable** workloads that need **guaranteed throughput / consistent latency**.

Sources: [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/) · [Provisioned Throughput (Bedrock docs)](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html).

| | **On-Demand (token-based)** | **Provisioned Throughput** |
|---|---|---|
| Billing | Per input/output token | Per **Model Unit**, **hourly** |
| Commitment | None | None / 1 month / 6 months (longer = cheaper) |
| Best for | Spiky, low, unpredictable traffic; prototyping | Steady, high-volume, predictable; guaranteed capacity |
| Latency/throughput | Shared, can vary | Reserved, consistent |
| Custom models | — | **Required for custom/fine-tuned models** |
| Batch option | **Batch inference ~50% cheaper** for large offline jobs | — |

**Other cost/architecture tradeoffs to recognize:**

| Lever | Trade-off |
|---|---|
| **Responsiveness (latency)** | Faster responses may need bigger instances / provisioned throughput → higher cost |
| **Availability & redundancy** | Multi-AZ / multi-region raises resilience **and** cost |
| **Performance** | Larger models = better quality but slower + pricier |
| **Regional coverage** | Not every model is in every Region; prices differ by Region |
| **Custom models** | Fine-tuning + hosting custom models adds cost (and needs provisioned throughput) |

> **Exam reflex:** "Unpredictable / low volume, prototype" → **On-Demand token pricing**. "High steady volume, need guaranteed throughput & consistent latency (or serving a custom model)" → **Provisioned Throughput**. "Huge one-off offline job, cost-sensitive, latency doesn't matter" → **Batch inference (~50% off)**.

---

## Exam traps & quick-fire review <a name="exam-traps"></a>

| # | Trap / Question shape | Correct reflex |
|---|---|---|
| 1 | "Underlying architecture of modern LLMs?" | **Transformer** (self-attention) |
| 2 | "Turn text into numbers that capture meaning for similarity search" | **Embeddings / vectors** |
| 3 | "Generate an image from a text prompt" | **Diffusion model** (Stable Diffusion / Nova Canvas) — **not** an LLM |
| 4 | "Model states false facts confidently" | **Hallucination** → mitigate with **RAG / grounding** |
| 5 | "Same prompt, different answers each time" | **Nondeterminism** → lower **temperature** |
| 6 | "Can't explain why the model answered that way" | **Interpretability / explainability** limitation |
| 7 | "No-code, free playground to try GenAI" | **PartyRock** |
| 8 | "Many FMs via one API, no infra to manage" | **Amazon Bedrock** |
| 9 | "One-click deploy / fine-tune open FMs with more control" | **SageMaker JumpStart** |
| 10 | "Enterprise assistant answering from our own docs" | **Amazon Q Business** |
| 11 | "AI coding assistant in the IDE" | **Amazon Q Developer** |
| 12 | "Order the FM lifecycle" | data → model selection → pre-train → fine-tune → evaluate → deploy → feedback |
| 13 | "Business value metric (not technical)" | **ROI / conversion rate / ARPU / CLV / efficiency** |
| 14 | "Spiky/unpredictable traffic, pay only for use" | **On-Demand token pricing** |
| 15 | "Steady high volume, guaranteed throughput, or serving a custom model" | **Provisioned Throughput** (1- or 6-month = cheaper) |
| 16 | "Block toxic content / PII in a GenAI app" | **Amazon Bedrock Guardrails** |
| 17 | "Standard way to connect an agent to external tools/data" | **Model Context Protocol (MCP)** |
| 18 | "Cheapest way to change model behavior without retraining" | **Prompt / context engineering** |
| 19 | "Split a long document before embedding for RAG" | **Chunking** |
| 20 | "FM that plans, calls tools, keeps memory to finish a task" | **Agent / agentic AI** (Bedrock Agents / AgentCore) |

**One-screen summary:**

```mermaid
mindmap
  root((Domain 2<br/>GenAI 24%))
    2.1 Concepts
      Tokens & tokenization
      Embeddings & vectors
      Chunking / RAG
      Transformer + self-attention
      FMs / LLMs / multimodal / diffusion
      Prompt & context engineering
      Agents / MCP
      FM lifecycle
    2.2 Caps & limits
      Advantages: adaptability, responsiveness
      Disadvantages: hallucination, nondeterminism, interpretability
      Selection factors: type, cost, latency, compliance
      Business metrics: ROI, ARPU, CLV, conversion
    2.3 AWS infra
      Bedrock / JumpStart / PartyRock / Amazon Q
      Advantages: accessibility, speed to market
      Benefits: security, compliance, safety, guardrails
      Cost: token pricing vs provisioned throughput
```

---

## References <a name="references"></a>

All sources are official AWS documentation unless noted (noted items are used only for *intuition*, not for AWS facts):

- [AIF-C01 Exam Guide — home](https://docs.aws.amazon.com/aws-certification/latest/ai-practitioner-01/ai-practitioner-01.html)
- [AIF-C01 Exam Guide — Domain 2 task statements (verbatim)](https://docs.aws.amazon.com/aws-certification/latest/ai-practitioner-01/ai-practitioner-01-domain2.html)
- [Amazon Bedrock — Pricing (token-based, batch, provisioned throughput)](https://aws.amazon.com/bedrock/pricing/)
- [Amazon Bedrock — Provisioned Throughput & Model Units](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html)
- [Amazon Titan Text Embeddings models (dimensions: 256/512/1024, V1 = 1536)](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)
- [Amazon Titan Multimodal Embeddings G1](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-multiemb-models.html)
- [Amazon SageMaker JumpStart — Foundation Models](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models.html)
- [Amazon SageMaker JumpStart — product page (one-click deploy)](https://aws.amazon.com/sagemaker/ai/jumpstart/)
- [Decision guide: Amazon Bedrock or Amazon SageMaker AI?](https://docs.aws.amazon.com/decision-guides/latest/bedrock-or-sagemaker/bedrock-or-sagemaker.html)
- [Unlocking the power of Model Context Protocol (MCP) on AWS](https://aws.amazon.com/blogs/machine-learning/unlocking-the-power-of-model-context-protocol-mcp-on-aws/)
- [Harness MCP servers with Amazon Bedrock Agents](https://aws.amazon.com/blogs/machine-learning/harness-the-power-of-mcp-servers-with-amazon-bedrock-agents/)
- [Model Context Protocol — official spec (Anthropic-originated open standard)](https://modelcontextprotocol.io/docs/getting-started/intro)
- Intuition only (not AWS facts): [Transformer architecture & self-attention — Codecademy](https://www.codecademy.com/article/transformer-architecture-self-attention-mechanism)

> **Internal deep-dives:** [Bedrock](../services/bedrock.md) · [Amazon Q](../services/amazon-q.md) · [SageMaker](../services/sagemaker.md) · Back to [Exam Overview & Study Plan](../00-exam-overview-and-study-plan.md)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Generative AI (GenAI)** | AI that creates new content — text, images, audio, or code — rather than just classifying existing content | The core technology this entire domain covers |
| **Foundation Model (FM)** | A large model pre-trained on enormous data that can be adapted to many different tasks with little or no extra training | The basis for all GenAI services like Bedrock |
| **Large Language Model (LLM)** | An FM specialized in text — it reads and writes natural language using a transformer architecture | Powers chatbots, summarization, translation, and code generation |
| **Token** | The smallest unit of text a model processes — roughly 0.75 English words or 4 characters | Pricing and context window limits are measured in tokens |
| **Tokenization** | Splitting text into tokens before feeding it to a model | Necessary because models work with numbers, not raw text |
| **Context window** | The maximum number of tokens (input + output combined) a model can handle in one request | Sets the limit on how much text you can send; larger = more expensive |
| **Input tokens** | Tokens that make up the prompt you send to the model | Billed at the input-token rate, usually cheaper than output |
| **Output tokens** | Tokens the model generates in its response | Usually priced higher than input tokens |
| **Embedding** | A list of numbers (a vector) that represents the meaning of a piece of text or other content | Captures semantic meaning so similar content lands close together in numeric space |
| **Vector** | The list of numbers that forms an embedding | The raw mathematical representation the model uses internally |
| **Dimensions** | How many numbers are in a vector — higher means richer but larger and costlier | Affects storage size, search speed, and semantic richness |
| **Vector database** | A database optimized to find the nearest (most similar) vectors quickly | Enables semantic search and is the storage layer in RAG pipelines |
| **Semantic search** | Retrieval based on meaning rather than exact keyword matching | Lets users find relevant content even when the wording differs |
| **Amazon Titan Text Embeddings** | AWS embedding models that output 256, 512, or 1024 dimensions (V2) or 1536 (V1) | The primary AWS embedding model used in Bedrock Knowledge Bases |
| **Chunking** | Splitting a long document into smaller passages before embedding each piece | Makes large documents searchable by meaning within a RAG pipeline |
| **Retrieval-Augmented Generation (RAG)** | A technique that retrieves relevant document chunks at query time and includes them in the prompt | Grounds model answers in current private documents without retraining |
| **Transformer** | The neural network architecture underlying virtually all modern LLMs | Enables parallel processing and long-range context understanding |
| **Self-attention** | The mechanism inside a transformer that lets each token consider every other token when computing its meaning | How the model resolves ambiguity and understands context |
| **Query / Key / Value (Q/K/V)** | The three components of the attention mechanism — Q asks, K advertises, V carries information | The mathematical building blocks of self-attention |
| **Attention head** | One parallel self-attention computation within a transformer layer | Multiple heads learn different types of relationships simultaneously |
| **RNN / LSTM** | Older sequential neural network architectures that process text one token at a time | Replaced by transformers for most NLP tasks due to speed and context limitations |
| **Multi-modal model** | A model that can process more than one type of input or output, such as text and images | Used when applications need to read images, generate visuals, or handle mixed content |
| **Diffusion model** | A model that generates images by starting from random noise and progressively removing it guided by a prompt | Used for text-to-image and text-to-video generation; not the same as an LLM |
| **Prompt engineering** | Crafting the instructions sent to a model to steer its output | The cheapest way to change model behavior without any retraining |
| **Context engineering** | Assembling everything the model sees at inference time — instructions, retrieved docs, tool outputs, and memory | Maximizes model performance by providing the right information within the token budget |
| **Zero-shot** | Prompting a model with no worked examples | Tests the model's built-in knowledge; works for common tasks |
| **Few-shot** | Including several worked examples in the prompt to teach the model a format or behavior | Improves performance on nuanced or unusual tasks |
| **Chain-of-thought (CoT)** | Asking the model to reason step by step before giving a final answer | Improves accuracy on multi-step math, logic, and reasoning problems |
| **System prompt** | Instructions placed at the start of a conversation to set persona, rules, and constraints | Sets the guardrails and role of the model for the entire session |
| **AI agent** | A foundation model equipped with tools, memory, and a goal so it can take multi-step actions | Enables autonomous task completion beyond simple question-and-answer |
| **Tool use** | An agent's ability to call external functions, APIs, or databases during a task | Connects the model to real-world data and actions |
| **Memory (agent)** | Stored context that an agent retains across turns or sessions | Enables coherent multi-turn interactions and long-running workflows |
| **Workflow orchestration** | Breaking a goal into steps and coordinating them across tools and agents | The planning capability that makes agents useful for complex tasks |
| **Multi-agent system** | Several specialized agents collaborating — e.g., a planner, researcher, and writer | Handles tasks too complex for a single agent |
| **Model Context Protocol (MCP)** | An open standard for connecting agents to external tools and data sources in a universal way | The "USB-C for AI tools" — any MCP agent can plug into any MCP server |
| **Amazon Bedrock** | AWS managed serverless service giving access to many FMs from multiple providers through one API | The primary AWS platform for building GenAI apps without managing infrastructure |
| **Amazon Bedrock Agents** | Bedrock feature for building agents that reason, call tools, and use knowledge bases | Used when the model needs to act, not just answer |
| **Bedrock AgentCore** | AWS service for deploying, managing, and scaling production agents | Provides the infrastructure layer for production agentic applications |
| **Strands Agents** | AWS open-source agent SDK for building custom agentic workflows | Gives developers programmatic control over agent behavior |
| **PartyRock** | AWS no-code web playground for experimenting with GenAI apps for free | Used for learning and demos before writing code |
| **SageMaker JumpStart** | SageMaker ML hub for one-click deploying and fine-tuning open and proprietary FMs | Offers more control than Bedrock, less than full SageMaker training |
| **Amazon Q** | AWS ready-made AI assistant; Q Business answers from enterprise docs, Q Developer helps with code | A turnkey AI assistant; no need to build one |
| **Amazon Q Business** | Amazon Q variant that answers questions from your private enterprise documents | Used for internal knowledge search without custom FM development |
| **Amazon Q Developer** | Amazon Q variant that acts as an AI coding assistant in the IDE | Helps write, debug, and understand code |
| **Kiro** | AWS AI developer tool for spec-driven and agentic software development workflows | An emerging GenAI coding and planning assistant |
| **Hallucination** | When a model confidently states false or invented information | A key limitation of LLMs; mitigated with RAG, grounding, and human review |
| **Nondeterminism** | The property that the same prompt can produce different outputs each run | Managed by lowering temperature or setting top-p/top-k constraints |
| **Interpretability** | How easily you can understand why a model produced a specific output | A key limitation of large black-box models |
| **Temperature** | A parameter that controls how random or creative the model's output is | Low temperature = deterministic and focused; high = creative and varied |
| **Top-K** | A parameter limiting token sampling to the K most likely next tokens | Narrows the candidate pool; lower K = safer, more predictable output |
| **Top-P (nucleus sampling)** | A parameter limiting token sampling to the smallest set whose cumulative probability reaches P | Dynamically adjusts the candidate pool based on probability mass |
| **Adaptability** | An FM's ability to handle many different types of tasks without separate models | A key business advantage of foundation models over narrow ML |
| **Cross-domain performance** | How well a single model performs across varied tasks and subject areas | A business metric for evaluating FM generalization |
| **ROI (Return on Investment)** | Value gained from a GenAI project minus its total cost | The primary business metric for justifying GenAI investment |
| **ARPU (Average Revenue Per User)** | Total revenue divided by the number of users | A business metric used to measure GenAI's impact on monetization |
| **CLV / CLTV (Customer Lifetime Value)** | Total revenue expected from a customer over their entire relationship | Business metric for measuring long-term GenAI impact on retention |
| **Conversion rate** | Percentage of users who take a desired action | Business metric for measuring GenAI impact on sales or engagement |
| **On-Demand pricing** | Pay-per-token Bedrock pricing with no capacity commitment | Best for variable, low, or unpredictable traffic |
| **Provisioned Throughput** | Bedrock pricing that reserves model capacity at a fixed hourly rate per Model Unit | Required for custom models; best for steady high-volume workloads |
| **Model Unit (MU)** | The capacity unit purchased for Provisioned Throughput; guarantees a fixed token rate per minute | Defines how much guaranteed capacity you have for a specific model |
| **Batch inference** | Bedrock option for running large offline jobs at roughly 50% lower cost than on-demand | Best for bulk scoring where latency does not matter |
| **IAM (Identity and Access Management)** | AWS service controlling who can call which APIs and access which resources | Always the customer's responsibility, even with fully managed services |
| **VPC (Virtual Private Cloud)** | AWS isolated private network for running resources away from the public internet | Used to isolate SageMaker training and inference resources |
| **AWS PrivateLink** | AWS feature for private VPC connections to AWS services that bypass the public internet | Keeps Bedrock and SageMaker traffic off the public network |
| **Bedrock Guardrails** | Bedrock feature that filters harmful content, blocks topics, and redacts PII at runtime | Adds safety controls to both inputs and outputs of GenAI applications |
| **Shared responsibility model** | AWS's framework defining which security tasks AWS handles versus which the customer handles | AWS secures the cloud infrastructure; customers secure what they put in it |
| **Data residency** | Keeping data stored and processed within specific geographic regions for compliance | Determined by which AWS Region you deploy in |
| **Pre-training** | The initial and very costly process of training an FM on massive unlabeled data | Usually done by model providers like Anthropic, Meta, or Amazon |
| **Fine-tuning** | Training an existing FM on a smaller labeled dataset to adapt its behavior | Much cheaper than pre-training; done in Bedrock or SageMaker |
| **FM lifecycle** | The ordered stages an FM goes through: data → model selection → pre-train → fine-tune → evaluate → deploy → feedback | Tests ordering questions on the exam |
| **Amazon Nova** | Amazon's family of multimodal foundation models (text, image, video) available in Bedrock | Includes Nova Canvas for images and Nova Reel for video generation |
| **Stable Diffusion** | A popular open-source diffusion model for generating images from text prompts | Available via Amazon Bedrock from Stability AI |
| **Anthropic Claude** | A family of LLMs from Anthropic available in Amazon Bedrock | A leading conversational AI model widely used for enterprise tasks |
| **Meta Llama** | Open-weight LLM family from Meta available via Bedrock and SageMaker JumpStart | Popular for fine-tuning because weights can be downloaded and modified |
| **Amazon OpenSearch Service** | AWS managed search engine that also supports vector and hybrid search | Default vector store backend for Amazon Bedrock Knowledge Bases |
| **Amazon Bedrock Knowledge Bases** | Managed RAG service that handles chunking, embedding, storage, retrieval, and citation | The easiest way to ground a Bedrock model on private documents |
