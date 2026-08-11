# Domain 5: Testing, Validation, and Troubleshooting

**Domain weight: 11% of scored questions (AIP-C01)**

This domain tests your ability to evaluate the quality of generative AI outputs, diagnose failures, and design resilient applications that degrade gracefully. Questions are scenario-heavy: you will be asked to pick the right evaluation method for a given situation, and to select the correct fix for a given error.

> **Plain English:** How do you know your GenAI app actually works? And what do you do when it breaks? This domain covers measuring quality (metrics, human review, LLM-as-judge), diagnosing the most common Bedrock errors, and building resilience so your app fails safely instead of catastrophically.

---

## Table of Contents

1. [Evaluation Frameworks and Metrics](#1-evaluation-frameworks-and-metrics)
   - 1.1 [Automated Text-Quality Metrics](#11-automated-text-quality-metrics)
   - 1.2 [Amazon Bedrock Model Evaluation](#12-amazon-bedrock-model-evaluation)
   - 1.3 [RAG-Specific Evaluation](#13-rag-specific-evaluation)
   - 1.4 [Human-in-the-Loop Review](#14-human-in-the-loop-review)
   - 1.5 [Regression Testing with Prompt Suites](#15-regression-testing-with-prompt-suites)
2. [Troubleshooting Common Errors](#2-troubleshooting-common-errors)
   - 2.1 [ThrottlingException (HTTP 429)](#21-throttlingexception-http-429)
   - 2.2 [ValidationException (HTTP 400)](#22-validationexception-http-400)
   - 2.3 [Token Limit and Context Window Errors](#23-token-limit-and-context-window-errors)
   - 2.4 [Timeouts and Latency Spikes](#24-timeouts-and-latency-spikes)
   - 2.5 [Poor Retrieval Quality in RAG](#25-poor-retrieval-quality-in-rag)
   - 2.6 [Hallucinations](#26-hallucinations)
   - 2.7 [Empty or Malformed Tool-Use Outputs](#27-empty-or-malformed-tool-use-outputs)
3. [Resilience Patterns](#3-resilience-patterns)
   - 3.1 [Graceful Degradation and Fallback Models](#31-graceful-degradation-and-fallback-models)
   - 3.2 [Retry with Exponential Backoff and Jitter](#32-retry-with-exponential-backoff-and-jitter)
   - 3.3 [Idempotency](#33-idempotency)
   - 3.4 [Dead-Letter Queues](#34-dead-letter-queues)
   - 3.5 [Error Handling Architecture](#35-error-handling-architecture)
4. [Glossary](#glossary)
5. [References](#references)

---

## 1. Evaluation Frameworks and Metrics

### 1.1 Automated Text-Quality Metrics

> **Why (the rationale):** Human evaluation is expensive and slow; automated metrics enable fast, cheap quality gates in CI/CD pipelines to catch regressions before they reach production. They are the baseline evaluation layer before more expensive LLM-as-a-judge or human review.
> **When to use:** Regression testing after model/prompt changes; bulk evaluation of many model outputs against a fixed ground truth set; integration into nightly CI pipelines. ROUGE for summarization, BERTScore for paraphrase-tolerant quality, BLEU for translation (legacy), Exact Match/F1 for extractive QA.
> **Nuances & gotchas:** BLEU and ROUGE penalize valid paraphrases — a response that says the same thing differently scores poorly. Perplexity measures **fluency, not factual correctness** — a hallucinated answer can have low perplexity (model is very confident about the wrong thing). None of these metrics measure faithfulness to a retrieved source; use Bedrock RAG Evaluation for that. Reference answers must be representative and high-quality — garbage-in, garbage-out for all automated metrics.

Automated metrics compare model outputs against human-authored **reference answers** without needing a judge model or human rater. They are fast and cheap, making them ideal for CI/CD regression gates.

| Metric | Full name | What it measures | Limitation |
|---|---|---|---|
| **BLEU** | Bilingual Evaluation Understudy | N-gram overlap between generated and reference text | Penalises valid paraphrases; poor for open-ended generation |
| **ROUGE** | Recall-Oriented Understudy for Gisting Evaluation | Recall of n-grams (ROUGE-N), longest common subsequence (ROUGE-L) | Similar n-gram brittleness to BLEU |
| **BERTScore** | — | Semantic similarity using contextual BERT embeddings; no exact n-gram matching | Requires a BERT-class model; less interpretable score |
| **Perplexity** | — | How surprised the model is by a text sequence (lower = model finds text more probable) | Measures fluency, not factual correctness |
| **Exact Match (EM)** | — | Percentage of predictions that exactly match the reference | Only useful for constrained tasks (QA with short answers) |
| **F1** | — | Token-level F1 between prediction and reference | Common in extractive QA benchmarks |

When to use which:
- **Summarisation, translation:** ROUGE-L and BERTScore.
- **Machine translation (historical):** BLEU.
- **Closed-domain QA:** Exact Match or F1.
- **Open-ended generation quality:** BERTScore or LLM-as-a-judge (Section 1.2).

#### 🎯 On the exam
- "Cheapest way to benchmark many model outputs against reference answers" → automated metrics (BLEU, ROUGE, BERTScore).
- "BLEU score is high but responses don't read well" → BLEU misses paraphrase quality; supplement with BERTScore or human eval.
- "Perplexity is low but the model hallucinates" → perplexity measures fluency, **not** factual accuracy.

---

### 1.2 Amazon Bedrock Model Evaluation

[Amazon Bedrock Model Evaluation](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html) provides three complementary evaluation modes, all accessible from the Bedrock console or API:

#### A. Automatic (Programmatic) Evaluation

> **Why (the rationale):** Provides a structured, repeatable baseline comparison of models or prompts using built-in or custom datasets — without writing evaluation infrastructure yourself. Ideal for rapid A/B testing of model versions before committing to a deployment.
> **When to use:** Fast regression checks after a model update, prompt version change, or RAG configuration change; comparing two candidate models on the same task.
> **Nuances & gotchas:** Automatic evaluation computes statistical metrics (accuracy, toxicity scores) — it is NOT an LLM-as-a-judge. It requires a ground truth dataset; without reference answers, only metrics like toxicity and perplexity can be computed. Results are available in S3 and the Bedrock console; there is no real-time streaming of evaluation progress.

- Runs built-in datasets or your **custom JSONL prompt dataset** against a target model.
- Computes metrics automatically: accuracy, robustness, toxicity, and task-specific scores.
- Returns a structured report with per-prompt scores.
- **Best for:** rapid baseline comparisons, regression testing after model or prompt changes.

#### B. Human Evaluation

- Routes model responses to **human workers** (your own workforce or AWS-managed workers) who rate responses on custom criteria (e.g., helpfulness, safety, tone).
- Uses an annotation UI; workers can compare two responses side by side (A/B preference).
- **Best for:** subjective quality criteria that automated metrics cannot capture; regulatory sign-off.

#### C. LLM-as-a-Judge

> **Why (the rationale):** Human evaluation is the gold standard but takes weeks and costs $$$. LLM-as-a-judge delivers near-human evaluation quality in hours at ~98% cost savings — making nuanced quality evaluation feasible at scale in production feedback loops.
> **When to use:** Evaluating open-ended generation quality (helpfulness, tone, correctness, harmlessness) where statistical metrics fail; comparing model versions on subjective criteria; ongoing production quality monitoring without human reviewer bottleneck.
> **Nuances & gotchas:** LLM-as-a-judge is NOT the same as Automatic evaluation — the judge is an LLM (Claude or Nova family) that reads and scores outputs, not a statistical formula. You can bring **your own model responses from any provider** as input; you are not limited to Bedrock-hosted models. Judge model bias is a real risk — judges tend to favor verbose, confident-sounding answers and responses stylistically similar to their own training. The judge provides a numeric score AND a natural-language explanation per response.

[LLM-as-a-judge](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-judge.html) is generally available (GA since March 2025). A second "judge" LLM scores the outputs of the model under evaluation.

Key facts:
- Evaluate metrics such as **correctness, completeness, faithfulness (hallucination detection)**, as well as responsible-AI metrics like **answer refusal** and **harmfulness**.
- **Up to 98% cost savings** compared to equivalent human evaluation, per AWS ([source](https://aws.amazon.com/blogs/machine-learning/llm-as-a-judge-on-amazon-bedrock-model-evaluation/)).
- Evaluation time reduced from weeks to **hours**.
- You can bring **your own inference responses** (from any model, anywhere) as input, so you are not limited to models hosted on Bedrock.
- The judge model provides both a numeric score and a **natural-language explanation** for each rating.
- Supported judge models include Anthropic Claude and Amazon Nova families on Bedrock.

**Workflow:**
1. Upload a prompt dataset (JSONL) to S3 — each row contains the prompt and optionally the expected/reference answer.
2. Create a model evaluation job; select judge model and metrics.
3. Bedrock invokes the model under evaluation, then passes (prompt, response, reference) to the judge model.
4. Review the report in the Bedrock console or download from S3.

#### D. Amazon Bedrock Agent Evaluations (Task 5.1.7)

> **Why (the rationale):** Standard LLM evaluation measures single-turn response quality; agent evaluation must capture correctness across **multi-step workflows** — whether the agent reached the goal, whether it called the right tools in the right order, and whether its reasoning stayed on track. Task completion rate and tool-usage effectiveness are not captured by BLEU/ROUGE or single-response faithfulness metrics.
> **When to use:** Before deploying or updating a Bedrock Agent — especially when action groups, knowledge bases, or orchestration prompts change. Also use for ongoing production quality monitoring of agent task completion.

[Amazon Bedrock Agent evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html) extend Bedrock Model Evaluation to **multi-step agentic workflows**. Key metrics differ from single-response evaluation:

| Metric | What it measures |
|---|---|
| **Task completion rate** | Did the agent successfully complete the end-to-end task? |
| **Tool-usage effectiveness** | Did the agent call the right tools with the right parameters? Were unnecessary tool calls avoided? |
| **Reasoning quality** | Is the chain-of-thought (trace) coherent and logically correct across steps? |
| **Step efficiency** | Did the agent complete the task in a reasonable number of steps (no loops or redundant actions)? |

**Key distinction from RAG/LLM evaluation:** Single-response evaluation scores one output against one expected answer. Agent evaluation tracks **sequences of actions and decisions** across a full workflow — a correct final answer reached via wrong tool calls still indicates a quality problem.

#### 🎯 On the exam (Agent Evaluations)
- "Evaluate whether a Bedrock Agent correctly calls an API action group before answering" → **Amazon Bedrock Agent evaluations** (task completion rate + tool-usage effectiveness), not standard LLM eval.
- "Distinguish agent evaluation from model evaluation" → agent eval tracks multi-step tool calls and reasoning across a workflow; model eval scores a single response.

#### 🎯 On the exam (all Model Evaluation modes)
- "Compare two prompts or two models on quality cheaply and quickly" → **LLM-as-a-judge** in Bedrock Model Evaluation.
- "Need human sign-off before deploying to production" → **Human Evaluation** job.
- "Automated nightly quality regression test" → **Automatic evaluation** job with a fixed prompt dataset.
- LLM-as-a-judge is NOT the same as Automatic evaluation — the judge is an LLM, not a statistical metric.

---

### 1.3 RAG-Specific Evaluation

RAG systems have two distinct failure modes: the **retriever** fetches the wrong chunks, or the **generator** produces answers that don't match the retrieved context. RAG evaluation measures both.

#### Amazon Bedrock RAG Evaluation (GA)

> **Why (the rationale):** RAG systems have two distinct failure modes — retrieval failure (wrong chunks) and generation failure (model ignores or misrepresents retrieved chunks). Generic text quality metrics cannot distinguish between these; RAG Evaluation provides metrics for each stage of the pipeline so you can target fixes precisely.
> **When to use:** After any change to chunking strategy, embedding model, top-k setting, vector store, or generation prompt — to verify you haven't regressed on retrieval or generation quality. Also use for initial baseline measurement before deploying a RAG system to production.
> **Nuances & gotchas:** RAG Evaluation uses an **LLM as the judge** — it is not a statistical metric, so judge model biases apply. Faithfulness measures consistency with retrieved context; it does NOT measure absolute factual correctness (the retrieved chunks themselves might be wrong). You must supply a ground truth dataset with `user_query` and `expected_response` fields; evaluation without reference answers only produces contextual metrics (faithfulness, context relevance) but not answer correctness.

[Amazon Bedrock RAG Evaluation](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-rag-evaluation-generally-available/) is GA as of March 2025. It evaluates Knowledge Bases or custom RAG pipelines using an **LLM-as-a-judge** to score each stage.

Supported metrics:

| Metric | What it measures | Failure it detects |
|---|---|---|
| **Faithfulness / Groundedness** | Does the generated answer stick to what the retrieved context actually says? | Hallucination — model invents facts not in retrieved chunks |
| **Context Relevance** | Are the retrieved chunks actually relevant to the user query? | Retrieval failure — poor embedding, chunking, or top-k setting |
| **Answer Relevance / Correctness** | Does the final answer address the user's question accurately? | End-to-end pipeline failure |
| **Completeness** | Does the answer cover all aspects of the question answered in the reference? | Incomplete retrieval or truncated generation |
| **Citation Precision** | Are the citations in the response accurate and traceable to retrieved chunks? | Fabricated citations |
| **Citation Coverage** | Do the retrieved chunks that support the answer have citations? | Uncited claims |

#### Setup

1. Prepare a dataset with: `user_query`, `expected_response` (ground truth), and optionally `retrieved_context`.
2. Create a RAG evaluation job in Bedrock, pointing to your Knowledge Base or providing custom RAG outputs.
3. Select a judge model and the metrics to compute.
4. Review the per-query scores and aggregate report.

#### Common RAG quality issues and fixes

| Symptom | Root cause | Fix |
|---|---|---|
| Low faithfulness score | Generator hallucinating beyond retrieved context | Add grounding instructions to system prompt; use guardrails; add a post-processing faithfulness check |
| Low context relevance | Retrieved chunks not matching query semantics | Improve chunking (smaller chunks, semantic boundaries); upgrade embedding model; tune top-k and similarity threshold |
| Low answer relevance | Retrieved context is relevant but answer misses the point | Improve the generation prompt; add few-shot examples |
| High latency in retrieval | Too many chunks retrieved (large top-k) | Reduce top-k; add a re-ranker to filter before generation |

#### 🎯 On the exam
- "RAG responses contain facts not in the knowledge base" → **faithfulness/groundedness** failure → add grounding instructions or a faithfulness guardrail.
- "RAG retrieves chunks that are off-topic" → **context relevance** failure → fix chunking or embedding model.
- "How to evaluate a Knowledge Base systematically" → **Bedrock RAG Evaluation** with an LLM judge.
- Bedrock Knowledge Bases evaluation supports metrics including citation precision and citation coverage in addition to the standard RAG metrics.

---

### 1.4 Human-in-the-Loop Review

When automated metrics are insufficient (e.g., nuanced tone, safety for regulated industries, subjective creativity), human review is required.

#### Amazon Augmented AI (Amazon A2I)

> **Why (the rationale):** Some decisions require human judgment that no automated metric can replace (safety-critical medical advice, high-stakes financial recommendations, nuanced compliance calls). A2I provides a managed routing mechanism to insert human review at specific trigger points in the inference pipeline without building custom review tooling.
> **When to use:** High-stakes or regulated applications where low-confidence model outputs need human verification before acting on them; cases where regulators require human sign-off; collecting human-reviewed examples for fine-tuning datasets.
> **Nuances & gotchas:** A2I introduces latency — the inference pipeline pauses until a reviewer completes the task; this is not suitable for sub-second SLA applications. Worker pool options (internal team, Mechanical Turk, AWS-vetted vendors) affect cost, quality, and PII handling — never route PHI to Mechanical Turk without explicit data processing agreements.

[Amazon A2I](https://docs.aws.amazon.com/sagemaker/latest/dg/a2i-use-augmented-ai-a2i-human-review-loops.html) integrates human review loops directly into ML inference pipelines:

- Define a **human loop** trigger (e.g., model confidence below threshold, random sampling percentage, or always review).
- Route flagged outputs to reviewers via a configurable UI task.
- Reviewers approve, reject, or edit model outputs.
- Reviewed data can be fed back for fine-tuning or used to update evaluation baselines.
- Worker pools: your own internal team, Amazon Mechanical Turk, or AWS-vetted vendors.

#### Amazon SageMaker Ground Truth

[SageMaker Ground Truth](https://docs.aws.amazon.com/sagemaker/latest/dg/sms.html) is used for **labelling datasets** at scale:

- Can generate human-labelled ground truth datasets for RAG evaluation or fine-tuning.
- Supports automatic labelling with active learning to reduce cost.
- Output can be used as the reference dataset for Bedrock Model Evaluation jobs.

#### 🎯 On the exam
- "Route low-confidence model outputs to human reviewers in production" → **Amazon A2I**.
- "Build a labelled dataset for evaluation or fine-tuning" → **SageMaker Ground Truth**.

---

### 1.5 Regression Testing with Prompt Suites

As models, prompts, and retrieved knowledge bases evolve, previously passing test cases can break. Establish a regression testing discipline:

1. **Build a golden dataset (Task 5.1.9 / 5.2):** a curated set of representative prompts paired with **known-good expected outputs** — covering key use cases, edge cases, and safety scenarios. Golden datasets are the ground truth used to **detect hallucinations and semantic drift**: run the dataset on every model or prompt update and compare faithfulness/accuracy scores against the baseline. A regression in faithfulness on the golden dataset gates the deployment. The same dataset drives both nightly CI checks and pre-production deployment gates.
2. **Automate evaluation:** run the golden dataset against the new model/prompt version using Bedrock Automatic Evaluation or LLM-as-a-judge.
3. **Gate deployments:** fail the CI/CD pipeline if any critical metric (e.g., faithfulness, exact-match accuracy) drops below threshold.
4. **Canary rollout of new model/prompt versions (Task 5.1.2):** after the evaluation gate passes, deploy the new version to a **small slice of live traffic** (e.g., 5–10%) before a full rollout — analogous to a canary deployment in software engineering. Monitor quality metrics (faithfulness, error rate, latency) on canary traffic in real time. If quality or error regressions exceed a threshold, **automatically roll back** to the previous version; if metrics hold, gradually shift 100% of traffic to the new version. This limits blast radius for prompt regressions or model behaviour changes that automated tests did not catch.
5. **Track metric trends over time:** store evaluation scores in a time-series database (e.g., CloudWatch custom metrics) to catch gradual quality drift.

#### 🎯 On the exam
- "Detect hallucinations or quality regressions after a model update" → **golden dataset** with faithfulness/accuracy metrics as a deployment gate.
- "Roll out a new prompt version safely without risking a full production outage" → **canary rollout** (small traffic slice) with automatic rollback on metric regression.
- "A/B test vs. canary test" → A/B runs two versions in parallel to compare quality; canary progressively shifts traffic to the new version after a quality gate passes — both appear on the exam.

---

## 2. Troubleshooting Common Errors

### 2.1 ThrottlingException (HTTP 429)

**Error:** `ThrottlingException` — request denied because account quotas were exceeded ([source](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html)).

**Causes:**
- Requests-per-minute (RPM) or tokens-per-minute (TPM) quota exceeded.
- Sudden traffic spike above provisioned capacity.

**Remediation (in priority order):**

| Step | Action |
|---|---|
| 1 | Implement **exponential backoff with jitter** — start with 1 s delay, double after each failure, cap at ~60 s, add random jitter to avoid retry storms |
| 2 | Analyse CloudWatch `InvocationThrottles` and `EstimatedTPMQuotaUsage` to quantify the quota gap |
| 3 | Request a **quota increase** via AWS Service Quotas (console or API) or contact your AWS account manager |
| 4 | Enable **cross-region inference** to spread load across regions |
| 5 | Move sustained high-volume workloads to **Provisioned Throughput** |
| 6 | Use **batch inference** for non-real-time workloads to avoid peak-hour quota contention |

AWS SDKs in "standard" retry mode implement exponential backoff automatically for throttling errors, but you must ensure the retry count and backoff ceiling are configured appropriately for your SLA.

#### 🎯 On the exam
- **429 / ThrottlingException** → the *first* answer is always **exponential backoff with jitter**.
- Long-term fix for sustained throttling → **quota increase** + potentially **Provisioned Throughput**.
- Do NOT confuse 429 ThrottlingException (quota limit) with 503 ServiceUnavailable (capacity constraint) — both need backoff, but 503 suggests cross-region inference or checking AWS Health Dashboard.

---

### 2.2 ValidationException (HTTP 400)

**Error:** `ValidationException` / `ValidationError` — input does not satisfy API constraints.

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Invalid model ID | Call `ListFoundationModels` to confirm the correct model ID and ensure the model is enabled in your account |
| Missing required parameter | Review the API reference; check that `modelId`, `body`, `contentType`, `accept` are present |
| Parameter value out of allowed range | Check `temperature` (0–1), `max_tokens`, `top_p` etc. against model-specific limits |
| Malformed JSON body | Validate JSON before sending; use an AWS SDK to construct the request rather than raw HTTP |
| Model not enabled | Enable the model in the Bedrock console under Model Access |

#### 🎯 On the exam
- "Getting 400 errors on a new model" → first check that the model is **enabled** in your account and that the model ID is correct.

---

### 2.3 Token Limit and Context Window Errors

**Error:** The prompt + expected output exceed the model's maximum context window, or `max_tokens` is set too high relative to remaining context.

**Symptoms:**
- `ValidationException` mentioning token count or context length.
- Response is truncated mid-sentence.
- Model returns an error before generating any output.

**Fixes:**

| Fix | When to apply |
|---|---|
| Truncate conversation history to a rolling window | Long multi-turn conversations |
| Summarise older turns into a compressed memory block | Preserves context without full history |
| Reduce `top_k` in RAG to send fewer retrieved chunks | Retrieval is consuming too much context |
| Increase `max_tokens` (if output is truncating and context allows) | Response is cut off |
| Choose a model with a larger context window | Workload genuinely requires long context |
| Split the task into smaller subtasks | Single prompt is too large structurally |

#### 🎯 On the exam
- "Model response cuts off mid-sentence" → either `max_tokens` is too low, or the context window is full — check both.
- "Context window error with RAG" → reduce `top_k`, trim chunk size, or use a model with a larger context window.

---

### 2.4 Timeouts and Latency Spikes

**Symptoms:** Requests time out at the SDK or application layer; `InvocationLatency` CloudWatch metric spikes.

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Long output generation (large `max_tokens`) | Enable streaming (`ConverseStream`) so partial tokens reach the client faster |
| Cold connection after idle period through NAT/VPC endpoint | Enable TCP keep-alive on the Bedrock client (`tcp_keepalive=True` in boto3); lower OS `net.ipv4.tcp_keepalive_time` below 350 s |
| Model under load (shared capacity) | Move to Provisioned Throughput; enable cross-region inference |
| Large context window input | Trim context; use prompt caching to offload repeated prefix processing |
| Network proximity | Deploy application in the same region as Bedrock endpoint |

> **TCP keep-alive note:** NAT Gateways and VPC endpoints drop idle TCP connections after 350 seconds. This appears as sudden timeouts on the first request after a quiet period. Enable both SDK-level and OS-level keep-alive ([source](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html)).

#### 🎯 On the exam
- "First request after idle period times out" → **TCP keep-alive** on the Bedrock client + OS kernel setting.
- "Users see long wait before anything appears in chat" → enable **streaming**.

---

### 2.5 Poor Retrieval Quality in RAG

Poor retrieval is the most common root cause of RAG quality failures. Diagnose with the context relevance and faithfulness metrics from Section 1.3.

**Diagnosis and fixes:**

| Symptom | Root cause | Fix |
|---|---|---|
| Retrieved chunks are off-topic | Embedding model too general; chunking too coarse | Switch to a domain-specific or higher-quality embedding model; reduce chunk size to capture finer semantic units |
| Retrieved chunks are relevant but answer is still wrong | Generator not using retrieved context | Improve system prompt to explicitly instruct grounding; check that chunks are formatted clearly |
| Correct information exists in KB but is not retrieved | top-k too low; similarity threshold too tight | Increase top-k; lower similarity threshold; add **hybrid search** (keyword + semantic) |
| Duplicate or near-duplicate chunks returned | Knowledge base has redundant content | De-duplicate documents; use a re-ranker (e.g., Cohere Re-Rank via Bedrock) to filter before generation |
| Retrieval is slow | top-k too large; vector index not optimised | Reduce top-k; use an approximate nearest-neighbour index (e.g., FAISS HNSW); move to Amazon OpenSearch Serverless |

#### 🎯 On the exam
- "Improve RAG answer quality" → **fix chunking and embedding first**, then tune top-k, then add a re-ranker.
- "Evaluate retrieval quality" → use **context relevance** metric in Bedrock RAG Evaluation.

---

### 2.6 Hallucinations

**Definition:** The model generates confident-sounding statements that are factually incorrect or not supported by the provided context.

**Detection:**
- **Faithfulness metric** in Bedrock RAG Evaluation (automatic LLM judge).
- Human review via Amazon A2I for high-stakes outputs.
- Prompt-level self-check: ask the model "Is every claim in your response supported by the provided context? Identify any that are not."

**Mitigation:**

| Strategy | Mechanism |
|---|---|
| Explicit grounding instruction | System prompt: "Only use information from the provided context. If the context does not contain the answer, say 'I don't know'." |
| Amazon Bedrock Guardrails — Grounding | Automatically checks that each sentence in the response is traceable to the provided source context; blocks or redacts unsupported claims |
| Retrieval quality improvement | Better chunks = less need for the model to "fill in gaps" |
| Lower temperature | Reduces creative output; model stays closer to training distribution and provided context |
| Citation requirement | Require the model to cite the specific chunk for each claim; unverifiable claims are easier to catch |
| Post-processing faithfulness check | Run a separate faithfulness LLM call to score the output before returning it to the user |

#### 🎯 On the exam
- "Reduce hallucinations in RAG" → **grounding instruction in system prompt + Bedrock Guardrails grounding check**.
- "Detect hallucinations at scale" → **faithfulness metric** in Bedrock RAG Evaluation (LLM-as-a-judge).

---

### 2.7 Empty or Malformed Tool-Use Outputs

**Symptom:** In an agentic workflow, the model returns an empty `toolUse` block, incorrect parameter values, or calls a tool that does not exist.

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Tool definition is ambiguous | Rewrite tool `description` to be explicit about required and optional parameters; provide parameter examples |
| Model chose the wrong tool | Add clearer discrimination in tool descriptions; add a routing instruction to the system prompt |
| Parameters are missing or incorrectly typed | Add JSON schema validation (`required` fields, `type` constraints) to the tool spec; validate the tool call before executing |
| Model returns tool call with empty parameters | Check that the model's context contains the information needed to populate the parameter; provide few-shot examples of correct tool calls |
| `stop_reason` is not `tool_use` | The model decided not to call a tool; check `stop_reason` in the response before assuming a tool call was made |

#### 🎯 On the exam
- "Agent keeps calling tools with wrong parameters" → improve tool `description` and add explicit JSON schema constraints.
- "Agent returns empty tool call" → ensure required information is in the context and provide a few-shot example of the correct call.

---

## 3. Resilience Patterns

### 3.1 Graceful Degradation and Fallback Models

> **Why (the rationale):** LLM APIs are shared infrastructure subject to throttling, capacity constraints, and occasional outages. Without fallback logic, any model-side issue immediately surfaces as an error to end users. Graceful degradation maintains service quality at reduced capability rather than total failure.
> **When to use:** Any customer-facing GenAI application where uptime matters. Implement a tiered fallback: primary model → smaller model → cached response → static response. Use Intelligent Prompt Routing as the automatic version of this within a model family.
> **Nuances & gotchas:** Falling back to a smaller model may produce lower-quality responses — this is intentional and acceptable for resilience, but users should be notified if possible. Circuit breakers must track error rate over a rolling window, not just the latest request, to avoid flapping between open and closed state during a sustained outage.

A production GenAI application should **never fail hard** simply because one model is unavailable or over capacity.

**Patterns:**

| Pattern | Description |
|---|---|
| **Fallback to a smaller model** | If the primary model returns a ThrottlingException or ServiceUnavailable, retry on a smaller/cheaper model (e.g., Claude Sonnet → Claude Haiku). Use Amazon Bedrock Intelligent Prompt Routing as an automatic version of this for the same model family. |
| **Fallback to a cached response** | For common queries, return a pre-cached answer rather than failing. |
| **Degrade to a rule-based response** | "I'm unable to process your request right now. Here are some helpful links…" |
| **Circuit breaker** | Track recent error rate; if it exceeds a threshold, stop sending requests to the failing endpoint temporarily and serve degraded responses until the circuit resets. |

---

### 3.2 Retry with Exponential Backoff and Jitter

This is the canonical AWS response to transient errors (ThrottlingException, InternalFailure, ServiceUnavailable):

```
delay = min(base_delay * (2 ** attempt), max_delay)
sleep(delay + random_jitter)
```

Recommended parameters ([source](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html)):
- **Base delay:** 1 second.
- **Multiplier:** 2× per attempt.
- **Max delay:** 60 seconds (for per-minute quota exhaustion, ensure total backoff covers 1 full minute).
- **Max attempts:** 6.
- **Jitter:** add `random(0, delay)` to the computed delay to prevent synchronised retry bursts.

AWS SDKs in **standard mode** implement this automatically. Verify your SDK retry mode is set to `standard` (not `legacy`):

```python
import boto3
from botocore.config import Config

config = Config(retries={"max_attempts": 6, "mode": "standard"})
client = boto3.client("bedrock-runtime", config=config)
```

#### 🎯 On the exam
- Whenever you see **ThrottlingException** or **InternalFailure** in a scenario → first fix is **exponential backoff with jitter**.
- "Retry loop seems to make throttling worse" → missing **jitter**; synchronised retries compound the burst.

---

### 3.3 Idempotency

Some Bedrock operations (e.g., creating an evaluation job, purchasing Provisioned Throughput) may be retried on failure. To avoid duplicate resource creation:

- Use **idempotency tokens** (`clientRequestToken` parameter) when creating Bedrock resources so that retrying the same logical operation does not create duplicates.
- For invocations, model inference is inherently stateless — retrying a failed `InvokeModel` call is safe (though the response may differ due to temperature).

---

### 3.4 Dead-Letter Queues

> **Why (the rationale):** In async SQS→Lambda→Bedrock pipelines, failed messages that are retried indefinitely block queue processing and cause exponential backoff delays for subsequent messages. DLQs capture poison messages after a configurable number of failures so the main queue keeps flowing and failed items can be analyzed and reprocessed separately.
> **When to use:** Any SQS-backed async GenAI processing pipeline (document ingestion, batch classification, event-driven summarization). Set `maxReceiveCount` to 3–5 before moving to DLQ; alarm on DLQ depth to detect systematic failures.
> **Nuances & gotchas:** DLQ messages are not automatically retried — you must trigger a **DLQ redrive** after fixing the root cause to replay them. DLQ redrive replays messages in the order they arrived in the DLQ, not the original processing order. Lambda function timeouts count as failures and increment the `ApproximateReceiveCount` — a 15-minute Bedrock call that times out Lambda will exhaust retries quickly.

For asynchronous GenAI workloads (e.g., batch document processing via SQS → Lambda → Bedrock):

- Configure an **Amazon SQS Dead-Letter Queue (DLQ)** on the source queue.
- After a configurable number of failed processing attempts (e.g., 3), SQS moves the message to the DLQ.
- Alerts on DLQ depth (CloudWatch alarm on `ApproximateNumberOfMessagesVisible`) notify the team.
- Inspect DLQ messages to diagnose systematic failures (e.g., all messages with a specific document type fail).
- After fixing the root cause, replay DLQ messages back to the source queue using the SQS DLQ redrive feature.

#### 🎯 On the exam
- "Async Lambda → Bedrock processing, some jobs keep failing — don't lose the messages" → **SQS DLQ**.

---

### 3.5 Error Handling Architecture

A complete resilience architecture for a Bedrock-based application:

```
Client
  │
  ▼
API Gateway ──► Lambda (orchestrator)
                  │
                  ├─ Retry with exponential backoff + jitter
                  ├─ Circuit breaker (track error rate)
                  ├─ Primary model: Claude Sonnet
                  │    └─ ThrottlingException / 503
                  │         └─► Fallback model: Claude Haiku
                  │               └─► Static cached response
                  │
                  ├─ SQS queue (async tasks)
                  │    └─ DLQ on failure
                  │
                  └─ CloudWatch metrics (custom: fallback_rate, error_rate)
```

Key principles:
1. **Never propagate raw AWS errors to the user** — catch and translate.
2. **Log every error with context** (model ID, input token count, request ID) to CloudWatch Logs / invocation logging.
3. **Alert on error rate thresholds**, not just individual errors.
4. **Test failure modes in staging** — deliberately inject throttling errors to verify retry and fallback logic.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| BLEU | N-gram overlap metric between generated and reference text | Fast automated quality check, best for translation |
| ROUGE | Recall-focused n-gram overlap; ROUGE-L uses longest common subsequence | Summarisation quality measurement |
| BERTScore | Semantic similarity using BERT embeddings; no exact n-gram matching required | Better for paraphrased but correct outputs |
| Perplexity | How surprised a language model is by a text; lower = more fluent | Measures fluency, not factual accuracy |
| LLM-as-a-judge | Using a second LLM to score the outputs of the model under evaluation | Fast, cheap, human-like quality evaluation |
| Faithfulness / Groundedness | Whether the generated response is supported by retrieved context | Core RAG hallucination metric |
| Context Relevance | Whether retrieved chunks actually match the user query | Retrieval quality metric |
| Answer Relevance / Correctness | Whether the final answer addresses the question accurately | End-to-end RAG quality metric |
| Amazon A2I | AWS service for routing ML outputs to human reviewers | Human-in-the-loop validation in production |
| SageMaker Ground Truth | Managed labelling service with active learning | Building ground-truth datasets for evaluation and fine-tuning |
| ThrottlingException | HTTP 429 — request denied because account quota was exceeded | Requires exponential backoff + jitter; may need quota increase |
| ValidationException | HTTP 400 — malformed or constraint-violating request | Fix request parameters (model ID, JSON structure, value ranges) |
| ServiceUnavailable | HTTP 503 — temporary service capacity constraint | Retry with backoff; enable cross-region inference |
| Exponential backoff with jitter | Retry strategy: delay doubles each attempt + random noise added | Avoids retry storms; standard AWS resilience pattern |
| Dead-Letter Queue (DLQ) | SQS queue that receives messages that fail processing repeatedly | Prevents message loss in async pipelines; enables root-cause analysis |
| Circuit breaker | Stops sending requests to a failing endpoint after repeated errors | Prevents cascade failures; allows partial recovery |
| Idempotency token | A unique identifier on a request so retrying does not create duplicate resources | Safe retries on resource-creation operations |
| Regression test suite (golden prompts) | Fixed set of prompts with expected outputs used to catch quality regressions | CI/CD gate for model and prompt changes |
| Golden dataset | Curated prompts with known-good expected outputs covering representative use cases | Detect hallucinations, semantic drift, and regressions across model/prompt updates |
| Canary rollout | Deploying a new model/prompt version to a small traffic slice before full rollout | Limits blast radius of quality regressions; enables automatic rollback |
| Agent evaluation | Bedrock evaluation mode measuring task completion rate, tool-usage effectiveness, and reasoning quality across multi-step workflows | Evaluate agents (not just single-turn responses) before deployment |
| Chunking | Splitting documents into smaller pieces for vector indexing in RAG | Affects retrieval precision and context window efficiency |
| Re-ranker | A second model that reorders retrieved chunks by relevance before generation | Improves context relevance without changing the embedding model |

---

## References

- [Evaluate the Performance of Amazon Bedrock Resources (incl. Agent Evaluations)](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)
- [Evaluate Model Performance Using Another LLM as a Judge](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-judge.html)
- [LLM-as-a-Judge on Amazon Bedrock Model Evaluation (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/llm-as-a-judge-on-amazon-bedrock-model-evaluation/)
- [Amazon Bedrock Model Evaluation LLM-as-a-Judge — GA Announcement](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-model-evaluation-llm-as-a-judge/)
- [Amazon Bedrock RAG Evaluation — GA Announcement](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-rag-evaluation-generally-available/)
- [Evaluating RAG Applications with Amazon Bedrock Knowledge Base Evaluation (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/evaluating-rag-applications-with-amazon-bedrock-knowledge-base-evaluation/)
- [Review Metrics for RAG Evaluations That Use LLMs (Console)](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-eval-llm-results.html)
- [Troubleshooting Amazon Bedrock API Error Codes](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html)
- [Quotas for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas.html)
- [Scaling and Throughput Best Practices — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/scaling-throughput-best-practices.html)
- [Retry Behavior — AWS SDKs and Tools Reference Guide](https://docs.aws.amazon.com/sdkref/latest/guide/feature-retry-behavior.html)
- [Retries with Exponential Backoff and Jitter — AWS Builders Library](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Optimize Applications for Scale and Reliability on Amazon Bedrock (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/optimize-your-applications-for-scale-and-reliability-on-amazon-bedrock/)
- [Implementing Resilience Patterns with Amazon Bedrock and LLM Gateway (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/)
- [New RAG Evaluation and LLM-as-a-Judge Capabilities in Amazon Bedrock (AWS Blog)](https://aws.amazon.com/blogs/aws/new-rag-evaluation-and-llm-as-a-judge-capabilities-in-amazon-bedrock/)
- [Amazon Augmented AI (A2I)](https://docs.aws.amazon.com/sagemaker/latest/dg/a2i-use-augmented-ai-a2i-human-review-loops.html)
- [Amazon SageMaker Ground Truth](https://docs.aws.amazon.com/sagemaker/latest/dg/sms.html)
- [Bedrock service — ../services/bedrock.md](../services/bedrock.md)

---

*Part of the [AWS AI Certification course](../README.md) · AIP-C01 track.*
