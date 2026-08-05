# AIP-C01 Practice Question Bank — AWS Certified Generative AI Developer – Professional

This bank contains **50 scenario-based practice questions** mapped to the five official domains of the **AWS Certified Generative AI Developer – Professional (AIP-C01)** exam. Each question mirrors the professional-level, developer-focused style you will encounter on exam day: realistic multi-sentence scenario stems with a single qualifier (*MOST cost-effective*, *LEAST operational overhead*, *fastest*), followed by four or five options. Multiple-response questions (MRQ) instruct you to choose two or more answers.

**Exam logistics (as of 2026):**
- **Duration:** 170 minutes
- **Questions:** 65 scored + 10 unscored (75 total)
- **Pass score:** 750 / 1000 (compensatory — no per-domain minimum)
- **Price:** US $300
- **Format:** Multiple-choice (pick 1 of 4) and multiple-response (pick 2+ of 5)
- **Delivery:** Pearson VUE — online proctored or test center

**How to use this bank:** Work domain by domain. After each answer, read every distractor explanation — understanding *why wrong* answers are wrong is the fastest path to 750+. Simulate exam pressure by timing yourself at ~2.3 minutes per question.

---

## Domain Weight Table

| Domain | Focus | Weight | Questions in this bank |
|--------|-------|--------|------------------------|
| 1 | Foundation Model Integration, Data Management, and Compliance | 31% | 16 |
| 2 | Implementation and Integration | 26% | 13 |
| 3 | AI Safety, Security, and Governance | 20% | 10 |
| 4 | Operational Efficiency and Optimization | 12% | 6 |
| 5 | Testing, Validation, and Troubleshooting | 11% | 5 |
| — | Mixed Review | — | 5 |
| **Total** | | **100%** | **~55** |

---

## Domain 1 — Foundation Model Integration, Data Management, and Compliance (31%)

### Q1. [MCQ]

A fintech startup is building a customer-facing loan-eligibility chatbot on Amazon Bedrock. The chatbot must answer questions grounded in a 5,000-page internal policy document corpus, return answers in under two seconds at p95, and never fabricate rates or eligibility rules. The team has no plans to retrain a model. Which approach BEST satisfies all three requirements?

- **A.** Fine-tune Amazon Titan Text on the policy corpus using Bedrock model customization, then invoke the fine-tuned model directly.
- **B.** Build a Bedrock Knowledge Base backed by OpenSearch Serverless; use the RetrieveAndGenerate API with Bedrock Guardrails contextual grounding checks enabled.
- **C.** Store the entire policy corpus in the system prompt and invoke a large context-window model on every request.
- **D.** Use Amazon Kendra to index the corpus and invoke a foundation model independently, stitching the pieces together in a Lambda function.

<details><summary>Answer & explanation</summary>

**Correct: B.**

RAG via Bedrock Knowledge Bases keeps the corpus fresh without retraining, satisfies latency by retrieving only the relevant chunks, and contextual grounding checks in Bedrock Guardrails will block or flag any response that is not supported by the retrieved passages — directly preventing hallucinated rates. OpenSearch Serverless scales automatically for production workloads.

- **A** is wrong — fine-tuning bakes knowledge into weights; the corpus will go stale, and fine-tuning does not prevent hallucination.
- **C** is wrong — stuffing 5,000 pages into every prompt is prohibitively expensive, exceeds the context window of most models, and adds latency rather than reducing it.
- **D** is wrong — this is operationally heavier than Bedrock Knowledge Bases (requires maintaining separate Kendra indexes and custom glue code), violates the no-extra-ops spirit, and does not natively apply grounding checks.

</details>

---

### Q2. [MCQ]

A media company wants to select a foundation model for a real-time live-captioning service that transcribes speech and converts it to structured JSON for downstream systems. Latency must be under 500 ms. The company wants to stay within Bedrock's on-demand pricing. Which model characteristic is MOST important to evaluate first?

- **A.** Maximum context window length
- **B.** Number of parameters (model size) and time-to-first-token latency
- **C.** Whether the model supports fine-tuning via Bedrock model customization
- **D.** The model's benchmark score on the MMLU reasoning dataset

<details><summary>Answer & explanation</summary>

**Correct: B.**

For real-time inference with a hard latency SLA, time-to-first-token (TTFT) and overall throughput are the primary model-selection criteria. Smaller models typically deliver lower TTFT. MMLU and context-window length are secondary for a low-latency streaming task.

- **A** is wrong — a live captioning request is short; context window length is not the binding constraint.
- **C** is wrong — fine-tuning availability is a customization concern, not a latency predictor.
- **D** is wrong — MMLU measures reasoning, not generation speed.

</details>

---

### Q3. [MRQ — choose TWO]

A healthcare organization is deploying a Bedrock-powered clinical summarization tool. It must comply with HIPAA. Which TWO actions are required to keep PHI out of model training pipelines and meet HIPAA obligations on AWS?

- **A.** Enable Amazon Bedrock model invocation logging to CloudWatch Logs, ensuring all prompts are stored for audit.
- **B.** Execute an AWS Business Associate Agreement (BAA) before processing PHI through Bedrock.
- **C.** Use VPC endpoints (AWS PrivateLink) for all Bedrock API calls so PHI never traverses the public internet.
- **D.** Store embeddings in an S3 bucket with default SSE-S3 encryption without any additional configuration.
- **E.** Disable model invocation logging or route logs to a dedicated S3 bucket with KMS encryption and restricted access so PHI in prompts is not exposed in plain-text logs.

<details><summary>Answer & explanation</summary>

**Correct: B and E.**

A BAA is the contractual prerequisite for using any AWS service with PHI under HIPAA. Invocation logs can contain the full prompt text (including PHI) in plain text; disabling logs or protecting them with a customer-managed KMS key and strict IAM is the required mitigation. AWS documents explicitly warn that blocked content from Guardrails and full prompt text appear as plain text in invocation logs.

- **A** is wrong — storing PHI-containing prompts unprotected in CloudWatch violates HIPAA.
- **C** is useful best practice (defense in depth) but is not strictly required by HIPAA; the BAA and data-at-rest controls take precedence.
- **D** is wrong — SSE-S3 uses AWS-managed keys, which does not satisfy HIPAA's requirement for customer control over encryption keys for PHI.

</details>

---

### Q4. [MCQ]

A retailer needs a Bedrock Knowledge Base for a product recommendation chatbot. Their catalog has 10 million product vectors and workload peaks heavily during Black Friday (10,000 queries per second) but is nearly idle the remaining 10 months. Which vector store provides the MOST cost-effective option for this pattern?

- **A.** Amazon OpenSearch Serverless (vector search collection)
- **B.** Amazon Aurora PostgreSQL with pgvector
- **C.** Amazon S3 Vectors
- **D.** Amazon Neptune Analytics

<details><summary>Answer & explanation</summary>

**Correct: C.**

Amazon S3 Vectors is designed specifically for infrequent-query, large-scale vector workloads. It provides durable, elastic storage with sub-second query performance and is priced per GB stored plus per query — there are no provisioned OCU/ACU charges sitting idle during off-peak months. This matches the sporadic-peak-then-idle pattern perfectly.

- **A** is wrong — OpenSearch Serverless bills minimum OCU capacity even when idle; it's cost-effective for sustained traffic, not extreme burstiness with long idle periods.
- **B** is wrong — Aurora requires provisioned instance capacity billed hourly regardless of query volume.
- **D** is wrong — Neptune Analytics charges for m-NCUs (memory-NCUs) continuously; it is the right choice for graph traversal / GraphRAG, not pure cost-optimization on a vector-only workload.

</details>

---

### Q5. [MCQ]

A legal tech company wants to build a contract analysis tool where multi-hop reasoning is required — for example, finding all contracts that reference a specific clause, then finding all addenda that modify those clauses. Simple semantic similarity search returns inconsistent results. Which Bedrock Knowledge Base vector store configuration BEST addresses multi-hop retrieval?

- **A.** OpenSearch Serverless with a cosine-distance vector index
- **B.** Amazon Aurora PostgreSQL with pgvector and HNSW indexes
- **C.** Neptune Analytics with GraphRAG enabled
- **D.** Amazon S3 Vectors with hierarchical chunking

<details><summary>Answer & explanation</summary>

**Correct: C.**

Neptune Analytics with GraphRAG creates nodes for document chunks and entities plus CONTAINS and FROM edges, enabling traversal across connected chunks — exactly what multi-hop retrieval requires. It excels when relationships between entities (clause references, addendum links) matter more than raw semantic similarity.

- **A** is wrong — pure cosine-similarity ANN search has no graph traversal capability.
- **B** is wrong — pgvector supports vector similarity but not graph-style multi-hop traversal.
- **D** is wrong — S3 Vectors is optimized for cost on infrequent workloads; it provides simple ANN lookup with no graph structure.

</details>

---

### Q6. [MCQ]

A developer is selecting an embedding model for a Bedrock Knowledge Base. The corpus is primarily technical English documentation. The team wants the highest retrieval accuracy. Binary vector embeddings are a priority because the OpenSearch Serverless index was configured specifically for binary vectors. Which constraint must they observe?

- **A.** Binary vector embeddings are not supported; the developer must switch to float32.
- **B.** Only Amazon OpenSearch Serverless and Amazon OpenSearch Managed Clusters support binary vectors; the developer must not choose S3 Vectors or Aurora pgvector as the vector store.
- **C.** Binary vectors require Neptune Analytics with GraphRAG enabled.
- **D.** Binary vector support is available in all Bedrock Knowledge Base vector store options.

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation explicitly states: "Amazon OpenSearch Serverless and Amazon OpenSearch Managed clusters are the only vector stores that support storing binary vectors." S3 Vectors only supports floating-point embeddings; Aurora and Neptune also do not support binary vectors in Bedrock Knowledge Bases.

- **A** is wrong — binary vectors are supported, just only with OpenSearch stores.
- **C** is wrong — Neptune Analytics does not support binary embeddings in Knowledge Bases.
- **D** is wrong — this is factually incorrect per official AWS docs.

</details>

---

### Q7. [MRQ — choose TWO]

A company is ingesting confidential documents into a Bedrock Knowledge Base backed by Amazon S3 Vectors. Which TWO encryption-related facts must the developer know when setting up the S3 vector bucket?

- **A.** The encryption type (SSE-S3 or SSE-KMS) can be changed at any time after the vector bucket is created.
- **B.** The encryption type cannot be changed once the vector bucket has been created.
- **C.** SSE-KMS can be specified at both the vector bucket level and can be overridden at the vector index level.
- **D.** S3 Vectors only supports SSE-S3; SSE-KMS is not available.
- **E.** S3 Vectors does not support encryption of stored vectors.

<details><summary>Answer & explanation</summary>

**Correct: B and C.**

AWS documentation states the encryption type can't be changed once the vector bucket is created, and separately that when creating a vector index you can override the bucket-level encryption to choose SSE-KMS or SSE-S3 at index granularity.

- **A** is wrong — this directly contradicts the documented immutability of bucket-level encryption choice.
- **D** is wrong — both SSE-S3 and SSE-KMS are available.
- **E** is wrong — encryption is fully supported.

</details>

---

### Q8. [MCQ]

A generative AI team wants to adapt a pre-trained Bedrock foundation model to understand highly specialized oncology terminology that does not appear in the model's training data. The team has 50,000 curated oncology documents. They want the knowledge to be baked into the model weights rather than retrieved at query time. Which technique is MOST appropriate?

- **A.** RAG with a Bedrock Knowledge Base backed by OpenSearch Serverless
- **B.** Continued pre-training (domain adaptation) via Bedrock model customization
- **C.** Few-shot prompting with ten representative oncology examples per request
- **D.** Bedrock Agents with a Lambda tool that queries a medical database

<details><summary>Answer & explanation</summary>

**Correct: B.**

Continued pre-training (also called domain adaptation fine-tuning in Bedrock) updates the model weights on an unlabeled corpus, making the model internalize domain vocabulary and facts. RAG retrieves at query time but doesn't change weights. Few-shot prompting is limited by context window and does not generalize. Agents add agentic capability but not medical vocabulary.

- **A** is wrong — RAG is effective for grounding but the question explicitly says knowledge must be in the weights.
- **C** is wrong — few-shot prompting with 10 examples per prompt cannot teach 50,000 documents worth of vocabulary.
- **D** is wrong — a Lambda tool can look up answers but the model weights don't learn specialized terminology.

</details>

---

### Q9. [MCQ]

A startup wants to use a large, expensive foundation model's capability in a smaller, cheaper model they can deploy on their own infrastructure. They have thousands of labeled examples where the large model performed well. Which Bedrock model customization technique MOST directly achieves this goal?

- **A.** Instruction fine-tuning (supervised fine-tuning)
- **B.** Model distillation
- **C.** Retrieval-augmented generation
- **D.** Prompt caching

<details><summary>Answer & explanation</summary>

**Correct: B.**

Model distillation trains a smaller "student" model to replicate the outputs of a larger "teacher" model. Bedrock supports distillation as a customization option. It directly produces a smaller, cheaper model that approximates the teacher's quality. SFT improves a specific task but doesn't compress knowledge from a teacher; RAG is a retrieval pattern, not a compression technique; prompt caching reduces cost per call but does not produce a new model.

- **A** is wrong — instruction fine-tuning adapts the same model to a task but doesn't transfer knowledge from a larger teacher.
- **C** is wrong — RAG is an inference-time retrieval pattern.
- **D** is wrong — prompt caching is a cost-reduction technique, not a model training method.

</details>

---

### Q10. [MCQ]

An enterprise compliance team wants to guarantee that the metadata for each vector chunk stored in a Bedrock Knowledge Base on Amazon S3 Vectors does not exceed platform limits, causing ingestion failures. What is the maximum amount of custom metadata (filterable + non-filterable combined) per vector that S3 Vectors supports within a Bedrock Knowledge Base?

- **A.** 512 bytes, with up to 20 metadata keys
- **B.** 1 KB, with up to 35 metadata keys
- **C.** 4 KB, with up to 100 metadata keys
- **D.** Unlimited; metadata is stored separately in S3 and not subject to per-vector limits

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation states: "you can attach up to 1 KB of custom metadata (including both filterable and non-filterable metadata) and 35 metadata keys per vector." Exceeding these limits causes the ingestion job to throw an exception.

- **A** is wrong — the limit is 1 KB, not 512 bytes.
- **C** is wrong — 4 KB and 100 keys exceed the documented limits.
- **D** is wrong — metadata is co-located with the vector and strictly limited per the S3 Vectors specification.

</details>

---

### Q11. [MCQ]

A developer is connecting an Aurora PostgreSQL knowledge base to Bedrock with metadata filtering enabled. After a deployment, selective metadata filters return fewer results than expected despite documents clearly matching the filter criteria. What is the MOST likely cause and fix?

- **A.** The Aurora cluster is in a different AWS account than the Bedrock knowledge base; move them to the same account.
- **B.** HNSW iterative index scans are not enabled; run `SET hnsw.iterative_scan = 'relaxed_order'` at the database level and wait for connection pool recycling.
- **C.** The GIN index on the metadata column is missing; add `CREATE INDEX ON bedrock_kb USING gin (to_tsvector('english', chunks))`.
- **D.** The pgvector extension version is below 0.7.0; upgrade to 0.7.0 or higher.

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation explicitly states: "If you use metadata filtering with your knowledge base, we recommend enabling HNSW iterative index scans (requires pgvector 0.8.0 or later). Without iterative scans, selective metadata filters can return fewer results than expected because filtering is applied after the HNSW index scan." The fix is the `hnsw.iterative_scan = 'relaxed_order'` setting, and the note that changes only affect new sessions (allow time for connection pool recycling).

- **A** is wrong — the Aurora cluster must be in the same account (a hard requirement), but this is a precondition, not the cause of under-retrieval.
- **C** is wrong — the GIN index description given (full-text on `chunks`) is for text search, not for metadata key-value filtering.
- **D** is wrong — iterative scan requires pgvector 0.8.0+, not 0.7.0.

</details>

---

### Q12. [MRQ — choose TWO]

A company is using Amazon Bedrock Knowledge Bases and wants to implement metadata filtering on a MongoDB Atlas vector store. Which TWO statements about metadata filtering with MongoDB Atlas in Bedrock Knowledge Bases are correct?

- **A.** Metadata filtering works by default without any additional MongoDB Atlas index configuration.
- **B.** Metadata filtering requires manually configuring filters in the MongoDB Atlas vector index; it does not work by default.
- **C.** You must provide credentials via AWS Secrets Manager using keys `username` and `password`.
- **D.** MongoDB Atlas vector stores support binary vector embeddings natively within Bedrock Knowledge Bases.
- **E.** The MongoDB Atlas cluster endpoint must be reachable only through AWS PrivateLink; public endpoints are not supported.

<details><summary>Answer & explanation</summary>

**Correct: B and C.**

AWS documentation states: "If you plan to use metadata filtering with your MongoDB Atlas knowledge base, you must manually configure filters in your vector index. Metadata filtering doesn't work by default and requires additional setup." Additionally, credentials are provided through Secrets Manager using the keys `username` and `password`.

- **A** is wrong — explicitly contradicted by documentation.
- **D** is wrong — only OpenSearch Serverless and OpenSearch Managed Clusters support binary vectors.
- **E** is wrong — MongoDB Atlas supports both public and PrivateLink-based connectivity; public endpoints are allowed.

</details>

---

### Q13. [MCQ]

A healthcare application running on Bedrock ingests patient intake forms (PDFs) into a Knowledge Base. The forms contain SSNs and dates of birth. The team wants PII automatically redacted from both the chunked text stored in the vector store AND from model responses before they reach the UI. Which service provides this capability natively within the Bedrock ecosystem?

- **A.** AWS Macie, configured to scan the S3 data source bucket before ingestion
- **B.** Amazon Comprehend Medical, invoked by a Lambda function in an API Gateway layer
- **C.** Amazon Bedrock Guardrails sensitive information filters, applied to the Knowledge Base's RetrieveAndGenerate API call
- **D.** An IAM policy condition that denies access to S3 objects matching known PII patterns

<details><summary>Answer & explanation</summary>

**Correct: C.**

Bedrock Guardrails sensitive information filters detect PII in standard formats (SSN, DOB, addresses, etc.) using probabilistic ML in both user inputs and FM responses, and can block or mask the detected PII. This is the only option that operates natively within the Bedrock inference pipeline on both input and output.

- **A** is wrong — Macie scans S3 for data governance but does not intercept runtime inference prompts or model outputs.
- **B** is wrong — Comprehend Medical can detect medical PII but requires custom integration code and does not natively hook into Bedrock's RetrieveAndGenerate flow.
- **D** is wrong — IAM policies govern access control, not content inspection or redaction.

</details>

---

### Q14. [MCQ]

A developer is choosing between RAG and fine-tuning for a customer support application. The product knowledge base changes weekly with new SKUs and pricing. Which statement BEST describes the tradeoff that should drive the decision toward RAG?

- **A.** RAG is always cheaper than fine-tuning regardless of query volume.
- **B.** Fine-tuning is better because it reduces inference latency by eliminating retrieval round-trips.
- **C.** RAG is preferred when knowledge must remain current without retraining, because only the vector store needs to be re-synced when the corpus changes.
- **D.** Fine-tuning is preferred when the corpus changes frequently because each new fine-tuning job updates only the delta.

<details><summary>Answer & explanation</summary>

**Correct: C.**

The core advantage of RAG for frequently changing knowledge is that only the knowledge base index needs to be re-synced — no model retraining, no deployment cycle. Fine-tuning bakes knowledge into weights at a point in time and must be repeated every time the corpus changes, which is expensive and slow.

- **A** is wrong — RAG incurs retrieval and embedding costs per query; at very high volume, a fine-tuned model without retrieval may be cheaper.
- **B** is wrong — while fine-tuning does eliminate retrieval latency, it cannot keep up with weekly corpus changes without continuous retraining.
- **D** is wrong — fine-tuning jobs process the full training set (or delta sets require careful management); there is no automatic delta fine-tuning.

</details>

---

### Q15. [MCQ]

A Bedrock Knowledge Base using Neptune Analytics for GraphRAG has been deployed. A product manager reports that multi-hop queries are returning fewer results than the configured retrieval limit of 4 chunks. What is the MOST likely explanation based on documented Neptune Analytics behavior?

- **A.** Neptune Analytics GraphRAG is in preview and has a hard limit of 3 chunks per query.
- **B.** Chunks falling below an internal similarity score threshold are excluded from results, so fewer chunks may be returned even when the limit allows more.
- **C.** The Neptune graph must be manually updated with semantic relationships; Bedrock does not auto-extract entity relationships.
- **D.** The Neptune Analytics graph does not support vector search; a separate OpenSearch index is required for cosine similarity.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Community testing and AWS documentation show that Neptune Analytics GraphRAG applies a score threshold to retrieved chunks; chunks below the threshold are excluded even if the configured retrieval limit would allow them. This is why "Neptune retrieved only 3 chunks despite a limit of 4."

- **A** is wrong — GraphRAG became generally available in March 2025; the 3-chunk observation is due to score filtering, not a hard limit.
- **C** is a valid nuance (Bedrock auto-extracts only CONTAINS/FROM relationships, not semantic ones like "hosted_on"), but it describes a quality issue, not the reason the result count is below the limit.
- **D** is wrong — Neptune Analytics natively supports vector similarity search alongside graph traversal.

</details>

---

### Q16. [MRQ — choose TWO]

A company stores sensitive embeddings in an Amazon OpenSearch Serverless (AOSS) collection used as a Bedrock Knowledge Base. The security team requires all traffic between Bedrock and AOSS to stay within the AWS network and that data at rest be encrypted with a customer-managed key. Which TWO configurations achieve both requirements?

- **A.** In the AOSS network access settings, select Standard create and choose VPC as the access type with a VPC endpoint.
- **B.** Configure the AOSS collection with default public access and rely on TLS in transit.
- **C.** Apply a customer-managed KMS key (CMK) to the AOSS collection for encryption at rest.
- **D.** Use SSE-S3 on the S3 data source bucket; AOSS inherits that key automatically.
- **E.** Enable AWS CloudTrail for the AOSS collection to satisfy encryption requirements.

<details><summary>Answer & explanation</summary>

**Correct: A and C.**

AWS documentation for Knowledge Base setup with AOSS explicitly states: to make a collection private, select Standard create and choose VPC as the access type. For encryption at rest with a customer-managed key, you apply a CMK to the AOSS collection directly.

- **B** is wrong — public access exposes traffic outside the AWS private network; TLS is in-transit only and does not satisfy the "within AWS network" requirement.
- **D** is wrong — SSE-S3 applies to the S3 source bucket; AOSS manages its own encryption independently.
- **E** is wrong — CloudTrail provides audit logs, not encryption.

</details>

---

## Domain 2 — Implementation and Integration (26%)

### Q17. [MCQ]

A developer is building a multi-turn chatbot using the Amazon Bedrock Converse API. They need to send conversation history, user messages, and a tool configuration in a single call while streaming the response. Which API call is correct?

- **A.** `InvokeModel` with `"stream": true` in the request body
- **B.** `ConverseStream` with `messages`, `system`, and `toolConfig` fields in the request
- **C.** `InvokeModelWithResponseStream` using a custom messages array
- **D.** `RetrieveAndGenerate` with streaming enabled via a `sessionConfiguration` field

<details><summary>Answer & explanation</summary>

**Correct: B.**

The Converse API (`Converse` and `ConverseStream`) is the purpose-built interface for multi-turn conversations: it accepts `messages` (conversation history), `system` (system prompt), and `toolConfig` (function/tool definitions) in a unified request schema. `ConverseStream` delivers the response as a stream. `InvokeModel` and `InvokeModelWithResponseStream` use model-specific request formats and do not have a unified tool-calling schema. `RetrieveAndGenerate` is the Knowledge Base API, not the general conversational API.

- **A** is wrong — `InvokeModel` is synchronous; it has no `stream` flag; each model has its own body schema.
- **C** is wrong — `InvokeModelWithResponseStream` streams but uses model-specific body formats, not the unified Converse schema.
- **D** is wrong — `RetrieveAndGenerate` is for RAG, not general multi-turn tool use.

</details>

---

### Q18. [MCQ]

A team is implementing function calling in a Bedrock Agents workflow. The agent should call an external weather API when a user asks about weather. The developer defines the tool in the agent's action group. At runtime, the agent returns a `ReturnControl` event instead of calling the Lambda function directly. What does this mean and how should the developer respond?

- **A.** The agent failed; the developer must redeploy the action group.
- **B.** `ReturnControl` means the agent has decided which tool to invoke and is returning the tool name and parameters to the calling application, which must execute the tool itself and return the result via `resumeSession`.
- **C.** `ReturnControl` means the model needs more context; the developer must append additional system prompt instructions.
- **D.** `ReturnControl` is a streaming artifact that can be ignored; the final answer will follow.

<details><summary>Answer & explanation</summary>

**Correct: B.**

`ReturnControl` (also called "inline agent" or "code interpreter return") is a Bedrock Agents feature where the agent delegates tool execution back to the calling application rather than invoking a Lambda itself. The application is responsible for executing the tool and then calling `resumeSession` (or `InvokeAgent` with the `sessionState.returnControlInvocationResults`) to pass the tool output back. This pattern is used when the developer wants to control tool execution in-process.

- **A** is wrong — `ReturnControl` is a valid, intentional design pattern.
- **C** is wrong — `ReturnControl` is an explicit protocol event, not a signal for more context.
- **D** is wrong — `ReturnControl` is a terminal event in that step; the agent is paused awaiting the tool result.

</details>

---

### Q19. [MCQ]

A developer is building an orchestration pipeline in which a Bedrock Agent must call three sequential tools: first look up an order, then check inventory, then trigger a shipment. If any step fails, the pipeline must roll back. Which AWS service provides the MOST appropriate mechanism for durable, error-handled sequential orchestration with rollback capability?

- **A.** AWS Lambda with nested function calls and try/except blocks
- **B.** Amazon EventBridge with three separate rules chained via events
- **C.** AWS Step Functions with a state machine using Task states, Catch blocks, and Compensating transactions
- **D.** Amazon SQS FIFO queue with Lambda consumers for each step

<details><summary>Answer & explanation</summary>

**Correct: C.**

Step Functions is the AWS-native durable workflow orchestration service. It supports sequential Task states (each calling a Lambda or Bedrock model), Catch blocks for error handling, and compensating transaction patterns for rollback. This is exactly the pattern described in the AIP-C01 domain on Lambda + Step Functions orchestration.

- **A** is wrong — in-process Lambda orchestration loses state on timeout and has no built-in retry/rollback primitives.
- **B** is wrong — EventBridge rules are event-driven and asynchronous; they do not provide sequential, stateful orchestration with rollback.
- **D** is wrong — SQS FIFO ensures ordering but not transactional rollback; each consumer is independent.

</details>

---

### Q20. [MRQ — choose TWO]

A developer is using the Bedrock Converse API with tool use (function calling). The model responds with a `toolUse` block. Which TWO actions must the developer take next to complete the tool-calling round-trip?

- **A.** Parse the `toolUse` block to extract the tool name and input parameters, then execute the corresponding function.
- **B.** Send a new `Converse` request including the original messages plus the assistant's `toolUse` message AND a `toolResult` user message containing the function's output.
- **C.** Call `InvokeModel` with the tool output appended to the system prompt.
- **D.** Restart the conversation from scratch; the Converse API does not support multi-turn tool calling.
- **E.** Store the `toolUse` block in DynamoDB and wait for the model to poll it.

<details><summary>Answer & explanation</summary>

**Correct: A and B.**

The Converse API tool-use loop requires: (1) parse the `toolUse` block to get the tool name and input; (2) execute the tool; (3) append the assistant's response (including the `toolUse` block) and a new user message with a `toolResult` block to the conversation history; (4) call `Converse` again. The model then synthesizes a final response using the tool result.

- **C** is wrong — `InvokeModel` uses model-specific body formats and does not use the `toolResult` message structure.
- **D** is wrong — multi-turn tool calling is a core Converse API feature.
- **E** is wrong — the model does not poll external storage; the result is passed inline in the next API call.

</details>

---

### Q21. [MCQ]

A company is adopting Amazon Bedrock AgentCore to deploy production agents built with the Strands Agents SDK. The architects need agents to communicate with each other across agent boundaries using a standardized protocol. Which protocol does AgentCore natively support for agent-to-agent communication?

- **A.** MQTT over WebSocket
- **B.** Agent-to-Agent (A2A) protocol
- **C.** OpenAI function-calling wire format
- **D.** LangChain Expression Language (LCEL) execution protocol

<details><summary>Answer & explanation</summary>

**Correct: B.**

Amazon Bedrock AgentCore natively supports the Agent-to-Agent (A2A) protocol, enabling multi-agent patterns (Graph, Swarm, Workflow) when combined with Strands Agents. This is documented in the AgentCore multi-agent architecture guidance.

- **A** is wrong — MQTT is an IoT messaging protocol and is not used for agent-to-agent orchestration in Bedrock.
- **C** is wrong — the OpenAI wire format is an inference API compatibility layer (bedrock-mantle), not an agent orchestration protocol.
- **D** is wrong — LCEL is a LangChain-specific composition syntax, not an AWS-native protocol.

</details>

---

### Q22. [MCQ]

A team wants to integrate external tools into a Bedrock Agent using the Model Context Protocol (MCP). What is the PRIMARY benefit MCP provides in this integration?

- **A.** MCP replaces Bedrock Guardrails for content moderation.
- **B.** MCP standardizes tool integration, allowing agents to discover and invoke tools across different frameworks without custom glue code for each tool.
- **C.** MCP encrypts all tool invocations with a customer-managed KMS key automatically.
- **D.** MCP enables real-time bidirectional streaming between the agent and foundation model without going through the Converse API.

<details><summary>Answer & explanation</summary>

**Correct: B.**

MCP (Model Context Protocol) standardizes how AI agents discover and call external tools. In the context of Bedrock AgentCore, MCP provides a common interface so tools built for one framework (e.g., LangGraph) work with agents built in another (e.g., Strands), reducing the custom integration burden.

- **A** is wrong — MCP is a tool-integration protocol, not a safety/content moderation layer.
- **C** is wrong — MCP does not handle KMS encryption; that is a Bedrock/IAM concern.
- **D** is wrong — the Converse/ConverseStream APIs handle streaming inference; MCP governs tool definitions.

</details>

---

### Q23. [MCQ]

A developer needs to invoke an Amazon Bedrock foundation model with Provisioned Throughput that was purchased for a fine-tuned model. In testing, the team notices that on-demand costs are still being incurred despite the provisioned capacity being active. What is the MOST likely cause?

- **A.** Provisioned Throughput only works with base foundation models, not fine-tuned models.
- **B.** The developer is passing the base foundation model ID as `modelId` instead of the `provisionedModelArn`.
- **C.** Provisioned Throughput requires enabling it via a CloudWatch alarm before it becomes active.
- **D.** The Provisioned Throughput model unit (MU) count is set to zero; it must be at least two.

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation states: "You must pass the `provisionedModelArn` as your `modelId` in inference calls. Using the foundation-model ID instead leaves provisioned capacity unused while consuming on-demand quota." This is the documented "ARN pitfall." Provisioned Throughput is mandatory for fine-tuned models but is invoked via the provisioned model ARN, not the base model ID.

- **A** is wrong — Provisioned Throughput is in fact mandatory for custom (fine-tuned) models.
- **C** is wrong — Provisioned Throughput is activated via purchase, not a CloudWatch alarm.
- **D** is wrong — MU count of zero would mean no capacity was purchased, but the scenario states capacity is active.

</details>

---

### Q24. [MCQ]

A startup is building a real-time coding assistant. They use prompt caching to avoid re-sending a 10,000-token system prompt on every keystroke event. A developer reports that after five minutes of inactivity, subsequent requests are not benefiting from the cache (cache miss). What is the MOST likely cause?

- **A.** Prompt caching only works with Amazon Nova models; the team is using a Claude model.
- **B.** The default cache TTL is 5 minutes; after 5 minutes of inactivity the cache expires and the next call is a cache write (not a cache hit).
- **C.** The cached tokens counted against TPM quotas and caused throttling, clearing the cache.
- **D.** Prompt caching requires Provisioned Throughput; on-demand invocations are not eligible.

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation states the default TTL for prompt caching is 5 minutes (extendable to 1 hour on supported models). After 5 minutes of inactivity, the cache expires and the next request is a cache write — billed at write rates — not a cache hit. The fix is to extend the TTL or ensure requests arrive within the TTL window.

- **A** is wrong — prompt caching is supported for Claude models (e.g., Claude models require a 4,096-token minimum for the cacheable prefix).
- **C** is wrong — cached tokens do NOT count toward TPM quotas; this is an explicit benefit of prompt caching.
- **D** is wrong — prompt caching works with on-demand invocations.

</details>

---

### Q25. [MCQ]

An enterprise wants to run 2 million document summarization tasks offline. Each document is about 2,000 tokens. The tasks are non-urgent and can complete within 24 hours. Which Bedrock invocation mode is MOST cost-effective?

- **A.** On-demand inference via `InvokeModel` with exponential backoff for throttling
- **B.** Provisioned Throughput with a 1-month commitment
- **C.** Batch inference via `CreateModelInvocationJob` pointing to S3 input/output locations
- **D.** Latency-optimized inference to process all jobs faster and reduce wall-clock cost

<details><summary>Answer & explanation</summary>

**Correct: C.**

Batch inference (also called batch invocation) is designed for large asynchronous workloads: you submit a JSONL file on S3, it runs against separate quotas, and it is priced at approximately 50% of on-demand rates. Non-urgent, large-scale, offline jobs are the canonical batch inference use case.

- **A** is wrong — on-demand inference at 2 million calls incurs full per-token pricing and competes with real-time quota.
- **B** is wrong — Provisioned Throughput with a monthly commitment is cost-effective only for sustained high-volume real-time inference; a one-time offline batch job does not justify the reservation.
- **D** is wrong — latency-optimized inference is for minimizing time-to-first-token on interactive requests; it costs more, not less.

</details>

---

### Q26. [MCQ]

A developer is building a Bedrock Agent that needs to access a private REST API hosted inside a VPC. The agent's action group is configured with an OpenAPI schema. Which integration approach keeps all traffic within the AWS network while minimizing operational overhead?

- **A.** Deploy the REST API on a public API Gateway endpoint and whitelist the Bedrock service IP ranges in the API Gateway resource policy.
- **B.** Configure the action group to invoke a Lambda function inside the same VPC using a VPC-connected Lambda with an ENI, placing the private API call inside that Lambda.
- **C.** Use a public internet-facing ALB in front of the private API and add a Bedrock-specific header for authentication.
- **D.** Grant Bedrock an IAM role with `ec2:DescribeVpcs` to route traffic directly into the VPC without Lambda.

<details><summary>Answer & explanation</summary>

**Correct: B.**

A VPC-connected Lambda function can reach private resources inside a VPC via ENIs without any public exposure. The Bedrock Agent invokes the Lambda (which is public-facing to Bedrock but private-facing to the API), keeping the internal traffic private. This is the standard, low-overhead pattern for Agents accessing private APIs.

- **A** is wrong — public API Gateway exposes the internal API to the internet; Bedrock service IP whitelisting is fragile and not recommended.
- **C** is wrong — an internet-facing ALB exposes the private API publicly.
- **D** is wrong — `ec2:DescribeVpcs` is a metadata API permission, not a data-path routing mechanism.

</details>

---

### Q27. [MRQ — choose TWO]

A developer is designing a Strands Agents multi-agent system where a supervisor agent dispatches subtasks to specialized worker agents. They want to use Amazon Bedrock AgentCore as the deployment infrastructure. Which TWO statements about Strands Agents + AgentCore are accurate?

- **A.** AgentCore supports Strands Agents, LangGraph, Claude Agent SDK, and OpenAI Agents SDK on the same managed infrastructure.
- **B.** Strands Agents uses the "Agents as Tools" pattern where specialized agents are wrapped as callable tools for the supervisor.
- **C.** AgentCore requires all agents to be written in Python using only the Strands Agents SDK; other frameworks are not supported.
- **D.** The A2A protocol is only supported between two agents; it cannot orchestrate more than two agents at once.
- **E.** Strands Agents multi-agent topology is limited to a linear pipeline; graph or swarm patterns require a different framework.

<details><summary>Answer & explanation</summary>

**Correct: A and B.**

Documentation confirms AgentCore supports multiple frameworks (Strands, LangGraph, Claude Agent SDK, Google ADK, OpenAI Agents SDK) on the same infrastructure. Strands uses the "Agents as Tools" pattern where agents are wrapped as callable tools. Additionally, AgentCore's A2A protocol support enables Graph, Swarm, and Workflow multi-agent patterns with Strands — not just linear pipelines.

- **C** is wrong — AgentCore is framework-agnostic.
- **D** is wrong — A2A supports multi-agent orchestration beyond two agents.
- **E** is wrong — Strands + AgentCore A2A supports Graph and Swarm patterns explicitly.

</details>

---

### Q28. [MCQ]

A developer wants to implement intelligent prompt routing in Amazon Bedrock to automatically send simple customer queries to a smaller, cheaper model and complex queries to a larger model. What is a KEY constraint of Bedrock's Intelligent Prompt Routing feature?

- **A.** It only supports models from a single provider (Amazon Nova family).
- **B.** It routes between exactly two models (pairwise routing) with configurable quality-difference thresholds; it does not support routing among three or more models.
- **C.** It requires Provisioned Throughput on both models before routing can be enabled.
- **D.** It only works with on-demand batch inference jobs, not real-time interactive traffic.

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation states intelligent prompt routing is "pairwise routing only" — it selects between exactly two models (a default router for Anthropic/Meta families, or custom configurations). Routing among three or more models is not supported. It is designed for "mixed-difficulty, interactive traffic."

- **A** is wrong — routing supports Anthropic and Meta families, not just Amazon Nova.
- **C** is wrong — routing works with on-demand pricing; Provisioned Throughput is not required.
- **D** is wrong — routing is specifically optimized for real-time interactive traffic, not batch jobs.

</details>

---

### Q29. [MCQ]

A developer is implementing exponential backoff for Bedrock `InvokeModel` calls to handle `ThrottlingException` responses. Which code pattern is MOST correct?

- **A.** Retry immediately on every `ThrottlingException` with no delay until the call succeeds.
- **B.** Wait a fixed 30 seconds on every `ThrottlingException` regardless of retry count.
- **C.** On each retry, wait `(2^attempt) + random_jitter` seconds before the next attempt, up to a configured maximum retry count.
- **D.** Switch to a different AWS region on `ThrottlingException` and retry there with no delay.

<details><summary>Answer & explanation</summary>

**Correct: C.**

The standard exponential backoff with jitter pattern — `wait = (2^attempt) + random_jitter` — distributes retry load across time, avoiding retry storms. This is the AWS Well-Architected recommended pattern for handling `ThrottlingException` from any AWS API, including Bedrock.

- **A** is wrong — immediate retries without delay aggravate throttling by maintaining peak request rate.
- **B** is wrong — fixed delays do not adapt to load and cause unnecessary slowdown when the throttle lifts early.
- **D** is wrong — cross-region fallback is a valid availability strategy but not a substitute for backoff; it also doesn't guarantee quota relief.

</details>

---

## Domain 3 — AI Safety, Security, and Governance (20%)

### Q30. [MCQ]

A financial services company deploys a Bedrock-powered investment advice assistant. They must ensure the model never provides advice on unregistered securities. They define a "unregistered securities advice" denied topic. Which Bedrock Guardrails component enforces this restriction?

- **A.** Sensitive information filters
- **B.** Content filters (harmful content categories)
- **C.** Denied topics
- **D.** Contextual grounding checks

<details><summary>Answer & explanation</summary>

**Correct: C.**

Denied topics in Bedrock Guardrails let you define a set of specific topics to avoid within a generative AI application. The AWS example of "illegal investment advice" in a banking assistant is precisely this use case. Content filters handle categories like hate, violence, and sexual content — not business-specific topic restrictions.

- **A** is wrong — sensitive information filters detect and redact PII formats (SSN, DOB, etc.), not business topic restrictions.
- **B** is wrong — content filters address predefined harmful content categories, not custom business domain restrictions.
- **D** is wrong — contextual grounding checks validate factual accuracy against retrieved sources, not topic restrictions.

</details>

---

### Q31. [MCQ]

A developer wants to mathematically verify that a Bedrock-powered insurance chatbot only recommends coverage options that are actually listed in the approved product catalog, using logical rule verification rather than probabilistic content filtering. Which Bedrock Guardrails feature provides this capability?

- **A.** Contextual grounding checks
- **B.** Automated Reasoning checks
- **C.** Word filters
- **D.** Sensitive information filters

<details><summary>Answer & explanation</summary>

**Correct: B.**

Automated Reasoning checks validate model responses against logical rules and policies defined in natural language. They use "mathematically sound, logic-based algorithmic verification" to verify compliance — achieving up to 99% accuracy. The use case of ensuring a chatbot only recommends available products is the exact example given in AWS documentation for Automated Reasoning checks.

- **A** is wrong — contextual grounding checks verify that responses are factually grounded in provided reference text (useful for hallucination detection in RAG), not for enforcing logical rules against a catalog.
- **C** is wrong — word filters block specific words/phrases by exact match.
- **D** is wrong — sensitive information filters detect and redact PII.

</details>

---

### Q32. [MCQ]

A company detects that users are attempting prompt injection attacks against their Bedrock chatbot — sending user messages designed to override system instructions and exfiltrate data. Which Bedrock Guardrails feature is SPECIFICALLY designed to detect and block prompt injection?

- **A.** Denied topics, configured with the topic "prompt injection"
- **B.** Prompt attacks filter (within content filters, Standard tier)
- **C.** Contextual grounding checks with a low relevance threshold
- **D.** Automated Reasoning checks with a "no system prompt override" policy

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation explicitly lists "prompt attacks" as a category within content filters: "this filter can help you detect and filter prompt attacks including jailbreaks, prompt injections, and prompt leakages (Standard tier only)." It is purpose-built for this threat.

- **A** is wrong — denied topics block specific business-domain topics, not adversarial prompt manipulation techniques.
- **C** is wrong — contextual grounding checks compare responses to a reference source for factual accuracy, not for detecting adversarial prompt structure.
- **D** is wrong — Automated Reasoning checks enforce logical policy compliance, not adversarial input detection.

</details>

---

### Q33. [MRQ — choose TWO]

A company must ensure that their Bedrock application complies with the principle of least privilege. They want the Bedrock service role to access only a specific S3 bucket for Knowledge Base ingestion and a specific KMS key for decryption. Which TWO IAM best practices BEST implement this?

- **A.** Attach an AWS-managed policy `AmazonBedrockFullAccess` to the service role.
- **B.** Write a custom IAM policy that specifies the exact S3 bucket ARN in the `Resource` element and allows only `s3:GetObject` and `s3:ListBucket`.
- **C.** Add a `kms:Decrypt` permission in the custom policy scoped to the specific KMS key ARN.
- **D.** Use a permission boundary that allows all `bedrock:*` actions and all `s3:*` actions.
- **E.** Attach the `AdministratorAccess` managed policy temporarily and remove it after setup.

<details><summary>Answer & explanation</summary>

**Correct: B and C.**

Least privilege requires scoping permissions to the specific resource ARNs and minimum required actions. B scopes S3 access to the exact bucket with only the needed actions. C scopes KMS access to the exact key ARN. Both together implement the principle of least privilege correctly.

- **A** is wrong — `AmazonBedrockFullAccess` is far broader than needed and violates least privilege.
- **D** is wrong — a permission boundary that allows all S3 actions is too broad.
- **E** is wrong — temporarily attaching `AdministratorAccess` is a security anti-pattern and an exam red flag.

</details>

---

### Q34. [MCQ]

A multinational enterprise requires that their Bedrock foundation model invocations never leave a specific AWS region and that API calls originate from within a VPC. Which combination of configurations enforces both requirements?

- **A.** Enable cross-region inference profiles and restrict S3 bucket policies to a single region.
- **B.** Create a VPC endpoint (PrivateLink) for the Bedrock service and configure an endpoint policy that denies calls that would trigger cross-region inference.
- **C.** Use Amazon CloudFront in front of the Bedrock endpoint to enforce regional routing.
- **D.** Configure a Service Control Policy (SCP) that denies `bedrock:InvokeModel` for all regions except the approved one, without a VPC endpoint.

<details><summary>Answer & explanation</summary>

**Correct: B.**

A VPC Interface Endpoint (PrivateLink) for `bedrock-runtime` ensures API calls originate from within the VPC and traffic stays on the AWS private network. An endpoint policy on that VPC endpoint can restrict which models and regions can be invoked, preventing cross-region inference. SCPs control which accounts/OUs can call APIs but don't enforce VPC origin.

- **A** is wrong — cross-region inference profiles are the opposite of what is needed; this increases, not restricts, regional spread.
- **C** is wrong — CloudFront is a CDN for HTTP content, not a mechanism for Bedrock API routing.
- **D** is wrong — an SCP can restrict which regions are callable but does not enforce that calls originate from within a VPC.

</details>

---

### Q35. [MCQ]

A developer wants to apply Bedrock Guardrails to a custom RAG pipeline that does NOT use the Bedrock Knowledge Base or Bedrock Agents — just direct `InvokeModel` calls from a Python application. Can Guardrails still be applied, and if so, how?

- **A.** No; Guardrails only work when using Bedrock Knowledge Bases or Agents.
- **B.** Yes; pass the `guardrailIdentifier` and `guardrailVersion` parameters in the `InvokeModel` or `Converse` API request body.
- **C.** Yes; attach the Guardrail to the model using the Bedrock console, and it applies automatically to all calls to that model.
- **D.** Yes; but only via the AWS Management Console, not the API or SDK.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Bedrock Guardrails can be applied to any inference call — `InvokeModel`, `Converse`, `ConverseStream` — by including the `guardrailIdentifier` and `guardrailVersion` (or `guardrailConfig`) parameters in the request. They are not restricted to Knowledge Base or Agents workflows.

- **A** is wrong — Guardrails are decoupled from Knowledge Bases and Agents; they apply at the inference API layer.
- **C** is wrong — there is no "attach to model" mechanism that automatically applies a Guardrail to all calls; it must be specified per-request or at the prompt/application level.
- **D** is wrong — full API and SDK support is available.

</details>

---

### Q36. [MRQ — choose TWO]

A company's AI governance team reviews Amazon Bedrock model invocation logs and finds that Guardrail-blocked content appears as plain text in the logs, including PII that was supposed to be redacted. Which TWO mitigations address this gap?

- **A.** Disable model invocation logging entirely to prevent any PII from appearing in logs.
- **B.** Route invocation logs to an S3 bucket with a customer-managed KMS key and a bucket policy restricting access to only the security team's IAM role.
- **C.** Apply an S3 Object Lock policy to the logging bucket so blocked content cannot be deleted.
- **D.** Enable CloudTrail data events on the logging bucket to detect unauthorized access.
- **E.** Add an S3 bucket policy that denies `s3:GetObject` on the logging bucket for all principals except an approved security auditor role.

<details><summary>Answer & explanation</summary>

**Correct: B and E.**

AWS documentation states: "All blocked content from the above [Guardrail] policies will appear as plain text in Amazon Bedrock Model Invocation Logs, if you have enabled them." The mitigations are: (B) encrypt the logs with a CMK so only authorized parties can decrypt them, and (E) restrict read access via bucket policy to an approved auditor role. Together these ensure PII in blocked content is encrypted at rest and access-controlled.

- **A** is wrong — disabling logging entirely removes audit capability, which is typically a compliance requirement.
- **C** is wrong — S3 Object Lock prevents deletion, but does not restrict who can read the plaintext PII.
- **D** is wrong — CloudTrail data events detect access but do not prevent unauthorized reads.

</details>

---

### Q37. [MCQ]

A company must ensure that all Bedrock model responses in their RAG application are factually consistent with the retrieved document passages, flagging or blocking responses that add information not present in the retrieved context. Which Guardrail feature addresses this MOST directly?

- **A.** Automated Reasoning checks
- **B.** Contextual grounding checks
- **C.** Content filters set to maximum strength
- **D.** Denied topics configured with the topic "hallucination"

<details><summary>Answer & explanation</summary>

**Correct: B.**

Contextual grounding checks are specifically designed for RAG: they compare the model response against the provided reference source and the user query, detecting when a response "is not grounded (factually inaccurate or adds new information) in the source information or is irrelevant to the user's query." They can block or flag such responses. The supported use cases are summarization, paraphrasing, and question answering.

- **A** is wrong — Automated Reasoning checks enforce logical policy rules (e.g., catalog compliance), not factual grounding against retrieved text.
- **C** is wrong — content filters address harmful content categories (hate, violence, sexual), not factual accuracy.
- **D** is wrong — denied topics block specific subject areas, not hallucinated facts.

</details>

---

### Q38. [MCQ]

A developer discovers that the Standard tier of Bedrock Guardrails is required for a specific capability. Which TWO capabilities are EXCLUSIVELY available in the Standard tier (not the Basic/default tier)?

- **A.** Denied topics
- **B.** Prompt attacks detection (jailbreaks, prompt injection, prompt leakage)
- **C.** Extension of content filters to code domains (code comments, variable names, string literals)
- **D.** Sensitive information filters (PII redaction)
- **E.** Word filters

<details><summary>Answer & explanation</summary>

**Correct: B and C.**

AWS documentation explicitly tags two capabilities as "Standard tier only": (1) Prompt attacks detection including jailbreaks, prompt injections, and prompt leakages; (2) Detection of harmful content extended to code domains (comments, variable/function names, string literals). The other options (denied topics, PII filters, word filters) are available in the basic/standard tier.

- **A** is wrong — denied topics are available in the basic configuration.
- **D** is wrong — sensitive information filters are available regardless of tier.
- **E** is wrong — word filters are a basic capability.

</details>

---

### Q39. [MCQ]

An enterprise is building a Bedrock application where different teams should be able to invoke models but NOT be able to create or delete Guardrails or Knowledge Bases. Which IAM approach BEST enforces this separation?

- **A.** Attach `AmazonBedrockFullAccess` to developer roles and rely on Guardrails to block misuse.
- **B.** Create a custom IAM policy that allows `bedrock:InvokeModel` and `bedrock:Converse` but explicitly denies `bedrock:CreateGuardrail`, `bedrock:DeleteGuardrail`, `bedrock:CreateKnowledgeBase`, and `bedrock:DeleteKnowledgeBase`.
- **C.** Use resource-based policies on the Bedrock service to restrict admin operations.
- **D.** Configure a VPC endpoint policy that blocks CreateGuardrail API calls.

<details><summary>Answer & explanation</summary>

**Correct: B.**

The correct IAM approach is to write a custom policy that allows only the inference actions developers need (`InvokeModel`, `Converse`) and explicitly denies the administrative actions (`CreateGuardrail`, `DeleteGuardrail`, `CreateKnowledgeBase`, `DeleteKnowledgeBase`). Explicit denies override any allows, enforcing the separation.

- **A** is wrong — `AmazonBedrockFullAccess` grants all Bedrock permissions including administrative ones.
- **C** is wrong — Bedrock does not use resource-based policies on the service itself (unlike S3 bucket policies).
- **D** is wrong — VPC endpoint policies control network-level access, not fine-grained API action restrictions based on team identity.

</details>

---

## Domain 4 — Operational Efficiency and Optimization (12%)

### Q40. [MCQ]

A developer notices that Bedrock `InvokeModel` calls are consistently returning `ThrottlingException` during business hours. The application serves real-time user queries that cannot tolerate queuing delays. The team's budget is fixed. Which optimization addresses the throttling with the LEAST operational overhead?

- **A.** Submit all queries as a batch inference job to bypass real-time quotas.
- **B.** Purchase Provisioned Throughput for the model to guarantee dedicated capacity.
- **C.** Enable prompt caching so that repeated system-prompt tokens do not consume TPM quota.
- **D.** Switch to a smaller, less capable model that has higher default quota limits.

<details><summary>Answer & explanation</summary>

**Correct: C.**

Prompt caching explicitly states that cached tokens do NOT count toward TPM quotas. If the system prompt is large (e.g., 4,096+ tokens), caching it means each real-time call consumes significantly fewer TPM-quota tokens, reducing throttling without infrastructure changes. This is the least-overhead option if cached tokens are the dominant quota consumer.

- **A** is wrong — batch inference is asynchronous and unsuitable for real-time queries.
- **B** is wrong — Provisioned Throughput is the right long-term solution but involves a commitment and cost, making it higher overhead than enabling prompt caching.
- **D** is wrong — switching models has capability tradeoffs and is a broader architectural change.

</details>

---

### Q41. [MCQ]

A company runs a Bedrock RAG application with highly variable traffic — idle overnight and peaking midday. They want to monitor both latency and token usage in real time to detect performance degradation before users are impacted. Which AWS service and metric combination should they configure?

- **A.** AWS X-Ray traces with `bedrock:InvokeModel` segments and a custom segment for retrieval latency
- **B.** Amazon CloudWatch metrics for `InvocationLatency`, `InputTokenCount`, and `OutputTokenCount` with CloudWatch Alarms on percentile thresholds (p99)
- **C.** AWS Cost Explorer with hourly granularity to detect token usage spikes
- **D.** Amazon EventBridge rules triggered by `ThrottlingException` CloudTrail events

<details><summary>Answer & explanation</summary>

**Correct: B.**

Amazon CloudWatch is the primary operational monitoring service for Bedrock. Bedrock publishes `InvocationLatency`, `InputTokenCount`, `OutputTokenCount`, and `ThrottleCount` metrics. Setting CloudWatch Alarms on p99 latency and token count thresholds enables proactive alerting before user impact.

- **A** is wrong — X-Ray is useful for distributed tracing but is not the primary latency/quota monitoring tool for Bedrock; Bedrock does not publish standard X-Ray segments natively.
- **C** is wrong — Cost Explorer operates at a billing dimension and is not real-time; it is not suitable for detecting latency degradation.
- **D** is wrong — EventBridge on CloudTrail events captures after-the-fact management events, not real-time inference metrics.

</details>

---

### Q42. [MCQ]

A company runs a Bedrock application with a mix of simple FAQ queries (average 500 tokens total) and complex multi-document synthesis queries (average 8,000 tokens total). They want to minimize cost while maintaining quality. Which approach is MOST cost-effective?

- **A.** Route all queries to the same large, high-quality model to ensure consistent output quality.
- **B.** Use Bedrock Intelligent Prompt Routing to route simple queries to a smaller model and complex queries to a larger model automatically.
- **C.** Process all queries via batch inference regardless of urgency to get the 50% discount.
- **D.** Fine-tune a small model on FAQ pairs so it handles all queries.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Intelligent Prompt Routing is purpose-built for exactly this mixed-difficulty scenario: it routes "a meaningful fraction of requests [that] are simple enough for a smaller, faster model" to the cheaper model while sending complex requests to the larger model. This achieves cost reduction without sacrificing quality on hard queries.

- **A** is wrong — routing all queries to a large model overpays for simple FAQ queries.
- **C** is wrong — batch inference is asynchronous and not appropriate for interactive queries that need real-time responses.
- **D** is wrong — fine-tuning a small model on FAQs helps with FAQ queries but would degrade quality on complex synthesis tasks.

</details>

---

### Q43. [MCQ]

A developer wants to reduce the latency of time-to-first-token for interactive Bedrock inference calls, especially for a large language model. They are willing to accept a slightly different internal implementation path. Which Bedrock feature is designed specifically to reduce TTFT?

- **A.** Batch inference with a small batch size of 1
- **B.** Prompt caching with a 1-hour TTL
- **C.** Latency-optimized inference via `performanceConfig={"latency": "optimized"}`
- **D.** Cross-region inference profiles

<details><summary>Answer & explanation</summary>

**Correct: C.**

Latency-optimized inference is a Bedrock feature (preview) that uses a different serving configuration to reduce TTFT. It is invoked by setting `performanceConfig={"latency": "optimized"}` in the runtime API call. If quotas are exceeded or token thresholds are breached, it falls back to standard.

- **A** is wrong — batch inference with a batch size of 1 is still asynchronous and adds overhead compared to on-demand.
- **B** is wrong — prompt caching reduces quota consumption and cost, but its primary latency benefit is eliminating re-processing of the cached prefix, not reducing TTFT for novel prompts.
- **D** is wrong — cross-region inference profiles improve availability/capacity by routing across regions, not specifically TTFT.

</details>

---

### Q44. [MCQ]

A company wants to evaluate whether switching from on-demand Bedrock inference to Provisioned Throughput (PT) would save money. Their application processes a sustained 1,000 requests per minute, 24/7, every day of the month. Which statement about PT economics is MOST accurate?

- **A.** PT is always cheaper than on-demand regardless of request volume.
- **B.** PT is billed per model unit (MU) per hour; at sustained high utilization (24/7), PT becomes more cost-effective than on-demand because reserved capacity amortizes the hourly rate.
- **C.** PT is cheaper only for batch inference jobs, not for real-time inference.
- **D.** PT eliminates all token charges; you only pay the hourly MU rate.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Provisioned Throughput provides dedicated model-invocation capacity billed per MU per hour. For sustained, consistently high traffic (24/7 at 1,000 RPM), the amortized hourly MU cost becomes lower than per-token on-demand pricing. PT is the right fit for "flat, sustained, interactive traffic." At low or variable utilization, PT can cost more than on-demand.

- **A** is wrong — at low utilization, PT overhead exceeds on-demand savings.
- **C** is wrong — PT is designed for real-time inference; batch inference has its own separate pricing.
- **D** is wrong — PT pricing covers the capacity reservation; token-based metrics still apply for tracking, though billing is by the hour rather than per token.

</details>

---

### Q45. [MCQ]

A team uses Amazon Bedrock's `CreateModelInvocationJob` API for batch inference. After submitting a job, they want to monitor its progress. Which mechanism should they use?

- **A.** Poll `GetModelInvocationJob` and check the `status` field (e.g., `Submitted`, `InProgress`, `Completed`, `Failed`).
- **B.** Subscribe to an SNS topic automatically created by Bedrock when the job is submitted.
- **C.** Check the S3 output bucket; Bedrock writes a `job_status.json` file after each processed record.
- **D.** Use CloudWatch Events to detect when the S3 input file is deleted, indicating completion.

<details><summary>Answer & explanation</summary>

**Correct: A.**

The `GetModelInvocationJob` API returns the current job status (`Submitted`, `InProgress`, `Completed`, `Failed`). This is the documented way to check batch inference job progress. Results appear in the specified S3 output bucket as a JSONL file when the job completes.

- **B** is wrong — Bedrock does not automatically create an SNS topic for batch inference job status.
- **C** is wrong — Bedrock writes results to the output JSONL file, not a `job_status.json` file.
- **D** is wrong — the S3 input file is not deleted by Bedrock; this is not a completion signal.

</details>

---

## Domain 5 — Testing, Validation, and Troubleshooting (11%)

### Q46. [MCQ]

A team has deployed a Bedrock-powered RAG application and wants to systematically evaluate whether the retrieved chunks are relevant to the user queries, and whether the model's responses are faithful to the retrieved chunks. Which Amazon Bedrock native feature addresses BOTH requirements?

- **A.** Amazon Bedrock Model Evaluation with human workers
- **B.** Amazon Bedrock Knowledge Base RAG evaluation with citation precision and citation coverage metrics, combined with LLM-as-a-judge for response faithfulness
- **C.** Amazon CloudWatch Contributor Insights on Bedrock invocation logs
- **D.** AWS Trusted Advisor checks for Bedrock Knowledge Base optimization

<details><summary>Answer & explanation</summary>

**Correct: B.**

Amazon Bedrock's RAG evaluation feature (GA since March 2025) includes citation coverage and citation precision metrics that directly measure retrieval relevance. LLM-as-a-judge (also GA since March 2025) allows an evaluator model to score the generator model's responses for faithfulness and other criteria. Together they cover retrieval quality AND response quality.

- **A** is wrong — human evaluation is expensive, slow, and addresses only response quality, not retrieval quality.
- **C** is wrong — CloudWatch Contributor Insights analyzes metric trends, not semantic evaluation of responses.
- **D** is wrong — Trusted Advisor does not evaluate RAG response or retrieval quality.

</details>

---

### Q47. [MCQ]

A developer is using LLM-as-a-judge in Amazon Bedrock Model Evaluation with built-in metrics. They want to use Amazon Nova Pro as the judge model. Which model IDs are valid evaluator models for built-in metrics in Bedrock Model Evaluation?

- **A.** Only Amazon Titan Text Express and Amazon Titan Text Lite are supported as judge models.
- **B.** Amazon Nova Pro (`amazon.nova-pro-v1:0`) and several Anthropic Claude models (e.g., Claude 3.5 Sonnet, Claude Sonnet 4) are supported as judge models for built-in metrics.
- **C.** Any model available in the Bedrock model catalog can be used as a judge model.
- **D.** Only Meta Llama 3.1 70B Instruct is supported as a judge model for built-in metrics.

<details><summary>Answer & explanation</summary>

**Correct: B.**

AWS documentation lists the supported evaluator models for built-in metrics, which include Amazon Nova Pro, Amazon Nova 2 Lite, Amazon Nova Micro, Amazon Nova Premier, Anthropic Claude 3.5 Sonnet (v1 and v2), Claude 3.7 Sonnet, Claude Sonnet 4, Claude Haiku models, Claude Opus 4.5, Meta Llama 3.1 70B Instruct, and Mistral Large. Amazon Titan models are not listed as judge models; not all catalog models are eligible.

- **A** is wrong — Titan Text models are not listed as supported judge models.
- **C** is wrong — only specific models appear on the supported evaluator model list.
- **D** is wrong — Meta Llama 3.1 70B is one of several options, not the only one.

</details>

---

### Q48. [MCQ]

A developer's Bedrock Knowledge Base RAG application is returning low-quality, incomplete answers on multi-part questions. Evaluation shows that citation coverage is low — only 1–2 of the 5 expected document chunks are being retrieved. What is the FIRST troubleshooting step?

- **A.** Increase the model's `maxTokens` parameter to allow longer responses.
- **B.** Review the chunking strategy: increase chunk size or overlap, and verify the retrieval configuration's `numberOfResults` parameter is set high enough to surface all relevant chunks.
- **C.** Switch the vector store from OpenSearch Serverless to Aurora pgvector for better recall.
- **D.** Enable cross-region inference to access models with larger context windows.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Low citation coverage (few relevant chunks retrieved) is a retrieval problem, not a generation problem. The first step is to check the retrieval configuration: ensure `numberOfResults` is not set too low (e.g., 1 or 2 when 5 are needed), and consider whether chunk size/overlap is appropriate so relevant content isn't split across chunks that individually fall below the similarity threshold.

- **A** is wrong — `maxTokens` controls response length, not retrieval recall.
- **C** is wrong — the vector store type is rarely the primary cause of retrieval recall issues; configuration and chunking strategy come first.
- **D** is wrong — cross-region inference addresses model availability and capacity, not retrieval recall.

</details>

---

### Q49. [MCQ]

A Bedrock agent invocation is intermittently failing with `ModelTimeoutException`. The agent's action group invokes a Lambda function that queries an internal database. What is the MOST likely root cause?

- **A.** The Bedrock model's context window is exceeded by the conversation history.
- **B.** The Lambda function is exceeding the response timeout window allowed by Bedrock Agents for action group invocations.
- **C.** The model's output token limit is too low; increase `maxTokens` in the model configuration.
- **D.** The agent's instruction prompt is triggering Bedrock Guardrails, which introduces latency until it times out.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Bedrock Agents impose a timeout on action group Lambda invocations. If the Lambda function's database query is slow (e.g., cold start + slow query), the agent can time out waiting for the tool result, producing `ModelTimeoutException`. The fix is to optimize Lambda performance (warm up, query optimization) or increase allowed execution time if the Agents configuration allows it.

- **A** is wrong — context window overflow would produce an error about token limits, not a timeout.
- **C** is wrong — `maxTokens` limits output generation length; a low value would truncate output, not cause a timeout exception.
- **D** is wrong — Guardrails adds milliseconds of latency, not seconds that would trigger a model timeout exception.

</details>

---

### Q50. [MCQ]

A developer is evaluating two fine-tuned models: Model A (fine-tuned on domain data) and Model B (the base model). They run a Bedrock Model Evaluation job using LLM-as-a-judge. The judge assigns higher scores to Model A on accuracy but lower scores on coherence. What is the MOST appropriate next action?

- **A.** Immediately deploy Model A because accuracy is more important than coherence.
- **B.** Analyze the judge model's per-prompt explanation output to understand which prompt types cause coherence degradation in Model A, then decide whether to adjust fine-tuning or prompt engineering.
- **C.** Discard Model A because any coherence regression disqualifies it.
- **D.** Switch to a human evaluation panel because LLM-as-a-judge cannot be trusted for coherence scoring.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Bedrock's LLM-as-a-judge provides per-prompt explanations (visible in the console for the first 5 prompts and in the full S3 report). These explanations reveal which specific prompt types or content patterns cause the coherence regression. This targeted analysis should drive the decision — perhaps coherence issues appear only on certain question types that can be addressed with prompt engineering, or the fine-tuning dataset needs curation.

- **A** is wrong — deploying without understanding coherence issues risks a degraded user experience.
- **C** is wrong — a blanket rejection ignores that accuracy improved; a nuanced analysis may show the tradeoff is acceptable or fixable.
- **D** is wrong — LLM-as-a-judge is a validated evaluation approach in Bedrock (GA since March 2025) and is appropriate for coherence scoring; switching to humans is slower and more expensive.

</details>

---

## Mixed Review

### Q51. [MCQ]

A developer receives a `ValidationException: The provided model identifier is not supported` error when calling `bedrock-runtime:InvokeModel`. What are the TWO most common causes to check first? *(Select the single BEST investigative path.)*

- **A.** The model ID format is incorrect (e.g., using the display name instead of the model ID), or the model has not been granted access in the Bedrock console for that region.
- **B.** The IAM role lacks `kms:Decrypt` for the model's encryption key.
- **C.** The VPC endpoint policy is blocking the specific model ID.
- **D.** The foundation model has been deprecated and must be replaced with a newer version.

<details><summary>Answer & explanation</summary>

**Correct: A.**

`ValidationException` for an unsupported model identifier almost always means either (1) the model ID string is wrong (e.g., using a display name like "Claude 3 Sonnet" instead of the model ID `anthropic.claude-3-sonnet-20240229-v1:0`), or (2) model access has not been requested and granted in the Bedrock console for the current region. Both can be resolved quickly without infrastructure changes.

- **B** is wrong — missing `kms:Decrypt` would produce an `AccessDeniedException`, not a `ValidationException`.
- **C** is wrong — VPC endpoint policy issues produce `AccessDeniedException`, not `ValidationException`.
- **D** is wrong — deprecated models produce a different error message specific to deprecation, and deprecated models often still accept requests until a cutoff date.

</details>

---

### Q52. [MRQ — choose TWO]

A team migrating from direct `InvokeModel` calls to the Converse API wants to understand the key advantages. Which TWO statements are accurate?

- **A.** The Converse API provides a unified request/response schema across all supported models, eliminating model-specific body formatting.
- **B.** The Converse API supports multi-turn conversation history natively via the `messages` array, maintaining context without manual prompt assembly.
- **C.** The Converse API is cheaper per token than `InvokeModel` because it uses a compression algorithm on the request payload.
- **D.** The Converse API only works with Amazon Nova models; Anthropic Claude models require `InvokeModel`.
- **E.** The Converse API eliminates the need for IAM permissions on `bedrock-runtime:InvokeModel`.

<details><summary>Answer & explanation</summary>

**Correct: A and B.**

The two documented benefits of the Converse API are: (1) a unified request schema that abstracts away model-specific payload formats, and (2) native `messages` array support for multi-turn conversations including tool results. It requires `InvokeModel` action permission under the hood (`ConverseStream` needs `InvokeModelWithResponseStream`), works across providers including Anthropic, and does not change per-token pricing.

- **C** is wrong — Converse API pricing is the same as InvokeModel; there is no compression discount.
- **D** is wrong — the Converse API supports multiple providers including Anthropic Claude models.
- **E** is wrong — Converse API requires `bedrock:InvokeModel` (and `bedrock:InvokeModelWithResponseStream` for ConverseStream) permissions.

</details>

---

### Q53. [MCQ]

A company is evaluating whether to use RAG or model distillation for a new product. They have a high-quality teacher model that performs excellently on their internal benchmark, and they want a cheaper model for production inference, but their knowledge is entirely static (the product manual is updated once per year). Which approach is MOST appropriate?

- **A.** RAG with a Bedrock Knowledge Base, re-synced annually.
- **B.** Model distillation to train a smaller student model using the teacher model's outputs on the static corpus.
- **C.** Continued pre-training (domain adaptation) of the student model on the static corpus.
- **D.** Instruction fine-tuning of the student model on a synthetic QA dataset generated by the teacher.

<details><summary>Answer & explanation</summary>

**Correct: B.**

Distillation is specifically designed to transfer the capability of a large teacher model into a smaller, cheaper student model. With a static corpus (updated once per year), distilling the teacher's knowledge is cost-effective: you pay for one distillation job per year and get cheap inference year-round. RAG would also work but adds per-query retrieval cost and infrastructure; distillation eliminates inference-time retrieval entirely.

- **A** is wrong — RAG is better for dynamic knowledge; for fully static content, distillation offers cheaper inference without per-query retrieval overhead.
- **C** is wrong — continued pre-training (domain adaptation) uses unlabeled text and does not transfer the specific task capability or quality of the teacher model.
- **D** is wrong — instruction fine-tuning on synthetic QA is a reasonable alternative but produces a task-specific model; distillation more broadly captures the teacher's general capability across the task distribution.

</details>

---

### Q54. [MRQ — choose TWO]

A company wants to evaluate models for their application using Amazon Bedrock Model Evaluation. Which TWO generator model types are supported in a Bedrock Model Evaluation job?

- **A.** Foundation models available in Bedrock
- **B.** Custom fine-tuned models imported via Bedrock Custom Model Import
- **C.** Models accessed through a completely separate SageMaker endpoint, without any Bedrock integration
- **D.** Any GPT-4o model via direct OpenAI API credentials
- **E.** Amazon Bedrock Marketplace models

<details><summary>Answer & explanation</summary>

**Correct: A and E.**

AWS documentation lists supported generator model types for Bedrock Model Evaluation: foundation models, customized foundation models, imported foundation models, models invoked through the OpenAI Responses API on the bedrock-mantle endpoint, Amazon Bedrock Marketplace models, prompt routers, and Provisioned Throughput models. All of these are Bedrock-mediated.

- **C** is wrong — standalone SageMaker endpoints without Bedrock integration are not directly supported; you would need to import the model into Bedrock first.
- **D** is wrong — direct OpenAI API credentials (external to AWS) are not supported; only models accessed via the bedrock-mantle OpenAI Responses API compatibility layer are supported.

</details>

---

### Q55. [MCQ]

A startup is building a coding assistant that streams responses character by character to an IDE plugin. The assistant must also call an internal code linting tool during generation. Which combination of Bedrock features BEST supports this scenario with the LEAST custom code?

- **A.** `InvokeModel` for streaming + a separate polling loop for tool calls
- **B.** `ConverseStream` with `toolConfig` defined; handle `contentBlockDelta` events for streaming text and `toolUse` events for tool invocation, then resume with `toolResult`
- **C.** Batch inference with a tool call field in the JSONL record
- **D.** Bedrock Knowledge Base `RetrieveAndGenerate` with a Lambda action group for linting

<details><summary>Answer & explanation</summary>

**Correct: B.**

`ConverseStream` is the streaming variant of the Converse API and natively supports tool use in a streaming context. The response stream emits `contentBlockDelta` events for streamed text and `toolUse` events when the model decides to call a tool. The developer handles the tool call and resumes the stream with a `toolResult` message. This minimizes custom code compared to building streaming + tool call handling from scratch on `InvokeModelWithResponseStream`.

- **A** is wrong — `InvokeModel` is synchronous; it does not stream.
- **C** is wrong — batch inference is asynchronous and doesn't support real-time streaming.
- **D** is wrong — `RetrieveAndGenerate` is a RAG API, not a coding assistant streaming API; it does not support arbitrary tool calls like code linting.

</details>

---

## Score Guide

| Raw score (out of 65 scored questions) | Approximate scaled score | Readiness |
|----------------------------------------|--------------------------|-----------|
| < 35 (< 54%) | < 600 | Not ready — significant gaps; revisit AWS docs per domain |
| 35–42 (54–65%) | 600–699 | Below passing — review wrong answers and weak domains |
| 43–48 (66–74%) | 700–749 | Close to passing — targeted review of missed topics needed |
| **49–52 (75–80%)** | **750–799** | **At the passing threshold — refine and schedule the exam** |
| 53–58 (82–89%) | 800–899 | Comfortably passing — practice edge cases to maximize score |
| 59–65 (91–100%) | 900–1000 | Mastery — ready for exam day |

> The pass mark is **750/1000**. On a 65-question scored exam this corresponds to approximately **75% correct answers**. AWS uses compensatory scoring — no per-domain minimum — so a strong Domain 1 score can offset a weaker Domain 5.

---

## Highest-Yield Reflexes for Exam Day

- **Provisioned Throughput + fine-tuned model → always pass `provisionedModelArn`, never the base model ID.**
- **Changing knowledge without retraining → RAG (Knowledge Base). Compressing a large model's capability into a small one → Distillation.**
- **Hallucination prevention in RAG → Contextual grounding checks. Policy/catalog compliance via logic → Automated Reasoning checks.**
- **Prompt injection / jailbreaks → Prompt attacks filter, Standard tier only.**
- **PHI + HIPAA on Bedrock → BAA first, then restrict invocation logs (disable or CMK + bucket policy).**
- **Blocked Guardrail content appears as plain text in invocation logs — always mitigate with CMK + bucket access restriction.**
- **S3 Vectors → best for infrequent/burst vector workloads (no provisioned infrastructure). Binary vectors → OpenSearch Serverless or OpenSearch Managed Clusters only.**
- **Neptune Analytics GraphRAG → multi-hop entity traversal. Scores below threshold → chunk exclusion (may return fewer than the configured limit).**
- **Prompt caching cached tokens → do NOT count toward TPM quotas. Default TTL = 5 minutes, extendable to 1 hour.**
- **Batch inference → `CreateModelInvocationJob` + S3 JSONL → ~50% off on-demand rate, asynchronous, separate quota.**
- **Converse/ConverseStream → unified schema across providers, native multi-turn + tool use. ConverseStream needs `InvokeModelWithResponseStream` IAM permission.**
- **AgentCore → framework-agnostic (Strands, LangGraph, Claude Agent SDK, Google ADK, OpenAI Agents SDK). A2A protocol for multi-agent.**
- **MCP → standardizes tool discovery and invocation; AgentCore hosts the infrastructure.**
- **LLM-as-a-judge GA March 2025. RAG evaluation includes citation precision + citation coverage.**
- **S3 Vectors metadata limit → 1 KB / 35 keys per vector. Exceeding this causes ingestion failures.**
- **MongoDB Atlas metadata filtering → NOT automatic; must manually configure vector index filters.**
- **Encryption type for S3 vector buckets → immutable after creation. Neptune graph requires vector search index set at creation time.**
- **Intelligent prompt routing → pairwise only (two models). Best for mixed-difficulty interactive traffic.**
- **Latency-optimized inference → set `performanceConfig={"latency": "optimized"}`. Falls back to standard if quotas exceeded.**

---

## References

1. **AWS Certified Generative AI Developer – Professional (AIP-C01) Exam Guide** — https://docs.aws.amazon.com/aws-certification/latest/examguides/ai-professional-01.html
2. **Amazon Bedrock Knowledge Bases — Vector store setup** — https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html
3. **Amazon Bedrock Guardrails — Components** — https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html
4. **Contextual grounding check** — https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html
5. **Automated Reasoning checks (Preview → GA)** — https://aws.amazon.com/about-aws/whats-new/2024/12/amazon-bedrock-guardrails-automated-reasoning-checks-preview
6. **GraphRAG — Build GraphRAG applications using Bedrock Knowledge Bases** — https://aws.amazon.com/blogs/machine-learning/build-graphrag-applications-using-amazon-bedrock-knowledge-bases/
7. **GraphRAG GA announcement** — https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-graphrag-generally-available
8. **Amazon Bedrock AgentCore** — https://aws.amazon.com/bedrock/agentcore/
9. **AgentCore Documentation** — https://docs.aws.amazon.com/bedrock-agentcore/
10. **Evaluate model performance using LLM-as-a-judge** — https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-judge.html
11. **RAG evaluation and LLM-as-a-judge GA** — https://aws.amazon.com/blogs/machine-learning/evaluate-models-or-rag-systems-using-amazon-bedrock-evaluations-now-generally-available
12. **Bedrock inference throughput and optimization** — https://hidekazu-konishi.com/entry/amazon_bedrock_inference_throughput_and_latency_optimization.html
13. **Amazon S3 Vectors** — https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html
14. **AWS Prescriptive Guidance — Choosing a vector database** — https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html
15. **Bedrock Guardrails overview** — https://aws.amazon.com/bedrock/guardrails/
