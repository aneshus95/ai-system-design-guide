# PO Automation — GPT Extraction + Fine-Tuned BERT Classifier

> **My project.** Cut seller friction by **40%** by automating purchase-order (PO) attachment to sales quotes: a **fine-tuned BERT classifier (Azure ML)** identifies PO emails, then **Azure OpenAI GPT-3.5 with chain-of-thought prompting** extracts structured PO data from PDFs at **95% F1**.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Two-Stage Architecture](#what-i-built--two-stage-architecture)
- [Deep Dive 1 — Why Two Stages (BERT Gate → GPT Extract)](#deep-dive-1--why-two-stages-bert-gate--gpt-extract)
- [Deep Dive 2 — Structured Extraction from PDFs](#deep-dive-2--structured-extraction-from-pdfs)
- [Deep Dive 3 — Chain-of-Thought Prompting](#deep-dive-3--chain-of-thought-prompting)
- [Deep Dive 4 — Measuring 95% F1](#deep-dive-4--measuring-95-f1)
- [Deep Dive 5 — Fine-Tuning BERT in Azure ML](#deep-dive-5--fine-tuning-bert-in-azure-ml)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** Sellers had to manually find the right purchase-order document and attach it to the matching sales quote — slow, error-prone busywork repeated across thousands of emails, most of which weren't even POs.

**Task.** Automate the pipeline end-to-end: figure out *which* incoming emails are POs, pull the *structured data* out of the attached PDFs, and attach them to the right quote — reliably enough that sellers trust it.

**Action.** I built a **two-stage system**. A cheap, high-volume **fine-tuned BERT classifier** (trained/deployed in Azure ML) gates the firehose of email down to actual PO emails. Only those go to the expensive stage: **Azure OpenAI GPT-3.5 with chain-of-thought prompting** extracts the PO fields (number, line items, quantities, prices, dates) from the PDF into structured JSON, validated before attachment.

**Result.** **95% F1** on field extraction and a **40% reduction in seller friction** — the manual find-and-attach step largely disappeared.

---

## What I Built — Two-Stage Architecture

```
 incoming emails (high volume, mostly NOT POs)
        │
        ▼
 ┌────────────────────────┐   cheap, fast, one forward pass
 │  STAGE 1: BERT gate     │   "is this a PO email?"  (binary classifier,
 │  (fine-tuned, Azure ML) │    fine-tuned + deployed as an endpoint)
 └───────────┬────────────┘
             │ only PO emails pass  (expensive stage runs on the small subset)
             ▼
 ┌────────────────────────────────────────────────┐
 │  STAGE 2: GPT-3.5 extraction                     │
 │  chain-of-thought prompt over the PDF text →     │
 │  structured JSON: {po_number, line_items[],      │
 │  quantities, prices, dates}                      │
 │  + schema validation / retry                     │
 └───────────┬────────────────────────────────────┘
             ▼
    attach PO to the matching sales quote
```

---

## Deep Dive 1 — Why Two Stages (BERT Gate → GPT Extract)

> **Why (the rationale):** The two tasks have fundamentally different cost/accuracy profiles. Binary routing (PO vs not) has labeled data available, is high-volume, and is stable enough to fine-tune — making a fast, cheap encoder the right tool. Structured extraction is complex, schema-driven, document-specific reasoning that benefits from a capable generative model. Running GPT on the full email firehose would be wasteful and slow; restricting it to the PO subset (a small fraction) makes the expensive stage economically viable.
> **When to use:** A two-stage gate → extract architecture is appropriate when: (a) the routing decision is binary/stable with labeled data, (b) the extraction stage is significantly more expensive, and (c) the two stages have separable error modes. If labeled data for stage 1 is unavailable, a zero-shot LLM can gate, but at higher cost.
> **Nuances & gotchas:** Stage 1 recall is the critical metric — any PO email the BERT gate misclassifies as non-PO is silently lost (never extracted). Stage 1 false negatives are more costly than false positives (a non-PO reaching stage 2 is just a wasted GPT call; a PO missed at stage 1 is a missed attachment). Tune the BERT classifier threshold to prioritize recall over precision for this reason.

This is the most important design decision and the first thing an interviewer will probe.

- **The classifier is a cheap filter over *all* email; the LLM is expensive and only runs on the PO subset.** Fine-tuned BERT-family encoders run **~1–2 orders of magnitude lower cost/latency** and far higher throughput than LLM prompting (a label in **one forward pass** vs token-by-token generation). Running GPT on every email would be wasteful and slow.
- **Separation of concerns:** routing (high-volume, stable, binary) vs extraction (complex, schema-driven, reasoning-heavy). Each stage uses the right tool.
- **Why fine-tune BERT rather than use GPT for classification too:** with labeled PO/non-PO data available, **fine-tuned encoders match or beat zero-shot LLMs on accuracy** *and* are far cheaper/faster at scale. GPT's open-ended reasoning is wasted on a binary routing decision.

Sources: [Cost-aware model selection (arXiv 2602.06370)](https://arxiv.org/html/2602.06370) · [Fine-tuned BERT vs zero-shot LLMs (arXiv 2406.08660)](https://arxiv.org/pdf/2406.08660)

---

## Deep Dive 2 — Structured Extraction from PDFs

> **Why (the rationale):** Raw PDF text extraction collapses table structure — line-item rows become interleaved text without row/column context, making field extraction unreliable. Azure AI Document Intelligence's Layout model returns text with spatial structure (table rows, columns, key-value pairs) preserved, giving the LLM structured input it can reason over reliably. Schema constraints (JSON schema / function-calling) then enforce output structure so downstream validation is deterministic.
> **When to use:** OCR + layout extraction is necessary for scanned PDFs and for any digital PDF where tables or form fields are the primary content. For pure prose PDFs, simple text extraction is sufficient. Azure AI Document Intelligence Layout specifically is justified when the document mix includes scanned images and table-heavy formats.
> **Nuances & gotchas:** Even Layout model output isn't perfect — merged cells, rotated text, and multi-page tables cause parsing errors that propagate to the LLM. The LLM's structured output can be syntactically valid JSON but semantically wrong (right field names, wrong values) — numeric cross-checks (line-item sums vs total) are essential, not optional. GPT-3.5 JSON-mode does not guarantee schema adherence; only GPT-4o's strict Structured Outputs do — on 3.5, a validation-and-retry loop is required.

**The task:** given a PO PDF, return fields (PO number, line items, quantities, prices, dates) as valid JSON. The model is prompted with the target schema and returns a structured object.

**The hard parts (know these):**
- **Digital vs scanned PDFs.** A digital PDF has extractable embedded text; a **scanned/image PDF needs OCR first** — the LLM can't "see" pixels, and naive extraction loses layout context.
- **Tables are the crux.** Line-item tables have merged cells, multi-line rows, and column structure that simple text parsers mangle (reading row-by-row or column-by-column incorrectly).
- **Robust pipeline:** OCR/layout extraction (e.g., **Azure AI Document Intelligence** Layout model, which returns text, key-value pairs, and table row/column structure) → feed that *clean, structured* text to the LLM so it reasons over layout, not raw pixels.

**Guarding against wrong/hallucinated extractions:**
1. **Schema constraint** — a JSON schema (or function-calling) forces valid structure. *(Note: strict Structured Outputs that guarantee schema adherence require GPT-4o+; on GPT-3.5 I used JSON-mode / function-calling + validation, not the strict feature — see Caveats.)*
2. **Validation + retry** — validate the JSON (e.g., Pydantic) and retry on failure.
3. **Grounding & cross-checks** — extract only from the source text; cross-check numeric fields (line-item sums vs total).
4. **Human-in-the-loop** — route low-confidence extractions for review.

Sources: [Azure AI Document Intelligence — Read/Layout OCR](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/read?view=doc-intel-4.0.0) · [Azure OpenAI — Structured Outputs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/structured-outputs)

---

## Deep Dive 3 — Chain-of-Thought Prompting

> **Why (the rationale):** A multi-line PO table requires the model to first locate the line-items table, identify each row's structure, map columns to field names, then assemble the JSON — a multi-step reasoning task that a single-shot "extract these fields" prompt handles poorly. CoT externalizes this plan as intermediate steps before the final answer, reducing field-mapping errors on complex documents.
> **When to use:** CoT improves accuracy on tasks that require multi-step reasoning, structure discovery, or ambiguity resolution (tables, nested forms, multi-page documents). For simple, single-field extraction from well-structured text, CoT's added tokens are wasted cost. Gains are strongest on larger, more capable models.
> **Nuances & gotchas:** CoT increases token usage 2–4×, directly increasing latency and cost — only economically viable because the BERT gate limits this to the PO subset. The "chain of thought" in the output doesn't guarantee the model's internal process actually follows those steps; it can produce coherent but wrong reasoning. Keeping CoT prose separate from the final JSON output (via a scratch field or post-processing strip) is critical to prevent reasoning text from polluting the structured result.

**Chain-of-thought (CoT)** prompting elicits a series of **intermediate reasoning steps** before the final answer (Wei et al., 2022 — note Jason *Wei* is first author, Denny *Zhou* last; it's often mis-cited as "Zhou et al.").

**Why it helps extraction:** for a complex multi-line PO, forcing the model to first *reason about structure* — "locate the line-items table, then parse each row, then read the totals" — before emitting JSON reduces field-mapping errors versus a single-shot dump. This is the same "reason-before-answer" mechanism that gave CoT its benchmark gains on math/reasoning tasks.

**Trade-offs (be ready):**
- **Cost/latency** — CoT typically increases token use **2–4×** and adds latency (reasoning tokens generated before the answer). *This is another reason the BERT gate matters — we only pay CoT cost on the PO subset.*
- **Needs capable models** — CoT gains are strongest on larger models; small models can produce "coherent but wrong" reasoning.
- **Faithfulness** — the stated reasoning doesn't always reflect the model's true internal process.
- **Keeping JSON clean** — let the model reason in a scratch field, then emit the final structured object (or reason then strip), so CoT prose doesn't pollute the JSON.

Sources: [Chain-of-Thought Prompting (arXiv 2201.11903)](https://arxiv.org/abs/2201.11903) · [CoT trade-offs — Comet](https://www.comet.com/site/blog/chain-of-thought-prompting/)

---

## Deep Dive 4 — Measuring 95% F1

**Why F1, not accuracy:** extraction is sparse/imbalanced — most possible fields are absent on any given document, so "accuracy" is inflated by trivially-correct negatives. **F1** focuses on the fields actually present and balances the two real failure modes: **recall** (not missing fields) and **precision** (not inventing fields).

- **Precision** = TP / (TP + FP) — of the values we extracted, how many were correct.
- **Recall** = TP / (TP + FN) — of the true values, how many we extracted.
- **F1** = harmonic mean = **2·P·R / (P + R)**.

**How it's computed per field:** for each field on each document, a correct value = **TP**, a wrong/spurious value = **FP**, a missed field = **FN**. Two things I'd specify when asked:
- **Match criterion** — exact match for IDs (PO number), **normalized** match for dates/currency/whitespace.
- **Micro vs macro** — **micro-F1** pools TP/FP/FN across all fields (dominated by common fields); **macro-F1** averages per-field F1 (rare fields count equally). Best practice: report both plus a per-field breakdown.

Sources: [F1 for extraction — Galileo](https://galileo.ai/blog/f1-score-ai-evaluation-precision-recall) · [Micro vs macro F1 — PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8936911/)

---

## Deep Dive 5 — Fine-Tuning BERT in Azure ML

> **Why (the rationale):** BERT's bidirectional pre-training gives it rich contextual representations of email text. Fine-tuning on labeled PO/non-PO examples adapts those representations to the specific vocabulary and patterns of PO emails (supplier names, order terminology, PDF attachment headers) in a handful of epochs, matching or exceeding zero-shot LLM accuracy at a fraction of the inference cost. Azure ML's managed endpoint removes infrastructure overhead for serving the classifier at scale.
> **When to use:** Fine-tune BERT (or a smaller variant like DistilBERT) when you have sufficient labeled data (hundreds to thousands of examples), the task is well-defined (binary classification), and inference throughput or cost is a constraint. Zero-shot prompting is preferable when labeled data is unavailable or the task definition changes frequently.
> **Nuances & gotchas:** BERT overfits quickly on small datasets — 2–4 epochs with a small learning rate (~2e-5) and early stopping are standard practice, not defaults. Class imbalance (PO emails are the minority) requires class weighting or threshold adjustment; accuracy is misleading — use F1/PR-AUC. BERT's 512-token input limit can truncate long emails; for longer documents, truncation strategy (head vs tail vs head+tail) affects recall of PO signals that appear at the end of an email.

**BERT recap:** Bidirectional Encoder Representations from Transformers — an **encoder-only** Transformer that conditions on left+right context. A special **`[CLS]` token**'s final hidden state is the aggregate sequence representation fed to a classification head. Fine-tuning adds one output layer and updates all parameters on the labeled task.

**For the PO-email classifier:** binary head (PO vs not) on the `[CLS]` embedding, fine-tuned on labeled emails. Considerations: **small learning rate** (~2e-5–5e-5, Adam), **class imbalance** (PO emails are the minority → class weighting/resampling; evaluate with F1/PR, not accuracy), **few epochs** (2–4) + dropout/early stopping (BERT overfits fast).

**Azure ML workflow:** provision a **GPU compute cluster** → run the fine-tune as an Azure ML **job/experiment** (metrics tracked per run) → **register** the model to the workspace → **deploy** to a **managed online endpoint** for real-time inference → send new emails to the endpoint to classify.

Sources: [BERT (arXiv 1810.04805)](https://arxiv.org/abs/1810.04805) · [Azure ML — foundation models & fine-tuning](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-use-foundation-models?view=azureml-api-2) · [Deep learning with BERT on Azure ML — Microsoft Community Hub](https://techcommunity.microsoft.com/t5/ai-customer-engineering-team/deep-learning-with-bert-on-azure-ml-for-text-classification/ba-p/1149262)

---

## Key Decisions & Tradeoffs

| Decision | Options considered | Why chosen | Tradeoff accepted |
|---|---|---|---|
| **Two-stage gate → extract vs single LLM for both** | GPT-3.5 for classification + extraction in one call; BERT gate + GPT extract; regex heuristics for routing | Fine-tuned BERT runs ~1–2 orders of magnitude cheaper/faster than LLM token generation at high email volume; with labeled PO/non-PO data available, fine-tuned encoder matches or beats zero-shot GPT on routing accuracy | Two separate deployed systems to maintain; BERT gate recall is the single point of failure — a missed PO never reaches extraction |
| **Fine-tuned BERT vs zero-shot GPT for classification** | Zero-shot GPT-3.5 with a routing prompt; fine-tuned BERT-base; rule-based keyword filter | Labeled PO/non-PO training data was available; fine-tuned encoders match or beat zero-shot LLMs on binary classification *and* run far cheaper at scale; keywords alone miss paraphrased PO subjects | Fine-tuned BERT is a separate artifact to version, retrain, and deploy; doesn't generalize automatically when PO email patterns change without retraining |
| **Azure OpenAI GPT-3.5 vs GPT-4 for extraction** | GPT-3.5 (lower cost/latency); GPT-4 (higher accuracy, strict Structured Outputs); fine-tuned extraction model | GPT-3.5 reached acceptable F1 with CoT prompting + validation/retry; GPT-4 cost at the PO volume was not justified; fine-tuning for extraction required curated extraction training data not yet available | GPT-3.5 doesn't support strict Structured Outputs — requires JSON-mode + Pydantic validation + retry loop rather than schema-guaranteed output |
| **Chain-of-thought prompting vs direct extraction prompt** | Single "extract these fields and return JSON" prompt; CoT prompt (reason-then-JSON); few-shot examples without CoT | Complex multi-line PO tables require the model to locate the table, identify row structure, then map columns — CoT externalizes this plan and reduces field-mapping errors on dense documents | CoT increases token usage 2–4× and adds latency; economically viable only because the BERT gate restricts GPT calls to the small PO subset |
| **Azure AI Document Intelligence for PDF layout vs pdfminer/PyPDF2** | pdfminer text extraction; PyPDF2; Azure AI Document Intelligence Layout | Scanned POs (image PDFs) require OCR; even digital PDFs have multi-column tables that simple text parsers mangle (reading row-by-row incorrectly); Layout model returns table row/column structure | Azure Document Intelligence adds per-page cost and an extra API call per document; overkill for simple single-page text-only PDFs |
| **Pydantic validation + retry vs human review only** | Accept first LLM response; validate + auto-retry up to 3 times; route all low-confidence to human | LLM responses frequently have minor schema violations that a retry resolves; fully trusting the first response would lower F1; pure human review removes automation benefit | Retries increase latency and API cost per failed extraction; a malformed response that passes validation but is semantically wrong (right keys, wrong values) still slips through |

---

## Production Issues & Fixes

*Representative issues for this system class — mapped to what I actually hit; tailor to your own incidents.*

- **Symptom:** Extraction F1 drops sharply for POs from a new supplier whose PDFs use a two-column layout with line items split across columns. **Root cause:** The Azure AI Document Intelligence Layout model parsed the two-column table left-to-right across both columns, interleaving unrelated cells into the same "row" — the LLM received a structurally corrupted table. **Fix:** Added a post-Layout preprocessing step that detects tables with column-count > 6 and re-stitches them by bounding-box X-coordinate before passing to the LLM. **Prevention:** Added a supplier-level per-field F1 breakdown in the evaluation dashboard; a new supplier with <90% F1 is flagged for layout inspection before full rollout.

- **Symptom:** GPT-3.5 confidently returns a line-item quantity of "1" for every item on a 10-line PO that has varying quantities clearly printed. **Root cause:** The CoT prompt's few-shot example showed a PO where every quantity happened to be 1 — the model anchored to this pattern rather than reading the actual table. **Fix:** Replaced the single few-shot example with three diverse examples covering varying quantities, merged cells, and multi-page tables. Added a cross-check: extracted total quantity vs sum of line-item quantities must agree within 5%. **Prevention:** Added extraction-specific regression tests for known tricky PO formats; any prompt change must pass the full regression suite before deployment.

- **Symptom:** A supplier changes their PO template (new header row, renamed "Qty" column to "Units Ordered") and F1 for that supplier drops from 95% to 71% overnight. **Root cause:** CoT prompt was hardcoded to look for "Qty" and "Price" column headers; the renamed column caused a missed-field (FN) spike. **Fix:** Made column-header matching fuzzy — instructed the model to identify quantity and price columns by semantic role, not exact header text, and updated the CoT template to enumerate common aliases. **Prevention:** Monitored per-supplier F1 weekly; automated an alert when any supplier's F1 dropped >5 percentage points week-over-week.

- **Symptom:** Azure OpenAI endpoint returns HTTP 429 (throttle) errors during end-of-month invoice rush, causing extraction jobs to queue and email attachment delays of 2–4 hours. **Root cause:** Azure OpenAI rate limits (tokens-per-minute and requests-per-minute) are set per deployment; the end-of-month PO volume spike exceeded the provisioned TPM limit on the shared GPT-3.5 deployment. **Fix:** Implemented exponential backoff with jitter for 429 responses (max 3 retries, backoff starting at 2 s); requested an Azure quota increase for the GPT-3.5 deployment TPM limit; added a priority queue so time-sensitive POs jump the backlog. **Prevention:** Monitored hourly token consumption against 80% of the TPM limit; an alert fires before saturation, allowing temporary fan-out to a second deployment.

- **Symptom:** The BERT gate is passing some internal invoice emails (not POs) to the GPT extraction stage, which then returns partial JSON and wastes API budget. **Root cause:** Internal invoices share vocabulary with POs (line items, amounts, vendor names) — the BERT classifier's decision boundary at 0.5 threshold was too permissive for this edge case. **Fix:** Collected 150 misclassified examples, added them to the fine-tuning dataset, and retrained with class weighting; raised the positive-class threshold to 0.65 to prioritize precision for the gate. Also added a "no PO number extracted" fallback that discards the extraction result without attaching anything. **Prevention:** Tracked false-positive rate on the BERT gate as a weekly metric alongside recall; both must stay within target bounds.

- **Symptom:** The 95% F1 metric reported in testing doesn't hold in production; actual seller-reported accuracy is ~85%. **Root cause:** The test set was drawn from POs collected during one quarter from established suppliers — it didn't represent the full diversity of PO layouts from newer suppliers, scanned legacy documents, and multilingual headers. **Fix:** Re-evaluated F1 on a stratified sample covering supplier age, PDF type (digital vs scanned), and language; found specific subgroups driving the gap and prioritized those in the training refresh. **Prevention:** Quarterly F1 re-evaluation on a fresh stratified gold-labeled sample; surface per-stratum breakdowns, not just overall F1.

---

## Interview Q&A

**Q: Why a two-stage design instead of one model doing everything?**
BERT is a cheap, fast filter over *all* email; GPT extraction is expensive and only runs on the small PO subset. Encoders are ~1–2 orders of magnitude cheaper/faster and much higher throughput than LLM prompting, so gating with BERT is the cost-correct architecture. It also separates routing (high-volume, binary) from extraction (complex, schema-driven).

**Q: Why fine-tune BERT instead of using GPT for classification too?**
At high volume, fine-tuned BERT matches or beats zero-shot LLMs on accuracy *and* runs far cheaper/faster (single forward pass vs token-by-token). With labeled PO/non-PO data available, fine-tuning is the better trade-off; GPT's open-ended reasoning is wasted on a binary decision.

**Q: How did you measure 95% F1?**
Per-field TP/FP/FN against a gold-labeled test set; F1 = harmonic mean of precision and recall. I'd specify exact-match for IDs and normalized match for dates/currency, and whether 95% is micro (pooled) or macro (per-field average). F1 over accuracy because extraction is sparse — accuracy is inflated by absent fields.

**Q: How did chain-of-thought help?**
Forcing step-by-step reasoning (locate table → parse rows → extract fields → assemble JSON) before output improves accuracy on complex, multi-line POs. Trade-off is 2–4× more tokens and latency, which we accepted only on the PO subset thanks to the BERT gate.

**Q: How do you handle hallucinated or wrong extractions?**
Schema constraint (JSON schema / function-calling) so structure is valid; validation + retry (Pydantic); grounding — extract only from source text and cross-check numeric fields (line-item sums vs total); and human-in-the-loop review for low-confidence cases.

**Q: How did you handle scanned PDFs?**
The LLM can't read pixels, so scanned PDFs go through OCR/layout first (e.g., Azure AI Document Intelligence Layout, which returns table row/column structure), and the clean structured text is fed to the model.

---

## Honest Caveats

- **GPT-3.5 does *not* support strict Structured Outputs** (GPT-4o+ only). On 3.5 I constrained output via JSON-mode / function-calling + a validation-and-retry loop — **don't claim strict schema guarantees on 3.5.**
- **Confirm the OCR step** actually used for scanned POs (Document Intelligence vs another engine) — a likely follow-up.
- **Specify micro vs macro F1** and the exact/fuzzy match rules behind "95% F1."
- BERT fine-tuning LR/epochs are standard ranges — verify exact values before quoting.
- The **40% friction reduction** and **95% F1** are the project's own numbers; external throughput/token figures are cited from papers — keep them attributed.

---

## References

- [BERT: Pre-training of Deep Bidirectional Transformers (arXiv 1810.04805)](https://arxiv.org/abs/1810.04805)
- [Chain-of-Thought Prompting (arXiv 2201.11903)](https://arxiv.org/abs/2201.11903)
- [Azure ML — foundation models & fine-tuning](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-use-foundation-models?view=azureml-api-2) · [BERT on Azure ML — Community Hub](https://techcommunity.microsoft.com/t5/ai-customer-engineering-team/deep-learning-with-bert-on-azure-ml-for-text-classification/ba-p/1149262)
- [Azure AI Document Intelligence — Read/Layout OCR](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/read?view=doc-intel-4.0.0) · [Azure OpenAI — Structured Outputs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/structured-outputs)
- [F1 for extraction — Galileo](https://galileo.ai/blog/f1-score-ai-evaluation-precision-recall) · [Micro vs macro F1 — PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8936911/)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **PO (Purchase Order)** | A buyer's formal document listing items, quantities, prices, and delivery terms for a vendor | The document being extracted and attached to the matching sales quote |
| **BERT** | Bidirectional Encoder Representations from Transformers — an encoder-only Transformer pre-trained on masked language modeling | Produces rich contextual text embeddings used for classification after fine-tuning |
| **Encoder-Only Transformer** | A Transformer architecture that reads the full input in both directions but does not generate text | Ideal for classification and extraction tasks; faster and cheaper than generative models |
| **[CLS] Token** | A special token prepended to BERT input whose final hidden state summarizes the whole sequence | The representation fed to the classification head for a sentence-level label |
| **Fine-Tuning** | Continuing to train a pre-trained model on a new, smaller, task-specific labeled dataset | Adapts BERT's general language knowledge to the PO/non-PO classification task |
| **Binary Classifier** | A model that outputs one of exactly two labels (here: PO email vs not) | Acts as a cheap, fast gate over all incoming emails before the expensive LLM stage |
| **Two-Stage Architecture** | A pipeline with a fast/cheap first stage (filter) and a slow/expensive second stage (extract) | Applies the costly LLM only to the small subset of emails that pass the first gate |
| **GPT-3.5** | OpenAI's mid-tier instruction-following generative language model | Used for structured PO field extraction with chain-of-thought prompting |
| **Chain-of-Thought (CoT) Prompting** | A prompting technique that instructs the model to reason through steps before emitting the final answer | Reduces field-mapping errors on complex multi-line tables by making the model plan first |
| **Structured Extraction** | Using a model to parse a document and return fields in a defined schema (e.g., JSON) | Converts unstructured PO PDFs into machine-readable data for CRM attachment |
| **JSON-Mode / Function-Calling** | OpenAI API features that constrain model output to valid JSON or a predefined argument schema | Enforces output structure on GPT-3.5 (strict Structured Outputs require GPT-4o+) |
| **Pydantic Validation** | Using Python's Pydantic library to validate that extracted JSON matches the expected schema | Catches structurally wrong extractions before they reach downstream systems |
| **OCR (Optical Character Recognition)** | Technology that reads text from images or scanned PDFs | Required for scanned PO PDFs whose text is not embedded as extractable characters |
| **Azure AI Document Intelligence** | Microsoft's managed OCR and form-parsing service that returns text, key-value pairs, and table structure | Converts scanned POs into clean structured text fed to the LLM |
| **F1 Score** | Harmonic mean of precision and recall; 2·P·R / (P+R) | Balanced metric for extraction quality; robust to sparse fields that inflate accuracy |
| **Precision** | Of the fields we extracted, the fraction that were correct | Measures how often the model invents wrong values |
| **Recall** | Of all true fields present, the fraction we extracted | Measures how often the model misses real fields |
| **Micro-F1** | F1 computed by pooling all TP/FP/FN counts across every field type | Dominated by the most common fields; gives an overall extraction score |
| **Macro-F1** | Average of per-field F1 scores | Treats rare and common fields equally; reveals if rare fields are poorly extracted |
| **Azure ML** | Microsoft's managed machine learning platform for training, tracking, and deploying models | Hosts the BERT fine-tuning job and serves the classifier as a managed online endpoint |
| **Managed Online Endpoint** | An Azure ML-managed REST endpoint that serves a registered model with auto-scaling | Provides production-ready inference for the BERT classifier without infrastructure management |
| **Class Imbalance** | When the minority class (PO emails) is far rarer than the majority class | Requires class weighting or resampling; makes accuracy a misleading metric |
| **Hallucination (LLM)** | Text the model generates that is not supported by the input document | Mitigated by schema constraints, Pydantic validation, and cross-checking numeric totals |

---

*Previous: [Shipping-Time Forecasting](06-shipping-time-forecasting-catboost.md) | Next: [Seller Behavior Analytics](08-seller-behavior-analytics.md) | Up: [Guide Home](../README.md)*
