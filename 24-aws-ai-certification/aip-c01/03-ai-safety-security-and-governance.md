# Domain 3: AI Safety, Security, and Governance

> **AIP-C01 Weight: 20% · ~13 scored questions out of 65**

AI Safety, Security, and Governance is the control layer that sits around every model invocation, data pipeline, and deployed agent. The exam expects you to know *which AWS service or Bedrock feature solves which safety or compliance problem*, and to choose the right tool for scenario-based questions about harmful content, PII leakage, hallucinated outputs, unauthorized access, and audit trails.

> **Plain English:** Think of this domain as the bouncer, the vault, and the auditor working together. **Amazon Bedrock Guardrails** is the bouncer — it decides what goes in and what comes out. **KMS, IAM, and PrivateLink** are the vault — they lock down who can touch the model and where data travels. **CloudTrail and model-invocation logging** are the auditor — they write down everything that happened. Responsible AI principles and the Generative AI Security Scoping Matrix are the rulebook the whole team follows.

---

## Table of Contents

1. [Amazon Bedrock Guardrails — Full Reference](#1-amazon-bedrock-guardrails--full-reference)
   - [Content Filters](#11-content-filters)
   - [Denied Topics](#12-denied-topics)
   - [Word Filters](#13-word-filters)
   - [Sensitive Information Filters — PII Detection & Redaction](#14-sensitive-information-filters--pii-detection--redaction)
   - [Contextual Grounding & Relevance Checks](#15-contextual-grounding--relevance-checks)
   - [Automated Reasoning Checks](#16-automated-reasoning-checks)
   - [The ApplyGuardrail API](#17-the-applyguardrail-api)
   - [Applying One Guardrail Across Models, Agents & Knowledge Bases](#18-applying-one-guardrail-across-models-agents--knowledge-bases)
2. [Identity & Access Management](#2-identity--access-management)
   - [IAM Policies for Bedrock](#21-iam-policies-for-bedrock)
   - [VPC Endpoints / AWS PrivateLink](#22-vpc-endpoints--aws-privatelink)
   - [Cross-Account Patterns](#23-cross-account-patterns)
3. [Data Protection](#3-data-protection)
   - [Encryption at Rest and in Transit with AWS KMS](#31-encryption-at-rest-and-in-transit-with-aws-kms)
   - [Bedrock Data-Privacy Guarantees](#32-bedrock-data-privacy-guarantees)
   - [Amazon Macie for PII Discovery in S3](#33-amazon-macie-for-pii-discovery-in-s3)
   - [Amazon Comprehend PII Detection & Toxicity](#34-amazon-comprehend-pii-detection--toxicity)
4. [Responsible AI & Governance](#4-responsible-ai--governance)
   - [Bias Mitigation, Fairness & Transparency](#41-bias-mitigation-fairness--transparency)
   - [AWS AI Service Cards](#42-aws-ai-service-cards)
   - [SageMaker & Bedrock Model Evaluation for Safety](#43-sagemaker--bedrock-model-evaluation-for-safety)
   - [Watermarking of Generated Content](#44-watermarking-of-generated-content)
   - [Generative AI Security Scoping Matrix](#45-generative-ai-security-scoping-matrix)
   - [Audit & Logging — CloudTrail and Model-Invocation Logging](#46-audit--logging--cloudtrail-and-model-invocation-logging)
   - [Compliance Frameworks](#47-compliance-frameworks)
5. [Prompt-Injection & Jailbreak Defense](#5-prompt-injection--jailbreak-defense)
6. [Glossary](#6-glossary)
7. [References](#7-references)

---

## 1. Amazon Bedrock Guardrails — Full Reference

[Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html) is the primary control plane for safe inference in AWS. A single guardrail is a versioned resource that bundles one or more policies; it is applied at the API call level and can wrap **any foundation model, Bedrock agent, or Knowledge Base** with the same controls.

Guardrails are offered in two **tiers**: *Basic* (default, lower cost) and **Standard** (adds code-domain scanning, standalone prompt-attack detection, and extended denied-topic support). All blocked content appears as plain text in [model invocation logs](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) — disable logging if that is a compliance concern.

---

### 1.1 Content Filters

> **Why (the rationale):** Without content filters, models can produce hate speech, explicit content, or instructions for illegal activities in response to adversarial or accidental prompts. Content filters intercept both inputs (blocking jailbreak attempts) and outputs (catching model-generated harmful content) in one policy.
> **When to use:** Any customer-facing application. Strength level should match risk tolerance: consumer apps targeting general audiences → HIGH for Hate/Sexual/Violence; internal developer tools → MEDIUM may suffice. Always enable Prompt Attack at Standard tier for applications exposed to untrusted user input.
> **Nuances & gotchas:** Content filters have four strength levels (`NONE`, `LOW`, `MEDIUM`, `HIGH`) — they are NOT a simple on/off toggle. Setting all to `HIGH` increases false positives and may block legitimate content. The Prompt Attack category is inside Content Filters at Basic tier, but at **Standard tier** it also produces a standalone `PROMPT_ATTACK` finding type — the exam may distinguish these two.

**What it does:** Detects and filters harmful text or image content in prompts **and** model responses across six predefined harm categories.

| Category | Description |
|---|---|
| **Hate** | Content that demeans a person or group based on identity attributes |
| **Insults** | Bullying, demeaning, or offensive language targeting individuals |
| **Sexual** | Explicit or suggestive sexual content |
| **Violence** | Content promoting or glorifying physical harm |
| **Misconduct** | Encouragement of illegal activity or harmful behavior |
| **Prompt Attack** | Jailbreaks, prompt injections, and prompt leakage attempts |

**Configurable strength levels:** `NONE`, `LOW`, `MEDIUM`, `HIGH` — set independently for inputs and outputs per category. Higher strength = stricter filtering, but may increase false positives.

With **Standard tier**, filtering extends into **code domains** — comments, variable names, function names, and string literals are also scanned. Prompt Attack detection is available as a **standalone check** at Standard tier, returning `PROMPT_ATTACK` as an independent finding.

Source: [Amazon Bedrock Guardrails components](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html), [Guardrails tiers announcement](https://aws.amazon.com/about-aws/whats-new/2025/06/amazon-bedrock-guardrails-tiers-content-filters-denied-topics)

#### 🎯 On the exam

- **Scenario:** "Block hate speech and sexually explicit output from a customer chatbot." → **Content filters** with appropriate strength levels.
- **Trap:** Content filters require you to choose a strength — they do not have a simple on/off toggle. Setting all categories to `HIGH` does not guarantee zero harmful output; it raises the blocking threshold.
- **Trap:** Prompt Attack is inside Content Filters (as a category), but at Standard tier it is also available as a standalone check — the exam may distinguish these.

---

### 1.2 Denied Topics

> **Why (the rationale):** Content filters cover universally harmful categories; denied topics cover domain-specific off-limits subjects that are benign in general but inappropriate for a specific application (competitor products, investment advice, political opinions). They solve the "keep the bot on topic" problem.
> **When to use:** When you need to block an application-specific domain that content filters don't cover — e.g., a cooking bot that must not discuss weight loss drugs, or a company bot that must not discuss competitors. Use natural-language definitions; provide example phrases to reduce false negatives.
> **Nuances & gotchas:** Denied topics use **semantic understanding** (ML), not keyword matching — a user asking about "the other AI company" might still be caught if the definition covers competitors semantically. For exact string blocking (brand name, specific word), use Word Filters instead. At Standard tier, denied-topic detection extends into code domain content (comments, strings).

**What it does:** Lets you define custom topics the application must never discuss — described in natural language, not code. A banking bot can be configured to refuse illegal investment advice; a children's platform can refuse any adult topic.

With **Standard tier**, denied-topic detection also extends into code domains.

You configure a topic with a short natural-language **definition** and, optionally, **example phrases** to tune detection accuracy. Guardrails evaluates whether user input or model output relates to any denied topic and returns `GUARDRAIL_INTERVENED` if so.

Source: [Amazon Bedrock Guardrails components](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html)

#### 🎯 On the exam

- **Reflex:** "Prevent a healthcare assistant from discussing competitor products." → **Denied topics** (content filters don't cover arbitrary off-topic areas; denied topics do).
- **Trap:** Denied topics use semantic understanding, not keyword matching — use word filters for exact string blocking.

---

### 1.3 Word Filters

> **Why (the rationale):** For specific, known strings (competitor brand names, internal code words, known slurs) where ML-based semantic detection is overkill or too slow, deterministic string matching provides guaranteed, zero-latency blocking with no false negatives for exact matches.
> **When to use:** Blocking exact brand names, internal project code names, regulatory-forbidden terms that must never appear verbatim. Combine with denied topics for broader semantic coverage around the same subject.
> **Nuances & gotchas:** Word filters are **case-sensitive exact matches** — "CompetitorX" won't catch "competitorx" or "Competitor X". Misspellings and synonyms bypass word filters entirely — content filters or denied topics are needed for semantic coverage. The built-in profanity list is AWS-curated and cannot be inspected or customized; use custom lists for precise control.

**What it does:** Exact-string blocking of specific words or phrases. You can enable a built-in **profanity list** or supply a custom list (competitor names, internal code names, slurs, etc.).

Word filters are deterministic — if the exact string appears, it is blocked. No ML inference involved.

Source: [Amazon Bedrock Guardrails components](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html)

#### 🎯 On the exam

- **Reflex:** "Block the word 'CompetitorX' from appearing in responses." → **Word filters**, not denied topics.
- **Trap:** Word filters are case-sensitive exact matches. Misspellings or synonyms won't be caught — combine with content filters or denied topics for broader coverage.

---

### 1.4 Sensitive Information Filters — PII Detection & Redaction

> **Why (the rationale):** Users routinely include SSNs, credit card numbers, and other PII in chat messages, and models sometimes reproduce PII from training data. Sensitive information filters catch these at the inference boundary — acting as a last-resort safety net even if upstream redaction (Comprehend) or data classification (Macie) missed them.
> **When to use:** Any application where PII leakage is a compliance risk (financial services, healthcare, HR). Use ANONYMIZE mode when the downstream prompt can still be useful with placeholders (e.g., "Dear {NAME}, …"). Use BLOCK when even a masked version is unacceptable (e.g., block any message containing a credit card number entirely).
> **Nuances & gotchas:** PII masking applies only to content sent to/from the model — it does **NOT** apply to model invocation logs. Raw PII still appears unmasked in CloudWatch Logs even with masking enabled; configure **CloudWatch log data protection** separately. `AWS_ACCESS_KEY` and `AWS_SECRET_KEY` are built-in PII types (no custom regex needed). Custom regex patterns cannot use lookahead/lookbehind assertions. Detection is context-dependent — a bare 9-digit string without surrounding context may not be classified as an SSN.

**What it does:** Uses probabilistic ML to detect sensitive data (PII and custom patterns) in prompts and responses, then either **blocks** all content containing it or **masks/anonymizes** it by replacing detected values with their type placeholder (e.g., `{NAME}`, `{EMAIL}`).

This filter is **context-dependent** — a short string may not be recognized without surrounding context.

**Built-in PII categories (selected):**

| Group | Entity types |
|---|---|
| General | ADDRESS, AGE, NAME, EMAIL, PHONE, USERNAME, PASSWORD, DRIVER\_ID, LICENSE\_PLATE, VEHICLE\_IDENTIFICATION\_NUMBER |
| Finance | CREDIT\_DEBIT\_CARD\_NUMBER, CREDIT\_DEBIT\_CARD\_CVV, CREDIT\_DEBIT\_CARD\_EXPIRY, PIN, INTERNATIONAL\_BANK\_ACCOUNT\_NUMBER, SWIFT\_CODE |
| IT | IP\_ADDRESS, MAC\_ADDRESS, URL, **AWS\_ACCESS\_KEY**, **AWS\_SECRET\_KEY** |
| USA | US\_SOCIAL\_SECURITY\_NUMBER, US\_PASSPORT\_NUMBER, US\_BANK\_ACCOUNT\_NUMBER, US\_BANK\_ROUTING\_NUMBER, US\_INDIVIDUAL\_TAX\_IDENTIFICATION\_NUMBER |
| Canada | CA\_HEALTH\_NUMBER, CA\_SOCIAL\_INSURANCE\_NUMBER |
| UK | UK\_NATIONAL\_HEALTH\_SERVICE\_NUMBER, UK\_NATIONAL\_INSURANCE\_NUMBER, UK\_UNIQUE\_TAXPAYER\_REFERENCE\_NUMBER |

**Custom regex:** For any pattern not in the built-in list (serial numbers, booking IDs, employee IDs), supply a named regex pattern (1–500 characters, no lookaround assertions). Action options: `BLOCK`, `ANONYMIZE`, or `NONE` (detect-only, return detection info for application logic).

**Handling modes:**
- **BLOCK** — Reject the entire input or output when sensitive data is detected.
- **ANONYMIZE / MASK** — Pass through the content but replace detected values with `{TYPE}` placeholders. Useful for call-center transcript summarization where the summary should not contain the customer's SSN.

**Important caveat:** PII masking applies only to content sent to/from the model — it does **not** apply to model invocation logs. The raw input always appears unmasked in CloudWatch logs. Use [CloudWatch log data protection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html) to mask PII in logs separately.

Source: [Remove PII from conversations by using sensitive information filters](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-sensitive-filters.html)

#### 🎯 On the exam

- **Reflex:** "Prevent SSNs and credit card numbers from appearing in chatbot responses." → **Sensitive information filters** with ANONYMIZE or BLOCK action.
- **Reflex:** "Detect employee badge numbers (format: EMP-#####) in outputs." → **Custom regex filter** inside sensitive information filters.
- **Trap:** AWS\_ACCESS\_KEY and AWS\_SECRET\_KEY are built-in PII types — you don't need a custom regex to detect accidental credential exposure.
- **Trap:** PII masking does not protect model invocation logs — you need CloudWatch log data protection for that.

---

### 1.5 Contextual Grounding & Relevance Checks

> **Why (the rationale):** Content filters catch harmful language; they do nothing about factually wrong but politely phrased answers. Contextual grounding fills this gap by verifying that the model's output is actually supported by the source material you provided — the only Guardrails feature specifically targeting hallucinations.
> **When to use:** RAG pipelines, document summarization, and Q&A where factual accuracy against a known source is critical (legal, medical, financial). Requires that you supply the grounding source text alongside the query and response.
> **Nuances & gotchas:** Contextual grounding **cannot fact-check against the internet or the model's parametric knowledge** — it only validates against the text you supply as the grounding source. It is a separate policy from content filters; enabling content filters does NOT enable grounding checks. Priced at $0.10 per 1,000 text units (source + query + response combined) — can add meaningful cost at scale. Supported use cases are summarization, paraphrasing, and Q&A; it is NOT designed for code generation or creative writing verification.

**What it does:** Detects and filters **hallucinations** in model responses for RAG, summarization, and Q&A workloads. The check compares the model's response against a **grounding source** (the retrieved passages) and the **user query** to produce two scores:

| Score | What it measures | Threshold range |
|---|---|---|
| **Grounding score** | Is the response factually consistent with the source? | 0.0–1.0 (set threshold; responses below are blocked or flagged) |
| **Relevance score** | Does the response actually answer the user's query? | 0.0–1.0 (same mechanism) |

You configure a **threshold** for each score. Responses that fall below either threshold are blocked or flagged. Contextual grounding filters **over 75% of hallucinated responses** in RAG and summarization tasks ([AWS Bedrock security page](https://aws.amazon.com/bedrock/security-privacy-responsible-ai)).

**Required inputs:** (1) grounding source text, (2) user query, (3) model response. These can be supplied via the `Converse` API, `InvokeModel`, or the `ApplyGuardrail` API directly.

**Supported use cases:** Summarization, paraphrasing, question answering.

**Pricing note:** Contextual grounding is priced at $0.10 per 1,000 text units (source + query + response combined).

Source: [Use contextual grounding check to filter hallucinations](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html)

#### 🎯 On the exam

- **Reflex:** "Verify a RAG response is factually grounded in the retrieved documents before returning it to the user." → **Contextual grounding check** (grounding score threshold).
- **Reflex:** "Ensure the model's answer actually addresses the user's question." → **Relevance score threshold** in contextual grounding.
- **Trap:** Contextual grounding requires a reference source — it cannot fact-check against the model's internal knowledge or the internet. It validates only against the text you supply.
- **Trap:** Contextual grounding is a separate policy from content filters — enabling content filters alone does NOT catch hallucinations.

---

### 1.6 Automated Reasoning Checks

> **Why (the rationale):** Grounding checks test whether a response aligns with a source document; Automated Reasoning goes further and **formally proves** compliance with business rules using mathematical logic — not sampling or probabilistic scoring. This is the only Guardrails feature that can provide a proof of compliance, suitable for regulated financial, insurance, and healthcare calculations.
> **When to use:** When the business rules can be expressed formally and you need mathematically provable correctness (e.g., insurance premium calculations must follow approved rate tables, financial advice must comply with GAAP rules). Not appropriate for open-ended creative or conversational outputs.
> **Nuances & gotchas:** Automated Reasoning is fundamentally different from contextual grounding — grounding checks factual alignment with a source document; Automated Reasoning checks logical/mathematical policy compliance. Policies are written in natural language and Bedrock translates them to formal logic internally — ambiguous natural-language policies reduce verification accuracy. This feature achieves up to 99% verification accuracy per AWS, but this metric is task-dependent.

**What it does:** Policy-based **mathematical and logical verification** of model responses. You write policies in natural language that express business or regulatory rules (e.g., "Only recommend medications that are listed in the approved formulary," "Financial calculations must follow GAAP rounding rules"). The automated reasoning engine evaluates whether the model's output **provably complies** with those rules — no sampling, no probabilistic scoring.

This is fundamentally different from content filters (which block harmful text) and grounding checks (which compare to a source). Automated reasoning checks use formal verification to **prove or disprove** compliance claims, achieving up to **99% verification accuracy** ([AWS Bedrock security page](https://aws.amazon.com/bedrock/security-privacy-responsible-ai)).

Policies are uploaded as natural-language documents; Bedrock translates them into formal logic internally.

Source: [Automated reasoning checks announcement](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-guardrails-policy-based-enforcement-responsible-ai/), [Guardrails components](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html)

#### 🎯 On the exam

- **Reflex:** "A compliance officer wants to mathematically prove that a chatbot's financial advice adheres to regulatory constraints." → **Automated Reasoning checks** (not content filters, not grounding).
- **Reflex:** "Ensure an insurance chatbot only quotes premiums consistent with approved rate tables." → **Automated Reasoning checks**.
- **Trap:** Automated Reasoning is not the same as contextual grounding. Grounding checks factual alignment with a source document. Automated Reasoning checks logical/mathematical policy compliance.

---

### 1.7 The ApplyGuardrail API

> **Why (the rationale):** The inline guardrail mechanism (passing `guardrailConfig` in `InvokeModel`/`Converse`) requires a Bedrock-hosted model. ApplyGuardrail breaks this coupling, enabling Bedrock safety controls to wrap any model — including open-source models on EC2, third-party API models, or self-hosted LLMs.
> **When to use:** Applying Bedrock Guardrails to non-Bedrock models; pre-screening user input before sending to any model; batch compliance evaluation of existing text corpora; applying guardrails to streaming outputs token by token.
> **Nuances & gotchas:** ApplyGuardrail calls the guardrail evaluation service independently — you are billed for the guardrail check separately from any model invocation. There is no automatic content blocked response sent to a user; your code must handle the guardrail verdict (`GUARDRAIL_INTERVENED`) and take action. Only one `source` type per call: either `INPUT` or `OUTPUT`.

The [`ApplyGuardrail`](https://aws.amazon.com/blogs/machine-learning/use-the-applyguardrail-api-with-long-context-inputs-and-streaming-outputs-in-amazon-bedrock/) API **decouples guardrail evaluation from model invocation**. You send text directly to the API and receive a guardrail verdict — no foundation model is called.

**Key use cases:**
- **Pre-screening user input** before sending it to any model (including third-party or self-hosted models).
- **Post-processing model output** from non-Bedrock models (open-source, on-prem, or third-party API).
- **Batch evaluation** of large corpora of text for compliance.
- **Long-context inputs and streaming outputs** — the API supports applying guardrail checks to streamed token sequences as they arrive.

```
POST /guardrails/{guardrailIdentifier}/versions/{guardrailVersion}/apply
{
  "source": "INPUT" | "OUTPUT",
  "content": [{"text": {"text": "user message here"}}]
}
```

Source: [ApplyGuardrail API blog](https://aws.amazon.com/blogs/machine-learning/use-the-applyguardrail-api-with-long-context-inputs-and-streaming-outputs-in-amazon-bedrock/), [New API targeting agentic workflows](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-guardrails-api-ai/)

#### 🎯 On the exam

- **Reflex:** "Apply Bedrock Guardrails to an open-source LLM running on EC2." → **ApplyGuardrail API** (no native Bedrock model needed).
- **Reflex:** "Screen all user messages before any model is called." → **ApplyGuardrail API** on `source: INPUT`.
- **Trap:** The regular `InvokeModel` / `Converse` APIs run guardrails inline but require a Bedrock-hosted model. `ApplyGuardrail` is the path to use guardrails independently.

---

### 1.8 Applying One Guardrail Across Models, Agents & Knowledge Bases

A single guardrail resource (identified by `guardrailId` + version) can be referenced by:

- **Direct model calls** (`InvokeModel`, `Converse`) — pass `guardrailConfig` with the ID and version.
- **Bedrock Agents** — attach the guardrail to the agent definition; it applies on every turn.
- **Knowledge Bases** — attach to the Knowledge Base so responses from RAG retrieval are also evaluated.
- **Third-party or custom models** — use the `ApplyGuardrail` API (see §1.7).

This "define once, enforce everywhere" architecture is the recommended approach for consistent safety policies across a multi-model application.

---

## 2. Identity & Access Management

### 2.1 IAM Policies for Bedrock

> **Why (the rationale):** Bedrock models are high-value API endpoints — without least-privilege IAM policies, any compromised AWS principal in your account could invoke expensive frontier models or exfiltrate data. IAM is the first line of access control before data even reaches a model.
> **When to use:** Grant `bedrock:InvokeModel` scoped to specific model ARNs (not `*`) for every Lambda, ECS task, or service that calls Bedrock. Use `bedrock:ModelId` condition keys to enforce approved-model-only policies at scale. Use SCPs to enforce organization-wide restrictions.
> **Nuances & gotchas:** **`Converse` is authorized by `bedrock:InvokeModel`** — there is no separate `bedrock:Converse` IAM action. `ConverseStream` is authorized by `bedrock:InvokeModelWithResponseStream`. Bedrock does NOT support resource-based policies on foundation models; access is controlled entirely via identity-based policies and VPC endpoint policies. SCPs apply to principals within your AWS organization but NOT to cross-account external principals accessing via a shared VPC endpoint.

Amazon Bedrock follows standard AWS IAM. The key permission actions to know:

| Action | Purpose |
|---|---|
| `bedrock:InvokeModel` | Call a foundation model synchronously — **also authorizes the `Converse` API** |
| `bedrock:InvokeModelWithResponseStream` | Streaming inference — **also authorizes `ConverseStream`** |
| `bedrock:GetFoundationModel` | Describe a model |
| `bedrock:ListFoundationModels` | List available models |
| `bedrock:CreateGuardrail` / `bedrock:ApplyGuardrail` | Guardrail management and evaluation |
| `bedrock:InvokeAgent` | Call a Bedrock Agent |
| `bedrock:Retrieve` | Query a Knowledge Base |

**Least privilege example:** A Lambda function that only needs to invoke Claude should be granted only `bedrock:InvokeModel` scoped to the specific model ARN:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-6-v1:0"
}
```

**Condition keys for fine-grained control:**
- `bedrock:ModelId` — restrict which models a principal can invoke.
- Use IAM condition `StringEquals` on `bedrock:ModelId` to deny unapproved models without modifying every policy.

**Service Control Policies (SCPs):** Apply at the organization level to enforce that no account in an OU can use unauthorized models or disable guardrails.

Source: [Implementing least privilege access for Amazon Bedrock](https://aws.amazon.com/blogs/security/implementing-least-privilege-access-for-amazon-bedrock/)

#### 🎯 On the exam

- **Reflex:** "Allow only specific teams to invoke specific models." → **IAM policies with `bedrock:ModelId` condition key**.
- **Reflex:** "Prevent all accounts in an OU from using unapproved models." → **SCP with Deny on `bedrock:InvokeModel` for disallowed model ARNs**.
- **Trap:** Bedrock does not support resource-based policies on foundation models themselves — access is controlled entirely via IAM identity-based policies and endpoint policies.

---

### 2.2 VPC Endpoints / AWS PrivateLink

> **Why (the rationale):** Regulated workloads (financial, healthcare, government) often prohibit data from traversing the public internet. VPC endpoints keep all Bedrock traffic on the private AWS backbone — eliminating the need for NAT Gateways, internet gateways, or VPN tunnels for Bedrock calls from private subnets.
> **When to use:** Any workload requiring private network connectivity (HIPAA, FedRAMP, financial regulations); EC2 or Lambda in a private subnet with no NAT; zero-trust architectures where public internet exposure must be minimized.
> **Nuances & gotchas:** You need **separate VPC interface endpoints** for each Bedrock API family — `bedrock` (control plane), `bedrock-runtime` (InvokeModel/Converse), `bedrock-agent-runtime` (agents), and FIPS variants are all different endpoints; one endpoint does NOT cover all API families. FIPS endpoints are only available in select regions (us-east-1, us-east-2, us-west-2, ca-central-1, us-gov regions). Enable "Private DNS" on the endpoint so standard AWS SDK DNS resolution works without code changes.

By default, Bedrock API calls traverse the public internet. To keep inference traffic **entirely within the AWS network**, create [VPC interface endpoints powered by AWS PrivateLink](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html).

**Available endpoint service names:**

| Endpoint suffix | Covers |
|---|---|
| `com.amazonaws.{region}.bedrock` | Control Plane API (model management, guardrail CRUD) |
| `com.amazonaws.{region}.bedrock-runtime` | Runtime APIs (InvokeModel, Converse) |
| `com.amazonaws.{region}.bedrock-mantle` | Responses API (newer endpoint) |
| `com.amazonaws.{region}.bedrock-agent` | Agents Build-time API |
| `com.amazonaws.{region}.bedrock-agent-runtime` | Agents Runtime API |
| `com.amazonaws.{region}.bedrock-fips` | FIPS-compliant control plane |
| `com.amazonaws.{region}.bedrock-runtime-fips` | FIPS-compliant runtime |

**Private DNS:** Enable "Private DNS Name" on the endpoint so that standard Bedrock DNS names (e.g., `bedrock-runtime.us-east-1.amazonaws.com`) automatically resolve to the private endpoint — no code changes required.

**Endpoint policies:** Attach a resource-based IAM policy to the VPC endpoint to restrict which actions and which principals can flow through it, even if the calling IAM identity has broader permissions.

**SCP vs. endpoint policy:** SCPs apply to principals within your AWS organization. If a cross-account principal calls through your endpoint, only the **endpoint policy** controls access (the SCP doesn't apply to external principals).

Source: [VPC interface endpoints for Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html), [PrivateLink blog](https://aws.amazon.com/blogs/machine-learning/use-aws-privatelink-to-set-up-private-access-to-amazon-bedrock)

#### 🎯 On the exam

- **Reflex:** "Keep all Bedrock inference traffic off the public internet." → **VPC interface endpoint (`bedrock-runtime`)** with private DNS enabled.
- **Reflex:** "An EC2 instance in a private subnet with no NAT must call Bedrock." → **VPC endpoint for bedrock-runtime** (no internet gateway or NAT needed).
- **Trap:** You need a **separate endpoint** for the runtime vs. the control plane vs. agents runtime — one endpoint does not cover all Bedrock API families.
- **Trap:** FIPS endpoints are only available in certain regions (us-east-1, us-east-2, us-west-2, ca-central-1, us-gov-east-1, us-gov-west-1).

---

### 2.3 Cross-Account Patterns

Common architecture: a **central AI account** hosts foundation model invocations, guardrails, and Knowledge Bases; **consumer accounts** assume cross-account IAM roles to call it.

**Pattern:**
1. Create an IAM role in the central account with `bedrock:InvokeModel` permissions.
2. Grant `sts:AssumeRole` to the consumer account's principal.
3. (Optional) Use **AWS Resource Access Manager (RAM)** to share the VPC endpoint service across accounts.
4. Apply **endpoint policies** in the central account's VPC to restrict which actions are allowed through the shared endpoint.

**GuardDuty Bedrock protection** can monitor cross-account model invocations for anomalous behavior.

---

## 3. Data Protection

### 3.1 Encryption at Rest and in Transit with AWS KMS

**In transit:** All Bedrock API calls use TLS 1.2+. PrivateLink endpoints keep traffic on the AWS backbone without public-internet exposure.

**At rest:** Bedrock encrypts stored resources (knowledge bases, model customization jobs, guardrail definitions, invocation logs) using:
- **AWS-managed keys** (default, no configuration required)
- **Customer-managed keys (CMK)** via AWS KMS — you control key rotation, access policies, and deletion

**Resources that support CMK encryption:**
- Bedrock Knowledge Base vector stores
- Model customization job outputs (fine-tuning artifacts)
- Guardrail definitions
- Model invocation log delivery (S3 or CloudWatch) — requires granting `kms:GenerateDataKey` to `bedrock.amazonaws.com` with source-account and source-ARN conditions

**KMS key policy excerpt for S3 invocation log encryption:**
```json
{
  "Effect": "Allow",
  "Principal": {"Service": "bedrock.amazonaws.com"},
  "Action": "kms:GenerateDataKey",
  "Resource": "*",
  "Condition": {
    "StringEquals": {"aws:SourceAccount": "{{accountId}}"},
    "ArnLike": {"aws:SourceArn": "arn:aws:bedrock:{{region}}:{{accountId}}:*"}
  }
}
```

Source: [Data encryption in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-encryption.html), [Bedrock compliance](https://aws.amazon.com/bedrock/security-compliance)

#### 🎯 On the exam

- **Reflex:** "A regulated customer wants to control key rotation for all Bedrock-stored data." → **Customer-managed KMS key (CMK)**.
- **Trap:** Default AWS-managed encryption is active automatically — you only need to configure KMS CMK if you need customer control over keys.

---

### 3.2 Bedrock Data-Privacy Guarantees

These are **contractual and architectural commitments** that the exam tests as facts:

| Guarantee | Detail |
|---|---|
| **No training on customer data** | Inputs and outputs are never used to train Amazon Nova, Amazon Titan, or any third-party model available in Bedrock |
| **No sharing with model providers** | AWS does not share your prompts or completions with Anthropic, Meta, Mistral, or other third-party providers |
| **Data residency** | Customer content stays encrypted within the AWS Region where Bedrock is used |
| **Isolated fine-tuning** | When you fine-tune a model, AWS creates an isolated copy for your exclusive use — it is not shared with other customers |
| **IP indemnity** | AWS provides uncapped intellectual property indemnity for copyright claims from generative output when Bedrock is used responsibly |

Source: [Bedrock security, privacy & responsible AI](https://aws.amazon.com/bedrock/security-privacy-responsible-ai), [Amazon model privacy page](https://aws.amazon.com/bedrock/amazon-models/privacy/), [Bedrock FAQs](https://aws.amazon.com/bedrock/faqs/)

#### 🎯 On the exam

- **Reflex:** "Will AWS use our customer support transcripts sent to Bedrock to improve Claude?" → **No** — prompts and outputs are never used to train base models.
- **Trap:** This guarantee applies to base model inference. If you deliberately fine-tune a model with your data, that data is used to train **your private copy** of the model — not the base model.

---

### 3.3 Amazon Macie for PII Discovery in S3

[Amazon Macie](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html) is a data security service that uses ML to **discover, classify, and protect sensitive data stored in Amazon S3**. In an AI governance context, use Macie to:

- Scan S3 buckets that contain training datasets, knowledge base source documents, or invocation log archives for inadvertent PII.
- Generate findings for buckets that are publicly accessible or unencrypted.
- Integrate findings with AWS Security Hub for centralized remediation tracking.

Macie is **not** a real-time inference guardrail — it operates on S3 objects at rest, not on live inference traffic.

#### 🎯 On the exam

- **Reflex:** "Scan the S3 bucket containing RAG source documents for sensitive customer data before indexing." → **Amazon Macie**.
- **Trap:** Macie is for S3 data discovery, not runtime PII filtering. For runtime, use Bedrock Guardrails sensitive information filters.

---

### 3.4 Amazon Comprehend PII Detection & Toxicity

[Amazon Comprehend](https://docs.aws.amazon.com/comprehend/latest/dg/what-is.html) provides NLP-based PII detection and toxicity classification for use in **custom application logic** outside of Bedrock Guardrails.

- **PII detection API** (`DetectPiiEntities`, `ContainsPiiEntities`) — identify PII in free-form text; supports batch processing.
- **Toxicity detection** (`DetectToxicContent`) — classify text across categories: GRAPHIC, HATE\_SPEECH, INSULT, PROFANITY, SEXUAL, VIOLENCE\_OR\_THREAT.
- Useful when you need **custom routing logic**: e.g., route flagged content to a human reviewer rather than simply blocking it.

#### 🎯 On the exam

- **Reflex:** "Need custom PII detection logic in a Lambda function independent of Bedrock Guardrails." → **Amazon Comprehend PII detection**.
- **Reflex:** "Classify toxicity of user-generated text before it reaches the model." → **Amazon Comprehend toxicity detection**.
- **Trap:** Comprehend is a standalone NLP service — it does not integrate natively into the Bedrock inference path. For inline guardrailing, use Bedrock Guardrails.

---

## 4. Responsible AI & Governance

### 4.1 Bias Mitigation, Fairness & Transparency

AWS advocates a set of **responsible AI dimensions** across all ML/AI services:

- **Fairness** — Evaluate model outputs for disparate performance across demographic groups.
- **Explainability** — Produce reasoning traces and use SageMaker Clarify SHAP values for feature attribution.
- **Agent reasoning traces (Task 3.4.1 — transparency)** — [Amazon Bedrock Agents can emit a detailed **trace**](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-trace.html) of each step: the model's reasoning, which tool/action group was invoked, which Knowledge Base chunks were retrieved, and the final synthesis. Enable traces in `InvokeAgent` requests (`enableTrace: true`) and surface them as **user-facing explanations or source-attribution evidence** — for example, showing end-users which document supported an answer, or giving auditors a full chain-of-thought log. This is the primary AWS mechanism for making multi-step agent decisions **transparent and auditable** without building custom logging.
- **Robustness** — Test models against adversarial inputs, edge cases, and out-of-distribution prompts.
- **Privacy** — Apply data minimization; avoid including personal data in prompts or training sets.
- **Human oversight** — Keep humans in the loop for high-stakes decisions; use "human review" nodes in SageMaker Pipelines or Amazon A2I (Augmented AI).
- **Transparency** — Document model capabilities, limitations, and design choices (see AI Service Cards below).

---

### 4.2 AWS AI Service Cards

[AWS AI Service Cards](https://aws.amazon.com/machine-learning/responsible-machine-learning/ai-service-cards/) are transparency documents published by AWS for each major AI service and model. Each card covers:

- **Intended use cases and limitations**
- **Responsible AI design choices** — methodology, fairness/bias testing approach, explainability mechanisms, performance expectations
- **Performance optimization best practices**
- **Deployment guidance**

Service Cards exist for services such as Amazon Rekognition, Amazon Transcribe, Amazon Comprehend, and Amazon Bedrock foundation models. They are the primary accountability artifact for "what did AWS intend this model to do, and what are its known limitations?"

Source: [Introducing AWS AI Service Cards](https://aws.amazon.com/blogs/machine-learning/introducing-aws-ai-service-cards-a-new-resource-to-enhance-transparency-and-advance-responsible-ai/)

#### 🎯 On the exam

- **Reflex:** "Where can a developer find documented intended use cases and bias testing methodology for an AWS AI service?" → **AWS AI Service Cards**.

---

### 4.3 SageMaker & Bedrock Model Evaluation for Safety

**Amazon Bedrock Model Evaluation** allows you to evaluate foundation models on custom or built-in prompt datasets and score responses for:
- Accuracy / quality
- Toxicity
- Robustness
- Prompt stereotyping and bias

**Amazon SageMaker Clarify** provides bias detection for both pre-training (data) and post-training (model predictions):
- Pre-training bias metrics: Class Imbalance (CI), Difference in Proportions of Labels (DPL)
- Post-training bias metrics: Disparate Impact (DI), Accuracy Difference
- Explainability: SHAP-based feature attribution, partial dependence plots

**SageMaker Model Cards** — structured documentation cards for SageMaker-trained models, covering intended use, evaluation results, and risk ratings. They are analogous to AWS AI Service Cards but are customer-authored for custom models.

**SageMaker Model Dashboard** — centralized view of all deployed models with monitoring status, bias drift alerts, and data quality metrics.

**Amazon A2I (Augmented AI)** — routes low-confidence or flagged model predictions to **human reviewers** via private or public workforce. Integrate with Bedrock pipelines via Lambda for human-in-the-loop escalation.

Source: [SageMaker governance announcement](https://aws.amazon.com/blogs/aws/new-amazon-sagemaker-role-manager-amazon-sagemaker-model-cards-and-amazon-sagemaker-model-dashboard/)

#### 🎯 On the exam

- **Reflex:** "Evaluate a foundation model for bias across demographic groups before deploying." → **SageMaker Clarify** or **Bedrock Model Evaluation**.
- **Reflex:** "Route uncertain model decisions to human reviewers." → **Amazon A2I**.
- **Trap:** SageMaker Clarify works on structured ML models (tabular data); for generative AI evaluation, Bedrock Model Evaluation is the primary service.

---

### 4.4 Watermarking of Generated Content

> **Why (the rationale):** Regulatory and ethical requirements increasingly mandate disclosure when content is AI-generated. An invisible watermark that survives common image transformations provides a cryptographically verifiable provenance signal without degrading image quality for end users.
> **When to use:** Any workflow that generates images with Titan Image Generator and needs to verify AI provenance — content moderation, regulatory disclosure, deepfake detection. Use `DetectGeneratedContent` API to check a submitted image.
> **Nuances & gotchas:** The watermark is **specific to Amazon Titan Image Generator** — images from Stable Diffusion, DALL·E, Midjourney, or other Bedrock image models do NOT carry this watermark and cannot be detected by this API. You **cannot opt out** of watermarking when using Titan Image Generator. The watermark survives moderate post-processing (JPEG compression, resizing, cropping) but may be destroyed by heavy editing or adversarial attacks.

**Amazon Titan Image Generator** automatically embeds an **invisible watermark** in every image it produces by default — you cannot opt out. The watermark is imperceptible to humans and survives common image transformations (cropping, resizing, JPEG compression).

**`DetectGeneratedContent` API** — use this Bedrock API to check whether an image was produced by Titan Image Generator. The API returns:
- A **confidence score** (0.0–1.0) indicating likelihood that the image is AI-generated by Titan
- Detection works even after moderate post-processing of the image

This addresses requirements for content provenance, deepfake disclosure, and regulatory mandates around AI-generated media transparency.

Source: [Watermark detection API announcement](https://aws.amazon.com/about-aws/whats-new/2024/04/watermark-detection-amazon-titan-image-generator-bedrock/), [Titan Image Generator availability](https://favtutor.com/articles/amazon-titan-image-generator-watermark-detection-api/)

#### 🎯 On the exam

- **Reflex:** "Verify whether a submitted image was AI-generated by Amazon Titan." → **`DetectGeneratedContent` API**.
- **Trap:** The watermark is specific to **Amazon Titan Image Generator** — images from Stable Diffusion, DALL·E, or other models on Bedrock do not carry this watermark and cannot be detected by this API.

---

### 4.5 Generative AI Security Scoping Matrix

The [Generative AI Security Scoping Matrix](https://aws.amazon.com/ai/security/generative-ai-scoping-matrix/) is AWS's framework for mapping security controls to how an organization engages with generative AI. It defines **five scopes** ordered from least to greatest organizational ownership:

| Scope | Description | Example | Key security concern |
|---|---|---|---|
| **Scope 1 — Consumer App** | Use a public third-party AI product as-is | ChatGPT, Google Bard | Data in prompts may train the provider's model; no contractual protection |
| **Scope 2 — Enterprise App** | Deploy a third-party AI product with enterprise contracts | Microsoft Copilot (M365), Salesforce Einstein | Enforce data processing agreements; prevent proprietary data use for training |
| **Scope 3 — Pre-trained Model API** | Build apps using foundation model APIs (e.g., Bedrock) | RAG chatbot on Claude | Prompt injection, access control, data residency, guardrails |
| **Scope 4 — Fine-tuned Model** | Fine-tune a foundation model on your own data | Domain-specific Llama fine-tune | Training data sensitivity (avoid PII in fine-tuning data), model exfiltration |
| **Scope 5 — Self-trained Model** | Build and train a model from scratch | Custom LLM trained on proprietary corpus | Full model lifecycle security, data governance, infrastructure security |

Most AIP-C01 scenarios fall in **Scope 3** (Bedrock) and **Scope 4** (fine-tuning). The matrix guides which controls are your responsibility vs. AWS's responsibility under the shared responsibility model.

Source: [Generative AI Security Scoping Matrix](https://aws.amazon.com/ai/security/generative-ai-scoping-matrix/), [Securing generative AI blog](https://aws.amazon.com/blogs/security/securing-generative-ai-an-introduction-to-the-generative-ai-security-scoping-matrix/)

#### 🎯 On the exam

- **Reflex:** "Classify the security posture for an organization using Amazon Bedrock with RAG." → **Scope 3** of the Generative AI Security Scoping Matrix.
- **Reflex:** "Who is responsible for preventing PII from entering a fine-tuning dataset?" → The **customer** (Scope 4 — fine-tuning is customer-managed data).

---

### 4.6 Audit & Logging — CloudTrail and Model-Invocation Logging

Two distinct logging systems work together; the exam tests whether you know what each captures.

#### AWS CloudTrail

> **Why (the rationale):** CloudTrail is the mandatory audit backbone for demonstrating who invoked which model, when, and from which identity — essential for compliance audits (SOC 2, HIPAA, PCI-DSS) and security investigations. It is the only way to prove unauthorized access after the fact without model invocation logging.
> **When to use:** Always (enabled by default for management events). Create a CloudTrail Trail delivering to an encrypted S3 bucket for long-term retention beyond the default 90-day console history.
> **Nuances & gotchas:** CloudTrail records **that** `InvokeModel` was called and by whom, but does **NOT** capture the prompt or response content — that requires Model Invocation Logging. CloudTrail data events for S3 and Lambda are NOT enabled by default; enable them separately if you need object-level access auditing on your ingestion buckets.

CloudTrail automatically records **management (control-plane) events** for Amazon Bedrock — who called which API action, when, from which IP, with which IAM principal. CloudTrail does **not** capture the content of prompts or model responses.

**What CloudTrail captures for Bedrock:**
- `InvokeModel` — records that the API was called, by whom, with which model, but not the prompt/response content
- `CreateGuardrail`, `UpdateGuardrail`, `DeleteGuardrail` — guardrail lifecycle events
- `CreateKnowledgeBase`, `StartIngestionJob` — knowledge base operations
- Failed/denied API calls — useful for detecting unauthorized access attempts

CloudTrail is **enabled by default** and stores 90 days of management events in the console Event History. For longer retention, create a **CloudTrail trail** delivering to an S3 bucket (optionally encrypted with KMS CMK).

#### Bedrock Model Invocation Logging

> **Why (the rationale):** CloudTrail tells you that a model was called; invocation logging tells you what was said. This is required for compliance regimes that mandate retaining the full content of AI interactions (e.g., financial services communications records) and for debugging unexpected model outputs in production.
> **When to use:** Enable when compliance requires retaining prompt/response content, when debugging production failures, or when collecting input/output pairs for fine-tuning dataset construction.
> **Nuances & gotchas:** **Disabled by default** — must be explicitly enabled per Region. Raw input appears unmasked in logs even when Guardrails PII masking is active; configure CloudWatch log data protection separately to mask PII in log streams. Log payloads > 100 KB are stored as separate S3 objects with a reference in the log entry. Invocation logging currently does NOT capture calls through the `bedrock-mantle` (Responses API) endpoint.

[Model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) captures the **full content** of requests and responses — prompts, completions, token counts, model IDs, and request metadata.

**Key facts:**
- **Disabled by default** — must be explicitly enabled per Region via the Bedrock console or `PutModelInvocationLoggingConfiguration` API.
- **Supported operations:** `Converse`, `ConverseStream`, `InvokeModel`, `InvokeModelWithResponseStream`
- **Destinations:** Amazon CloudWatch Logs log group, Amazon S3 bucket, or both (must be in same account and Region)
- **Log entry fields:** `schemaType`, `timestamp`, `accountId`, `region`, `requestId`, `operation`, `modelId`, `identity.arn`, `input.inputBodyJson` (up to 100 KB), `output.outputBodyJson` (up to 100 KB), `input.inputTokenCount`, `output.outputTokenCount`
- **Large payloads:** Bodies > 100 KB and binary image data are stored as separate S3 objects; the log entry contains a reference

**S3 destination setup:** Grant `s3:PutObject` to `bedrock.amazonaws.com` with `aws:SourceAccount` and `aws:SourceArn` conditions. Disable bucket ACLs; enable SSE-KMS with a key policy granting `kms:GenerateDataKey` to `bedrock.amazonaws.com`.

**CloudWatch destination setup:** Create a log group; create an IAM role with `logs:CreateLogStream` and `logs:PutLogEvents` trusted by `bedrock.amazonaws.com`.

**Important:** Raw input always appears unmasked in logs, even when PII masking is active at the guardrail layer. Use [CloudWatch log data protection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html) to mask sensitive fields in log streams.

**Useful CloudWatch Logs Insights query — token usage by caller:**
```
fields identity.arn as principal, input.inputTokenCount as inTokens, output.outputTokenCount as outTokens
| stats sum(inTokens) as totalInput, sum(outTokens) as totalOutput, count() as calls by principal
| sort totalInput desc
```

Source: [Monitor model invocation using CloudWatch Logs and Amazon S3](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)

#### 🎯 On the exam

- **Reflex:** "Determine who called which Bedrock model at what time." → **CloudTrail** (management events).
- **Reflex:** "Capture the full prompt and response content for compliance review." → **Model invocation logging** to S3 or CloudWatch.
- **Trap:** CloudTrail records the invocation event but **not the prompt or response content** — model invocation logging captures content, but it is **off by default**.
- **Trap:** Model invocation logging only works for calls through the `bedrock-runtime` endpoint — Responses API calls through `bedrock-mantle` are currently not captured.
- **Reflex:** "Long-term archival of invocation logs with Athena querying." → **Model invocation logging to S3** (gzipped JSON queryable via Athena or S3 Select).

---

### 4.7 Compliance Frameworks

Amazon Bedrock maintains compliance with the following standards (current as of 2026):

| Framework | Scope |
|---|---|
| **GDPR** | Data processing agreements, data residency within Region |
| **HIPAA** | Bedrock is HIPAA-eligible with a BAA |
| **FedRAMP Moderate** | US government workloads |
| **SOC 1, 2, 3** | Operational controls |
| **ISO 27001, 27017, 27018, 27701** | Information security, cloud security, PII |
| **ISO 22301** | Business continuity |
| **ISO 9001** | Quality management |
| **CSA STAR Level 2** | Cloud security assurance |

For the exam, remember: **Bedrock is HIPAA-eligible** (requires BAA) and **FedRAMP Moderate** certified (FIPS endpoints exist for government use).

Source: [Amazon Bedrock security and compliance](https://aws.amazon.com/bedrock/security-compliance), [Bedrock FAQs](https://aws.amazon.com/bedrock/faqs/)

---

## 5. Prompt-Injection & Jailbreak Defense

Prompt injection and jailbreak attacks attempt to override a model's instructions, bypass safety controls, or extract confidential system prompts. Defense requires a layered approach:

### Detection Layer — Bedrock Guardrails

- **Prompt Attack content filter** (Standard tier) — detects jailbreaks, prompt injections, and prompt leakage with a configurable strength level and an independent `PROMPT_ATTACK` finding type.
- **ApplyGuardrail API** — pre-screen user inputs before they reach the model.

### Architecture Layer

- **System prompt isolation** — place instructions in the `system` role (Converse API) or use a designated system prompt field; never concatenate user input directly into the system prompt.
- **Input validation** — apply strict input length limits and character filtering before passing user content to the model.
- **Minimal context exposure** — do not include credentials, internal configuration, or sensitive business logic in prompts. Use external knowledge retrieval (RAG/Knowledge Bases) instead.
- **Agent tool permission scoping** — in Bedrock Agents, define Action Group functions with minimal permissions; validate tool inputs before executing them.

### Output Validation Layer

- **Structured output enforcement** — use Converse API tool use (function calling) to constrain model outputs to a defined schema, making injection-influenced free-form text less likely to reach downstream systems.
- **Contextual grounding check** — verify that outputs align with retrieved sources; injected fabrications will score low on grounding.
- **Human-in-the-loop** — for high-risk actions (write operations, financial transactions), require human approval before acting on model output.

### Logging & Detection Layer

- **CloudTrail + model invocation logging** — detect anomalous patterns (unusual prompt lengths, off-hours access, unexpected model selections) with CloudWatch alarms or GuardDuty anomaly detection.
- **AWS WAF** — if the Bedrock API is fronted by an API Gateway or ALB, apply WAF rules to filter known injection strings before they reach the application layer.

#### 🎯 On the exam

- **Reflex:** "Detect when a user is trying to jailbreak the model." → **Prompt Attack category in content filters** (Standard tier) or standalone prompt-attack check.
- **Reflex:** "Prevent a malicious document in a RAG pipeline from overriding the system prompt." → **Input validation + Prompt Attack guardrail + contextual grounding check**.
- **Trap:** No single control eliminates prompt injection risk — the exam expects you to recommend a **layered defense** (guardrails + architecture + output validation).

---

## 6. Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Guardrail** | A versioned policy bundle applied to Bedrock model calls | Enforce safety, content, PII, and grounding rules at inference time |
| **Content filter** | ML classifier for six harm categories with configurable strength | Block or flag harmful language in prompts and responses |
| **Denied topic** | Natural-language description of a subject the app must not discuss | Keep a chatbot on-topic |
| **Word filter** | Exact-string blocklist | Block specific words or phrases deterministically |
| **Sensitive information filter** | ML-based PII detection with block/mask modes | Prevent PII leakage in inputs and outputs |
| **Custom regex filter** | User-defined pattern (e.g., `EMP-\d{5}`) inside sensitive information filters | Detect proprietary identifiers not in built-in PII list |
| **Contextual grounding check** | Scores a response's factual alignment with a source document | Detect and filter RAG hallucinations |
| **Grounding score** | 0–1 measure of factual consistency between response and source | Configurable threshold; responses below threshold are blocked |
| **Relevance score** | 0–1 measure of whether the response answers the user's query | Configurable threshold; low relevance responses are blocked |
| **Automated Reasoning checks** | Formal logic/mathematical verification of policy compliance | Prove (not just estimate) that a response follows business rules |
| **ApplyGuardrail API** | Guardrail evaluation decoupled from model invocation | Apply Bedrock safety controls to any text, including non-Bedrock models |
| **VPC interface endpoint** | Private network entry point for a Bedrock API family | Keep inference traffic off the public internet |
| **AWS PrivateLink** | Technology powering VPC interface endpoints | Secure, private connectivity to AWS services |
| **Endpoint policy** | IAM resource-based policy attached to a VPC endpoint | Restrict which principals and actions flow through the endpoint |
| **Customer-managed key (CMK)** | KMS key the customer owns and controls | Customer-controlled encryption for Bedrock stored resources |
| **Model invocation logging** | Bedrock feature that records full prompt/response content | Compliance audit trail; off by default |
| **CloudTrail** | AWS service recording all API management events | Who called which API, when — does not capture prompt/response content |
| **AWS AI Service Card** | AWS-published transparency doc for an AI service or model | Documents intended use, limitations, and responsible AI design choices |
| **Generative AI Security Scoping Matrix** | AWS framework mapping security controls to five AI adoption patterns | Determine your security responsibilities based on how deeply you customize AI |
| **Amazon Macie** | ML-based S3 data classification service | Discover PII in S3 buckets (training data, logs) |
| **Amazon Comprehend** | NLP service with PII detection and toxicity classification | Custom PII/toxicity logic outside Bedrock inference path |
| **DetectGeneratedContent API** | Bedrock API to check for Titan Image Generator watermark | Verify AI provenance of an image |
| **Invisible watermark** | Imperceptible mark embedded in Titan-generated images by default | Content provenance and AI disclosure |
| **SageMaker Clarify** | Bias detection and explainability tool for ML models | Measure and mitigate bias in structured ML or generative output |
| **SageMaker Model Card** | Customer-authored documentation for a custom ML model | Governance artifact recording intended use and evaluation results |
| **Amazon A2I** | Human review workflow service | Route uncertain or flagged AI decisions to human reviewers |
| **Agent reasoning trace** | Step-by-step log of a Bedrock Agent's thinking, tool calls, and KB lookups | Transparency and auditability of multi-step agent decisions; surfaced as user-facing explanations |
| **Prompt injection** | Attack where a user's input overrides the model's system instructions | Defeated by input validation, prompt architecture, and Prompt Attack guardrail |
| **Jailbreak** | Attempt to make a model ignore safety training | Detected by Prompt Attack content filter and guardrail checks |
| **FedRAMP** | US government compliance framework | Required for Bedrock use in federal workloads; FIPS endpoints available |
| **HIPAA-eligible** | Bedrock can be used for PHI with a signed BAA | Required for healthcare applications |

---

## 7. References

- [Amazon Bedrock Agents — trace and reasoning logs](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-trace.html)
- [Amazon Bedrock Guardrails — components overview](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-components.html)
- [Sensitive information filters — PII types and configuration](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-sensitive-filters.html)
- [Contextual grounding check documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html)
- [Automated Reasoning checks in Bedrock Guardrails](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-guardrails-policy-based-enforcement-responsible-ai/)
- [ApplyGuardrail API blog post](https://aws.amazon.com/blogs/machine-learning/use-the-applyguardrail-api-with-long-context-inputs-and-streaming-outputs-in-amazon-bedrock/)
- [Guardrails tiers (Standard vs. Basic)](https://aws.amazon.com/about-aws/whats-new/2025/06/amazon-bedrock-guardrails-tiers-content-filters-denied-topics)
- [New Guardrails API for agentic AI workflows (2026)](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-guardrails-api-ai/)
- [VPC interface endpoints for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html)
- [Use AWS PrivateLink to set up private access to Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/use-aws-privatelink-to-set-up-private-access-to-amazon-bedrock)
- [Implementing least privilege access for Amazon Bedrock](https://aws.amazon.com/blogs/security/implementing-least-privilege-access-for-amazon-bedrock/)
- [Data encryption in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-encryption.html)
- [Amazon Bedrock security, privacy & responsible AI](https://aws.amazon.com/bedrock/security-privacy-responsible-ai)
- [Amazon Bedrock security and compliance certifications](https://aws.amazon.com/bedrock/security-compliance)
- [Amazon Bedrock FAQs — security and privacy section](https://aws.amazon.com/bedrock/faqs/)
- [Amazon model training privacy](https://aws.amazon.com/bedrock/amazon-models/privacy/)
- [Monitor model invocation using CloudWatch Logs and Amazon S3](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [Configure model invocation logging using CloudFormation](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/configure-bedrock-invocation-logging-cloudformation.html)
- [Generative AI Security Scoping Matrix](https://aws.amazon.com/ai/security/generative-ai-scoping-matrix/)
- [Securing generative AI — intro to the Scoping Matrix](https://aws.amazon.com/blogs/security/securing-generative-ai-an-introduction-to-the-generative-ai-security-scoping-matrix/)
- [Introducing AWS AI Service Cards](https://aws.amazon.com/blogs/machine-learning/introducing-aws-ai-service-cards-a-new-resource-to-enhance-transparency-and-advance-responsible-ai/)
- [AWS AI security hub](https://aws.amazon.com/ai/security/)
- [Watermark detection for Amazon Titan Image Generator](https://aws.amazon.com/about-aws/whats-new/2024/04/watermark-detection-amazon-titan-image-generator-bedrock/)
- [Guardrails can now detect hallucinations (contextual grounding launch)](https://aws.amazon.com/about-aws/whats-new/2024/07/guardrails-bedrock-hallucinations-safeguard-apps-fm/)
- [Amazon Bedrock Security and Governance reference](https://hidekazu-konishi.com/entry/amazon_bedrock_security_and_governance_guide.html)
- [SageMaker Role Manager, Model Cards, and Model Dashboard](https://aws.amazon.com/blogs/aws/new-amazon-sagemaker-role-manager-amazon-sagemaker-model-cards-and-amazon-sagemaker-model-dashboard/)

---

*See also: [Security and Governance services reference](../services/security-and-governance.md) · [Amazon Bedrock service deep-dive](../services/bedrock.md)*

*Part of the [AWS AI Certification course](../README.md) · AIP-C01 track.*
