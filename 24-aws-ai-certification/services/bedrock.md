# Amazon Bedrock

Amazon Bedrock is a fully managed, serverless service that gives you a single API to access high-performing foundation models (FMs) from Amazon and leading AI companies, plus the building blocks (RAG, agents, guardrails, customization, evaluation) to build generative AI applications without managing any infrastructure.

## 🧠 Mental model

Think of Bedrock as a **universal API socket for foundation models** — like a USB-C port for GenAI.

Instead of signing separate contracts, learning separate SDKs, and provisioning separate GPU clusters for Anthropic, Meta, Mistral, Cohere, and Amazon's own models, you plug into **one AWS API** and swap the model behind it by changing a single `modelId` string. AWS handles the servers, scaling, patching, and security; you never see a GPU.

And because it lives inside AWS, that same socket also plugs into the rest of your stack: your data stays in your account, traffic can stay on the AWS private network (PrivateLink/VPC), and IAM controls who can call what. **Your prompts and data are never used to train the base models**, and are not shared with the model providers.

Bedrock is also the **engine underneath other AWS GenAI products** — Amazon Q Business, Amazon Q Developer, and generative AI features in services like QuickSight and Connect all call foundation models through Bedrock under the hood.

## Architecture / flow

```mermaid
flowchart TD
    subgraph App["Your application / AWS service"]
        U["User request"]
        Q["Amazon Q, QuickSight, Connect (built on Bedrock)"]
    end

    U --> API["Amazon Bedrock unified API (InvokeModel / Converse)"]
    Q --> API

    API --> GR["Guardrails (content filters, denied topics, PII, contextual grounding)"]

    GR --> ROUTE{"What kind of call?"}

    ROUTE -->|"Direct inference"| FM["Foundation models: Amazon Nova / Titan, Anthropic Claude, Meta Llama, Mistral, Cohere, AI21, Stability AI, DeepSeek"]
    ROUTE -->|"Grounded answer"| KB["Knowledge Bases (managed RAG)"]
    ROUTE -->|"Multi-step task"| AG["Agents (action groups, tool use)"]
    ROUTE -->|"Orchestrated pipeline"| FL["Flows / Prompt management"]

    KB --> VS[("Vector store: OpenSearch Serverless / Managed, Aurora pgvector, Neptune Analytics, S3 Vectors, Pinecone, MongoDB Atlas, Redis")]
    KB --> S3[("Your data in Amazon S3")]
    AG --> LAM["AWS Lambda action groups / APIs"]
    AG --> KB

    FM --> RESP["Response (re-checked by Guardrails)"]
    KB --> RESP
    AG --> RESP
    FL --> RESP

    RESP --> App

    classDef store fill:#f5f5f5,stroke:#888;
    class VS,S3 store;
```

## What it does

Key capabilities:

- **Single unified API for many models.** One consistent API surface — including the `Converse` API for a standardized message format and the `InvokeModel` API — lets you call and swap models from multiple providers without rewriting integration code.
- **Broad foundation model catalog** (see below) spanning text, chat, embeddings, image, and video generation.
- **Amazon Bedrock Knowledge Bases** — fully managed Retrieval Augmented Generation (RAG): point it at your data in S3, it handles chunking, embedding, storing vectors, retrieval, and citation-backed responses.
- **Amazon Bedrock Agents** — break a user goal into steps, call your APIs/Lambda functions via **action groups**, query Knowledge Bases, and reason across multiple turns to complete tasks (tool use / function calling, managed for you).
- **Amazon Bedrock Guardrails** — configurable safety and responsible-AI controls applied independently of the model.
- **Model customization** — fine-tuning (labeled data) and continued pre-training (unlabeled data) to adapt models to your domain; the base model stays private and a private copy is customized for you.
- **Model evaluation** — automatic and human-based evaluation, plus **LLM-as-a-judge**, to compare models on your own data before you commit.
- **Bedrock Flows** — a visual, drag-and-drop way to chain prompts, models, Knowledge Bases, Agents, and Lambda into an orchestrated generative-AI workflow. **Prompt Management** stores, versions, and tests reusable prompts.
- **Serverless & fully managed** — no infrastructure to provision, automatic scaling, pay for what you use.
- **Enterprise controls** — IAM, VPC/PrivateLink for private connectivity, CloudWatch/CloudTrail integration, encryption with KMS, and data isolation.

### Foundation models available (current catalog)

Bedrock offers a large and growing catalog (100+ models across 15+ providers). Core families you should know for the exam:

| Provider | Representative models | Typical use |
|---|---|---|
| **Amazon** | **Nova** (Micro, Lite, Pro, Premier — text/multimodal; Canvas for images, Reel for video) and the older **Titan** family (text, embeddings, image) | AWS-native, cost-effective, multimodal, embeddings |
| **Anthropic** | **Claude** (Opus, Sonnet, Haiku tiers) | Strong reasoning, long context, agentic/tool use |
| **Meta** | **Llama** (Llama 3.x, Llama 4 Scout/Maverick) | Open-weight, customizable, multimodal |
| **Mistral AI** | **Mistral** Large / Small, and others | Efficient European models |
| **Cohere** | **Command** (text) and **Embed** (embeddings), Rerank | Enterprise text + retrieval/embeddings |
| **AI21 Labs** | **Jamba / Jurassic** | Long-context text generation |
| **Stability AI** | **Stable Diffusion / Stable Image** | Image generation |
| **DeepSeek** and others | DeepSeek-R1 etc. | Reasoning models (catalog expands frequently) |

> The exam does **not** require memorizing every model or version. Know the **providers**, that **Amazon Nova/Titan are AWS's own FMs**, and that the catalog is chosen via a single `modelId`. Also note the **Amazon Bedrock Marketplace**, which extends the catalog with 100+ specialized/emerging models you can deploy to managed endpoints.

### Vector stores for Knowledge Bases

Knowledge Bases can create/manage vectors for you, or connect to an existing store. Supported options include: **Amazon OpenSearch Serverless** (default managed option), **Amazon OpenSearch Managed Cluster**, **Amazon Aurora PostgreSQL (pgvector)**, **Amazon Neptune Analytics** (GraphRAG), **Amazon S3 Vectors**, **Pinecone**, **MongoDB Atlas**, and **Redis Enterprise Cloud**.

## When to use it (and when not to)

| Dimension | **Amazon Bedrock** | **SageMaker JumpStart / SageMaker AI** | **Amazon Q** | **Self-hosting (EC2/EKS + your own models)** |
|---|---|---|---|---|
| What it is | Serverless API to managed FMs + GenAI building blocks | ML platform to deploy/train/host models (incl. open FMs) on endpoints you control | Ready-to-use GenAI assistant (Q Business for enterprise data, Q Developer for coding) | You run the models yourself on raw compute |
| Infra to manage | **None** (serverless) | You choose/manage instances & endpoints | None (fully managed app) | **All of it** (GPUs, scaling, patching) |
| Best when | You want to build a custom GenAI app fast, mix models, add RAG/agents/guardrails | You need deep control, custom training, MLOps, or to host a model not in Bedrock | You want an out-of-the-box assistant, not to build one | You need full control, an unsupported model, or on-prem/edge |
| Customization | Fine-tune / continued pre-training (managed) | Full training, fine-tuning, custom containers | Minimal (configure data sources/plugins) | Unlimited |
| Pricing shape | Per-token / per-image / per-second, or provisioned throughput | Per instance-hour (endpoints) + training | Per-user subscription | Per compute-hour you provision |
| Pick it if you see... | "serverless", "unified API to multiple FMs", "managed RAG/agents/guardrails" | "custom training", "host my own model", "MLOps", "full control of endpoint" | "ready-made assistant over my business data / IDE coding helper" | "must run a specific model / on-prem / air-gapped" |

Rule of thumb: **Build a GenAI app → Bedrock. Do heavy custom ML / own the endpoint → SageMaker. Just want a finished assistant → Amazon Q. Need total control → self-host.**

## Pricing model

Bedrock is consumption-based. Don't memorize dollar figures (they change and vary by model/region) — know the **pricing dimensions** and when each applies:

- **On-Demand** — pay-as-you-go with no commitment. Billed per **input token** and **output token** (text/chat), per **image** (image models), or per **second** (video). Most flexible; ideal for spiky or exploratory workloads.
- **Batch inference** — submit large jobs asynchronously; results land in S3. Roughly **~50% cheaper** than on-demand. Use for non-latency-sensitive bulk processing (embeddings, classification, offline generation).
- **Prompt caching** — cache repeated prompt prefixes (e.g., long system prompts/context) to cut cost and latency dramatically on repeated inputs.
- **Provisioned Throughput** — reserve dedicated model capacity in **model units**, billed **hourly** with **1-month or 6-month commitments** (discounted vs on-demand for steady, high-volume traffic, and gives guaranteed throughput). **Required to run custom (fine-tuned or imported) models**, since they can't live in the shared on-demand pool.
- **Model customization** — you pay for **training** (fine-tuning / continued pre-training compute), plus **storage** of the custom model, plus **inference** (which needs Provisioned Throughput).
- **Add-on features** — Knowledge Bases (you also pay for the underlying vector store, e.g., OpenSearch), Guardrails, and model evaluation have their own usage-based charges layered on top of model inference.

Intuition: **On-demand = pay per sip. Batch = bulk discount for patient jobs. Provisioned = rent a dedicated tap for steady heavy flow (and mandatory for custom models). Customization = pay to build + park + serve your own tuned model.**

## 🎯 On the exam

Reflexes — **if you see X, pick Bedrock (or the noted feature)**:

- **"Access multiple foundation models through a single/unified API" / "serverless GenAI" / "no infrastructure to manage"** → **Amazon Bedrock**.
- **"Add my company's documents so the model answers from them, with citations" / "managed RAG"** → **Bedrock Knowledge Bases** (not custom-built RAG).
- **"Model should call APIs / take multi-step actions / use tools to complete a task"** → **Bedrock Agents** (action groups).
- **"Block harmful content / deny certain topics / redact PII / reduce hallucinations in RAG"** → **Bedrock Guardrails**. Content filters cover Hate, Insults, Sexual, Violence, Misconduct, and **Prompt Attack**; plus **denied topics**, **word filters**, **sensitive information (PII) filters**, **contextual grounding** (hallucination check for RAG), and **Automated Reasoning checks** (math-based factual verification). Guardrails work **across models** and can be reused.
- **"Adapt a model to my domain with labeled data"** → **fine-tuning**; **"with large unlabeled domain corpus"** → **continued pre-training**. Both keep your data private and produce a private custom model.
- **"Choose the best model for my use case / compare models on my data"** → **Bedrock model evaluation** (automatic, human, or **LLM-as-a-judge**).
- **"Visually chain prompts, models, KBs, and Lambda into a workflow"** → **Bedrock Flows**; **"store/version/test reusable prompts"** → **Prompt Management**.
- **"Guaranteed throughput / steady high volume / run a custom model"** → **Provisioned Throughput**. **"Cheap bulk, latency-tolerant"** → **Batch**. **"Flexible pay-per-use"** → **On-Demand**.
- **"Keep data private / not used to train the base model"** → true of Bedrock by default; add **VPC endpoints / AWS PrivateLink** for private network connectivity so traffic doesn't traverse the public internet.
- **Amazon's own models** on Bedrock = **Amazon Nova** (newer, multimodal, incl. Canvas/Reel) and **Amazon Titan** (embeddings, text, image).
- **Amazon Q is built on Bedrock.** If the question is "ready-made assistant over my enterprise data or in my IDE," pick **Amazon Q**, not Bedrock.
- **PartyRock** = a fun, no-code **playground app** (built on Bedrock) for experimenting with generative AI and building shareable apps without writing code or needing an AWS account for the app itself. Know it as the learning/experimentation surface.
- **Bedrock vs SageMaker**: Bedrock = serverless FM API + GenAI building blocks; SageMaker = full ML platform where you train/host and control endpoints. "Full control / custom training / host unsupported model" → SageMaker (JumpStart to quickly deploy open FMs).
- Bedrock is **regional** and supports **IAM**, **KMS encryption**, **CloudWatch/CloudTrail**, and **cross-region inference** for higher availability/throughput.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| Amazon Bedrock | A fully managed AWS service that gives you a single API to access many AI foundation models | Lets you build generative AI apps without managing any servers or GPU infrastructure |
| Foundation Model (FM) | A large pre-trained AI model that can handle many tasks like writing, summarizing, and answering questions | The core "brain" powering generative AI features |
| Serverless | A cloud delivery model where AWS runs and scales the infrastructure for you automatically | You pay only for what you use and never provision or manage servers |
| Unified API | A single consistent programming interface that works with multiple different AI model providers | Lets you swap models by changing one parameter without rewriting your integration code |
| modelId | The string identifier you pass to Bedrock to specify which foundation model to use | Allows model selection and easy model swapping within the same API call |
| Converse API | A standardized Bedrock API that uses a consistent message format across all supported models | Makes it easy to switch between chat-style models without changing your code structure |
| InvokeModel API | A Bedrock API for sending a prompt to a foundation model and getting a response | The core inference call for direct text and image generation requests |
| Knowledge Bases | A fully managed Bedrock feature that connects your documents to a foundation model so it can answer questions from them | Provides managed Retrieval-Augmented Generation (RAG) without building the pipeline yourself |
| RAG (Retrieval-Augmented Generation) | A technique where the AI looks up relevant documents before generating an answer | Makes AI responses grounded in your actual data rather than just the model's training |
| Bedrock Agents | A Bedrock feature that lets a model break a task into steps, call your APIs, and take multi-step actions | Enables autonomous AI that can complete complex goals, not just answer single questions |
| Action groups | Configured connections between a Bedrock Agent and your AWS Lambda functions or APIs | Let an agent actually do things, like query a database or submit a form |
| Bedrock Guardrails | Configurable safety controls you apply on top of any foundation model to filter harmful or unwanted content | Enforces responsible AI policies like blocking hate speech, denying topics, or redacting PII |
| PII (Personally Identifiable Information) | Data that can identify a specific person, like names, SSNs, or credit card numbers | Must be detected and redacted to comply with privacy regulations |
| Contextual grounding | A Guardrails feature that checks whether an AI answer is actually supported by the retrieved documents | Reduces hallucinations in RAG systems by verifying answers against sources |
| Automated Reasoning checks | A Guardrails feature that uses math-based logic to verify factual claims in model responses | Catches errors that are provably wrong, adding an extra accuracy layer |
| Prompt Attack | A category of harmful input where someone tries to trick the model into ignoring its instructions | Guardrails can detect and block this category of jailbreak attempt |
| Fine-tuning | Training an existing model further on your labeled examples to improve it for your specific task | Adapts a general-purpose model to your domain without training from scratch |
| Continued pre-training | Training an existing model further on large amounts of unlabeled domain text | Teaches the model your industry's vocabulary and context without needing labeled examples |
| Provisioned Throughput | A Bedrock pricing option where you reserve dedicated model capacity billed per hour with a commitment | Guarantees throughput for steady high-volume workloads and is required for custom models |
| Model units | The unit of reserved capacity in Provisioned Throughput | Each unit represents a certain amount of inference throughput you are guaranteeing |
| On-Demand pricing | Pay-as-you-go Bedrock billing with no commitment, charged per token or per image | Best for variable or exploratory workloads where traffic is unpredictable |
| Batch inference | Submitting a large job of many prompts to run asynchronously overnight, with results stored in S3 | Roughly 50% cheaper than on-demand and ideal for non-time-sensitive bulk work |
| Prompt caching | Storing repeated prompt prefixes so the model doesn't reprocess them on every call | Cuts cost and latency when you use the same long system prompt over and over |
| Bedrock Flows | A visual drag-and-drop tool to chain prompts, models, Knowledge Bases, Agents, and Lambda into a workflow | Lets you build multi-step AI pipelines without writing orchestration code |
| Prompt Management | A Bedrock feature for storing, versioning, and testing reusable prompt templates | Keeps prompts organized, reproducible, and shareable across your team |
| LLM-as-a-judge | Using another language model to automatically score and evaluate model outputs | Allows scalable, automated model evaluation without requiring human reviewers |
| Model evaluation | Bedrock tooling to compare multiple models on your own test data before committing to one | Helps you pick the best model for your specific task based on objective metrics |
| Amazon Nova | Amazon's own family of multimodal foundation models on Bedrock (Micro, Lite, Pro, Premier, Canvas, Reel) | AWS-native models optimized for cost-effectiveness across text, image, and video tasks |
| Amazon Titan | Amazon's older family of foundation models on Bedrock covering text, embeddings, and images | General-purpose AWS-native models, especially useful for generating embeddings |
| Anthropic Claude | A family of AI models from Anthropic available on Bedrock, known for strong reasoning and long context | Popular choice for agentic tasks, complex reasoning, and tool use scenarios |
| Meta Llama | An open-weight family of AI models from Meta available on Bedrock | Provides a customizable, open-source alternative with multimodal capabilities |
| Cohere Embed | A Cohere model on Bedrock designed to turn text into vector embeddings for search | Used in RAG pipelines to convert documents and queries into searchable numerical representations |
| Vector store | A database that stores and searches high-dimensional number arrays (embeddings) by similarity | The retrieval backbone of RAG systems that finds relevant documents for a given query |
| OpenSearch Serverless | A fully managed, auto-scaling version of Amazon OpenSearch Service | Used as the default vector store for Bedrock Knowledge Bases |
| Aurora pgvector | Amazon Aurora PostgreSQL with an extension for storing and querying vector embeddings | An option for teams already using PostgreSQL who want RAG without a separate vector database |
| Neptune Analytics | Amazon's graph database service with vector search support for GraphRAG use cases | Used when your knowledge has complex relationships best represented as a graph |
| S3 Vectors | Amazon S3 with native support for storing vector embeddings | A simple, low-cost vector storage option integrated directly into S3 |
| VPC (Virtual Private Cloud) | A logically isolated network in AWS where your resources communicate privately | Keeps traffic between your application and Bedrock off the public internet |
| PrivateLink | An AWS service that creates private network connections between your VPC and AWS services | Ensures Bedrock API calls never traverse the public internet for security compliance |
| IAM (Identity and Access Management) | AWS's system for controlling who can access which services and actions | Used to restrict which users or applications can call Bedrock APIs |
| KMS (Key Management Service) | AWS's service for creating and managing encryption keys | Ensures data sent to and stored by Bedrock is encrypted with your own keys |
| CloudWatch | AWS's monitoring and logging service | Tracks Bedrock usage metrics and errors for observability |
| CloudTrail | AWS's audit logging service that records every API call | Provides a complete record of who called Bedrock and when for compliance |
| Cross-region inference | Bedrock's ability to route requests to the same model in a different AWS region | Improves availability and throughput when one region is congested |
| Amazon Bedrock Marketplace | An extension of the Bedrock model catalog featuring 100+ specialized third-party models | Lets you access niche or emerging models that aren't in the standard catalog |
| PartyRock | A no-code web app built on Bedrock for experimenting with generative AI and building shareable demos | A learning and prototyping playground that requires no AWS account or coding |
| Amazon Q | A family of managed AI assistants built on Bedrock for enterprise data and software development | Ready-made AI products so you don't have to build a custom assistant yourself |
| SageMaker JumpStart | A SageMaker feature for deploying and fine-tuning foundation models on your own managed endpoints | Used when you need more control over the model infrastructure than serverless Bedrock provides |

## References

- Amazon Bedrock — product page: https://aws.amazon.com/bedrock/
- Supported foundation models in Amazon Bedrock: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
- Models at a glance (model catalog): https://docs.aws.amazon.com/bedrock/latest/userguide/model-cards.html
- Amazon Bedrock Knowledge Bases: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
- Vector store prerequisites for Knowledge Bases: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html
- Amazon Bedrock Agents: https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
- Amazon Bedrock Guardrails: https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- Guardrails components (content filters, denied topics, PII, contextual grounding): https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html
- Model customization: https://docs.aws.amazon.com/bedrock/latest/userguide/custom-models.html
- Model evaluation: https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html
- Bedrock Flows / Prompt management: https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html
- Provisioned Throughput: https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html
- Amazon Bedrock pricing: https://aws.amazon.com/bedrock/pricing/
- Data protection in Amazon Bedrock: https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html
- Use AWS PrivateLink with Amazon Bedrock: https://docs.aws.amazon.com/bedrock/latest/userguide/usingVPC.html
- PartyRock: https://partyrock.aws/
- Amazon Q: https://aws.amazon.com/q/
