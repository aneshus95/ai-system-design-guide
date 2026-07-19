# Amazon Comprehend

**Amazon Comprehend** is a fully managed natural-language-processing (NLP) service that uses machine learning to find insights and relationships in **unstructured text** — entities, key phrases, sentiment, language, syntax, PII, and custom labels — without you training or hosting any models.

---

## 🧠 Mental model

Think of Comprehend as a **highlighter pen that reads for you**. You hand it a wall of raw text (support tickets, reviews, contracts, emails) and it automatically highlights *who/what is mentioned* (entities), *what it's about* (key phrases / topics), *how the writer feels* (sentiment), *what language it is*, and *where the sensitive personal data hides* (PII) — then hands the highlights back as clean JSON you can act on.

You don't teach it grammar or feelings. AWS already trained the models; you just call an API. When the built-in categories aren't enough, you give it a few thousand labeled examples and it trains a **custom** highlighter for *your* domain (e.g., "which of these are insurance claim numbers?").

```mermaid
flowchart LR
    T["📄 Raw text<br/>(reviews, tickets,<br/>docs, emails)"] --> C{Amazon<br/>Comprehend}
    C --> E["Entities<br/>(people, places, orgs, dates)"]
    C --> K["Key phrases"]
    C --> S["Sentiment<br/>+ Targeted sentiment"]
    C --> L["Dominant language"]
    C --> Y["Syntax / PoS"]
    C --> P["PII detect<br/>& redact"]
    C --> M["Topic modeling<br/>(batch only)"]
    C --> CU["Custom classification<br/>+ custom entities"]
```

**Input → output at a glance:**

| You send | Comprehend returns |
|---|---|
| "The staff at AnyCompany in Seattle were amazing!" | Entities: `AnyCompany`(ORG), `Seattle`(LOCATION); Sentiment: `POSITIVE`; Language: `en` |
| A review: "The tacos were great but the wait was terrible." | **Targeted** sentiment: `tacos`→POSITIVE, `wait`→NEGATIVE |
| "Call me at 555-0100, SSN 123-45-6789" | PII entities `PHONE`, `SSN` with offsets → redacted copy: "Call me at \*\*\*, SSN \*\*\*" |
| 10,000 support emails | Custom classification label per email (e.g., `billing` / `bug` / `feature-request`) |

---

## What it does

**Built-in (pre-trained) APIs — no training required:**

- **Entity recognition** — detects named entities: `PERSON`, `LOCATION`, `ORGANIZATION`, `COMMERCIAL_ITEM`, `EVENT`, `DATE`, `QUANTITY`, `TITLE`, `OTHER`.
- **Key phrase extraction** — pulls out the main noun phrases ("what the text is about").
- **Sentiment analysis** — one dominant sentiment per document: `POSITIVE`, `NEGATIVE`, `NEUTRAL`, or `MIXED`, each with a confidence score.
- **Targeted sentiment** — sentiment *per entity* within the text (e.g., positive about "tacos", negative about "service"), including co-reference groups. English only.
- **Language detection** — identifies the dominant language (100+ languages) and confidence; great as a pre-step before Translate.
- **Syntax analysis** — part-of-speech tagging (noun, verb, adjective…) for each token.
- **PII detection & redaction** — `ContainsPiiEntities` (does this doc contain PII?), `DetectPiiEntities` (where + what type), and redaction jobs that produce a masked copy. Covers names, SSNs, credit cards, addresses, bank/routing numbers, emails, etc.
- **Toxicity detection & prompt-safety classification** — flag harmful content and unsafe LLM prompts (used in responsible-AI / GenAI guardrail scenarios).

**Batch-only (asynchronous) analysis:**

- **Topic modeling** — unsupervised; groups a corpus of documents into topics by common word groups (based on LDA). **No labeling needed**, runs only as an async job over S3.

**Custom (you supply labeled data, Comprehend trains + hosts the model):**

- **Custom classification** — train on your own labels/document types (e.g., route tickets, tag documents). Multi-class or multi-label.
- **Custom entity recognition (CER)** — teach it to find domain-specific entities the built-in model doesn't know (policy numbers, part IDs, gene names).
- Comprehend can **auto-extract text from PDF, image, and Word inputs** before running custom classification / CER (built-in OCR-style handling).

**Comprehend Medical** — a *separate*, HIPAA-eligible service for **clinical text**:

- `DetectEntitiesV2` — medical conditions, medications, dosages, anatomy, tests, treatments, procedures.
- `DetectPHI` — protected health information (a medical-specific PII).
- **Ontology linking** — `InferICD10CM` (diagnoses), `InferRxNorm` (medications/RxCUI), `InferSNOMEDCT` (medical concepts) map extracted terms to standard medical codes.
- Real-time and async batch modes.

**Delivery modes:** most APIs offer **real-time (synchronous, single doc)** and **batch/async (over S3)**. Topic modeling is async-only.

---

## When to use it (and vs alternatives)

| Scenario | Use | Why |
|---|---|---|
| Extract entities / sentiment / PII / key phrases from text | **Amazon Comprehend** | Purpose-built, pre-trained, single API call, cheap |
| Classify or tag docs with *your own* labels | **Comprehend custom classification** | Train with labeled examples; managed hosting |
| Find domain-specific entities (part #s, claim IDs) | **Comprehend custom entity recognition** | Managed training for entities not in the built-in set |
| PHI / medical entities / ICD-10 / RxNorm codes | **Comprehend Medical** | HIPAA-eligible, medical ontologies |
| Group an unlabeled corpus into themes | **Comprehend topic modeling** | Unsupervised, no labels needed |
| You need full control over the model / novel NLP task | **SageMaker (build your own)** | Custom architecture, but you train, tune, host, MLOps |
| Free-form generation, summarization, Q&A, translation, reasoning over text | **Amazon Bedrock (LLM)** | Generative; Comprehend is *analysis/extraction*, not generation |
| Extract text from scanned docs/forms first | **Amazon Textract** → then Comprehend | Textract = OCR/document layout; Comprehend = NLP on the text |

**Comprehend vs building your own NLP:** Comprehend is managed and pay-per-use with no ML expertise required. Build-your-own (SageMaker) only wins when you need a task Comprehend can't do, or full model control — at the cost of labeling, training, tuning, and hosting.

**Comprehend vs Bedrock/LLM:** Comprehend is *deterministic, structured extraction/classification* returning JSON with confidence scores and character offsets — ideal for pipelines, redaction, and analytics at low cost. An LLM (Bedrock) can also extract entities/sentiment via a prompt and is more flexible, but is pricier, less structured, and can hallucinate. On the exam, **extract structured NLP signals from text → Comprehend**, not an LLM.

---

## Pricing model

Pay-per-use, no minimums. The pricing **unit for built-in NLP APIs is 100 characters**, with a **3-unit (300-character) minimum charge per request**. Representative US pricing:

| Dimension | Unit | Price (approx.) |
|---|---|---|
| Entity recognition, Sentiment, Language detection, Syntax, **Targeted sentiment** | per unit (100 chars) | ~$0.0001 |
| **Contains PII** | per unit | ~$0.000002 (much cheaper — just a yes/no) |
| **Detect PII / redaction** | per unit | ~$0.0001 |
| Toxicity / prompt-safety | per unit (tiered) | ~$0.0001, dropping with volume |
| Event detection | per unit | ~$0.003 |
| **Custom — training** | per hour | ~$3/hour |
| **Custom — model management** | per model/month | ~$0.50 |
| **Custom — async inference** | per unit | ~$0.0005 |
| **Custom — real-time endpoint** | per second per Inference Unit (1 IU = 100 chars/s) | ~$0.0005, **60-second minimum**, billed while endpoint is up |
| Topic modeling | first 100 MB flat, then per MB | ~$0.004/MB above 100 MB |

**Free tier:** ~50K units/month (5M characters) across eligible built-in APIs for 12 months. **Custom Comprehend is excluded** from the free tier.

> 💡 Exam-relevant cost trap: a **custom real-time endpoint bills continuously** (per Inference Unit-second) while it exists — delete/scale it down when idle. For sporadic bulk work, use **async batch** instead of a live endpoint. Comprehend Medical is billed separately (per 100-char unit). *Always confirm current numbers on the pricing page.*

---

## 🎯 On the exam

**Reflexes — "if you see X, pick Comprehend":**

- "Extract **entities / key phrases / sentiment / PII** from text" → **Comprehend**.
- "Detect the **language** of incoming text before translating" → **Comprehend** (language detection) → then **Translate**.
- "**Redact** SSNs / credit cards / personal data from documents" → **Comprehend PII detection & redaction**.
- "Sentiment **for each product/feature** mentioned in a review" → **Comprehend targeted sentiment** (not plain sentiment).
- "Route/tag documents using **our own categories**" → **Comprehend custom classification**.
- "Find **our domain-specific** entities (policy #, part ID)" → **Comprehend custom entity recognition**.
- "Group **unlabeled** documents into themes / topics" → **Comprehend topic modeling** (unsupervised, batch-only).
- "Extract **medical conditions, medications, PHI, ICD-10 / RxNorm** codes" → **Comprehend Medical** (HIPAA-eligible).

**Traps & distractors:**

- **Comprehend vs Textract:** Textract does **OCR** (get text/tables/forms *out of* scanned docs); Comprehend does **NLP** *on* text. Scanned PDF → Textract first, then Comprehend. (For custom classification/CER, Comprehend can now auto-extract text from PDF/image/Word itself.)
- **Sentiment vs Targeted sentiment:** plain sentiment = one label for the *whole* document; targeted = per-entity. "Which dish did customers like/dislike?" ⇒ *targeted*.
- **Topic modeling ≠ classification.** Topic modeling is **unsupervised** (no labels, discovers themes). Custom classification is **supervised** (your labels). Async-only is a giveaway for topic modeling.
- **Comprehend Medical is a separate service** — pick it (not plain Comprehend) whenever PHI / clinical text / medical ontologies appear. It's HIPAA-eligible.
- **Don't reach for an LLM/Bedrock** when the ask is structured extraction/classification/redaction at scale — Comprehend is cheaper, structured, and returns offsets + confidence.
- **Real-time vs batch:** single doc / low latency → sync API; large volumes in S3 → async batch job.
- **Custom endpoints cost money while running** — a cost-optimization answer is "delete the endpoint / use batch."

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| Amazon Comprehend | A fully managed NLP service that extracts insights like entities, sentiment, and PII from unstructured text | Lets you analyze large volumes of text without training or hosting any ML models |
| NLP (Natural Language Processing) | The branch of AI that helps computers understand, interpret, and derive meaning from human language | The underlying technology powering all Comprehend features |
| Entity recognition | Identifying and classifying named things in text such as people, organizations, locations, and dates | Structures raw text so downstream systems can act on specific mentions automatically |
| Key phrase extraction | Pulling out the main noun phrases that summarize what a document is about | Quickly indexes or summarizes content without reading the full text |
| Sentiment analysis | Classifying the overall emotional tone of a document as Positive, Negative, Neutral, or Mixed | Useful for monitoring customer feedback, reviews, and support tickets at scale |
| Targeted sentiment | Sentiment measured per entity within a document rather than for the document as a whole | Answers questions like "what did customers think specifically about the delivery vs. the product?" |
| Language detection | Identifying which language a piece of text is written in | Commonly used as a first step before routing text to Amazon Translate |
| Syntax analysis | Labeling each word in a sentence with its grammatical role, such as noun, verb, or adjective | Used in pipelines that need fine-grained linguistic structure, like grammar checkers or parsers |
| PII detection | Finding personally identifiable information in text such as names, SSNs, phone numbers, or credit card numbers | Enables automated compliance workflows that must locate and handle sensitive personal data |
| PII redaction | Replacing detected PII in a document copy with masked placeholders | Produces a sanitized version of a document safe to share or store without exposing personal data |
| Toxicity detection | Flagging harmful, abusive, or offensive content in text | Used in moderation pipelines for user-generated content platforms |
| Prompt-safety classification | Categorizing whether an input to a language model is safe or a potential jailbreak attempt | Adds a guardrail layer to GenAI applications before the prompt reaches the model |
| Topic modeling | Unsupervised grouping of a large document corpus into clusters of related themes | Discovers hidden structure in unlabeled text without needing predefined categories |
| LDA (Latent Dirichlet Allocation) | The statistical algorithm behind Comprehend's topic modeling feature | Infers topics by finding words that frequently co-occur across documents |
| Custom classification | Training Comprehend on your own labeled examples to categorize documents into your own categories | Routes tickets, tags content, or classifies documents into domain-specific buckets |
| Custom entity recognition (CER) | Training Comprehend to find domain-specific entities not in the built-in model | Detects things like policy numbers, part IDs, or gene names that a generic model doesn't know |
| Multi-class classification | A classification mode where each document gets exactly one label | Used when documents belong to one clear category, like assigning a support ticket to one team |
| Multi-label classification | A classification mode where each document can receive multiple labels simultaneously | Used when a document legitimately belongs to more than one category at once |
| Amazon Comprehend Medical | A separate HIPAA-eligible Comprehend service specialized for clinical and medical text | Extracts medical entities, PHI, and maps them to standard medical codes like ICD-10 |
| PHI (Protected Health Information) | Medical data that can identify a patient, such as diagnoses, medications, or dates of service | Comprehend Medical detects PHI so healthcare systems can comply with HIPAA |
| ICD-10-CM | A standardized medical coding system for diagnoses used for billing and clinical records | Comprehend Medical maps extracted conditions to ICD-10 codes via InferICD10CM |
| RxNorm | A standard vocabulary for medications used across clinical systems | Comprehend Medical maps extracted drug mentions to RxNorm codes via InferRxNorm |
| SNOMED CT | A comprehensive international clinical terminology standard | Comprehend Medical maps extracted medical concepts to SNOMED CT via InferSNOMEDCT |
| Ontology linking | Mapping extracted medical terms to standardized code systems like ICD-10, RxNorm, or SNOMED CT | Allows downstream clinical systems to interpret extracted terms in a common language |
| Synchronous API | A Comprehend API mode that processes one document at a time and returns results immediately | Used for real-time, low-latency applications needing instant text analysis |
| Asynchronous batch job | A Comprehend API mode that processes large volumes of documents stored in S3 and returns results when done | Used for bulk processing of thousands or millions of documents cost-effectively |
| Inference Unit (IU) | The billing unit for a live Comprehend custom real-time endpoint representing 100 characters per second | Determines how much capacity your endpoint provides and how much you are billed per second |
| Amazon Textract | AWS's document OCR service that extracts text and structure from scanned files | Often used upstream of Comprehend to extract text from PDFs and images before NLP analysis |
| Amazon Translate | AWS's managed neural machine translation service | Used downstream of Comprehend's language detection to translate text into a target language |
| Amazon Bedrock | AWS's managed foundation model API service for generative AI | An alternative to Comprehend for flexible but costlier and less structured text analysis via LLMs |
| Confidence score | A number between 0 and 1 indicating how certain the model is about a prediction | Used to filter low-quality results or route uncertain predictions to human review |
| Character offset | The exact position in the source text where a detected entity or PII begins and ends | Allows precise redaction or highlighting of specific text spans in downstream processing |

## References

- Amazon Comprehend — product page: https://aws.amazon.com/comprehend/
- Amazon Comprehend Developer Guide: https://docs.aws.amazon.com/comprehend/latest/dg/what-is.html
- Targeted sentiment: https://docs.aws.amazon.com/comprehend/latest/dg/how-targeted-sentiment.html
- PII detection & redaction: https://docs.aws.amazon.com/comprehend/latest/dg/pii.html
- Custom classification: https://docs.aws.amazon.com/comprehend/latest/dg/how-document-classification.html
- Custom entity recognition: https://docs.aws.amazon.com/comprehend/latest/dg/custom-entity-recognition.html
- Topic modeling: https://docs.aws.amazon.com/comprehend/latest/dg/topic-modeling.html
- Amazon Comprehend Medical — product page: https://aws.amazon.com/comprehend/medical/
- Comprehend Medical Developer Guide (how it works): https://docs.aws.amazon.com/comprehend-medical/latest/dev/comprehendmedical-howitworks.html
- Amazon Comprehend pricing: https://aws.amazon.com/comprehend/pricing/
